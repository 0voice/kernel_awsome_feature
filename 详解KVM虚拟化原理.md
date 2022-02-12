## KVM架构

###### KVM（Kernel-based Virtual Machine）包含一个为处理器提供底层虚拟化、可加载的核心 模块kvm.ko（kvm-intel.ko或kvm-amd.ko），使用QEMU（QEMU-KVM）作为虚拟机上层 控制工具。KVM无需改变Linux或Windows系统就能运行。

###### KVM就是内核的一个模块，用户空间通过QEMU模拟硬件提供给虚拟机使用，一台虚拟机就 是一个普通的Linux进程，通过对这个进程的管理，就可以完成对虚拟机的管理

## qemu

###### QEMU本身并不是KVM的一部分，其本身是一个著名的开源虚拟机软件，与KVM不同， QEMU是一个纯软件的实现，所以性能低下。但是，其优点是可以模拟很多硬件。

###### 以盖房子为例，KVM可以理解为开发商，房子盖的很好，但是不会装修；QEMU可以理解为 装修公司，房子盖的不好，但是装修做的很好；所以我们采用KVM盖房子（硬件虚拟化，实 现CPU和内存计算资源的模拟），使用QEMU装修（软件模拟，实现网卡、显卡、存储控制器 和硬盘等）。

###### KVM只是一个内核的模块，没有用户控件的管理工具，KVM虚拟机可以借助QEMU的管理工 具来管理，QEMU也可以借助KVM来加速，提升虚拟机的性能

## Libvirt

###### Libvirt是一套开源的虚拟化的管理工具，Libvirt的设计目标是通过相同的方式管理不同的虚拟 化引擎，比如KVM、Xen、Hyper-V、VMware ESXi等。目前大多数场景使用Libvirt管理 KVM

##### Libvirt主要由3部分组成：

###### ①一套API的lib库，支持主流的编程语言，包括C、Python等。

###### ②Libvirtd服务。

###### ③命令行工具virsh

## 虚拟机的概念

###### 虚拟机是运行在物理服务器上的一个完整的系统，它包含 有自己的虚拟CPU、内存、磁盘和网卡等虚拟硬件资源。

###### 虚拟硬件信息在虚拟机的配置文件中定义。

###### 操作系统和应用程序在虚拟机中的运行方式与它们在普通 物理机上的运行方式没有任何区别。

###### 虚拟机的xml描述文件存储在物理机**/etc/libvirt/qemu**中， xml文件描述了此虚拟机中的硬件配置信息。 MyVM.xm

## 虚拟CPU的实现原理

### 基于X86处理器的虚拟化

###### X86架构存在17条敏感的非特权指令，运行时不会产生异常，这些指令在客户操作系统上的执行 会破坏整个系统

![在这里插入图片描述](https://www.icode9.com/i/ll/?i=bff795b21e3f406f8ec946854bd4c478.png?,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MTEzNDE4OA==,size_16,color_FFFFFF,t_70)

### 基于VMX处理器的虚拟化

###### 硬件辅助虚拟化解决敏感非特 权指令无法陷入问题的解决思 路：引入VMX模式(Virtual Machine eXtension)

![在这里插入图片描述](https://www.icode9.com/i/ll/?i=f9da477ccbdf4898bd90202536985eef.png?,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MTEzNDE4OA==,size_16,color_FFFFFF,t_70)

### 基于VMCS处理器的虚拟化

###### 客户机状态域：保存非根模式下VCPU运行状 态；

###### 宿主机状态域：保存根操作模式下CPU的运行 状态；

###### VM执行控制域：控制VM-Exit操作发生时的行 为，比如某些敏感指令、异常和中断是否产生 VM-Exit操作；

###### VM-Entry控制域和VM-Exit控制域：对VM- Entry和VM-Exit操作的具体行为进行控制规定；

###### VM-Exit信息域：存放VM-Exit产生的原因

![在这里插入图片描述](https://www.icode9.com/i/ll/?i=630262e37c854deab4c6c6a1b6fa234d.png?,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MTEzNDE4OA==,size_16,color_FFFFFF,t_70)

### 虚拟CPU

##### 每个VM分配一个时间片，轮流执行，在虚拟化应用环境中，高效利用CPU资源

###### 每个虚拟机（VM）都是虚拟机操作系统内核 的一个进程，每个进程被分配一个时间段（时 间片），即允许VM运行的时间

###### 每个虚拟机（VM）都是虚拟机操作系统内核 的一个进程，每个进程被分配一个时间段（时 间片），即允许VM运行的时间

###### 如果VM在时间片结束前阻塞或结束，CPU立 即切换，最大程度利用CPU资源

###### 默认情况下，所有VM被视为同等重要，时间 片长度相同

### CPU的工作模式

##### custom (推荐) ：

###### QEMU模拟 的CPU，类型为qemu64兼容 性好，但不能为虚拟机提供最 优性能，如aes加密等

##### host-mode：

###### QEMU模拟的 与主机CPU接近的CPU ，性 能尽可能的与主机CPU接近， 但迁移兼容性较差。

##### host-passthrough：

###### 透传主 机CPU型号和大部分功能给虚 拟机，性能最优，但迁移兼容 性很差，同一厂家的不同

###### 代 CPU也有可能不能迁移。

## 虚拟内存的实现原理

#### 内存虚拟化-虚拟机物理地址

![在这里插入图片描述](https://www.icode9.com/i/ll/?i=3defc694cbd5419ca5cfd474d671a88e.png?,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MTEzNDE4OA==,size_16,color_FFFFFF,t_70)

#### 内存虚拟化-EPT（Extend Page Table扩展页表）

![在这里插入图片描述](https://www.icode9.com/i/ll/?i=6aedd42a5b3a44a7907899f83980f76b.png?,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MTEzNDE4OA==,size_16,color_FFFFFF,t_70)

## 虚拟硬盘的实现原理

### 虚拟机的磁盘类型

##### KVM使用的虚拟机磁盘主要有如下类型：

###### IDE硬盘

###### 高速硬盘（Virtio硬盘，默认）

### 虚拟磁盘类型

###### 虚拟机磁盘支持块设备和文件 两种类型。

###### 对虚拟机磁盘有较高要求时可 以使用块设备，比如Oracle数 据库。

###### 虚拟机磁盘选择文件类型时， 对应的存储卷格式支持智能 （QCOW2）和高速（RAW） 两种格

###### 式，建议使用智能格式 （支持动态分配磁盘空间、快 照等优点）。

 ![在这里插入图片描述](https://www.icode9.com/i/ll/?i=cd4496b8db9c4dd1bf2e45ba1e773c86.png?,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MTEzNDE4OA==,size_16,color_FFFFFF,t_70)

### 智能存储卷

##### 虚拟机的虚拟磁盘使用智能格式时，需要 以文件的方式为虚拟机提供虚拟磁盘。

##### 虚拟磁盘文件使用qcow2文件格式。

##### Qcow2文件的开始2M为qcow2文件头， 记录了文件大小、L1表实际地址和快照信 息等重要数据

##### L1表记录了L2表的实际地址信息

##### L1表记录了L2表的实际地址信息

##### 数据块用于存放虚拟机的数据，默认大小 为2M

![在这里插入图片描述](https://www.icode9.com/i/ll/?i=ea4762bc50b4400e9f11f7b0e88608f7.png?,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MTEzNDE4OA==,size_16,color_FFFFFF,t_70)

## 虚拟网卡

### 输入/输出设备的虚拟化

###### 网卡虚拟化（SR-IOV，Single Root I/O Virtualization and Sharing）技术将一块物理网卡 可以虚拟出多个虚拟网卡；

###### VT-d（Intel® Virtualization Technology for Directed I/O） 技术将网卡分配给虚拟机。

### 虚拟网卡

###### KVM中虚拟机的网卡是通过QEMU模拟实 现的，包括了普通网卡、Intel E1000和 Virtio网卡

###### 普通网卡：模拟了RealTek Link 8139百兆 网卡 l Intel e1000网卡：模拟了Intel 82540E千 兆网卡

###### Virtio网卡：虚拟化内核平台软件驱动的网 卡，速度最快、功能最全，推荐使用 （CSMP中部分虚拟机默认）

### 虚拟网卡IO处理

![在这里插入图片描述](https://www.icode9.com/i/ll/?i=54069623d6e748f5a9a9988c46cc319c.png?,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MTEzNDE4OA==,size_16,color_FFFFFF,t_70)

### 虚拟交换机

##### 虚拟交换机是通过软件模拟的、 具有实体交换机系统功能的网 络平台。

##### 为虚拟机、主机、外部网络提 供网络连接。

##### 每个上行端口对应一个物理适 配器，多对上行端口和物理适 配器时可做聚合，达到链路冗 余。

## KVM常用的命令

### virsh相关

###### cat /proc/cpuinfo | grep -E ‘(vmx|svm)’ ##确认CPU是否支持虚拟化

```
[root@ostack1 home]# cat /proc/cpuinfo | grep -E '(vmx|svm)'
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc aperfmperf eagerfpu pni pclmulqdq vmx ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch epb tpr_shadow vnmi ept vpid fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 invpcid rtm rdseed adx smap xsaveopt dtherm ida arat pln pts
```

###### virsh list ##查看正在运行的虚拟机

```
[root@ostack1 home]# virsh list 
 Id    Name                           State
----------------------------------------------------
 2     csmpha-master                  running
 52    instance-00000037              running
 53    instance-00000038              running
 54    instance-00000039              running
 59    instance-0000003b              running
 60    instance-0000003c              running
 63    instance-0000003f              running
 64    instance-0000003e              running
```

###### virsh list --all ##查看所有的虚拟机

```
[root@ostack1 home]# virsh list --all 
 Id    Name                           State
----------------------------------------------------
 2     csmpha-master                  running
 52    instance-00000037              running
 53    instance-00000038              running
 54    instance-00000039              running
```

###### virsh start/shutdown windows ##开启关闭windows的主机

###### virsh destroy win7 ##关闭名字为win7的虚拟机，类似直接给虚拟机关闭电源

###### virsh reboot win7 ##重起虚拟机

###### virsh suspend win7 ##暂停虚拟机

###### virsh resume win7 ##暂停之后恢复虚拟机

###### virsh define /etc/libvirt/qemu/win7.xml ##根据文件定义生成一个虚拟机

###### virsh undefine win7 ##取消定义，即删除win7虚拟机

###### virsh edit win7 ##编辑虚拟机

###### systemctl restart libvirtd ##重启libvirt-bin

### 

### 磁盘相关的命令

###### df –Th 查看分区以及空间使用情况

###### qemu-img info win7.qcow2 ##查看名字为win7.qcow2的虚拟机磁盘文件物理信息

###### qemu-img info win7.qcow2 ##查看名字为win7.qcow2的虚拟机磁盘文件物理信息

###### qemu-img create win7-2.qcow2 -f qcow2 5G ##创建一个磁盘文件,格式为qcow2,5G大小

###### qemu-img convert -f raw -O qcow2 123.raw 123.qcow2 ##将raw格式的磁盘修改为qcow2

###### qemu-img snapshot -c test2 /vms/2T/ucen ##给虚拟机磁盘文件创建快照

###### qemu-img snapshot –l /vms/2T/ucen ##查看虚拟机磁盘文件的快照信息

###### qemu-img snapshot -a test2 /vms/2T/ucen ##使用磁盘文件的快照恢复

###### qemu-img resize win7.qcow2 +1G ##给磁盘文件增加1GB空间（只能增加，不能减小）

###### qemu-img rebase -u -b /vms/test/VSR_base_1 VSR ## 设置磁盘的base文件，用于三级镜像 文件切换路径，修改base文件等

### 网络相关的命令

###### ovs-vsctl list-br ##列出所有网桥信息

###### ovs-vsctl list-br ##列出所有网桥信息

###### ovs-vsctl add-port vswitch1 eth1 ##将eth1挂接到vswitch1上

###### ovs-vsctl list-ports vswitch0 ##列出挂接到vswitch0上的接口

###### ovs-vsctl port-to-br eht0 ##列出已挂接eth

###### ovs-vsctl show ##查看网桥信息

###### ovs-vsctl del-port vswitch1 eth1 ##删除网桥vswitch1上挂接的eth1接口

###### ovs-vsctl del-br vswitch1 ##删除名为vswitch1的网桥

###### ifconfig vswitch1 up/down ##将网桥vswitch1 开启、

###### ovs-ofctl dump-flows br-int ##查看流表信息

r eht0 ##列出已挂接eth

###### ovs-vsctl show ##查看网桥信息

###### ovs-vsctl del-port vswitch1 eth1 ##删除网桥vswitch1上挂接的eth1接口

###### ovs-vsctl del-br vswitch1 ##删除名为vswitch1的网桥

###### ifconfig vswitch1 up/down ##将网桥vswitch1 开启、

###### ovs-ofctl dump-flows br-int ##查看流表信息

###### service openvswitch-switch status/restart/start/stop #查看、重启、启动、关闭虚拟交换机服务
