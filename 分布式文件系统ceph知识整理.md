# 1简介

 Ceph是加州大学Santa Cruz分校的Sage Weil（ DreamHost的联合创始人） 专为博士论文设计的新一代自由软件分布式文件系统。
 2004年， Ceph项目开始， 提交了第一行代码。
 2006年， OSDI学术会议上， Sage发表了介绍Ceph的论文， 并在该篇论文的末尾提供了Ceph项目的下载链接。
 2010年， Linus Torvalds将CephClient合并到内核2.6.34中， 使Linux与Ceph磨合度更高。
 2012年， 拥抱OpenStack， 进入Cinder项目， 成为重要的存储驱动。
 2014年， Ceph正赶上OpenStack大热， 吸引来自不同厂商越来越多的开发者加入， Intel、 SanDisk等公司都参与其中， 同时Inktank公司被Red Hat公司1.75亿美元收购。
 2015年， Red Hat宣布成立Ceph顾问委员会， 成员包括Canonical、 CERN、 Cisco、 Fujitsu、Intel、 SanDisk和SUSE。 Ceph顾问委员会将负责Ceph软件定义存储项目的广泛议题， 目标是使Ceph成为云存储系统。
 2016年， OpenStack社区调查报告公布， Ceph仍为存储首选， 这已经是Ceph第5次位居调查的首位了。

![image](https://user-images.githubusercontent.com/87457873/153867063-7c2896c3-3687-4b85-bed9-efe1dc168c52.png)

目前，ceph FS 可以作为Hadoop后端数据存储池，可代替HDFS的存储方案。
 Ceph与KVM虚拟化结合，ceph块存储RBD可作为KVM虚拟化的后端存储。
 ceph块存储RBD可作为openstack的后端存储。
 Ceph的RGW与OwnCloud搭配，搭建本地网盘。

特别说明：Ceph支持三种调用接口：对象存储，块存储，文件系统挂载。
 后面主要讲解对象存储模块相关。

# 2ceph架构图

Ceph基本组成结构：

![image](https://user-images.githubusercontent.com/87457873/153867086-e085a48b-af90-42f0-98df-111b1f12af04.png)

更详细的架构图如下：

![image](https://user-images.githubusercontent.com/87457873/153867111-5f9b4a1d-91b2-49e6-9c45-9db0585ee8e9.png)

Ceph 分上下两层， 下面为ceph RADOS Cluster 集群。上面为对外提供的应用层。
 **RADOS Cluster**是分布式文件存储。
 **LIBRADOS** 是调用库，允许应用程序访问RADOS Cluster。
 **RGW** 是一套基于当前流行的RESTful协议网关，兼容S3和SWIFT。
 **RBD** 通过Linux内核（Kernel）客户端和QEMU/KVM驱动，提供一个完全分布式的块存储。
 **CEPH FS** 通过Linux内核（Kernel）客户端结合FUSE，来提供一个兼容POSIX的文件系统。

# 3 Rados Cluster

首先介绍ceph的后端存储：Rados Cluste。Rados Cluster属于存储层级， 没有REST、用户、桶的概念。

## 3.1 Rados Cluster基本组件

![image](https://user-images.githubusercontent.com/87457873/153867142-f24482f5-3608-4aba-acd5-5cbfaab389e8.png)

一个ceph集群组件必有：MON、OSD组件， 可选MDS（供ceph FS使用）。
 **Osd ：**用于集群中所有数据与对象的存储。处理集群数据的复制、恢复、回填、再均衡。并向其他osd守护进程发送心跳，然后向Mon提供一些监控信息。当Ceph存储集群设定数据有两个副本时（一共存两份），则至少需要两个OSD守护进程即两个OSD节点，集群才能达到active+clean状态。
 **Monitor ：**监控整个集群的状态，维护集群的cluster MAP二进制表，保证集群数据的一致性。Monitor map包含以下map：OSD MAP、PG MAP、MDS MAP和CRUSH MAP等。
 **MON ：**服务利用Paxos实例，把每个映射图存储为一个文件， MON高可用集群实现一般为3台。
 **MDS(可选) ：**为Ceph文件系统提供元数据计算、缓存与同步。在ceph中，元数据也是存储在osd节点中的，mds类似于元数据的代理缓存服务器。MDS进程并不是必须的进程，只有需要使用CEPHFS时，才需要配置MDS节点。

## 3.2 Rados Cluster I/O流

数据分布是分布式存储系统的一个重要部分。
 数据分布算法目前主流有两种:一致性HASH和ceph的CRUSH算法。 在Aamzon的Dyanmo键值系统中采用一致性HASH算法，openstack的swift也使用一致性HASH算法，在ceph中使用CRUSH算法。

### 3.2.1 crush算法简介

CRUSH（controlled replication under scalable hashing）是一种基于伪随机控制数据分布、复制算法。它是一种伪随机的算法，在相同的环境下，相似的输入得到的结果之间没有相关性，相同的输入得到的结果是确定的。它只需要一个集群的描述地图和一些规则就可以根据一个整型的输入得到存放数据的一个设备列表。
 Crush同时支持多种数据备份策略，典型如镜像、RAID及其衍生的纠错码等，并受控地将数据的多个备份映射到集群不同物理区域中的底层存储设备之上，从而保证数据可靠性。

### 3.2.2 CEPH I/O流

Rados Cluster中，monitor组件存储了以下map： OSD MAP、 pg map、mds map、crush map、mds map、以及monitoer map。

![image](https://user-images.githubusercontent.com/87457873/153867166-81246583-b661-45b9-9d80-0328f02a78ed.png)


 （1） 将文件切割为按照一个特定的size（ceph系统默认是4M）被切分成若干个对象。（这里的对象时RADOS层的对象， 不是RGW中的对象）。
 （2） 通过对Oid进行Hash， 可以得到对应PG_ID，(poolid, hash(oid) & mask)
 （3） 通过CRUSH算法得到一个OSD列表（OSD1，OSD2，OSD3），Crush(pgid)->（osd1,osd2…）
 （4） 获取到osd后，应用层客户端直接向OSD（primary osd）发起I/O请求。
 所以一句话说明CRUSH的作用就是，根据pgid得到一个OSD列表。

### 3.3 Rados Cluster 部署实例

下图为雅虎利用ceph构建存储的服务器部署图：

![image](https://user-images.githubusercontent.com/87457873/153867207-7f6c8594-923b-4820-95fb-c8c7e40edf8c.png)

# 4 CEPH-RGW

对象存储是一种新型的存储形态，通常情况提供HTTP访问接口。从狭义上讲，对象存储即云存储。

## 4.1 RGW架构

Ceph的核心模块RADOS是一个基于对象的分布式存储系统（注意：这里的对象时指RADOS内部的一种数据存储单元，与对象存储中的对象概念有区别），通常情况下应用通过RADOS抽象库librados提供的接口访问RADOS集群，但是librados只提供私有接口，并不提供HTTP协议访问，Ceph为了支持通用的HTTP接口设计了RGW（RADOS GateWay，即对象存储网关）。

![image](https://user-images.githubusercontent.com/87457873/153867219-638877d9-72c8-4e49-92eb-9c71bf89a227.png)

## 4.2 数据组织和存储

通常情况，一个对象存储（RGW）的实体包含用户、存储桶和对象，三者之间是一种层级关系。

![image](https://user-images.githubusercontent.com/87457873/153867254-1f143dab-e23d-4d54-a586-e87f8f708f35.png)

（1） 用户管理
 用户管理设计包含以下：用户认证信息、访问控制权限信息和配额信息。
 S3用户认证流程请参考《对象存储概述》文档。
 RGW将用户信息保存在RADOS对象的数据部分，一个用户对应一个RADOS对象。该对象用用户ID命名。
 （2） 存储桶信息
 一个存储桶对应一个RADOS对象。一个存储桶包含的信息分两类，一类是对RGW网关透明的信息，例如：用户自定义的元数据 type:音频文件；一类是RGW网关关注的信息，这类信息包含存储桶中对象的存储策略、存储桶中索引对象的数目以及应用与索引对象的映射关系、存储桶配额等，此类信息由RGWBucketInfo管理。
 （3 ）对象
 应用上传的对象包含数据和元数据两部分，数据部分保存在一个活多个RADOS对象的数据部分，元数据保存在其中一个RADOS对象的扩展属性中。RGW对单个对象提供了两种上传接口: 整体上传和分段上传。这里不详细介绍。

## 4.3 功能实现

RGW近几年一直在不停对齐S3 API
 基本功能包含用户、存储桶、对象的增删改查。
 引申的功能有存储桶和对象的访问控制功能，用户认证功能，桶配额功能等。
 访问控制策略如下：

![image](https://user-images.githubusercontent.com/87457873/153867286-c89d648f-a08b-468b-b958-76dc299eba45.png)

# 5CEPH-RBD

   RBD是CEPH对外三大存储服务组件之一，也是当前CEPH最稳定、应用最广泛的存储接口。上传应用访问RBD块存储有两种途径：librdb和krdb。其中librdb是一个基于librados的用户态接口库，而krbd是集成在GNU/LINUX内核中的一个内核模块，通过用户态的rbd命令行工具，可以将RBD块存储映射为本地的一个块设备文件。

## 5.1 RBD架构

   RBD架构：

![image](https://user-images.githubusercontent.com/87457873/153867330-8428148b-0151-4bbd-a8a8-19bc5c200c76.png)

# 6 CEPH-FS

   Ceph FS是为了接管传统存储系统产生的。
 Ceph FS的MDS是基于动态子树分区法实现，这种算法是一种高效的元数据组织和索引方式。
 MDS的基于动态子树分区法和CRUSH算法并称Ceph的两大核心。

## 6.1 Ceph FS架构

![image](https://user-images.githubusercontent.com/87457873/153867353-8f7ef5ec-399a-4bbe-861e-7371d7d7f49e.png)

（1） CephFs kernel object
 为内核态接口，3.10以后的内核版本默认支持，它通过mount –t ceph将cephFs挂载到操作系统指定目录下。
 （2） CephFs FUSE
 指基于FUSE（Filesystem in Userspace， 即用户空间文件系统。Linux 在2.6.14内核增加FUSE模块）的用户态接口，通过 ceph-fuse命令将CephFs挂载到操作系统指定目录。
 （3） User Space Client
 为直接通过客户端应用程序调用CephFs提供的文件系统接口，比如Hadoop调用CephFs提供的java文件系统接口实现文件存储。

# 7     Ceph与K8s

Kubernetes支持Ceph的块存储（Ceph RDB)和文件存储（CephFS）作为Kubernetes的持久存储后端。
 k8s提供了非常丰富的组件：volume，Persistent Volumes，Dynamic Volume Provisioning。对于volume，如果K8s pod挂掉，对应的volume也就消失了，所有后面主要介绍Persistent Volumes和Dynamic Volume Provisioning利用ceph创建。

## 7.1 RBD与K8S

参考资料：https://zhangchenchen.github.io/2017/11/17/kubernetes-integrate-with-ceph/
 Persistent Volumes
 （1） 在ceph中创建一个2G的image。
 创建image过程中有个坑：在jewel版本下默认format是2，开启了rbd的一些属性，而这些属性有的内核版本是不支持的，会导致map不到device的情况，可以在创建时指定feature（我们就是这样做的）,也可以在ceph配置文件中关闭这些新属性：rbd_default_features = 2。参考rbd无法map(rbd feature disable)。
 需要指定feature：rbd create test-image -s 2G --image-feature  layering
 （2） k8s在image上创建pv和pvc，并创建pod使用Pvc。

Dynamic Volume Provisioning
 （1） 创建一个storageclass
 （2） 创建一个PVC，指定storageclass
 （3） 创建 rbd-provisioner
 （注意：如果直接创建pod挂载的话会报错如下：
 Error creating rbd image: executable file not found in $PATH
 原因：这是因为我们的k8s集群是使用kubeadm创建的，k8s的几个服务也是跑在集群静态pod中，而kube-controller-manager组件会调用rbd的api，但是因为它的pod中没有安装rbd，所以会报错，如果是直接安装在物理机中，因为我们已经安装了ceph-common，所以不会出现这个问题。）
 （4） 就可以创建pod使用pvc了。

## 7.2 cephFs与K8s

参考资料：https://tonybai.com/2017/05/08/mount-cephfs-acrossing-nodes-in-kubernetes-cluster/
 CephRBD的问题：在不同node上多个Pod是无法以ReadWriteMany模式挂载同一个CephRBD的，在一个node上多个Pod是可以以ReadWriteMany模式挂载同一个CephRBD的。
 CephFs: 不同节点可以挂载同一CephFS。

CephFs挂载方式：
 在K8s中，至少可以通过两种方式挂载CephFS，一种是通过Pod直接挂载；另外一种则是通过pv和pvc挂载，此方法与挂载RBD类似。
