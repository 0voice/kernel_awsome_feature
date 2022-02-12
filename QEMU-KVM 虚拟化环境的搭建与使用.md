## QEMU/KVM 虚拟化

QEMU/KVM 是目前最流行的虚拟化技术，它基于 Linux 内核提供的 kvm 模块，结构精简，性能损失小，而且开源免费（对比收费的 vmware），因此成了大部分企业的首选虚拟化方案。

目前各大云厂商的虚拟化方案，新的服务器实例基本都是用的 KVM 技术。即使是起步最早，一直重度使用 Xen 的 AWS，从 EC2 C5 开始就改用了基于 KVM 定制的 Nitro 虚拟化技术。

但是 KVM 作为一个企业级的底层虚拟化技术，却没有对桌面使用做深入的优化，因此如果想把它当成桌面虚拟化软件来使用，替代掉 VirtualBox/VMware，有一定难度。

本文是我个人学习 KVM 的一个总结性文档，其目标是使用 KVM 作为桌面虚拟化软件。

## 一、安装 QUEU/KVM

QEMU/KVM 环境需要安装很多的组件，它们各司其职：

1. qemu: 模拟各类输入输出设备（网卡、磁盘、USB端口等）
   - qemu 底层使用 kvm 模拟 CPU 和 RAM，比软件模拟的方式快很多。
2. libvirt: 提供简单且统一的工具和 API，用于管理虚拟机，屏蔽了底层的复杂结构。（支持 qemu-kvm/virtualbox/vmware）
3. ovmf: 为虚拟机启用 UEFI 支持
4. virt-manager: 用于管理虚拟机的 GUI 界面（可以管理远程 kvm 主机）。
5. virt-viewer: 通过 GUI 界面直接与虚拟机交互（可以管理远程 kvm 主机）。
6. dnsmasq vde2 bridge-utils openbsd-netcat: 网络相关组件，提供了以太网虚拟化、网络桥接、NAT网络等虚拟网络功能。
   - dnsmasq 提供了 NAT 虚拟网络的 DHCP 及 DNS 解析功能。
   - vde2: 以太网虚拟化
   - bridge-utils: 顾名思义，提供网络桥接相关的工具。
   - openbsd-netcat: TCP/IP 的瑞士军刀，详见 [socat & netcat](https://thiscute.world/posts/socat-netcat/)，这里不清楚是哪个网络组件会用到它。

安装命令：

```shell
# archlinux/manjaro
sudo pacman -S qemu virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat

# ubuntu,参考了官方文档，但未测试
sudo apt install qemu-kvm libvirt-daemon-system virt-manager virt-viewer virtinst bridge-utils

# centos,参考了官方文档，但未测试
sudo yum groupinstall "Virtualization Host"
sudo yum install virt-manager virt-viewer virt-install

# opensuse
# see: https://doc.opensuse.org/documentation/leap/virtualization/html/book-virt/cha-vt-installation.html
sudo yast2 virtualization
# enter to terminal ui, select kvm + kvm tools, and then install it.

```

安装完成后，还不能直接使用，需要做些额外的工作。请继续往下走。

### 1. libguestfs - 虚拟机磁盘映像处理工具

[libguestfs](https://libguestfs.org/) 是一个虚拟机磁盘映像处理工具，可用于直接修改/查看/虚拟机映像、转换映像格式等。

它提供的命令列表如下：

1. `virt-df centos.img`: 查看硬盘使用情况
2. `virt-ls centos.img /`: 列出目录文件
3. `virt-copy-out -d domain /etc/passwd /tmp`：在虚拟映像中执行文件复制
4. `virt-list-filesystems /file/xx.img`：查看文件系统信息
5. `virt-list-partitions /file/xx.img`：查看分区信息
6. `guestmount -a /file/xx.qcow2(raw/qcow2都支持) -m /dev/VolGroup/lv_root --rw /mnt`：直接将分区挂载到宿主机
7. `guestfish`: 交互式 shell，可运行上述所有命令。
8. `virt-v2v`: 将其他格式的虚拟机(比如 ova) 转换成 kvm 虚拟机。
9. `virt-p2v`: 将一台物理机转换成虚拟机。

学习过程中可能会使用到上述命令，提前安装好总不会有错，安装命令如下：

```sh
# opensuse
sudo zypper install libguestfs

# archlinux/manjaro，目前缺少 virt-v2v/virt-p2v 组件
sudo pacman -S libguestfs

# ubuntu
sudo apt install libguestfs-tools

# centos
sudo yum install libguestfs-tools

```



### 2. 启动 QEMU/KVM

通过 systemd 启动 libvirtd 后台服务：

```sh
sudo systemctl enable libvirtd.service
sudo systemctl start libvirtd.service

```



### 3. 让非 root 用户能正常使用 kvm

qumu/kvm 装好后，默认情况下需要 root 权限才能正常使用它。 为了方便使用，首先编辑文件 `/etc/libvirt/libvirtd.conf`:

1. `unix_sock_group = "libvirt"`，取消这一行的注释，使 `libvirt` 用户组能使用 unix 套接字。
2. `unix_sock_rw_perms = "0770"`，取消这一行的注释，使用户能读写 unix 套接字。

然后新建 libvirt 用户组，将当前用户加入该组：

```sh
`newgrp libvirt 
sudo usermod -aG libvirt $USER `
```

最后重启 libvirtd 服务，应该就能正常使用了：

```sh
`sudo systemctl restart libvirtd.service `
```



### 3. 启用嵌套虚拟化

如果你需要**在虚拟机中运行虚拟机**（比如在虚拟机里测试 katacontainers 等安全容器技术），那就需要启用内核模块 kvm_intel 实现嵌套虚拟化。

```sh
# 临时启用 kvm_intel 嵌套虚拟化
sudo modprobe -r kvm_intel
sudo modprobe kvm_intel nested=1
# 修改配置，永久启用嵌套虚拟化
echo "options kvm-intel nested=1" | sudo tee /etc/modprobe.d/kvm-intel.conf

```

验证嵌套虚拟化已经启用：

```sh
$ cat /sys/module/kvm_intel/parameters/nested 
Y

```

至此，KVM 的安装就大功告成啦，现在应该可以在系统中找到 virt-manager 的图标，进去就可以使用了。 virt-manager 的使用方法和 virtualbox/vmware workstation 大同小异，这里就不详细介绍了，自己摸索摸索应该就会了。

------

> 如下内容是进阶篇，主要介绍如何通过命令行来管理虚拟机磁盘，以及 KVM。 如果你还是 kvm 新手，建议先通过图形界面 virt-manager 熟悉熟悉，再往下继续读。

## 二、虚拟机磁盘映像管理

这需要用到两个工具：

1. libguestfs: 虚拟机磁盘映像管理工具，前面介绍过了
2. qemu-img: qemu 的磁盘映像管理工具，用于创建磁盘、扩缩容磁盘、生成磁盘快照、查看磁盘信息、转换磁盘格式等等。

```sh
# 创建磁盘
qemu-img create -f qcow2 -o cluster_size=128K virt_disk.qcow2 20G

# 扩容磁盘
qemu-img resize ubuntu-server-cloudimg-amd64.img 30G

# 查看磁盘信息
qemu-img info ubuntu-server-cloudimg-amd64.img

# 转换磁盘格式
qemu-img convert -f raw -O qcow2 vm01.img vm01.qcow2  # raw => qcow2
qemu-img convert -f qcow2 -O raw vm01.qcow2 vm01.img  # qcow2 => raw

```



### 1. 导入 vmware 镜像

直接从 vmware ova 文件导入 kvm，这种方式转换得到的镜像应该能直接用（网卡需要重新配置）：

```sh
`virt-v2v -i ova centos7-test01.ova -o local -os /vmhost/centos7-01  -of qcow2 `
```

也可以先从 ova 中解压出 vmdk 磁盘映像，将 vmware 的 vmdk 文件转换成 qcow2 格式，然后再导入 kvm（网卡需要重新配置）：

```sh
# 转换映像格式
qemu-img convert -p -f vmdk -O qcow2 centos7-test01-disk1.vmdk centos7-test01.qcow2
# 查看转换后的映像信息
qemu-img info centos7-test01.qcow2

```

直接转换 vmdk 文件得到的 qcow2 镜像，启会报错，比如「磁盘无法挂载」。 根据 [Importing Virtual Machines and disk images - ProxmoxVE Docs](https://pve.proxmox.com/pve-docs/chapter-qm.html#_importing_virtual_machines_and_disk_images) 文档所言，需要在网上下载安装 MergeIDE.zip 组件， 另外启动虚拟机前，需要将硬盘类型改为 IDE，才能解决这个问题。

### 2. 导入 img 镜像

img 镜像文件，就是所谓的 raw 格式镜像，也被称为裸镜像，IO 速度比 qcow2 快，但是体积大，而且不支持快照等高级特性。 如果不追求 IO 性能的话，建议将它转换成 qcow2 再使用。

```sh
`qemu-img convert -f raw -O qcow2 vm01.img vm01.qcow2 `
```



## 三、虚拟机管理

虚拟机管理可以使用命令行工具 `virsh`/`virt-install`，也可以使用 GUI 工具 `virt-manager`.

GUI 很傻瓜式，就不介绍了，这里主要介绍命令行工具 `virsh`/`virt-install`

先介绍下 libvirt 中的几个概念：

1. Domain: 指代运行在虚拟机器上的操作系统的实例 - 一个虚拟机，或者用于启动虚拟机的配置。
2. Guest OS: 运行在 domain 中的虚拟操作系统。

大部分情况下，你都可以把下面命令中涉及到的 `domain` 理解成虚拟机。

### 0. 设置默认 URI

`virsh`/`virt-install`/`virt-viewer` 等一系列 libvirt 命令，sudo virsh net-list –all 默认情况下会使用 `qemu:///session` 作为 URI 去连接 QEMU/KVM，只有 root 账号才会默认使用 `qemu:///system`.

另一方面 `virt-manager` 这个 GUI 工具，默认也会使用 `qemu:///system` 去连接 QEMU/KVM（和 root 账号一致）

`qemu:///system` 是系统全局的 qemu 环境，而 `qemu:///session` 的环境是按用户隔离的。 另外 `qemu:///session` 没有默认的 `network`，创建虚拟机时会出毛病。。。

因此，你需要将默认的 URI 改为 `qemu:///system`，否则绝对会被坑:

```sh
`echo 'export LIBVIRT_DEFAULT_URI="qemu:///system"' >> ~/.bashrc `
```



### 1. 虚拟机网络

qemu-kvm 安装完成后，`qemu:///system` 环境中默认会创建一个 `default` 网络，而 `qemu:///session` 不提供默认的网络，需要手动创建。

我们通常使用 `qemu:///system` 环境就好，可以使用如下方法查看并启动 default 网络，这样后面创建虚拟机时才有网络可用。

```sh
# 列出所有虚拟机网络
$ sudo virsh net-list --all
 Name      State      Autostart   Persistent
----------------------------------------------
 default   inactive   no          yes

# 启动默认网络
$ virsh net-start default
Network default started

# 将 default 网络设为自启动
$ virsh net-autostart --network default
Network default marked as autostarted

# 再次检查网络状况，已经是 active 了
$ sudo virsh net-list --all
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes

```

也可以创建新的虚拟机网络，这需要手动编写网络的 xml 配置，然后通过 `virsh net-define --file my-network.xml` 创建，这里就不详细介绍了，因为暂时用不到…

### 2. 创建虚拟机 - virt-intall

```sh
# 使用 iso 镜像创建全新的 proxmox 虚拟机，自动创建一个 60G 的磁盘。
virt-install --virt-type kvm \
--name pve-1 \
--vcpus 4 --memory 8096 \
--disk size=60 \
--network network=default,model=virtio \
--os-type linux \
--os-variant generic \
--graphics vnc \
--cdrom proxmox-ve_6.3-1.iso

# 使用已存在的 opensuse cloud 磁盘创建虚拟机
virt-install --virt-type kvm \
  --name opensuse15-2 \
  --vcpus 2 --memory 2048 \
  --disk opensuse15.2-openstack.qcow2,device=disk,bus=virtio \
  --disk seed.iso,device=cdrom \
  --os-type linux \
  --os-variant opensuse15.2 \
  --network network=default,model=virtio \
  --graphics vnc \
  --import

```

其中的 `--os-variant` 用于设定 OS 相关的优化配置，官方文档**强烈推荐**设定，其可选参数可以通过 `osinfo-query os` 查看。

### 3. 虚拟机管理 - virsh

虚拟机创建好后，可使用 virsh 管理虚拟机：

查看虚拟机列表：

```sh
# 查看正在运行的虚拟机
virsh list

# 查看所有虚拟机，包括 inactive 的虚拟机
virsh list --all

```

使用 `virt-viewer` 以 vnc 协议登入虚拟机终端：

```sh
# 使用虚拟机 ID 连接
virt-viewer 8
# 使用虚拟机名称连接，并且等待虚拟机启动
virt-viewer --wait opensuse15

```

启动、关闭、暂停(休眠)、重启虚拟机：

```sh
virsh start opensuse15
virsh suuspend opensuse15
virsh resume opensuse15
virsh reboot opensuse15
# 优雅关机
virsh shutdown opensuse15
# 强制关机
virsh destroy opensuse15

# 启用自动开机
virsh autostart opensuse15
# 禁用自动开机
virsh autostart --disable opensuse15

```

虚拟机快照管理：

```sh
# 列出一个虚拟机的所有快照
virsh snapshot-list --domain opensuse15
# 给某个虚拟机生成一个新快照
virsh snapshot-create <domain>
# 使用快照将虚拟机还原
virsh snapshot-restore <domain> <snapshotname>
# 删除快照
virsh snapshot-delete <domain> <snapshotname>

```

删除虚拟机：

```sh
`virsh undefine opensuse15 `
```

迁移虚拟机：

```sh
# 使用默认参数进行离线迁移，将已关机的服务器迁移到另一个 qemu 实例
virsh migrate 37 qemu+ssh://tux@jupiter.example.com/system
# 还支持在线实时迁移，待续

```

cpu/内存修改：

```sh
# 改成 4 核
virsh setvcpus opensuse15 4
# 改成 4G
virsh setmem opensuse15 4096

```

虚拟机监控：

```sh
 ``# 待续 `
```

修改磁盘、网络及其他设备：

```sh
# 添加新设备
virsh attach-device
virsh attach-disk
virsh attach-interface
# 删除设备
virsh detach-disk
virsh detach-device
virsh detach-interface

```



## 参考

- [Virtualization Guide - OpenSUSE](https://doc.opensuse.org/documentation/leap/virtualization/html/book-virt/index.html)
- [Complete Installation of KVM, QEMU and Virt Manager on Arch Linux and Manjaro](https://computingforgeeks.com/complete-installation-of-kvmqemu-and-virt-manager-on-arch-linux-and-manjaro/)
- [virtualization-libvirt - ubuntu docs](https://ubuntu.com/server/docs/virtualization-libvirt)
- [RedHat Docs - KVM](https://developers.redhat.com/products/rhel/hello-world#fndtn-kvm)
