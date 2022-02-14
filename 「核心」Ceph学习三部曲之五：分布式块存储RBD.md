本blog包括理论和实践两个部分，实践部分需要您事先部署成功Ceph集群！

参考《Ceph设计与实现》谢型果等，第六章。以及[官方RADOS指南](https://docs.ceph.com/en/latest/man/8/rados/)，以及[官方RBD教程](https://docs.ceph.com/en/latest/man/8/rbd/)

# 无心插柳——分布式块存储RBD

> RADOS(Reliable, Autonomic Distributed Object Store)
>
> **rbd**是用于处理rados块设备（RBD）映像的实用程序，由Linux rbd驱动程序和QEMU / KVM的rbd存储驱动程序使用。RBD映像是简单的块设备，在对象上划分条带并存储在RADOS对象存储中。分割image的对象的大小必须是2的幂。——来自官方

RBD(RADOSBlockDeviceRADOSBlockDevice)指分布式块存储服务组件，是Ceph对外的三大存储服务组件之一。另外两个分别是CephFS以及Radosgw 。我们将在后面介绍。上层应用访问RBD有两种途径：librbd以及krbd。其中librbd是基于librados的用户态接口库，而krbd是继承在GNU/Linux内核中的一个内核模块。通过librbd命令行工具，将RBD块设备映射为本地的一个块设备文件。

[![img](https://durantthorvalds.top/img/rados.png)](https://durantthorvalds.top/img/rados.png)

一个块通常是512字节，块设备接口无处不在，非常适合与包括Ceph在内的海量数据存储进行交互。Ceph块设备是精简配置的，可调整大小的，并且可以在多个OSD上条带化存储数据。Ceph块设备利用了 RADOS功能，包括快照，复制和强大的一致性。Ceph块存储客户端通过内核模块或`librbd`库与Ceph集群通信。

Ceph的的块设备提供与广阔的可扩展性，高性能的 [内核模块](https://docs.ceph.com/en/latest/rbd/rbd-ko/)，或者KVM系列如[QEMU](https://docs.ceph.com/en/latest/rbd/qemu-rbd/)和基于云计算系统，如[OpenStack的](https://docs.ceph.com/en/latest/rbd/rbd-openstack)和[的CloudStack](https://docs.ceph.com/en/latest/rbd/rbd-cloudstack)依赖的libvirt和QEMU与Ceph的块设备集成。可以同一群集同时运行[Ceph RADOS网关](https://docs.ceph.com/en/latest/radosgw/#object-gateway)， [Ceph文件系统](https://docs.ceph.com/en/latest/cephfs/#ceph-file-system)和Ceph块设备。

[![image-20201220224241218](https://durantthorvalds.top/img/rbd1.png)](https://durantthorvalds.top/img/rbd1.png)

RBD架构如上图所示，由于元数据信息非常少，且访问不频繁，因此RBD在Ceph集群中不需要daemon守护进程直接将元数据加载到内存进行元数据访问加速，所有数据操作直接与MON和OSD进行交互。

## §1§1 元数据

RBD块设备在Ceph中被称为image，由元数据和数据组成。其中元数据有三种存储方式，第一种将元数据编码后以二进制文件的形式存储在RADOS对象的数据部分，后面将该类型标识为data。第二种将元数据以键值对的形式存储在RADOS对象的扩展属性中，称为xattr；第三种将元数据以键值对形式存储在RADOS对象omap中，称为omap。更多关于元数据的讨论见[BlueStore](https://durantthorvalds.top/2020/12/27/下一代对象存储引擎BlueStore/#6-ObjectStore-OS)

### 1.1 image元数据对象

- `rbd_id.<name>`，data类型，记录image名称到image id 的单向映射关系。
- `rbd_header.<id>`，omap\xattr类型，记录image所支持的功能特性、容量大小等基本信息以及配置参数、自定义元数据，锁信息等。
- `rbd_object_map.<id>`，data类型，记录组成image的所有数据对象的存在状态。

[![image-20201221173651999](https://durantthorvalds.top/img/rbd3.png)](https://durantthorvalds.top/img/rbd3.png)

通常情况下，image的数据和元数据存储在同一个存储池下。但是当前纠删码池不支持omap，必须将数据对象和元数据分开存储，需要一个独立的元数据`data_pool_id`用于记录数据对象所在的存储池。

§1§1 rbd_id

image内部的元数据和数据的名称以id为基础，这样即使image重命名，内部结构也基本不发生改变。

§2§2 rbd_header

这是image最主要的元数据对象，其对象名由rbd_header.,表示rbd_id所记录的内部id。

§3§3 rbd_object_map

为了解决克隆image数据I/O对象执行时间过长的问题，Ceph引入object-map，它将所有数据对象的存在记录在一个独立的元数据对象，总共有四种状态，b00对象不存在、b01对象存在、b10对象待删除、b11对象存在且从第一次快照创建后没有进行写。

### 1.2 RBD管理元数据对象

- rbd_directory

  omap类型。记录存储池中所有image列表。其中 `name_<name>` 记录image名称所对应的image id；`id_<id>`记录image名称。

- rbd_children

  omap类型。记录父image快照到克隆image之间的单向映射关系（parent->children）。其中`<parent>`记录当前存储池下基于父image快照创建的一个或多个克隆image id列表。由三个字段组成，用于表示克隆image所关联的父image快照，而元数据内容为克隆image的id集合。

## §2§2 数据

如rbd_header的定义，在创建image时可以通过`object-size`控制数据对象的容量大小，默认为4MB，image数据以该大小为单元进行等量划分。每个数据对象的名称由rbd_header元数据中对象前缀和对象序号组成。

**数据条带化**

RBD image在许多对象上分条，然后由Ceph分布式对象存储（RADOS）存储。结果，对图像的读取和写入请求分布在群集中的许多节点上，通常可以防止在单个image变大或繁忙时任何单个节点成为瓶颈。默认为类似于RAID-0的方式进行条带化（Stripping V2）.

条带化由三个参数控制：

- `object-size`

  我们分割的对象的大小是2的幂。将四舍五入到最接近的2的幂。默认对象大小为4 MB，最小为4K，最大为32M。

- `stripe_unit`

  在继续下一个对象之前，每个[ *stripe_unit* ]连续字节都存储在同一对象中。

- `stripe_count`

  之后，我们写[ *stripe_unit* ]字节为[ *stripe_count* ]对象，我们循环回到初始对象和写入另一个条纹，直到对象达到其最大尺寸。在这一点上，我们继续下一个[ *stripe_count* ]对象。

默认情况下，[ *stripe_unit* ]与对象大小相同，[ *stripe_count* ]为1。指定不同的[ *stripe_unit* ]和/或[ *stripe_count* ]通常被称为使用“花式”条带，并且需要v2。

## §3§3 功能特性

### RADOS快照

由于快照的存在，一个RADOS对象可能由一个head对象和多个克隆组成。在OSD端使用SnapSet结构体来保存对象的快照信息，其中clone_overlap字段记录clone对象与head对象的数据内容重叠的区间，在数据恢复时，可以减少OSD直接的信息传输。

RADOS对象创建快照后数据读取流程非常简单，RADOS客户端在读操作中携带需要读取的RADOS对象的snapid，通过snapid定位到clone对象或head对象即可读到相应的数据。

假设初始时的head对象是一个完整的使用默认4MB大小的对象，且之前未有过COW操作。对该对象制作快照snap1，然后写数据至[512K~1M]区间，此时会触发COW，通过底层的克隆操作生成一个clone对象clone1，然后将新数据写入head对象。写操作会同步更新这个head对象上记录的clone_overlap[snap1]，对于一个新的快照对象一开始这个重叠区间是整个对象的[0 ~ 4M]， 然后每个新的写入操作会在这个区间减去新写的区间。

### RBD快照

RBD快照只需要保存少量的快照元数据信息，其底层数据I/O的实现完全依赖于RADOS快照实现，

### 克隆

RBD克隆是在RBD快照基础上实现的可写快照，与RBD快照功能相似，RBD克隆的实现也依赖COW。与RBD快照不同的是 快照功能依赖于RADOS层的对象快照实现，但是功能完全在RBD客户端实现。

创建克隆image的过程基本上就是创建一个新的image，但是在image的元数据中会记录一个parent键值对，也就是记录克隆image与快照相连的父子关系。所以才会有parent和rbd_children等属性。由于克隆关系可能存在多层，因此RBD客户端会尝试访问最顶层的parent。

**如果访问克隆对象遇到不一致如何处理**？

克隆image读流程

- RBD客户端读取指定区间的数据，假设该区间最终落到第一个数据对象；
- 由于第一个对象不存在，故会返回对象不存在错误；
- RBD会访问parent并得到关联的快照信息，需要注意的是，创建克隆image时可以指定与快照image不同的条带化参数，因此从快照image读取数据的区间可能落到与第一步不同的数据对象；
- 读操作返回

克隆image写流程

- RBD客户端读取指定区间的数据，假设该区间最终落到第一个数据对象；
- 由于第一个对象不存在，故会返回对象不存在错误；
- 与读操作不同，写入操作需要从快照image读取克隆image第一个数据对象整个对象区间的数据；
- 对快照image的读操作返回；
- 将从快照读取的数据写入克隆image的第一个数据对象，然后重新执行原始的针对克隆image的第一个数据对象的写操作，最终写操作完成。

------

# 实践部分

## 创建和删除

1. 我们可以创建一个rbd_image并且指定它的大小：

   ```
   rbd create mypool/myimage --size 102400Copy
   ```

   或者指定对象大小（8M）

   ```
   rbd create mypool/myimage --size 102400 --object-size 8MCopy
   ```

2. 删除（小心！）

```
rbd rm mypool/myimageCopy
```

1. 创建快照

```
rbd snap create mypool/myimage@mysnapCopy
```

## 获取rbd_id

```
rados get -p rbd rbd_id.<image_name> file_rbd_id
cat file_rbd_idCopy
```

可以获得类似`ac62a15cbf99`这样的id标识。

- features 已启用的功能特性

| 特性           | bit位  | 注解                                                         |
| -------------- | ------ | ------------------------------------------------------------ |
| layering       | 0(LSB) | 是否支持image克隆操作，克隆image与关联的父image快照之间通过COW实现数据共享 |
| striping       | 1      | 是否进行数据对象间数据条带化，类似于RAID 0，在创建image时如果指定了条带化参数，数据会在多个image数据对象之间进行条带化 |
| exclusive-lock | 2      | 是否支持分布式锁，即image自带互斥访问锁机制以限制以限制同时只能有一个客户端访问image，主要应用于虚拟机热迁移 |
| object-map     | 3      | 是否记录组成image的数据对象存在的状态位图，通过查表加速类似于导入、导出、克隆分离、已使用容量计算等操作，同时有助于减少COW机制带来的克隆image的I/O时延，依赖于exclusive-lock特性 |
| fast-diff      | 4      | 用于计算快照间增量数据等操作加速，依赖于object-map特性       |
| deep-flatten   | 5      | 克隆分离时十分同时解除克隆image创建的快照与父image之间的关联关系。该特性只是为了阻止老的RBD客户端访问image而设置 |
| journaing      | 6      | 是否记录image修改操作到日志对象，用于远程异步镜像功能，依赖于exclusive-lock特性 |
| data-pool      | 7(MSB) | 是否将数据对象存储于与元数据不同的存储池，用于支持将image的数据对象存储于EC纠删码存储池 |

------

```
value (8 bytes) :
00000000  3d 00 00 00 00 00 00 00                           |=.......|
00000008Copy
```

0x3d对应二进制`0'b00111101` 表示启用了layering、exclusive-lock、object-map，fast-diff，deep-flatten等特性。

- object_prefix 数据对象名称前缀

  ```
  value (25 bytes) :
  00000000  15 00 00 00 72 62 64 5f  64 61 74 61 2e 61 63 36  |....rbd_data.ac6|
  00000010  32 61 31 35 63 62 66 39  39                       |2a15cbf99|
  00000019Copy
  ```

- order 组成image的数据对象容量大小，以2为底的指数

  ```
  value (1 bytes) :
  00000000  16                                                |.|
  00000001Copy
  ```

- parent 当存在克隆关系时，克隆image记录的关联的父image快照信息

- size 容量大小

  ```
  value (8 bytes) :
  00000000  00 00 00 40 00 00 00 00                           |...@....|
  00000008Copy
  ```

  64位整型，上述表示1GB

- snap_seq 用于记录image最后一次创建的快照的id

  ```
  value (8 bytes) :
  00000000  04 00 00 00 00 00 00 00                           |........|
  00000008Copy
  ```

  上图表示最后一次创建快照id为0x04

- snap_所记录的是一个cls_rbd_snap结构体实例，记录快照名称、id等基本信息。

  ```
  rados getomapval -p cephfs_data rbd_header.ac62a15cbf99 snapshot_0000000000000004 f_snap
  ceph-decoder type cls_rbd_snap import \
  > f_snap decode dump_json
  ceph-dencoder type cls_rbd_snap import f_snap decode dump_jsonCopy
  ```

  ```
  {
      "id": 4,
      "name": "snap1",
      "image_size": 1073741824,
      "protection_status": "unprotected",
      "child_count": 0
  }Copy
  ```

  与普通的image不同，克隆image在创建时不能指定容量大小，而是由image_size决定克隆image的初始容量大小。features是父image在创建快照时的features元数据记录，创建克隆image时如果不显示指定需要启用的功能特性`--image-feature <feature_name>`，则默认会使用features所有记录的值。protection_status 用于标示快照被保护状态，处于被保护的快照不能被删除，克隆image必须基于被保护的快照进行创建，主要是为了防止克隆image所引用的父image被误删除。

- stripe_count\stripe_unit所记录的元素据是64位整数，用于记录image条带化信息。详细见理论部分。

## RBD管理元数据对象

- ### rbd_directory

用于记录当前存储池中image列表。

```
rados listomapvals -P cephfs_data rbd_directory id_ac62a15cbf99Copy
value (12 bytes) :
00000000  08 00 00 00 72 62 64 69  6d 61 67 65              |....rbdimage|
0000000c

id_aca0d915627f
value (13 bytes) :
00000000  09 00 00 00 72 62 64 69  6d 61 67 65 32           |....rbdimage2|
0000000d

name_rbdimage
value (16 bytes) :
00000000  0c 00 00 00 61 63 36 32  61 31 35 63 62 66 39 39  |....ac62a15cbf99|
00000010

name_rbdimage2
value (16 bytes) :
00000000  0c 00 00 00 61 63 61 30  64 39 31 35 36 32 37 66  |....aca0d915627f|
00000010Copy
```

### rbd_children

当image之间具有克隆关系时，rbd_children 元数据对象用于记录父image快照到克隆image之间的单向映射关系。
