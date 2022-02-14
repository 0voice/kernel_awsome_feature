# 第一部分

## **1 Ceph简介**

> Ceph是一个统一的分布式存储系统，设计初衷是提供较好的性能、可靠性和可扩展性。它是一个统一的存储系统，既支持传统的块、文件存储协议，例如SAN和NAS，也支持新兴的对象存储协议，如S3和Swift，这使得Ceph理论上可以满足时下一切主流的存储应用的要求。

Ceph项目最早起源于Sage就读博士期间的工作（最早的成果于2004年发表），并随后贡献给开源社区。在经过了数年的发展之后，目前已得到众多云计算厂商的支持并被广泛应用。RedHat及OpenStack都可与Ceph整合以支持虚拟机镜像的后端存储。

- Ceph摒弃了传统的集中式存储元数据的方案，采用CRUSH算法，数据分布均衡，并行度高。
- 考虑了容灾区的隔离，能够实现各类负载的副本放置规则，例如跨机房，机架感知。
- 能够支持上千个存储节点的规模，支持TB到PB级的数据。

**高可用性**

- a. 副本数可以灵活控制。
- b. 支持故障域分隔，数据强一致性。
- c. 多种故障场景自动进行修复自愈。
- d. 没有单点故障，自动管理。

**高可扩展性**

- a. 去中心化。
- b. 扩展灵活。
- c. 随着节点增加而线性增长。

**特性丰富**

a. 支持三种存储接口：块存储、文件存储、对象存储。

b. 支持自定义接口，支持多种语言驱动。

特点：

- 高性能
- 高可用性
- 高可扩展性
- 特性丰富

**支持三种接口**：

- Object：有原生的API，而且也兼容Swift和S3的API。
- Block：支持精简配置、快照、克隆。
- File：Posix接口，支持快照。

[![640?wx_fmt=png](https://durantthorvalds.top/img/ceph-st1.jpg)](https://durantthorvalds.top/img/ceph-st1.jpg)

- Monitor

  `ceph-mon`，一个Ceph集群需要多个Monitor组成的小集群，它们通过Paxos同步数据，用来保存OSD的元数据。[*Ceph Monitor*](http://docs.ceph.org.cn/glossary/#term-ceph-monitor)维护着展示集群状态的各种图表，包括监视器图、 OSD 图、归置组（ PG ）图、和 CRUSH 图。 Ceph 保存着发生在Monitors 、 OSD 和 PG上的每一次状态变更的历史信息（称为 epoch ）。通常至少需要三个监视器才能实现冗余和高可用性。

- Manager

  [Ceph Manager](https://docs.ceph.com/en/latest/glossary/#term-Ceph-Manager)守护进程（`ceph-mgr`）负责跟踪运行时指标和Ceph集群的当前状态，包括存储利用率，当前性能指标和系统负载。Ceph Manager守护进程还托管基于python的模块，以管理和公开Ceph集群信息，包括基于Web的[Ceph仪表板](https://docs.ceph.com/en/latest/mgr/dashboard/#mgr-dashboard)和 [REST API](https://docs.ceph.com/en/latest/mgr/restful)。通常，至少需要两个管理器才能实现高可用性。

- OSD

  OSD全称Object Storage Daemon（`ceph-osd`），也就是负责响应客户端请求返回具体数据的进程。一个Ceph集群一般都有很多个OSD。[*Ceph OSD 守护进程*](http://docs.ceph.org.cn/glossary/#term-56)（ Ceph OSD ）的功能是存储数据，处理数据的复制、恢复、回填、再均衡，并通过检查其他OSD 守护进程的心跳来向 Ceph Monitors 提供一些监控信息。当 Ceph 存储集群设定为有2个副本时，至少需要2个 OSD 守护进程，集群才能达到 `active+clean` 状态（ Ceph 默认有3个副本，但你可以调整副本数）

- MDS

  MDS全称Ceph Metadata Server（`ceph-mds`），是CephFS服务依赖的元数据服务。[*Ceph 元数据服务器*](http://docs.ceph.org.cn/glossary/#term-63)（ MDS ）为 [*Ceph 文件系统*](http://docs.ceph.org.cn/glossary/#term-45)存储元数据（也就是说，Ceph 块设备和 Ceph 对象存储不使用MDS ）。元数据服务器使得 POSIX 文件系统的用户们，可以在不对 Ceph 存储集群造成负担的前提下，执行诸如 `ls`、`find` 等基本命令。

- Object

  Ceph最底层的存储单元是Object对象，每个Object包含元数据和原始数据。

- PG

  PG全称Placement Groups归置组，是一个逻辑的概念，一个PG包含多个OSD。引入PG这一层其实是为了更好的分配数据和定位数据。

- RADOS

  RADOS全称Reliable Autonomic Distributed Object Store （可靠自治的分布式对象存储），是Ceph集群的**精华**，用户实现数据分配、Failover等集群操作。具有自愈，自管理能力的智能存储节点构建的高可靠，自治，分布式对象存储系统。

- Libradio

  Librados是Rados提供库，因为RADOS是协议很难直接访问，因此上层的RBD、RGW和CephFS都是通过librados访问的，目前提供PHP、Ruby、Java、Python、C和C++支持。

- CRUSH[[1\]](https://durantthorvalds.top/2020/11/22/CEPH基础理论/#fn:1)

  CRUSH是Ceph使用的数据分布算法，类似一致性哈希，让数据分配到预期的地方。

- RBD

  RBD全称RADOS block device，是Ceph对外提供的块设备服务。采用全分布式，可靠的块设备访问接口，同时提供Linux内核态和用户态客户端访问支持，以及QEMU/KVM驱动。

- RGW

  RGW全称RADOS gateway，基于Bucket的REST网关，是Ceph对外提供的对象存储服务，接口与S3和Swift兼容。

- CephFS

  CephFS全称Ceph File System，是Ceph对外提供的文件系统服务。与POSIX兼容，同时提供Linux内核态用户端和FUSE访问支持。

> 基于 RADOS 的 Ceph 对象存储集群包括两类守护进程：对象存储守护进程（ OSD ）把存储节点上的数据存储为对象； Ceph 监视器（ MON ）维护集群运行图的主拷贝。一个 Ceph 集群可以包含数千个存储节点，最简系统至少需要一个监视器和两个 OSD 才能做到数据复制。

### 1.1 Ceph架构

[![image-20201231160559413](https://durantthorvalds.top/2020/11/22/CEPH%E5%9F%BA%E7%A1%80%E7%90%86%E8%AE%BA/CEPH%E5%9F%BA%E7%A1%80%E7%90%86%E8%AE%BA/image-20201231160559413.png)](https://durantthorvalds.top/2020/11/22/CEPH基础理论/CEPH基础理论/image-20201231160559413.png)

系统架构。 客户端通过直接与OSD通信来执行文件I / O。 每个进程可以直接链接到客户端实例，也可以与已安装的文件系统进行交互。

### 1.2 ceph读写流程

Read：

- Client app 发送读请求，RADOS将请求发送给Primary OSD。
- 主要OSD在本地磁盘读数据并完成读请求。

Write:

- Client App 写数据，RADOS将数据发送给Primary OSD。
- Primary OSD识别Replica OSDs并且向他们发送数据，由他们写数据到本地磁盘。
- Replica OSDs 完成写并通知Primary OSD。
- Primary OSDs 通知client APP 写完成。
- [![image-20201122180847737](https://durantthorvalds.top/img/ceph-b.png)](https://durantthorvalds.top/img/ceph-b.png)

### 1.3 三种存储方式

#### 1. 块设备

**典型设备：** 磁盘阵列，硬盘

主要是将裸磁盘空间映射给主机使用的。

**优点：**

- 通过RAID与LVM（逻辑卷管理）等手段，对数据提供了保护。
- 多块廉价的硬盘组合起来，提高容量。
- 多块磁盘组合出来的逻辑盘，提升读写效率。

**缺点：**

- 采用SAN架构组网时，光纤交换机，造价成本高。
- 主机之间无法共享数据。

**使用场景：**

- docker容器、虚拟机磁盘存储分配。
- 日志存储。
- 文件存储。
- …

#### 2.文件存储

**典型设备：** FTP、NFS服务器
为了克服块存储文件无法共享的问题，所以有了文件存储。
在服务器上架设FTP与NFS服务，就是文件存储。

**优点：**

- 造价低，随便一台机器就可以了。
- 方便文件共享。

**缺点：**

- 读写速率低。
- 传输速率慢。

**使用场景：**

- 日志存储。
- 有目录结构的文件存储。
- …

#### 3.对象存储

[![img](https://durantthorvalds.top/img/ceph-c.jpg)](https://durantthorvalds.top/img/ceph-c.jpg)

**典型设备：** 内置大容量硬盘的分布式服务器(swift, s3)
多台服务器内置大容量硬盘，安装上对象存储管理软件，对外提供读写访问功能。

**优点：**

- 具备块存储的读写高速。
- 具备文件存储的共享等特性。

**使用场景：** (适合更新变动较少的数据)

- 图片存储。
- 视频存储。
- …

### 

------

## 2 Ceph I/O流程和数据分布

[![img](https://durantthorvalds.top/img/ceph-d.png)](https://durantthorvalds.top/img/ceph-d.png)

[![img](https://durantthorvalds.top/img/ceph_io2.png)](https://durantthorvalds.top/img/ceph_io2.png)

**步骤：**

1. client 创建cluster handler。
2. client 读取配置文件。
3. client 连接上monitor，获取集群map信息。
4. client 读写io 根据crushmap 算法请求对应的主osd数据节点。
5. 主osd数据节点同时写入另外两个副本节点数据。
6. 等待主节点以及另外两个副本节点写完数据状态。
7. 主节点及副本节点写入状态都成功后，返回给client，io写入完成。

### 2.1 新主I/O流程图

[![img](https://durantthorvalds.top/img/ceph-e.jpg)](https://durantthorvalds.top/img/ceph-e.jpg)

**步骤：**

1. client连接monitor获取集群map信息。
2. 同时新主osd1由于没有pg数据会主动上报monitor告知让osd2临时接替为主。
3. 临时主osd2会把数据全量同步给新主osd1。
4. client IO读写直接连接临时主osd2进行读写。
5. osd2收到读写io，同时写入另外两副本节点。
6. 等待osd2以及另外两副本写入成功。
7. osd2三份数据都写入成功返回给client, 此时client io读写完毕。
8. 如果osd1数据同步完毕，临时主osd2会交出主角色。
9. osd1成为主节点，osd2变成副本。

### 2.2 Ceph I/O算法流程

[![img](https://durantthorvalds.top/img/ceph-arch.png)](https://durantthorvalds.top/img/ceph-arch.png)

1. File用户需要读写的文件。File->Object映射：

- a. ino (File的元数据，File的唯一id)。
- b. ono(File切分产生的某个object的序号，默认以4M切分一个块大小)。
- c. oid(object id: ino + ono)。

1. Object是RADOS需要的对象。Ceph指定一个静态hash函数计算oid的值，将oid映射成一个近似均匀分布的伪随机值，然后和mask按位相与，得到pgid。Object->PG映射：

- a. hash(oid) & mask-> pgid 。
- b. mask = PG总数m(m为2的整数幂)-1 。

1. PG(Placement Group),用途是对object的存储进行组织和位置映射, (类似于redis cluster里面的slot的概念) 一个PG里面会有很多object。采用CRUSH算法，将pgid代入其中，然后得到一组OSD。PG->OSD映射：

- a. CRUSH(pgid)->(osd1,osd2,osd3) 。

```
locator = object_name
obj_hash =  hash(locator)
pg = obj_hash % num_pg
osds_for_pg = crush(pg)  # returns a list of osds
primary = osds_for_pg[0]
replicas = osds_for_pg[1:]Copy
```

### 2.3 Ceph RBD IO流程

[![img](https://durantthorvalds.top/img/ceph-f.png)](https://durantthorvalds.top/img/ceph-f.png)

1. 客户端创建一个pool，需要为这个pool指定pg的数量。
2. 创建pool/image rbd设备进行挂载。
3. 用户写入的数据进行切块，每个块的大小默认为4M，并且每个块都有一个名字，名字就是object+序号。
4. 将每个object通过pg进行副本位置的分配。
5. pg根据cursh算法会寻找3个osd，把这个object分别保存在这三个osd上。
6. osd上实际是把底层的disk进行了格式化操作，一般部署工具会将它格式化为xfs文件系统。
7. object的存储就变成了存储一个文rbd0.object1.file。

[![img](https://durantthorvalds.top/img/ceph-g.png)](https://durantthorvalds.top/img/ceph-g.png)

**客户端写数据osd过程：**

1. 采用的是librbd的形式，使用librbd创建一个块设备，向这个块设备中写入数据。
2. 在客户端本地同过调用librados接口，然后经过pool，rbd，object、pg进行层层映射,在PG这一层中，可以知道数据保存在哪3个OSD上，这3个OSD分为主从的关系。
3. 客户端与primay OSD建立SOCKET 通信，将要写入的数据传给primary OSD，由primary OSD再将数据发送给其他replica OSD数据节点。

### 2.4 Ceph Pool和PG分布情况

[![img](https://durantthorvalds.top/img/ceph-h.jpg)](https://durantthorvalds.top/img/ceph-h.jpg)

- pool是ceph存储数据时的逻辑分区，它起到namespace的作用。
- 每个pool包含一定数量(可配置)的PG。
- PG里的对象被映射到不同的Object上。
- pool是分布到整个集群的。
- pool可以做故障隔离域，根据不同的用户场景不一进行隔离。

### 2.5 Ceph 数据扩容PG分布

**场景数据迁移流程：**

- 现状3个OSD, 4个PG
- 扩容到4个OSD, 4个PG

**扩容前**

[![img](https://durantthorvalds.top/img/ceph-i.jpg)](https://durantthorvalds.top/img/ceph-i.jpg)

**扩容后**

[![img](https://durantthorvalds.top/img/ceph-j.png)](https://durantthorvalds.top/img/ceph-j.png)

**说明**
每个OSD上分布很多PG, 并且每个PG会自动散落在不同的OSD上。如果扩容那么相应的PG会进行迁移到新的OSD上，保证PG数量的均衡。

[![image-20210105203533111](https://durantthorvalds.top/img/image-20210105203533111.png)](https://durantthorvalds.top/img/image-20210105203533111.png)

上图：在将写入应用于复制对象的所有OSD上的缓冲区高速缓存后，RADOS会以ack响应。 只有在将其安全地提交到磁盘之后，才将最终提交通知发送到客户端。这确保了数据的安全性。

主服务器将更新转发到副本，并在将更新应用到所有OSD的内存缓冲区高速缓存后回复确认，从而允许客户端上的同步POSIX调用返回。 当数据安全地提交到磁盘时，将发送一次最终提交（可能在几秒钟后）。 仅在完全复制更新日期之后，我们才会将确认发送给客户端，以无缝地容忍任何单个OSD的故障，即使这样做会增加客户端的延迟。 默认情况下，客户端还会缓冲写入操作，直到它们承诺避免在放置组中所有OSD同时掉电的情况下避免数据丢失为止。 在这种情况下进行恢复时，RADOS允许在接受新的更新之前，以固定的间隔重播先前已确认（因此有序）的更新。

详细请见[PG读写及迁移](https://durantthorvalds.top/2020/12/15/迁移之美PG读写流程与状态迁移详解/)

------

## 3 Ceph心跳机制

心跳是用于节点间检测对方是否故障的，以便及时发现故障节点进入相应的故障处理流程。

**问题：**

- 故障检测时间和心跳报文带来的负载之间做权衡。
- 心跳频率太高则过多的心跳报文会影响系统性能。
- 心跳频率过低则会延长发现故障节点的时间，从而影响系统的可用性。

**故障检测策略应该能够做到：**

- **及时**：节点发生异常如宕机或网络中断时，集群可以在可接受的时间范围内感知。
- **适当的压力**：包括对节点的压力，和对网络的压力。
- **容忍网络抖动**：网络偶尔延迟。
- **扩散机制**：节点存活状态改变导致的元信息变化需要通过某种机制扩展到整个集群。

### 心跳检测

[![img](https://durantthorvalds.top/img/ceph-webp.png)](https://durantthorvalds.top/img/ceph-webp.png)

**OSD节点会监听public、cluster、front和back四个端口**

- **public端口**：监听来自Monitor和Client的连接。
- **cluster端口**：监听来自OSD Peer的连接。
- **front端口**：供客户端连接集群使用的网卡, 这里临时给集群内部之间进行心跳。
- **back端口**：供客集群内部使用的网卡。集群内部之间进行心跳。
- **hbclient**：发送ping心跳的messenger。

### Ceph OSD之间相互心跳检测

[![img](https://durantthorvalds.top/img/ceph-k.jpg)](https://durantthorvalds.top/img/ceph-k.jpg)

- 同一个PG内OSD互相心跳，他们互相发送PING/PONG信息。
- 每隔6s检测一次(实际会在这个基础上加一个随机时间来避免峰值)。
- 20s没有检测到心跳回复，加入failure队列。

### Ceph OSD与Mon心跳检测

[![img](https://durantthorvalds.top/img/ceph-l.jpg)](https://durantthorvalds.top/img/ceph-l.jpg)

**OSD报告给Monitor：**

- OSD有事件发生时（比如故障、PG变更）。
- 自身启动5秒内。
- OSD周期性的上报给Monito
  - OSD检查failure_queue中的伙伴OSD失败信息。
  - 向Monitor发送失效报告，并将失败信息加入failure_pending队列，然后将其从failure_queue移除。
  - 收到来自failure_queue或者failure_pending中的OSD的心跳时，将其从两个队列中移除，并告知Monitor取消之前的失效报告。
  - 当发生与Monitor网络重连时，会将failure_pending中的错误报告加回到failure_queue中，并再次发送给Monitor。

Monitor统计下线OSD

- Monitor收集来自OSD的伙伴失效报告。
- 当错误报告指向的OSD失效超过一定阈值，且有足够多的OSD报告其失效时，将该OSD下线。

### Ceph心跳检测总结

Ceph通过伙伴OSD汇报失效节点和Monitor统计来自OSD的心跳两种方式判定OSD节点失效。

- **及时**：伙伴OSD可以在秒级发现节点失效并汇报Monitor，并在几分钟内由Monitor将失效OSD下线。

- **适当的压力**：由于有伙伴OSD汇报机制，Monitor与OSD之间的心跳统计更像是一种保险措施，因此OSD向Monitor发送心跳的间隔可以长达600秒，Monitor的检测阈值也可以长达900秒。Ceph实际上是将故障检测过程中中心节点的压力分散到所有的OSD上，以此提高中心节点Monitor的可靠性，进而提高整个集群的可扩展性。

- 容忍网络抖动

  ：Monitor收到OSD对其伙伴OSD的汇报后，并没有马上将目标OSD下线，而是周期性的等待几个条件：

  - 目标OSD的失效时间大于通过固定量osd_heartbeat_grace和历史网络条件动态确定的阈值。
  - 来自不同主机的汇报达到mon_osd_min_down_reporters。
  - 满足前两个条件前失效汇报没有被源OSD取消。

- **扩散**：作为中心节点的Monitor并没有在更新OSDMap后尝试广播通知所有的OSD和Client，而是惰性的等待OSD和Client来获取。以此来减少Monitor压力并简化交互逻辑。

## 4 Ceph通信框架

**Simple线程模式**

- **特点**：每一个网络链接，都会创建两个线程，一个用于接收，一个用于发送。
- **缺点**：大量的链接会产生大量的线程，会消耗CPU资源，影响性能。

**Async事件的I/O多路复用模式**

- **特点**：这种是目前网络通信中广泛采用的方式。k版默认已经使用Asnyc了。

**XIO方式使用了开源的网络通信库accelio来实现**

- **特点**：这种方式需要依赖第三方的库accelio稳定性，目前处于试验阶段。

### Ceph通信框架设计模式

**设计模式(Subscribe/Publish)**

订阅发布模式又名观察者模式，它意图是“定义对象间的一种一对多的依赖关系，
当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新”。

[![img](https://durantthorvalds.top/img/ceph-m.jpg)](https://durantthorvalds.top/img/ceph-m.jpg)

Accepter监听peer的请求, 调用 SimpleMessenger::add_accept_pipe() 创建新的 Pipe 到 SimpleMessenger::pipes 来处理该请求。

Pipe用于消息的读取和发送。该类主要有两个组件，Pipe::Reader，Pipe::Writer用来处理消息读取和发送。

Messenger作为消息的发布者, 各个 Dispatcher 子类作为消息的订阅者, Messenger 收到消息之后， 通过 Pipe 读取消息，然后转给 Dispatcher 处理。

Dispatcher调度员是订阅者的基类，具体的订阅后端继承该类,初始化的时候通过 Messenger::add_dispatcher_tail/head 注册到 Messenger::dispatchers. 收到消息后，通知该类处理。

DispatchQueue该类用来缓存收到的消息, 然后唤醒 DispatchQueue::dispatch_thread 线程找到后端的 Dispatch 处理消息。

[![img](https://durantthorvalds.top/img/ceph-u.png)](https://durantthorvalds.top/img/ceph-u.png)

### 通信类框架图

[![img](https://durantthorvalds.top/img/ceph-n.jpg)](https://durantthorvalds.top/img/ceph-n.jpg)

### 通信数据格式

通信协议格式需要双方约定数据格式。

**消息的内容主要分为三部分：**

- header //消息头类型消息的信封
- user data //需要发送的实际数据
  - payload //操作保存元数据
  - middle //预留字段
  - data //读写数据
- footer //消息的结束标记

```
class Message : public RefCountedObject {
protected:
  ceph_msg_header  header;      // 消息头
  ceph_msg_footer  footer;      // 消息尾
  bufferlist       payload;  // "front" unaligned blob
  bufferlist       middle;   // "middle" unaligned blob
  bufferlist       data;     // data payload (page-alignment will be preserved where possible)

  /* recv_stamp is set when the Messenger starts reading the
   * Message off the wire */
  utime_t recv_stamp;       //开始接收数据的时间戳
  /* dispatch_stamp is set when the Messenger starts calling dispatch() on
   * its endpoints */
  utime_t dispatch_stamp;   //dispatch 的时间戳
  /* throttle_stamp is the point at which we got throttle */
  utime_t throttle_stamp;   //获取throttle 的slot的时间戳
  /* time at which message was fully read */
  utime_t recv_complete_stamp;  //接收完成的时间戳

  ConnectionRef connection;     //网络连接

  uint32_t magic = 0;           //消息的魔术字

  bi::list_member_hook<> dispatch_q;    //boost::intrusive 成员字段
};

struct ceph_msg_header {
    __le64 seq;       // 当前session内 消息的唯一 序号
    __le64 tid;       // 消息的全局唯一的 id
    __le16 type;      // 消息类型
    __le16 priority;  // 优先级
    __le16 version;   // 版本号

    __le32 front_len; // payload 的长度
    __le32 middle_len;// middle 的长度
    __le32 data_len;  // data 的 长度
    __le16 data_off;  // 对象的数据偏移量


    struct ceph_entity_name src; //消息源

    /* oldest code we think can decode this.  unknown if zero. */
    __le16 compat_version;
    __le16 reserved;
    __le32 crc;       /* header crc32c */
} __attribute__ ((packed));

struct ceph_msg_footer {
    __le32 front_crc, middle_crc, data_crc; //crc校验码
    __le64  sig; //消息的64位signature
    __u8 flags; //结束标志
} __attribute__ ((packed));Copy
```

------

## 5 Ceph CRUSH算法

> Controlled Replication Under Scalable Hashing, 可扩展哈希下的可控复制。以数据唯一标识符、当前存储集群的拓扑结构以及数据备份策略作为CRUSH输入，可以随时随地的通过计算获取数控所在的底层存储设备的位置并直接与其通信，从而避免查表操作，实现去中心化和高度并发。
>
> CRUSH是一种伪随机算法，采用**一致性哈希**。
>
> OSD MAP: 包含当前所有pool的状态，和所有OSD状态。
>
> CRUSH MAP: 包含当前磁盘、服务器、机架的层次结构。
>
> CRUSH Rules：数据映射的策略。以便灵活放置Object。

### 数据分布算法挑战

**数据分布和负载均衡**：

- a. 数据分布均衡，使数据能均匀的分布到各个节点上。
- b. 负载均衡，使数据访问读写操作的负载在各个节点和磁盘的负载均衡。

**灵活应对集群伸缩**：

- a. 系统可以方便的增加或者删除节点设备，并且对节点失效进行处理。
- b. 增加或者删除节点设备后，能自动实现数据的均衡，并且尽可能少的迁移数据。

**支持大规模集群**：

- a. 要求数据分布算法维护的元数据相对较小，并且计算量不能太大。随着集群规模的增 加，数据分布算法开销相对比较小。

### Ceph CRUSH算法原理

**CRUSH算法因子：**

- 层次化的Cluster Map
  实际应用中设备具有形如“数据中心 → 机架→主机→磁盘”这样的树状层级，所以Cluster Map采用树来实现，每个叶子节点都是真实的最小物理存储设备，称为devices，而所有中间节点称为root，是整个集群的入口。每个节点都拥有唯一的数字ID和类型，但是只有叶子节点才拥有非负ID，表明它们是终端设备。

>  下表展示了Cluster Map一些常见节点的层级

- | 类型ID | 类型名称 |
  | :——: | :————: |
  | 0 | osd |
  | 1 | host |
  | 2 | chassis |
  | 3 | rack |
  | 4 | row |
  | 5 | pdu |
  | 6 | pod |
  | 7 | room |
  | 8 | datacenter |
  | 9 | region |
  | 10 | root |

[![ ](https://durantthorvalds.top/img/ceph-o.png)](https://durantthorvalds.top/img/ceph-o.png)

- CRUSH Map是一个树形结构，OSDMap更多记录的是OSDMap的属性(epoch/fsid/pool信息以及osd的ip等等)。

叶子节点是device（也就是osd），其他的节点称为bucket节点，这些bucket都是虚构的节点，可以根据物理结构进行抽象，当然树形结构只有一个最终的根节点称之为root节点，中间虚拟的bucket节点可以是数据中心抽象、机房抽象、机架抽象、主机抽象等。

### 数据分布策略Placement Rules

在完成了使用clustermap建立对应的集群的拓扑结构描述后，可以定义placement rule 来完成**数据映射**.

这些操作有三种类型：

- take

  take从cluster map选择指定编号的bucket ，并以此作为后续步骤的输入。例如系统默认以root节点作为输入。

- select*

  select从输入的bucket中随机选择指定类型和数量的条目。Ceph支持两种类型的备份策略，多副本和**纠删码**，对应两种算法，firstn和**indep**。以上两种算法都是dfs，无明显区别，唯一区别是纠删码是要求结果是有序的，i.e.总是返回指定长度的结果，如果对应条目不存在，采用空穴进行填充。

  select操作也支持容灾模式，例如设置为rack，select保证所有选出的副本位于不同的机架上，也可以设置为host，即所有选出的副本位于不同的主机的磁盘上。

- emit

  输出最终的选择结果给上级调用并返回。

**数据分布策略Placement Rules主要有特点：**

- a. 从CRUSH Map中的哪个节点开始查找
- b. 使用那个节点作为故障隔离域
- c. 定位副本的搜索模式（广度优先 or 深度优先）

```
rule replicated_ruleset  #规则集的命名，创建pool时可以指定rule集
{
    ruleset 0                #rules集的编号，顺序编即可   
    type replicated          #定义pool类型为replicated(还有erasure模式)   
    min_size 1                #pool中最小指定的副本数量不能小1
    max_size 10               #pool中最大指定的副本数量不能大于10       
    step take default         #查找bucket入口点，一般是root类型的bucket    
    step chooseleaf  firstn  0  type  host #选择一个host,并递归选择叶子节点osd     
    step emit        #结束
}Copy
```

### CRUSH算法案例

集群中有部分sas和ssd磁盘，现在有个业务线性能及可用性优先级高于其他业务线，能否让这个高优业务线的数据都存放在ssd磁盘上。

**普通用户：**

[![img](https://durantthorvalds.top/img/ceph-q.jpg)](https://durantthorvalds.top/img/ceph-q.jpg)

**高优用户**

[![img](https://durantthorvalds.top/img/ceph-r.jpg)](https://durantthorvalds.top/img/ceph-r.jpg)

配置规则

作者：[![img](https://durantthorvalds.top/img/ceph-t.jpg)](https://durantthorvalds.top/img/ceph-t.jpg)

限于篇幅，我们对CRUSH的介绍十分简略，更详细的请看[A First Galance At Crush](https://durantthorvalds.top/2020/11/27/A first glance at CRUSH/)一文.

------

## 6 定制化Ceph RBD QOS

QoS （Quality of Service，服务质量）起源于网络技术，它用来解决网络延迟和阻塞等问题，能够为指定的网络通信提供更好的服务能力。

我们总的Ceph集群的iIO能力是有限的，比如带宽，IOPS。如何避免用户争取资源，如果保证集群所有用户资源的高可用性，以及如何保证高优用户资源的可用性。所以我们需要把有限的IO能力合理分配。

### Ceph IO操作类型

- **ClientOp**：来自客户端的读写I/O请求。
- **SubOp**：osd之间的I/O请求。主要包括由客户端I/O产生的副本间数据读写请求，以及由数据同步、数据扫描、负载均衡等引起的I/O请求。
- **SnapTrim**：快照数据删除。从客户端发送快照删除命令后，删除相关元数据便直接返回，之后由后台线程删除真实的快照数据。通过控制snaptrim的速率间接控制删除速率。
- **Scrub**：用于发现对象的静默数据错误，扫描元数据的Scrub和对象整体扫描的deep Scrub。
- **Recovery**：数据恢复和迁移。集群扩/缩容、osd失效/从新加入等过程。

详细请见[Ceph QoS策略](https://durantthorvalds.top/2020/12/28/控制先行-Ceph如何实现QoS/)

参考资料

https://www.jianshu.com/p/cc3ece850433

------

1. Weil S A Brandt S A , Miller E L , et al. CRUSH: Controlled, Scalable, Decentralized Placement of Replicated Data[C]// IEEE Sc Conference. ACM, 2006. 
2. Weil S A, Brandt S A, Miller E L, et al. Ceph: A scalable, high-performance distributed file system[C]//Proceedings of the 7th symposium on Operating systems design and implementation. 2006: 307-320. 

