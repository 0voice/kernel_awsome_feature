> 原文链接：https://www.qiyacloud.cn/2021/05/2021-06-07/

## 前情提要

终于到了动手的环节，今天我们直接搞起一个叫做 hello world 的文件系统，附上全部代码实现，且可以体验测试。

## 环境准备

环境准备：

1. go 编程环境（准备个 go 1.13 以上版本的环境即可）
2. 随便搞一台 Linux 虚拟机（支持 fuse）

确认 Linux 内核支持 FUSE

```sh
root@ubuntu:~# modprobe fuse
```

命令没有报错的话，就说明内核支持 fuse ，并且已经加载。

## 开始自制文件系统

### 第一步：解析 FUSE 协议

在 02 FUSE 框架篇我们介绍了 FUSE 协议，说到了 FUSE 框架的 3 组件：内核 fuse 文件系统，用户态 libfuse 库，fusermount 工具。

内核的 fuse 文件系统只有一份，用于承接 vfs 请求，封装成 FUSE 协议包，走 `/dev/fuse` 建立起来的通道转发用户态。用户态的任务就是把 FUSE 协议包解析出来并且处理，然后把请求响应按照 FUSE 协议封装起来，走 `/dev/fuse` 通道传回内核，由 vfs 传回用户。

所以，我们看到**用户态 libfuse 库**这个东西其实就只是 FUSE 协议解析和封装用的。童鞋们，注意啦，重点来了。**划重点：只要是`数据协议`，就有一个特点：和`具体语言无关`。数据协议格式只不过是对字节流的分析方式而已**

FUSE 的协议也是如此，libfuse 这个是用 c 语言实现的 FUSE 协议库，官方的 Github 地址是：https://github.com/libfuse/libfuse/ 。Go 语言不能直接用 libfuse 库，因为 libfuse 库全都是封装成了 c 的结构体。

这是我们要迈过的第一道关，就是用 Go 语言来解析 FUSE 协议。好吧，准备开始啦。首先看一下 FUSE 的数据包格式：

![24ef490cc016a3db6051a97acd7ad06f.png](https://www.qiyacloud.cn/posts/2021-06-07/1.png)

**先思考一个问题：libfuse 做了哪些事情？**

- 然后，要和建立 `/dev/fuse` 通道；
- 然后，要实现一个 Server 服务端，监听这个通道，这样就和内核 fuse 建立了联系，接收和发送消息；
- 然后，对不同的请求做不同的解析；
  - 比如 read 的 Opcode 是，write 的 Opcode 是 ；
  - write 请求携带用户数据，其他请求不携带；
- 然后，把解析好的 FUSE IO 请求转发给用户态文件系统；
- 然后，接收用户态文件系统的 IO 响应，封装成 FUSE 响应格式；
- 然后，不同的请求有不同的响应，做不同的处理，算了吧，太麻烦了，
  - read 请求的响应携带用户数据，其他请求则不同；

Go 的 FUSE 协议库也要做以上这些东西。其实，**任何一种协议数据格式的解析从来都是索然无味的**，因为代码实现的逻辑功能是确定的。这里推荐一个 Go 的 FUSE 库：[bazil/fuse](https://github.com/bazil/fuse)，这是一个纯 Go 写的 FUSE 协议解析库，作用和 libfuse 这个纯 c 语言写的库作用完全一样。

> `bazil.org/fuse` is a Go library for writing FUSE userspace filesystems.

有了这个 Go FUSE 协议解析库，就可以开始写文件系统的程序了。我们自己能参与创造的部分才是真正感兴趣的。

### 第二步：Go 自制文件系统

我们下面写了一个 `helloworld` 的文件系统，首先说结论，`hellofs` 实现了以下功能：

1. 挂载点根目录下面只有一个叫做 `hello` 的文件（注意：不需要用户创建哦，直接挂载之后就有了）；
2. `cat` 这个 `hello` 将会返回 `hello, world` 的内容；
3. 挂载点目录的属性：inode 为 20210601，mode 为 555；
4. `hello` 文件的属性：inode 为 20210606，mode 为 444；

跟我一起创建出一个叫做 helloword.go 的文件，写入下面的代码：

```go
// 实现一个叫做 hellfs 的文件系统
package main

import (
    "context"
    "flag"
    "log"
    "os"
    "syscall"

    "bazil.org/fuse"
    "bazil.org/fuse/fs"
    _ "bazil.org/fuse/fs/fstestutil"
)

func main() {
    var mountpoint string
    flag.StringVar(&mountpoint, "mountpoint", "", "mount point(dir)?")
    flag.Parse()

    if mountpoint == "" {
        log.Fatal("please input invalid mount point\n")
    }
    // 建立一个负责解析和封装 FUSE 请求监听通道对象；
    c, err := fuse.Mount(mountpoint, fuse.FSName("helloworld"), fuse.Subtype("hellofs"))
    if err != nil {
        log.Fatal(err)
    }
    defer c.Close()

    // 把 FS 结构体注册到 server，以便可以回调处理请求
    err = fs.Serve(c, FS{})
    if err != nil {
        log.Fatal(err)
    }
}

// hellofs 文件系统的主体
type FS struct{}

func (FS) Root() (fs.Node, error) {
    return Dir{}, nil
}

// hellofs 文件系统中，Dir 是目录操作的主体
type Dir struct{}

func (Dir) Attr(ctx context.Context, a *fuse.Attr) error {
    a.Inode = 20210601
    a.Mode = os.ModeDir | 0555
    return nil
}

// 当 ls 目录的时候，触发的是 ReadDirAll 调用，这里返回指定内容，表明只有一个 hello 的文件；
func (Dir) Lookup(ctx context.Context, name string) (fs.Node, error) {
    // 只处理一个叫做 hello 的 entry 文件，其他的统统返回 not exist
    if name == "hello" {
        return File{}, nil
    }
    return nil, syscall.ENOENT
}

// 定义 Readdir 的行为，固定返回了一个 inode:2 name 叫做 hello 的文件。对应用户的行为一般是 ls 这个目录。
func (Dir) ReadDirAll(ctx context.Context) ([]fuse.Dirent, error) {
    var dirDirs = []fuse.Dirent{{Inode: 2, Name: "hello", Type: fuse.DT_File}}
    return dirDirs, nil
}

// hellofs 文件系统中，File 结构体实现了文件系统中关于文件的调用实现
type File struct{}

const fileContent = "hello, world\n"

// 当 stat 这个文件的时候，返回 inode 为 2，mode 为 444
func (File) Attr(ctx context.Context, a *fuse.Attr) error {
    a.Inode = 20210606
    a.Mode = 0444
    a.Size = uint64(len(fileContent))
    return nil
}

// 当 cat 这个文件的时候，文件内容返回 hello，world
func (File) ReadAll(ctx context.Context) ([]byte, error) {
    return []byte(fileContent), nil
}
```

简单说下上面做了什么事情：

1. 定义了根目录 `readdir` 和 `getattr` 的行为回调；
2. 定义了 hello 文件的 `readall` 和 `getattr` 的行为回调；

### 第三步：让文件系统 Go 起来

好，激动人心的时候到了，我们先编译出这个程序，然后跑起来就是可用一个极简的文件系统了。麻雀虽小，五脏俱全。

#### 编译 helloworld.go

```sh
root@ubuntu:~/gopher/src# go build -gcflags "-N -l" ./helloworld.go 
```

成功编译，获得二进制文件 `helloworld` 。

#### 创建一个空目录

创建一个空目录当做挂载点，笔者是在 `/mnt` 目录下创建了一个叫做 `myfs` 的目录。

```sh
root@ubuntu:~# mkdir  /mnt/myfs/
```

#### 挂载运行

好，现在我们**用户文件系统**程序准备好了，挂载点也准备好了，万事俱备了，可以运行了。命令如下：

```go
root@ubuntu:~/gopher/src# ./helloworld --mountpoint=/mnt/myfs --fuse.debug=true
```

参数说明：

- `mountpoint` ：指定挂载点目录，也就是上面创建的空目录 `/mnt/myfs/` ；
- `fuse.debug` ：为了更好的理解用户文件系统，可以把这个开关设置成 `true` ，这样**用户发送的请求对应了后端什么逻辑**就一目了然了；

测试跑起来之后，如果没有任何异常，`helloworld` 就是作为一个守护进程，卡主执行，没有任何日志。直到收到请求。

这个时候，我们这个终端窗口就不要动了（待会可以看日志），再新开一个终端用来测试。

### 第四步：极简文件系统 `hellofs` 的测试

#### 系统角度探测

现在我们从多个角度测试下 hellofs ，感受下自己做的第一个用户文件系统是什么样子的。

首先，文件系统一定要挂载才能用，所以 `df` 命令可以看到挂载情况：

```sh
root@ubuntu:~# df -aTh|grep hello
helloworld                  fuse.hellofs  0.0K  0.0K  0.0K    - /mnt/myfs
```

看到了不？有一个叫做 `helloworld` ，类型为 `fuse.hellofs` 的文件系统。这两个名字都是代码里指定的。

然后，如果挂载了 fusectl 文件系统（内核 fuse 文件系统），那么还可以在 `/sys/fs/fuse/connections` 看到比以前多一个数字命名的目录。

#### 文件操作探测

我们通过 ls，stat，cat 等命令对 hellofs 文件系统探测一下。

**第一个问题：`stat` 查看一下挂载点 `stat /mnt/myfs` ？能得到什么数据？**

```sh
root@ubuntu:~# stat /mnt/myfs/
  File: '/mnt/myfs/'
  Size: 0         	Blocks: 0          IO Block: 4096   directory
Device: 29h/41d	Inode: 20210601    Links: 1
Access: (0555/dr-xr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2021-06-06 13:49:02.463926775 +0800
Modify: 2021-06-06 13:49:02.463926775 +0800
Change: 2021-06-06 13:49:02.463926775 +0800
 Birth: -
```

我们看到特殊的 **inode：20210601，权限：555** （回忆下上面的代码实现，目录的 inode 为 20210601 就是我们指定的）。如下：

注意，在 `stat /mnt/myfs` 的同时，用户文件系统会打印出日志：

```sh
root@ubuntu:~/code/gopher/src/myfs# ./helloworld --mountpoint=/mnt/myfs --fuse.debug=true
2021/06/06 13:49:04 FUSE: <- Getattr [ID=0x2 Node=0x1 Uid=0 Gid=0 Pid=891] 0x0 fl=0
2021/06/06 13:49:04 FUSE: -> [ID=0x2] Getattr valid=1m0s ino=20210601 size=0 mode=dr-xr-xr-x
```

这个日志明确的告诉了我们，先收到了一个 `Getattr` 的请求，请求参数是什么，然后 `hellofs` 处理完成之后，返回了什么样的响应。

**第二个问题：`ls /mnt/myfs` 的反应呢？**

```sh
root@ubuntu:~# ls -l /mnt/myfs/
total 0
-r--r--r-- 1 root root 13 Jun  6 13:49 hello
```

我们看到了一个 hello 的文件（体会一下，我们没有创建过这个文件哦）。

那么，`stat /mnt/myfs/hello` 会得到什么？

```sh
root@ubuntu:~# stat /mnt/myfs/hello 
  File: '/mnt/myfs/hello'
  Size: 13        	Blocks: 0          IO Block: 4096   regular file
Device: 29h/41d	Inode: 20210606    Links: 1
Access: (0444/-r--r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2021-06-06 13:49:02.463926775 +0800
Modify: 2021-06-06 13:49:02.463926775 +0800
Change: 2021-06-06 13:49:02.463926775 +0800
 Birth: -
```

我们看到了特殊的 inode 20210606。

**第三个问题：`cat /mnt/myfs/hello` 这个文件？**

```sh
root@ubuntu:~# cat /mnt/myfs/hello 
hello, world
```

我们看到 hello，world 的返回，虽然你从来没写过这个文件。

第四个问题：请大家体会一下 `hello` 这个文件，这个文件和你平时见的文件有什么区别呢？

这个是一个额外的问题，也是要读者朋友重点思考的一个问题。思考 hello 这个文件的特殊性：

- 这个文件明明你没有创建过，却出现了？
- 这个文件是存在磁盘的吗？
- 这个文件能写吗？

以上的问题你想通了吗？可以先思考下，或者找我交流。

**请记住，文件这个概念从来都是一个逻辑的对象。\**是文件系统给你的一个抽象的对象。换句话说，\*\*文件表现的任何信息\*\*都只是\**文件系统想要展现给你的而已**。

**你看到的只是 FS 想要你看到的而已！！！**

## 总结

1. FUSE 框架三大组件：内核 fuse 模块，用户态 FUSE 协议解析库，fusermount 工具；
2. FUSE 协议解析本身跟具体语言无关，c 可以实现，Go 可以实现，甚至 Python 都可以实现；
3. `libfuse` 是纯 c 实现的 FUSE 协议解析库，如果你想用 c 语言实现一个用户文件系统，那么选它就对了；
4. `bazil.org/fuse` 是纯 Go 实现的 FUSE 协议库，我们用 Go 语言实现用户文件系统，那么选它就对了；
5. 实现一个用户文件系统有多简单？只需要定义 FS，Dir，File 这三大结构的处理逻辑，以上我们实现了一个名叫 `hello，world` 的文件系统；

## 后记

这次实现了一个完整的 helloworld 用户态文件系统，最后留个思考题？实现 hellofs 之后，你理解**“文件”**是什么呢？

有了这一次的基础，下一次我们实现一个更复杂的文件系统：加密的分布式文件系统，敬请期待。
