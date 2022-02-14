本blog包括理论和实践两个部分，力求深入浅出，实践部分需要您事先部署成功Ceph集群！

参考《Ceph设计与实现》谢型果等，第三章。以及[官方纠删码教程](https://docs.ceph.com/en/latest/rados/operations/erasure-code/)。

# Part I

## Ceph 纠删码操作

------

## 术语

K ——数据块数。

M——编码块数。

N——条带中块的个数，N=K+MN=K+M。

块（chunk）——将对象基于纠删码进行编码时，每次编码将产生若干大小的块（要求是有序的），Ceph通过数量相等的PG将这些块分别存储至不同的OSD之中。每次编码时，序号相同的块总是由同一个PG负责存储。

条带（stripe）——如果待编码的对象太大，无法一次性完成，那么可以分成多次进行，每次完成编码的部分称为一个条带。其大小为k*块大小。

分片（shard）——同一个对象所有序号相同的块位于同一个PG之上，它们组成的对象的一个分片。

rate——空间利用率，即K/NK/N

## 1 Ceph纠删码库

Ceph的默认纠删码库是Jerasure，即Jerasure库；除此之外还有 Clay, ISA-L, LRC, Shec(Octopus版本15.2.5).
当管理员创建一个erasure-coded后端时，可以指定数据块和代码块参数。Jerasure库是第三方提供的中间件。Ceph环境安装时，已经默认安装了Jerasure库。

## 1.1 一个简单的纠删码池样例

最简单的erasure pool等效于RAID5，至少需要3个主机：【更多关于池的操作可以参考《Ceph pool》博文】

```
$ ceph osd pool create ecpool erasure
pool 'ecpool' created
$ echo ABCDEFGHI | rados --pool ecpool put NYAN -
$ rados --pool ecpool get NYAN -
ABCDEFGHICopy
```

## 1.2 获取纠删码配置文件

最简单的 k = 2, m = 2, 允许两个节点同时失效。相当于三副本，但是空间节省了。

```
$ ceph osd erasure-code-profile get default
k=2
m=2
plugin=jerasure
crush-failure-domain=host
technique=reed_sol_vanCopy
```

选择正确的配置文件很重要，因为在创建池之后无法对其进行修改：需要创建具有不同配置文件的新池，并且将先前池中的所有对象都移到新的池中。

概要文件的最重要参数是*K*，*M*和 *rush-failure域，*因为它们定义了存储开销和数据持久性。例如，如果所需的架构必须承受两个机架的损失，而存储开销(m/k*100%)为开销的67％，则可以定义以下配置文件：

```
$ ceph osd erasure-code-profile set myprofile \
   k=3 \
   m=2 \
   crush-failure-domain=rack
$ ceph osd pool create ecpool 128 erasure myprofile #128在这类是PG的数量
$ echo ABCDEFGHI | rados --pool ecpool put NYAN -
$ rados --pool ecpool get NYAN -
ABCDEFGHICopy
```

该*NYAN*对象将在三个（被划分*K = 3*）和两个附加 *的块*将被创建（*M = 2*）。*M*的值定义了在不丢失任何数据的情况下可以同时丢失多少个OSD。所述`crush-failure-domain=rack`将创建一个CRUSH规则，以确保没有两个`chunks`被存储在同一个机架。

```
ceph osd erasure-code-profile ls #显示所有profile
ceph osd erasure-code-profile rm {profilr} #删除特定profileCopy
```

读写文件test.txt

```
rados -p ecpool put test test.txt
rados -p ecpool get test file.txtCopy
```

更多信息在 [erasure code profiles](https://docs.ceph.com/en/latest/rados/operations/erasure-code-profile)

[![img](https://durantthorvalds.top/img/nyan.png)](https://durantthorvalds.top/img/nyan.png)

上图展示的是一种最简单的情况，我们称之为“满条带写”，向k=3，m=2的纠删码存储池写入NYAN对象。针对同一个逻辑PG，将对象分片并写入不同的PG实例。每个PG实例都认为字节保存的是一个完整而独立的对象。因此其保存的内容在逻辑上是连续的，以块大小为单位。5个OSD最终都向名为“NYAN”的对象写入三个字节，它们在对象内的逻辑地址都为[0,2]。

## 1.3 写覆盖

默认情况下，纠删码池仅适用于执行完整对象写入和追加的RGW之类的用途。

自从luminous版本，每个池设置启用对纠删码池的**部分写**入。这使RBD和CephFS将其数据存储在纠删码池中：

```
ceph osd pool set ec_pool allow_ec_overwrites trueCopy
```

这是针对bluestore的osd，这是因为bluestore的校验和用于检测deep-scrub 期间的bitrot和其它损坏。除了不安全之外，overwrite还将降低性能。

纠删码不支持`omap`,因此需要和RBD和CephFS一起使用，必须指示它们将数据存储在ec池中，并将元数据存储在复制池中。对于RBD，这意味着`--data-pool`在图像创建过程中使用纠删码池：

```
rbd create --size 1G --data-pool ec_pool replicated_pool/image_name
Copy
```

对于CephFS，可以在文件系统创建过程中或通过[文件布局](https://docs.ceph.com/en/latest/cephfs/file-layouts)将纠删码池设置为默认数据池。

## 1.4 缓存层

纠删码比副本需要更多资源，并且缺失`omap`这样的功能。为了克服这些限制，需要设置一个[缓存层](https://docs.ceph.com/en/latest/rados/operations/cache-tiering)

```
$ ceph osd tier add ecpool hot-storage
$ ceph osd tier cache-mode hot-storage writeback
$ ceph osd tier set-overlay ecpool hot-storageCopy
```

将放置热存储池的ecpool 在*写回* 模式。提供灵活性和速度。

## 1.5 恢复

如果纠删码池丢失了一些碎片，则必须从其他碎片中恢复它们。通常，这涉及读取其余分片，重建数据并将其写入新对等方。在Octopus中，只要至少有*K个*碎片可用，擦除编码池就可以恢复。（使用少于*K个分*片，您实际上已经丢失了数据！）

在使用Octopus之前，即使*min_size*大于*K*，擦除编码池也至少需要*min_size分*片可用。（我们通常建议min_size为*K + 2*或更大，以防止写入和数据丢失。）这种保守的决定是在设计新的池模式时出于谨慎考虑而做出的，但是这也意味着丢失OSD但没有数据丢失的池无法进行操作恢复并开始活动，而无需手动干预来更改*min_size*。

## 2 OSD erasure-code-profile 参数

通用

```
ceph osd erasure-code-profile set {name} \
     [{directory=directory}] \
     [{plugin=plugin}] \
     [{stripe_unit=stripe_unit}] \
     [{key=value} ...] \
     [--force]Copy
```

- `{directory}:string`

  设置从中加载擦除代码插件的**目录**名称. 默认`/ usr / lib / ceph / erasure-code`

- `crush-failure-domain={bucket-type}`

  确保一个桶两个数据块没有相同的容灾域. 它被用于创建 CRUSH 规则 **step chooseleaf host**. 默认host。

- `crush-device-class={device-class}`

  使用CRUSH映射中的Crush设备类名称，将布局限制为特定类（例如 `ssd`或`hdd`）的设备。

- `{plugin}:string`

  默认: `jerasure` , 可选`isa\ lrc \shec\clay`

## 2.1 jerasure

```
ceph osd erasure-code-profile set {name} \
     plugin=jerasure \
     k={data-chunks} \
     m={coding-chunks} \
     technique={reed_sol_van|reed_sol_r6_op|cauchy_orig|cauchy_good|liberation|blaum_roth|liber8tion} \
     [crush-root={root}] \
     [crush-failure-domain={bucket-type}] \
     [crush-device-class={device-class}] \
     [directory={directory}] \
     [--force]Copy
```

我们可以选择具体技术`technique`。更灵活的技术是*reed_sol_van*：足以设置*k*和*m*。该*cauchy_good*技术可以更快，但你需要选择的*PACKETSIZE* 小心。从只能使用*m = 2*进行配置的意义上来说，所有*reed_sol_r6_op*，*liberation*， *blaum_roth*，*liber8tion*都是*RAID6*等效项。

## 2.2 ISA

```
ceph osd erasure-code-profile set {name} \
     plugin=isa \
     technique={reed_sol_van|cauchy} \
     [k={data-chunks}] \
     [m={coding-chunks}] \
     [crush-root={root}] \
     [crush-failure-domain={bucket-type}] \
     [crush-device-class={device-class}] \
     [directory={directory}] \
     [--force]Copy
```

## 2.3 LRC

```
ceph osd erasure-code-profile set {name} \
     plugin=lrc \
     k={data-chunks} \
     m={coding-chunks} \
     l={locality} \
     [crush-root={root}] \
     [crush-locality={bucket-type}] \
     [crush-failure-domain={bucket-type}] \
     [crush-device-class={device-class}] \
     [directory={directory}] \
     [--force]Copy
```

*LRC*创建本地校验块，使用更少的存活OSD。例如，如果*lrc*配置为 *k = 8*，*m = 4*和*l = 4*，它将为每4个OSD创建一个额外的奇偶校验块。当1个OSD丢失时，只能使用4个OSD（而不是8个）来恢复它。

## 2.4 SHEC

```
ceph osd erasure-code-profile set {name} \
     plugin=shec \
     [k={data-chunks}] \
     [m={coding-chunks}] \
     [c={durability-estimator}] \
     [crush-root={root}] \
     [crush-failure-domain={bucket-type}] \
     [crush-device-class={device-class}] \
     [directory={directory}] \
     [--force]Copy
```

- `c={durability-estimator}:int`

  奇偶校验块的数量，每个奇偶校验块包括其计算范围内的每个数据块。该数字用作**耐久性估算器**。例如，如果c = 2，则2个OSD可以关闭而不会丢失数据。默认为2.

## 2.5 CLAY

全称是coupled-layer. 此编码目标是在修复时减少网络带宽和磁盘IO。

令d为修复时沟通的OSD数量。比如Jerasure中k=8，m=4，修复1GiB数据

需要下载8GiB数据。

在clay中允许设置d， k+1≤d≤k+m−1k+1≤d≤k+m−1。默认情况下d=k+m−1d=k+m−1，这将最大化节省网络带宽和磁盘IO。比如 k = 8, m = 4, d = 11. *则*当单个OSD发生故障时，将沟通d = 11 osds并从每个插件中下载250MiB，导致总下载量为11 X 250MiB = 2.75GiB。下面提供了更多常规参数。当对存储量达到TB级的信息的机架进行维修时，好处是巨大的。

| Plugin  | 磁盘IO总量                                 |
| :------ | :----------------------------------------- |
| Jeraure | kSkS                                       |
| Clay    | dS/(d−k+1)=(k+m−1)S/mdS/(d−k+1)=(k+m−1)S/m |

其中*S*是正在修复的单个OSD上存储的数据量。在上表中，我们使用了*d*的最大可能值，因为这将导致从OSD故障恢复所需的最小数据下载量。

```
ceph osd erasure-code-profile set {name} \
     plugin=clay \
     k={data-chunks} \
     m={coding-chunks} \
     [d={helper-chunks}] \
     [scalar_mds={plugin-name}] \
     [technique={technique-name}] \
     [crush-failure-domain={bucket-type}] \
     [directory={directory}] \
     [--force]Copy
```

- `d={helper-chunks}`

  恢复单个块期间请求发送数据的OSD数量。需要选择*d*，以使k + 1 <= d <= k + m-1。在较大的*d*，节省越多。默认 k + m -1.

- `scalar_mds={jerasure|isa|shec}`

  **scalar_mds**指定在分层构造中用作构建块的插件。可以是*jerasure*，*isa*，*shec之一*

- `technique={technique}`

  **technique**指定将在指定的“ scalar_mds”插件中采用的技术。支持的技术是’reed_sol_van’，’reed_sol_r6_op’，’cauchy_orig’，’cauchy_good’，’liber8tion’用于jerasure，’reed_sol_van’，’cauchy’用于isa和’single’，’multiple’用于shec。

  默认reed_sol_van (for jerasure, isa), single (for shec)

## MORE

Clay代码是矢量代码，因此能够节省磁盘IO和网络带宽，并且能够以称为子块的更精细的粒度查看和操作块中的数据。Clay代码的块中子块的数量由下式给出：

> 子块计数= q(k+m)/qq(k+m)/q， q=d−k+1q=d−k+1

在OSD修复期间，从可用OSD请求的帮助者信息只是块的一小部分。实际上，修复期间访问的块内子块的数量由下式给出：

> 修复子块计数= sub—−chunkcountqsub—−chunkcountq

### 例子

1. 对于*k = 4*，*m = 2*，*d = 5的配置*，子块计数为8，修复子块计数为4。因此，在修复期间仅读取一半的块。
2. 当*k = 8*，*m = 4*，*d = 11时*，子块计数为64，修复子块计数为16。从可用OSD中读取四分之一的块以修复故障块。

## 如何在给定工作量的情况下选择配置

块中所有子块中只有几个子块被读取。这些子块不必连续存储在块中。为了获得最佳的磁盘IO性能，读取连续的数据很有帮助。因此，建议您选择条带大小，以使子块大小足够大。

对于给定的条带大小（这是基于固定的工作负载），选择`k`，`m`，`d`使得：

> 子块大小= stripe−sizeksub−chunkcountstripe−sizeksub−chunkcount = 4KB，8KB，12KB…

1. 对于条带大小较大的大型工作负载，很容易选择k，m，d。例如，考虑大小为64MB的条带大小，选择*k = 16*，*m = 4*和*d = 19*将导致子块计数为1024，子块大小为4KB。
2. 对于较小的工作负载，*k = 4*，*m = 2*是一个很好的配置，可同时带来网络和磁盘IO的好处。

## 与LRC的比较

还设计了本地可恢复代码（LRC），以便在网络带宽方面节省单个OSD恢复期间的磁盘IO。但是，LRC的重点是使修复（d）期间接触的OSD数量保持最少，但这是以存储开销为代价的。clay代码有一个存储开销 m/k。在*lrc*的情况下，除奇偶校验外，它还存储（k + m）/ d个奇偶`m`校验，从而导致存储开销（m +（k + m）/ d）/ k。两个*粘土*和*LRC* 可以从任何的故障中恢复`m`的OSD。

> | 参量              | 磁盘IO，存储开销（LRC） | 磁盘IO，存储开销（CLAY） |
> | :---------------- | :---------------------- | :----------------------- |
> | （k = 10，m = 4） | 7 * S，0.6（d = 7）     | 3.25 * S，0.4（d = 13）  |
> | （k = 16，m = 4） | 4 * S，0.5625（d = 4）  | 4.75 * S，0.25（d = 19） |

`S`是恢复单个OSD的存储数据量。

## 覆盖写思考

因为数据更新必须以条带为单位进行，如果覆盖写的起始或者结束位置没有进行条带对齐，那么不足一个完整条带的部分，其写入只能通过“读取完整条带→修改数据→基于条带重新计算校验数据→写入（被修改部分和校验和）”。这个过程被称为RMW。

整个RMW过程补齐读阶段最耗时，由两种解决思路：1. 减少RMW次数，2.如果RMW不可避免，那么尽量减少补齐读的数据量。一种常见的做法是引入写缓存。将驻留于缓存的写操作进行合并。 另外是尽可能减少读的次数，基于被修改写的数据范围预先计算出需要执行补齐读的块，而不是每次都执行满条带写。

## Scrub的问题

Scrub指数据扫描，通过读取对象数据并重新计算校验和，再与之前存储在对象属性的校验和进行比对，以判断有无静默错误（磁盘自身无法感知的错误）。目前Ceph纠删码没有自动修复功能。其中Scrub只扫描元数据，而Deep Scrub对对象整体进行扫描。

## 总结与展望

例如对象大小为4MB，那么每4KB原始数据采用CRC32生成固定四个字节的校验和，则整个对象的校验和最大只能是4KB，这显然无法直接使用对象扩展属性存储，而只能使用对象的omap存储(kv pairs)，但是纠删码目前不支持omap!

Ceph中纠删码一直未达到商业水平，无外乎以下几个原因：

- 相较于多副本，纠删码实现更复杂
- 相较于多副本，纠删码性能较差，尤其是读性能。其最适合的场景一般是追加写或者删除。

这也是笔者的研究方向，路漫漫其修远兮，吾将上下而求索。
