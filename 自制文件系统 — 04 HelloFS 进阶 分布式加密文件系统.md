> 原文链接：https://www.qiyacloud.cn/2021/06/2021-06-21/

## 前情提要

在上篇文章我们已经通过一份完整的代码实现了一个 `hello world` 的文件系统。这个文件系统向你展示了一段有趣的 IO 路径，想必读者朋友也注意到了，这个文件系统是一个只读的文件系统，不支持写。文件 hello 的内容并不是存储在磁盘上，而是直接由程序返回。

今天我们将扩展这个 `hellofs` 文件系统，增强三个功能：

- 文件系统增加可写的功能；
- 存储加解密；
- 分布式存储数据；

所以，我们将完成一个分布式加密的 `hellofs` 极简文件系统。期待吗？

## 三大功能增强

### 增加写功能

上篇演示的 `hellofs` 是个只读的文件系统，要知道文件系统是管理数据的存储和读取的，不能写数据的文件系统是不完整的。

```sh
root@ubuntu:~# echo test > /mnt/myfs/hello
-bash: echo: write error: Input/output error
```

我们要给 `hellofs` 增加一个写入的支持，怎么增加？

**划重点：只需要给 `File` 对象实现 `Write` 接口即可。**

```go
func (File) Write(ctx context.Context, req *fuse.WriteRequest, resp *fuse.WriteResponse) error {
    // 接收 IO 请求，把数据存储到文件
    err := ioutil.WriteFile("/tmp/hellofs.01", req.Data, 0666)
    if err != nil {
        return err
    }
    // 写成功之后设置 size
    resp.Size = len(req.Data)
    return nil
}
```

如上，把写请求收到的数据写入到 `/tmp/hellofs.01` 这个文件

### 增加加密的功能

加密是什么意思？**就是让别人看不懂！**

目的是什么？**为了数据安全**。

一般来讲，敏感数据不能明文存储，这样就算数据被盗取，也看不懂内容，这样损失就可控。

举个例子，小明每天写日记（旁白：正经人谁写日记），有天日记被人偷偷看了，大概率人设就要蹦了。但是，如果小明是用的甲骨文来写，那就算日记被偷也没人看得懂，损害最多也就丢个笔记本，于人设无害。

对于存储数据来讲，加密的方法有很多，但有一个原则一定要准守：**加密算法一定是可逆的**。这个原则应该很容易理解吧？加密只是为了防盗，但存储数据自然是为了读取数据，如果用了顶天牛逼的算法存储到磁盘，读的时候却解密不出原数据，那就尴了个尬了。

> **题外小思考**：这个和网站存储密码有不一样哦，网站存储密码用不可逆的算法加密，你知道原因吗？ **原因**：密码校验相等并不是直接对比密码字符串是否相等，而是对比**密码 hash 之后的值**是否相等哦。

好，我们已经明白了加密的目的和形式。为了演示方便，我们不准备使用牛逼的加密算法，我们就用 base64 算法对原数据做一些序列化之后再写入文件（让它看起来像个乱码），然后读取的时候用 base64 算法反序列化下就可以读到原始数据了。

```go
str := "hello world"
encoded := base64.StdEncoding.EncodeToString([]byte(str))
```

加密效果：

```sh
hello world  =>  aGVsbG8gd29ybGQ=
```

这些乱码一样的字符串感觉还不错，是不是第一眼看不出是个啥！感兴趣的可以找个 base64 编解码的网站试下，也是一样的效果哦。

### 增加分布式的属性

分布式这是个很大的话题，本质就是**把一些离散的节点组织成一个有机的整体**。一般分布式架构有三大组件：

- Client 客户端
- Meta 元数据中心
- Server 服务端

为了简化处理，本次去掉元数据中心，**使用最经典的 C/S 的结构，演示一个最简单的分布式文件系统。** 我们通过 Client 解析 Fuse 协议得到 IO 请求包，然后转发给 Server 节点进行存储。你将会发现，一个貌似本地的文件系统，**数据竟然存储到别的机器上**。

我们把上篇 [Go实战：hello world 的文件系统](https://www.qiyacloud.cn/2021/06/2021-06-21/) 的 `helloworld.go` 文件扩展、拆分成两个二进制文件，`hellofs-client.go` 和 `hellofs-server.go` ，一个做 Client ，一个做 Server，分工如下：

- Client ：`hellofs-client.go` 文件，负责监听 fuse 请求，并且解析出来，加密之后转发到 Server ，也负责接收 Server 到响应，解密之后，回复 fuse 响应；
- Server ：`hellofs-server.go` 文件，负责监听 Client 发过来的请求，从磁盘中读写文件；

**模块功能示意图**：

![8ff756020622bfd7382636bb299c99e1.png](https://www.qiyacloud.cn/posts/2021-06-21/1.png)

## Go 代码实战

好啦，现在我们看下客户端和服务端的代码怎么写？

注意：我们这里为了突出文章重点，减少代码篇幅，我们这里只贴出重点片段代码。但是奇伢已经给各位准备好了可编译的完整代码。**关注公众号，回复 `自制文件系统` 即可获取。**

### hellofs-client

首先，我们看一眼增加的 `Write` 接口的实现：

```go
// hellofs-client.go
// 实现了这个接口就有了写的功能
func (File) Write(ctx context.Context, req *fuse.WriteRequest, resp *fuse.WriteResponse) error {
    // 写请求转发到 hellofs-server 
    _, err := http.Post(serverNodeAddr+"/hello/write", "text/plain", bytes.NewReader(req.Data))
    if err != nil {
        return err
    }
    // 写成功了，则设置长度
    resp.Size = len(req.Data)
    return nil
}
```

小结：

1. 实现了 `Write` ，则为文件系统增加了写功能；
2. 内部实现非常简单，直接代理给 Server 进程，走 Http 协议。比如，写请求就直接转发给 Server 的 `/hello/write` 接口；

### hellofs-server

`hellofs-server.go` 实现了一个 Http Server，接收并处理 `hellofs-client.go` 转发过来的请求。我们先看一个 `Write` 实现：

```go
// hellofs-server.go
// Server 实现了一个 http server

func main() {
    // 。。。。
    http.HandleFunc("/hello/write", hellofsWrite)

    log.Println("start")
    err := http.ListenAndServe(":8899", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}

func hellofsWrite(w http.ResponseWriter, r *http.Request) {
	log.Printf("hellfs server get req=> write")
	data, err := ioutil.ReadAll(r.Body)
	if err != nil {
		w.WriteHeader(500)
	}
	// 加密用户数据
	encryptStr := base64.StdEncoding.EncodeToString(data)
	// 存储加密后的数据
	err = ioutil.WriteFile(helloFilePath, []byte(encryptStr), 0666)
	if err != nil {
		w.WriteHeader(500)
	}
	return
}
```

## 我们测试下吧

### 步骤一：编译二进制文件

首先，把 `hellofs-client.go` （ Client ） 和 `hellofs-server.go` （ Server ）编译出来，命令如下：

**编译客户端**

```sh
root@ubuntu:~/code/gopher/src/myfs# go build -gcflags "-N -l" ./hellofs-client.go
```

**编译服务端**

```sh
root@ubuntu:~/code/gopher/src/myfs# go build -gcflags "-N -l" ./hellofs-server.go
```

生成 `hellofs-client`，`hellofs-server` 这两个二进制文件。

### 步骤二：挂载文件系统（hellofs 的客户端）

这个步骤在上一篇也有介绍过，目的是为了启动 hellofs 的客户端，监听 FUSE 请求。

```sh
root@ubuntu:~/code/gopher/src/myfs#  ./hellofs-client --mountpoint=/mnt/myfs --fuse.debug=true
```

进程启动，没有任何报错，就可以走下一步了。

### 步骤三：启动 hellofs 的服务端

这个服务端和客户端是走网络传输数据的，`hellofs-server` 可以运行在任何地方，**不需要和 `hellofs-client` 部署在同一个机器上**。 只需要 `hellofs-client.go` 里面的 `serverNodeAddr` 正确配置到 `hellofs-server` 运行的地址即可。

这也是**分布式的特点：通过网络把多个节点连接成一个有机的整体。**

注意：笔者为了简单，是部署在一台机器上，走 127.0.0.1 的 loop 网络：

```go
// 如果你有多个机器节点，那么可以试下把 hellofs-server 放到其他节点运行哦，效果更佳
var (
    serverNodeAddr = "http://localhost:8899"
)
```

好啦，尝试启动服务端程序吧：

```sh
root@ubuntu:~/code/gopher/src/myfs#  ./hellofs-server
2021/06/20 11:18:51 start
```

至此，我们的环境就已经完全准备好了。

### 开始测试吧

`ls -l` 看一下自制文件系统下的文件吧，我们看到 hello 的权限变成了 666（有可写权限啦）。

```sh
root@ubuntu:~# ll /mnt/myfs/hello 
-rw-rw-rw- 1 root root 0 Jun 20 11:20 /mnt/myfs/hello
```

`cat` 一下，内容啥都没有。

```sh
root@ubuntu:~# cat /mnt/myfs/hello
```

尝试写一些字符串进去吧，如下，我把 “qiyacloud” 这个字符串 `echo` 写入文件。

```sh
root@ubuntu:~# echo "qiyacloud" > /mnt/myfs/hello
```

**这就是写成功了哦。** 我们看到 `hellofs-server` 有收到写请求的日志哦。

```sh
root@ubuntu:~/code/gopher/src/myfs# ./hellofs-server 
2021/06/20 11:18:51 start
......
2021/06/20 11:25:50 hellfs server get req=> write
```

**这也说明了分布式节点直接网络通信是成功的哦。**

好，现在我们读一下 hello 文件，看看有什么内容。

```sh
root@ubuntu:~# cat /mnt/myfs/hello 
qiyacloud
```

**读到了正确的数据呢**，“qiyacloud”正是我们之前写入的内容。

**重点来了**：好，现在探索一个究极问题，我们写入的数据存在那里？

我们找到 `hello-server` 进程所在的目录，发现目录下有一个 `qiya.hellofs.001` 的文件。

```sh
root@ubuntu:~/code/gopher/src/myfs# ls -l
total 15076
...
-rwxr-xr-x 1 root root 7459058 Jun 20 11:18 hellofs-server
-rw-r--r-- 1 root root      16 Jun 20 11:25 qiya.hellofs.001
```

我们 cat 下这个文件，看下里面的内容。

```sh
root@ubuntu:~/code/gopher/src/myfs# cat qiya.hellofs.001 
cWl5YWNsb3VkCg==
```

这个文件存储的是 “cWl5YWNsb3VkCg==” 这个字符串，这个字符串就是 “qiyacloud” 的经过 base64 编码之后的内容。

![7ee1585319c0fd0a31c46a0fcdb99c41.png](https://www.qiyacloud.cn/posts/2021-06-21/2.png)

所以，我们观察到两个事实：

1. 当我们 echo “qiyacloud” 这个字符串写入到一个本地文件的时候，发现字符串竟然跨越内核，跨越节点，存储到了 `hello-server` 所在的目录的 `qiya.hellofs.001` 文件里；
2. “qiyacloud” 不是明文存储，而是经过 base64 算法加密存储到文件的，但这个行为对用户不感知；

至此，我们的分布式加密文件系统就完美完成啦。

**看一眼完整的 IO 请求流程，如下图**：

![0ff02fa131bd4a171979a3560ac630de.png](https://www.qiyacloud.cn/posts/2021-06-21/3.png)

## 总结

1. `File` 实现 `Write` 接口之后，就能承接 vfs 调用 write 的逻辑，从而文件系统就有了写的能力；
2. 用 base64 编码来加密用户数据，这样就能让人一眼看不出来原始数据了，这样的加密简单吧；
3. 自制的 hellfs 采用 C/S 结构，从而成为了极简分布式的样子；
4. `hellofs-client` 从内核 `/dev/fuse` 通道中获取消息，解析 FUSE 协议，然后把 IO 请求通过 Http 协议转发到 Server 节点；
5. `hellofs-server` 接收 `hellofs-client` 网络传输的数据，加密之后存储到文件。读取请求的时候，读取文件，base64 解密之后，回传给 `hellofs-client`；

## 后记

自制 FS 演示的是极简的情况，核心是让童鞋们直观的感受**文件、分布式、加密**的一些概念，省略了复杂的异常处理。大家要注意，如果是生产环境的编码，远不止如此哦。

为了突出文章重点，减少代码篇幅，文章内只贴出重点片段。**关注公众号，回复 “`自制文件系统`” 即可获取可编译的完整代码**，`hellofs-client.go` ，`hellofs-server.go`。

所以，分布式加密的 hellofs 你跑起来了吗？
