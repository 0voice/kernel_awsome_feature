在上一篇我们介绍了KVM最重要的管理工具libvirt，它是KVM其他管理工具的基础，处于KVM管理架构的中间适配层。本篇我们主要要介绍libvirt的命令行管理工具的virsh，它也是将libvirt作为基础，通过封包调用实现的。所以，在看本篇内容之前，最好将上一篇的内容做个预习。

## **libvirt命令行工具virsh**

virsh通过调用libvirt API实现虚拟化管理，与virt-manager工具类似，都是管理虚拟化环境中的虚拟机和Hypervisor的工具，只不过virsh是命令行文本方式的，也是更常用的方式。

在使用virsh命令行进行虚拟化管理操作时，可以使用两种工作模式：交互模式和非交互模式。交互模式是连接到Hypervisor上，然后输入一个命令得到一个返回结果，直到输入quit命令退出。非交互模式是直接在命令上通过建立URI连接，然后执行一个或多个命令，执行完后将命令的输出结果返回到终端上，然后自动断开连接。两种操作模式的截图如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659502986-da32b85d-6ce9-4866-96a6-b0961448f285.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659503040-05c51b34-1404-4db8-bb80-96acf5146232.jpeg)

我们经常在本地使用virsh命令，这是一种特殊的交互模式，也是最常用的模式，它本质上就是默认连接到本节点的Hypervisor上。

libvirt中实现的功能和最新的QEMU/KVM中的功能相比有一定的滞后性，因此virsh只是实现了对QEMU/KVM中的大多数而不是全部功能的调用。同时，由于virsh还实现了对Xen、VMware等其他Hypervisor的支持，因此有部分功能对QEMU/KVM无效。下面，我们还是按照“边验证边理解”的套路，对virsh常用的命令进行分类整理说明。

## **域管理（虚拟机管理）命令**

virsh的最重要的功能之一就是实现对域（虚拟机）的管理，但是与虚拟机相关的命令是很多的，包括后面的网络管理、存储管理也都有很多是对域（虚拟机）的管理。为了简单起见，本文使用“”来表示一个域的唯一标识（而不专门指定为“”这样冗长的形式）。常用的virsh域管理命令如下：

| **virsh中的虚拟机管理命令** |                                                              |
| --------------------------- | ------------------------------------------------------------ |
| 命令                        | 功能描述                                                     |
| list                        | 获取当前节点上多有虚拟机的列表                               |
| domstate                    | 获取一个虚拟机的运行状态                                     |
| dominfo                     | 获取一个虚拟机的基本信息                                     |
| domid                       | 根据虚拟机的名称或UUID返回虚拟机的ID                         |
| domname                     | 根据虚拟机的ID或UUID返回虚拟机的名称                         |
| dommemstat                  | 获取一个虚拟机的内存使用情况的统计信息                       |
| setmem                      | 设置一个虚拟机的大小（默认单位的KB）                         |
| vcpuinfo                    | 获取一个虚拟机的vCPU的基本信息                               |
| vcpupin                     | 将一个虚拟机的vCPU绑定到某个物理CPU上运行                    |
| setvcpus                    | 设置一个虚拟机的vCPU的个数                                   |
| vncdisplay                  | 获取一个虚拟机的VNC连接IP地址和端口                          |
| create<dom.xml>             | 根据虚拟机的XML配置文件创建一个虚拟机                        |
| define<dom.xml>             | 定义一个虚拟机但不启动，此时虚拟机处于预定义状态             |
| start                       | 启动一个预定义的虚拟机                                       |
| suspend                     | 暂停一个虚拟机                                               |
| resume                      | 恢复一个虚拟机                                               |
| shutdown                    | 对一个虚拟机下电关机                                         |
| reboot                      | 让一个虚拟机重启                                             |
| reset                       | 与reboot的区别是，它是强制一个虚拟机重启，相当于在物理机上长按reset按钮，可能会循环虚拟机系统盘的文件系统 |
| destroy                     | 立即销毁一个虚拟机，相当于物理机上直接拔电源                 |
| save<file.img>              | 保存一个运行中的虚拟机状态到一个文件中                       |
| restore<file.img>           | 从一个被保存的文件中恢复一个虚拟机的运行                     |
| migrate<dest_uri>           | 将一个虚拟机迁移到另一个目标地址                             |
| dump<core.file>             | coredump一个虚拟机保存到一个文件                             |
| dumpxml                     | 以XML格式转存出一个虚拟机的信息到标准输出中                  |
| attach-device<device.xml>   | 向一个虚拟机添加xml文件中的设备，也就是热插拔                |
| detach-device<device.xml>   | 将一个XML文件中的设备从虚拟机中移除                          |
| console                     | 连接到一个虚拟机的控制台                                     |

上表中，只是列出常用的几个KVM虚拟机全生命周期管理命令，如果想查找全量KVM虚拟机全量生命周期管理命令，可以使用命令帮助。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659503269-d627ffdc-570a-43e3-8111-c73df2d8a74d.jpeg)

如上图，由于输出太长，我们只截取了一部分。上图中每个命令后面都有详细的文字说明来描述命令的用途。

## **虚拟机全生命周期管理实战**

### **虚拟机状态信息查询**

有了上面的命令储备后，我们下来在环境中进行实操验证。首先，我们通过**list命令，查看下当前节点的虚拟机个数**，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659503002-cd2f2def-9252-4a32-b1fd-532df3103ef2.jpeg)

如上图，本节点有2个虚拟机，当前状态均为shut off，也就是没有启动。那么，我们现在**通过start命令，将上述两个虚拟机同时启动**，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659503005-825f7828-d170-47cd-9a8e-d47e23243033.jpeg)

启动后，我们可以**通过domstate命令查看虚拟机当前的状态**，并且**通过dominfo命令查看虚拟机的详细信息**，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659503538-71256788-310a-4705-910c-90a2a1f242f4.jpeg)

同时，我们可以**通过dommemstate和vcpuinfo命令，查看虚拟机的虚拟内存信息及vCPU信息。**如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659503604-ceecb111-f144-43b8-8b81-8379080a4126.jpeg)

### **虚拟机vCPU与vMEM管理操作**

如上图，通过上面的vCPU的详细信息，我们发现cto74d的两个vCPU都与pCPU0默认绑定。那么，我们现在**可以通过vcpupin命令将vCPU1与pCPU1进行绑定。**如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659503753-6e5741f4-3d10-4609-8a14-04d776c67c5c.jpeg)

同时，我们发现cto74d虚拟机只有2个vCPU。现在，我们需要给它增加到4个vCPU怎么办？在KVM中，可以**通过setvcpus命令完成cto74d虚拟机的vCPU在线热添加。**但是，在热添加之前，我们需要明确一个概念，那就是**虚拟机的vCPU最大可分配数与当前vCPU数的区别**。为了讲明白这个概念，我们还是先来看虚拟机cto74d的XML配置文件，可以**通过dumpxml命令将虚拟机XML配置文件输出到屏幕上。**如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659503721-490c11fe-980f-4a49-8e0b-3842c1eca0b1.jpeg)

上图中表示cto74d虚拟机的vCPU最大可分配数为2。而且，我们还可以**通过指令emulatorpin查看虚拟机当前的vCPU使用情况。**如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659503975-53bd2a42-a36b-4b53-85d3-1a853e6cb29d.jpeg)

上图中表示虚拟机cto74d当前使用的vCPU个数也为2。所以，在这种配置要求下，自然无法通过指令将cto74d虚拟机的vCPU数量调整为4个。为了实现我们的需求，首先需要**通过shutdown命令将虚拟机下电**，然后修改配置文件如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659504288-f5cdaab7-502d-4a65-8e74-a9ac0e4ca277.jpeg)

上图中，红色框部分表示我们将虚拟机下电，黄色框部分我们将虚拟机的vCPU配置修改为：最大支持4个vCPU，当前使用2个vCPU。我们可以查看XML文件验证下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659504405-3aadf738-7f1b-485b-b37f-f60b195fe8cb.jpeg)

完成上述修改后，我们需要**通过define命令重新定义下cto74d这个虚拟机**，然后通过start命令启动虚拟机。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659504401-70d31d8d-1124-4a6c-ba5e-616acfaf4677.jpeg)

现在，我们具备条件后，就可以使用命令setvcpus命令在线对cto74d虚拟机的vCPU进行热添加，添加到4个。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659504748-500cd973-09c5-4812-914e-a25cf2776fd1.jpeg)

在完成vCPU的热插后，那么vCPU是否可以热拔呢？**答案是不可以**，因为当vCPU被分配给虚拟机使用时，虚拟机中的程序进程就会占用刚分配的vCPU，此时如果我们进行热拔操作，前提是需要将上面的进程迁移到别的vCPU，但是我们是无法确切知道有哪些进程的。所以，vCPU自然不支持热拔操作。那么，既然KVM支持CPU和内存的虚拟化，内存是否也支持在线调整呢？答案是必须支持。我们首先通过dominfo命令看下当前虚拟机的内存情况。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659504630-4d20339c-6496-4960-9a4a-d4f64e868baa.jpeg)

上图表示，虚拟机当前最大内存和可用内存都是4194304KB，也就是4GB大小。我们可以通过**通过setmem指令设置虚拟机的可用内存。**与vCPU概念类似，**这种方式支持的在线调整范围只能小于当前虚拟机的最大内存配额，也就是4GB以内。否则，需要关机，先调整虚拟机支持的最大内存，然后再调整虚拟机支持的可用内存。**如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659504821-8fcd14a6-2714-4b30-bbba-de704dac9fef.jpeg)

从上图，我们发现，内存在线调整内存是可以减小的，即在线将虚拟机原内存从4G调整为3G。另外，**setmaxmem指令用于调整虚拟机的最大可支持内存，只能在虚拟机下电后执行。**否则会报如下错误：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659505035-495c4fb3-81cf-4669-b4d9-00adb6358924.jpeg)

### **虚拟机磁盘和网卡的热插拔**

完成了vCPU和内存的在线调整实战后，我们接下来玩玩热插拔。先来玩热插拔磁盘，**在KVM中通过virsh管理工具向虚拟机热插拔磁盘，可以使用attach-disk命令。**为了实现上述需求，首先我们需要创建一个虚拟磁盘文件，然后通过attach-disk命令将其挂载到虚拟机中，然后通过控制台进入虚拟机对新挂载的磁盘进行格式化，并创建对应目录对其挂载。最后，我们向挂载目录中写入测试文件测试。同时，需注意向虚拟机挂载raw格式qcow2格式的磁盘是不一样的。

我们先来热插raw格式的磁盘，操作过程如下：

**Step1：创建一个虚拟磁盘cto74d_data.raw，磁盘格式为raw，大小为5G**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659505137-dd6a6a94-57b6-4dc1-acd5-f88cf3fee3ab.jpeg)

**Step2：为虚拟机热插磁盘，需要注意指定磁盘的位置要使用绝对路径，并设置虚拟盘符为vdb**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659505690-8acec2e6-735a-43c7-b687-16f174a4a2c5.jpeg)

**Step3：通过控制台进入虚拟机cto74d，查看磁盘信息**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659505514-baf674ef-6af9-4d0b-bee6-60a40b609c45.jpeg)

**Step4：对新挂载的磁盘格式化，并创建一个/data目录进行挂载**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659505545-ff4cb4a4-3657-405d-b32c-16d8bb56ca3b.jpeg)

**Step5：写入一个大文件进行测试**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659505727-b2b02048-6dee-4bb0-958a-aa6d0d5cd3df.jpeg)

**Step6：我们在宿主机上通过domblklist命令查看虚拟机的磁盘信息**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659505753-df07aee8-5155-4236-9fc8-57f5cabfbeeb.jpeg)

如上图，可以看见当前虚拟机有两块磁盘，一块系统盘，一块数据盘。我们查看虚拟机XML文件信息进行验证。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659506075-da587fde-de18-4114-ba71-23a350e90a32.jpeg)

以上就是raw格式磁盘的热插操作过程，如果想热插qcow2格式的磁盘（后缀为*.qcow2）或想热插qcow2格式的镜像文件（后缀为*.img），在Step1创建磁盘时，需要通过**-o size选项指定预分配大小**，否则系统会默认挂载一个几百K大小虚拟盘，这样会导致空间不够。同事，还需要通过preallocation选项指定磁盘格式为metadata，也就是元数据格式。除此之外，这两种类型磁盘的热插操作与raw格式磁盘一致。

上面完成了磁盘的热插，那么**要实现磁盘的热拔呢？可以通过detach-disk这个命令完成。**但是，这时需要注意一点：**使用virsh指令删除磁盘会直接强制将虚拟机中磁盘删除，如果磁盘已经挂载使用，要停止该磁盘的写操作，否则会造成数据丢失，拔掉的磁盘并没有删除，仍然存储在当时创建的位置，需要使用可以再挂载使用。**因此，在热拔磁盘之前，我们需要先进入虚拟机将对该磁盘的写入进程全部暂停，然后将磁盘从挂载点卸载，最后回到宿主机通过detach-disk命令完成磁盘的热拔。这里，大家自己玩吧，我就不陪着演示了。。。。。

磁盘的热插拔我们验证了，接下来我们玩网卡的热插拔。**在KVM虚拟机中，要实现网卡的热插，可以使用attach-interface命令完成。同理，热拔的命令就是detach-interface。**首先，我们通过domiflist命令查看当前虚拟机的网络和网卡信息，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659506282-97c977ec-2ff6-4916-a467-d0ccfb9a3ce4.jpeg)

如上图，黄色框部分就表示当前虚拟机cto74d的网络信息，类型（type）为network，源设备（source）为default。如果在创建KVM虚拟机时，网络配置是自定义网桥，那么这里的类型（type）为bridge，源设备（source）为自定义网桥名称，如br0，

有了上面的知识储备后，下面我们**通过attach-interface命令来热插一个网卡**，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659506378-a3e74de5-5162-4463-84c4-5d077d816ece.jpeg)

同时，我们可以**通过domifaddr命令查看新增加的网卡动态获得的IP地址**，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659506448-657f73e7-2638-4159-91b9-9263fa52e33e.jpeg)

接下来，我们尝试下在宿主机通过SSH方式远程连接vnet2的地址是否成功。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659506356-c9532556-bb97-459c-8158-5c63ba836e42.jpeg)

上图表示，可以通过vnet2的地址192.168.122.230远程连接虚拟机cto74d，表明我们新增加的网卡有效且地址分配正常（这个地址是通过DHCP方式动态分配的），通过输入密码就可以远程登录。

搞定网卡的热插后，我们接下来自然要实现网卡的热拔。与磁盘类似，在对网卡热拔时，首先需要停止网卡的工作，这样后续数据包就不会转发到该网卡上，不到导致网络丢包。而且，在对网卡热拔时，我们要根据网卡的MAC地址来实现，而不能通过网卡名称，否则libvirt内部会将该网卡名称归属的同一个二层设备上所有端口销毁。**对网卡热拔的命令就是detach-ifinterface。**如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659506734-327bd2b5-342d-455a-8f76-93ae517f57ef.jpeg)

上图中，通过–mac参数指定mac地址删除，就不会发生误删除事件。

## **虚拟机的热迁移**

接下来，我们再来玩一个更高级的操作，就是**KVM虚拟机的热迁移**。在KVM中虚拟机的迁移分为热迁移和冷迁移。每一种迁移方式又细分为基于本地存储和基于共享存储两种。

两种方式下的冷迁移均比较简单，就是将原虚拟机的磁盘文件和配置文件拷贝至目标服务器，然后通过配置文件定义一个新的虚拟机启动即可。而基于本地存储的热迁移与此类似，也需要在目标主机创建同一个存储文件，通过暴露TCP端口，基于socket完成迁移，同样比较简单，但是过程很慢。有兴趣的可参考https://www.cnblogs.com/lyhabc/p/6104555.html这篇文章。

我们这里演示的是**基于共享存储的热迁移（电信云采用分布式存储，每一个计算组规划区都是一个共享存储池）**，它实现的前提就是需要源目服务器之间有共享存储。因此，为了演示这个操作，我们需要在源主机（192.168.101.251）和目标主机（192.168.101.252）之间通过NFS方式建立共享文件系统，也可以使用GFS2集群文件系统来实现。至于，NFS网络共享文件系统如何配置，请参见本站的Linux常用运维工具分类中的NFS网络文件共享系统一文，这里不再赘述。

整个实验的拓扑如下，源主机和目标主机都作为NFS的客户端与NFS服务器（192.168.101.11）之间互通，实现数据的共享存储。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659506803-bff728cf-7207-434d-ade5-abea62020b14.jpeg)

**Step1：首先确保两个NFS客户端服务器与NFS服务器端共享目录正常，且两个客户端服务器上存放虚拟机磁盘的目录一致。**如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659506969-4e73068f-1cf4-49f2-81ef-8ebcfeb0fad0.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659506929-15b05b50-762a-46a4-ad33-08ba0228948e.jpeg)

**Step2：在251服务器首先将虚拟机磁盘创建在/mydata/nfsdir下，格式为qcow2格式，然后去252服务器的对应目录下检查确保文件共享正常。**如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659507044-8246211a-210f-4209-bcdc-490cd6699911.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659507193-c6631759-7df7-4203-b44a-edd9f2048a91.jpeg)

**Step3：在251服务器创建虚拟机cto67s，并启动。**如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659507455-f23627f3-4005-4998-ba07-de7debad7b04.jpeg)

在安装过程中，我们手动通过图形化方式安装，因此需要查询vnc图形化界面的连接地址和服务端口。**可以通过vncdisplay命令查看虚拟机的VNC客户端连接地址及服务端口**，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659507495-a17ebc61-0114-4e19-a0ef-ffc349639920.jpeg)

然后，还是通过vnc界面安装虚拟机，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659507577-6361f76b-1083-4556-bac6-004504f46a7f.jpeg)

完成，虚拟机OS安装后，需要重启生效。然后，源主机（192.168.101.251）中通过start命令，启动虚拟机cto67s。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659507680-484f51a2-a684-4a06-87f0-c304a1166f38.jpeg)

**Step4：开始虚拟机cto67s的热迁移。**如下：

如上图，当前虚拟机cto67s在源主机（192.168.101.251）处于开机运行状态，为了清晰说明热迁移的过程，我们需要知道虚拟机cto67s的ip地址，这样在虚拟机热迁移时，由于内存热数据的迭代拷贝，会有一个暂停-恢复-暂停-恢复的过程，反映在网络测就会出抖动或丢包。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659507755-d2296931-ae86-444a-b02d-38004fba602f.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659508129-97b214ad-f37c-4bfb-b6c1-fa2dc514d996.jpeg)

完成上述的准备工作后，我们**通过migrate命令开始进行虚拟机热迁移**。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659508236-709bf319-f1c0-405e-9017-efcc292a3667.jpeg)

**如上图，我们期望的热迁移失败啦。。。。这是为啥？**我们看上图红色框部分的报错内容，**提示：域名解析失败。**其实，在迁移期间，目标主机上运行的libvirtd会从其希望接收迁移数据的地址和端口创建URI，并将其发送回在源主机上运行的libvirtd。在我们的例子中，目标主机（192.168.122.252）的名称设置为“c7-test02”。出于某种原因，在该主机上运行的libvirtd无法将该名称解析为可以发回的IP地址，因此它发送了主机名，希望源libvirtd在解析名称时会更成功。但是，源主机的libvirt因为也没有配置这个主机名的解析，所以提示：Name or service not know。

有了上面的分析，那就简单了，无非就是在源主机和目的主机上配置主机名和IP的解析就可解决上述问题。这里，有两种配置方法：一种是在/etc/resolve中配置DNS解析，但是不建议这样配置，因为这里一般是配置公网域名解析的地方。另一种就是在/etc/hosts中配置主机名与IP的映射关系。我们采用第二种方法解决。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659508253-015ce955-cfe1-4823-bb6e-7c218f226c2f.jpeg)

完成上面的配置后，我们再次尝试热迁移。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659508314-b59a1f36-e181-4877-a32c-5ecaa971527d.jpeg)

同时，我们在源主机和目标主机分别查看虚拟机cto67s的状态。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659508465-b9b57f7f-307f-4690-af03-6a91f4b96173.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659508836-560c6c50-9cae-411c-b9a7-ec2ba704ed2f.jpeg)

此时，虽然虚拟机cto67s已经在目标主机上启动，但是目标主机上还没有虚拟机cto67s的配置文件。所以需要根据当前虚拟机的状态，创建配置文件并定义虚拟机。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659508752-b064dffe-e880-479e-8a46-b6be2e179231.jpeg)

最后，我们从目标主机上进入虚拟机cto67s进行验证，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659508833-0e758083-e5cf-480f-aa35-d4ec94a36433.jpeg)

至此，KVM虚拟机热迁移演示完成。这里需要提示一下，由于我们创建的虚拟机都是nat网络模式的，这样在迁移后，在源主机ping虚拟机测试会提示目标不可达，但是在虚拟机内部ping源主机可以ping通，只是迁移过程中会有1-2个ICMP包丢失或抖动。如果，非要在宿主机上ping测试虚拟机来观察迁移过程中的丢包或抖动过程，可以将虚拟机的网络模式改为桥接bridge。而且，源主机和目标主机必须归属同一个网桥bridge。

以上，就是我们KVM虚拟机全生命周期管理实战的全部内容，至于虚拟机的挂起、恢复等操作都比较简单，我也就懒得举例演示，各位可以自行玩玩，挺有意思的，真的。。。。
