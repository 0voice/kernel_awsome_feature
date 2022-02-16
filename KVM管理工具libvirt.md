通过前面两篇文章的介绍，相信大家对KVM虚拟机的原理、软件架构、运行机制和部署方式等有所了解。但是，那些东西只能算是非常基础入门的东西。尤其我们介绍的部署KVM虚拟机时用到的virt-install命令，不仅参数很多，很难记忆，而且要想用好virt-*系列工具首先需要对libvirt工具有更深刻的了解。因为，virt-*系列工具其实是对libvirt工具的封装调用，而libvirt工具又是对底层qemu工具的封装调用，其目的都是为了使命令更加友好与用户交互。本篇文章就详细介绍这几种与libvirt相关的管理工具，为后续各种配置打下良好基础。

## **libvirt管理工具**

提到KVM的管理工具，就不得不提大名鼎鼎的libvirt，因为libvirt是目前使用最为广泛的对KVM虚拟机进行管理的工具和应用程序接口，而且一些常用的虚拟机管理工具（如virsh、virt-install、virt-manager等）和云计算框架平台（如OpenStack、ZStack、OpenNebula、Eucalyptus等）都在底层使用libvirt的应用程序接口，也就是说**libvirt实际上是一个连接底层Hypervisor与上层应用的中间适配层。**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017251736-17df4c93-cd86-460d-b3be-767e8bdd63f9.jpeg)

如上图，**libvirt作为一个中间适配层，可以让底层Hypervisor对上层用户空间的管理工具是完全透明的，**也就是说上层管理应用无需关注底层的差别，只需要调用libvirt去完成，因此只需要对接libvirt的对外开放api接口即可。同时，**libvirt对底层多种不同的Hypervisor的支持是通过一种基于驱动的架构来实现的，类似neutron中的ml2层。libvirt对不同的Hypervisor提供了不同的驱动。**比如：对Xen有xen_dirver.c驱动文件，对vmware有vmware_dirver.c驱动文件，对kvm有qemu_driver.c驱动文件等等。。。

**在libvirt中涉及节点Node、Hypervisor、域Domain等几个概念，其逻辑关系如下：**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017251749-19ed7d0b-8910-4743-a17a-7b7789aa6e2b.jpeg)

- **节点（Node）是一个物理机器**，上面可能运行着多个虚拟客户机。Hypervisor和Domain都运行在节点上。
- **Hypervisor也称虚拟机监控器（VMM），如KVM、Xen、VMware、Hyper-V等，是虚拟化中的一个底层软件层**，它可以虚拟化一个节点让其运行多个虚拟机（不同虚拟机可能有不同的配置和操作系统）。

- **域（Domain）是在Hypervisor上运行的一个虚拟机操作系统实例。**域也被称为实例（instance，如在亚马逊的AWS云计算服务中客户机就被称为实例）、客户机操作系统（guest OS）、虚拟机，它们都是指同一个概念。

libvirt相关的配置文件都在/etc/libvirt/目录之中，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017251732-2c58c0d6-f321-432d-b116-fcf38fe58187.jpeg)

主要涉及四个文件：

**1）/etc/libvirt/libvirt.conf：**用于配置一些常用libvirt连接（通常是远程连接）的别名。比如：uri_aliases = [ “remote1=qemu+ssh:[//root@192.168.101.252](mailto://root@192.168.101.252)/system” ]，有这个别名后，就可以在用virsh等工具或自己写代码调用libvirt API时使用这个别名，比如在Python可以这样调用：

复制

conn = libvirt.openReadOnly('remote1')

**2）/etc/libvirt/libvirtd.conf：**是libvirt的守护进程libvirtd的配置文件，被修改后需要让libvirtd重新加载配置文件（或重启libvirtd）才会生效。在文件的每一行中使用“配置项=值”（如tcp_port=”16509”）这样key-value格式来设置。我们上篇文章介绍的说监听tcp链接和端口，就是在这里配置。

**3）/etc/libvirt/qemu.conf：**是libvirt对QEMU的驱动的配置文件，包括VNC、SPICE等，以及连接它们时采用的权限认证方式的配置，也包括内存大页、SELinux、Cgroups等相关配置。

**4）/etc/libvirt/qemu/目录：**存放的是使用QEMU驱动的域的配置文件，也就是kvm虚拟机的配置文件以及networks目录存放默认网络配置文件。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017251745-5fd6f700-9cc3-40b3-82a6-abffed9ea4bc.jpeg)

**要让某个节点能够利用libvirt进行管理（无论是本地还是远程管理），都需要在这个节点上运行libvirtd这个守护进程，以便让其他上层管理工具可以连接到该节点，libvirtd负责执行其他管理工具发送给它的虚拟化管理操作指令**，比如：virsh、virt-manager、nova-compute等等。在CentOS 7.x系统中，libvirtd作为一个系统服务存在，可以利用systemctl工具进行启动、停止、重启、重载等操作。另外，libvirtd守护进程的启动或停止，并不会直接影响正在运行中的客户机。libvirtd在启动或重启完成时，只要客户机的XML配置文件是存在的，libvirtd会自动加载这些客户的配置，获取它们的信息。

**libvirtd除了是系统服务外，本身还是一个可执行程序，可以通过一些选项完成系统进程的启动。**比如：-d选项可以让libvirtd作为守护进程（daemon）在后台运行。-f选项可以指定libvirtd的配置文件启动服务，而不使用默认的配置文件。-l选项开启配置文件中指定的TCP/IP连接，监听TCP socket和服务端口。-p指定进程PID，不使用系统默认分配的PID号等等。

## **解读虚拟机的XML配置文件**

在使用libvirt对虚拟化系统进行管理时，很多地方都是以XML文件作为配置文件的，包括虚拟机的配置、宿主机网络接口配置、网络过滤、各个客户机的磁盘存储配置、磁盘加密、宿主机和虚拟机的CPU特性等等。所以，对XML文件进行解析，对后续使用libvirtd管理KVM虚拟机理解的更为深刻。

我们首先来看下，我们前一篇文章创建的虚拟机ubt1204d的XML配置文件如下：

复制

[root@c7-test01 ~]# cat /etc/libvirt/qemu/ubt1204d.xml <!--WARNING: THIS IS AN AUTO-GENERATED FILE. CHANGES TO IT ARE LIKELY TO BEOVERWRITTEN AND LOST. Changes to this xml configuration should be made using:  virsh edit ubt1204dor other application using the libvirt API.--><domain type='kvm'>  <name>ubt1204d</name>  <uuid>4c4797b9-776c-44e4-8ef5-3ad445092d0f</uuid>  <memory unit='KiB'>2097152</memory>  <currentMemory unit='KiB'>2097152</currentMemory>  <vcpu placement='static'>2</vcpu>  <os>    <type arch='x86_64' machine='pc-i440fx-rhel7.0.0'>hvm</type>    <boot dev='hd'/>  </os>  <features>    <acpi/>    <apic/>  </features>  <cpu mode='custom' match='exact' check='partial'>    <model fallback='allow'>Broadwell-IBRS</model>  </cpu>  <clock offset='utc'>    <timer name='rtc' tickpolicy='catchup'/>    <timer name='pit' tickpolicy='delay'/>    <timer name='hpet' present='no'/>  </clock>  <on_poweroff>destroy</on_poweroff>  <on_reboot>restart</on_reboot>  <on_crash>destroy</on_crash>  <pm> 。。。。省略若干行。。。</domain>

如上，可以看到在该虚拟机的XML文件中，所有有效配置都在和标签之间，这表明该配置文件是一个域的配置。而XML文档中的注释是在两个特殊的标签之间，如<！–注释–>。

通过libvirt启动客户机，经过文件解析和命令参数的转换，最终也会调用qemu命令行工具来实际完成客户机的创建。用这个XML配置文件启动的客户机，它的qemu命令行参数是非常详细、非常冗长的一行，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017251761-36dc9ef3-da0c-4a3d-a410-b6df14ce127e.jpeg)

下面，我们就逐个模块来解析虚拟机的XML配置文件。

### **CPU的配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017253241-20bdab98-811b-4be9-b8d2-99cf3e611942.jpeg)

上面的XML文件中关于CPU的配置项如上图所示，vcpu标签，表示客户机中vCPU的个数，这里为2。features标签，表示Hypervisor为客户机打开或关闭CPU或其他硬件的特性，这里打开了ACPI、APIC等特性。而cpu标签中定义了CPU的基础特性，在创建虚拟机时，libvirt自动检测硬件平台，默认使用Broadwell类型的CPU分配给虚拟机。在CPU模型中看见的Broadwell-IBRS，可以在/usr/share/libvirt/cpu_map.xml文件查看其具体信息，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017253302-588be08d-c7d1-43ae-9a89-86fb2f755134.jpeg)

**对于CPU模型的配置，有以下3种模式。**

1）custom模式：就是这里示例中表示的，基于某个基础的CPU模型，再做个性化的设置。

2）host-model模式：根据物理CPU的特性，选择一个与之最接近的标准CPU型号，如果没有指定CPU模式，默认也是使用这种模式。xml配置文件为：。

3）host-passthrough模式：直接将物理CPU特性暴露给虚拟机使用，在虚拟机上看到的完全就是物理CPU的型号。xml配置文件为：。

对vCPU的分配，可以有更细粒度的配置，如下：

复制

<domain>
  ...
  <vcpu placement='static' cpuset="1-4,^3,6" current="1">2</vcpu>
  ...
</domain>

cpuset表示允许到哪些物理CPU上执行，这里表示客户机的两个vCPU被允许调度到1、2、4、6号物理CPU上执行（^3表示排除3号）；而current表示启动客户机时只给1个vCPU，最多可以增加到使用2个vCPU。除了这种方式外，libvirt还提供cputune标签来对CPU的分配进行更多调节，如下：

复制

<domain>  ...  <cputune>    <vcpupin vcpu="0" cpuset="1"/>    <vcpupin vcpu="1" cpuset="2,3"/>    <vcpupin vcpu="2" cpuset="4"/>    <vcpupin vcpu="3" cpuset="5"/>    <emulatorpin cpuset="1-3"/>    <shares>2048</shares>    <period>1000000</period>    <quota>-1</quota>    <emulator_period>1000000</emulator_period>    <emulator_quota>-1</emulator_quota>  </cputune>  ...</domain>

还记得我们在介绍DPDK的亲和性技术不？这里就是**设置亲和性特性的调优配置**，其中vcpupin标签表示将虚拟CPU绑定到某一个或多个物理CPU上，如“<vcpupin vcpu=”2”cpuset=”4”/>”表示客户机2号虚拟CPU被绑定到4号物理CPU上；“”表示将QEMU emulator绑定到1~3号物理CPU上。在不设置任何vcpupin和cpuset的情况下，虚拟机的vCPU可能会被调度到任何一个物理CPU上去运行。而“2048”表示虚拟机占用CPU时间的加权配置，一个配置为2048的域获得的CPU执行时间是配置为1024的域的两倍。如果不设置shares值，就会使用宿主机系统提供的默认值。

除了亲和性绑定外，**还有NUMA架构的调优设置，可以配置虚拟机的NUMA拓扑，以及让虚拟机针对宿主机NUMA特性做相应的策略设置等，主要在标签和标签中完成配置。**NUMA特性配置需要在真实物理服务器上完成，手头暂时没有对应环境，也就不列举配置项了。但是，能够对NUMA进行配置的前提是了解NUMA的原理，所以理解前面DPDK技术和计算虚拟化中NUMA技术原理是重点。

### **内存的配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017253307-2006dfb1-0b22-4783-b01c-33919640f200.jpeg)

在KVM虚拟机的XML配置文件中，内存的大小配置如上，大小为2,097,152KB（即2GB），memory标签中的内存表示虚拟机最大可使用的内存，currentMemory标签中的内存表示启动时分配给虚拟机使用的内存。在使用QEMU/KVM时，一般将二者设置为相同的值。

另外，还记得我们讲内存虚拟化时，提到的内存气球技术不？在KVM虚拟机创建时，默认采用内存气球技术给虚拟机提供虚拟内存，它的配置项在标签对中的memballoon子标签中，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017253789-2433b2bd-d592-453e-8f1b-6cfcf1451fb8.jpeg)

如上图，该配置将为虚拟机分配一个使用virtio-balloon驱动的内存气球设备，以便实现虚拟机内存的ballooning调节。该设备在客户机中的PCI设备编号为0000:00:06.0，也就是设备的I/O空间地址。

### **虚拟机启动项配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017253764-1296fd17-7730-49b3-b808-05973d68ea89.jpeg)

如上图，这样的配置表示客户机类型是hvm类型，HVM（hardware virtual machine，硬件虚拟机）原本是Xen虚拟化中的概念，它表示在硬件辅助虚拟化技术（Intel VT或AMD-V等）的支持下不需要修改客户机操作系统就可以启动客户机。因为KVM一定要依赖于硬件虚拟化技术的支持，所以**在KVM中，客户机类型应该总是hvm**，操作系统的架构是x86_64，机器类型是pc-i440fx-rhel7.0.0，这个机器类型是libvirt中针对RHEL 7系统的默认类型，也可以根据需要修改为其他类型。

boot选项用于设置虚拟机启动时的设备，这里只有hd（即硬盘）一种，表示从虚拟机硬盘启动，如果有两种及以上，就与物理机中BIOS设置一样，按照从上到下的顺序先后启动。

### **网络的配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017253911-4d589092-85b8-472c-9288-f3fc53981159.jpeg)

如上图，上面虚拟机XML文件的网络时配置是一种**NAT方式的网络配置**，这里type=’network’和就是表示使用NAT的方式，并使用默认的网络配置，虚拟机将会分到192.168.122.0/24网段中的一个IP地址，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017254001-1c576ac6-0c6d-4044-b10e-6e0e355e3b4a.jpeg)

使用NAT网络配置的前提是宿主机中要开启DHCP和DNS服务，一般libvirtd进程默认使用dnsmasq同时提供DHCP和DNS服务，在宿主机中通过以下命令可以查看：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017254300-fcce2e6e-ae56-4336-91fe-c567152b3e90.jpeg)

这里的NAT网络的配置在/etc/libvirt/qemu/networks/defaule.xml中配置，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017254423-dd554867-1936-4e21-b4e1-b4118cc6bc01.jpeg)

如上，这里配置就是提供分布式虚拟交换机virbr0，用于KVM虚拟机的虚拟接入。详细配置在前一篇文章有介绍，这里不再赘述。由于使用的Linux bridge作为虚拟交换机，因此可以在宿主机中查看该网桥的实际端口，指令如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017254593-27c05666-14c5-4096-87bc-a66ffb98e89e.jpeg)

上图中的vnet0就是KVM虚拟机ubt1204d的虚拟网卡，这样就实现了虚拟机ubt1204d与宿主机之间的网络互通。

除了NAT网络模式外，还可以通过自建一个Linux bridge的网桥，在创建虚拟机时，–network选项中指定自建的网桥，这样虚拟机安装完毕后，就会自动分配自建的网桥IP地址段中的一个地址，这种方式称为**桥接模式**。同理，这种方式也需要开启DHCP和DNS服务。通过自建网桥方式，在虚拟机XML文件的配置字段如下：

复制

<interface type='bridge'>
   <mac address='52:54:00:e9:e0:3b'/>
   <source bridge='br0'/>
   <model type='virtio'/>
   <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
  </interface>

上述配置表示，是用bridge为虚拟机提供网络服务，mac address指的是虚拟机的MAC地址，表示使用宿主机中的br0网络接口来建立网桥，这个网络接口需要在宿主机中创建配置文件，类似普通网卡的配置，并制定一个物理网卡与该网桥进行绑定，作为上联接口。表示在虚拟机中使用virtio-net驱动的网卡设备，也配置了该网卡在虚拟机中的PCI设备编号为0000:00:03.0。

除了上述两种常见的配置外，还有一种**用户模式网络**的配置，类似Virtual Box中的内部网络模式。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017254599-bb673154-9ccf-4899-98d3-913f04642ffc.jpeg)

其在虚拟机的XML字段描述如下：

复制

<interface type='user'>
  <mac address="00:11:22:33:44:55"/>
 </interface>

如上，type=’user’表示该客户机的网络接口是用户模式网络，是完全由QEMU软件模拟的一个网络协议栈，也就是一个纯用户态内部虚拟交换机。因为没有虚拟接口与宿主机互通，所以宿主机无法与这样的虚拟机通信，这种网络模式下只有同一个内部虚拟交换机上不同虚拟机才能互通。

还记得我们在介绍I/O虚拟化时，讲过网卡的硬件直通技术VMDq和SR-IOV吗？在KVM虚拟机中也可以**绑定这类硬件直通网卡**，在其XML文件中目前有两种方式：**新方式通过标签来绑定**，在其中指定驱动的名称为vfio，并指定硬件的I/O地址即可，但是这种新方式目前只支持SR-IOV方式。由于我们条件限制，特意找了个网上的配置给大家曹侃，示例如下：

复制

<interface type='hostdev'>
   <driver name='vfio'/>
   <source>
   <address type='pci' domain='0x0000' bus='0x08' slot='0x10' function= '0x0'/>
   </source>
  <mac address='52:54:00:6d:90:02'>
  </interface>

如上，用指定使用哪一种分配方式（默认是VFIO，如果使用较旧的传统的device assignment方式，这个值可配为’kvm’），用标签来指示将宿主机中的哪个VF分配给宿主机使用，还可使用来指定在客户机中看到的该网卡设备的MAC地址。

由于新方式支持的直通网卡类型较少，为了支持更多的硬件直通设备，一般还是采用老方式通过标签来绑定。**这种方式不仅支持有SR-IOV功能的高级网卡的VF的直接分配，也支持无SR-IOV功能的普通PCI或PCI-e网卡的直接分配。但是，这种方式并不支持对直接分配的网卡在客户机中的MAC地址的设置，在客户机中网卡的MAC地址与宿主机中看到的完全相同。**同样，我们在网上找了个配置示例给大家参考：

复制

<hostdev mode='subsystem' type='pci' managed='yes'>
   <source>
   <address domain='0x0000' bus='0x08' slot='0x00' function='0x0'/>
   </source>
</hostdev>

如上，表示将宿主机中的PCI 0000:08:00.0设备直接分配给客户机使用。

### **存储的配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017255062-757669a9-f264-47a7-8cb6-e076aa4e2fe3.jpeg)

如上图，每个存储设备都由一对标签来描述，上述配置我们当前的虚拟机有两个存储设备，一个虚拟机硬盘，另一个是虚拟机光驱。

在虚拟机硬盘和光驱设备描述中，类型均是file，表示虚拟机硬盘使用文件方式。除此之外，还有**block、dir或network取值**，分别表示块设备、目录或网络文件系统作为虚拟机磁盘的来源。后面device属性表示让虚拟机如何来使用该磁盘设备，其取值为floppy、disk、cdrom或lun中的一个，分别表示软盘、硬盘、光盘和LUN（逻辑存储单元），默认值为disk（硬盘）。

**子标签用于定义Hypervisor如何为该磁盘提供驱动**，它的name属性用于指定宿主机中使用的后端驱动名称，QEMU/KVM仅支持name=’qemu’，但是它支持的类型type可以是多种，包括raw、qcow2、qed、bochs等。除了这两个属性外，还有cache属性（我们没有设置），它表示在宿主机中打开该磁盘时使用的缓存方式，可以配置为default、none、writethrough、writeback、directsync和unsafe，其具体含义详见本站DPDK系列文章。

**子标签表示磁盘的位置**，当标签的type属性为file时，应该配置为这样的模式，而当type属性为block时，应该配置为这样的模式。

**子标签表示将磁盘暴露给虚拟机时的总线类型和设备名称。dev属性表示在客户机中该磁盘设备的逻辑设备名称**，而**bus属性表示该磁盘设备被模拟挂载的总线类型**，bus属性的值可以为ide、scsi、virtio、xen、usb、sata等。如果省略了bus属性，libvirt则会根据dev属性中的名称来“推测”bus属性的值，比如，sda会被推测是scsi，而vda被推测是virtio。

**子标签表示该磁盘设备在虚拟机中的I/O总线地址**，这个标签在前面网络配置中也是多次出现的，如果该标签不存在，libvirt会自动分配一个地址。

### **域的配置**

我们前面介绍过，在libvirtd进程中，域、实例和Gust OS是一个概念，就是我们常说的虚拟机。所以，在KVM虚拟机的XML配置文件中，标签是范围最大、最基本的标签，是其他所有标签的根标签。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017255293-ef80f7cb-0f3b-4a64-9156-353b238238cb.jpeg)

在标签中可以配置两个属性：一个是type，用于表示Hypervisor的类型，可选的值为xen、kvm、qemu、lxc、kqemu、VMware中的一个；另一个是id，其值是一个数字，用于在该宿主机的libvirt中唯一标识一个运行着的客户机，如果不设置id属性，libvirt会按顺序分配一个最小的可用ID。这个系统自动的分配的ID可以通过如下指令查看：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017255378-ecf9878e-7adb-4b14-8b08-32f2c560d82d.jpeg)

如上图，我们知道系统给我们当前ubt1204d虚拟机分配的最小可用ID是2。

### **域的元数据配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017255712-d5a051b0-c8c7-4996-b65e-3733cfcca10c.jpeg)

在域的XML文件中，有一部分是用于配置域的元数据，即meta data。元数据的概念我们不陌生，在讲解存储虚拟化时介绍过，它主要用来描述数据的属性。在KVM虚拟机中域的元数据就表示域的属性，主要用于区别其他的域。其中，name用于表示该虚拟机的名称，uuid是唯一标识该虚拟机的UUID。在同一个宿主机上，各个虚拟机的名称和UUID都必须是唯一的。除此之外，还有其他的配置，在后续讲解操作实例时遇到了我们还会介绍，这里不再列举。

### **QEMU模拟器配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017255746-7ac24b67-fa53-48e6-b04e-0b8116c04229.jpeg)

在KVM虚拟机的XML配置文件中，需要制定使用的设备模型的模拟器。如上图，在emulator标签中配置模拟器的绝对路径为/usr/libexec/qemu-kvm。如果我们自己下载最新的QEMU源码编译了一个QEMU模拟器，要使用的话，需要将这里修改为/usr/local/bin/qemu-system-x86_64。不过，创建虚拟机时，**可能会遇到error: internal error Process exited while reading console log output错误**，这是因为自己编译的QEMU模拟器不支持配置文件中的pc-i440fx-rhel7.0.0机器类型，因此，需要同步修改如下选项：

复制

<type arch='x86_64' machine='pc'>hvm</type>

### **图形显示方式配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017255926-de9c9522-63de-4cde-ba0d-03d66cc1ec25.jpeg)

如上图，表示通过VNC的方式连接到客户机，其VNC端口为libvirt自动分配，也就是用VNC客户端模拟虚拟机的显示器。除此之外，也支持其他多种类型的图形显示方式，以下示例代码就表示就配置了SDL、VNC、RDP、SPICE等多种客户机显示方式。

复制

<graphics type='sdl' display=':0.0'/> <graphics type='vnc' port='5904'>   <listen type='address' address='1.2.3.4'/> </graphics> <graphics type='rdp' autoport='yes' multiUser='yes' /> <graphics type='desktop' fullscreen='yes'/> <graphics type='spice'>   <listen type='network' network='rednet'/> </graphics>

### **虚拟机的声卡和显卡配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017256159-5662bdfb-0a3a-4617-9740-297cd25c144a.jpeg)

如上图，**标签表示的是显卡配置**，其中子标签表示为虚拟机模拟的显卡的类型，它的类型（type）属性可以为vga、cirrus、vmvga、xen、vbox、qxl中的一个，vram属性表示虚拟显卡的显存容量（单位为KB），heads属性表示显示屏幕的序号。在我们的虚拟机中，显卡的配置为cirrus类型、显存为16384（即16MB）、使用在第1号屏幕上。

由于我们的虚拟机中没有配置声卡，也就没有**标签**，即使有也非常简单，只需要了解他的model属性即可，也就是声卡的类型，常用的选项有es1370、sb16、ac97和ich6。

### **串口和控制台配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017256149-648b64f4-5302-4d69-ae95-b37d42188e46.jpeg)

如上图，设置了虚拟机的编号为0的串口（即/dev/ttyS0），使用宿主机中的伪终端（pty），由于这里没有指定使用宿主机中的哪个伪终端，因此libvirt会自己选择一个空闲的伪终端（可能为/dev/pts/下的任意一个），如下伪终端均为字符型设备：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017256572-ef218bb5-b729-45a2-8374-afc8e13a07b8.jpeg)

除了系统默认指定外，也可以加上配置来明确指定使用宿主机中的哪一个虚拟终端。在部署安装虚拟机virt-install命令增加–extra-args ‘console=ttyS0,115200n8 serial’来指定。

通常情况下，控制台（console）配置在客户机中的类型为’serial’，此时，如果没有配置串口（serial），则会将控制台的配置复制到串口配置中，如果已经配置了串口（我们前面安装的虚拟机ubt1204d就是如此），则libvirt会忽略控制台的配置项。同时，为了让控制台有输出信息并且能够与虚拟机交互，也需在虚拟机中配置将信息输出到串口。比如，在Linux虚拟机内核的启动行中添加“console=ttyS0”这样的配置。

### **输入设备配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017256616-68cbe965-2740-43f5-bd20-60f6e7c0ada0.jpeg)

如上图，这里的配置会让QEMU模拟PS2接口的鼠标和键盘，还提供了tablet这种类型的设备，即模拟USB总线类型的键盘和鼠标，并能让光标可以在客户机获取绝对位置定位。

### **PCI控制器配置**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017256886-d1ab1514-7ac0-428b-a91f-b626b05e10c1.jpeg)

如上图，libvirt会根据虚拟机的不同架构，默认会为虚拟机模拟一些必要的PCI控制器，这类PCI控制器不需要在XML文件中指定，而上图中的需要显式指定都是特殊的PCI控制器。这里显式指定了4个USB控制器、1个pci-root和1个idel控制器。libvirt默认还会为虚拟机分配一些必要的PCI设备，如PCI主桥（Host bridge）、ISA桥等。使用上面的XML配置文件启动虚拟机，在客户机中查看到的PCI信息如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017256957-a8cef4c9-65e6-4634-9160-c8a109fb1383.jpeg)

## **使用libvirt API进行虚拟化管理**

要使用libvirt API进行虚拟化管理，就必须先建立到Hypervisor的连接，有了连接才能管理节点、Hypervisor、域、网络等虚拟化要素。对于libvirt连接，可以简单理解为C/S架构模式，服务器端运行Hypervisor，客户端通过各种协议去连接服务器端的Hypervisor，然后进行相应的虚拟化管理。前面提到过，要实现这种连接，**前提条件是libvirtd这个守护进行必须处于运行状态。但是，这里面有个例外，那就是VMware ESX/ESXi就不需要在服务器端运行libvirtd，依然可以通过libvirt客户端以另外的方式连接到VMware。**

**为了区分不同的连接，libvirt使用了在互联网应用中广泛使用的URI（Uniform Resource Identifier，统一资源标识符）来标识到某个Hypervisor的连接。**libvirt中连接的标识符URI，其本地URI和远程URI是有一些区别的，具体如下：

**1）在libvirt的客户端使用本地的URI连接本系统范围内的Hypervisor，本地URI的一般格式如下：**

复制

driver[+transport]:///[path][?extral-param]

其中，driver是连接Hypervisor的驱动名称（如qemu、xen、xbox、lxc等），transport是选择该连接所使用的传输方式，可以为空；path是连接到服务器端上的某个路径，？extral-param是可以额外添加的一些参数（如Unix domain sockect的路径）。

**2）libvirt可以使用远程URI来建立到网络上的Hypervisor的连接。远程URI和本地URI是类似的，只是会增加用户名、主机名（或IP地址）和连接端口来连接到远程的节点。远程URI的一般格式如下：**

复制

driver[+transport]://[user@][host][:port]/[path][?extral-param]

其中，transport表示传输方式，其取值可以是ssh、tcp、libssh2等；user表示连接远程主机使用的用户名，host表示远程主机的主机名或IP地址，port表示连接远程主机的端口。其余参数的意义与本地URI中介绍的完全一样。

无论本地URI还是远程URI，在libvirt中KVM使用QEMU驱动，而QEMU驱动是一个多实例的驱动，它提供了一个root用户的实例system和一个普通用户的实例session，来设定客户端连接到服务器端后，操作权限的范围，类似Linux系统中的root用户与普通用户的区别。使用root用户实例连接的客户端，拥有最大权限，可以查询和控制整个节点范围虚拟机以及相关设备等系统资源；使用普通用户实例连接的客户端，只拥有服务器端对应用户的操作权限。那么，我们通过这一点也就知道**libvirtd也是具备分权分域虚拟化管理能力的。**

有了上面的知识储备，我们就可以使用URI建立到Hypervisor的连接，比如，我们在252机器上，通过SSH方式连接掉251机器上，查看251机器上的KVM虚拟机信息，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017257132-d8ba1d34-7c75-4acf-a191-cca61ee90dd0.jpeg)

也可以在251机器上创建本地URI连接，进行管理，大家参照自行练习，我就懒得截图了，谁让我佛系嘛。。。

我们前面提到过，virsh等工具都是对libvirt的封装和调用，所以我们上面的指令看似简单，但是对应libvirt底层就不是那么回事了。在libvirt的底层是由**virConnectOpen函数**来建立到Hypervisor的连接的，这个函数需要一个URI作为参数，当传递给virConnectOpen的URI为空值（NULL）时，libvirt会依次根据如下3条规则去决定使用哪一个URI。

**1）试图使用LIBVIRT_DEFAULT_URI这个环境变量。**

**2）试用使用客户端的libvirt配置文件中的uri_default参数的值。**

**3）依次尝试用每个Hypervisor的驱动去建立连接，直到能正常建立连接后即停止尝试。**

如果这3条规则都不能够让客户端libvirt建立到Hypervisor的连接，就会报出建立连接失败的错误信息**（“failed to connect to the hypervisor”）。**

除了针对QEMU、Xen、LXC等真实Hypervisor的驱动之外，**libvirt自身还提供了一个名叫“test”的傀儡Hypervisor及其驱动程序。**test Hypervisor是在libvirt中**仅仅用于测试和命令学习的目的**，因为在本地的和远程的Hypervisor都连接不上（或无权限连接）时，test这个Hypervisor却一直都会处于可用状态。使用virsh连接到test Hypervisor的示例操作如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017257684-c77d2f22-5431-47c6-b948-25799bcc2b1d.jpeg)

如上图，输入help后，就会出现各种命令行指令，新手可以那这个test虚拟机来进行指令练习和学习，不会影响正常的虚拟机，类似Linux中namespace对所有资源拷贝一份然后与真实资源隔离。

下面，我们通过Python来调用libvirt API查询虚拟机信息，在使用Python调用libvirt API之前，需要确定系统是否安装了libvirt-Python，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017257654-b83a2771-b017-4e43-8a30-4c2994d874c6.jpeg)

上图结果表示系统已经安装libvirt-Python。否则，需要手动安装或自行编译安装相应基础包。

完成上面的确认后，我们写一个Python小程序脚本GetinfoDm.py，通过调用libvirt的Python API来查询虚拟机的一些信息。代码如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017257670-11867353-c4ac-4817-aa96-b6126e142263.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1645017257687-24aae452-9c02-4f62-984f-760a90cd4a73.jpeg)



上面的脚本只是简单地调用libvirt Python API获取一些信息，需要注意的是“import libvirt”语句引入了libvirt.py这个API文件，然后才能够使用libvirt.openReadOnly、conn.lookupByName等libvirt中的方法。在本示例中，引入的libvirt.py这个API文件的绝对路径是/usr/lib64/python2.7/site-packages/libvirt.py，它实际调用的是/usr/lib64/python2.7/site-packages/libvirtmod.so这个共享库文件。运行上面的脚本后，结果如下：



通过上面的示例，我们知道无论是virsh等命令还是Python等高级语言，在对KVM虚拟机进行操作时都是调用libvirt的API库函数来完成，这就说明libvirt API是libvirt管理虚拟机的核心。在libvirt中，外部应用接口API函数大致分为8类，是libvirt实现虚拟化管理的基石。在本文的最后，我们通过一张表对其中最常用的6类API进行总结，方便我们后续编写自动化脚本时查询。

| **库函数类别**                                               | **功能说明**                                                 | **常用函数**                                                 |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 连接Hypervisor相关API，以virConnect开头的系列函数            | 只有在与Hypervisor建立连接之后，才能进行虚拟机管理操作，所以连接Hypervisor的API是其他所有API使用的前提条件。与Hypervisor建立的连接为其他API的执行提供了路径，是其他虚拟化管理功能的基础。 | virConnetcOpen()：建立一个连接，返回一个virConnectPtr对象。virConnetcReadOnly()：建立一个只读连接，只能使用查询功能。virConnectOpenAuth()：根据认证建立安全连接。virConnectGetCapabilities()：返回对Hypervisor和驱动的功能描述XML字符串。virConnectListDomains()：返回一系列域标识符，只返回活动域信息。 |
| 域管理API，以virDomain开头的系列函数                         | 要管理域，首先要获取virDomainPtr这个域对象，然后才能对域进行操作。 | virDomainLookupByID(virConnectPtr conn,int id)：根据域的id值到conn连接上去查找相应的域。virDomainLookupByName(virConnectPtr conn,string name)：根据域的名称去conn连接上查找相应的域。virDomainLookupByUUID(virConnectPtr conn,string uuid)：根据域的UUID去conn连接上查找相应的域。virDomainGetHostname()：获取相应域的主机名。virDomainGetinfo()：获取相应域的信息。virDomainGetVcpus()：获取相应域vcpu信息。virDomainCreate()：创建域。virDomainSuspend()：挂起域。virDomainResume()：恢复域等等。。。 |
| 节点管理API，以virNode开头的系列函数                         | 节点管理的多数函数都需要使用一个连接Hypervisor的对象作为其中的一个传入参数，以便可以查询或修改该连接上的节点信息。 | virNodeGetInfo()：获取节点的物理硬件信息。virNodeGetCPUStats()：获取节点上各个CPU的使用统计信息virNodeGetMemoryStats()：获取节点上内存使用统计信息virNodeGetFreeMemory()：获取接上空闲内存信息virNodeSetMemoryParameters()：设置节点上内存调度参数virNodeSuspendForDurarion()：控制节点主机运行 |
| 网络管理API，以virNetwork开头的系列函数和部分virInterface开头系列函数 | libvirt首先需要创建virNetworkPtr对象，然后才能查询或控制虚拟网络。 | virNetworkGetName()：获取网络名称。virNetworkGetBridgeName()：获取网桥名称。virNetworkGetUUID()：获取网络UUID标识。virNetWorkGetXMLDesc()：获取网络以XML格式描述信息。virNetworkIsActive()：查询网络是否在用。virNetworkCreateXML()：根据XML格式创建一个网络。virNetworkDestroy()：注销一个网络。virNetworkUpdate()：更新一个网络。virInterfaceCreate()：创建一个网络端口。。。。等等 |
| 存储卷管理API，以virStorageVol开头的系列函数                 | libvirt对存储卷的管理，首先需要创建virStorageVolPtr这个存储卷对象，然后才能对其进行查询或控制操作。 | virStorageVolLookupByKey()：根据全局唯一键值获取一个存储卷对象。virStorageVolLookupByName()：根据名称在一个存储资源池获取一个存储卷对象。virStorageVolLookupByPath()：根据节点上路径获取一个存储卷对象。virStorageVolGetInfo()：查询某个存储卷的使用信息。virStorageVolGetName()：获取存储卷的名称。。。。等等 |
| 存储池管理API，以virStoragePool开头的系列函数                | libvirt对存储池（pool）的管理包括对本地的基本文件系统、普通网络共享文件系统、iSCSI共享文件系统、LVM分区等的管理。libvirt需要基于virStoragePoolPtr这个存储池对象才能进行查询和控制操作。 | virStoragePoolLookupByName()：可以根据存储池的名称来获取一个存储池对象。virStoragePoolLookupByVolume()：可以根据一个存储卷返回其对应的存储池对象。virStoragePoolCreateXML()：可以根据XML描述来创建一个存储池（默认已激活）virStoragePoolDefineXML()：可以根据XML描述信息静态地定义一个存储池（尚未激活）virStoragePoolCreate()：可以激活一个存储池。virStoragePoolIsActive()：可以查询存储池状态是否处于使用中。。。。等等 |


 
