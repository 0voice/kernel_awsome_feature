# 什么是虚拟化

我们常说的虚拟化通常是指平台虚拟化（plaform virtualization），通过在原有的硬件资源的基础上，利用某些技术手段创建一套虚拟的硬件资源，然后在这个虚拟的硬件平台上安装一个完整的操作系统。一台性能良好的服务器可以安装多个系统，这些虚拟的系统称为客户操作系统。

# 虚拟化的类型

按照实现方式的不同，虚拟化可以分为完全虚拟化（full virtualization）和半虚拟化（paravirtualization），在解释这两种虚拟化的区别之前，需要先了解一些知识：cpu的运行级别。如下图

![image-20220212173646338](C:\Users\Mr.wang\AppData\Roaming\Typora\typora-user-images\image-20220212173646338.png)

从内到外，一共有4个运行级别：它们是ring0、ring1、ring2还有ring3，ring0通常称为内核空间，ring3则称为用户空间。内核代码运行于ring0中，它要求的权限是最高的，任何直接操作硬件的代码都必须运行在ring0中，例如cpu操作、内存操作和其他IO操作。再来看ring3，它的权限是最低的，我们平时使用的应用程序基本上都是运行在这个级别，这些程序必须通过ring0的协助，才能完成它们的指令，例如写硬盘。在linux中，ring3和ring0之间的交互层称为系统调用（system call），例如read、write函数，这些都是典型的系统调用，由Linux kernel提供。

## hypervisor和VMM

可以把hypervisor和VMM（virtual machine monitor）理解为同一样东西，它们都是用于监控和控制虚拟机操作系统。在创建虚拟机的过程中，会虚拟出一整套的硬件资源，例如硬盘、内存、CPU、网络设备等，这些工作都是由hypervisor负责，除此之外，它还负责虚拟系统运行过程的资源分配和虚拟机的生命周期管理等。

## type 1虚拟化和type 2虚拟化

根据hypervisor所处的位置不同，可以划分为type 1虚拟化和type 2虚拟化。

一个显著的特点是：

- type 1虚拟化的hypervisor可以直接运行在裸机上面，hypervisor和实体硬件之间不存在操作系统层面。VMware ESXi就属于典型的1型虚拟化。
- type 2虚拟化的hypervisor必须基于现有操作系统（host system），不能直接运行于裸机中。virtualbox和vmware workstation就属于这一类，有人说kvm也是属于type 2虚拟化，但RedHat 官方把kvm归为type 1虚拟化。

下面是Redhat的原文：

> KVM converts Linux into a type-1 (bare-metal) hypervisor. All hypervisors need some operating system-level components—such as a memory manager, process scheduler, input/output (I/O) stack, device drivers, security manager, a network stack, and more—to run VMs. KVM has all these components because it’s part of the Linux kernel. Every VM is implemented as a regular Linux process, scheduled by the standard Linux scheduler, with dedicated virtual hardware like a network card, graphics adapter, CPU(s), memory, and disks.

文章链接: [What is KVM](https://www.redhat.com/en/topics/virtualization/what-is-KVM)

## 全虚拟化和半虚拟化

现在回过头来解释全虚拟化和半虚拟化。下面是它们的一些区别

- 全虚拟化中的虚拟系统不知道自己安装在虚拟机中，它像安装在实体机中一样执行任务
- 全虚拟化系统的内核不需要经过修改
- 半虚拟化的系统知道自己处于虚拟机环境
- 半虚拟化系统的内核必须经过更改才能被安装

在虚拟化中，我们的宿主操作系统的内核已经占据了CPU的ring0运行级别，也就是说客户机操作系统已经不能在这个级别运行，怎么办呢？虽然客户机操作系统不能运行在ring0，但是hypervisor可以。所以客户操作系统退而求其次，运行于ring1中，造成的问题是：内核空间中的代码无法运行。解决办法如下：

- 客户操作系统如常运行
- hypervisor捕捉客户操作系统无法执行ring0指令时抛出的异常，把指令翻译后进行模拟，最后返回给客户操作系统

这个翻译的过程称为二进制翻译（binary transaction），它的问题是这样一来一回，会造成性能损耗。针对这个问题，出现了半虚拟化，半虚拟化的核心是：

- 一个被修改过内核的操作系统，替换掉不能虚拟化的指令（把正常的system call变为hyper call）
- hyper call直接和hypervisor打交道，省去了捕获和翻译的过程，从而提升性能

就目前的趋势来看，全虚拟化的性能现在已经越来越高，在x64平台上甚至已经超过了半虚拟化，因此未来基本上都是全虚拟化的天下。

# QEMU和KVM的关系

QEMU初期是一个完整的虚拟化解决方案，但它的缺点是笨重，且性能不怎样。但后来被KVM项目组的人看中，经过精简和优化后整合到KVM，因此现在它们被称为QEMU-KVM，qemu的作用就是模拟虚拟机的硬件。虽然它们两是一个整体，但是它们的运行级别不同：

- KVM是一个内核模块，因此它运行于内核空间
- QEMU是运行于用户空间的一个应用程序

KVM包含三个模块：

- kvm.ko
- kvm-intel.ko
- kvm-amd.ko

kvm.ko是一个和cpu架构无关的通用内核模块，其他两个是cpu相关的。如果内核检测到vmx标记，那么就加载kvm-intel.ko，相反，如果检测到svm标记，就加载kvm-amd.ko。

单独的内核不是hypervisor，qemu和kvm也不是hypersior；qemu-kvm加上已经加载了kvm模块的内核，就组成了一个功能完整的hypervisor。

除了模拟硬件外，QEMU有两个重要功能：

- 执行二进制翻译功能 ，前面已经介绍了binary transaction的重要性，这个功能就是由qemu负责的
- 初始化vcpu线程和IO线程

一个虚拟机实际上是内存中的一个qemu进程，vcpu和io线程就在qemu进程中。一开始我们已经说过，qemu被kvm项目组的人优化过，那么优化了什么？

在解释全虚拟化的时候，我们说过hypervisor会执行动态二进制翻译过程，结果是导致虚拟机性能低下，为了解决这个问题，kvm项目组在优化和整合qemu的时候，使它可以直接和kvm模块交互，从而可以***不使用二进制翻译就能安全地执行虚拟机的指令\***。

既然kvm和qemu的运行级别不同，那么它们是如何通讯的呢？答案是`/dev/kvm`设备。只要加载了kvm模块，这个设备就存在，通过这个设备，qemu就可以实现`ioctl`操作。

# libvirt

libvirt是一个应用程序编程接口，它提供了一系列的API，便于我们和hypervisor通讯。简单来说，libvirt包含一个守护进程（libvirtd）和一系列hypervisor管理工具。基于libvirt开发的程序有virsh、ovirt、virt-manager等。它不仅可以用于kvm/qemu，还支持Xen、LXC、Vmware ESX等。

libvirt不仅可以管理本地节点，通过`--connect`参数，还可以管理远程节点。以virsh为例子，可用于virsh的connection URI有两种类型：

- qemu://xxxxx/system
- qumu://xxxxx/session

第一个URI用于连接本地虚拟机节点，且请求连接的用户是root，第二个URI同样用于连接本地节点，但它的请求连接用户则是一个普通用户。

例如：

```
virsh -c qemu:///system list  
```

对于远程节点，connection URI格式如下：

```
driver[+transport]://[username@][hostname][:port]/[path][?extraparameters] 
```

例如：

```
virsh --connect qemu+ssh://root@remoteserver.yourdomain.com/system list --all 
```

但是请不要误解：libvirt连接的只是hypervisor，而不是虚拟机。
