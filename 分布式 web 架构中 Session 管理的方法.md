# 分布式 web 架构中 Session 管理的方法

​	在分布式架构中，必须保证用户在一个服务器节点进行登陆后，不仅该节点需要在内存中保存用户Session，还需要让其他应用服务器节点也能共享到用户Session。分布式架构进行Session共享的常用方案有如下几种。

### 一、方案列表

#### 1、存放在 Cookie 中

​	这种方案是基于cookie的传输来实现的，核心思想很简单，就是把完整的会话数据经过处理后写入到客户端cookie，以后客户端每次请求都带上这个cookie，然后服务端通过解析cookie数据来获取会话信息。

##### 缺点：

- 通过cookie来传递关键数据肯定是不安全的。
- 如果客户端禁用了cookie，将直接导致服务不可用。
- cookie的数据是有大小限制的，如果传递的数据超出限制大小，将会导致数据异常。
- 在http请求中携带大量的数据进行传输会增加网络负担，同样，服务端响应大量数据会导致请求变慢，并发量大的时候会非常可怕。

#### 2、Session 复制

​	部分Web服务器能够支持Session复制功能，例如Tomcat。用户可以通过修改Web服务器的配置文件，让Web服务器进行Session复制，保持每一个服务器节点的Session数据都能达到一致。

##### 缺点：

- 当Web应用中Session数量较多的时候，每个服务器节点都需要有一部分内存用来存放Session，将会占用大量内存资源。
- 大量的Session对象通过网络传输进行复制，不但占用了网络资源，还会因为复制同步出现延迟，导致程序运行错误。

#### 3、Session 粘滞

​	Session粘滞是通过负载均衡器，将同一用户的请求都分发到固定的一个服务器节点上，让固定服务器进行响应处理，如此一来，只需要这个节点上保存了用户Session，即可保持用户的状态一致性。

##### 缺点：

- 如果这台服务器宕机或重启了，那么所以的会话数据都会丢失，失去了分布式集群带来的高可用特性。
- 增加了负载均衡器的负担，使它变得有状态了，而且资源消耗会更大，容易成为性能瓶颈。
- Session粘滞方案依赖于负载均衡器，而且只能满足水平扩展的集群场景，无法满足进行应用分割后的分布式场景。

#### 4、Session 集中存储

​	这种方式的思路就是把所有的会话数据统一存储和管理，所有应用服务器需要对session进行读写都要通过session服务器来操作。

​	这种方案的好处是独立了session的管理，职责单一化，session服务器采用什么方式存储（内存、数据库、文档、NoSql等等），什么方式对外提供服务都是透明的。不会给应用系统和负载均衡带来额外的开销，不需要进行数据同步就能保证一致性，看起来应该是非常完美了，不过也有自己的一些小缺陷：

- 对session读写需要网络操作，相比较session直接存储在web服务器的时候增加了时延和不稳定性，好在session服务器和web服务器一般是部署在局域网中，可以最大化减少这个问题。
- session服务器出现问题将影响所有web服务，如果采用多机部署同时也会带来数据一致性问题。

### 二、分布式 Session 集中存储方案

1、参照SpringSession的实现，使用 SessionFilter 进行请求拦截，然后通过 Request 包装类接管 Web 服务器的 Session 管理。在 Request 包装类中，重写 getSession 方法，Session 使用方法和过去一样，对使用者透明。

2、基于 jedis 开发一个分布式缓存 SDK 模块，用于 Session 共享模块和 Redis 中间进行通信，能够增加 Session 集中管理的可扩展性，如果需要支持其他的缓存服务器，对缓存 SDK 进行扩展开发即可。

3、搭建 Redis 集群用于存放微应用 Session，以保证 Session 数据的高可用。Redis 集群示意图如下：

![img](https://upload-images.jianshu.io/upload_images/6430003-6b967add86dca19b.png?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)

​	集群中包含两个 Master 和两个 Slave，两个 Master 对 Session 数据进行分片存储，而 Slave 可用于进行数据备份和读写分离。

4、实现
**基于session集中存储的实现方案：**
新增Filter，拦截请求，包装HttpServletRequest
改写getSession方法，从session存储中获取session数据，返回自定义的HttpSession实现。
在生成新Session后，写入sessionid到cookie中。
