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

### 学术论文

- [Linux-based Virtualization](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%20Linux-based%20Virtualization.pdf)
- [Architecture of the Kernel-based Virtual Machine (KVM)](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/Architecture%20of%20the%20Kernel-based%20Virtual%20Machine%20(KVM).pdf)
- [IBM-Best practices for KVM](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/IBM-Best%20practices%20for%20KVM.pdf)
- [Introduction to KVM](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/Introduction%20to%20KVM.pdf)
- [Virtio-blk Performance Improvement](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/Virtio-blk%20Performance%20Improvement.pdf)
- [Virtualization with KVM](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/Virtualization%20with%20KVM.pdf)
- [KVM客户机主动共享的内存超量使用策略研究](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/KVM%E5%AE%A2%E6%88%B7%E6%9C%BA%E4%B8%BB%E5%8A%A8%E5%85%B1%E4%BA%AB%E7%9A%84%E5%86%85%E5%AD%98%E8%B6%85%E9%87%8F%E4%BD%BF%E7%94%A8%E7%AD%96%E7%95%A5%E7%A0%94%E7%A9%B6.pdf)
- [KVM系统任务管理的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/KVM%E7%B3%BB%E7%BB%9F%E4%BB%BB%E5%8A%A1%E7%AE%A1%E7%90%86%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [KVM虚拟化动态迁移技术的安全防护模型](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/KVM%E8%99%9A%E6%8B%9F%E5%8C%96%E5%8A%A8%E6%80%81%E8%BF%81%E7%A7%BB%E6%8A%80%E6%9C%AF%E7%9A%84%E5%AE%89%E5%85%A8%E9%98%B2%E6%8A%A4%E6%A8%A1%E5%9E%8B.pdf)
- [KVM虚拟机CPU虚拟化的研究与调度策略的优化](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/KVM%E8%99%9A%E6%8B%9F%E6%9C%BACPU%E8%99%9A%E6%8B%9F%E5%8C%96%E7%9A%84%E7%A0%94%E7%A9%B6%E4%B8%8E%E8%B0%83%E5%BA%A6%E7%AD%96%E7%95%A5%E7%9A%84%E4%BC%98%E5%8C%96.caj)
- [KVM虚拟机热迁移算法分析及优化](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/KVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%83%AD%E8%BF%81%E7%A7%BB%E7%AE%97%E6%B3%95%E5%88%86%E6%9E%90%E5%8F%8A%E4%BC%98%E5%8C%96.pdf)
- [KVM虚拟机的性能研究与改进](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/KVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E6%80%A7%E8%83%BD%E7%A0%94%E7%A9%B6%E4%B8%8E%E6%94%B9%E8%BF%9B.pdf)
- [KVM虚拟机的漏洞验证与利用方式研究](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/KVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E6%BC%8F%E6%B4%9E%E9%AA%8C%E8%AF%81%E4%B8%8E%E5%88%A9%E7%94%A8%E6%96%B9%E5%BC%8F%E7%A0%94%E7%A9%B6.pdf)
- [QEMU-KVM设备虚拟化研究与改进](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/QEMU-KVM%E8%AE%BE%E5%A4%87%E8%99%9A%E6%8B%9F%E5%8C%96%E7%A0%94%E7%A9%B6%E4%B8%8E%E6%94%B9%E8%BF%9B.pdf)
- [Xen与KVM虚拟化方案的设计与性能评比](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/Xen%E4%B8%8EKVM%E8%99%9A%E6%8B%9F%E5%8C%96%E6%96%B9%E6%A1%88%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E6%80%A7%E8%83%BD%E8%AF%84%E6%AF%94.pdf)
- [Xen和KVM等四大虚拟化架构对比分析](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/Xen%E5%92%8CKVM%E7%AD%89%E5%9B%9B%E5%A4%A7%E8%99%9A%E6%8B%9F%E5%8C%96%E6%9E%B6%E6%9E%84%E5%AF%B9%E6%AF%94%E5%88%86%E6%9E%90.pdf)
- [基于KVM的虚拟桌面基础架构设计与优化](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8E%20%EF%BC%AB%EF%BC%B6%EF%BC%AD%20%E7%9A%84%E8%99%9A%E6%8B%9F%E6%A1%8C%E9%9D%A2%E5%9F%BA%E7%A1%80%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E4%BC%98%E5%8C%96.pdf)
- [基于IEEE1588的虚拟集群任务同步测量技术研究](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EIEEE1588%E7%9A%84%E8%99%9A%E6%8B%9F%E9%9B%86%E7%BE%A4%E4%BB%BB%E5%8A%A1%E5%90%8C%E6%AD%A5%E6%B5%8B%E9%87%8F%E6%8A%80%E6%9C%AF%E7%A0%94%E7%A9%B6.pdf)
- [基于KVM云计算平台的分布式关系型数据库的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EKVM%E4%BA%91%E8%AE%A1%E7%AE%97%E5%B9%B3%E5%8F%B0%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E5%85%B3%E7%B3%BB%E5%9E%8B%E6%95%B0%E6%8D%AE%E5%BA%93%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于KVM的桌面虚拟化VDI研究以及实现](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EKVM%E7%9A%84%E6%A1%8C%E9%9D%A2%E8%99%9A%E6%8B%9F%E5%8C%96VDI%E7%A0%94%E7%A9%B6%E4%BB%A5%E5%8F%8A%E5%AE%9E%E7%8E%B0.caj)
- [基于KVM的私有云应用平台的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EKVM%E7%9A%84%E7%A7%81%E6%9C%89%E4%BA%91%E5%BA%94%E7%94%A8%E5%B9%B3%E5%8F%B0%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于KVM的虚拟机自省系统设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EKVM%E7%9A%84%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%87%AA%E7%9C%81%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于KVM的虚拟机调度方法研究](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EKVM%E7%9A%84%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%B0%83%E5%BA%A6%E6%96%B9%E6%B3%95%E7%A0%94%E7%A9%B6.pdf)
- [基于KVM虚拟化技术的研究与实验评估](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EKVM%E8%99%9A%E6%8B%9F%E5%8C%96%E6%8A%80%E6%9C%AF%E7%9A%84%E7%A0%94%E7%A9%B6%E4%B8%8E%E5%AE%9E%E9%AA%8C%E8%AF%84%E4%BC%B0.pdf)
- [基于KVM虚拟化的TCP_IP协议栈隔离](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EKVM%E8%99%9A%E6%8B%9F%E5%8C%96%E7%9A%84TCP_IP%E5%8D%8F%E8%AE%AE%E6%A0%88%E9%9A%94%E7%A6%BB.pdf)
- [基于KVM虚拟机动态迁移的研究与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EKVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%8A%A8%E6%80%81%E8%BF%81%E7%A7%BB%E7%9A%84%E7%A0%94%E7%A9%B6%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于KVM虚拟机的恶意行为检测系统设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EKVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E6%81%B6%E6%84%8F%E8%A1%8C%E4%B8%BA%E6%A3%80%E6%B5%8B%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于KVM设备虚拟化技术的研究](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EKVM%E8%AE%BE%E5%A4%87%E8%99%9A%E6%8B%9F%E5%8C%96%E6%8A%80%E6%9C%AF%E7%9A%84%E7%A0%94%E7%A9%B6.pdf)
- [基于KVM集群的负载均衡机制系统的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EKVM%E9%9B%86%E7%BE%A4%E7%9A%84%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E6%9C%BA%E5%88%B6%E7%B3%BB%E7%BB%9F%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于Linux的虚拟化技术研究和应用](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8ELinux%E7%9A%84%E8%99%9A%E6%8B%9F%E5%8C%96%E6%8A%80%E6%9C%AF%E7%A0%94%E7%A9%B6%E5%92%8C%E5%BA%94%E7%94%A8.pdf)
- [基于QEMU-KVM的办公桌面云系统的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EQEMU-KVM%E7%9A%84%E5%8A%9E%E5%85%AC%E6%A1%8C%E9%9D%A2%E4%BA%91%E7%B3%BB%E7%BB%9F%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于QEMU-KVM的桌面云服务端软件架构设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EQEMU-KVM%E7%9A%84%E6%A1%8C%E9%9D%A2%E4%BA%91%E6%9C%8D%E5%8A%A1%E7%AB%AF%E8%BD%AF%E4%BB%B6%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于oVirt_Qemu_Kvm云平台系统分析与安全加固设计](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8EoVirt_Qemu_Kvm%E4%BA%91%E5%B9%B3%E5%8F%B0%E7%B3%BB%E7%BB%9F%E5%88%86%E6%9E%90%E4%B8%8E%E5%AE%89%E5%85%A8%E5%8A%A0%E5%9B%BA%E8%AE%BE%E8%AE%A1.pdf)
- [基于内核的虚拟机的研究](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8E%E5%86%85%E6%A0%B8%E7%9A%84%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%A0%94%E7%A9%B6.pdf)
- [基于多核的虚拟化技术研究](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E5%9F%BA%E4%BA%8E%E5%A4%9A%E6%A0%B8%E7%9A%84%E8%99%9A%E6%8B%9F%E5%8C%96%E6%8A%80%E6%9C%AF%E7%A0%94%E7%A9%B6.pdf)
- [网络功能虚拟化平台研究](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E7%BD%91%E7%BB%9C%E5%8A%9F%E8%83%BD%E8%99%9A%E6%8B%9F%E5%8C%96%E5%B9%B3%E5%8F%B0%E7%A0%94%E7%A9%B6.pdf)
- [虚拟机应用系统的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/kvm/%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%BA%94%E7%94%A8%E7%B3%BB%E7%BB%9F%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)


### 视频(提取码：1024)

- [Analysis of AMD HW-assisted vIOMMU Implementation and Performance](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Background Snapshots in QEMU- Towards Asynchronous Revert - Denis Lunev, Virtuozzo](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Debugging Secured Windows OS guest using KVM_QEMU and Windbg - Marek Kędzierski, Red Hat](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Dirty Quota-Based VM Live Migration Auto-Converge - Manish Mishra & Shivam Kumar, Nutanix India](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Don't Peek Into my Container! - Alice Frosi, Christophe de Dinechin & Sergio Lopez Pascual, Red Hat](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Encrypted Virtual Machine Images for Confidential Computing - James Bottomley, IBM & Brijesh Singh](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [HCK-CI- Enabling CI for Windows Guest Paravirtualized Drivers - Kostiantyn Kostiuk](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [High Performance NVMe Offloading in SPDK Using the New vfio-user Protocol](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Host & Guest Tracing in Virtualization- -To sync, or not to sync](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [How Hard Could it be to Flip a bit- KVM PV Feature Enablement up the Virtualization Stack_2](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Hyperscale vDPA - Jason Wang, Red Hat](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Hypervisor-less Virtio for Real-time and Safety - Maarten Koning, Wind River](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Is QEMU too Complex, and What Can we do About It- - Paolo Bonzini, Red Hat, Inc.](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Is QEMU too Complex, and What Can we do About It- - Paolo Bonzini, Red Hat, Inc._2](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Know your QEMU and KVM Test Frameworks - Thomas Huth, Red Hat](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Kubevirt and the Cost of Containerizing VMs](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [KVM Dirty Page Tracking - Peter Xu, Red Hat](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [KVM Memory Cost Optimization in Alibaba Cloud - Huaitong Han, Alibaba Cloud](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Lessons Learned Building a Production Memory-Overcommit Solution - Florian Schmidt & Ivan Teterevkov](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [libkrun- More than a VMM, in Dynamic Library Form - Sergio Lopez Pascual, Red Hat_2](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [libvfio-user- Status Update - Thanos Makatos & John Levon, Nutanix](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [libvfio-user- Status Update - Thanos Makatos & John Levon, Nutanix_2](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Live Migrating VFIO, vhost-user, and vfio-user Devices - Stefan Hajnoczi, Red Hat](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Live Migrating VFIO, vhost-user, and vfio-user Devices - Stefan Hajnoczi, Red Hat_2](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Mitigating Excessive Pause-Loop-Exiting in VM-Agnostic KVM - Kenta Ishiguro, Keio University](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [New Qemu Backup Architecture and API - Vladimir Sementsov-Ogievskiy, Virtuozzo](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Passthrough_Headless GPU Gets Ahead - Tina Zhang & Vivek Kasireddy, Intel](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Protecting from MaliciousHypervisor Using AMD SEV-SNP - Brijesh Singh, AMD](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [QEMU Emulated NVMe - Lessons Learned and Future Work - Klaus Jensen, Samsung Electronics](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [QEMU Emulated NVMe - Lessons Learned and Future Work - Klaus Jensen, Samsung Electronics_2](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Qemu Performance Regression CI - Lukáš Doktor, Red Hat Czech, s. r. o.](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Receive Side Scaling (RSS) with eBPF in QEMU and virtio-net - Yan Vugenfirer, Daynix](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [rust-vmm- A Security Journey - Andreea Florescu, Amazon](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Secure Live Migration of Encrypted VMs - Tobin Feldman-Fitzthum & Dov Murik, IBM](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Securing Linux VM boot with AMD SEV measurement - Dov Murik & Hubertus Franke, IBM Research](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Securing the Hypervisor with Control-Flow Integrity - Daniele Buono, IBM](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Sharing IOMMU PageTables with TDP in KVM - Lu Baolu & Zhao Yan, Intel Corporation](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Status Update on TDX Support - Isaku Yamahata, Intel](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Support SDEI Virtualization and Asynchronous Page Fault for arm64 - Gavin Shan, Redhat](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [TDX Live Migration - Wei Wang, Intel Corp.](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [The Traps of Using Hyper-V Features in KVM Environment - Liang Li, Alibaba](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Towards a More Efficiently Synchronization in KVM - Wanpeng Li, Tencent Cloud](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Towards High-availability for Virtio-fs - Jiachen Zhang & Yongji Xie, ByteDance](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [Unmapped Guest Memory - Yu Zhang, Intel](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [vdpa-blk- Unified Hardware and Software Offload for virtio-blk - Stefano Garzarella, Red Hat_2](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [VDUSE - vDPA Device in Userspace - Yongji Xie, ByteDance](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)
- [VFIO User - Using VFIO as the IPC Protocol in Multi-process QEMU - John Johnson & Jagannathan Raman](https://pan.baidu.com/s/1aG_BJRY1rggPsM2HPdMQBQ)

## 🔥 ceph

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633285-0e03e44f-8755-4b12-9b62-8f8030d44c94.png"/>
  
#### —— 存储的未来
</div>

### 文档
- 官方文档: https://docs.ceph.com/en/pacific/#
  - GitHub仓：https://github.com/ceph/ceph
- 其他文档：
  - IMB：Ceph: A Linux petabyte-scale distributed file system：https://developer.ibm.com/tutorials/l-ceph/
  - 红帽 Ceph：https://www.redhat.com/en/technologies/storage/ceph
  - 红帽 文件系统指南：https://access.redhat.com/documentation/zh-cn/red_hat_ceph_storage/4/html/file_system_guide/introduction-to-the-ceph-file-system
  - Ceph v10.0 中文文档：https://www.bookstack.cn/read/ceph-10-zh/cd0dcad3545db7c0.md
  - Ceph 手册：https://www.kancloud.cn/willseecloud/ceph/1788233
  - Ceph 中文文档：https://www.wenjiangs.com/doc/trfbacev
  - Ceph 学习笔记：https://www.bookstack.cn/read/zxj_ceph/deploy
  - Ceph 运维手册：https://lihaijing.gitbooks.io/ceph-handbook/content/
  - Ceph 13.2.1 常用命令手册：https://www.siguadantang.com/cloud/ceph/ceph-command/

![image](https://user-images.githubusercontent.com/87457873/153714645-072731c5-bdfe-4692-9ad5-1836269861a1.png)

### 学术论文

- [RELIABLE, SCALABLE, AND HIGH-PERFORMANCE DISTRIBUTED STORAGE](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%20RELIABLE%2C%20SCALABLE%2C%20AND%20HIGH-PERFORMANCE%20DISTRIBUTED%20STORAGE.pdf)
- [Ceph分层存储优化策略研究与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/Ceph%E5%88%86%E5%B1%82%E5%AD%98%E5%82%A8%E4%BC%98%E5%8C%96%E7%AD%96%E7%95%A5%E7%A0%94%E7%A9%B6%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [Ceph分布式文件系统的研究及性能测试](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/Ceph%E5%88%86%E5%B8%83%E5%BC%8F%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E7%9A%84%E7%A0%94%E7%A9%B6%E5%8F%8A%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95.pdf)
- [Ceph存储引擎中基于固态盘的日志机制优化](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/Ceph%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E%E4%B8%AD%E5%9F%BA%E4%BA%8E%E5%9B%BA%E6%80%81%E7%9B%98%E7%9A%84%E6%97%A5%E5%BF%97%E6%9C%BA%E5%88%B6%E4%BC%98%E5%8C%96.pdf)
- [Ceph存储系统纠删码覆盖写的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/Ceph%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9F%E7%BA%A0%E5%88%A0%E7%A0%81%E8%A6%86%E7%9B%96%E5%86%99%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.caj)
- [Ceph对象文件系统添加任务迁移特性的研究](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/Ceph%E5%AF%B9%E8%B1%A1%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E6%B7%BB%E5%8A%A0%E4%BB%BB%E5%8A%A1%E8%BF%81%E7%A7%BB%E7%89%B9%E6%80%A7%E7%9A%84%E7%A0%94%E7%A9%B6.caj)
- [Ceph异构存储优化机制研究](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/Ceph%E5%BC%82%E6%9E%84%E5%AD%98%E5%82%A8%E4%BC%98%E5%8C%96%E6%9C%BA%E5%88%B6%E7%A0%94%E7%A9%B6.pdf)
- [Ceph文件系统元数据访问性能优化研究](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/Ceph%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E5%85%83%E6%95%B0%E6%8D%AE%E8%AE%BF%E9%97%AE%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E7%A0%94%E7%A9%B6.pdf)
- [OpenStack和Ceph结合的云存储设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/OpenStack%E5%92%8CCeph%E7%BB%93%E5%90%88%E7%9A%84%E4%BA%91%E5%AD%98%E5%82%A8%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [一种面向云环境的Ceph集群能耗管理策略研究](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E4%B8%80%E7%A7%8D%E9%9D%A2%E5%90%91%E4%BA%91%E7%8E%AF%E5%A2%83%E7%9A%84Ceph%E9%9B%86%E7%BE%A4%E8%83%BD%E8%80%97%E7%AE%A1%E7%90%86%E7%AD%96%E7%95%A5%E7%A0%94%E7%A9%B6.pdf)
- [基于CEPH分布式文件系统的云盘系统的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECEPH%E5%88%86%E5%B8%83%E5%BC%8F%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E7%9A%84%E4%BA%91%E7%9B%98%E7%B3%BB%E7%BB%9F%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于Ceph分布式存储系统副本一致性研究](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9F%E5%89%AF%E6%9C%AC%E4%B8%80%E8%87%B4%E6%80%A7%E7%A0%94%E7%A9%B6.pdf)
- [基于Ceph分布式存储系统的安全访问控制设计和实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9F%E7%9A%84%E5%AE%89%E5%85%A8%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6%E8%AE%BE%E8%AE%A1%E5%92%8C%E5%AE%9E%E7%8E%B0.pdf)
- [基于Ceph分布式系统的存储虚拟化技术的研究与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E7%9A%84%E5%AD%98%E5%82%A8%E8%99%9A%E6%8B%9F%E5%8C%96%E6%8A%80%E6%9C%AF%E7%9A%84%E7%A0%94%E7%A9%B6%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于Ceph构建EB级对象存储系统](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E6%9E%84%E5%BB%BAEB%E7%BA%A7%E5%AF%B9%E8%B1%A1%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9F.pdf)
- [基于Ceph的企业云存储系统的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E7%9A%84%E4%BC%81%E4%B8%9A%E4%BA%91%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9F%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于Ceph的分布式存储在数据中心的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9F%E8%B7%A8%E6%9C%BA%E6%9E%B6%E4%BF%AE%E5%A4%8D%E6%8A%80%E6%9C%AF%E4%BC%98%E5%8C%96.pdf)
- [基于Ceph的分布式存储系统跨机架修复技术优化](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9F%E8%B7%A8%E6%9C%BA%E6%9E%B6%E4%BF%AE%E5%A4%8D%E6%8A%80%E6%9C%AF%E4%BC%98%E5%8C%96.pdf)
- [基于Ceph的分布式存储节能技术研究与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E8%8A%82%E8%83%BD%E6%8A%80%E6%9C%AF%E7%A0%94%E7%A9%B6%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于Ceph的分布式异构存储系统数据处理优化研究](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E5%BC%82%E6%9E%84%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9F%E6%95%B0%E6%8D%AE%E5%A4%84%E7%90%86%E4%BC%98%E5%8C%96%E7%A0%94%E7%A9%B6.pdf)
- [基于Ceph的分布式网盘系统的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E7%BD%91%E7%9B%98%E7%B3%BB%E7%BB%9F%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于Ceph的对象存储服务系统的研究与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E7%9A%84%E5%AF%B9%E8%B1%A1%E5%AD%98%E5%82%A8%E6%9C%8D%E5%8A%A1%E7%B3%BB%E7%BB%9F%E7%9A%84%E7%A0%94%E7%A9%B6%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于Ceph的数据读写性能优化研究](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E7%9A%84%E6%95%B0%E6%8D%AE%E8%AF%BB%E5%86%99%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E7%A0%94%E7%A9%B6.pdf)
- [基于Ceph的非共享存储虚拟机动态迁移系统的优化](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8ECeph%E7%9A%84%E9%9D%9E%E5%85%B1%E4%BA%AB%E5%AD%98%E5%82%A8%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%8A%A8%E6%80%81%E8%BF%81%E7%A7%BB%E7%B3%BB%E7%BB%9F%E7%9A%84%E4%BC%98%E5%8C%96.pdf)
- [基于ceph文件系统的元数据缓存备份技术的研究与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8Eceph%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E7%9A%84%E5%85%83%E6%95%B0%E6%8D%AE%E7%BC%93%E5%AD%98%E5%A4%87%E4%BB%BD%E6%8A%80%E6%9C%AF%E7%9A%84%E7%A0%94%E7%A9%B6%E4%B8%8E%E5%AE%9E%E7%8E%B0.caj)
- [基于ceph的云存储系统的设计与实现](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8Eceph%E7%9A%84%E4%BA%91%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9F%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)
- [基于客户端缓存与请求调度的Ceph文件系统读时延优化策略研究](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8E%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%BC%93%E5%AD%98%E4%B8%8E%E8%AF%B7%E6%B1%82%E8%B0%83%E5%BA%A6%E7%9A%84Ceph%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E8%AF%BB%E6%97%B6%E5%BB%B6%E4%BC%98%E5%8C%96%E7%AD%96%E7%95%A5%E7%A0%94%E7%A9%B6.pdf)
- [基于对象存储的Ceph分布式文件系统的研究](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%9F%BA%E4%BA%8E%E5%AF%B9%E8%B1%A1%E5%AD%98%E5%82%A8%E7%9A%84Ceph%E5%88%86%E5%B8%83%E5%BC%8F%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E7%9A%84%E7%A0%94%E7%A9%B6.pdf)
- [开源分布式存储系统Ceph测试及在桌面虚拟化平台中的应用](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E5%BC%80%E6%BA%90%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9FCeph%E6%B5%8B%E8%AF%95%E5%8F%8A%E5%9C%A8%E6%A1%8C%E9%9D%A2%E8%99%9A%E6%8B%9F%E5%8C%96%E5%B9%B3%E5%8F%B0%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8.pdf)
- [漫谈分布式存储方案，GPFS 对话 CEPH](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E6%BC%AB%E8%B0%88%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E6%96%B9%E6%A1%88%EF%BC%8CGPFS%20%E5%AF%B9%E8%AF%9D%20CEPH.pdf)
- [面向分布式存储系统Ceph的遥感影像瓦片存储及其关键技术](https://github.com/0voice/kernel_awsome_feature/blob/main/Ceph/%E9%9D%A2%E5%90%91%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9FCeph%E7%9A%84%E9%81%A5%E6%84%9F%E5%BD%B1%E5%83%8F%E7%93%A6%E7%89%87%E5%AD%98%E5%82%A8%E5%8F%8A%E5%85%B6%E5%85%B3%E9%94%AE%E6%8A%80%E6%9C%AF.pdf)

### 文章

- [「基础理论」CEPH 基础介绍](https://github.com/0voice/kernel_awsome_feature/blob/main/%E3%80%8C%E5%9F%BA%E7%A1%80%E7%90%86%E8%AE%BA%E3%80%8DCEPH%20%E5%9F%BA%E7%A1%80%E4%BB%8B%E7%BB%8D.md)
- [「核心」Ceph学习三部曲之一:A First Glance At CRUSH](https://github.com/0voice/kernel_awsome_feature/blob/main/%E3%80%8C%E6%A0%B8%E5%BF%83%E3%80%8DCeph%E5%AD%A6%E4%B9%A0%E4%B8%89%E9%83%A8%E6%9B%B2%E4%B9%8B%E4%B8%80:A%20First%20Glance%20At%20CRUSH.md)
- [「核心」Ceph学习三部曲之二:CEPH 纠删码操作&API](https://github.com/0voice/kernel_awsome_feature/blob/main/%E3%80%8C%E6%A0%B8%E5%BF%83%E3%80%8DCeph%E5%AD%A6%E4%B9%A0%E4%B8%89%E9%83%A8%E6%9B%B2%E4%B9%8B%E4%BA%8C:CEPH%20%E7%BA%A0%E5%88%A0%E7%A0%81%E6%93%8D%E4%BD%9C%26API.md)
- [「核心」Ceph学习三部曲之三:迁移之美——PG读写流程与状态迁移详解 ](https://github.com/0voice/kernel_awsome_feature/blob/main/%E3%80%8C%E6%A0%B8%E5%BF%83%E3%80%8DCeph%E5%AD%A6%E4%B9%A0%E4%B8%89%E9%83%A8%E6%9B%B2%E4%B9%8B%E4%B8%89:%E8%BF%81%E7%A7%BB%E4%B9%8B%E7%BE%8E%E2%80%94%E2%80%94PG%E8%AF%BB%E5%86%99%E6%B5%81%E7%A8%8B%E4%B8%8E%E7%8A%B6%E6%80%81%E8%BF%81%E7%A7%BB%E8%AF%A6%E8%A7%A3%20.md)
- [「核心」Ceph学习三部曲之四:下一代对象存储引擎BlueStore](https://github.com/0voice/kernel_awsome_feature/blob/main/%E3%80%8C%E6%A0%B8%E5%BF%83%E3%80%8DCeph%E5%AD%A6%E4%B9%A0%E4%B8%89%E9%83%A8%E6%9B%B2%E4%B9%8B%E5%9B%9B:%E4%B8%8B%E4%B8%80%E4%BB%A3%E5%AF%B9%E8%B1%A1%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8EBlueStore.md)
- [「核心」Ceph学习三部曲之五：分布式块存储RBD](https://github.com/0voice/kernel_awsome_feature/blob/main/%E3%80%8C%E6%A0%B8%E5%BF%83%E3%80%8DCeph%E5%AD%A6%E4%B9%A0%E4%B8%89%E9%83%A8%E6%9B%B2%E4%B9%8B%E4%BA%94%EF%BC%9A%E5%88%86%E5%B8%83%E5%BC%8F%E5%9D%97%E5%AD%98%E5%82%A8RBD.md)
- [「核心」Ceph学习三部曲之六：对象存储网关RGW](https://github.com/0voice/kernel_awsome_feature/blob/main/%E3%80%8C%E6%A0%B8%E5%BF%83%E3%80%8DCeph%E5%AD%A6%E4%B9%A0%E4%B8%89%E9%83%A8%E6%9B%B2%E4%B9%8B%E5%85%AD%EF%BC%9A%E5%AF%B9%E8%B1%A1%E5%AD%98%E5%82%A8%E7%BD%91%E5%85%B3RGW.md)
- [「核心」Ceph学习三部曲之七：分布式文件系统CephFS](https://github.com/0voice/kernel_awsome_feature/blob/main/%E3%80%8C%E6%A0%B8%E5%BF%83%E3%80%8DCeph%E5%AD%A6%E4%B9%A0%E4%B8%89%E9%83%A8%E6%9B%B2%E4%B9%8B%E4%B8%83%EF%BC%9A%E5%88%86%E5%B8%83%E5%BC%8F%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9FCephFS.md)
- [「核心」Ceph学习三部曲之八:控制先行——Ceph的QoS策略](https://github.com/0voice/kernel_awsome_feature/blob/main/%E3%80%8C%E6%A0%B8%E5%BF%83%E3%80%8DCeph%E5%AD%A6%E4%B9%A0%E4%B8%89%E9%83%A8%E6%9B%B2%E4%B9%8B%E5%85%AB:%E6%8E%A7%E5%88%B6%E5%85%88%E8%A1%8C%E2%80%94%E2%80%94Ceph%E7%9A%84QoS%E7%AD%96%E7%95%A5.md)
- [「模块」Ceph-Monitor 集群的代脑](https://github.com/0voice/kernel_awsome_feature/blob/main/%E3%80%8C%E6%A8%A1%E5%9D%97%E3%80%8DCeph-Monitor%20%E9%9B%86%E7%BE%A4%E7%9A%84%E4%BB%A3%E8%84%91.md)
- [「模块」Ceph-OSD 集群的奠基石](https://github.com/0voice/kernel_awsome_feature/blob/main/%E3%80%8C%E6%A8%A1%E5%9D%97%E3%80%8DCeph-OSD%20%E9%9B%86%E7%BE%A4%E7%9A%84%E5%A5%A0%E5%9F%BA%E7%9F%B3.md)
- [分布式存储 Ceph 的演进经验 · SOSP '19](https://github.com/0voice/kernel_awsome_feature/blob/main/%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%20Ceph%20%E7%9A%84%E6%BC%94%E8%BF%9B%E7%BB%8F%E9%AA%8C%20%C2%B7%20SOSP%20'19.md)
- [分布式存储Ceph](https://github.com/0voice/kernel_awsome_feature/blob/main/%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8Ceph.md)
- [分布式文件系统ceph知识整理](https://github.com/0voice/kernel_awsome_feature/blob/main/%E5%88%86%E5%B8%83%E5%BC%8F%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9Fceph%E7%9F%A5%E8%AF%86%E6%95%B4%E7%90%86.md)
- [基于Ceph打造高性能高可靠的分布式块存储系统](https://github.com/0voice/kernel_awsome_feature/blob/main/%E5%88%86%E5%B8%83%E5%BC%8F%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9Fceph%E7%9F%A5%E8%AF%86%E6%95%B4%E7%90%86.md)
- [Ceph论文译文 Ceph：一个可扩展，高性能分布式文件系统](https://github.com/0voice/kernel_awsome_feature/blob/main/%E5%9F%BA%E4%BA%8ECeph%E6%89%93%E9%80%A0%E9%AB%98%E6%80%A7%E8%83%BD%E9%AB%98%E5%8F%AF%E9%9D%A0%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E5%9D%97%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9F.md)


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


