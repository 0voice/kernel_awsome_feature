# 🔰 深入研究 `kvm`,`ceph`,`fuse`,`virtio`,`vhost` 特性，包含开源项目，代码案例，文章，视频，架构脑图等

所有数据来源于互联网。所谓取之于互联网，用之于互联网。

如果涉及版权侵犯，请邮件至 wchao_isvip@163.com ，我们将第一时间处理。

如果您对我们的项目表示赞同与支持，欢迎您 lssues我们，或者邮件 wchao_isvip@163.com 我们，更加欢迎您 pull requests 加入我们。

感谢您的支持！


## 🔥 kvm

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633416-9961e8b7-ff81-488b-8cfe-b69ba739c1ff.png"/>
  
#### —— Linux内核中的虚拟化基础设施
</div>


### 文档
- 官方文档:
  - 官方网址：https://www.linux-kvm.org/page/Main_Page
  - Avi Kivity 在Linux 内核中的邮件: http://lkml.iu.edu/hypermail/linux/kernel/0610.2/1369.html
  - KVM 博客：http://planet.virt-tools.org/
  - KVM 论坛：https://events.linuxfoundation.org/kvm-forum/
- 其他文档：
  - Linux_2_6_20 版本文献：https://kernelnewbies.org/Linux_2_6_20#head-bca4fe7ffe454321118a470387c2be543ee51754
  - kvm源码托管仓库 : https://git.kernel.org/pub/scm/virt/kvm/kvm.git/
  - kvm源码下载：https://sourceforge.net/projects/kvm/files/?source=navbar

### 与虚拟化相关的程序包

- [qemu-kvm](https://www.qemu.org/)：主要的KVM程序包
- [libvirt](https://libvirt.org/)：用于管理超级监视程序的libvirtd服务
  - 代码管理仓： https://gitlab.com/libvirt/libvirt 
- [libvirt-client](https://centos.pkgs.org/7/centos-updates-x86_64/libvirt-client-4.5.0-36.el7_9.3.x86_64.rpm.html)：用于管理虚拟机的virsh命令和客户端API
- [virt-install](https://linux.die.net/man/1/virt-install)：创建虚拟机需要的命令行工具
- [virt-manager](https://virt-manager.org/)：GUI虚拟机管理工具（图形界面）
- [virt-top](https://linux.die.net/man/1/virt-top)：虚拟机统计命令
- [virt-viewer](https://gitlab.com/virt-viewer/virt-viewer)：用于连接到虚拟机的图形控制台

## 图形管理工具
- Kimchi（英语：[Kimchi (software)](https://www.wikiwand.com/en/Kimchi_(software))） – 网页版KVM虚拟化管理工具
- [Virtual Machine Manager](https://www.wikiwand.com/zh-sg/Virtual_Machine_Manager) – 支持创建、编辑、启动与停止基于KVM的虚拟机，同时也支持对宿主之间的实时或冷拖拽虚拟机迁移。
- [Proxmox虚拟环境](https://www.wikiwand.com/zh-sg/Proxmox_VE) – 一项开源的虚拟化管理包，包括KVM与[LXC](https://www.wikiwand.com/zh-sg/LXC)。同时它还有裸机安装器、网页版远程管理界面、HA集群堆栈、统一存储、柔性网络及可选的商业支持。
- OpenQRM（英语：[OpenQRM](https://www.wikiwand.com/en/OpenQRM)） – 用于管理不同数据中心基础设施的平台。
- [GNOME 机柜](https://www.wikiwand.com/zh-sg/GNOME_機櫃) – Linux上用于管理libvirt客户机的Gnome界面。
- oVirt（英语：[oVirt](https://www.wikiwand.com/en/oVirt)） – 用于管理基于libvirt的KVM开源工具。

### 文章

- [KVM 学习笔记](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%20%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.md)
- [KVM与VMware哪个好？如何选择更好的 Hypervisor](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%E4%B8%8EVMware%E5%93%AA%E4%B8%AA%E5%A5%BD%EF%BC%9F%E5%A6%82%E4%BD%95%E9%80%89%E6%8B%A9%E6%9B%B4%E5%A5%BD%E7%9A%84%20Hypervisor.md)
- [KVM之内存虚拟化](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%E4%B9%8B%E5%86%85%E5%AD%98%E8%99%9A%E6%8B%9F%E5%8C%96.md)
- [KVM详解](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%E8%AF%A6%E8%A7%A3.md)
- [QEMU-KVM 虚拟化环境的搭建与使用](https://github.com/0voice/kernel_awsome_feature/blob/main/QEMU-KVM%20%E8%99%9A%E6%8B%9F%E5%8C%96%E7%8E%AF%E5%A2%83%E7%9A%84%E6%90%AD%E5%BB%BA%E4%B8%8E%E4%BD%BF%E7%94%A8.md)
- [详解KVM虚拟化原理](https://github.com/0voice/kernel_awsome_feature/blob/main/%E8%AF%A6%E8%A7%A3KVM%E8%99%9A%E6%8B%9F%E5%8C%96%E5%8E%9F%E7%90%86.md)
- [KVM到底是个啥？](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%E5%88%B0%E5%BA%95%E6%98%AF%E4%B8%AA%E5%95%A5%EF%BC%9F.md)
- [KVM实践初步](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%E5%AE%9E%E8%B7%B5%E5%88%9D%E6%AD%A5.md)
- [KVM管理工具libvirt](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%E7%AE%A1%E7%90%86%E5%B7%A5%E5%85%B7libvirt.md)
- [KVM虚拟机的各种安装方法](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E5%90%84%E7%A7%8D%E5%AE%89%E8%A3%85%E6%96%B9%E6%B3%95.md)
- [KVM虚拟机全生命周期管理实战](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%85%A8%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E7%AE%A1%E7%90%86%E5%AE%9E%E6%88%98.md)
- [KVM虚拟机存储管理实战（上篇）](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%98%E5%82%A8%E7%AE%A1%E7%90%86%E5%AE%9E%E6%88%98%EF%BC%88%E4%B8%8A%E7%AF%87%EF%BC%89.md)
- [KVM虚拟机存储管理实战（下篇）](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%98%E5%82%A8%E7%AE%A1%E7%90%86%E5%AE%9E%E6%88%98%EF%BC%88%E4%B8%8B%E7%AF%87%EF%BC%89.md)
- [KVM虚拟机网络管理实战](https://github.com/0voice/kernel_awsome_feature/blob/main/KVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%BD%91%E7%BB%9C%E7%AE%A1%E7%90%86%E5%AE%9E%E6%88%98.md)

## 学术论文

### 视频

## 🔥 ceph

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633285-0e03e44f-8755-4b12-9b62-8f8030d44c94.png"/>
  
#### —— 存储的未来
</div>

### 文档
- 官方文档:
- 其他文档：

### 开源项目 

### 文章

### 视频

## 🔥 fuse

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633338-fde17a17-9cfb-4a32-bc51-d76e02b6904e.png"/>
  
#### —— 用户态文件系统
</div>

### 文档
- 官方文档:
- 其他文档：

### 开源项目 

### 文章

### 视频

## 🔥 virtio

<!--
<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633338-fde17a17-9cfb-4a32-bc51-d76e02b6904e.png"/>
  
#### —— 用户态文件系统
</div>
-->
### 文档
- 官方文档:
- 其他文档：

### 开源项目 

### 文章

### 视频

## 🔥 vhost

<!--
<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633338-fde17a17-9cfb-4a32-bc51-d76e02b6904e.png"/>
  
#### —— 用户态文件系统
</div>
-->
### 文档
- 官方文档:
- 其他文档：

### 开源项目 

### 文章

### 视频


