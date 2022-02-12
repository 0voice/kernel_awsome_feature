在x86-64架构的处理器中，KVM需要的硬件辅助虚拟化分别为Intel的虚拟化技术（Intel VT）和AMD的AMD-V技术。在前面我们介绍过，CPU不仅要在硬件上支持VT技术，还要在BIOS中将其功能打开，KVM才能使用到。目前，大多数服务器和PC的BIOS都默认已经打开VT。至于如何在VMware Workstations虚拟机和物理服务器上打开，详见本站《KVM到底是个啥？》一文。

## **部署和安装KVM**

**KVM的部署和安装主要有两种方式：一种源码编译安装，另一种就是通过CentOS的YUM工具安装。**作为学习实验途径，我们这里主要介绍YUM工具安装方式。至于源码编译安装一般属于生产研发人员的操作，这里我们只给一些关键提示，有兴趣的同学可自行研究。

**1）源码编译安装方式。**KVM作为Linux内核的一个module，是从Linux内核版本2.6.20开始的，所以要编译KVM，你的Linux操作系统内核必须在2.6.20版本以上，也就是CentOS 6.x和CentOS 7.x都天生支持，但是CentOS 5.x以下版本需要先升级内核。

**下载KVM源代码，主要通过以下三种方式：**

1. 下载KVM项目开发中的代码仓库kvm.git。
2. 下载Linux内核的代码仓库linux.git。

1. 使用git clone命令从github托管网站上下载KVM的源代码。

根据上面三种途径下载源代码后就可以通过make install的方式编译安装了，编译安装完成后还需要根据KVM官方指导手册进行相关的配置。。。我们这里不展开，大家可自行搜索源码方式安装。**需要注意一点儿的是，除了下载KVM的源码外，还需要同时下载QEMU的源码进行编译安装，因为它俩是一辈子的好基友嘛。。。**

**2）YUM工具安装。**首先，需要查看 CPU是否支持VT技术，就可以判断是否支持KVM。然后，再进行相关工具包的安装。最后，加载模块，启动libvirtd守护进程即可。具体步骤如下：

**Step1：**确认服务器是否支持硬件虚拟化技术，如果有返回结果，也就是结果中有vmx（Intel）或svm(AMD)字样，就说明CPU的支持的，否则就是不支持。如下图所示，表示服务器支持Intel的VT-x技术，且有4个CPU。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659232402-7954c4a6-5fc6-426f-9107-d29289e43bb7.jpeg)

**Step2：**确认系统关闭SELinux。如下图所示，表示系统已经关闭Linux。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659232058-ea64992a-1bc7-437b-b050-e73827767ff9.jpeg)

如果没有关闭，可以使用如下命令永久关闭：

复制

[root@C7-Server01 ~]# setenforce 0  # 临时关闭
\# 永久关闭
[root@C7-Server01 ~]# sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
[root@C7-Server01 ~]# sed -i 's/^SELINUXTYPE=.*/SELINUXTYPE=disabled/g' /etc/selinux/config

**Step3：**安装rpm软件包。

由于KVM是嵌入到Linux内核中的一个模块，我们这里首先安装用户安装用户空间QEMU与内核空间KMV交互的软件协议栈QEMU-KVM，代码如下：

复制

[root@C7-Server01 ~]# yum -y install qemu-kvm

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659232011-abc803da-f801-48a8-bdee-78bbad70319f.jpeg)

如上图，不仅完成qemu-kvm（红色框部分）安装，还同步安装qemu-img、qemu-kvm-comm两个依赖包的安装（黄色框部分）。注意：由于我已经安装过，上面提示是updated，没有安装的话，应该是installed。

完成QEMU-KVM软件协议栈的安装后，我们安装libvirt*、virt-*等管理工具，代码如下：

复制

[root@c7-test01 ~]# yum install -y libvirt* virt-*

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659232013-bd0a465f-90ee-478b-bc70-457bf1363075.jpeg)

如上图，将KVM的管理工具libvirt系列软件包和virt-系列软件包（红色框部分）及相关依赖软件包（黄色框部分）进行了安装。

最后，需要安装一个二层分布式虚拟交换机，用于KVM虚拟机的虚拟接入，可以选择OVS，也可以选择Linux bridge或是其他商用的分布式交换机如VMware、华为FC等。我们这里只出于学习的目的，选择安装Linux bridge即可。代码如下：

复制

[root@c7-test01 ~]# yum install -y bridge-utils

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659232010-606caf8a-538e-4775-9294-1c28cde40f87.jpeg)

如上图，提示我们系统已经安装该软件包且是最新版本（红色框部分），如果你没有安装过，系统会直接安装。

**3）加载KVM模块，使得Linux Kernel变成一个Hypervisor。**首先，加载KVM内核模块，然后查看KVM内核模块的加载情况即可。具体步骤如下：

首先，通过modprobe指令加载kvm内核模块，如下：

复制

[root@c7-test01 ~]# modprobe kvm

然后，通过lsmod指令查看kvm模块的加载情况，如下：

复制

[root@c7-test01 ~]# lsmod | grep kvm

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659232632-83248d21-3a54-41b3-a0d8-e66aaee8caa9.jpeg)

**4）启动libvirtd守护进程，并设置开机启动。**因为libvirt管理工具，是用于KVM虚机全生命周期的统一管理工具，它由外部统一API接口、守护进程libvirtd和CLI工具virsh三部分组成，其中守护进程libvirtd用于所有虚机的全面管理。同时，其他管理工具virt-*、openstack等都是调用libvirt的外部统一API接口完成KVM的虚机的管理。所以，我们需要启动libvirtd守护进程，并设置开机启动。代码如下：

复制

[root@c7-test01 ~]# systemctl enable libvirtd && systemctl start libvirtd && systemctl status libvirtd

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659232693-a8fa96db-e900-479c-8567-51b805c1daed.jpeg)

这里注意一下，一般情况我们单机玩KVM虚拟机时，libvirtd守护进程默认监听的是UNIX domain socket套接字，并没有监听TCP socket套接字。为了同时让libvirtd监听TCP socket套接字，需要修改/etc/libvirt/libvirtd.conf文件中，将tls和tcp，以及tcp监听端口前面的注释取消。然后重新通过libvirtd的原生命令加载配置文件，使其生效。而系统默认的systemctl命令由于不支持–listen选项，所以不能使用。参考代码如下：

复制

[root@c7-test01 ~]# systemctl stop libvirtd 
[root@c7-test01 ~]# libvirtd -d --listen

通过netstat命令查看libvirtd监听的tcp端口为16509，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659232692-fe9c24ac-6571-44df-8fc6-776b0f1eb2b4.jpeg)

最后，进行tcp链接验证，如果连接成功，表示服务启动正常且tcp监听正常。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659232939-e65fda12-557a-48a2-a3eb-1a44fefd8ce2.jpeg)

## **安装第一个KVM虚拟机**

安装虚拟机之前，我们需要创建一个镜像文件或者磁盘分区等，来存储虚拟机中的系统和文件。首先，我们利用**qemu-img工具**创建一个镜像文件。**这个工具不仅能创建虚拟磁盘，还能用于后续虚拟机镜像管理。**比如，我们要创建一个raw格式的磁盘，具体代码如下：

复制

[root@c7-test01 ~]# qemu-img create -f raw ubuntu1204.img 20G 
Formatting 'ubuntu1204.img', fmt=raw size=21474836480

如上，表示在当前目录下(/root）创建了一个20G大小的，raw格式的虚拟磁盘，名称为：ubuntu1204.img。虽然，我们看它的大小时20G，实际上并不占用任何存储空间，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659233246-3dc1ba7a-e771-4b25-b92b-3be666f4c09b.jpeg)

这是因为qemu-img聪明地为你按实际需求分配文件的实际大小，它将随着image实际的使用而增大。如果想一开始就分配实际大小为20G的空间，不仅要使用raw格式磁盘，还需加上-o preallocation=full参数选项，这样创建速度会很慢，但是创建的磁盘实实在在占用20G空间。我们这里用创建一个5G磁盘来演示，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659233380-10c09289-efb9-497a-a986-e74b01822369.jpeg)

除raw格式以外，qemu-img还支持创建其他格式的image文件，比如qcow2，甚至是其他虚拟机用到的文件格式，比如VMware的vmdk、vdi、Hyper-v的vhd等，不同的文件格式会有不同的“-o”选项。为了演示我们的第一个虚拟机，我们现在创建一个qcow2格式的虚拟磁盘用作虚拟机的系统磁盘，大小规划40G，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659233373-d81627ba-ddf2-45df-8576-770049b20ca5.jpeg)

上面创建的虚拟磁盘实际就是KVM虚拟机后续的系统盘，在创建完虚拟机磁盘后，我们将要安装的系统镜像盘上传到当前目录下或者通过光驱挂载也可。我们这里为了安装速度问题，采用上传到宿主机本地的方式，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659233496-7dc2d6e3-7935-499d-b113-8032f8182df9.jpeg)

**注意：这里的宿主机指的是我们的VMware虚拟机，这里的虚拟机指的是我们在VMware虚拟机中创建的KVM虚拟机，千万别搞混了。**

然后，就是我们的关键一步，通过virt-install命令安装虚拟机，代码如下：

复制

[root@c7-test01 ~]# virt-install --name ubt1204d \
--virt-type kvm \
--ram 2048 --vcpus=2 \
--disk path=/root/ubt1204d.img,format=qcow2,size=40 \
--network network=default,model=virtio \
--graphics vnc,listen=192.168.101.251 --noautoconsole \
--cdrom /root/ubuntu-12.04.5-desktop-amd64.iso 

Starting install...
Domain installation still in progress. You can reconnect to 
the console to complete the installation process.

如上，上面的指令表达的意思为：–name指定虚拟机的名称，–virt-type指定虚拟机的类型为kvm，–ram指定给虚拟机分配的虚拟内存大小为2GB，–vcpus指定给虚拟机分配的虚拟cpu为2个，–disk指定虚拟机的系统盘，就是我们刚才创建的虚拟磁盘，–network指定虚拟机使用的虚拟交换机，这里使用系统的默认配置，默认配置文件为/etc/libvirt/qemu/networks/default.xml，其详细信息如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659233657-cd813214-68aa-4812-920f-fedc7ea55a3f.jpeg)

上图中网络模式采用nat方式就表示虚拟机可以通过网桥访问internet或者外部网络。–graphics指定虚拟机的模拟显示器界面，这里采用vnc方式，并监听宿主机地址192.168.101.251。–cdrom选项指定虚拟机的安装镜像配置，也就是系统安装盘iso的位置。

上面虚拟机正式启动后，可以通过本地机器的vnc客户端连接到KVM虚拟机，作为虚拟机的模拟显示器。为此，首先需要查看VNC连接的端口，代码如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659233919-a49aa368-1b52-4770-b52b-c9948b6fe361.jpeg)

如上图黄色框部分，通过本地机器VNC客户端连接目标机器192.168.101.251的0端口就能模拟虚拟机的显示器，进一步完成图形化安装配置，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659233972-60aecdf0-750b-4ec0-8abd-00aeb10c67cb.jpeg)

连接成功后，与真实物理机装系统的操作一致，可以使用键盘、鼠标完成各类安装配置操作。虚拟机模拟显示器的界面如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659233984-afcdaa3d-253b-4e48-b20b-a6e54275334e.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659234313-19e167e7-5f00-4e17-8622-3523f8bc3c5e.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659234318-361f9a17-6100-4de6-bb78-2b77d610fdc8.jpeg)

由于我们安装的是ubuntu12.04的桌面版系统，所以完成虚拟机的安装后，后续每次使用虚拟机都需要使用VNC客户端进行连接操作。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659234736-4b69e191-d68f-4930-9371-5041d174cf73.jpeg)

同时，我们可以通过virsh命令工具查看本机上所有虚拟机的状态，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659234522-d0bafae5-5530-43d4-bced-78a37a56ef15.jpeg)

最后，提醒一下：我们在自动化运维中介绍过通过kicstart或cobbler批量安装宿主机操作系统，这种方式也是可以在KVM虚拟机安装系统中使用，通过在virt-install命令增加–extra-args选项就可实现。但是，一般我们不这么玩，在云计算和虚拟化场景下，均是通过手动安装一台模板虚拟机，将其系统盘转换为模板镜像格式文件，然后通过批量分发虚拟机的方式（就是我们在存储虚拟化中讲的链接克隆和快照方式）完成虚拟机批量部署操作，电信云中通过VNFD描述的多台虚拟机资源部署也是同样的方式完成，只不过里面借助了OpenStack的编排引擎。这种方式我们在后续介绍KVM虚拟镜像格式实操一文中会详细介绍。
