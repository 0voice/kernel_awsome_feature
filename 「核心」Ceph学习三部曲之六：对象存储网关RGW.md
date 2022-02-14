本blog包括理论和实践两个部分，实践部分需要您事先部署成功Ceph集群！

参考《Ceph设计与实现》谢型果等，第六章。以及[官方RADOS指南](https://docs.ceph.com/en/latest/man/8/rados/)，以及[官方RGW教程](https://docs.ceph.com/en/latest/radosgw/)

# 1.概述

前面我们讲过，Ceph集成了BlueStore分布式对象存储，针对非结构化的数据，比如静态数据，备份存储以及流媒体等场景。上一节，我们介绍了RADOS 中的RBD模块，并且rados提供了API接口librados供用户使用。这一节我们将介绍RADOS Gateway , 即RADOS网关，它主要支持两种类型的接口：

- **与S3兼容**：为对象存储功能提供与Amazon S3 RESTful API的大部分子集兼容的接口。
- **兼容Swift**：为对象存储功能提供与OpenStack Swift API的大部分子集兼容的接口。

因为Ceph核心模块RADOS提供的访问接口是私有接口，不支持通用的HTTP协议访问，因而为了支持HTTP协议访问，涉及了支持RESTful接口访问而设计的RADOS gateway.

> **官方定义**：Ceph对象存储使用Ceph对象网关守护进程（`radosgw`），它是一个用于与Ceph存储群集进行交互的HTTP服务器。由于它提供与OpenStack Swift和Amazon S3兼容的接口，因此Ceph对象网关具有自己的用户管理。Ceph对象网关可以将数据存储在用于存储来自Ceph文件系统客户端或Ceph块设备客户端的数据的同一Ceph存储群集中。S3和Swift API共享一个公共的名称空间，因此可以使用一个API编写数据，而使用另一个API检索数据。
>
> 另外，**Ceph对象存储也未使用MDS服务器！**

[![img](https://durantthorvalds.top/img/1ae399f8fa9af1042d3e1cbf31828f14eb3fe01a6eb3352f88c3d2a04ac4dc50.png)](https://durantthorvalds.top/img/1ae399f8fa9af1042d3e1cbf31828f14eb3fe01a6eb3352f88c3d2a04ac4dc50.png)

RGW作为对象存储网关系统，一方面扮演RADOS集群客户端角色，为对象存储应用提供RESTful接口；另一方面，扮演HTTP角色，接收并解析互联网传输数据。RGW目前支持主流的WEB服务器：Civetweb，Apache，Nginx等，其中Civetweb是一个C++库，可以内嵌到RGW框架中，是RGW默认的WEB服务器；Apache与Nginx需要以独立进程存在，收到应用请求后，通过RGW注册的监听端口号将请求转发到RGW上进行处理。

## 1.1 数据组织

一个对象存储系统包括三个部分：用户、存储桶和对象。

- **用户**：指对象存储应用的使用者。一个用户拥有一个或多个存储桶。
- **存储桶**：是对象的容器，设置这一层级的目的是方便关联和操作具有同一属性的一类对象而引入的一层关联单元。
- **对象**：对象是存储的基本单位，包括数据和元数据两个部分。其中元数据在类型和数目上不受限制。与文件系统不同，对象存储系统中所有对象以扁平的方式存储，对象之间没有之间的关联。并且，对象存储不提供部分编辑功能，这意味着，即使更新一个字符，也必须将整个对象从云端下载下来，更新后上传。

以Amazon S3为例，它数据实体包括user、bucket、object，如下图所示；而OpenStack将用户的概念细分为account和user，其中account对应一个项目或者租户，每个account可以被多个user共享，其他的集成实体比如container和object与以上的存储桶、对象概念相符。

RGW为了兼容Amazon S3和OpenStack接口，所以将用户分为user和subuser，分别对应S3用户和Swift用户。

[![image-20210103132412544](https://durantthorvalds.top/img/image-20210103132412544.png)](https://durantthorvalds.top/img/image-20210103132412544.png)

[![image-20210103132643135](https://durantthorvalds.top/img/image-20210103132643135.png)](https://durantthorvalds.top/img/image-20210103132643135.png)

[![image-20210103132828794](https://durantthorvalds.top/img/image-20210103132828794.png)](https://durantthorvalds.top/img/image-20210103132828794.png)

我们将详细讨论，这些数据实体所包含的信息和数据组织形式，由上一期我们知道，数据存在RADOS有三种方式，第一种是二进制；第二种是以键值对存在扩展属性xattr中；第三种是存在扩展属性omap中。

## 1.2 用户

对用户的设计管理主要包含以下几个方面：首先是为了对RESTful API进行请求认为，其次是为了控制用户对存储资源的访问权限，最后是为了控制用户的可用存储空间，因此一个用户包含的信息包括用户认证信息、访问认证控制权限信息和配额信息。

我们首先介绍RGW的认证机制，RGW针对S3 API和Swift API采用不同的认证机制。

S3用户认证兼容AWS2和AWS4两种认证机制，它们都是基于密匙认证。

认证过程如下：

1）应用在发送请求之前，使用用户私有密匙（secret_key）、请求内容等，采用与RGW网关约定好的算法计算出数字签名后，将数字签名以及用户访问密匙（access_key）封装在请求中发送给RGW网关。

2）RGW网关收到请求后，使用用户访问密匙作为索引从RADOS集群中读取用户信息，并从用户信息中获取用户私有密匙。

3）使用用户私有密匙、请求内容等，采用与约定好的算法计算数字签名。

4）判断RGW生成的数字签名与请求的签名是否匹配，如果是匹配的，则认为请求是真实的，用户认证通过。

对于Swift，采用的是令牌认证(token)。

1）应用在发出真正的操作请求前，向RGW网关请求一个有时限的令牌。

2）RGW收到令牌后，使用子用户ID作为索引从RADOS集群中读取出子用户信息，并使用子用户信息中获取到的Swift私有密匙生成一个令牌返回给应用。

3）应用在后续的操作中携带该令牌，RGW收到操作请求后，采用与（2）相同的方式生成一个令牌，并判断生成的令牌与请求中的令牌是否一致，如果一致，身份验证通过。

值得注意的是，对于每种资源所要求的权限是不同的，用户必须具备相应的权限。

此外，为了防止某些用户占用太多的存储空间，以及方便根据付费分配空间，RGW允许对用户进行配额限制。

RGW使用`RGWUserInfo`管理元数据。

| 字段        | 意义                                                         |
| ----------- | ------------------------------------------------------------ |
| users.uid   | 在“ ”对象中包含每个用户信息（RGWUserInfo），并在“ .buckets”对象的omaps中包含存储桶的每个用户列表。如果非空，则“ <用户>”可以包含租户，例如：`prodtx$prodt test2.buckets prodtx$prodt.buckets test2` |
| users.email |                                                              |
| access_keys | 用户认证。包括用户访问密匙Id，和私有密匙key                  |
| swift_keys  | Swift用户认证。包括子用户ID：Subuser，以及子用户私有密匙Key。 |
| subusers    | 子用户。包括Name和perm_mask（子用户访问权限）。              |
| op_mask     | 用户访问权限。包括read、write、delete。                      |
| Caps        | 授权用户权限。由组成                                         |

RGW将用户信息保存在RADOS对象的数据部分，一个用户对应一个RADOS对象。由于大部分情况下，使用“pool名+对象名”来查询一个对象。

由于认证过程中需要使用用户访问密匙、子用户作为索引读取用户信息，并且在设置存储桶和对象的访问权限时，允许在存储桶和对象的访问权限授予email为xxx的用户，在操作进行鉴权检查时需要使用email作为索引获取用户信息。RGW采用了二级索引方式，即分别创建以用户访问密匙、子用户、email命名的三个RADOS对象，并将用户ID保存在对象的数据部分。当需要使用某个索引查询用户信息时，首先从索引对象读出用户ID，然后使用用户ID作为索引读取用户信息。

## 1.3 存储桶

一个存储桶对应一个RADOS对象。包含两类信息，一种是用户自定义的元数据信息，通常以键值对形式存储 。

另一类信息是对象存储策略、存储桶中索引对象的数目以及应用对象与索引对象的映射关系、存储桶的配额等，由RGWBucketInfo管理。

在创建存储桶的同时，RGW网关会同步创建一个或多个索引（index）对象，用于保存该存储桶下的对象列表，以支持查询存储桶对象列表（List Bucket）功能，因此在存储桶中有新的对象上传或者删除必须更新索引对象。

 为了避免索引对象的更新成为对象上传删除的瓶颈，RGW采用了Ceph惯用的伎俩——shard，即分片，它的确会带来性能上的提升，但这也会影响查询存储桶对象列表操作的性能。

### 存储桶的创建

流程如下：

1. 从HTTP请求解析出相关参数；
2. 判断存储桶是否存在；若存在则依次判断已存在的bucket的拥有者是否为前用户，以及已存在bucket与带创建的bucket的存储策略是否相同，若为否，则返回bucket已存在；否则转到3；
3. 创建bucket实体；
4. 更新user_id.buckets对象；
5. 返回创建成功。

> 注意：同一租户下不同用户不能创建同名的存储桶。
>
> 我们知道OMAP由一个头部和多个KV条目组成，针对user_id.buckets对象，OMAP头部保存用户使用空间统计信息`cls_user_header`; OMAP的KV条目保存一个存储桶使用的空间统计信息`cls_user_bucket_entry`。

## 1.4 对象

RGW对单个对象提供了两种上传接口：**整体上传**与**分段上传**。RGW限制了整体上传一个对象不能大于5GB（与Amazon S3）相同。用户上传的对象不能大于该限制，否则会上传失败。

我们介绍两个宏值：

- `rgw_max_chunk_size`：该宏值用来表示RGW下发到RADOS集群单个I/O的大小，同时决定对象分成多个RADOS对象时首对象的大小，以下简称分块大小。
- `rgw_obj_stripe_size`：该宏值用来指定当一个对象被分为多个RADOS对象时中间对象的大小，以下简称条带大小。
- `Class RGWObjManifest`：用来管理用户上传的对象和RADOS对象的对应关系，以下简称manifest。

用户上传一个大小小于分块的对象，那么很容易理解，该RADOS对象以应用对象名称命名，应用对象元数据也保存在该RADOS对象的扩展属性中；若用户上传的对象大于分块，那么将被分解为一个大小等于分块大小的首对象，多个大小等于条带大小的中间对象，和一个小于条带大小的尾对象。

当所有分段上传完成之后，RGW会生成一个RADOS对象，用于保存应用对象元数据和所有分段的manifest。

值得注意的是，用户上传的元数据的大小最好能被条带大小整除，否则会造成RADOS对象比每个分段条带数多且对象大小分布不均匀。从而数据管理复杂度增加。

## 1.5 数据存储位置

不同的用户数据最终以RADOS对象为单位存储到RADOS集群，RGW使用zone来管理用户数据的存储位置，zone由一组存储池（pool）组成，不同的存储池用来保存不同的数据，RGW使用RGWZoneParams来管理不同的存储池。

## 1.6 I/O流程

如图所示，OP线程从HTTP前端接收到I/O请求后，首先在REST API通用处理层，从HTTP语义中解析出S3或Swift数据并进行一系列检查，之后再根据不同API请求执行不同处理流程，如需从RADOS集群获取数据或者往RADOS集群写入数据，则通过RGW与RADOS接口适配层调用librados接口来进行交互。

[![image-20210110211049069](https://durantthorvalds.top/img/image-20210110211049069.png)](https://durantthorvalds.top/img/image-20210110211049069.png)

### 用户认证

对于S3 API，RGW支持认证用户和匿名用户访问。RGW V2支持本地认证、LDAP和keystone三种认证方式。

对于Swift，支持临URL认证、RGW本地认证、Keystone认证、匿名认证和匿名认证等五种认证引擎。

### 用户、存储桶、对象访问ACL

对于S3 API：权限如下：

- READ：查询对象
- WRITE：上传、删除对象
- READ_ACP：允许读取存储桶访问控制表
- WRITE_ACP：允许修改存储桶访问控制表
- FULL_CONTROL: 完全控制权限

对于Swift，则分为用户访问控制和存储桶访问控制：

- read-only: 读取本用户下所有内容，比如存储桶列表，对象列表等；
- read-write：授权指定用户读或写任意一个存储桶；
- admin：可以创建删除更新用户头部信息，并且授予其它用户权限。
- 

### 配额

用户和存储桶的配额分别用user_quota和bucket_quota表示。RGW实例的配额通过如下方式：

- 全局配置，适用于所有用户；
- 创建用户或更新用户配置时设置；
- 对于Swift接口，可在更新用户数据时配置或修改user_quota，在更新存储桶元数据时，配置或修改bucket_quota。

关于配额，开发者们也进行了富有工程价值的思考，为了避免I/O重复读写，RGW网关设计了一个LRU缓存，将用户和存储桶配额以一个map的形式保存在缓存中，优先从缓存中进行读写，然后再定时将缓存数据刷到对象中。

这样的话对于数据在缓存保存时间，刷新间隔都有要求。设计者提出了用三个参数进行控制：

- `rgw_bucket_quota_ttl`: 存储桶配额缓存可信任时间段。在检查配额是否达到限制时，如果缓存中记录的使用量达到配额上限（默认95%）或者距离上次从RADOS集群中更新配额到缓存中的事件超过该时间时，则需要更新缓存，并重置该参数。
- `rgw_user_quota_bucket_sync_interval`：控制存储桶已用空间从缓存刷到RADOS集群的间隔时间。
- `rgw_user_quota_sync_interval`：控制用户已用空间从缓存刷到RADOS集群的间隔时间。

## 1.7 对象上传

RGW针对对象上传设计了两个接口：**整体上传接口**和**分段上传接口**。

整体上传有三个步骤：

1. prepare：初始化manifest数据结构；
2. handle_data：RGW每次从HTTP server中取出`rgw_max_chunk_size`字节数据，存放在一个bufferlist中，然后分成一个或多个I/O异步下发送到RADOS层，每个I/O的大小等于`min(rgw_max_chunk_size,next_part_ofs - data_ofs)` ，`next_part_ofs`表示下一个RADOS对象保存的用户数据偏移位置，`data_ofs`表示当前数据的偏移位置。

以一个块大小为2MB，条带大小为5MB为例，用户上传一个9MB对象，那么应用对象先按照2MB进行分割，最后会余下1MB。其中0-2MB为对象1， 2MB-4MB和4MB-6MB，6MB-7MB共同拼接为对象2，最后的7MB-8MB以及8MB-9MB组成对象3.

1. complete：该阶段主要是将对象元数据更新到head_obj，同时将对象条目更新到索引对象中。
