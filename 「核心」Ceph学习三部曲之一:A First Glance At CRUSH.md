本blog包括理论和实践两个部分，实践部分需要您事先部署成功Ceph集群！

参考《Ceph设计与实现》谢型果等，第一章。以及[官方CRUSH教程](https://docs.ceph.com/en/latest/rados/operations/crush-map/?s)。

# 浅析CRUSH算法与应用

> CRUSH论文地址：https://ceph.com/wp-content/uploads/2016/08/weil-crush-sc06.pdf
>
> 一个通过增加额外时间维度来提升性能的方案，MapX（FAST2020‘）：https://www.usenix.org/system/files/fast20-wang_li.pdf

# 1. CRUSH背景

大部分存储系统将数据写入到存储设备之后，数据很少在设备之间相互移动，这会导致一个潜在问题，即使是一个数据分布趋于完美的系统，随着时间的迁移，新的空闲设备不断加入，老设备不断退出，数据变得不均匀。
一种可行方案是将数据以足够小的粒度打散，均匀分布于整个存储系统，这样做有两个问题:

- 如果设备数量发生变化， 如何最小化迁移量使得整个系统尽快恢复平衡。
- 在大型（PB以上）分布式存储系统，为保证数据可靠性一般采用多副本或者纠删码，如何合理分布它们。。

CRUSH: Controlled Replication Under Scalable Hashing[[1\]](https://durantthorvalds.top/2020/11/25/A first glance at CRUSH/#fn:1) 基于可扩展哈希的受控副本分布策略。
它基于伪随机数哈希算法，以数据唯一标识符、当前存储集群拓扑以及数据备份策略作为输入，可以随时随地通过计算获取数据所在的底层存储设备的位置并之间与其通信，从而避免查表，失效去中心化和高度并发。

按照Sage Weil系统讲解Ceph论文的说法，CRUSH其实解决了两个问题：一个是我应该把数据存在哪？另一个是我把数据存在哪了？[5.1节]

> 实际上，无论是ceph引以为傲的自动数据恢复还是平衡功能，还是用于守护数据一致性与正确性的Scrub机制，都依赖于“可以通过某种手段不重复地遍历所有对象”这个前提，也就是时间复杂度O(N)，这是要求PG能够对每个对象进行严格排序。一种比较直观的想法是将对象标识的所有特征值按照一定规则组合为哈希串，它在集群内是唯一的。Ceph采用的是32位哈希。

# 2. Straw选择算法

网络中不同层级具有不同容忍灾难的能力，称之为容灾域。

[![image-20201128210418154](https://durantthorvalds.top/img/image-20201128210418154.png)](https://durantthorvalds.top/img/image-20201128210418154.png)

Sage weil一共设计了四种选择算法，并按照添加删除数据的性能进行比较。结论是：考虑到存储空间需求爆炸式增长，在大型分布式存储系统中某些部件故障是常态，以及数据性可靠性要求，Straw将是不错的选择。我们重点分析。

- straw算法将所有元素（设备）比作吸管，为每个元素随机计算一个长度，最后从中选择长度最长的那个元素作为结果输出，这个过程被形象地称为抽签（draw）。
- CRUSH引入了权重（weight）来区分不同容量的设备。大容量设备理应获得更大的权重。设输入为x，元素编号为i，权重为w，随机数种子为r。每根“吸管”的长度是根据权重决定的,i.e.f(wi)f(wi)。

C(r,x)=maxi(f(wi)hash(x,r,i))C(r,x)=maxi(f(wi)hash(x,r,i))

> 其实这里的x是输入PGID。[[2\]](https://durantthorvalds.top/2020/11/25/A first glance at CRUSH/#fn:2)[2.1]节

- 当添加一个元素，straw会随机将一些原有元素中的数据随机映射至新加入的元素中；当删除一个元素x，straw会将全部数据重新映射到除x以外所有元素。

当然straw也存在问题

- Straw算法将所有元素按权重逆序排列后逐个计算每个元素的Item_straw，会导致最终选择结果不断取决于每个元素自身权重还与集合当助其他元素强相关。因而会引起不相干的数据迁移。因而Sage Weil进行修正：在计算straw长度时仅使用元素自身的权重。从而得到straw改进算法straw2。

原Straw算法：

```
max_x = -1
max_item = -1
for each item:
	x = hash(input,r)
	x = x*item_straw
	if x > max_x:
		max_x = x
		max_item = item
return max_itemCopy
```

Straw2算法：

```
max_x = -1
max_item = -1
for each item:
	x = hash(input,r)
	x = ln(x/65536)/weight
	if x > max_x:
		max_x = x
		max_item = item
return max_itemCopy
```

上述逻辑中，针对输入input和随机因子r执行哈希后，结果落在[0,65536]之间，x/65536必然小于1，取其自然对数ln(x/65536)后结果为负值，将其除以自身权重后，表现为权重越大，x越大，从而体现了我们所期望的每个元素对于抽签结果的正反馈作用。

# 3 . CRUSH算法

- 针对特定输入x，CRUSH将输出一个包含n个不同存储对象的集合。我们称集群的拓扑为 Cluster Map , 不同数据分布是通过制定不同的placement rule实现的，它实际是一组包括最大副本数或纠删码策略、容灾级别的自定义约束条件。
- x和cluster map和placement rule是CRUSH的哈希函数输入参数。因为使用伪随机哈希函数，CRUSH选择每个目标存储对象概率是相对独立的。

## 3.1 Cluster Map

[![image-20201128210954633](https://durantthorvalds.top/img/cluster_map.png)](https://durantthorvalds.top/img/cluster_map.png)

实现上cluster map具有诸如 “数据中心→机架→主机→磁盘”这样的树状层次关系。每个叶子节点都是真实的物理设备（比如磁盘）称为device；所有的中间节点称为bucket；根节点称为root，是整个集群的入口。每个节点都用于唯一的数字ID和类型，但是只有叶子节点采用与非负ID。父节点的权重是所有孩子节点权重之和。

CRUSH放置策略可在故障域中分布对象副本，同时保持所需的分布。例如，为了解决并发故障的可能性，可能需要确保数据副本位于使用不同架子，机架，电源，控制器和/或物理位置的设备上。CRUSH算法通过（i）将故障域的信息（如共享电源或网络）编码到群集图中，并（ii）让管理员定义用于指定副本放置方式的放置规则，从而支持可靠约束副本放置的灵活约束。 通过递归选择存储桶项目。

常见的节点层级

- `osd` (or `device`)
- `host`
- `chassis`
- `rack`
- `row`
- `pdu` 电源分配单元
- `pod`
- `room`
- `datacenter`
- `zone`
- `region`
- `root`

[![image-20201128211039048](https://durantthorvalds.top/img/cluster_mapdemo.png)](https://durantthorvalds.top/img/cluster_mapdemo.png)

## 3.2 数据分布策略——Placement Rule

CRUSH算法的核心包括三个步骤：TAKE，SELECT和EMIT。
TAKE(a)TAKE(a)：从cluster_map选择指定编号的bucket并放入工作向量作为下一级SELECT的输入。系统默认采用root作为输入。

[![img](https://durantthorvalds.top/img/crush1.png)](https://durantthorvalds.top/img/crush1.png)

SELECT(n,t)SELECT(n,t):从bucket随机选择指定类型和数量的item。n: number，t: type。type可以设为 容灾域类型，比如rack 或host。Ceph当前支持两种备份策略——多副本和纠删码，相应的有两种选择方法first n 和 indep. 主要区别是纠删码要求结果是有序的。

- ff在这里表示失败的尝试数，初始设为0.
- rr表示副本编号，它的范围是[1,n]。
- CRUSH采用深度优先搜索方式遍历所有副本。

[![img](https://durantthorvalds.top/img/crush2.png)](https://durantthorvalds.top/img/crush2.png)

我们下面再看一下选择算法。

[![image-20201129194851670](https://durantthorvalds.top/img/crush4.png)](https://durantthorvalds.top/img/crush4.png)

- （左图）first n：比如 n = 6， select(6, disk), 当第二个item被拒绝，其余节点会填充空位。尝试次数f 将更新 副本编号r。
- （右图）每一个队列有概率上独立的顺序，这儿fr=1,r′=r+frn=8fr=1,r′=r+frn=8，对应device:h，因此会用h进行“填充”。

因为在容灾域模式下会产生递归调用，所以还需要限制产生递归调用时作为下一级输入的全局尝试次数（`choose_total_tries`），因为这个限制会导致递归调用时全局尝试次数成倍增长，按照递归的概念，多次递归后这个全局尝试次数应该成指数增长，但是实际上至多调用一次，所以这里是将原始尝试次数放大N倍后作为下一级输入的全局尝试数，实现上采用一个布尔变量（`chooseleaf_descent_once`,i.e. “first n”）进行控制，如果为真，则在产生递归调用至多重试一次，否则则不进行重试，由调用者自身进行重试。N由`chooseleaf_vary_r`进行决定。

[![image-20201129195221719](https://durantthorvalds.top/img/crush5.png)](https://durantthorvalds.top/img/crush5.png)

这儿的b.c(r′,x)b.c(r′,x)即为第二节谈到的 Bucket Choose ， 我们采用Straw2算法。
如果得到的结果不是目标类型，则继续向下递归。并设置重试标记retrybucketretrybucket为true.

[![image-20201129195537731](https://durantthorvalds.top/img/crush6.png)](https://durantthorvalds.top/img/crush6.png)

当输出已经在输出条目中或者发生冲突或过载时，如果fr≥3fr≥3会执行29行。冲突（Collision）： 选中的条目已经存在于输出条目列表之中。
OSD过载或失效：

1. 由于集群规模较小，导致集群PG总数有限，CRUSH输入不够。
2. CRUSH本身缺陷，每次选择是单个条目被选中的独立概率，但是CRUSH所要求的副本策略使得针对同一个输入、多个副本直接的选择变成了条件概率。

在老的CRUSH实现，为了避免每次回到初始输入的bucket下重试，可以在当前的bucket下直接进行重试。此时同样需要对局部尝试次数进行限制，称为(`choose_local_retries`)。

Overload(o,x)Overload(o,x)

除了由容量计算得到的真实权重之外，Ceph还设置了可以人工调整的权重（reweight）。算法正常选中一个OSD之后，最后还基于此reweight进行一次过载测试，如果测试失败，则将仍然拒绝该item。
我们可以通过设置reweight介于[0,0x10000]之间，如果为0就不能通过测试，为0x10000就是一定会通过测试。在实际应用中通过降低过载OSD或者增加空闲reweight都可以触发数据在OSD之间重新分布。并且可以区分暂时失效的OSD和永久失效的OSD。

[![reweight](https://durantthorvalds.top/img/reweight.png)](https://durantthorvalds.top/img/reweight.png)

EMITEMIT:输出最终选择结果给上级调用者并返回。

[![image-20201129195731168](https://durantthorvalds.top/img/crush8.png)](https://durantthorvalds.top/img/crush8.png)

总结

我们以firstn为例展示从指定bucket查找指定数量item的过程。

[![image-20201129195815305](https://durantthorvalds.top/img/crush9.png)](https://durantthorvalds.top/img/crush9.png)

------

# 4 操作CRUSH

1 查看osd tree （含权重）

```
sudo ceph osd treeCopy
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         0.05846  root default                             
-5         0.01949      host node3                           
 1    ssd  0.01949          osd.1       up   1.00000  1.00000
-3         0.01949      host node4                           
 0    ssd  0.01949          osd.0       up   1.00000  1.00000
-7         0.01949      host node5                           
 2    ssd  0.01949          osd.2       up   1.00000  1.00000Copy
```

`ID`: 每个节点在集群唯一ID。`class`：每个节点的类别。`weight`:每个节点的权重。

2 查看整个集群空间利用率

```
sudo ceph osd df treeCopy
ID  CLASS  WEIGHT   REWEIGHT  SIZE    RAW USE  DATA     OMAP    META      AVAIL   %USE  VAR   PGS  STATUS  TYPE NAME     
-1         0.05998         -  60 GiB  3.0 GiB  2.4 MiB  59 KiB   3.0 GiB  57 GiB  5.01  1.00    -          root default  
-5         0.00999         -  20 GiB  1.0 GiB  824 KiB  18 KiB  1024 MiB  19 GiB  5.01  1.00    -              host node3
 1    ssd  0.00999   1.00000  20 GiB  1.0 GiB  824 KiB  18 KiB  1024 MiB  19 GiB  5.01  1.00   81      up          osd.1 
-3         0.01999         -  20 GiB  1.0 GiB  824 KiB  17 KiB  1024 MiB  19 GiB  5.01  1.00    -              host node4
 0    ssd  0.01999   1.00000  20 GiB  1.0 GiB  824 KiB  17 KiB  1024 MiB  19 GiB  5.01  1.00   81      up          osd.0 
-7         0.03000         -  20 GiB  1.0 GiB  828 KiB  24 KiB  1024 MiB  19 GiB  5.01  1.00    -              host node5
 2    ssd  0.03000   1.00000  20 GiB  1.0 GiB  828 KiB  24 KiB  1024 MiB  19 GiB  5.01  1.00   81      up          osd.2 
                       TOTAL  60 GiB  3.0 GiB  2.4 MiB  61 KiB   3.0 GiB  57 GiB  5.01                                   
MIN/MAX VAR: 1.00/1.00  STDDEV: 0Copy
```

## 4.1 Rules

查看集群的rules

```
sudo ceph osd crush rule lsCopy
replicated_rule
ecpoolCopy
```

你也能打印出rules的细节

```
sudo ceph osd crush rule dumpCopy
[
    {
        "rule_id": 0,
        "rule_name": "replicated_rule",
        "ruleset": 0,
        "type": 1,
        "min_size": 1,
        "max_size": 10,
        "steps": [
            {
                "op": "take",
                "item": -1,
                "item_name": "default"
            },
            {
                "op": "chooseleaf_firstn",
                "num": 0,
                "type": "host"
            },
            {
                "op": "emit"
            }
        ]
    },
    {
        "rule_id": 1,
        "rule_name": "ecpool",
        "ruleset": 1,
        "type": 3,
        "min_size": 3,
        "max_size": 3,
        "steps": [
            {
                "op": "set_chooseleaf_tries",
                "num": 5
            },
            {
                "op": "set_choose_tries",
                "num": 100
            },
            {
                "op": "take",
                "item": -1,
                "item_name": "default"
            },
            {
                "op": "chooseleaf_indep",
                "num": 0,
                "type": "host"
            },
            {
                "op": "emit"
            }
        ]
    }
]
Copy
```

## 4.2 Device class

可以通过以下命令设置device的类别（`hdd`, `ssd`, or `nvme`）

```
sudo ceph osd crush set-device-class <class> <osd-name> [...]Copy
```

可以通过以下命令更改为其它类别

```
sudo ceph osd crush rm-device-class <osd-name> [...]Copy
```

如果我们想创建新的placement rule

```
sudo ceph osd crush rule create-replicated <rule-name> <root> <failure-domain> <class>Copy
```

对于pool而言就是

```
sudo ceph osd pool set <pool-name> crush_rule <rule-name>Copy
```

通过为使用中的每个仅包含该类设备的设备类创建一个“影子” CRUSH层次结构来实现设备类。然后，CRUSH规则可以在影子层次结构上分布数据。

```
sudo ceph osd crush tree --show-shadowCopy
ID  CLASS  WEIGHT   TYPE NAME         
-2    ssd  0.05846  root default~ssd  
-6    ssd  0.01949      host node3~ssd
 1    ssd  0.01949          osd.1     
-4    ssd  0.01949      host node4~ssd
 0    ssd  0.01949          osd.0     
-8    ssd  0.01949      host node5~ssd
 2    ssd  0.01949          osd.2     
-1         0.05846  root default      
-5         0.01949      host node3    
 1    ssd  0.01949          osd.1     
-3         0.01949      host node4    
 0    ssd  0.01949          osd.0     
-7         0.01949      host node5    
 2    ssd  0.01949          osd.2Copy
```

## 4.3 权重集

权重集使群集可以根据群集的详细信息（层次结构，池等）执行数值优化，以实现平衡分配。

支持两种类型的权重集，目前支持两种类型的weight set：

- Compat权重集，针对集群每个节点而设计的权重，具有良好的向后兼容性。
- Per-pool权重集，针对数据池的权重集。

## 4.4 修改CRUSH Map

要在正在运行的群集的CRUSH映射中添加或移动OSD，请执行以下操作：

```
sudo ceph osd crush set {name} {weight} root={root} [{bucket-type}={bucket-name} ...]Copy
```

{name}指osd名称

## 4.5 调整OSD权重[¶](https://docs.ceph.com/en/latest/rados/operations/crush-map/#adjust-osd-weight)

要在正在运行的群集的CRUSH映射中调整OSD的CRUSH权重，请执行以下操作：

```
sudo ceph osd crush reweight {name} {weight}Copy
```

要从正在运行的群集的CRUSH映射中删除OSD，请执行以下操作：

```
sudo ceph osd crush remove {name}Copy
```

要在正在运行的集群的CRUSH映射中添加存储桶，请执行以下 命令：`sudo ceph osd crush add-bucket`

```
sudo ceph osd crush add-bucket {bucket-name} {bucket-type}Copy
```

要将存储桶移动到CRUSH地图层次结构中的其他位置或位置，请执行以下操作：

```
sudo ceph osd crush move {bucket-name} {bucket-type}={bucket-name}, [...]Copy
```

要从CRUSH层次结构中删除存储桶，请执行以下操作：

```
sudo ceph osd crush remove {bucket-name}Copy
```

## 4.6 创建一个Compat权重集

要创建*兼容的权重集*：

```
sudo ceph osd crush weight-set create-compatCopy
```

兼容重量组的重量可以通过以下方式调整：

```
sudo ceph osd crush weight-set reweight-compat {name} {weight}Copy
```

可以用以下方法删除：

```
sudo ceph osd crush weight-set rm-compatCopy
```

## 4.7 创建per-pool的权重集

要为特定池创建权重集，请执行以下操作：

```
sudo ceph osd crush weight-set create {pool-name} {mode}Copy
```

调整权重：

```
sudo ceph osd crush weight-set reweight {pool-name} {item-name} {weight [...]}Copy
```

要列出现有的权重集，请执行以下操作：

```
sudo ceph osd crush weight-set lsCopy
```

要删除，请执行以下操作：

```
sudo ceph osd crush weight-set rm {pool-name}Copy
```

## 4.8 为副本创建规则

```
sudo ceph osd crush rule create-replicated {name} {root} {failure-domain-type} [{class}]Copy
```

## 4.9 为纠删码创建规则

对于纠删码（EC）池，需要做出相同的基本决策：故障域是什么，层次结构中的哪个节点将数据放置在（通常为`default`）下，并且放置位置将限制为特定的设备类。但是，纠删码池的创建方式略有不同，因为需要根据所使用的删除代码仔细构建它们。因此，您必须在*纠删码配置文件中*包含此信息。使用配置文件创建池时，将根据该规则显式或自动创建CRUSH规则。

纠删码配置文件可以列出：

```
sudo ceph osd erasure-code-profile lsCopy
```

查看某个特定配置

```
sudo ceph osd erasure-code-profile get {profile-name}Copy
```

通常，绝对不要修改配置文件。而是在创建新池或为现有池创建新规则时创建并使用新配置文件。

感兴趣的纠删码配置文件属性为：

> - **rush-root**：要在其下放置数据的CRUSH节点的名称[默认值：`default`]。
> - **rush-failure-domain**：在其上分配擦除编码分片的CRUSH存储桶类型[默认值：`host`]。
> - **rush-device-class**：放置数据的设备类[默认：无，表示使用了所有设备]。
> - **k**和**m**（对于`lrc`插件，为**l**）：它们确定擦除代码分片的数量，影响最终的CRUSH规则。

定义配置文件后，您可以使用以下方法创建CRUSH规则：

```
sudo ceph osd crush rule create-erasure {name} {profile-name}#{name}为规则名称Copy
```

可以通过以下方式删除池未使用的规则：

```
sudo ceph osd crush rule rm {rule-name}Copy
```

## 4.10 自定义CRUSH规则♥

> https://docs.ceph.com/en/latest/rados/operations/crush-map-edits/

我们可以通过CLI来很方便的修改CRUSH各项配置，但是如果修改项目过多，而集群较多，直接编辑CRUSH map 会是更好的选择。在一些特殊情况下，比如集群HDD和SSD和NVME混合则必须单独配置CRUSH rules。

### （1）获取CRUSH Map

```
sudo ceph osd getcrushmap -o {compilefilename}Copy
```

之后我们将其解码为txt

```
crushtool -d {compilefilename} -o {outputfilename}.txtCopy
```

例如

```
crushtool -d mycrushmap -o mycrushmap.txtCopy
```

典型的crush map如下:

它包含6个部分：

1. **可调项：** tunable
2. **设备：**设备是存储数据的单个OSD。
3. **types**：存储桶`types`定义在CRUSH层次结构中使用的存储桶的类型。存储桶由存储位置（例如，行，机架，机箱，主机等）及其分配的权重的分层聚合组成。
4. **存储桶：**定义存储桶类型后，必须定义层次结构中的每个节点，其类型以及它包含的设备或其他节点。
5. **规则：**规则定义有关数据如何在层次结构中的各个设备之间分配的策略。
6. choice_args **：** Choose_args是与层次结构关联的替代权重，这些权重已进行调整以优化数据放置。单个choose_args映射可以用于整个集群，也可以为每个单独的池创建一个映射。

```
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable chooseleaf_vary_r 1
tunable chooseleaf_stable 1
tunable straw_calc_version 1
tunable allowed_bucket_algs 54

# devices
device 0 osd.0 class ssd
device 1 osd.1 class ssd
device 2 osd.2 class ssd

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 zone
type 10 region
type 11 root

# buckets
host node4 {
        id -3           # do not change unnecessarily
        id -4 class ssd         # do not change unnecessarily
        # weight 0.020
        alg straw2
        hash 0  # rjenkins1
        item osd.0 weight 0.020
}
...
root default {
        id -1           # do not change unnecessarily
        id -2 class ssd         # do not change unnecessarily
        # weight 0.060
        alg straw2
        hash 0  # rjenkins1
        item node4 weight 0.020
        item node3 weight 0.010
        item node5 weight 0.030
}
#rules
rule replicated_rule {
        id 0
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}

rule ecpool {
        id 1
        type erasure
        min_size 3
        max_size 3
        step set_chooseleaf_tries 5
        step set_choose_tries 100
        step take default
        step chooseleaf indep 0 type host
        step emit
}
...
# choose_args
choose_args 4 {
  {
    bucket_id -1
    weight_set [
      [ 0.020 0.010 0.030 ]
    ]
  }
}
...
# end crush mapCopy
```

我们对ruleset中参数稍作解释

`step` 包括三个部分 take, chooseleaf, emit

- chooseleaf, 容灾域模式，可以替换为choose，即非容灾域模式。
- firstn, 两种选择算法之一，参看前面理论部分，可以替换为indep.
- 0, 表示由具体的调用者指定输出的副本数，例如不同的pool可以使用同一套ruleset（拥有相同的备份策略），但是可以拥有不同的副本数。
- type，对应chooseleaf操作，指示输出必须是分布在由本选项指定类型的、不同的bucket之下的叶子节点；对应choose操作，指示输出类型。

`set_chooseleaf_tries`: 容灾域下产生递归调用时的尝试次数。

`set_choose_tries`: 非容灾域下产生递归调用的尝试次数。

`min_size`: 如果池中的副本数少于此数量，则CRUSH将 **不会**选择此规则。

`max_size`: 如果池中的副本数量超过此数量，则CRUSH将 **不会**选择此规则。

> ```
> firstn` 与 `indep
> ```
>
> - 描述
>
> 控制在CRUSH映射中标记了项目（OSD）时CRUSH使用的替换策略。**如果此规则将用于复制池，则应使用`firstn`；如果是擦除编码池，则应使用`indep`**。原因与先前选择的设备发生故障时它们的行为有关。假设您有一个PG存储在OSD 1、2、3、4、5上。然后3下降。在“ firstn”模式下，CRUSH只需将其计算调整为选择1和2，然后选择3，但发现它已关闭，因此它重试并选择4和5，然后继续选择一个新的OSD6。因此最终的CRUSH映射更改为1、2、3、4、5-> 1、2、4、5、6。但是，如果要存储EC池，则意味着您只需更改映射到OSD 4、5和6的数据！因此，“独立”模式试图不这样做。相反，您可以期望它在选择失败的OSD 3时再次尝试并选择6，以进行以下最终转换：1，2，3，4，5-> 1，2，6，4，5

重要 :给定的CRUSH规则可以分配给多个池，但是单个池不可能具有多个CRUSH规则。

### （2）修改并编译CRUSH Map

我们需要进行编译才能被ceph识别

```
crushtool - c {decompiled_filename} -o {compiled_filename}Copy
```

### （3）测试

例如我们打印出输入范围为[0,9]、副本数3，采用编号为0的ruleset映射的结果。

```
sudo crushtool -i {compiled_filename} --test --min-x 0 --max-x 9 --num-rep 3 --ruleset 0\
 --show_mappingsCopy
CRUSH rule 0 x 0 [1,0,2]
CRUSH rule 0 x 1 [2,0,1]
CRUSH rule 0 x 2 [2,0,1]
CRUSH rule 0 x 3 [0,1,2]
CRUSH rule 0 x 4 [2,1,0]
CRUSH rule 0 x 5 [0,2,1]
CRUSH rule 0 x 6 [2,0,1]
CRUSH rule 0 x 7 [2,1,0]
CRUSH rule 0 x 8 [2,0,1]
CRUSH rule 0 x 9 [1,2,0]Copy
```

也可以仅统计结果分布情况，输入变为[0,100000]

```
sudo crushtool -i mycrushmap --test --min-x 0 --max-x 100000 --num-rep 3\
 --ruleset 0 --show_utilizationCopy
rule 0 (replicated_rule), x = 0..100000, numrep = 3..3
rule 0 (replicated_rule) num_rep 3 result size == 2:	3/100001
rule 0 (replicated_rule) num_rep 3 result size == 3:	99998/100001
  device 0:		 stored : 100001	 expected : 100001
  device 1:		 stored : 99998	 expected : 100001
  device 2:		 stored : 100001	 expected : 100001
Copy
```

需要注意的是，除了叶子节点，其它的层级均为虚拟的，例如下面这个例子，我们让所有副本都必须分布在编号为0,1,2者三个特定的osd上。

可以通过如下语句声明存储桶：

```
[bucket-type] [bucket-name] {
        id [a unique negative numeric ID]
        weight [the relative capacity/capability of the item(s)]
        alg [the bucket type: uniform | list | tree | straw | straw2 ]
        hash [the hash type: 0 by default]
        item [item-name] weight [weight]
}Copy
vim mycrushmap.txt
host virtualhost{
	id -14
	#weight 3.00
	alg straw2
	hash 0 # rejenkins1
	item osd.0 weight 1.00
	item osd.1 weight 1.00
	item osd.2 weight 1.00
}
#rules
rule customized_ruleset{
	ruleset 1
	type replicated 
	min_size 1
	max_size 10
	step take virtualhost
	step chooseleaf firstn 0 type osd
	step emit
}Copy
```

### （4）注入集群

```
sudo ceph osd setcrushmap -i {compiledfilename}Copy
```

## 4.11 数据自动平衡

在实际的生产环境中，Ceph集群的空间利用率并不高，均值为23%左右；另一方面，当磁盘空间利用率超过80%都会变得及其缓慢，系统整体性能受到木桶原理制约。我们希望集群所有OSD的空间利用率尽可能趋于一致。

数据管理的最小单元是PG，因此我们进一步思考如何让每个OSD上的PG数量尽可能趋于均衡。一种常见方法是采用`reweight`。

找到空间利用率比较高的osd然后执行

```
sudo ceph osd reweight {osd_num_id} {reweight}Copy
```

当然也可以批量调整：目前有两种模式

- 按照OSD当前的空间利用率(`reweight-by-utilization`) ;
- 按照PG在OSD之间的分布(`reweight-by-pg`)。

为防止影响前端业务，可以先进行测试，这会触发PG进行迁移量统计。

例如：

```
sudo ceph osd test-reweight-by-utilization {oload} {max_change}\
{max_osds} {--no-increasing}Copy
```

|      参数      |                             含义                             |
| :------------: | :----------------------------------------------------------: |
|     oload      | 可选；整型，≥100，默认值120；当且仅当某个OSD的空间利用率大于等于集群瓶颈空间利用率的overload/100时，调整其reweight |
|   max_change   | 可选，浮点数，[0,1]；默认受`mon_reweight_max_change`控制，目前为0.05.每次调整reweight的最大幅度，即调整上限。实际每个osd调整幅度取决于自身空间利用率与集群平均空间利用率的偏离程度——偏离越多调整越大 |
|    max_osds    | 可选，整型，默认受`mon_reweight_max_osds`控制，目前为4.每次至多调整的osd数目。 |
| —no-increasing | 可选,字符类型，如果携带，则从不将reweight进行上调（上调指将当前的underload的OSD权重调大，让其分担更多PG）；如果不携带，至多将OSD的reweight调整至1.0/ |

### weight-set

通过CRUSH选择不同位置的副本时，对应的OSD呈现的概率应有所不同，也可以针对存储池中么一个OSD按照副本数设置一个权重组，称为weight-set。每个weight-set所包含的元素数目和对应存储的副本数相同，与CRUSH weight 和reweight直接和OSD绑定不同，weight-set是和存储池绑定的。weight-set有两种模式：兼容模式和非兼容模式。

列出已有的weight-set:

```
ceph osd crush weight-set lsCopy
```

(1)兼容模式

首先创建weight-set:

```
ceph osd crush weight-set create-compatCopy
```

创建成功后，weight-set中的权重会被自动初始化为当前的CRUSH weight，可以通过下面的命令修改，以调整PG在OSD之间的分布。

```
ceph osd crush weight-set reweight-compat <item> <weight>Copy
```

这里的item形式是`osd.0`，之后可以通过下面的命令查看

```
ceph osd crush dumpCopy
```

如果不需要则可以删除

```
ceph osd crush weight-set rm-compatCopy
```

（2）逐池模式

这种模式需要与具体的存储池相绑定，它还有两种模式：`flat`和`positional`。

flat模式，weight-set退化为一维数组，效果与兼容模式完全相同；positional模式，需要根据副本的个数以及当前所处的位置为每个OSD指定一组权重，命令如下：

```
ceph osd crush weight-set create <poolname> flat|positionalCopy
```

调整权重，（假设test为存储池名称，假定其为三副本）

```
ceph osd crush weight-set create test positional
ceph osd crush weight-set reweight test osd.0 0.3 0.2 0.1Copy
```

删除

```
ceph osd crush weight-set rm <poolname>Copy
```

随着副本数量的上升，每个weight-set条目数目呈指数上升，故十分影响其应用。

### upmap

为了解决由于永久性故障、扩容等因素可能导致正常业务长时间中断而引入PG Temp机制。虽然PG到OSD的映射是通过CRUSH计算得到的，但是事后仍然可以通过PG Temp进行调整，将其进行推广我们可以得到随心所欲调整PG分布的方法。upmap是对PG的up集合进行部分或全部替换。视调整粒度不同：

1. 对CRUSH选择整体进行替换

```
ceph osd pg-upmap <pgid> <osdname (id|osd.id)> [<osdname (id|osd.id)> ...]Copy
```

1. 部分替换

```
ceph osd pg-upmap-items <pgid> <osdname (id|osd.id)> [<osdname (id|osd.id)> ...]Copy
```

upmap可以对CRUSH结果进行精确调整，但如果客户端不支持，则无法启用upmap机制。

### balancer

利用reweight、upmap、weight-set进行权重调整的共同点是都需要人工干预，当集群发生扩容或者缩减时，由于PG自动迁移的特性，又会导致权重变化。

为此我们希望Ceph能自动调节权重，Ceph的mgr组件引入了一个balancer模块，其本质上也是调用reweight、upmap、weight-set三种方法，

 我们计算空间利用率的方差，以反映空间利用率分布情况，并且只考虑那些过载的OSD，据此建立评分系统，如果评分高说明集群空间分布越不均衡，评分为0表示 *完美均衡* 状态。

 如果采用reweight/compat weight-set方式，Ceph将当前所有挂载的OSD进行排序，逐个将其权重调低到一个比较合适的值，如果评分升高，说明步长太大了，需要调小。如果采用upmap，则本身可以针对PG的每个副本进行精确调整，只需要考虑每次调整的PG数量在合理的范围之内即可。

 为了更好的了解balancer工作流程，我们先介绍一些相关的术语：

**plan**

plan是balancer内部的一个优化方案，记录集群当前的状态，包括PG分布情况、空间使用率等。

**eval**

指针对集群当前的数据分布状态的评估，实际就是我们前面所说的方差，衡量指标包括PG数量、对象数量和字节数。

**optimize**

创建一个plan。

**execute**

执行一个plan

**mode**

指创建执行plan的方法，包括`crush-compat`、`upmap`和`none`，

1. 查看balance状态

```
ceph balancer status//默认balancer关闭Copy
```

1. 手动创建plan

```
ceph balancer optimize <plan> {<pools> [pools ... ]}Copy
```

恢复之前的操作

```
ceph balancer resetCopy
```

1. 自动开启和关闭（默认60s周期数据自动平衡）

```
ceph balancer on
ceph balancer offCopy
```

1. 设置模式

```
ceph mode crush-compat// upmapCopy
```

1. 更改数据平衡时间

```
ceph config-key set mgr/balancer/begin_time 0000
ceph config-key set mgr/balancer/end_time 0600
ceph config-key set mgr/balancer/max_misplaced .01
Copy
```

上面表示自动优化在0：00-6：00进行，且每次优化最多影响1%的PG。

**官方**

要评估当前分布并为其评分：

```
ceph balancer evalCopy
```

您还可以使用以下方法评估单个池的分布：

```
ceph balancer eval <pool-name>Copy
```

评估的更多细节可以通过以下方式看到：

```
ceph balancer eval-verbose ...Copy
```

该平衡器可以生成一个计划，使用当前配置的模式，具有：

```
ceph balancer optimize <plan-name>Copy
```

该名称由用户提供，可以是任何有用的标识字符串。计划的内容可以通过以下方式查看：

```
ceph balancer show <plan-name>Copy
```

所有计划都可以显示：

```
ceph balancer lsCopy
```

旧计划可以通过以下方式丢弃：

```
ceph balancer rm <plan-name>Copy
```

当前记录的计划显示为status命令的一部分：

```
ceph balancer statusCopy
```

执行计划后将产生的分布质量可通过以下公式计算：

```
ceph balancer eval <plan-name>Copy
```

假设该计划有望改善分布（即，其得分低于当前集群状态），则用户可以使用以下命令执行该计划：

```
ceph balancer execute <plan-name>Copy
```

# 5 扩展知识

## 1 故障恢复

OSD群集映射将因OSD故障，恢复和显式群集更改（例如，部署新存储）而更改。 为了促进快速恢复，OSD主要为每个对象保留一个版本号和每个PG的最近更改日志（已更新或删除的对象的名称和版本）。

当活动的OSD收到更新的群集映射时，它将遍历所有本地存储的放置组，并计算CRUSH映射以确定它负责哪个映射（作为主副本或副本副本）。 如果PG的成员资格已更改，或者OSD刚刚启动，则OSD必须与PG的其他OSD进行peerpeer。

对于复制的PG，OSD向主数据库提供其当前PG版本号。 如果OSD是PG的主要版本，则OSD会收集当前（和以前）副本的PG版本。 如果主数据库缺少最新的PG状态，它将从PG中的当前OSD或先前的OSD检索最近PG更改的日志（或完整的内容摘要），以确定正确的（最新）PG内容。 然后主数据库向每个副本发送一次重要的日志更新（或完整的内容摘要），以便所有各方都知道PG的内容，即便本地存储的对象并不匹配。只有在主服务器确定正确的PG状态并与任何副本共享它之后，才允许对PG中的对象进行I / O。 然后，OSD将独立负责从其对等方检索丢失或过时的对象。 如果OSD收到对陈旧或丢失对象的请求，它将延迟处理并将该对象移到恢复队列的最前面。

例如，假设osd1崩溃并被标记为down，而osd2接替pgA作为主要对象。 如果osd1恢复，它将在启动时请求最新的映射，并且监视器将其标记为已启动。 当osd2收到结果映射更新时，它将意识到它不再是pgA的主要版本，并将pgA版本号发送给osd1。

osd1将从osd2检索最近的pgA日志条目，告诉osd2其内容是最新的，然后开始处理请求，同时在后台恢复任何更新的对象。
由于故障恢复完全由单独的OSD驱动，因此受故障OSD影响的每个PG将与不同的替换OSD并行恢复。

## 2 EBOFS

```
 POSIX接口不能支持原子数据和元数据（例如，属性）更新事务，这对于保持RADOS的一致性很重要。
Copy
```

 每个Ceph OSD都使用**EBOFS**（基于范围和B树的对象文件系统）来管理其本地对象存储。 完全在用户空间中实现EBOFS并直接与原始块设备进行交互，使我们能够定义自己的低级对象存储接口和更新语义，从而将更新序列化（用于同步）与磁盘提交（出于安全性）分开。

EBOFS支持原子事务（例如，在多个对象上进行写和属性更新），并且当提供内存中的提交的异步通知时，更新功能在存储器中的高速缓存被更新时返回。

- 避免了与Linux VFS和页面缓存的繁琐交互，这两者都是针对不同的界面和工作负载而设计的。
- 更容易地最终确定工作负载的优先级（例如，客户端I / O与恢复）或提供服务质量保证。
- EBOFS可以在磁盘上的写入位置或相关数据附近快速定位可用空间，同时还可以限制长期碎片。
- EBOFS积极地执行写时复制：除超级块(superblock)更新外，数据始终写入磁盘的未分配区域。

2020/12/22更新

## MapX[

[“>[2\] (FAST20’) 中对CRUSH算法的改进](https://durantthorvalds.top/2020/11/25/A first glance at CRUSH/#fn:2)

[传统的CRUSH架构如下图（图1）](https://durantthorvalds.top/2020/11/25/A first glance at CRUSH/#fn:2)

[![image-20201224164010122](https://durantthorvalds.top/img/mapx1.png)](https://durantthorvalds.top/img/mapx1.png)

传统的CRUSH算法在增加节点时会导致大量的数据迁移，导致系统性能下降。下图（图2）模拟了两个模拟CRUSH集群在扩展时发生的数据迁移。

[![image-20201224162115959](https://durantthorvalds.top/img/mapx2.png)](https://durantthorvalds.top/img/mapx2.png)

定性的表示，数据迁移量可以高达hΔw/W，其中h是层次结构中的级别数，而Δw和W是扩展权重以及 所有OSD的总重量.

存储系统通常在集群扩展时会避免数据迁移，这会导致数据暂时不平衡。比如，Haystack和HDFS采用中心目录来避免已有的对象受到影响。因此本文思考通过加入适当的中心化成分来优化CRUSH。

[![image-20201224163557319](https://durantthorvalds.top/img/mapx3.png)](https://durantthorvalds.top/img/mapx3.png)

上图(3)表示，MAPX对每次扩展记录为“一层”。MAPX 会加入select到placement rule当中。可与传统CRUSH对比理解。为了支持时间维度的映射并且对CRUSH改动最小，我们在root下插入一个虚拟层，每一个虚拟节点表示一个扩展层。 虚拟层使MAPX能够通过在将新对象映射到新层之前对CRUSH算法进行进一步处理来实现无迁移扩展。 由于新层不会影响旧层的权重，因此旧对象在旧层中的放置不会改变。

### 映射对象到PG

在每次扩展中，都会为新层分配一定数量的新创建的PG，每个PG的时间戳（tpgstpgs）等于该层的扩展时间（tltl）。 写入/读取对象O（带有创建时间戳记t0t0）时，我们首先通过以下方法计算O的PG的ID（pgid）：

pgid=Hash(name) mod INIT_PG_NUM[j]+j−1∑i=0INIT_PG_NUM[i]pgid=Hash(name) mod INIT_PG_NUM[j]+∑i=0j−1INIT_PG_NUM[i]

这里name表示对象名，`INT_PG_NUM[i]`表示PG第i层的初始值，并且第j层拥有最小的时间标签tl≤tOtl≤tO，PG可能会重新映射到其他层，例如进行负载平衡（第3.2节），`INIT_PG_NUM`是层的常量，因此从对象到PG的映射是不可变的。
因此，每个对象在创建过程中都映射到负责的PG，所有PG中的最新时间戳tpgs≤tOtpgs≤tO。 例如，图3（b）中的三个RBD1，RBD2和RBD3是分别在layer0，layer1和layer2扩展之后创建的。 RBD1，RBD2和RBD3的对象将使用三层的`INIT_PG_NUM`分别计算其在layer0，layer1和layer2内的PG。

### 将PG映射到OSD

[![image-20201224170254087](https://durantthorvalds.top/img/mapx4.png)](https://durantthorvalds.top/img/mapx4.png)

下面我们看下这篇文章具体是如何改进CRUSH的。

- 如果type不是“layer”那么就等同于原始CRUSH算法（2~4行）。
- 否则，我们将初始化一个图层数组，该数组按图层时间戳的升序（第5行）存储当前正在处理的存储桶（通常是根目录）下的所有图层。 我们还在第6到8行初始化num层（层数），pg（放置组）和〜o（输出列表）。然后循环（第9-21行）在层阵列中添加数字层 到输出列表〜o。 在大多数情况下，层数为 1，PG可以在一层中映射到OSD，也有number很大的情况，比如在两个扩展层进行镜像。
- 请注意，对象的副本不一定都放置在最新层上。 例如，假设最后一个扩展（第2层）在图3（a）中仅添加了两个机柜（即m = 2），但是第二个`select()`函数（`select(3,cabinet)`）需要三个机柜。 这将导致第一个`select()`函数`(select(1,layer)` 被调用两次，以满足遵循CRUSH回溯机制的规则：当`select()`函数无法在“ layer”存储桶下选择足够的项目时，MAPX 将保留（而不是放弃）选定的项目，并回溯到根以选择上一层下面的缺少的项目。 第12至14行检查先前的`select()`是否选择了图层，如果是，我们将继续进行下一个循环，以避免执行回溯时重复的图层选择。 仔细检查可确保算法1正确处理此情况，并分别为第一个和第二个`select()`函数返回layer2和layer1。

### 迁移控制

由于原始CRUSH的随机性和均匀性，基于MAPX的免迁移放置算法可在每层内提供（统计）负载平衡，当当前层的负载增加到与先前层相同的水平时，通过适时扩展群集来实现不同层之间的近似负载平衡。但是，层的负载可能由于例如对象的移除，OSD的故障或不可预测的工作负载变化而改变。 例如，在图3中，当第一个扩展（第1层）的负载与原始集群（第0层）的负载一样高时，该集群可能会执行第二个扩展（第2层），但随后会有大量 删除第1层的对象，因此前两层的负载可能会变得不平衡。
为了解决潜在的负载不平衡问题，我们设计了三种灵活的策略来动态管理MAPX中的负载，即放置组重新映射，群集收缩和层合并。

**PG重新映射**。 MAPX支持通过动态重新映射PG来控制对象数据的迁移。 每个PG都有两个时间戳，即等于PG初始层扩展时间的静态时间戳（tpgstpgs）和可以设置为任何层的扩展时间的动态时间戳（tpgdtpgd）。 与使用静态时间戳的从对象到PG的映射不同，从PG到图层的映射是通过将PG的动态时间戳与图层的时间戳进行比较来进行的（算法1中的第11行）。 因此，可以通过操纵动态时间戳（如图3（b）所示）将PG轻松地重新映射到任何层，该时间戳将通过增量映射更新通知所有OSD和客户端。 PG的时间戳存储开销适中。 例如，如果我们为每个PG时间戳使用一个字节索引（指向相应层的时间戳），该索引最多支持2828层= 256层），并且假设一台机器有20个OSD，每个OSD负责200个PG，则 1000个机器集群的时间戳的内存开销为1000×20×200×2×1B = 8MB。
**集群收缩**。 当层的负载低于阈值时，MAPX会通过从集群中删除该层的设备（例如OSD，机器和机架）来收缩集群，这是集群扩展的逆向操作。
给定要从群集中删除的层Ω，我们首先将Ω中的所有PG根据其总权重分配给其余层（为简单起见，重新分配不考虑层的实际负载），然后将PG迁移到 通过重新映射确定目标层（如上所述）。 缩小后，逻辑上保留了Ω层（没有物理设备或PG），并且它的`INIT_PG_NUM`不会更改，以免影响从对象到PG的映射（根据等式（2））。
**层合并**。 MAPX通过层合并来平衡两层（Ω和Ω’）的负载，这可以通过将一层（Ω’）的扩展时间设置为与另一层（Ω）相同的扩展时间来轻松实现。

我们通过将物理设备ID和该层的时间戳连接起来，为特定层（即，特定虚拟节点下方）的内部设备分配了虚拟设备ID。 我们使用虚拟节点的权重字段来记录图层的时间戳，并将其与PG的动态时间戳进行比较以进行图层选择。
MAPX不适合用于一般对象存储，主要是因为维护和检索任意对象的时间戳很重要。 按对象时间戳维护的开销与维护中央目录的开销类似，因此在诸如CRUSH和MAPX的集中式放置方法中应避免这种开销。
但是，MAPX适用于各种基于对象的存储系统，例如块存储（Ceph-RBD ）和文件存储（Ceph-FS），其中对象时间戳可以保持为更高级别 元数据。

### 实现

**Ceph-RBD**

Ceph-RBD。 我们已经为Ceph-RBD（RADOS块设备）实现了基于元数据的时间戳检索机制。 Ceph将RBD的元数据（例如数据对象名称的前缀以及卷，快照，条带等信息）存储在其`rbd_header`结构中，当客户端通过`rbd_open`挂载RBD时将检索该元数据。 由于RBD的对象可以在任何扩展之后创建，因此我们继承当前层的时间戳（创建对象时）作为对象的时间戳。 因此，我们在`rbd_header`结构中添加了一个每个对象的索引（称为对象时间戳`object_timestamp`），该索引指向每一层的扩展时间。 额外元数据的存储开销适中。 例如，如果我们为每个对象索引使用一个字节，而每个对象为4MB，则4TB RBD的对象时间戳数组的存储开销最多为4TB/4MB×1B = 1MB。

**CephFS**

我们还（部分）为CephFS（Ceph文件系统）实现了时间戳检索机制。
Ceph将文件元数据（包括文件创建时间）存储在inode结构中。 客户端在打开文件时读取inode并获取文件创建时间。 当前，我们让文件的所有对象继承文件的时间戳，以便我们可以按文件的粒度控制时维映射。 我们还计划支持更精细的对象时间戳维护。 如果文件大小超过阈值T（例如T = 100 MB），我们可以将其划分为每个小于100 MB的子文件。 文件的元数据既维护了从文件到其子文件的映射，又维护了每个子文件的创建时间戳，因此我们可以以子文件的粒度控制时间维度映射。

### 测试

[![image-20201224174039878](https://durantthorvalds.top/img/mapx6.png)](https://durantthorvalds.top/img/mapx6.png)

>  左图4，第99百分位 I/O延迟对比；右图5 ，IOPS对比

我们使用Ceph所有参数的默认值，但`OSD_max_backfills`除外。 如第1节所述，Ceph通过实现级优化减轻了CRUSH的迁移问题。 它使用参数`OSD_max_back_fill≥1`在数据迁移导致的性能下降的严重性和持续时间之间进行权衡。默认情况下，Ceph将参数`OSD_max_backfills`设置为1，这使迁移具有最低优先级，因此PG中的对象可能以极低的速度迁移。 尽管部分缓解了降级问题，但将`OSD_max_backfills`设置为1会大大延长迁移时间，并在迁移完成之前大大增加写入负载：等待迁移的PG写入将首先对原始OSD执行，然后异步进行 迁移到目标OSD。
显然，这使Ceph遭受的性能下降的幅度较小，但时间较长。 我们设置`OSD_max_backfills=10`，在此实验中更合理，因此可以优先考虑迁移，以证明MAPX和CRUSH在算法级别上的差异。

 图4显示了第99个百分位尾延迟的评估结果。 请注意，云存储方案通常关心的是（第99、99.9或99.99个百分位）尾部延迟，而不是平均延迟或中值延迟，以确保SLA（服务等级保障协议）。 MAPX的性能比CRUSH高出4.25倍，这主要是因为CRUSH中的迁移与正常的I / O请求严重竞争。 在此实验中，MAPX始终使用初始群集的六个OSD来满足I / O请求，因为它不会将现有RBD迁移到新OSD。 相比之下，CRUSH分别使用六个，九个和十二个OSD，但是CRUSH引起的数据迁移会严重降低性能，这对于延迟敏感的应用程序是不可接受的。
图5分别显示了MAPX和CRUSH中IOPS的评估结果。 每个结果均为20次运行的平均值，我们省略了误差线，因为与平均值的方差相对较小（小于5％）。 与延迟测试类似，在IOPS测试中，MAPX的性能明显优于CRUSH，最高可达到74.3％，这是因为CRUSH的数据迁移可以应付正常的I / O请求。

[![image-20201224200625534](https://durantthorvalds.top/img/mapx7.png)](https://durantthorvalds.top/img/mapx7.png)

> 图7：MAPX和CRUSH的第99个百分点的I / O延迟（在群集收缩期间）。

[![image-20201224200711635](https://durantthorvalds.top/img/mapx8.png)](https://durantthorvalds.top/img/mapx8.png)

> 图8：在MAPX中合并的层中受影响的PG的数量（四个扩展之后）。 由于CRUSH不支持合并，因此我们在每次CRUSH扩展后测量受影响的PG的数量以供参考。

我们使用CrushTool模拟MAPX中的图层合并。
我们采用三向复制，其中每个对象在三个OSD上存储三个副本。 最初，存储集群由5个机架组成，每个机架有20台计算机。 一台机器有20个OSD。 总共有100台机器和2000个OSD，可存储20万个PG。 我们将集群扩展了四倍。 在每个扩展中，我们将一个机架的新层（包含20台计算机和400个OSD）添加到一个新层，并在新层中添加40,000个新PG。 显然，MAPX将所有新PG映射到新添加的OSD上，因此不会发生迁移。 四个扩展之后，总共有9个机架，180台机器和3600个OSD，可存储360,000个PG。 然后，我们合并第一扩展和第二扩展的40台机器，并测量MAPX中的合并影响了多少个PG。
结果如图8所示，其中MAPX中的图层合并影响了两个合并图层的所有80,000 PG中的70,910 PG。 MAPX的层合并中受影响的PG的相对较高比例取决于作为参考，我们还模拟了CRUSH中的四个扩展，其中让群集最初具有360,000个PG，并且在扩展期间不添加新PG，因为否则CRUSH会将映射从对象更改为PG，从而导致更多PG迁移。
图8还显示了CRUSH的每次扩展会影响多少PG。 例如，当机器数量从160增加到180时，几乎90％的PG在第四次扩展中都受到影响。

------

## 引用和参考文献

1. WEIL, S. A., BRANDT, S. A., MILLER, E. L., AND MALTZAHN,C. Crush: Controlled, scalable, decentralized placement of replicateddata. In *SC*’06: Proceedings of the 2006 ACM/IEEE Conference on*Supercomputing* (2006), IEEE, pp. 31–31. [↩](https://durantthorvalds.top/2020/11/25/A first glance at CRUSH/#fnref:1)
2. Wang L, Zhang Y, Xu J, et al. {MAPX}: Controlled Data Migration in the Expansion of Decentralized Object-Based Storage Systems[C]//18th {USENIX} Conference on File and Storage Technologies ({FAST} 20). 2020: 1-11. [↩](https://durantthorvalds.top/2020/11/25/A first glance at CRUSH/#fnref:2)

