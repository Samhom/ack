## tomcat 集群同步原理

​	对一个有四台计算机的集群进行访问，这时假设根据负载均衡分配到了 Node1，如果在 Node1 上创建了一个 session 对象，这时，在服务器响应客户端之前，一定是要先将创建 session 对象的信息同步到其它节点上的。这样，在客户端第二次发起请求时，假设分到了 Node2，也可以直接获取 session 信息，照常进行会话。如果在某个服务器上删除了会话，那么同样，在响应之前也会同步其它节点也删除会话。

### 一、tomcat 同步组件

​	在上述无论是发送还是接收信息的过程中，使用到的组件主要有三个：Manage、Cluster、Tribes。

​	Manager 的作用是将操作的信息记录下来，然后序列化后交给 Cluster，接着 Cluster 是依赖于 Tribes 将信息发送出去的。其余节点收到信息后，按照相反的流程一步步传到 Manager，经过反序列化之后使该节点同步传递过来的操作信息。如图，假设访问的是中间的节点，该节点将信息同步出去。信息是以 Cluster Message 对象发送的：

![这里写图片描述](https://img-blog.csdn.net/20170616071807994?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5NDY1NjU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 二、同步方式

##### tomcat 共提供了两种集群同步机制：

- 集群增量会话管理器（**DeltaManager**）
- 集群备份会话管理器（**BackupManager**）

​	DeltaManager 会话管理器是 tomcat 默认的集群会话管理器，集群增量会话管理器的职责是将某节点的会话改变同步到集群内其他成员节点上，它属于全节点复制模式。

​	所谓全节点复制是指集群中某个节点的状态变化后需要同步到集群中剩余的节点，非全节点方式可能只是同步到其中某个或若干节点，如下图所示：

![这里写图片描述](https://img-blog.csdn.net/20170615221549687?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5NDY1NjU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	全节点复制模式存在的一个很大的问题就是用于备份的网络流量会随着节点数的增加而急速增加，这也就是无法构建较大规模集群的原因。为了解决这个问题，tomcat 提出了集群备份会话管理器。每个会话只有一个备份。这样就可构建大规模的集群。

### 三、原理分析

#### 1、DeltaManager 会话管理器

​	为区分不同的动作必须要先定义好各种事件，例如会话创建事件、会话访问事件、会话失效事件、获取所有会话事件、会话增量事件、会话 ID 改变事件等等，实际上 tomcat 集群会有 9 种事件，集群根据这些不同的事件就可以彼此进行通信，接收方对不同事件做不同的操作。

​	node1 节点创建完一个会话后，即向其他三个节点发送 EVT_SESSION_CREATED 事件，其他三个节点接收到此事件后则各自在自己本地创建一个会话，会话包含了两个很重要的属性——会话 ID 和创建时间，这两个属性都必须由 node1 节点跟着 EVT_SESSION_CREATED 一起发送出去，本地会话创建成功后即完成了会话创建同步工作，此时通过会话 ID 查找集群中任意一个节点都可以找到对应的会话。同样对于会话访问事件，node1 向其他节点发送 EVT_SESSION_ACCESSED 事件及会话 ID，其他节点根据会话 ID 找到对应会话并更新会话最后访问时间，以免被认为是过期会话而被清理。

​	类似的还有会话失效事件（同步集群销毁某会话）、会话 ID 改变事件（同步集群更改会话 ID）等等操作。

​	Tomcat 使用 SessionMessageImpl 类定义了各种集群通信事件及操作方法，在整个集群通信过程中就是按照此类定义好的事件进行通信，SessionMessageImpl 包含的事件如下：

1. EVT_SESSION_CREATED
2. EVT_SESSION_EXPIRED
3. EVT_SESSION_ACCESSED
4. EVT_GET_ALL_SESSIONS
5. EVT_SESSION_DELTA
6. EVT_ALL_SESSION_DATA
7. EVT_ALL_SESSION_TRANSFERCOMPLETE
8. EVT_CHANGE_SESSION_ID
9. EVT_ALL_SESSION_NOCONTEXTMANAGER

​	除此之外它继承了序列化接口、集群消息接口（集群的操作）、会话消息接口（事件定义及会话操作）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020040920513776.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NvbWh1,size_16,color_FFFFFF,t_70)

​	集群增量会话管理器 DeltaManager 通过 SessionMessageImpl 消息来管理 DeltaSession，即根据 SessionMessageImpl 里面的事件响应不同的操作。

​	Tomcat 的集群通信使用的是 tribes 组件，网络 IO 都交由 tribes 后应用可以更专注逻辑处理，DeltaManager 存在一个 messageDataReceived(ClusterMessage cmsg) 方法，此方法会在本节点接收到其他节点发送过来的消息后被调用，且传入的参数为 ClusterMessage 类型，可转化为 SessionMessage 类型，然后根据 SessionMessage 定义的 9 种事件做不同处理。

​	其中有一个事件需要关注的是 EVT_SESSION_DELTA，它是对会话增量同步处理的事件，某个节点在一个完整的请求过程中对某会话相关属性的所有操作被抽象到了 DeltaRequest 对象中，而 DeltaRequest 被序列化后会放到 SessionMessage 中，所以 EVT_SESSION_DELTA 事件处理逻辑就是从 SessionMessage 获取并反序列化出 DeltaRequest 对象，再将 DeltaRequest 包含的对某个会话的所有操作同步到本地该会话中，至此完成会话增量同步。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020040920593186.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NvbWh1,size_16,color_FFFFFF,t_70)

​	总的来说 DeltaManager 就是 DeltaSession 的管理器，它提供了会话增量的同步方式而不是全量同步，极大提高了同步效率。

#### 2、BackupManager

​	全节点复制的网络流量在 tomcat 集群比较庞大的时候随节点数量增加呈平方趋势增长，也正是因为这个因素导致无法构建较大规模的集群，为了使集群节点能更加大，首要解决的就是数据复制时流量增长的问题，于是 tomcat 提出了另外一种会话管理方式，每个会话只会有一个备份，它使会话备份的网络流量随节点数量的增加呈线性趋势增长，大大减少了网络流量和逻辑操作，可构建较大的集群。

​	集群一般是通过负载均衡对外提供整体服务，所有节点被隐藏在后端组成一个整体。最常见的负载方式是前面用 apache 拖住所有节点，它支持将类似“326257DA6DB76F8D2E38F2C4540D1DEA.tomcat1”的会话 id 进行分解，定位到 tomcat 集群中以 tomcat1 命名的节点上（这种方式称为 Session Stick，由 apache jk 模块实现）。

​	每个会话存在一个原件和一个备份，且备份与原件不会保存在同一个节点上，当客户端发起请求后通过负载均衡被分发到tomcat1实例节点上，生成一个包含.tomcat1后缀的会话标识，并且tomcat1节点根据一定策略选出此次会话对象备份的节点，然后将包含了{会话id，备份ip}的信息发送给tomcat2、tomcat3、tomcat4，这样每个节点都有一个会话id、备份ip列表，即每个节点都有每个会话的备份ip地址。

​	完成上面一步后就是将会话内容备份到备份节点上，假如 tomcat1 的 s1、s2 两个会话的备份地址为 tomcat2，则把会话对象备份到 tomcat2 中，类似的有 tomcat2 把 s3 会话备份到 tomcat4，tomcat4 把 s4、s5 两个对话备份到 tomcat3，这样集群中所有的会话都已经有了一份备份。

​	当tomcat1一直不出故障，由于Session Stick技术客户端将一直访问到tomcat1节点上，保证一直能获取到会话。而当tomcat1出故障了，这时tomcat也提供了一个failover机制，apache感知到后端集群tomcat1节点被移除了，这时它会把请求随机分配到其他任意节点上，接下去会有两种情况：

- 刚好分到了备份节点 tomcat2 上，此时仍能获取到 s1 会话，除此之外，tomcat2 还要另外做的事是将这个 s1会话标记为原件且继续选取一个备份地址备份 s1 会话，这样一来又有了备份。
- 假如分到了非备份节点 tomcat3，此时肯定找不到s1会话，于是它将向集群所有节点发问，“请问谁有s1会话的备份ip地址信息？”，因为只有 tomcat2有 s1 的备份地址信息，它接收到询问后应答告知 tomcat3 节点 s1会话的备份在 tomcat2，根据这个信息就能查到 s1 会话了，并且 tomcat3 在自己本地生成 s1 会话并标为原件，tomcat2 上的副本不变，这样一来同样能找到 s1 会话，正常完整整个请求处理。

接着分析 Tomcat 对上面机制详细的实现，正常情况下为了支持高效的并发操作，tomcat 的所有会话集使用ConcurrentHashMap：

```java
public Object put(Object key, Object value) {
  ①实例化MapEntry，将key和value传入，并设置源节点为目前节点。
  ②判断本地内存是否已包含key，如是则不仅要本地remove掉，还要跨节点remove。
  ③通过Round robin算法从MapMember中选择一个作为备份节点。
  ④实例化一个包含MSG_BACKUP标识的MapMessage对象并发送给备份节点。
  ⑤实例化一个包含MSG_PROXY标识的MapMessage对象并发送给除了备份节点外的其他（代理）节点。
  ⑥put进本地缓存。
}
```

其次，再看看它如何通过get实现获取会话对象操作：

```java
public Object get(Object key) {
 ①获取本地的MapEntry对象，它或许直接包含了会话对象，或许包含了会话对象的存放位置信息。
 ②判断本节点是否属于源节点，如为源节点则直接获取MapEntry对象里面的会话对象并返回。
 ③判断本节点是否属于备份节点，若为备份节点则直接获取MapEntry对象里面的会话对象作为返回对象，并且还要将本节点升为源节点、重新选取一个新备份节点，把MapEntry对象拷贝到新备份节点。
 ④判断本节点是否属于代理节点，若为代理节点则向其他节点发送会话对象拷贝请求，“集群中谁有此会话对象请发送给我”，把接收到的会话对象放到本节点并作为返回对象，最后将本节点升为源节点。
}
```

最后，看看删除会话对象remove操作的实现：

```java
public Object remove(Object key) {
 ①删除本地此MapEntry对象。
 ②广播其他节点删除此MapEntry对象。
}
```

​	通过上面三个方法已经很清晰描述了新的 Map 是如何进行跨节点的增删改查的，BackupManager 会话管理器就是通过这个新的 Map 进行会话管理。