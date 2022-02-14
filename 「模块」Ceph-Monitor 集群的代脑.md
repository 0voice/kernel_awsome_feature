# Ceph Monitor

> 参考：http://docs.ceph.org.cn/rados/configuration/mon-config-ref/#data

 Monitor是基于Paxos构建的具有分布式强一致性的小型集群，主要负责维护和传播集群表的副本。Paxos要求集群中超过半数的Monitor处于活跃状态才能正常工作，同时有一个主Monitor作为leader，其它的Monitor称为Peon。Leader通过投票产生，且Monitor担任Leader的时间称为租期。

关于集群表OSDMap

 集群表由两部分组成：一是集群拓扑结构和用于计算寻址的CRUSH规则；二是所有OSD的身份和状态信息。

 监视器们维护着集群运行图的“主副本”，就是说客户端连到一个监视器并获取当前运行图就能确定所有监视器、 OSD 和元数据服务器的位置。 Ceph 客户端读写 OSD 或元数据服务器前，必须先连到一个监视器，靠当前集群运行图的副本和 CRUSH 算法，客户端能计算出任何对象的位置，故此客户端有能力直接连到 OSD ，这对 Ceph 的高伸缩性、高性能来说非常重要。更多信息见[伸缩性和高可用性](http://docs.ceph.org.cn/architecture#scalability-and-high-availability)。

监视器的主要角色是**维护集群运行图的主副本，它也提供认证和日志记录服务**。 Ceph 监视器们把监视器服务的所有更改写入一个单独的 Paxos 例程，然后 Paxos 以键/值方式存储所有变更以实现高度一致性。同步期间， Ceph 监视器能查询集群运行图的近期版本，它们通过操作键/值存储快照和迭代器（用 leveldb ）来进行存储级同步。

[![img](https://durantthorvalds.top/img/ceph-mon.jpg)](https://durantthorvalds.top/img/ceph-mon.jpg)

### 集群运行图

集群运行图是多个图的组合，包括监视器图、 OSD 图、归置组图和元数据服务器图。集群运行图追踪几个重要事件：哪些进程在集群里（ `in` ）；哪些进程在集群里（ `in` ）是 `up` 且在运行、或 `down` ；归置组状态是 `active` 或 `inactive` 、 `clean` 或其他状态；和其他反映当前集群状态的信息，像总存储容量、和使用量。

当集群状态有明显变更时，如一 OSD 挂了、一归置组降级了等等，集群运行图会被更新以反映集群当前状态。另外，监视器也维护着集群的主要状态历史。监视器图、 OSD 图、归置组图和元数据服务器图各自维护着它们的运行图版本。我们把各图的版本称为一个 epoch 。

运营集群时，跟踪这些状态是系统管理任务的重要部分。详情见[监控集群](http://docs.ceph.org.cn/rados/operations/monitoring)和[监控 OSD 和归置组](http://docs.ceph.org.cn/rados/operations/monitoring-osd-pg)。

### 监视器法定人数

本文入门部分提供了一个简陋的 [Ceph 配置文件](http://docs.ceph.org.cn/start/quick-start/#add-a-configuration-file)，它提供了一个监视器用于测试。只用一个监视器集群可以良好地运行，然而**单监视器是一个单故障点**，生产集群要实现高可用性的话得配置多个监视器，这样单个监视器的失效才**不会**影响整个集群。

集群用多个监视器实现高可用性时，多个监视器用 [Paxos](http://en.wikipedia.org/wiki/Paxos_(computer_science)) 算法对主集群运行图达成一致，这里的一致要求大多数监视器都在运行且够成法定人数（如 1 个、 3 之 2 在运行、 5 之 3 、 6 之 4 等等）。

### 一致性

你把监视器加进 Ceph 配置文件时，得注意一些架构问题， Ceph 发现集群内的其他监视器时对其有着**严格的一致性要求**。尽管如此， Ceph 客户端和其他 Ceph 守护进程用配置文件发现监视器，监视器却用监视器图（ monmap ）相互发现而非配置文件。

一个监视器发现集群内的其他监视器时总是参考 monmap 的本地副本，用 monmap 而非 Ceph 配置文件避免了可能损坏集群的错误（如 `ceph.conf` 中指定地址或端口的拼写错误）。正因为监视器把 monmap 用于发现、并共享于客户端和其他 Ceph 守护进程间， **monmap可严格地保证监视器的一致性是可靠的**。

严格的一致性也适用于 monmap 的更新，因为关于监视器的任何更新、关于 monmap 的变更都是通过称为 [Paxos](http://en.wikipedia.org/wiki/Paxos_(computer_science)) 的分布式一致性算法传递的。监视器们必须就 monmap 的每次更新达成一致，以确保法定人数里的每个监视器 monmap 版本相同，如增加、删除一个监视器。 monmap 的更新是增量的，所以监视器们都有最新的一致版本，以及一系列之前版本。历史版本的存在允许一个落后的监视器跟上集群当前状态。

如果监视器通过配置文件而非 monmap 相互发现，这会引进其他风险，因为 Ceph 配置文件不是自动更新并分发的，监视器有可能不小心用了较老的配置文件，以致于不认识某监视器、放弃法定人数、或者产生一种 [Paxos](http://en.wikipedia.org/wiki/Paxos_(computer_science)) 不能确定当前系统状态的情形。

### 初始化监视器

在大多数配置和部署案例中，部署 Ceph 的工具可以帮你生成一个监视器图来初始化监视器（如 `ceph-deploy` 等），一个监视器需要 4 个选项：

- **文件系统标识符：** `fsid` 是对象存储的唯一标识符。因为你可以在一套硬件上运行多个集群，所以在初始化监视器时必须指定对象存储的唯一标识符。部署工具通常可替你完成（如 `ceph-deploy` 会调用类似 `uuidgen` 的程序），但是你也可以手动指定 `fsid` 。
- **监视器标识符：** 监视器标识符是分配给集群内各监视器的唯一 ID ，它是一个字母数字组合，为方便起见，标识符通常以字母顺序结尾（如 `a` 、 `b` 等等），可以设置于 Ceph 配置文件（如 `[mon.a]` 、 `[mon.b]` 等等）、部署工具、或 `ceph` 命令行工具。
- **密钥：** 监视器必须有密钥。像 `ceph-deploy` 这样的部署工具通常会自动生成，也可以手动完成。见[监视器密钥环](http://docs.ceph.org.cn/rados/operations/authentication#monitor-keyrings)。

关于初始化的具体信息见[初始化监视器](http://docs.ceph.org.cn/dev/mon-bootstrap)。

## 监视器的配置

要把配置应用到整个集群，把它们放到 `[global]` 下；要用于所有监视器，置于 `[mon]` 下；要用于某监视器，指定监视器例程，如 `[mon.a]` ）。按惯例，监视器例程用字母命名。

```
[global]

[mon]

[mon.a]

[mon.b]

[mon.c]Copy
```

### 最小配置

Ceph 监视器的最简配置必须包括一主机名及其监视器地址，这些配置可置于 `[mon]` 下或某个监视器下。

```
[mon]
        mon host = hostname1,hostname2,hostname3
        mon addr = 10.0.0.10:6789,10.0.0.11:6789,10.0.0.12:6789
[mon.a]
        host = hostname1
        mon addr = 10.0.0.10:6789Copy
```

详情见[网络配置参考](http://docs.ceph.org.cn/rados/configuration/network-config-ref)。

一旦部署了 Ceph 集群，监视器 IP 地址**不应该**更改。然而，如果你决意要改，必须严格按照[更改监视器 IP 地址](http://docs.ceph.org.cn/rados/operations/add-or-rm-mons#changing-a-monitor-s-ip-address)来改。

### 集群 ID

每个 Ceph 存储集群都有一个唯一标识符（ `fsid` ）。如果指定了，它应该出现在配置文件的 `[global]` 段下。部署工具通常会生成 `fsid` 并存于监视器图，所以不一定会写入配置文件， `fsid` 使得在一套硬件上运行多个集群成为可能。

```
fsidCopy
```

| 描述:     | 集群 ID ，一集群一个。         |
| :-------- | ------------------------------ |
| 类型:     | UUID                           |
| 是否必需: | Yes.                           |
| 默认值:   | 无。若未指定，部署工具会生成。 |

### 初始成员

我们建议在生产环境下最少部署 3 个监视器，以确保高可用性。运行多个监视器时，你可以指定为形成法定人数成员所需的初始监视器，这能减小集群上线时间。

```
[mon]
        mon initial members = a,b,c
mon initial membersCopy
```

| 描述:   | 集群启动时初始监视器的 ID ，若指定， Ceph 需要奇数个监视器来确定最初法定人数（如 3 ）。 |
| :------ | ------------------------------------------------------------ |
| 类型:   | String                                                       |
| 默认值: | None                                                         |

集群内的*大多数*监视器必须能互通以建立法定人数，你可以用此选项减小初始监视器数量来形成。

------

# 实践部分

## 1 .集群管理

1. 创建OSD。如果未提供UUID，则它将在OSD启动时自动设置。以下命令将输出OSD号，您将在后续步骤中使用该OSD号。

   ```
   ceph osd create [{uuid} [{id}]]Copy
   ```

   如果给出了可选参数{id}，它将用作OSD ID。请注意，在这种情况下，如果该号码已被使用，则该命令可能会失败。

   警告

   通常，不建议显式指定{id}。ID作为数组分配，跳过条目会占用一些额外的内存。如果存在很大的差距和/或集群很大，这可能变得很重要。如果未指定{id}，则使用最小的可用值。

2. 在新的OSD上创建默认目录。

   ```
   ssh {new-osd-host}
   sudo mkdir /var/lib/ceph/osd/ceph-{osd-number}Copy
   ```

3. 如果OSD用于OS驱动器以外的驱动器，请准备将其与Ceph一起使用，并将其安装到刚创建的目录中：

   ```
   ssh {new-osd-host}
   sudo mkfs -t {fstype} /dev/{drive}
   sudo mount -o user_xattr /dev/{hdd} /var/lib/ceph/osd/ceph-{osd-number}Copy
   ```

4. 初始化OSD数据目录。

   ```
   ssh {new-osd-host}
   ceph-osd -i {osd-num} --mkfs --mkkeyCopy
   ```

   在运行目录之前，该目录必须为空`ceph-osd`。

5. 注册OSD身份验证密钥。路径中`ceph`for 的值`ceph-{osd-num}`是`$cluster-$id`。如果您的集群名称不同于`ceph`，请改用您的集群名称。：

   ```
   ceph auth add osd.{osd-num} osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-{osd-num}/keyringCopy
   ```

6. 将OSD添加到CRUSH映射中，以便OSD可以开始接收数据。该 命令允许您将OSD添加到CRUSH层次结构中的任何位置。如果您至少指定一个存储桶，该命令会将OSD放入您指定的最特定的存储桶中，*并将*该存储桶移动到您指定的任何其他存储桶下方。**重要说明：**如果仅指定根存储桶，该命令会将OSD直接附加到根，但是CRUSH规则要求OSD位于主机内部。`ceph osd crush add`

   执行以下命令：

   ```
   ceph osd crush add {id-or-name} {weight}  [{bucket-type}={bucket-name} ...]Copy
   ```

   您还可以反编译CRUSH映射，将OSD添加到设备列表中，将主机添加为存储桶（如果尚未在CRUSH映射中添加），将该设备添加为主机中的一项，为其分配权重，然后重新编译并设置它。有关详细信息，请参见 [添加/移动OSD](https://docs.ceph.com/en/latest/rados/operations/crush-map#addosd)。

### 更换OSD

当磁盘发生故障时，或者如果管理员想用新的后端重新配置OSD（例如，从FileStore切换到BlueStore），则需要更换OSD。与[删除OSD](https://docs.ceph.com/en/latest/rados/operations/add-or-rm-osds/#removing-the-osd)不同，在销毁OSD进行替换之后，需要保持替换后的OSD的ID和CRUSH映射条目不变。

1. 确保销毁OSD是安全的：

   ```
   while ! ceph osd safe-to-destroy osd.{id} ; do sleep 10 ; doneCopy
   ```

2. 首先销毁OSD：

   ```
   ceph osd destroy {id} --yes-i-really-mean-itCopy
   ```

3. 如果以前将该磁盘用于其他目的，请为该新OSD更换磁盘。不需要新磁盘：

   ```
   ceph-volume lvm zap /dev/sdXCopy
   ```

4. 使用以前销毁的OSD ID准备要更换的磁盘：

   ```
   ceph-volume lvm prepare --osd-id {id} --data /dev/sdXCopy
   ```

5. 并激活OSD：

   ```
   ceph-volume lvm activate {id} {fsid}Copy
   ```

或者，除了准备和激活外，还可以在一个呼叫中重新创建设备，例如：

```
ceph-volume lvm create --osd-id {id} --data /dev/sdXCopy
```

### 启动OSD

在将OSD添加到Ceph之后，该OSD已在您的配置中。但是，它尚未运行。OSD是`down`和`in`。您必须先启动新的OSD，然后才能开始接收数据。您可以从管理主机使用，也可以 从其主机启动OSD：`service ceph`

```
sudo systemctl start ceph-osd@{osd-num}Copy
```

一旦启动OSD，它就是`up`和`in`。

### 观察数据迁移

将新的OSD添加到CRUSH映射后，Ceph将通过将展示位置组迁移到新的OSD开始重新平衡服务器。您可以使用[ceph](https://docs.ceph.com/en/latest/rados/operations/monitoring)工具观察此过程。

```
ceph -wCopy
```

您应该看到放置组状态从`active+clean`变为 ，最后看到迁移完成。（Ctrl-c退出。）`active, some degraded objects``active+clean`
