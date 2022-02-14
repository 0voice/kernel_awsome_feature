本blog包括理论和实践两个部分，实践部分需要您事先部署成功Ceph集群！由于篇幅较大，建议先看看完理论部分的术语再看官方实践部分，最后到理论部分搜索关键字进行理解。

参考《Ceph设计与实现》谢型果等，第四章。以及[官方PG教程](https://docs.ceph.com/en/latest/rados/operations/placement-groups/)。

# 1 基本概念

PG是Ceph最难理解的部分之一，但是它也是Ceph最精妙有意思的部分。

Placement Groups。即归置组（又称放置组）。Ceph对所有的存储资源都进行池化管理，对对象进行两级映射存储Objects→PGs→OSDsObjects→PGs→OSDs

- 第一级映射是静态的，负责将任何前端类型的应用数据按照固定大小进行切割、编号后作为随机哈希函数输入，均匀映射至PG，以实现负载均衡。
- 第二级映射实现PG到OSD的映射。

PG最引人关注的特性是它可以在OSD之间自由的迁移，这是Ceph赖以实现自动数据恢复、自动数据平衡等高级特性的基础。因为PG的数量远小于对象的数量，因此以PG为单位进行操作更具灵活性。

[![img](https://durantthorvalds.top/img/ceph-arch.png)](https://durantthorvalds.top/img/ceph-arch.png)

存储池中的对象到OSD的映射是通过PG来完成。一方面，存储池中PG数目决定了其并发处理多个对象的能力；另一方面，过多的PG会消耗大量CPU，同时容易使得磁盘长期处于过载状态。（经验表明：磁盘利用率保持在70%左右可以使得I/O并发能力和平均相应时延最佳）。

因此在创建存储池时，需要合理的指定PG的数目，一般将每个OSD中PG限制在100个左右最佳。

需要注意的是，创建存储池中指定的PG数目其实是指**逻辑PG数目**。为了数据的可靠性，Ceph会将每个逻辑PG转换为多个实例PG，由它们负责将对象的不同备份或者部分写入不同的OSD。

如果使用多副本，那么每个逻辑PG被转换为与副本数相等的PG实例。处于数据一致性考虑，我们可以选择Paxos作为数据分布一致性算法，但这样过于重量级。其实我们只需要在PG实例中选择一个主要的PG作为通用的入口进行操作分发或者集中（例如peering）。

如果使用纠删码，每个逻辑PG会被分为k+m个实例。与多副本不同，这些PG只保存每个对象的一个分片（shard），所以需要对其身份进行严格区分。同样，需要在PG实例中选择一个主要的PG。

按照约定，主要PG由CRUSH返回的第一个OSD充当。PGID由CRUSH计算，使得PG在所有OSD之间均匀分布。

## 1.1 术语和约定

- PGID

  PG有一个全局唯一的ID——PGIDPGID，所有的pool由Monitor统一管理，由pool-id+PG在pool内唯一ID+shard（仅适用于纠删码存储池）组成。

- OS

  指对象存储的种类（Object Store）例如FileStore和BlueStore。

- Info

  PG内基本元数据信息。

- Log

  基于Eversion顺序记录所有客户端发起的修改操作的历史信息，为后续提供历史操作回溯和数据同步的依据。

- Authoritative History

  指权威日志。它是Peering过程中进行数据同步的依据，通过交换Info并基于一定的规则从所有的PG实例中选举产生。通过重放权威日志，可以使得PG内部每个对象的版本号达成一致。

- PGBackend

  字面意思是PG后端。负责将对原始对象的操作转化为副本之间的分布式操作。对于多副本而言是`ReplicatedBackend`；对于纠删码而言是`ECBackend`。

- Epoch

  一般情况下指OSDMap（OSDMap 是 Ceph 集群中所有 OSD 的信息）的版本号，由Monitor生成，总是单调递增。Epoch变化意味着OSDMap发生变化，需要通过一定的策略扩散至所有客户端和位于服务端的OSD。

- Version

  version指本次修改生效之后的版本号。

- Eversion

  由Epoch和Version组成。Version总是当前的Primary产生，连续单调增，和Epoch一起标志一次PG内修改操作。如223’23。

- Interval

  指OSDMap一个连续的Epoch的持续时间，Interval和具体的PG绑定。

- Acting Set

  指一个有序的OSD集合。当前或者曾在某个Interval负责承载对应PG的PG实例。通常与Up Set相同，但有时候设置了PG Temp会导致两者不相同。

- Primary

  指Acting Set的第一个OSD，负责处理来自客户端的读写请求，同时也是peering的发起者和协调者。

- Peering

  指归属于同一个PG的所有PG实例就本PG所存储的全部对象以及对象相关的元数据操作进行协商并最终一致的过程。

  Peering 基于Log和Info进行。这里的达成一致，并不表示每个PG实例都能获得最新的内容。事实上，为例尽快恢复对外业务，一旦Peering完成，在满足条件下就可以切换为Active状态，后续的数据恢复可以在后台进行。

- Recovery

  指针对PG某些实例进行数据同步的过程，其最终目标是将PG重新变为Active+Clean状态。它可以在后台进行。

- Backfill

  Backfill字面意思是回填，是Recovery的一种特殊场景，指Peering完成后，如果基于当前的权威日志无法对Up Set当中的某些PG实例实现增量同步，则通过完全拷贝当前的Primary所有对象的方式进行**全量同步**。

- PG Temp

  作为PG临时载体的OSD集合。Peering过程中，如果当前的Interval通过CRUSH计算的Up Set不合理（例如Up Set中的一些OSD新加入集群，根本没有PG的任何历史信息），那么可以通知OSDMonitor设置PG Temp的方式来显式的指定一些仍然具有相对完备PG信息的OSD加入Acting Set，使得Acting Set中的OSD再完成Peering之后能够临时处理客户端发起的读写请求，以尽可能减少业务中断的时间。上述过程会导致Up Set和Acting Set临时不一致。UpSet是CRUSH原始计算的映射结果；因为Peering过程中不能处理客户端读写请求，引入PG Temp可以缩短业务中断的时间，当Up Set中的副本在后台通过Recovery 或者Backfill 完成数据同步时，此时可以通知OSDMonitor取消PG Temp.

  - 之所以需要PG Temp来修改OSDMap，是因为需要同步通知到所有客户端，让它们后续将读写请求发送到Acting Set而不是Up Set中的Primary。
  - PG Temp生效之后，PG将处于Remapped状态。
  - Peering完成之后，Up Set中与Acting Set不一致的OSD将在后台通过Recovery或者Backfill的方式与当前的Primary进行数据同步；数据同步完成后，PG需要重新修改PG Temp为空集合，完成Acting Set至Up Set的切换，此时取消Remapped标记。

- Stray

  指PG所在的OSD不是PG当前的Acting Set中。

[![img](https://durantthorvalds.top/img/ceph-d.png)](https://durantthorvalds.top/img/ceph-d.png)

> 上图1：客户侧，Monitor和 Primary、Replica的关系

[![img](https://durantthorvalds.top/img/ceph_io2.png)](https://durantthorvalds.top/img/ceph_io2.png)

> 上图2：正常的读写流程
>
> 客户侧先产生一个cluster handle（也就是后文所说的op）。之后连接monitor，再从Primary OSD进行读写。

[![img](https://durantthorvalds.top/img/ceph-e.jpg)](https://durantthorvalds.top/img/ceph-e.jpg)

> 上图3：Backfill的读写流程. 由于一些OSD离线太久，或者新的OSD加入到集群导致PG实例整体迁移，上图明显属于后者，需要通过Backfill指定临时主进行全增量同步并且选择新的Primary。

客户端读写流程详细分析：

1. OSD收到客户端发出的读写请求，将其封装为一个op（客户端发出的读写请求），并基于其携带的PGID发送至对应PG。
2. PG收到op之后，完成一系列检查，所有条件均满足后，开始真正执行op。
   - 如果op只包含读操作，那么直接执行同步读（对应多副本），或者异步读（对应纠删码），等待操作完成后向客户端应答。
   - 如果op包含写操作，首先由primary基于op生成一个针对原始对象的事务及相关操作，然后将其提交给PGBackend安装备份策略转化为每个PG实例（包含所有primary和所有Replica）真正需要执行的本地事务并进行分发，当primary收到所有副本的写入完成应答之后，对应的op执行完成，此时由primary向客户端回应写完成。

## 1.2 PG快速定位对象stable_mod

> 对应参考书 2.2.1 PG

普通的取模无法保证“归属于某个PG的所有对象的低n位相同”这个特性，为此人们提出了stable_mod。

首先由特定类型的Client根据其操作的对象名计算出一个32位的哈希值，然后根据其操作的对象名计算出一个32位的哈希值，然后根据归属的pool及此时的哈希值，通过简单的计算，比如取模，即可找到最终承载该对象的PG。

我们发现如果pool内的PG数目如果能写成2n2n形式，那么其低n比特都是相同的。我们将2n−12n−1称为PG的**掩码**。否则，若PG不能写成2n2n的形式，则不能保证针对不同的输入低n比特相同这一“稳定”的性质。（比如有12个PG，那么对于属于同一个pool的PGID只有低两位相同）

因此一种行之有效的方法是用掩码代替取模。取hash低n-1位，即hash&(2n−1)hash&(2n−1) .但这种映射存在问题，如果PG数目不能被2整除，那么采用这种方式会导致空穴，也就是取模结果没有实际PG对应。

比如一个pool有12个PG，n=4，但是12-15这些值都浪费了：

> | 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    | 8    | 9    | 10   | 11   | 12   | 13   | 14   | 15   |
> | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
> |      |      |      |      |      |      |      |      |      |      |      |      | x    | x    | x    | x    |
>
> 我们可以想办法压缩空间 ，hash&(2n−1−1)hash&(2n−1−1)，使得不能被2整除的PGID仍能被合理映射。
>
> | 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    | 8    | 9    | 10   | 11   | 12   | 13   | 14   | 15   |
> | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
> |      |      |      |      |      |      |      |      | 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    |
>
> 如果hash&(2n−1)<pgnumhash&(2n−1)<pgnum，那么可以直接返回hash&(2n−1)hash&(2n−1).
>
> 否则，我们返回hash&(2n−1−1)hash&(2n−1−1).
>
> 比如：
>
> ```
> 0x05 stable_mod 12 = 5
> 0x0D stable_mod 12 = 5
> 0x15 stable_mod 12 = 5
> 0x1D stable_mod 12 = 5Copy
> ```
>
> 在参考书上被称为稳定哈希（stable_mod hash）。

## 1.3 PG分裂

Ceph主要设计理念之一是高扩展性。当集群中PG增加，新的PG会被随机均匀地映射至所有OSD上。作为stable hash的输入的PG数目已经发生变化，导致某些对象从旧PG重新映射至新PG，因此需要转移这部分对象，我们称为**PG分裂**（这也是为什么PG数目必须是2的次幂的原因！），为了避免业务长时间停顿，我们必须尽可能减少对象移动的次数。

下面我们通过一个简单的例子来了解PG分裂吧！

某个存储池的pg_num由2424增加到2626，容易验证其中所有对象的哈希值都可以分成下图中的四种类型：

| MSB  |      |      |      |      | LSB  |
| :--: | :--: | :--: | :--: | :--: | :--: |
|  0   |  0   | X3X3 | X2X2 | X1X1 | X0X0 |
|  0   |  1   | X3X3 | X2X2 | X1X1 | X0X0 |
|  1   |  0   | X3X3 | X2X2 | X1X1 | X0X0 |
|  1   |  1   | X3X3 | X2X2 | X1X1 | X0X0 |

针对这四种类型的PG使用新的pg_num再stable_mod，结果如下表所示：

| 对象32位哈希                                           | stable_mod |
| ------------------------------------------------------ | ---------- |
| 0’b???? ???? ???? ???? ???? ??? ???00 X3X2X1X0X3X2X1X0 | X          |
| 0’b???? ???? ???? ???? ???? ??? ???01 X3X2X1X0X3X2X1X0 | X+16       |
| 0’b???? ???? ???? ???? ???? ??? ???10 X3X2X1X0X3X2X1X0 | X+2*16     |
| 0’b???? ???? ???? ???? ???? ??? ???11 X3X2X1X0X3X2X1X0 | X+3*16     |

因此仅有第一种类型不用迁移，我们需要创建三个新的PG来转移余下的类型。因此原来一个PG变成了4个。这被形象的称为PG分裂。

由于归属于同一个PG的所有对象总是可以保证其至少n-1位相同，移动自然的想法是使用哈希值逆序之后作为目录分层的依据。例如0xA4CEE0D2，一种可能的分层依据为`./2/D/0/E/E/C/4/A/0xA4CEE0D2`。可以看的出来Ceph的创作者是何等富有革新精神！

引入PG分裂机制后，如果仍然使用PGID作为CRUSH输入，据此计算新增孩子PG在OSD之间的映射结果，由于此时每个PGID都不同，必然触发大量迁移。考虑到分裂之前PG在OSD之间的分布已经趋于平衡，更好的方法是让同一个祖先诞生的所有孩子PG分布保存相同的副本分布，整个集群的PG分布仍是均衡的。为了应对PG分裂，还需要额外记录分裂之前祖先PG数量，后者称为PGP数量(Placement Group Placement)。

# 2 详细剖析PG读写流程

## 2.1 消息接收与分发

OSD绑定的Public Messenger 收到客户端发送的读写请求后，通过OSD注册的回调函数——`ms_fast_dispatch`进行快速派发：

- 基于消息(Messenger)创建一个op， 用于对消息进行跟踪，并记录消息携带的Epoch。
- 查找OSD关联的客户端会话上下文，，将op加入其内部的`waiting_on_map`队列，获取OSDMap，，并将其与`waiting_on_map`队列的所有op进行逐一比较——如果OSD当前的OSDMap的Epoch不小于op所携带的Epoch，则进一步将其派发至OSD的`op_shardedwq`队列（OSD内部的工作队列）；否则直接终止派发。
- 如果会话的上下文的`waiting_on_map`不为空，说明至少存在一个op，其携带的Epoch比OSDMap更新，此时将其加入OSD全局`session_waiting_for_map`集合，该集合汇集了当前所有需要等待OSD更新完OSDMap之后才能继续处理的会话上下文；否则将对应的会话从`session_waiting_for_map`中移除。

上面有些概念我们先抛砖引玉，下面我们具体讲

## 2.2 do_request

`do_request`作为PG处理op的第一步，主要完成全局（PG级别的）检查：

- Epoch——如果op携带的Epoch更新，那么需要等待PG完成OSDMap同步之后才能进行处理。

- op能否被直接丢弃——一些可能的场景有：

  - op对应的客户端链接已经断开
  - 收到op时，PG当前已经切换到一个更新的Interval（OSD生成一个连续Epoch的间隔），后续客户端会重发。
  - op在PG分裂之前发送，后续客户端会重发。

- PG自身的状态如果不为Active，op同样会被阻塞。

  PG内部维护了许多不同类型的重试队列，保证请求按顺序被处理。当对应限制解除之后，op会重新进入`op_shardedwq`，等待被PG执行。

## 2.3 do_op

do_op进行对象级别的检查：

1）按照op携带的操作类型，初始化op中各种标志位。

2）完成对op的合法性校验，不合法的情况包括：1.PG包含op所携带的对象；2.op之间携带可以并发执行的标志 ；3. 客户端权限不足；4. op携带对象名称、key或者命名空间长度超过最大限制（只有在FileStore下存在此限制）5.op对应客户端被纳入黑名单 6. op在集群被标记为Full之前发送 7. PG所在的OSD存储空间不足 8. op包含写操作并且企图访问快照对象 9. op包含写操作并且一次写入量太大（超过`osd_max_write_size`）。

3）检查op携带的对象是否不可读或者处于降级状态或者正在被scrub（读取数据对象并重新计算校验和），加入相应队列。

4）检查op是否为重发（基于op的repid在当前的Log中查找，如果找到说明为重发）。

5）获取对象上下文，创建OpContext对op进行跟踪，并通过`execute_ctx`真正开始执行op。

> 💬关于可用存储空间控制
>
> Ceph使用四个配置项，`mon_osd_full_ratio`、 `mon_osd_nearfull_ratio`、`osd_backfill_full_ratio`（OSD空间利用率大于此值，PG被拒绝以backfill方式迁入）、`osd_failsafe_full_ratio`（防止OSD磁盘被100%写满的最后一道屏障）。
>
> 每个OSD通过**心跳**机制周期性的检测自身空间利用率，并上报至Monitor。`osd_backfill_full_ratio`的存在意义是有些数据迁移是自动触发的，我们无法预料到自动数据平衡后数据会落到哪个磁盘，因此必须设置此项进行控制。
>
> 💬关于对象上下文 (原书134页图4-3)
>
> 对象上下文主要保存了对象OI(Object Info)和SS(Snap Set)属性。同时表明对象是否仍然存在。查找head对象上下文相对简单，如果没有在缓存中命中，直接在磁盘中读取即可。然而如果op直接操作快照或者对象克隆，这个过程将变得很复杂。其难点在于一个克隆对象可以对应多个快照，因此需要根据快照序列号定位到某个特定的克隆对象，然后通过分析其位于OI中的snap属性才能进一步判断快照序列号是否位于克隆对象之中。

## 2.4 execute_ctx

`execute`是真正执行op的步骤。它首先基于当前的快照模式，更新OpContext中的快照上下文(SnapContext)——如果是自定义快照模式，直接基于op携带的快照信息更新；否则基于PGPool更新。

为了保证数据的一致性，所以包含修改操作的PG会预先由Primary通过`prepare_transcation`封装为一个PG事务，然后由不同类型的PGBackend负责转化为OS(objectStore,笔者注)能够识别的本地事务，最后在副本间分发和同步。

## 2.5 事务准备

针对多副本，因为每个副本保存的对象完全相同，所以由Primary生成的PG事务也可以直接作为每个副本的本地事务直接执行。引入纠删码之后，每个副本保存的都是独一无二的分片，所以需要对原始对象的整体操作（对应PG操作）和每个分片操作（对应OS事务）加以区分。

本节介绍如何基于op生成PG级别的事务，这个过程通过`prepare_transaction`完成。

1. 通过`do_osd_ops`生成原始op对应的PG事务。
2. 如果op针对`head`对象操作，通过`make_writable`检查是否需要预先执行克隆操作。
3. 通过`finish_ctx`检查是否需要创建或者删除`snapdir`对象，生成日志，并更新对象的OI及SS属性。

下面我们具体介绍相关流程：

### 1 do_osd_ops

1. 检查write操作携带的`trancate_seq`，并和对象上下文保存的`truncate_seq`比较从而判定客户端执行write操作和trimtrunc/truncate操作的真实顺序，并对write操作涉及的逻辑地址范围进行修正。
2. 检查本地写入逻辑地址范围是否合法——例如我们当前限制对象大小不超过100GB。(对应`osd_max_object_size`)。
3. 将write对象转化为PGTransaction中的事务。
4. 如果是满对象写（包括新写或者覆盖写），或者为追加写并且之前存在数据校验和，则重新计算并更新OI中数据校验和，作为后续执行Deep Scrub的依据；否则清除校验和。在OpContext中积累本次write修改的逻辑地址范围以及其它统计（例如写操作次数、写入字节数），同时更新对象大小。

### 2 make_writable

如果op针对head对象进行修改

1. 判断head对象是否需要执行克隆：取对象当前的SnapSet，和OpContext中SnapContext 中内容进行比较——如果SnapSet中最新的快照序列号比SnapContext中最新的快照序列号小，说明自上次快照之后，又产生新的快照。此时不能直接对head对象进行修改，而是需要先执行克隆（默认为全对象克隆）。如果SnapContext携带了多个新的快照序列号，那么所有比SnapSet中更新的快照序列号都将关联至同一个克隆对象。

> 这里有一个特殊情况——如果当前操作为删除head对象，并且该对象自创建之后没有经历任何修改（此时SnapSet为空），也需要该head对象正常执行克隆后再删除，后续将创建一个snapdir对象来转移这些快照及相关的克隆信息。

1. 创建克隆对象，需要同步更新SS属性中相关信息：
   - 在`clones`集合中记录当前克隆对象中最新快照序列号。
   - 在`clone_size`集合中更新当前克隆对象的大小——因为默认使用全对象克隆，所以克隆对象大小为执行克隆时head对象head对象的实时大小。
   - 在`clone_overlap`集合中记录当前克隆对象与前一个克隆对象之间的重合部分。
2. 为克隆对象生成一条新的、独立的日志，更新op中日志版本号。
3. 最后，基于SnapContext更新对象SS属性中快照信息。

### 3 finish_ctx

顾名思义，`finish_ctx`完成事务准备阶段最后的清理工作。

1）如果创建head对象并且snapdir对象存在，则删除snapdir对象，同时生成一条删除snapdir对象的日志；如果删除head对象并且对象仍然被快照引用，则创建snapdir对象，同时生成一条创建snapdir对象的日志，并将head对象的OI和SS属性用snapdir’对象转存

2）如果对象存在，则更新对象OI属性——例如version、last_reqid、mtime等；进一步，如果是head对象，同步更新其SS属性。

3）生成一条op操作原始对象的日志，并追加至现有的OpContext中的日志集合中。

4）在OpContext关联的对象上下文中应用最新的对象状态和SS上下文。

## 2.6 注册回调函数

PG事务准备后，如果是纯粹的读操作，如果是同步读（对于多副本），op已经执行完毕，此时可以直接向客户端应答；如果是异步读（针对纠删码），则将op挂入PG内部的异步读队列，等待异步读完成之后再向客户端应答。

如果是写操作，则注册如下几类回调函数：

- `on_commit`——执行时，向客户端发送写入完成应答；
- `on_success`——执行时，进行与Watch\Notify相关的处理；
- `on_finish`——执行时，删除OpContext。

## 2.7 事务分发与同步

事务的分发与同步由Primary完成，具体而言是通过RepGather实现的。RepGather被提交到PGBackend，后者负责将PG事务转化为每个副本之间的本地事务之后再进行分发。

对于纠删码而言，当涉及覆盖写时，如果改写的部分不足一个完整条带（指写入的起始地址或者数据长度没有进行条带对齐），则需要执行RMW，这期间需要多次执行补齐读、重新生成完整条带并重新计算校验块、单独生成每个副本的事务并构造消息进行分发（Write）、同时在必要时执行范围克隆和对PG日志进行修正，以支持Peering期间的回滚操作。

# 3 PG 状态迁移详解

PG状态分为外部状态和内部状态，其中外部状态是可以直接被用户感知的。

>  PG外部状态表

|    PG状态    | 含义                                                         |
| :----------: | :----------------------------------------------------------- |
|  Activating  | Peering已经完成，PG正在等待所有PG实例同步并固化Peering结果（Info、Log） |
|    Active    | PG可以正常处理来自客户端的读写请求                           |
| Backfilling  | 见前面术语——Backfill部分                                     |
|    Clean     | PG当前不存在待修复的对象，Acting Set与Up Set一致，并且大小等于存储池副本数 |
|   Creating   | PG正在被创建                                                 |
|     Deep     | PG正在进行对象一致性扫描（总是与Scrubbing同时出现）          |
|  Scrubbing   | PG正在进行对象一致性扫描，但Scrubbing仅扫描元数据            |
|   Degraded   | Peering完成后，PG检测到任意一个PG实例存在不一致的对象；或者当前ActingSet小于存储池副本数。 |
|     Down     | Peering过程中，PG检测到某个Interval中，当前剩余的OSD不足以完成数据修复 |
|  Incomplete  | Peering过程中，由于：1）无法获得权威日志 2）通过`choose_acting`选出的Acting Set后续不足以完成数据修复（例如针对纠删码，存活的副本数小于k） |
| Inconsistent | PG通过Scrub检测到某些对象在PG实例间出现不一致（主要是因为静默错误） |
|    Peered    | 指Peering完成，但是PG当前的ActingSet小于存储池规定的最小副本数。 |
|  Recovering  | PG正在对不一致对象进行同步/修复。                            |
|   Remapped   | Peering完成，PG当前的Acting Set和Up Set出现不一致。          |
|    Repair    | PG在下一次执行Scrub的过程中，如果发现存在不一致的对象，并且能够进行修复，则自动进行修复。 |
|    Stale     | Monitor检测到当前Primary所在的OSD宕机；Primary超时未向Monitor上报心跳信息。 |
|  Undersized  | PG当前的Acting Set小于存储池副本数                           |

注意上述外部状态是可以叠加的。比如Active+clean表示一切正常。

## 3.1 状态机描述

[![img](https://durantthorvalds.top/img/PG_DFA.png)](https://durantthorvalds.top/img/PG_DFA.png)

## 3.2 具体流程分析

## 1 创建PG

OSDMonitor收到存储池创建命令之后，最终通过PGMonitor异步向每个OSD下发批量创建PG命令。创建PG是在Primary主导下进行的。Replica会在随后由Primary发起的Peering过程中自动被创建。

## 2 Peering

所有需要执行Peering 的PG也会专门安排一个peering_wq工作队列，当PG从peering_wq出列时：

1. 创建一个RecoveryCtx，用于批量处理所有PG与Peering相关的消息，例如Query(Log, Info等)，Notify等。
2. 逐个PG处理：取OSD最新的OSDMap，通过advance_pg检查PG是否需要执行OSDMap更新操作。如果为否，说明直接由Peering事件触发，将该事件从PG内部的peering_queue出列，投递到PG内部的状态机进行处理；如果为是，则在advance_pg内部执行OSDMap更新操作，完成之后再将PG再次加入peering_wq队列。
3. 检查是否需要通知Monitor设置本OSD的`up_thru`（我们规定PG在切换至新的Interval之后，成功完成Peering并重新开始接受客户端读写请求之前，必须先通知OSDMonitor设置其归属的up_thru参数）.
4. 批量派发RecoveryCtx中积累的Query\Notify消息。

下面是几个必须的操作，包括GetInfo, GetLog, GetMissing和Activate

GetInfo：获取PG元数据信息。

GetLog：开始着手进行日志同步。按照以下原则：

- 优先选取具有最新内容的日志（即Info中的`last_update`最大）；
- 如果有多份满足1）的日志，优先选择保存更多日志条目的日志，即Info中`log_tail`最小；
- 如果有多份满足2）的日志，优先选择当前的Primary。

GetMissing

Missing列表记录了自身所有需要通过Recovery进行修复的对象信息。只保留两个：

- `need`：对象被同步的目标版本号。
- `have`：对象当前归属PG实例的本地版本号。

当Primary收到每个Peer的本地日志之后，可以通过日志合并的方式得到每个Peer的missing列表，这一过程是通过解决日志分歧得到的。

为解决日志分歧，我们先将所有日志按照对象进行分类——即所有针对同一个对象操作的分歧日志都使用同一个队列进行管理，然后逐个队列进行管理。我们假定最老的那条分歧日志生效之前对应的版本号为prior_version，则针对每个队列的处理 都可以归结为以下五种情形：

- 本地存在比分歧日志更新的日志。
- 对象此前不存在。此时可以直接删除对象。
- 对象当前位于missing列表之中（例如上一次peering完成之后，Primary刚更新了自己的missing列表，但是其中的对象还没来得及修复，系统发生断电）。
- 对象不在missing列表之中同时所有分歧日志都可以回滚。此时将所有分歧日志按照从新到老的顺序依次进行回滚。
- 对象不在missing列表之中并且至少存在一条分歧日志不可回滚。此时将本地对象直接删除，将其加入missing列表，同时设置其need为prior_version，have为0.

Activate

在PG正式变为Active状态接受客户端请求之前，还必须先固化本次Peering的结果（也就是写入磁盘，开机bootstrap），遇到系统掉电时不会前功尽弃；同时需要初始化后续在后台执行的Recovery或者Backfill所依赖的元数据信息。以上过程便是Activate。

下面我们重点对两个元数据进行分析：

- `last_epoch_started`

它本来用于指示上一次peering成功时完成的epoch，但是因为peering涉及在多个osd之间进行数据和状态同步，所以存在进度不一致的可能。 为此我们设计两个`last_epoch_started`，一个用于标识每个参与本次Peering的PG实例本地Activate已经完成，直接作为本身Info的子属性存盘；另一个保存在Info的History属性下，由Primary在检测到所有副本的Activate过程都完成后统一更新和存盘。

- `MissingLoc`

在进行Recovery之前我们需要先引入一种同时包含所有missing条目和它们（目标版本）所在位置信息的全局数据结构，称为`MissingLoc`，它包含两个子表，分别是`needs_recovery_map`和`missing_loc`.它们分别保存当前PG的所有待修复对象，以及这些对象的目标版本可能同时存在于多个PG实例之上。因为目标版本可能位于多个PG实例之上，注意`missing_loc`不是一个PG而是一些PG的集合。后者由Primary统一生成。

生成`missing_loc`需要两步：首先，将所有的Peer missing列表之中的条目依次加入到needs_recovery_map之中；其次，以每个Peeri的Info和missing列表作为输入，针对`needs_recovery_map`中的每个对象逐一进行检查，以进一步确认其目标版本的位置信息并填充`missing_loc`.

成功生成`MissingLoc`之后，如果`needs_recovery_map`不为空，即存在任何需要被Recovery的对象，则Primary设置自身状态为**Degraded+Activating**；进一步，如果Primary检测到当前的Acting Set小于存储池副本数，则同时设置为**Undersized**状态。之后，Primary通过本地OS接口开始固化Peering结构；当Primary检测自身以及所有Peer的Activate操作都完成时，通过向状态机投递一个`AllReplicasActivated`事件来清除自身的Activating状态和Creating状态。同时检测PG此时Acting Set是否小于存储池最小副本数，如果小于，则设置PG状态为**Peered**并终止后续处理，否则将PG设置为Active，同时将之前来自客户端被阻塞的op重新入列。

随后PG进入**Active**状态，可以正常执行客户端的读写请求。

## 3 Recovery

Recovery是在Primary检测到自身或者任意一个peer存在待修复的对象进行的操作。为了防止集群中大量PG同时执行Recovery造成客户端响应速度过慢，需要限制Recovery。我们有几种配置项可供修改：

- `osd_max_bakfills`: 单个OSD运行同时执行Recovery或者Backfill的PG实例个数。 注意虽然单个PG的Recovery或者Backfill不能并发，但是不同PG的Recovery和Backfill可以并发。
- `osd_max_push_cost/osd_max_push_objects`:指示通过Push操作执行Recovery时，以OSD为单位，单个op所能携带的字节数，对象数。
- `osd_recovery_max_active`: 单个OSD允许同时执行Recovery的对象数。
- `osd_recovery_op_priority`: 指示Recovery op的默认优先级，它将与客户端op进行竞争，优先级设置越低，竞争劣势更大。
- `osd_recovery_sleep`: Recovery op每次在`op_shardedwq`中被处理前，设置此参数将导致对应的服务线程先休眠对应的时间。

不难看出，考虑到集群总IOPS和带宽有限，可以通过降低Recovery权重，或者通过QoS对Recovery总的IOPS和带宽加以限制，可以有效抑制Recovery对资源的消耗。

Recovery有以下两种方式：

1. `pull`: 指Primary自身存在待修复对象，由Primary按照`missing_loc`选择合适的副本去拉取待修复对象目标版本到本地，完成修复。
2. `push`: 指Primary感知到一个或者多个Replica当前存在待修复对象，主动推送每个待修复对象目标版本至相应的Replica，然后在本地完成修复。

Primary必须先完成自我修复才能修复别的Replica。它是基于日志进行的：

- 日志中的`last_requested`用于指示Recovery的起始版本号，在Activate中生成。因此我们首先将所有待修复对象按照日志版本号进行顺序排列，找到版本号不小于`last_requested`的第一个对象，记为v；
- 如果不为head对象，那么检查是否需要优先修复head对象后者snapdir对象；
- 根据具体的PGBackend生成一个Pull类型的op；
- 更新last_requested，使其指向v；
- 如果尚未达到单次最大修复数码，则从顺序队列中处理下一个待修复对象；否则返回。

以多副本为例子，因为PG日志中并未记录任何关于修改的详细信息，目前都是通过简单的全对象拷贝，因而效率低下，这也是Ceph为人所诟病的地方。

## 4 Backfill

Backfill的理论依据是“PG中所有对象可以基于全精度哈希排序”，所以是按照从小到大对当前对象进行遍历，并依次将它们按照全对象拷贝的方式写入待Backfill的PG实例。

当空的PG Temp在新的OSDMap生效之后，PG关联的Acting Set和Up Set重新变得一致，再次经过Peering之后，PG最终进入**Active+Clean**状态，此时PG一切恢复正常，可以删除不必要的副本（Stray）。

# 4 总结

PG的主要定位如下：

- 作为存储池的基本组成单位，负责执行存储池所绑定的副本策略。
- 以OSD作为单位，进行副本分布，将前端应用任何针对PG中原始对象的操作，转化为OSD所能理解的事务操作，并保证副本之间的强一致性。

不足之处在于：因为PG日志中并未记录任何关于修改的详细信息，目前都是通过简单的全对象拷贝，因而效率低下，这也是Ceph为人所诟病的地方。

------

# ——

# 实践部分

## 0 官方的指导和讲解

数据的持久性以及所有OSD之间的均匀分配都需要更多的放置组，但应将其数量减少到最少，以节省CPU和内存。

### 0.1 数据持久性

OSD发生故障后，数据丢失的风险会增加，直到完全恢复其中包含的数据为止。让我们想象一下在单个放置组中导致永久性数据丢失的情况：

- OSD失败，并且它包含的对象的所有副本均丢失。对于放置组中的所有对象，副本的数量突然从三个减少到两个。
- Ceph通过选择一个新的OSD来重新创建所有对象的第三个副本，从而开始对该放置组的恢复。
- 在同一放置组内的另一个OSD在新OSD完全填充第三份副本之前发生故障。某些对象将只有一个幸存副本。
- Ceph选择了另一个OSD并保持复制对象以恢复所需的副本数。
- 在同一放置组内的第三个OSD在恢复完成之前发生故障。如果此OSD包含对象的唯一剩余副本，则它将永久丢失。

在三个副本池中包含10个OSD和512个放置组的群集中，CRUSH将为每个放置组提供三个OSD。最后，每个OSD将托管（512 * 3）/ 10 =〜150个放置组。当第一个OSD发生故障时，以上情形将因此同时开始恢复所有150个放置组的操作。

恢复的150个放置组可能均匀分布在剩余的9个OSD上。因此，每个剩余的OSD都有可能将对象的副本发送给所有其他对象，并且还可能接收一些要存储的新对象，因为它们已成为新放置组的一部分。

完成恢复所需的时间完全取决于Ceph集群的架构。假设每个OSD由一台机器上的1TB SSD托管，并且所有OSD都连接到10Gb / s交换机，并且单个OSD的恢复将在M分钟内完成。如果每台计算机使用不带SSD日志的微调器和1Gb / s开关的两个OSD，则速度至少要慢一个数量级。

在这种大小的群集中，放置组的数量几乎对数据持久性没有影响。可能是128或8192，恢复速度不会变慢或变快。

**但是，将相同的Ceph群集增加到20个OSD而不是10个OSD可能会加快恢复速度，从而显着提高数据的持久性**。现在，每个OSD只能参与约75个放置组，而不是只有10个OSD时的约150个放置组，并且仍然需要全部19个剩余OSD执行相同数量的对象副本才能恢复。但是，如果10个OSD必须每个复制大约100GB，则现在它们必须每个复制50GB。如果网络是瓶颈，恢复将以两倍的速度进行。换句话说，当OSD数量增加时，恢复速度会更快。

如果该群集增长到40个OSD，则每个OSD将仅托管约35个放置组。如果OSD死亡，则恢复将保持更快的速度，除非它被另一个瓶颈阻止。但是，如果该群集增长到200个OSD，则每个OSD将仅托管约7个放置组。如果OSD死亡，则在这些放置组中最多将有约21（7 * 3）个OSD之间发生恢复：恢复将比有40个OSD时花费更长的时间，这意味着应该增加放置组的数量。

无论恢复时间有多短，第二个OSD在进行过程中都有可能发生故障。在上述10个OSD集群中，如果其中任何一个失败，则〜17个放置组（即，已恢复的〜150/9个放置组）将只有一个幸存副本。并且，如果剩余的8个OSD中的任何一个失败，则两个放置组的最后一个对象很可能会丢失（即，〜17/8个放置组，仅恢复了一个剩余副本）。

当群集的大小增加到20个OSD时，丢失三个OSD会损坏的放置组的数量会减少。第二个OSD丢失将降低〜4个（即，恢复到约75个/ 19个放置组），而不是〜17个，而第三个OSD丢失则仅在它是包含尚存副本的四个OSD之一时才丢失数据。换句话说，如果在恢复时间范围内丢失一个OSD的概率为0.0001％，则它从具有10个OSD的群集中的17 *10* 0.0001％变为具有20个OSD的群集中的4 *20* 0.0001％。

**简而言之，更多OSD意味着更快的恢复速度和更低的导致安置组的永久损失的级联故障风险。就数据持久性而言，在少于50个OSD的群集中，具有512或4096个放置组大致等效。**

注意：添加到群集中的新OSD可能需要很长时间才能分配有分配给它的放置组。但是，不会降低任何对象的质量，也不会影响群集中包含的数据的持久性。

### 0.2 池中的对象分布

理想情况下，对象在每个放置组中均匀分布。由于CRUSH计算每个对象的放置组，但实际上不知道该放置组内每个OSD中存储了多少数据，因此放置组数与OSD数之比可能会显着影响数据的分布。

例如，如果在三个副本池中有一个用于十个OSD的放置组，则仅使用三个OSD，因为CRUSH别无选择。当有更多的放置组可用时，对象更有可能在其中均匀分布。CRUSH还尽一切努力在所有现有的放置组中平均分配OSD。

只要放置组比OSD多一个或两个数量级，则分布应该均匀。例如，用于3个OSD的256个放置组，用于10个OSD的512或1024个放置组等。

数据分布不均可能是由OSD与放置组之间的比率以外的因素引起的。由于CRUSH没有考虑对象的大小，因此一些非常大的对象可能会造成不平衡。假设有100万个4K对象（共4GB）均匀分布在10个OSD的1024个放置组中。他们将在每个OSD上使用4GB / 10 = 400MB。如果将一个400MB对象添加到池中，则支持放置对象的放置组的三个OSD将填充400MB + 400MB = 800MB，而其余七个将仅占据400MB。

### 0.3 内存，CPU和网络使用率

对于每个放置组，OSD和MON始终需要内存，网络和CPU，并且在恢复期间甚至更多。通过对放置组内的对象进行聚类来共享此开销是它们存在的主要原因之一。

**最小化放置组的数量可以节省大量资源。**

### 0.4 选择放置组的数量

**如果您有超过50个OSD，我们建议每个OSD大约有50-100个放置组**，以平衡资源使用，数据持久性和分发。如果OSD少于50个，则最好在上述[预选](https://docs.ceph.com/en/latest/rados/operations/placement-groups/?#preselection)中进行[选择](https://docs.ceph.com/en/latest/rados/operations/placement-groups/?#preselection)。对于单个对象池，您可以使用以下公式获取基准

Total PGs=OSDs∗100pool_sizeTotal PGs=OSDs∗100pool_size

poolsizepoolsize在副本池表示副本数，而在纠删码池表示K+MK+M

然后，您应该检查结果是否与您设计Ceph集群的方式有意义，以最大程度地提高[数据持久性](https://docs.ceph.com/en/latest/rados/operations/placement-groups/?#data-durability)， [对象分配](https://docs.ceph.com/en/latest/rados/operations/placement-groups/?#object-distribution)并最小化[资源使用](https://docs.ceph.com/en/latest/rados/operations/placement-groups/?#resource-usage)。

结果应始终**四舍五入到最接近的2的幂**。

只有2的幂可以平衡放置组中的对象数量。其他值将导致OSD上的数据分布不均。它们的使用应仅限于从两个方的一种逐步增加到另一种。

例如，对于具有200个OSD和3个副本的池大小的群集，您可以如下估算PG的数量

200×1003=6667200×1003=6667

最近的2的次幂是8192.

当使用多个数据池存储对象时，您需要确保在每个池的放置组数量与每个OSD的放置组数量之间取得平衡，以便获得合理的放置组总数，从而使每个OSD的方差很小而不会增加系统资源的负担或使对等进程太慢。

例如，一个由10个池组成的群集，每个池在10个OSD上具有512个放置组，则总共有5120个放置组分布在10个OSD上，即每个OSD 512个放置组。那不会使用太多资源。但是，如果创建了1,000个池，每个池有512个放置组，则OSD将分别处理约50,000个放置组，并且将需要更多的资源和时间来进行对等。

您可能会发现[PGCalc](http://ceph.com/pgcalc/)工具很有帮助。这是一个很有意思的工具。

> **建议的PG计数背后的逻辑**
>
> ( Target PGs per OSD ) x ( OSD # ) x ( %Data )/ ( Size )
>
> 1. 如果以上计算的值小于**（OSD＃）/（Size）**的值，则该值将更新为**（OSD＃）/（Size）的值**。这是通过为每个池向每个OSD分配至少一个主PG或辅助PG来确保均匀的负载/数据分配。
> 2. 然后将输出值舍入到**最接近的2的幂**。
>    **提示：**最接近的2的幂提供了[CRUSH](http://ceph.com/docs/master/rados/operations/crush-map/)算法效率的少量提高。
> 3. 如果最接近的2的幂比原始值低**25％**以上，则使用下一个更高的2的幂。
>
> **目的**
>
> - 此计算的目的和上面“关键”部分所述的目标范围是为了确保有足够的放置组，以便在整个群集中进行均匀的数据分布，同时每个OSD PG的PG值不够高，从而在恢复期间引起问题和/或回填操作。
>
> **无效或无效池的影响：**
>
> - 空池或其他非活动池不应被认为有助于整个群集中的数据均匀分布。
> - 但是，与这些空/非活动池关联的PG仍会消耗内存和CPU开销。

------

## 1 设置放置组数

要设置池中的放置组数量，必须在创建池时指定放置组的数量。有关详细信息，请参见[创建池](https://docs.ceph.com/en/latest/rados/operations/pools#createpool)。即使在创建池之后，您也可以使用以下方法更改放置组的数量：

```
ceph osd pool set {pool-name} pg_num {pg_num}Copy
```

增加放置组的数量之后，还必须增加放置（`pgp_num`）的放置组的数量，群集才能重新平衡。该`pgp_num`会是将由CRUSH算法可考虑放置位置的组数。增加会`pg_num`拆分放置组，但数据将不会迁移到较新的放置组，直到放置的放置组，即`pgp_num`增加的值`pgp_num` 应等于`pg_num`。要增加用于放置的放置组的数量，请执行以下操作：

```
ceph osd pool set {pool-name} pgp_num {pgp_num}Copy
```

减少PG数量时，`pgp_num`将自动为您调整。

笔者注：

关于PG和PGP：

- PG =放置组( Placement Group)
  PGP =用于放置的放置组(Placement Group for Placement purpose)

  pg_num = 映射到OSD的PG的数量，它必须是2的幂次。

  当对任何一个池增加pg_num时，该池的每个PG都会**分裂**成一半，但它们都将始终映射到其父OSD。

  在此之前，Ceph不会开始重新平衡。 现在，当您为同一池增加pgp_num值时，PG开始从父级迁移到其他OSD，并且群集重新平衡开始。 这就是PGP扮演重要角色的方式。

## 2 获取放置组的数量

要获取池中的放置组数，请执行以下操作：

```
ceph osd pool get {pool-name} pg_numCopy
```

## 3 Auto_scaling

### 自动调整

这是一种自动调整PG的方式。有三个参数`off`, `on`,`warn`. 当设置为off就需要人为控制PG数目。

要为现有池设置自动缩放模式，请执行以下操作：

```
ceph osd pool set <pool-name> pg_autoscale_mode <mode>Copy
```

例如，要在pool上启用自动缩放`foo`，请执行以下操作：

```
ceph osd pool set foo pg_autoscale_mode onCopy
```

您还可以使用以下命令配置`pg_autoscale_mode`应用于以后创建的任何池的默认值：

```
ceph config set global osd_pool_default_pg_autoscale_mode <mode>Copy
```

## 4 Autoscale_status

### 查看PG缩放建议

您可以使用以下命令查看每个池，池的相对利用率以及对PG计数的任何建议更改：

```
ceph osd pool autoscale-statusCopy
```

输出将类似于：

```
POOL    SIZE  TARGET SIZE  RATE  RAW CAPACITY   RATIO  TARGET RATIO  EFFECTIVE RATIO PG_NUM  NEW PG_NUM  AUTOSCALE
a     12900M                3.0        82431M  0.4695                                     8         128  warn
c         0                 3.0        82431M  0.0000        0.2000           0.9884      1          64  warn
b         0        953.6M   3.0        82431M  0.0347                                     8              warnCopy
```

**SIZE**是存储在池中的数据量。**TARGET SIZE**（如果存在）是管理员指定的数据量，管理员希望最终将其存储在此池中。系统使用两个值中的较大者进行计算。

**RATE**是池的乘数，它确定要消耗多少原始存储容量。例如，3个副本池的比率为3.0，而k = 4，m = 2纠删码池的比率为1.5。

**RAW CAPACITY**是OSD上负责存储此池（可能还有其他池）数据的原始存储容量的总量。 **比率**是该池消耗的总容量的比率（即比率=大小*比率/原始容量）。

**TARGET RATIO**（如果存在）是管理员已指定他们希望该池相对于设置了目标比率的其他池消耗的存储比率。如果同时指定了目标大小字节和比率，则比率优先。

**EFFECTIVE RATIO**是通过两种方式进行调整后的目标比率：

1. 减去设置了目标大小的池预期使用的任何容量
2. 设定目标比率后，对池中的目标比率进行标准化，以便它们共同针对其余空间。例如，target_ratio 1.0的4个池的有效比率为0.25。

系统使用实际比率和有效比率中的较大者进行计算。

**PG_NUM**是该池的当前PG数量（如果`pg_num` 正在进行更改，则为该池正在使用的PG的当前数量）。 系统认为应该将`pg_num`更改为**NEW PG_NUM**。它始终是2的幂，并且仅在“理想”值与当前值的差异大于3时才存在。

最后一列，**AUTOSCALE**，是池`pg_autoscale_mode`，并将于要么`on`，`off`或`warn`。

## 5 Automated_scaling

### 自动缩放

最简单的方法是允许群集根据使用情况自动扩展PG。Ceph将查看整个系统的PG的总可用存储量和目标数量，查看每个池中存储了多少数据，并尝试相应地分配PG。该系统的方法相对保守，仅当当前PG（`pg_num`）数量比其认为的数量多3倍时才对池进行更改。

每个OSD的PG的目标数量基于可 `mon_target_pg_per_osd`配置（默认值：100），可以通过以下方式进行调整：

```
ceph config set global mon_target_pg_per_osd 100Copy
```

自动缩放器将分析池并在每个子树的基础上进行调整。因为每个池可能映射到不同的CRUSH规则，并且每个规则可能在不同的设备之间分配数据，所以Ceph将考虑独立使用层次结构的每个子树。例如，映射到ssd类的OSD的池和映射到hdd类的OSD的池将分别具有最佳PG计数，这取决于这些相应设备类型的数量。

## 6 指定期望池大小

首次创建集群或池时，它将消耗集群总容量的一小部分，并且在系统中似乎只需要少量的放置组。但是，在大多数情况下，群集管理员会很好地知道哪些池会随着时间消耗掉大部分系统容量。通过将此信息提供给Ceph，可以从一开始就使用更合适数量的PG，从而避免进行后续调整 `pg_num`以及在进行这些调整时与移动数据相关的开销。

池的*目标大小*可以通过两种方式指定：要么以池的绝对大小（即字节）为单位，要么以相对于具有一`target_size_ratio`组其他池的权重为单位。

例如，：

```
ceph osd pool set mypool target_size_bytes 100TCopy
```

会告诉系统mypool预计会占用100 TiB的空间。或者，：

```
ceph osd pool set mypool target_size_ratio 1.0Copy
```

会告诉系统mypool与`target_size_ratio`set的其他池相比预期消耗1.0 。如果mypool是群集中唯一的池，则意味着预期使用了总容量的100％。如果第二个池的`target_size_ratio` 1.0，则两个池都将使用50％的群集容量。

您还可以在创建时使用命令的可选参数`--target-size-bytes <bytes>`或参数`--target-size-ratio <ratio>`设置池的目标大小。

请注意，如果指定了不可能的目标大小值（例如，容量大于整个群集的容量），则会发出健康警告（`POOL_TARGET_SIZE_BYTES_OVERCOMMITTED`）。

如果为池指定了`target_size_ratio`和`target_size_bytes`，则仅考虑比率，并发出运行状况警告（`POOL_HAS_TARGET_SIZE_BYTES_AND_RATIO`）。

## 7 设置PG的下界

也可以为一个池指定最小数量的PG。这对于确定执行IO时客户端将看到的并行度的数量的下限很有用，即使池中大多数都是空的。设置下限可以防止Ceph将PG编号减少（或建议减少）到配置的编号以下。

您可以使用以下方法设置池的最小PG数量：

```
ceph osd pool set <pool-name> pg_num_min <num>Copy
```

您还可以使用命令的可选参数在创建池时指定最小PG计数。`--pg-num-min <num>``ceph osd pool create`

## 8 获取集群的PG统计信息

要获取集群中放置组的统计信息，请执行以下操作：

```
ceph pg dump [--format {format}]Copy
```

有效格式为`plain`（默认）和`json`。

## 9 获取卡住的PG的统计信息

要获取处于指定状态的所有放置组的统计信息，请执行以下操作：

```
ceph pg dump_stuck inactive|unclean|stale|undersized|degraded [--format <format>] [-t|--threshold <seconds>]Copy
```

**inactive**放置组无法处理读写，因为它们正在等待OSD包含最新数据。

**unclean**放置组包含未复制所需次数的对象。他们应该正在恢复。

**stale**放置组处于未知状态-承载它们的OSD暂时未向监视集群报告（由配置`mon_osd_report_timeout`）。

有效格式为`plain`（默认）和`json`。阈值定义了放置组停留在返回的统计信息中之前所停留的最小秒数（默认为300秒）。

## 10 获取PG Map

要获取特定放置组的放置组映射，请执行以下操作：

```
ceph pg map {pg-id}Copy
```

例如：

```
ceph pg map 1.6cCopy
```

Ceph将返回放置组图，放置组和OSD状态：

```
osdmap e13 pg 1.6c (1.6c) -> up [1,0] acting [1,0]Copy
```

解释一下，这里表示 pg 1.6c被映射到 编号为 [1,0]的两个OSD上。 Acting 表示Acting Set。

## 11 获取PG统计信息

要检索特定放置组的统计信息，请执行以下操作：

```
ceph pg {pg-id} queryCopy
```

这里的query其实是一种元数据信息，部分形式如下：

```
{
	"snap_trimq": "[]",
    "snap_trimq_len": 0,
    "state": "active+clean",
    "epoch": 236,
    "up": [
        1,
        2,
        0
    ],
    "acting": [
        1,
        2,
        0
    ],
    "acting_recovery_backfill": [
        "0",
        "1",
        "2"
    ],
    "info": {
        "pgid": "1.0",
        "last_update": "223'23",
        "last_complete": "223'23",
        "log_tail": "0'0",
        "last_user_version": 23,
        "last_backfill": "MAX",
        "purged_snaps": [],
        "history": {
            "epoch_created": 2,
            "epoch_pool_created": 2,
            "last_epoch_started": 221,
            "last_interval_started": 220,
            "last_epoch_clean": 221,
            "last_interval_clean": 220,
            "last_epoch_split": 0,
            "last_epoch_marked_full": 0,
            "same_up_since": 220,
            "same_interval_since": 220,
            "same_primary_since": 212,
            "last_scrub": "189'21",
            "last_scrub_stamp": "2020-12-14T06:56:57.181447+0800",
            "last_deep_scrub": "189'20",
            "last_deep_scrub_stamp": "2020-12-13T04:09:17.431508+0800",
            "last_clean_scrub_stamp": "2020-12-14T06:56:57.181447+0800",
            "prior_readable_until_ub": 0
        },
...
}Copy
```

我们可以看到很多理论部分讲过的元数据，比如 up 、acting、info、epoch、peer、interval等。`snap_trimq`表示快照删除队列。

当前的版本号为236.

## 12 Scrub一个放置组

关于Scrub的含义，我们在《Ceph纠删码部署》已经介绍了，Scrub指数据扫描，通过读取对象数据并重新计算校验和，再与之前存储在对象属性的校验和进行比对，以判断有无静默错误（磁盘自身无法感知的错误）。要Scrub，请执行以下操作：

```
ceph pg scrub {pg-id}Copy
```

Ceph检查主节点和任何副本节点，生成放置组中所有对象的目录并进行比较，以确保没有丢失或不匹配的对象，并且它们的内容一致。假设所有副本都匹配，则最终的语义扫描可确保所有与快照相关的对象元数据都是一致的。通过日志报告错误。

要从特定池中清理所有放置组，请执行以下操作：

```
ceph osd pool scrub {pool-name}Copy
```

## 12 设置PG Backfill/Recovery的优先级

请注意，这些命令可能会破坏Ceph内部优先级计算的顺序，因此请谨慎使用！特别是，如果您有多个当前共享相同底层OSD的池，并且某些特定的池比其他池更重要，则建议您使用以下命令以更好的顺序重新排列所有池的恢复/回填优先级：

```
ceph osd pool set {pool-name} recovery_priority {value}Copy
```

例如，如果您有10个池，则可以将最重要的一个优先级设置为10，下一个9，等等。或者您可以不理会大多数池，而说3个重要的池分别设置为优先级1或优先级3、2、1。

在恢复或者回填比用户op的优先级更高的时候。我们可以执行：

```
ceph pg force-recovery {pg-id} [{pg-id #2}] [{pg-id #3} ...]
ceph pg force-backfill {pg-id} [{pg-id #2}] [{pg-id #3} ...]Copy
```

如果您认为这是一个不好的决定，请使用：

```
ceph pg cancel-force-recovery {pg-id} [{pg-id #2}] [{pg-id #3} ...]
ceph pg cancel-force-backfill {pg-id} [{pg-id #2}] [{pg-id #3} ...]Copy
```

这将从这些PG中删除“ force”标志，并将以默认顺序对其进行处理。同样，这不会影响当前正在处理的放置组，只会影响仍在排队的放置组。

恢复或回填组后，将自动清除“ force”标志。

同样，您可以使用以下命令强制Ceph首先对指定池中的所有放置组执行恢复或回填：

```
ceph osd pool force-recovery {pool-name}
ceph osd pool force-backfill {pool-name}Copy
```

要么：

```
ceph osd pool cancel-force-recovery {pool-name}
ceph osd pool cancel-force-backfill {pool-name}Copy
```

如果您改变主意，则可以恢复到默认的恢复或回填优先级。

## 13 还原丢失

如果群集丢失了一个或多个对象，并且您决定放弃对丢失数据的搜索，则必须将未找到的对象标记为`lost`。

如果已查询所有可能的位置并且仍然丢失了对象，则可能必须放弃丢失的对象。鉴于异常的异常组合使集群能够了解恢复写本身之前执行的写，这是可能的。

当前唯一受支持的选项是“还原”，它可以回滚到该对象的先前版本，或者（如果是新对象）则完全忘记它。要将“未找到”的对象标记为“丢失”，请执行以下操作：

```
ceph pg {pg-id} mark_unfound_lost revert|delete
```

