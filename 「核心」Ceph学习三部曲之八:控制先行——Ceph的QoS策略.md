# 控制先行——Ceph的QoS策略

本blog包括理论和实践两个部分，实践部分需要您事先部署成功Ceph集群！

参考《Ceph设计与实现》谢型果等，第五章。9/7/2017 韩国SK团队进展 .PPT见[链接](https://www.slideshare.net/ssusercee823/implementing-distributed-mclock-in-ceph),Code见[链接](https://github.com/ceph/ceph/pull/16369).

> 

Ceph作为开源社区的明星，因为其高可扩展性、高可靠性，受到各大厂商的热烈追捧，并成为OpenStack事实上的默认存储后端。但是人非圣贤，孰能无过?它也同样面临I/O资源分配的问题。为了保证客户能够体验更好的服务，Ceph引入了QoS(QualityofServiceQualityofService).

dmClockdmClock是一种分布式系统的I/O调度算法，它起源于mClock（[论文地址](https://www.usenix.org/legacy/events/osdi10/tech/full_papers/Gulati.pdf)）,目前被应用于Ceph中。

Ceph作为分布式存储系统集大成者，不能像传统QoS实现位置首选的中心节点，必须在每个OSD中实现QoS。下图展示了社区目前（2017版本）的Ceph QoS pool单元：

[![image-20201219164136113](https://durantthorvalds.top/img/image-20201219164136113.png)](https://durantthorvalds.top/img/image-20201219164136113.png)

## 1 dmClock算法原理

首先我们先了解mClock，这是一种基于时间标签的I/O调度算法，最先被VMware提出了的用于集中式管理的存储系统。它使用了reservation（预留，表示客户端获得的最低I/O资源）、weight（权重，表示客户端占用共享I/O资源的权重，共享I/O指满足预留之后剩余的I/O资源）以及limit（上限，表示用户所能使用的最大I/O）作为一套模板（QoS spec），作用于不同的客户端。下图是其典型应用模型：

[![image-20201216151614635](https://durantthorvalds.top/img/dmClock.png)](https://durantthorvalds.top/img/dmClock.png)

mClock是典型的C/S架构，Client可以驻留在实际的客户端或者服务器端，主要负责下发QoS模板的参数值、收集请求的完成信息等；server为mClock的服务端，实现I/O调度的核心功能。

其算法流程如下：

1. Server为每个客户端设置一套QoS模板参数，包括预留（r），权重（w）和上限（l）三个部分，并依次计算出I/O请求的时间标签，其中预留和上限为绝对时间，权重标签为相对时间。
2. 服务器分为两个阶段来处理I/O请求：一是Constarint-based，只处理满足预留时间标签的请求；二是Weight-based阶段，处理满足上限的时间标签的权重标签请求。
3. 服务器先工作也Constraint-based，再转入Weight-based阶段。

如果用qiqi表示QoS的模板参数qi∈{ri,wi,li}qi∈{ri,wi,li}，QriQir表示来自第i个客户的第r个请求的时间标签。有如下公式：

Qri=max{Qr−1i+1/qi,current_time}Qir=max{Qir−1+1/qi,current_time}

以第一个请求到达时间作为初始基准标签，后续标签依据预设的模板参数值，对单位时间进行均匀切分计算而来。在Constraint-based阶段 ，各客户端请求被均匀的处理，而在weight-based阶段，则将出现竞争，由于权重标签是相对值，它和真实的时间之差通常会很大，从而出现饥饿现象。因此需要调整旧client的权重标签，以新client的权重时间标签为基准，添加一个补偿值。

dmClock是mClock 的分布式版本，两者的基本原理相同。每个请求的时间标签计算公式如下：

Rri=max{Rr−1i+ρi/ri,current_time}Wri=max{Wr−1i+δi/wi,current_time}Lri=max{Lr−1i+δi/li,current_time}Rir=max{Rir−1+ρi/ri,current_time}Wir=max{Wir−1+δi/wi,current_time}Lir=max{Lir−1+δi/li,current_time}

dmClock和mClock的主要区别在于：

- 分布式系统具有多个服务器，服务器回应每个I/O请求时，返回其在哪个阶段被处理完成。
- 客户端记录每个服务器完成的请求个数，在向服务器下发请求时，携带距上次下发请求以来，收到完成的请求个数的增量，并且是除目标服务器之外，其它服务器完成的请求数之和，分别用ρρ和δδ表示两个阶段的增量处理个数。
- 服务器计算请求的时间标签，使用ρρ和δδ作为调整因子，不再以1/q1/q均匀递增。从而减小了每个服务器处理的请求的个数。

通过对ρρ和δδ的调整，使得集群整体对外提供预期的I/O处理效果。

## 2 QoS的设计与实现

在OSD中，存在`op_shardedwq`队列处理各种来自上级的I/O，并且这是一个复合队列，通常包含若干子队列。I/O请求从队列出列后，通过ObjectStore接口与磁盘交互。

OSD支持多种不同的子队列，目前主要包括优先级队列（prio）和基于权重的优先级队列（wpq）两种，

I/O操作类型主要包括以下几种：

1. ClientOp：来自客户端的读写I/O请求；
2. SubOp：OSD之间的I/O请求。主要包括客户端I/O产生的副本间数据读写请求，以及由数据同步、数据扫描、负载均衡等引起的I/O请求。
3. SnapTrim：快照数据删除。
4. Scrub：用于发现对象的静默数据错误。其中Scrub只扫描元数据，而Deep Scrub对对象整体进行扫描。
5. Recovery：数据恢复和迁移。集群扩容、OSD添加与移除、手动进行数据重平衡都有可能触发recovery过程。

[![image-20201218175910498](https://durantthorvalds.top/img/dmClock2.png)](https://durantthorvalds.top/img/dmClock2.png)

上图表示OSD内部结构，我们对原有的prior队列，wpq队列以及新增的dmClock队列加以分析。

## 2.1 优先级队列prior

prior是一个基于令牌桶的优先队列，由三个级别组成：1.I/O类型的优先级prior； 2. 客户端级别的client队列；3.真实的list请求；每个元素包括请求r以及数据大小cost。可以把prior看成一个三维的队列。

每个prior队列，在其第一个请求入队时，被创建，并分配一个大小为`max_tokens`的令牌桶。

关于出队的规则，有以下几点：

1. 选择合理的prior：从小到大轮询所有prior，只要满足条件则被选中。即，该prior队列的令牌桶中剩余 的令牌数量足够多，可以容纳将被选中的请求（每个请求出队时，必须拿到与其大小cost相当的令牌的个数）。
2. 选择合理的client：对同优先级下的client进行轮询，即第一个client出队一个请求后，将请求的出队权交给第二个client，该优先级再次被选中时，从第二个client出队请求。
3. 选择合理的请求：从被选中的client的请求list表中出队一个请求（FIFO策略）。

当出队一个请求时，从令牌桶中拿掉与请求大小cost相当的令牌个数，随后将拿到的令牌数分发、交还至所有prior队列，使得令牌总数维持不变。令牌分发的规则是，按照各自prior的占用比重，每个prior队列可回收的令牌总数token：

token=priortotal_prior×costtoken=priortotal_prior×cost

对于prio较大的队列将优先被考虑，I/O类型到达优先级可以用过配置参数修改，如下表所示：

| 优先级配置参数             | 默认值 |
| -------------------------- | ------ |
| `osd_client_op_priority`   | 63     |
| `osd_snap_trim_priority`   | 5      |
| `osd_scrub_prority`        | 5      |
| `osd_recovery_op_priority` | 3      |

优先队列同样存在一些局限性，如果集群中某个OSD分布了比其它OSD更多的PG或者Object对象时，该OSD由于需要处理更多的副本请求，导致客户端长时间得不到处理出现饥饿现象。

为此社区引入了基于权重的优先级队列wpq.

## 2.2 权重优先级队列wpq

基于权重的wpq不需要创建令牌桶，与prior仅在出队方式上有区别：

- 采用权重概率的方式确定prior级别，每个队列的优先级prior作为其权重，该prior队列被选中的概率即为其权重占总权重的比例。通过随机数对total_prior取余的方式得到在[0,totalprior−1][0,totalprior−1]的范围内完全随机分布。(rand()%total_prior)
- 被选中的prior队列并不一定能出队请求，还需要根据将要出队的请求大小来确定，即是否满足rand()%maxcost≤(maxcost−(requestsize∗9/10))rand()%maxcost≤(maxcost−(requestsize∗9/10))

max_cost指该prior队列最大的请求的大小。较小请求对应右边值更大，因而出队概率更高。

- client级别和真实请求的选择和prior相同。

## 2.3 dmClock队列

dmClock是一个两级映射的队列，第一级为客户端的client队列，第二级是真实的请求队列，每个请求包含三个时间标签$$, 其中i表示所属的client编号，没有使用优先级prior。

dmClock采用**完全二叉树**这种数据结构来处理大量的请求。分别构建预留时间标签、权重时间标签、上限时间标签二叉树，树节点为每个client对应的请求队列，节点在二叉树的位置，则根据其队首元素的三个标签决定，总体原则是父节点时间标签小于子节点。

### 入队

对于已存在的client，将请求直接挂入请求队列的尾部；对于新增client，除了新创建一个对应的请求队列，还要将队列作为一个新节点加入标签二叉树。根据完全二叉树的特点，采用顺序的变长数组结构存储，新节点先加入二叉树的尾部，再调整至合适的位置，二叉树的调整规则如下：

- R预留标签二叉树：以节点的队首元素的预留标签为基础，值小的节点调整至树的上层，反之调整至下层，最终根节点的预留标签值最小；
- W权重标签二叉树：该树的节点中有两种状态，一种满足出队条件，其上限小于或等于当前时间，ready标记被置为true；另一种不满足出队条件，ready被置为false。对节点位置调整时，根据请求队列队首元素的ready状态，满足出队条件的节点调至上层，不满足出队条件的调至下层。相同状态的节点再由权重标签的大小决定，标签值较小的往上调整，反之往下调整。
- L上限标签二叉树：用于判决权重二叉树的节点中的请求是否满足出队条件，也使用ready进行标记区分。但与权重二叉树不同，ready为true的节点向下调整。ready相同的节点根据上限标签值的大小决定。

### 出队

首先进入constraint-based阶段取预留标签二叉树的根节点的请求队列，判断其队首元素的预留标签是否小于当前时间，作为是否满足出队条件的依据。如果条件满足，则选取该节点对应的client，从其请求队列的队首出队一个元素；否则进入weight-based阶段，从上限标签二叉树的根节点开始，逐个判断队首元素的上限标签是否小于等于当前时间，并，设置满足条件的请求的ready为true，以决定其是否可以参加随后的**权重竞争**。所谓的权重竞争，指对所有满足上限条件的clients，依据其队首元素的权重标签值，调整自身在权重二叉树的位置的过程，最终位于根节点的client胜出。

## 2.4 Client的设计

目前Client 的设计有三种初步方案：

1）使用mClock作为一种分配调度策略，控制客户端的I/O请求和Ceph内部产生的I/O调度。这将所有不同真实客户端作为同一个抽象的client考虑；

2）使用dmClock以存储池或者卷为粒度，为其设置QoS模板参数，客户端请求以消息的形式发送至OSD。这是将每个存储池或存储池中的卷作为一个client；

3）使用dmClock为每个真实客户端设置一套QoS模板，这是将每个真实client作为一个client；

对于每一个 QoS 对象来说，首先需要在 OSD 中实现以下前置条件:

1. 对于每个客户端来说，每个请求具有唯一的标识符，客户端和请求形成全局唯一
2. 必须将每个请求对应的 QoS 控制信息持久化
3. OSD 能够通过标识符从来访的请求中找到 QoS 控制信息

## 2.5 总结与展望

目前对QoS的优化有以下几种方向：

1. 合理模板参数的设置

   只有集群运行于超负荷时（入队速率大于出队速率），权重的效果才能体现出来。

2. I/O带宽的限制

   QoS限速体系的设计，比如通过OIO throttling来进行限速【参考SK团队PPT】。

3. 突发I/O的处理

   dmClock的做法是为每个client预先设置一个可调整参数σσ，当出现突发I/O状况，减小该client的权重标签至t−σi/wit−σi/wi，从而使其在权重竞争中更有优势。这里有一个问题，就是服务端如何判断客户端产生突发I/O访问？通过记录客户端的状态是一种可行的方式。
