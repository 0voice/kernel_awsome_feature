## **virt-instal工具简介**

virt-install是一个命令行工具，它能够为KVM、Xen或其它支持libvrit API的hypervisor创建虚拟机并完成GuestOS安装；此外，它能够基于串行控制台、VNC或SDL支持文本或图形安装界面。安装过程可以使用本地的安装介质如CDROM，也可以通过网络方式如NFS、HTTP或FTP服务实现。对于通过网络安装的方式，virt-install可以自动加载必要的文件以启动安装过程而无须额外提供引导工具。当然，virt-install也支持PXE方式的安装过程，也能够直接使用现有的磁盘映像直接启动安装过程。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659332833-a385e195-92a5-4f10-84a2-8f8f3bc53abd.jpeg)

virt-install命令有许多选项，这些选项大体可分为下面几大类，同时对每类中的常用选项也做出简单说明。

| **普通选项**    |                                  |                                                              |
| --------------- | -------------------------------- | ------------------------------------------------------------ |
| **选项**        | **子选项**                       | **说明**                                                     |
| -n或–name       |                                  | 虚拟机名称，全局唯一                                         |
| -r或–ram        |                                  | 虚拟机内存大小，单位为MB                                     |
| –vcpus          | maxvcpu，sockets，cores，threads | vCPU的个数及相关配置                                         |
| –cpu            |                                  | CPU模式及特性，可以使用qemu-kvm -cpu ? 来获取支持的类型      |
| 安装方法选项    |                                  |                                                              |
| 选项            | 子选项                           | 说明                                                         |
| -c或–cdrom      |                                  | 光盘安装介质                                                 |
| -l或–location   |                                  | 安装源URL，支持FPT，HTTP及NFS，如:[ftp://172.0.6.1/pub](ftp://172.0.6.1/pub) |
| –pxe            |                                  | 基于PXE完成安装                                              |
| –livecd         |                                  | 把光盘当做启动引导CD                                         |
| –os-type        |                                  | 操作系统类型，如linux、windows或unix等                       |
| –os-variant     |                                  | 某类型操作系统的发行版，如rhel7、Ubuntud等                   |
| -x或–extra-args |                                  | 根据–location指定的方式安装GuestOS时，用于传递给内核的参数选项，例如指定kickstart文件的位置。 |
| –boot           |                                  | 指定安装过程完成后的配置选项，如指定引导设备的次序、使用指定的而非安装kernel/initrd来引导系统。比如：–boot cdrom、hd、network：指定引导次序分别为光驱、硬盘和网络；–boot kernel=KERNEL,initrd=INITRD,kernel_args=”console=/dev/ttyS0”：指定启动系统的内核及initrd文件，并创建一个模拟终端 |
| **存储配置**    |                                  |                                                              |
| **选项**        | **子选项**                       | **说明**                                                     |
| –disk           |                                  | 指定存储设备的路径及其属性                                   |
|                 | device                           | 设备类型，如cdrom、disk或floppy等，默认为disk                |
|                 | bus                              | 磁盘总线类型，可以为ide、scsi、usb、virtio或xen              |
|                 | perms                            | 访问权限，如rw、ro或sh（共享可读写），默认为rw               |
|                 | size                             | 新建磁盘镜像大小，单位为GB                                   |
|                 | cache                            | 缓存类型，可以为none、writethrouth及writeback                |
|                 | format                           | 磁盘镜像格式，如raw、qcow2、vmdk等                           |
|                 | sparse                           | 磁盘镜像的存储数据格式为稀疏格式，即不立即分配指定大小的空间 |
| –nodisks        |                                  | 不适用本地磁盘，在LiveCD模式中常用                           |
| **网络配置**    |                                  |                                                              |
| **选项**        | **子选项**                       | **说明**                                                     |
| -w或–network    |                                  | 将虚拟机连入虚拟网络中                                       |
|                 | bridge=BRIDGE                    | 虚拟网络为名称为BRIDGE的网桥设备                             |
|                 | network=NAME                     | 虚拟网络为名称NAME的网络                                     |
|                 | model                            | GuestOS中看见的虚拟网络设备类型                              |
|                 | mac                              | 配置固定的MAC地址，省略此选项时，MAC地址随机分配，但是无论何种方式，KVM的网络设备的MAC地址前三段必须为52:54:00 |
| –nonetworks     |                                  | 虚拟机不使用网络                                             |
| **图形配置**    |                                  |                                                              |
| **选项**        | **子选项**                       | **说明**                                                     |
| –graphics       |                                  | 指定图形显示相关的配置，此选项不会配置任何硬件，而是指定虚拟机启动后对其访问的图形界面接口 |
|                 | TYPE                             | 指定显示类型，可以为vnc，spice、sdl或none                    |
|                 | port                             | TYPE为vnc或spice时其监听的端口                               |
|                 | listen                           | TYPE为vnc或spice时其监听的IP地址，默认为127.0.0.1，可以通过修改/etc/libvirt/qemu.conf定义新的默认值 |
|                 | password                         | TYPE为vnc或spice时为远程访问监听的服务指定认证密码           |
| –noautoconsole  |                                  | 禁止自动连接到虚拟机的控制台                                 |
| **设备选项**    |                                  |                                                              |
| **选项**        | **子选项**                       | **说明**                                                     |
| –serial         |                                  | 附加一个串行设备到当前虚拟机，根据设备类型不同，可以使用不同的选项，格式为–serial type,opt1=val1,opt2=val2 |
|                 | pty                              | 创建伪终端                                                   |
|                 | dev,path=HOSTPATH                | 附加主机设备至此虚拟机                                       |
| –video          |                                  | 指定显卡设备类型，如cirrus、vga、qxl或vmvga                  |
| **虚拟化平台**  |                                  |                                                              |
| **选项**        | **子选项**                       | **说明**                                                     |
| -v或–hvm        |                                  | 当宿主机同时支持全虚和半虚时，使用此选项指定硬件辅助的全虚   |
| -p或–paravirt   |                                  | 指定使用半虚                                                 |
| –virt-type      |                                  | 指定使用的hypervisor，如kvm、xen、qemu等，其值可以通过virsh capabilities获得 |
| **其它**        |                                  |                                                              |
| **选项**        | **子选项**                       | **说明**                                                     |
| –autostart      |                                  | 指定虚拟机是否在物理机启动后自动启动                         |
| –print-xml      |                                  | 如果虚拟机不需要安装过程(–import、–boot)，则显示生成的XML而不是创建虚拟机。默认情况下，此选项仍会创建虚拟磁盘 |
| –force          |                                  | 禁止进入命令交互模式，如需回答yes或no，默认自动回答yes       |
| –dry-run        |                                  | 执行创建虚拟机的整个过程，但不创建虚拟机，改变主机上的设备配置信息及将其创建的需求通知给libvirt； |
| -d或–debug      |                                  | 显示debug信息；                                              |

**尽管virt-install命令有着类似上述的众多选项，但实际使用中，其必须提供的选项仅包括–name、–ram、–disk（也可是–nodisks）及安装过程相关的选项。此外，有时还需要使用括–connect=CONNCT选项来指定连接至一个非默认的hypervisor。**

## **图形化界面安装**

这里的使用图形化界面不是通过virt-manager和virt-view工具在图形化服务器或宿主机来安装，而是通过–graphics选项指定vnc或spice方式安装，安装过程中需要宿主机通过vnc client连接图形化接口实现。在我们的KVM实践初步一文中介绍安装ubuntu1204桌面版虚拟机就是采用这种方式。忘了的，可以回顾本站那篇文章。

现在，我们这里演示的是另一种图形化安装方法，还是通过vnc实现，安装介质使用ubuntu18.10桌面版ISO。如下：

**step1：**我们先查看下当前激活的虚拟网络有哪些，方便后续安装虚拟机时配置网络（图中红色框部分）。同时，我们创建一个qcow2格式的虚拟机磁盘镜像，大小为120G（图中黄色框部分）。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659332720-dc65e77b-0657-497c-b1d7-0fc2018c4534.jpeg)

**step2：**通过virt-install工具安装第一个模板虚拟机ubt1810d。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659332797-bd46c2a2-41d2-4e07-a049-739bcc2c0629.jpeg)

上图中，我们指定了虚拟机当前vcpu为2个，最大可分配4个（黄色框部分），同时指定了虚拟机的镜像磁盘格式为qcow2格式（采用qcow2格式，此项必须明确指定，并且通过size选项指定大小，否则生成的磁盘大小异常，或者在创建镜像磁盘时通过-o选项明确指定也可以），总线类型为virtio类型（图中蓝色框部分）

**step3：**通过vncdisplay命令查询当前虚拟机图形界面接口服务vnc监听的端口号，并通过物理机的vnc client连接开始安装。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659332687-21b25e2d-9396-4549-b758-b61ed7f87311.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659332920-0ae14a87-33d2-45e3-aa7f-9066ee178eb6.jpeg)

**step4：**完成虚拟机系统安装后，我们通过vnc client登录虚拟机，打开虚拟机的console控制接口，便于后续通过宿主机console连接虚拟机控制台。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659333364-f035c64f-c712-4278-b85b-59621413cc38.jpeg)

**以上，就是通过图形界面接口实现kvm虚拟机的安装过程。一般通过图形界面接口安装，主要用于安装桌面版的操作系统，比如上面的ubuntu desktop版本，windows的各种版本等。**由于ubuntu系统默认不开启root用户的ssh登录，为了方便后续管理，我们可以在ubuntu系统打开ssh的root登录权限。但是，记得首先要设置root的登录密码。这些操作就可以通过宿主机console控制台来完成。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659333507-4d1e1429-467b-447f-ac7a-277fa6a61017.jpeg)

## **文本字符界面安装**

上面的图形化安装方式主要借助vnc或spice客户端来实现图形界面安装接口，主要用于桌面操作系统的安装。在实际运维时，一般用于app的虚拟机大部分采用非桌面系统，而且打开vnc或spice监听端口不便于批量安装。这种情况下，就需要通过文本界面实现自动安装。主要通过-x选项打开一个串行接口终端tty和ttyS0实现。但是，如果使用-x选项安装，安装介质必须使用–location选项，而不能使用–cdrom来完成。此时，有两个选择，一是将iso镜像拷贝到宿主机的某个目录，通过-l（–location）选项指定即可，类似–cdrom方式。另一种是将iso挂载到某个目录，通过http、nfs或ftp方式发布，然后在-l选项指定URL即可。我们这里采用第二种方式的http方式进行安装，如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659333587-99f14df4-7c47-48e7-9c3d-fa25af6f858f.jpeg)

上述安装方式，通过-l选项指定安装介质的挂载目录，通过-x选项安装使用kiskstart文件，以及打开串行终端console，通过–nographics选项指定不采用图形界面方式安装。命令执行后，安装界面如下，且通过kickstart文件自动化安装。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659333507-4792256c-85d4-4f38-af35-b854695d321c.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659333909-4a3a4f41-1469-4a87-af60-1a9e95a7ddd1.jpeg)

以上这种安装方式，也是实际运维中常用的安装方式，同时可以在kickstart文件中指定初始化脚本，这样在系统安装完成后自动完成初始化配置。后续，就可以将此虚拟机作为模板镜像，批量分发。

但是，如果要将当前虚拟机作为模板镜像，需要删除当前虚拟机中的MAC和UUID，同时清空本地规则和系统规则中网卡配置文件，这样后续通过该虚拟机的模板镜像生成的新的虚拟机就会自动生成新的MAC和网卡名称，而不用通过在XML配置文件中手动指定MAC，实现自动化运维。上面的要求，需要在虚拟机中完成以下配置，然后关机作为后续分发虚拟机的模板。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659334181-5d827e9c-6046-4eea-b862-aa81217e6191.jpeg)

**虚机模板方式安装**

模板方式安装一般有两种方式：一种是通过–import选项导入镜像虚拟机的磁盘来生成一个新的虚拟机，这种方式只能生成一个新虚拟机，当要生成第二个虚拟机时会提示磁盘被占用错误。另一种方式就是利用模板镜像虚拟机的XML文件，生成新的虚拟机的XML配置文件，需要删除源模板镜像虚拟机XML文件中的uuid、mac address，并修改磁盘路径为新虚拟机的磁盘（差分盘或新磁盘都可），而且需要修改XML文件中的虚拟机name为新建虚拟机name。完成后，再通过virsh define预定义一个新虚拟机，然后通过virsh start启动即可。下面，我们就对这两种方式逐一进行验证。

**1）通过–import选项导入现有虚拟机磁盘生成一个新的虚拟机**

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659334201-781ad2b7-3fba-4e40-87e2-61bb8f94f621.jpeg)

完成上面的操作后，我们就能立即生成一个新的虚拟机cirros01。然后，我们可以通过virsh console命令登录并执行相关操作。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659334236-1bf82ed8-7b31-4fd8-be93-30fcaa200c3c.jpeg)

此时，我们如果需要再通过上面镜像盘生成一个虚拟机时，会提示镜像盘被占用的错误。如下：

因此，这种方式只能生成一个新虚拟机，一般用于很小的初始化配置测试。所以，在实际运维中，主要通过第二种方式利用模板镜像虚拟机来批量创建虚拟。

**2）通过模板镜像虚机利用virsh-define和virsh-start实现新虚机创建**

首先，我们利用模板镜像虚机磁盘创建一个差分磁盘，用于新虚机的系统盘（也可以创建非差分盘，看实际业务需求而定）。如下，通过-b选项指定backing file为模板镜像虚机的磁盘。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659334230-2d3dbe47-f5c8-418f-8b05-3589671e3a6b.jpeg)

然后，利用模板镜像虚机的xml文件生成新虚机的xml配置文件。需要删除uuid，mac address配置，修改磁盘存储位置为新创建的虚机差分盘，修改name为新虚机的name。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659334701-a528fef5-199a-491c-b3ce-d17773f841d2.jpeg)

然后，通过virsh-define定义新虚机，再通过virsh-start启动新虚机，然后通过console控制台进入虚机。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659334802-f57213f3-9119-4d8f-a608-017505665c36.jpeg)

如下图，可以发现系统为我们自动生成mac和ip，而且mac必然是52:54:00开头。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659334927-3cb6b7a8-5a86-48ba-8ba4-4ffbdffc705b.jpeg)

而且，新生成的虚拟机具备访问外网和宿主机的权利。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659334945-83aae98c-0c2b-422a-b520-437f3dd0953c.jpeg)

## **批量创建KVM虚拟机的脚本**

掌握上面各种创建KVM虚拟机的方法后，我们可以写一个shell脚本来完成批量创建虚拟机的任务。

首先，我们需要准备创建虚拟机的镜像磁盘，主要有两种方法：一种是拷贝模板虚拟机的镜像磁盘，这种主要用于创建独立虚拟机，优点是不依赖于模板虚拟机磁盘镜像，可以完整保留数据。缺点就是占用存储空间较大。另一种方式就是我们上面的通过建立差分磁盘实现，优点是存储空间占用较小，缺点就是模板镜像磁盘一旦损坏将造成所有依赖虚拟机无法启动。

完成虚拟磁盘创建后，我们就可以按照上面办法通过模板镜像虚机的XML配置文件批量创建新的虚拟机XML配置文件，并修改里面的NAME、UUID、MAC ADDRESS和DISK PATH等配置项。最终，通过define和start命令完成新虚拟机的定义和启动。

通过上面的分析，要实现KVM虚拟机批量创建shell脚本，只需要通过两个for循环就能完成。同时，在程序开始要检查程序脚本的执行权限，从管理角度来说非root用户不能执行。还要给脚本进行传参，用来设定需要批量拉起的KVM虚拟机的上限值和下限值，也就是确定批量拉起的虚机个数。那么，传递的参数就必须是2个，不能多也不能少，且两个参数都必须是数字，参数1的值要小于参数2。

以上，就是我们程序要实现的逻辑，可以采用函数式编程的概念封装成三个函数，实现简单模块化设计。为了讲述清楚，我这里就不封装了。接下来，我们就来一步步写这个脚本。

**Step1：**通过第一个for循环批量创建虚拟机的虚拟磁盘，这里我们采用差分磁盘的方式。如下

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659335161-11714341-7faa-4371-9e1f-af842a5d9088.jpeg)

脚本执行后效果如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659335390-8107fd18-3901-4523-99d9-583f3d327051.jpeg)

**Step2：**通过第二个for循环批量创建虚拟机的XML配置文件，并定义和启动虚拟机。如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659335490-d46fbb7d-fdd3-4284-84f2-680e89de4811.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659335518-66feeac2-29fe-4b64-9873-a06213aec121.jpeg)

完成上面的替换，在脚本执行virsh defiine和virsh start命令定义新的虚拟机并完成启动。脚本执行后效果如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659335648-99dc1b39-c2d2-40b4-ac00-22a312d0b61d.jpeg)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659336009-658f3b72-ff05-40e7-850a-5a385deb47a3.jpeg)

至此，KVM虚拟机的各种创建方法介绍完毕，网上还有一种通过qemu-kvm命令创建虚拟机的方法，与通过virt-install方法创建大同小异，且红帽官方并不建议这种方法。因此，只要掌握virt-install方法创建虚拟机即可。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/21375641/1644659336204-65d9e3f0-b9eb-403f-bd12-1d08ce8b9910.jpeg)


 
