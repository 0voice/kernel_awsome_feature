本blog包括理论和实践两个部分，实践部分需要您事先部署成功Ceph集群！

参考《Ceph设计与实现》谢型果等，第二章。以及[官方BlueStore教程](https://docs.ceph.com/en/latest/rados/operations/bluestore-migration/)。

推荐博客[Ceph存储引擎BlueStore简析](http://www.itworld123.com/2019/06/04/storage/ceph/Ceph存储引擎BlueStore简析/)

# 下一代对象存储引擎BlueStore

相比于目前FileStore，BlueStore拥有无与伦比的优势：

- 充分考虑下一代全SSD以及NVMe SSD闪存阵列的适配。例如将高效索引元数据的引擎由LevelDB替换为RocksDB。
- 传统的基于POSIX接口的FileStore需要通过操作系统自带的文件系统间接管理磁盘。BlueStore选择绕开文件系统，从而使得I/O路径大大减小。
- 在设计中将元素据和用户数据严格分离，因此元素据可以单独采用高速固态存储设备，诸如NVMe SSD，以实现性能加速。
- 与传统机械硬盘相比，SSD普遍采用4k 或者更大的块大小，因此采用位图进行管理可以获得更高的空间收益。

## 1 设计理念

在存储系统中，所有读操作都是同步的，即除非在缓存命中，否则必须从磁盘中读到指定内容才向客户端返回。而写操作则不一样，一般处于效率考虑，所有写操作都会在内存中进行缓存，由文件系统进行组织后再批量写入磁盘。

数据可靠性：我们考虑写的期间发生断电的情况，因为内存是易失性的，所有数据会丢失。针对这个问题，有人提出用一个掉电不丢失的中间设备作为过渡设备，等数据写入普通磁盘后再释放中间设备上的空间，这个写中间设备的过程被称为**写日志**。中间设备被称为日志设备。但这样会消耗额外硬件资源。

数据一致性：数据修改要么全部完成，要么没有变化（All or nothing）. 具体而言，我们用ACID（A: Atomicity, C: Consistency, I:Isolation, D:Durability）来描述这种系统，即**事务型系统**。

**术语**

块大小： 指对磁盘进行操作的最小粒度。 对普通机械硬盘为512字节，而SSD为4KB。

RMW：覆盖写。 如果本次改写的内容不足一个块，那么需要将对应的块读进来，将待修改的内容与原先内容进行合并。它的问题在于：额外的读惩罚，以及潜在的数据丢失风险。

COW：写时重定向。在磁盘分配新的空间，再写入，写完成后再释放旧数据。

## 2 BlueStore写策略

BlueStore综合运用了RMW和COW，任何一个写请求，根据磁盘块大小，分为三个部分，即首尾非块大小对齐部分和中间块大小对齐部分，针对两边RMW，针对中间采用COW。

BlueStore提供的读写访问接口都是基于PG粒度的。

## 3 缓存替换机制

LRU算法：最近最少使用，时间局部性原理。

LFU算法：最近不经常使用，SDD访问模型。

ARC算法，同时考虑了LRU和LFU的长处，同时使用两个队列对缓存中页面进行管理：

- MRU (Most Recently Used) 队列保存最近访问过的页面
- MFU（Most Frequently Used）队列保存最近一段时间**至少被访问过两次**的界面。
- 两个队列的长度是可变的，会根据请求队列的特征自动进行调整，取LRU和LFU共同之所长。
  - 当系统中请求序列呈现明显的时间局部性，MFU队列长度变为0，从而退化为LRU。
  - 当系统中请求序列呈现明显的空间局部性，MRU队列长度变为0，从而退化为LFU。

2Q算法：双队列热点算法，一种针对数据库特别是关系数据库系统优化的缓存淘汰算法：

数据库系统由于需要保证每个操作的原子性，所以经常存在多个事务操作同一块热点数据的场景，因此针对数据库系统的缓存淘汰算法主要关注如何识别多个并发事务之间的数据相关性。

与ARC类似，2Q也使用了多个队列来管理整个缓存空间，分布称为A1in,A1out,AmA1in,A1out,Am。这些队列都是LRU队列，其中A1inA1in与AmAm是真正的缓存队列，A1outA1out是影子队列，i.e.只保存相关页面的管理结构。

- 新的页面一开始总是被加入A1in，当某个页面被频繁访问，2Q认为这些访问是相关的，不会针对该页面执行任何热度提升的操作，直到其被正常淘汰至Aout。这个时间间隔被称为“相关时间间隔”。
- 当A1out中某个页面被再次访问时，2Q认为这些访问不再相关，此时执行页面热度提升，将其加入Am头部。Am队列中的页面再次被命中时，同样将其加入Am队列头部进行页面热度提升。从Am中淘汰的页面也进入A1out。这个时间间隔被称为“热度保留间隔”。

## 4 缓存管理

BlueStore 目前采用了LRU和2Q两种算法。

参考Theodore和Dennis的测试结论，推荐A1in和Am队列的容量配比1:1.

BlueStore的cache既可以用于缓存用户数据，也可以用于缓存元数据。bluestore中默认元数据的比重位90%。

BlueStore中元素据分为两类：Collection和Onode. Collection是PG在BlueStore中内存管理结构。每个OSD最多承载100个PG而且Collection管理结构本身比较小，故被设计成常驻内存。而Onode的数量和其管理的磁盘空间成正比，因而不可能常驻内存，需要引入淘汰机制。Onode采用LRU。

## 5 BlueFS

诞生于2011年的LevelDB是基于Google的BigTable数据库系统发展而来。然而随着SSD普及，LevelDB无法发挥SSD全部性能，因而诞生了RocksDB。

- RocksDB适合存储小型或者中型键值对；性能随着键值对长度上升下降很快。
- 性能随CPU核数以及后端存储设备的I/O能力呈线性扩展。

传统的本地文件系统（XFS，ext4，ZFS）等不能与RocksDB完全兼容，因而专门为其量身打造一款本地文件系统——BlueFS。在逻辑空间上分为三个层次

（1）慢速空间

主要用于存储对象数据，可由大容量机械硬盘担任存储。

（2）高速空间（DB）

主要存储BlueStore内部的元素据，比如Onode。 可以由SSD提供。

（3）超高速（WAL）

WAL(Write Ahead Log)指日志。 可以由NVMe SSD或NVRAM等高速设备充当。

BlueFS上的磁盘数据包括文件、目录、日志三种类型。其定位文件分为两步：1. 通过`dir_map`找到文件的最底层文件夹 2.通过`file_map`找到对应的文件。其磁盘数据结构如下：

|    成员     |                    含义                    |
| :---------: | :----------------------------------------: |
|     ino     |             唯一标识一个fnode              |
|    size     |                  文件大小                  |
|    mtime    |            文件上一次被修改时间            |
| prefer_bdev |          存储该文件优先使用的设备          |
|   extents   | 磁盘上物理段集合包括{bdev，offset，length} |

[![image-20201202001057524](https://durantthorvalds.top/img/image-20201202001057524.png)](https://durantthorvalds.top/img/image-20201202001057524.png)

## 6 ObjectStore(OS)

Ceph是一个指导原则是所有存储的不管是块设备、对象存储、文件存储最后都转化成了底层的对象object，这个object包含3个元素data，xattr，omap。data是保存对象的数据；xattr是保存对象的扩展属性，每个对象文件都可以设置文件的属性，这个属性是一个key/value值对，这类操作的特征是kv对并且与某一个Object关联，但是受到文件系统的限制，key/value对的个数和每个value的大小都进行了限制。如果要设置的对象的key/value不能存储在文件的扩展属性中；还存在另外一种方式保存omap(在Ceph中称为omap)，omap实际上是保存到了key/vaule 值对的RocksDB中，在这里value的值限制要比xattr中好的多。

对于FileStore实现，每个Object在FileStore层会被看成是一个文件，Object的属性(xattr)会利用文件的xattr属性存取，因为有些文件系统(如Ext4)对xattr的长度有限制，因此超出长度的Metadata会被存储在DBObjectMap里。而Object的omap则直接利用DBObjectMap实现。因此，可以看出xattr和omap操作是互通的，在用户角度来说，前者可以看作是受限的长度，后者更宽泛(API没有对这些做出硬性要求)。目前纠删码还不支持omap。

而在BlueStore则没有这种限制。

------

# 部署和操作BlueStore

# BLUESTORE迁移

每个OSD都可以运行BlueStore或FileStore，并且单个Ceph集群可以包含两者的混合。先前已部署FileStore的用户可能希望过渡到BlueStore，以利用改进的性能和健壮性。有几种策略可以实现这种过渡。

单个OSD不能单独进行原地转换，但是：BlueStore和FileStore根本不同，以致于无法实用。“转换”将依靠群集的正常复制和修复支持，或者依靠将OSD内容从旧的（FileStore）设备复制到新的（BlueStore）设备的工具和策略。

## 部署新的OSD与BLUESTORE

可以使用BlueStore部署任何新的OSD（例如，在扩展群集时）。这是默认行为，因此不需要进行特定更改。

同样，更换故障驱动器后重新配置的任何OSD都可以使用BlueStore。

## 将现有的OSD

### 标记并替换

最简单的方法是依次标记每个设备，等待数据在群集中复制，重新配置OSD，然后再次将其标记回。它很容易实现自动化。但是，它需要的数据迁移量超出了必要，因此不是最佳选择。

1. 确定要替换的FileStore OSD：

   ```
   ID=<osd-id-number>
   DEVICE=<disk-device>Copy
   ```

   您可以使用以下命令判断给定的OSD是FileStore还是BlueStore：

   ```
   ceph osd metadata $ID | grep osd_objectstoreCopy
   ```

   您可以使用以下命令获取文件存储与bluestore的当前计数：

   ```
   ceph osd count-metadata osd_objectstoreCopy
   ```

2. 将文件存储OSD标记为：

   ```
   ceph osd out $IDCopy
   ```

3. 等待数据从有问题的OSD迁移：

   ```
   while ! ceph osd safe-to-destroy $ID ; do sleep 60 ; doneCopy
   ```

4. 停止OSD：

   ```
   systemctl kill ceph-osd@$IDCopy
   ```

5. 记下此OSD使用的设备：

   ```
   mount | grep /var/lib/ceph/osd/ceph-$IDCopy
   ```

6. 卸载OSD：

   ```
   umount /var/lib/ceph/osd/ceph-$IDCopy
   ```

7. 销毁OSD数据。请*格外小心，*因为这会破坏设备的内容；在继续操作之前，请确保不需要设备上的数据（即，群集运行状况良好）。

   ```
   ceph-volume lvm zap $DEVICECopy
   ```

8. 告诉集群OSD已被破坏（并且可以使用相同的ID重新配置新的OSD）：

   ```
   ceph osd destroy $ID --yes-i-really-mean-itCopy
   ```

9. 使用相同的OSD ID在其位置重新配置BlueStore OSD。这要求您确实根据上面看到的内容确定要擦除的设备。小心！

   ```
   ceph-volume lvm create --bluestore --data $DEVICE --osd-id $IDCopy
   ```

10. 重复。

您可以允许替换OSD的重新填充与下一个OSD的排空同时进行，或者对多个OSD并行执行相同的步骤，只要确保在销毁群集之前群集是完全干净的（所有数据具有所有副本）即可。任何OSD。否则，将减少数据的冗余，并增加（甚至可能导致）数据丢失的风险。

优点：

- 简单。
- 可以逐个设备完成。
- 不需要备用设备或主机。

缺点：

- 数据通过网络复制了两次：一次复制到集群中的其他OSD（以保持所需的副本数），然后再次返回到重新配置的BlueStore OSD。

### 整个主机更换

如果集群中有一个备用主机，或者有足够的可用空间来疏散整个主机以用作备用主机，则可以在每个主机的基础上使用存储的每个数据副本进行转换仅迁移一次。

首先，您需要有一个没有数据的空主机。有两种方法可以执行此操作：从尚未包含在群集中的新的空主机开始，或者从群集中现有主机上卸载数据。

#### 使用新的，空的主机

理想情况下，主机应具有与将要转换的其他主机大致相同的容量（尽管并不严格）。

```
NEWHOST=<empty-host-name>Copy
```

将主机添加到CRUSH层次结构，但不要将其附加到根目录：

```
ceph osd crush add-bucket $NEWHOST hostCopy
```

确保已安装ceph软件包。

#### 使用现有的主机

如果要使用已经是群集一部分的现有主机，并且该主机上有足够的可用空间，以便可以迁移其所有数据，则可以执行以下操作：

```
OLDHOST=<existing-cluster-host-to-offload>
ceph osd crush unlink $OLDHOST defaultCopy
```

其中“默认”是CRUSH地图中的直接祖先。（对于具有未修改配置的较小群集，通常将是“默认”，但也可能是机架名称。）现在，您应该在OSD树输出的顶部看到没有父节点的主机：

```
$ bin/ceph osd tree
ID CLASS WEIGHT  TYPE NAME     STATUS REWEIGHT PRI-AFF
-5             0 host oldhost
10   ssd 1.00000     osd.10        up  1.00000 1.00000
11   ssd 1.00000     osd.11        up  1.00000 1.00000
12   ssd 1.00000     osd.12        up  1.00000 1.00000
-1       3.00000 root default
-2       3.00000     host foo
 0   ssd 1.00000         osd.0     up  1.00000 1.00000
 1   ssd 1.00000         osd.1     up  1.00000 1.00000
 2   ssd 1.00000         osd.2     up  1.00000 1.00000
...Copy
```

如果一切正常，请直接跳到下面的“等待数据迁移完成”步骤，然后从那里继续进行操作以清理旧的OSD。

#### 迁移过程

如果您使用的是新主机，请从步骤1开始。对于现有主机，请跳至下面的步骤5。

1. 为所有设备配置新的BlueStore OSD：

   ```
   ceph-volume lvm create --bluestore --data /dev/$DEVICECopy
   ```

2. 验证OSD通过以下方式加入集群：

   ```
   ceph osd treeCopy
   ```

   您应该看到新主机`$NEWHOST`与它下面的所有的OSD的，但主机应该*不*被嵌套任何其他节点下的层次结构（像）。例如，如果是空主机，则可能会看到以下内容：`root default``newhost`

   ```
   $ bin/ceph osd tree
   ID CLASS WEIGHT  TYPE NAME     STATUS REWEIGHT PRI-AFF
   -5             0 host newhost
   10   ssd 1.00000     osd.10        up  1.00000 1.00000
   11   ssd 1.00000     osd.11        up  1.00000 1.00000
   12   ssd 1.00000     osd.12        up  1.00000 1.00000
   -1       3.00000 root default
   -2       3.00000     host oldhost1
    0   ssd 1.00000         osd.0     up  1.00000 1.00000
    1   ssd 1.00000         osd.1     up  1.00000 1.00000
    2   ssd 1.00000         osd.2     up  1.00000 1.00000
   ...Copy
   ```

3. 确定要转换的第一个目标主机

   ```
   OLDHOST=<existing-cluster-host-to-convert>Copy
   ```

4. 将新主机交换到群集中旧主机的位置：

   ```
   ceph osd crush swap-bucket $NEWHOST $OLDHOSTCopy
   ```

   此时，所有数据`$OLDHOST`将开始迁移到上的OSD `$NEWHOST`。如果新旧主机的总容量不同，您可能还会看到一些数据迁移到集群中的其他节点或从集群的其他节点迁移，但是只要这些主机的大小相同，这将是相对少量的数据。

5. 等待数据迁移完成：

   ```
   while ! ceph osd safe-to-destroy $(ceph osd ls-tree $OLDHOST); do sleep 60 ; doneCopy
   ```

6. 停止所有空的旧OSD `$OLDHOST`：

   ```
   ssh $OLDHOST
   systemctl kill ceph-osd.target
   umount /var/lib/ceph/osd/ceph-*Copy
   ```

7. 销毁并清除旧的OSD：

   ```
   for osd in `ceph osd ls-tree $OLDHOST`; do
       ceph osd purge $osd --yes-i-really-mean-it
   doneCopy
   ```

8. 擦拭旧的OSD设备。这要求您确定要手动擦除哪些设备（请小心！）。对于每个设备：

   ```
   ceph-volume lvm zap $DEVICECopy
   ```

9. 将现在为空的主机用作新主机，然后重复：

   ```
   NEWHOST=$OLDHOSTCopy
   ```

优点：

- 数据只能通过网络复制一次。
- 一次转换整个主机的OSD。
- 可以并行转换为一次转换多个主机。
- 每个主机上都不需要备用设备。

缺点：

- 需要备用主机。
- 整个主机的OSD值将同时迁移数据。这很可能会影响整个群集的性能。
- 所有迁移的数据仍然在网络上进行了一整跳。

### 每OSD设备副本

可以使用的`copy`功能转换单个逻辑OSD `ceph-objectstore-tool`。这要求主机具有一个或多个空闲设备来供应新的空BlueStore OSD。例如，如果群集中的每个主机都有12个OSD，则需要第13个可用设备，以便可以依次转换每个OSD，然后再收回旧设备以转换下一个OSD。

注意事项：

- 此策略要求准备一个空白的BlueStore OSD，而无需分配该`ceph-volume` 工具不支持的新OSD ID 。更重要的是，*dmcrypt*的设置与OSD身份紧密相关，这意味着该方法不适用于加密的OSD。
- 设备必须手动分区。
- 工具未实现！
- 没有记录！

优点：

- 在转换期间，很少或没有数据在网络上迁移。

缺点：

- 工具尚未完全实现。
- 流程未记录。
- 每个主机必须具有备用或空设备。
- OSD在转换过程中处于脱机状态，这意味着新的写入操作将仅写入OSD的一部分。这会增加由于后续故障而导致数据丢失的风险。（但是，如果在转换完成之前出现故障，则可以启动原始FileStore OSD来提供对其原始数据的访问。）
