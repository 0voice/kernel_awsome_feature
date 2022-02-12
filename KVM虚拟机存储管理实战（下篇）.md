在上一篇中我们介绍qemu-img这个对磁盘镜像操作的命令，包括检查镜像磁盘，创建镜像磁盘、查看镜像磁盘信息、转换镜像磁盘格式、调整镜像磁盘大小以及镜像磁盘的快照链操作。但是，在镜像磁盘快照链操作中，我们也提到了通过qemu-img命令创建快照链只能在虚拟机关机状态下运行。如果虚拟机为运行态，只能通过virsh save vm来保存当前状态。那么，本文就专门讲述在KVM虚拟机中如何通过virsh命令来创建快照链，以及链中快照的相互关系，如何缩短链，如何利用这条链回滚我们的虚拟机到某个状态等等。

## **什么是虚拟机快照链**

虚拟机快照保存了虚拟机在某个指定时间点的状态（包括操作系统和所有的程序），利用快照，我们可以恢复虚拟机到某个以前的状态。比如：测试软件的时候经常需要回滚系统，以及新手安装OpenStack时为了防止重头再来，每安装成功一个服务就做一次快照等等。

**快照链就是多个快照组成的关系链**，这些快照按照创建时间排列成链，像下面这样。

复制

base-image<--guest1<--snap1<--snap2<--snap3<--snap4<--当前(active)

如上，base-image是制作好的一个qcow2格式的磁盘镜像文件，它包含有完整的OS以及引导程序。现在，以这个base-image为模板创建多个虚拟机，简单点方法就是每创建一个虚拟机我们就把这个镜像完整复制一份，但这种做法效率底下，满足不了生产需要。这时，就用到了qcow2镜像的**copy-on-write（写时复制）特性。**

qcow2(qemu copy-on-write)格式镜像支持快照，具有创建一个base-image，以及在base-image(backing file)基础上创建多个**copy-on-write overlays镜像**的能力。这里需要解释下backing file和overlay的概念。在上面那条链中，我们为base-image创建一个guest1，那么此时base-image就是guest1的backing file，guest1就是base-image的overlay。同理，为guest1虚拟机创建了一个快照snap1，此时guest1就是snap1的backing file，snap1是guest1的overlay。

**backing files和overlays十分有用，可以快速的创建瘦装备实例，特别是在开发测试过程中可以快速回滚到之前某个状态。**以CentOS系统来说，我们制作了一个qcow2格式的虚拟机镜像，想要以它作为模板来创建多个虚拟机实例，有两种方法实现：

**1）每新建一个实例，把centosbase模板复制一份，创建速度慢。说白了就是复制原始虚拟机。**

**2）使用copy-on-write技术(qcow2格式的特性)，创建基于模板的实例，创建速度很快，可以通过查看磁盘文件信息，进行大小比较。也就是我们将存储虚拟化特性中提到的链接克隆。**

如下，我们有一个centosbase的原始镜像(包含完整OS和引导程序)，现在用它作为模板创建多个虚拟机，每个虚拟机都可以创建多个快照组成快照链，但是不能直接为centosbase创建快照。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666804604-f76ce390-c612-452a-8a95-43c129595546.jpeg)

上图中centos1，centos2，centos3等都是基于centosbase模板创建的虚拟机(guest)，接下来做的测试需要用到centos1_sn1、centos1_sn2、centos1_sn3等centos1的快照链实现。也就是说，我们可以只用一个backing files创建多个虚拟机实例(overlays)，然后可以对每个虚拟机实例做多个快照。**这里需要注意：backing files总是只读的文件。换言之，一旦新快照被创建，他的后端文件就不能更改(快照依赖于后端这种状态)。**

## **virsh命令实现KVM虚拟机的内置快照**

qemu/kvm有三种快照，分别是**内部（保存在硬盘镜像中）/外部（保存为另外的镜像名）/虚拟机状态** ，很多网站上提供的资料和教程也大多是内部快照功能。**内部快照不支持raw格式的镜像文件，所以如果想要使作内部快照，需要先将镜像文件转换成qcow2格式。**

内置快照在虚拟机运行状态和关闭状态都可以创建。在关机状态下，它通过单个qcow2镜像磁盘存储快照时刻的磁盘状态，并没有新磁盘文件产生。在虚机开机状态下，可以同时保存内存状态，设备状态和磁盘状态到一个指定文件中。当需要还原虚拟机状态时，将虚机关机后通过virsh restore命令还原回去。以上这些就是虚拟机的内置快照链操作，一般用于测试场景中的不断的将vm还原到某个起点，然后重新开始部署和测试。下面，我们就来玩玩KVM虚拟机的内置快照链。

**Stpe1：**首先，我们找一台运行的虚拟机，然后通过控制台console登录，建一个空目录flags并写一个测试文件test01作为标记。在后面快照回滚时，可以通过对比该文件查看具体的效果 。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666804365-d1b094b0-9464-4aee-a125-a75ad004fa92.jpeg)

**Step2：**由于内部快照不支持raw格式的磁盘镜像文件，所以我们首先需要查看下当前虚拟机的磁盘镜像格式，如果不符合要求，需要利用qemu-img命令进行磁盘格式转换。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666804200-8427ab7f-a0c3-4de6-971b-7dc65309282c.jpeg)

**Step3：**使用snapshot-create-as命令创建第一个快照snap01。其实，还有个类似命令的snapshot-create，它创建的快照名称是系统随机生成的，一般我们使用snap-create-as命令指定快照名创建。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666805301-0839eaec-faf4-48b1-91ee-31e3f63c9b99.jpeg)

**step4：**使用snapshot-current命令查看当前快照的详细信息。这里，其实查看的是/var/libvirt/qemu/snapshot/centos7.5/下的虚拟机的快照xml配置文件信息。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666804207-f37940a0-e226-4497-9d12-a21e6cd947c7.jpeg)

**step5：**此时，我们再次在虚拟机创建测试内容，在test01文件追加内容456，然后再次创建快照snap02。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666804932-7e1cbd39-0ae9-4313-ad3d-76535d82dfa5.jpeg)

同时，我们利用qemu-img命令查看虚拟机的磁盘信息，发现创建的快照后磁盘的容量有所增加，这也符合常理。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666804958-89b0a3ed-082c-4cce-9501-32715574d96b.jpeg)

**step6：**此时，我们利用快照回滚虚拟机。在回滚之前最好先关闭虚拟机 virsh shutdown 或virsh destroy ，在不关闭的情况下也可以做回滚。但是，此时如有新数据写入时不过会出现问题。所以，在实际运维中，还是建议先停机再做回滚。如果真的要求不停机回滚，最好加上–force选项表示强制回滚，此时即使有数据写入也会被丢弃。不加–force选项时，老版本的libvirt会报错，但是新版本不会报错，不过不建议这样使用。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666805041-47f8f815-ac1b-4096-b76a-22c651641d2c.jpeg)

此时，我们在虚拟机运行态下，再次利用快照snap02进行回滚。而且，我们没有加–force选项。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666805250-399008f4-fb0a-443d-b973-b33ad96a92ef.jpeg)

上面完成快照的创建，回滚验证后，还有个操作时快照的删除的，也就是利用snapshot-delete命令删除，这个没有什么好说的。但是需要提示一点，快照删除后，虚拟机的磁盘大小并不会变小。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666805857-12f46e1a-8d81-4122-8044-39a39d3c04d5.jpeg)

以上，就是KVM虚拟机的内置快照操作内容，我觉得我是讲明白了。。。关于kvm虚拟机的状态备份，也就是save命令，时间比较长，一般要5－10分钟左右，造成该问题的原因是：save vm保存的是当前客户机系统的运行状态（包括：内存、寄存器、CPU执行等的状态），保存为一个文件，而且要在load vm时可以完全恢复，这个过程比较复杂，如果客户机里面的内存很大、运行的程序很多，save vm比较耗时，也是可以理解的。暂时很难有什么改进方法。

而往往我们并不需要去备份一个虚拟机当前状态完整的快照，实际运维中中可能只需要对disk做一个快照就OK了。所以，这就要提到外部快照（External snapshot）。

## **virsh命令实现KVM虚拟机的外置快照**

KVM的外部快照功能比较实用，可以支持仅对disk进行快照，也支持live snapshot，很多虚拟化云方案中一般也会使用外部快照功能创建快照链。不过，KVM虚拟机要支持外部快照功能，需要qemu版本在1.5.3及以上，否则只能通过下载最新的qemu源码包进行编译安装。可以通过rpm -aq qemu查看当前宿主机的qemu软件包的版本，也可以通过qemu-kvm –version命令来查看。

这里需要注意一点，centos系统默认将qemu-kvm命令放在/usr/libexec/目录下，所以在当前命令行执行qemu-kvm会提示找不到命令，需要带上命令的全路径执行，也就是/usr/libexec/qemu-kvm –version，或者通过创建软链接的方法，在/usr/bin目录下创建一个qemu-kmv命令的软链接，就可以执行qemu-kvm命令了。（我采用的就是这种办法）

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666805825-f7436298-29cc-471b-a66f-1d856aea18a3.jpeg)

**step1：**在虚拟机关机状态下，我们创建一个外部快照ext_snap01。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666805763-0f12dd6f-2a4e-43e3-9a62-b3cdf2d13047.jpeg)

从上面的查询中不难看出centos75.ext_snap01的backing file来自于/kvm_host/vmdisk/centos75.img 。同样，也可以利用下面的命令进行查看：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666805911-39c947ef-c216-4f95-bc95-c663c4df72f7.jpeg)

**创建完外部快照后，原磁盘镜像文件会变成只读，新的变更都会写入到新的快照文件中。也就是我们常说链接克隆中创建的差分磁盘。**忘了的，请回顾本站存储虚拟化相关文章。。。。。。

**step2：**我们做完快照后，写一个200M的测试文件到虚拟机centos7.5的系统/opt/目录下，并做如下验证。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666805899-ddd16d24-453e-487f-a9d5-9665ab23a58d.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666806573-6c5eac17-6d7c-45f8-9b2a-feaa7d4f86d3.jpeg)

如上图，在未写入测试文件data01前，虚拟机外部快照盘centos75.ext_snap01的大小为5.5M（上图红框部分），在写入data01测试文件后变为207M（上图黄框部分），且虚拟机原始磁盘大小仍为4.3G（上图蓝框部分）。至于，显示大小不准确的问题，是因为我们通过-h选项通过ls命令查看，在转换成可读模式大小时的转换误差导致，并非真实文件大小不准确。其实，用原始字节数表示就没有误差，但是那样反人类啊。。。

至于外部快照的删除、回滚等操作与内部快照一直，我就懒得演示验证了。。。。**不过，外部快照多了一个快照合并的功能，也就是将差分磁盘中的数据变化写入到原始磁盘镜像中。主要有两种方式：一种是通过blockcommit命令完成，从top文件合并数据到base，也就是从快照文件向backing file合并；另一种是blockpull命令完成，从base文件合并数据到top，也就是从backing file向快照文件合并。**截止目前只能将backing file合并至当前的active的快照镜像中，也就是说还不支持指定top的合并。

但是，在centos系统执行virsh blockcommit或者virsh blockpull等命令时，都会报QEMU不支持的二进制操作错误或者是版本不支持等错误，这是CentOS内部集成的qemu-kvm版本问题导致。因此，此时有两种解决办法，一种是升级QEMU-KVM，通过源码编译内核完成升级。另一种就是使用qemu-img命令，commit对应上面blockcommit，rebase对应blockpull操作。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666806615-575a5abb-f33f-421d-8858-273a602da04c.jpeg)

通过qemu-img commit命令将快照文件的数据的合并到原始镜像文件中（图中红色框部分），此时不需要指定backing file文件。而且，我们此时查看原始镜像文件大小，发现变大为4.5G（图中黄色框部分）。此时，我们可以删除磁盘快照文件，进入虚拟机查看我们当初的测试文件是否还存在。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666806551-96f996ff-1bee-4542-8caa-5c99400447dd.jpeg)

**以上，commit的操作，如果采用rebase命令，反向合并时，就必须要指定backing file文件。其他操作与commit操作一致，大家自己尝试。。。。。**

在openstack等平台中，使用的快照功能大都是基于外置快照的形式。这里需要特别注意一点，KVM的快照不要和Vmware的快照混为一谈，Vmware的所有快照默认情况下都是基于baseimage，创建的overlays，也就是所有快照的backing file都是baseimage，删除任一个快照对其他快照的使用和还原都不会有影响，而KVM不同，其快照之间存在链式关系，snap01是基于baseimage，snap02是基于snpa01，以此类推。。。基中任一环节出现问题，都会出现无法还原到之间状态。**所以，在做快照的合并，删除等操作前，一定要提前通过qemu-img info –backing-chain查看虚拟机的快照链关系，避免发生不可挽回的数据丢失错误。**

## **libguestfs-tools工具使用总结**

libguestfs是一组Linux下的C语言的API ，用来访问虚拟机的磁盘映像文件。其项目官网是http://libguestfs.org/ 。该工具包含的软件有virt-cat、virt-df、virt-ls、virt-copy-in、virt-copy-out、virt-edit、guestfs、guestmount、virt-filesystems、virt-partitions等工具，具体用法也可以参看官网。该工具可以在不启动KVM guest主机的情况下，直接查看guest主机内的文内容，也可以直接向img镜像中写入文件和复制文件到外面的物理机，甚至也可以像mount一样，支持挂载操作。

现在，我们将虚拟机centos7.5关机，然后查看其镜像磁盘使用情况，可以通过virt-df命令实现。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666806723-0b29503d-fa6c-483e-b759-ac72068d8109.jpeg)

同样，我们想查看虚拟机关机态下根分区下详细文件/目录信息，可以使用virt-ls命令实现。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666806653-5f80ce4e-8989-46b9-bc94-508ea086e1e5.jpeg)

此时，我们需要从关机态下的虚拟中拷贝一个文件到宿主机中，可以使用copy-out命令实现。但是，需要带上-d选项指定哪个虚拟机。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666807508-d7a2746c-b1e0-454e-87dc-6ef22f71bce0.jpeg)

既然能从虚拟机中拷贝文件到本地宿主机，自然也能从本地宿主机拷贝文件到虚拟机，通过copy-in命令就能实现。其实，这些命令工具的用法和Linux的原始命令非常类似，所以熟悉Linux常用命令的使用是必须要掌握的技能。

接下来，我们查看下虚拟机的文件系统以及分区信息，通过filesystems命令实现。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666807279-1ef72525-93c5-44a4-ac95-f6bf867b8b83.jpeg)

完成虚拟机文件系统和分区信息的查询后，我们可以通过guestmount命令，将虚拟机centos7.4的系统盘离线挂载到宿主机的/mnt分区，与挂载光驱操作类似，但是我们可以设置挂载镜像的操作方式。比如：只读、只写、读写等。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666807592-92b38f23-12a8-456a-90c8-ebf9c9f7654f.jpeg)

以上是挂载linux系统的镜像磁盘，如果需要挂载windows虚拟机的磁盘，需要额外安装ntfs -3g的工具来识别windows系统的NTFS文件系统格式。

实际应用中的KVM主机也会遇到像物理机一样的情况，如系统崩溃、无法引导等情况。物理机出现该情况时，我们可以通过光盘引导、单用户模式、PE引导、修复或升级安装等方式获取系统内的文件和数据，KVM中同样也可以使用上述方法，既可以上面的利用libguestfs-tools的工具进行挂载修改，也可以通过linux系统原生的mount -o loop方式进行挂载修改。下面，我们就通过mount命令对raw格式和qcow2格式磁盘进行挂载做个演示。

## **raw磁盘镜像的挂载**

由于raw格式简单原始，其通常做为多种格式互相转换的中转格式，所以对raw格式的磁盘挂载操作时需重点掌握。raw格式的分区挂载也有两种方法：**一种是通过计算偏移量offset方式挂载。另一种是通过kpartx分区映射方式实现挂载。**

**计算偏移量offset方式的思路为找出分区的开始位置，使用mount命令的offset参数偏移掉前面不需要的，即可得到真正的分区。**如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666807511-64fcc7f0-770e-49f0-ad11-ec0d7dac8c4d.jpeg)

上图中，我们可以通过fdisk -lu命令查看磁盘镜像的分区信息，可以发现centos74.img的磁盘一共有3个分区，每个分区都有对应的起始扇区和结束扇区编号，且每个扇区的大小为512字节。通过这些信息，我们就可以计算分区的offset值，从而实现找到真正的分区内容存放位置。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666807256-89cf7dec-0736-4d51-bbaa-ca12a5dbb581.jpeg)

然后，通过mount -o loop，offset=xxxx命令用其实扇区的偏移字节数实现分区挂载，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666808824-0b30b671-07f1-4cb1-9801-ed0bbd3f5042.jpeg)

除了上述计算偏移量的方法外，还可以通过**kpartx分区映射方式实现挂载。首先通过kpartx工具对磁盘镜像做个分区映射。如下：**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666807956-be03d5b3-d426-46fc-b771-10513c8ee92f.jpeg)

上图中，通过-av选项添加磁盘镜像并将映射结果显示出来（图中红色框部分），然后就可以知道磁盘镜像有3个分区，分别映射为loop0p1、loop0p2和loop0p3（图中黄色框部分）。完成映射后，我们就可以像挂载普通磁盘或光驱那样对映射分区进行挂载。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666808174-670aa64a-f82e-405c-ae30-423093da76de.jpeg)

注意映射的设备的文件位置在/dev/mapper目录下。我们可以看下该目录下的具体信息，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666808414-d6a93518-44d1-4c6c-ab55-75cbe412135e.jpeg)

上图中可以很清楚的发现上面的映射分区是通过软链接的方式建立的与磁盘内部分区的映射关系。通过磁盘映射方式下实现挂载后，记得使用完成不仅要卸载挂载点，还需要删除映关系。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644666808602-fe9fdcf8-961a-4c9d-8d98-8af5bbf3d67c.jpeg)

以上就是raw格式镜像挂载，需要注意的如果虚拟机使用了LVM逻辑卷，那么针对逻辑卷的挂载操作需要使用losetup工具完成，具体可以查询使用方法，非常简单这里就不在赘述了。而qcow2格式的镜像的挂载不能通过kaprtx直接映射，可以先转换成raw的格式进行处理，也可以通过libguestfs-tools工具处理，还可以使用qemu-nbd直接挂载。就速度上而言qemu-nbd的速度肯定是最快的。不过由于centos/redhat原生内核和rpm源里并不含有对nbd模块的支持及qemu-nbd（在fedora中包含在qemu-common包里）工具，所以想要支持需要编译重新编译内核并安装qemu-nbd包 。

至此，KVM虚拟机的存储管理实战全部介绍完毕，后面我们进入KVM虚拟机的网络管理实战。
