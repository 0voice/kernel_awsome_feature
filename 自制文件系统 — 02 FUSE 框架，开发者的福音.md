> 原文链接：https://www.qiyacloud.cn/2021/05/2021-05-31/

## 前情提要

前文介绍了一些探索文件系统的命令和方法，并且提到内核文件系统开发的艰难，笔者说过要带读者朋友一起动手做一个极简的文件系统。

但笔者总不能带娇嫩的读者趟一次内核的浑水吧？但奇伢的话已经放出去了，总不能食言，怎么办？

巧了，这个问题内核开发者也苦思冥想过，最终提供了一套名为 FUSE 的框架，支持用户在用户态制作文件系统。

希望看完这篇文章，大家对以下问题有所了解：

1. FUSE 是什么？
2. FUSE 能做什么？
3. FUSE 的实现有哪些？

本篇文章旨在帮大家在心里建立一个 FUSE 框架的模型，并且熟悉用户态文件系统整条 IO 路径。

## FUSE 是什么？

[Linux内核官方文档](https://www.kernel.org/doc/html/latest/filesystems/fuse.html)对 FUSE 的解释如下：

> What is FUSE? FUSE is a userspace filesystem framework. It consists of a kernel module (fuse.ko), a userspace library (libfuse.*) and a mount utility (fusermount).

**划重点：FUSE 是一个用来实现用户态文件系统的框架**，这套 FUSE 框架包含 3 个组件：

1. **内核模块 `fuse.ko`** ：用来接收 vfs 传递下来的 IO 请求，并且把这个 IO 封装之后通过管道发送到用户态；
2. **用户态 lib 库 `libfuse`** ：解析内核态转发出来的协议包，拆解成常规的 IO 请求；
3. **mount 工具 `fusermount`** ；

这就是 FUSE 框架的 3 大内容了，下面我们解释下。这 3 个组件只为了完成一件事：让 IO 在内核态和用户态之间自由穿梭。

一般我们认为 FUSE 是 Filesystem in Userspace 的缩写，也就是常说的用户态文件系统。

## FUSE 原理

接下来我们看下 IO 的路径，来理解下 FUSE 的原理。首先看一眼 wiki 上有对 FUSE 的 `ls -l /tmp/fuse` 命令的演示图：

![0bf5cb5d5b1ad55e89089cd274988b62.png](https://www.qiyacloud.cn/posts/2021-05-31/1.png)

这个图的意思是：

1. 背景：一个用户态文件系统，挂载点为 `/tmp/fuse` ，用户二进制程序文件为 `./hello` ；

2. 当执行

    

   ```
   ls -l /tmp/fuse
   ```

    

   命令的时候，流程如下：

   1. IO 请求先进内核，经 vfs 传递给内核 FUSE 文件系统模块；
   2. 内核 FUSE 模块把请求发给到用户态，由 `./hello` 程序接收并且处理。处理完成之后，响应原路返回；

简化的 IO 动画示意图：

![b57128d4d7140ced9a3a74e680bd5da2.gif](https://www.qiyacloud.cn/posts/2021-05-31/2.gif)

通过这两张图，对 FUSE IO 的流程应该就清晰了，内核 FUSE 模块在内核态中间做协议包装和协议解析的工作。承接 vfs 下来的请求并按照 FUSE 协议转发到用户态，然后接收用户态的响应，回复给用户。

FUSE 在这条 IO 路径是是指做了一个透明中转站的作用，用户完全不感知这套框架。我们把中间的 FUSE 当作一个黑盒遮住，就更容易理解了。

![4f25c73748b2eddff422b50706bc49cb.gif](https://www.qiyacloud.cn/posts/2021-05-31/3.gif)

**思考问题：内核的 fuse.ko 模块，还有 libfuse 库。这两个角色是的作用？**

划重点：这两个模块一个位于内核，一个位于用户态，是配套使用的，**最核心的功能是协议封装和解析**。

举给例子，内核 fuse.ko 用于承接 vfs 下来的 io 请求，然后封装成 FUSE 数据包，转发给用户态。这个时候，用户态文件系统收到这个 FUSE 数据包，它如果想要看懂这个数据包，就必须实现一套 FUSE 协议的代码，这套代码是公开透明的，属于 FUSE 框架的公共的代码，这种代码不能够让所有的用户文件系统都重复实现一遍，**于是 libfuse 用户库就诞生了**。

回到开篇的问题，FUSE 能做什么？

看到这里你应该就清晰了，FUSE 能够转运 vfs 下来的 io 请求到用户态，用户程序处理之后，经由 FUSE 框架回应给用户。**从而就可以把文件系统的实现全部放到用户态实现了**。

### FUSE 协议格式

我们分析一眼 FUSE 数据转运的数据格式（ fuse 协议的格式 ），请求包和响应包是什么样子的呢？好奇不？

#### FUSE 请求包

FUSE 请求包分为两部分：

1. `Header` ： 这个是所有请求共用的，比如 `open` 请求，`read` 请求，`write` 请求，`getxattr` 请求，头部都至少有这个结构体，`Header` 结构体能描述整个 FUSE 请求，其中字段能区分请求类型；
2. `Payload` ：这个东西是每个 IO 类型会是不同的，比如 `read` 请求就没这个，`write` 请求就有这个，因为 `write` 请求是携带数据的；

数据包分为两部分：header，payload 。

```go
type inHeader struct {
	Len    uint32
	Opcode uint32
	Unique uint64
	Nodeid uint64
	Uid    uint32
	Gid    uint32
	Pid    uint32
	_      uint32
}
```

- Len: 是整个请求的字节数长度（`Header` + `Payload`）
- Opcode: 请求的类型，比如区分 open、read、write 等等；
- Unique: 请求唯一标识（和响应中要对应）
- Nodeid: 请求针对的文件 nodeid，目标文件或者文件夹的 nodeid；
- Uid: 文件/文件夹操作的进程的用户ID
- Gid: 文件/文件夹操作的进程的用户组ID
- Pid: 文件/文件夹操作的进程的进程ID

#### FUSE 响应包

FUSE 响应包也分为两部分：

- `Header` ：这个结构体也是在数据头部的，所有 IO 类型的响应都至少有这个结构体。该结构体用于描述整个响应请求；
- `Payload` ：每个请求的类型可能不同，比如 `read` 请求就会有这个，因为要携带 `read` 出来的用户数据，`write` 请求就不会有；

```go
type outHeader struct {
	Len    uint32
	Error  int32
	Unique uint64
}
```

- Len: 整个响应的字节数长度（ `Header` + `Payload` ）；
- Error: 响应错误码，成功返回 0，其他对应着系统的错误代码，负数；
- Unique: 对应者请求的唯一标识，和请求对应；

### 内核态、用户态的纽带

现在对数据协议的格式，转发和转运的模块我们也知道了。现在还差一个关键的点：**数据包的通道**，也就是高速公路。

换句话说，内核模块的“包裹”发到哪里？用户程序又从哪里读取拿到这个“包裹”。

**答案是：`/dev/fuse` ，这个虚设备文件就是内核模块和用户程序的桥梁。**

一切都顺理成章了，内核在这个过程中相当于一个信使，用户的 io 通过正常的系统调用进来，走到内核文件系统 fuse ，fuse 文件系统把这个 io 请求封装起来，打包成特定的格式，通过 `/dev/fuse` 这个管道传递到用户态。在此之前有守护进程监听这个管道，看到有消息出来之后，立马读出来，然后利用 `libfuse` 库解析协议，之后就是用户文件系统的代码逻辑了。

示意图如下（省略了拆解包的步骤）：

![65f12efab48144a0858b06faf21f6773.gif](https://www.qiyacloud.cn/posts/2021-05-31/4.gif)

### FUSE 的使用

现在我们知道了 FUSE 框架的 3 大组件，FUSE 的数据包协议，现在就尝试着使用一下 FUSE 文件系统。

> 提示，以下命令在 ubuntu 16 版本上执行的。

#### Linux 内核是否支持？

前面说过内核里面也有一个 fuse.ko 模块，这个模块是公用的，内核的位置也是位于文件系统层。我们想要自制一个文件系统，那么第一步需要确保内核支持这个模块。可以直接运行如下命令，如果没有报错，说明你的 Linux 机器支持 fuse 模块，并且已经加载。

```sh
root@ubuntu:~# modprobe fuse
```

如果当前 Linux 不支持这个内核模块，那么就会报错，比如（ubuntu16）：

```sh
root@ubuntu:~# modprobe xyz
modprobe: FATAL: Module xyz not found in directory /lib/modules/4.4.0-142-generic
```

或者也可以去目录 `/lib/modules/4.4.0-142-generic/kernel/fs/` 里看是否有 fuse 这个目录。

**这些前置知识，在上一篇文章有提到哈。**

#### 挂载 fuse 内核文件系统，便于管理

fuse 这个内核文件系统其实是可以挂载，也可以不挂载，挂载了主要是方便管理多个用户系统而已，fuse 内核文件系统的 Type 名称为 `fusectl`，挂载命令：

```sh
mount -t fusectl none /sys/fs/fuse/connections
```

可以用 `df -aT` 命令查看：

```sh
root@ubuntu:~# df -aT|grep -i fusectl
fusectl                     fusectl              0        0         0    - /sys/fs/fuse/connections
```

通过挂载内核 fuse 文件系统，可以看到所有实现的用户文件系统，如下：

```sh
root@ubuntu:~# ls -l /sys/fs/fuse/connections/
total 0
dr-x------ 2 root root 0 May 29 19:58 39
dr-x------ 2 root root 0 May 29 20:00 42
```

在 `/sys/fs/fuse/connections` 对应两个目录，目录名为 `Unique ID`，能够唯一标识一个用户文件系统。这里表示内核 fuse 模块通过 `/dev/fuse` 设备文件，建立了两个通信管道，分别对应了两个用户文件系统，可以在用 `df -aT` 对照确认：

```sh
root@ubuntu:~# df -aT|grep -i fuse
fusectl                     fusectl              0        0         0    - /sys/fs/fuse/connections
lxcfs                       fuse.lxcfs           0        0         0    - /var/lib/lxcfs
helloworld                  fuse.hellofs         0        0         0    - /mnt/myfs
```

每个 Uniqe ID 名录下，有若干个文件，通过这些文件，我们可以获取到当前用户文件系统的状态，或跟 fuse 文件系统交互，比如：

```sh
root@ubuntu:~# ls -l /sys/fs/fuse/connections/42/
total 0
--w------- 1 root root 0 May 29 20:00 abort
-rw------- 1 root root 0 May 29 20:00 congestion_threshold
-rw------- 1 root root 0 May 29 20:00 max_background
-r-------- 1 root root 0 May 29 20:00 waiting
```

- waiting 文件：cat 一下就能获取到当前正在处理的 IO 请求数；
- abort 文件：该文件写入任何字符串，都会终止这个用户文件系统和上面所有的请求；

#### 用户文件系统怎么挂载？

现在只剩最后一个问题：用户文件系统怎么挂载（比如上面的 `hellofs` 和 `lxcfs` ）？

这就用到了 FUSE 框架的第 3 个组件了，`fusermount` 工具，这个工具就是专门用来方便挂载用户文件系统才诞生的。

```sh
fusermount -o fsname=helloworld,subtype=hellofs -- /mnt/myfs/
```

FUSE 的作用在于使用户能够绕开内核代码来编写文件系统，但是请注意，文件系统要实现对具体的设备的操作的话必须要使用设备驱动提供的接口，而设备驱动位于内核空间，这时可以直接读写块设备文件，**就相当于只把文件系统摘到用户态，用户直接管理块设备空间**。

## FUSE 有哪些？

实现了 FUSE 的用户态文件系统有非常多的例子，比如，GlusterFS，SSHFS，CephFS，Lustre，GmailFS，EncFS，S3FS、、、等等

上面这些都是实现了 fuse 的用户态程序：

- GmailFS 可以让我们管理文件一样，管理邮件；
- S3FS 可以让我们管理文件一样，管理对象；

## 总结

通过这篇文章，我们了解了 FUSE 的知识点，总结如下：

1. FUSE 框架就是内核开发者为了日益多样的用户需求开发出来的，使得用户态程序参与到 IO 路径的处理成为可能；
2. FUSE 框架的 3 大组件分别是：内核 fuse 模块，用户态 `libfuse` 库，`fusermount` 挂载工具；
3. 内核 fuse 模块用于承接 vfs 的请求，并且通过 `/dev/fuse` 建立的管道，把封装后的请求发往用户态；
4. `libfuse` 则是用户态封装用来解析 FUSE 数据包协议的库代码，服务于所有的用户态文件系统；
5. `fusermount` 则是用户态文件系统用来挂载的工具而已；
6. `/dev/fuse` 就是连接内核 fuse 和用户态文件系统的纽带；
7. 上面动图演示了 FUSE 文件系统的完整 IO 路径，你学fei了吗？

## 后记

你已经准备好自制文件系统的全部前置条件了，包括怎么查看，学习文件系统，挂载和卸载。也了解了 FUSE 框架，接下来就是搞一个真正可用的文件系统了，敬请期待。

