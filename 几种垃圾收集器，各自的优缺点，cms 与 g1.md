## 几种垃圾收集器，各自的优缺点，cms 与 g1

### 一、CMS(Concurrent Mark and Sweep 并发-标记-清除)：

​	是一款基于并发、使用标记清除算法的垃圾回收算法，只针对老年代进行垃圾回收，而 Full GC 指的是整个堆的GC 事件，包括新生代、老年代、元空间等，两者有所区分。CMS 收集器工作时，尽可能让 GC 线程和用户线程并发执行，以达到降低 STW 时间的目的。

启用 CMS 垃圾收集器命令行参数：`-XX:+UseConcMarkSweepGC`。

#### 1、7个阶段（1、5：STW；2、3、4、6、7：GC线程与应用线程并发执行）：

1、初始标记—2、并发标记—3、并发预清理—4、并发可取消预清理—5、最终标记—6、并发清除—7、并发重置。

- ##### 阶段 1: 初始标记(Initial Mark)

  此阶段的目标是标记老年代中所有存活的对象, 包括 GC Root 的直接引用, 以及由新生代中存活对象所引用的对象，触发第一次 STW 事件，这个过程是支持多线程的（JDK7 之前单线程，JDK8 之后并行，可通过参数 **CMSParallelInitialMarkEnabled** 调整）

- ##### 阶段 2: 并发标记(Concurrent Mark)

  此阶段 GC 线程和应用线程并发执行，遍历阶段 1 初始标记出来的存活对象，然后继续递归标记这些对象可达的对象。

- ##### 阶段 3: 并发预清理(Concurrent Preclean)

  此阶段 GC 线程和应用线程也是并发执行，因为阶段 2 是与应用线程并发执行，可能有些引用关系已经发生改变。通过卡片标记 (Card Marking)，提前把老年代空间逻辑划分为相等大小的区域(Card)，如果引用关系发生改变，JVM 会将发生改变的区域标记位“脏区”(Dirty Card)，然后在本阶段，这些脏区会被找出来，刷新引用关系，清除“脏区”标记。

- ##### 阶段 4: 并发可取消的预清理(Concurrent Abortable Preclean)

  此阶段也不停止应用线程. 本阶段尝试在 STW 的最终标记阶段(Final Remark)之前尽可能地多做一些工作，以减少应用暂停时间。在该阶段不断循环处理：标记老年代的可达对象、扫描处理 Dirty Card 区域中的对象。循环的终止条件有：
  a.达到循环次数
  b.达到循环执行时间阈值
  c.新生代内存使用率达到阈值

- ##### 阶段 5: 最终标记(Final Remark)

  这是 GC 事件中第二次(也是最后一次)STW阶段，目标是完成老年代中所有存活对象的标记。在此阶段执行： 
  a.遍历新生代对象，重新标记		
  b.根据 GC Roots，重新标记
  c.遍历老年代的 Dirty Card，重新标记

- ##### 阶段 6: 并发清除(Concurrent Sweep)

  此阶段与应用程序并发执行，不需要 STW 停顿，根据标记结果清除垃圾对象。

- ##### 阶段 7: 并发重置(Concurrent Reset)

  此阶段与应用程序并发执行，重置 CMS 算法相关的内部数据, 为下一次 GC 循环做准备。

#### 2、CMS 常见问题

##### 最终标记阶段停顿时间过长问题：

​	CMS 的 GC 停顿时间约 80% 都在最终标记阶段(Final Remark)，若该阶段停顿时间过长，常见原因是新生代对老年代的无效引用，在上一阶段的并发可取消预清理阶段中，执行阈值时间内未完成循环，来不及触发 Young GC，清理这些无效引用。

​	通过添加参数：-XX:+CMSScavengeBeforeRemark。在执行最终操作之前先触发 Young GC，从而减少新生代对老年代的无效引用，降低最终标记阶段的停顿，但如果在上个阶段(并发可取消的预清理)已触发 Young GC，也会重复触发Young GC。

##### 并发模式失败(concurrent mode failure)：

​	当 CMS 在执行回收时，新生代发生垃圾回收，同时老年代又没有足够的空间容纳晋升的对象时，CMS 垃圾回收就会退化成单线程的 Full GC。所有的应用线程都会被暂停，老年代中所有的无效对象都被回收。

##### 晋升失败(promotion failed)：

​	当新生代发生垃圾回收，老年代有足够的空间可以容纳晋升的对象，但是由于空闲空间的碎片化，导致晋升失败，此时会触发单线程且带压缩动作的 Full GC。

##### 并发模式失败和晋升失败都会导致长时间的停顿，常见解决思路如下：

- 降低触发 CMS GC 的阈值，即参数 -XX:CMSInitiatingOccupancyFraction 的值，让 CMS GC 尽早执行，以保证有足够的空间。
- 增加 CMS 线程数，即参数 -XX:ConcGCThreads。
- 增大老年代空间。
- 让对象尽量在新生代回收，避免进入老年代。

##### CMS 如何解决内存碎片：

​	内存碎片问题是指只负责将标记的需要回收的内存给回收，对于一块内存，会留下很多小的空的位置，但是这些位置不足以去申请一个连续的对象。

​	对于内存碎片的解决，我们首先要明确，产生内存碎片的主要是老年代。

​	CMS 提供了 -XX：+UseCMSCompactAtFullCollection 用于在 CMS 收集器顶不住要进行 FullGC 时开启内存碎片的合并整理过程。虽然内存碎片没有了，但是停顿时间会变长很多很多，虚拟机设计者提供了 -XX：CMSFullGCsBeforeCompaction，在上一次 CMS 并发 GC 执行过后，到底还要再执行多少次 full GC 才会做压缩。默认是 0，也就是在默认配置下每次 CMS GC 顶不住了而要转入 full GC 的时候都会做压缩。

### 二、G1(Garbage-First）

​	G1 是一款面向服务器的垃圾收集器，支持新生代和老年代空间的垃圾收集，主要针对配备多核处理器及大容量内存的机器，G1 最主要的设计目标是: **实现可预期及可配置的 STW 停顿时间**。

#### 1、G1堆空间划分：

##### Region：

​		为实现大内存空间的低停顿时间的回收，将划分为多个大小相等的 Region。每个小堆区都可能是 Eden 区，Survivor 区或者 Old 区，但是在同一时刻只能属于某个代。

​		在逻辑上, 所有的 Eden 区和 Survivor 区合起来就是新生代，所有的 Old 区合起来就是老年代，且新生代和老年代各自的内存 Region 区域由 G1 自动控制，不断变动。

##### 巨型对象：

​		当对象大小超过 Region 的一半，则认为是巨型对象(Humongous Object)，直接被分配到老年代的巨型对象区(Humongous regions)，这些巨型区域是一个连续的区域集，每一个 Region 中最多有一个巨型对象，巨型对象可以占多个 Region。

#### 2、G1 把堆内存划分成一个个 Region 的意义在于

​	每次 GC 不必都去处理整个堆空间，而是每次只处理一部分 Region，实现大容量内存的 GC。

​	通过计算每个 Region 的回收价值，包括回收所需时间、可回收空间，在有限时间内尽可能回收更多的垃圾对象，把垃圾回收造成的停顿时间控制在预期配置的时间范围内，这也是 G1 名称的由来: garbage-first。

#### 3、G1 工作模式：

​	针对新生代和老年代，G1 提供 2 种 GC 模式，Young GC 和 Mixed GC，两种会导致 Stop The World。

##### Young GC：

​	当新生代的空间不足时，G1 触发 Young GC 回收新生代空间。Young GC 主要是对 Eden 区进行GC，它在 Eden 空间耗尽时触发，基于分代回收思想和复制算法，**每次 Young GC 都会选定所有新生代的 Region**，同时计算下次 Young GC 所需的 Eden 区和 Survivor 区的空间，动态调整新生代所占 Region 个数来控制 Young GC 开销。

##### Mixed GC：

​	当老年代空间达到阈值会触发 Mixed GC，选定所有新生代里的 Region，根据全局并发标记阶段(下面介绍到)统计得出收集收益高的若干老年代 Region。在用户指定的开销目标范围内，尽可能选择收益高的老年代 Region 进行 GC，通过选择哪些老年代 Region 和选择多少 Region 来控制 Mixed GC 开销。

#### 4、全局并发标记：

主要是为 Mixed GC 计算找出回收收益较高的 Region 区域，具体分为5个阶段：

##### 阶段 1: 初始标记(Initial Mark)——STW

​	并发地进行标记从 GC Root 开始直接可达的对象（原生栈对象、全局对象、JNI 对象），当达到触发条件时，G1 并不会立即发起并发标记周期，而是等待下一次新生代收集，利用新生代收集的 STW 时间段，完成初始标记，这种方式称为借道（Piggybacking）。

##### 阶段 2: 根区域扫描（Root Region Scan）

​	在初始标记暂停结束后，新生代收集也完成的对象复制到 Survivor 的工作，应用线程开始活跃起来；此时为了保证标记算法的正确性，所有新复制到 Survivor 分区的对象，需要找出哪些对象存在对老年代对象的引用，把这些对象标记成根(Root)，这个过程称为根分区扫描（Root Region Scanning），同时扫描的 Suvivor 分区也被称为根分区（Root Region）。

​	根分区扫描必须在下一次新生代垃圾收集启动前完成（接下来并发标记的过程中，可能会被若干次新生代垃圾收集打断），因为每次 GC 会产生新的存活对象集合。

##### 阶段 3: 并发标记（Concurrent Marking）

​	标记线程与应用程序线程并行执行，标记各个堆中 Region 的存活对象信息，这个步骤可能被新的 Young GC 打断所有的标记任务必须在堆满前就完成扫描，如果并发标记耗时很长，那么有可能在并发标记过程中，又经历了几次新生代收集。

##### 阶段 4: 再次标记(Remark) ——STW

​	以完成标记过程短暂地停止应用线程, 标记在并发标记阶段发生变化的对象，和所有未被标记的存活对象，同时完成存活数据计算。

##### 阶段 5: 清理(Cleanup) 

为即将到来的转移阶段做准备, 此阶段也为下一次标记执行所有必需的整理计算工作：

- 整理更新每个 Region 各自的 RSet
  remember set，HashMap 结构，记录有哪些老年代对象指向本 Region，key 为指向本 Region 的 对象的引用，value 为指向本 Region 的具体 Card 区域，通过 RSet 可以确定 Region 中对象存活信息，避免全堆扫描
- 回收不包含存活对象的 Region
- 统计计算回收收益高（基于释放空间和暂停目标）的老年代分区集合

#### G1调优注意点：

##### a.Full GC问题：

​	G1 的正常处理流程中没有 Full GC，只有在垃圾回收处理不过来(或者主动触发)时才会出现， **G1的Full GC就是单线程执行的 Serial old gc**，会导致非常长的 STW，是调优的重点，需要尽量避免 Full GC。

常见原因如下：

- 程序主动执行 System.gc()
- 全局并发标记期间老年代空间被填满（并发模式失败）
- Mixed GC 期间老年代空间被填满（晋升失败）
- Young GC 时 Survivor 空间和老年代没有足够空间容纳存活对象

类似CMS，常见的解决方案是：

- 增大-XX:ConcGCThreads=n 选项增加并发标记线程的数量，或者 STW 期间并行线程的数量：-XX:ParallelGCThreads=n
- 减小 -XX:InitiatingHeapOccupancyPercent 提前启动标记周期
- 增大预留内存 -XX:G1ReservePercent=n ，默认值是10，代表使用 10% 的堆内存为预留内存，当 Survivor 区域没有足够空间容纳新晋升对象时会尝试使用预留内存

##### b.巨型对象分配：

​	巨型对象区中的每个 Region 中包含一个巨型对象，剩余空间不再利用，导致空间碎片化，当 G1 没有合适空间分配巨型对象时，G1 会启动串行 Full GC 来释放空间。可以通过增加 -XX:G1HeapRegionSize 来增大 Region 大小，这样一来，相当一部分的巨型对象就不再是巨型对象了，而是采用普通的分配方式。

##### c.不要设置 Young 区的大小：

​	原因是为了尽量满足目标停顿时间，逻辑上的 Young 区会进行动态调整。如果设置了大小，则会覆盖掉并且会禁用掉对停顿时间的控制。

##### d.平均响应时间设置：

​	使用应用的平均响应时间作为参考来设置 MaxGCPauseMillis，JVM 会尽量去满足该条件，可能是 90% 的请求或者更多的响应时间在这之内， 但是并不代表是所有的请求都能满足，平均响应时间设置过小会导致频繁 GC。

### 三、调优方法与思路：

​	GC 优化的核心思路在于尽可能让对象在新生代中分配和回收，尽量避免过多对象进入老年代，导致对老年代频繁进行垃圾回收，同时给系统足够的内存减少新生代垃圾回收次数，进行系统分析和优化也是围绕着这个思路展开。也就是说将转移到老年代的对象数量降低到最小；减少 full GC 的执行时间；

##### 需要做的事情有：

- 减少使用全局变量和大对象；
- 调整新生代的大小到最合适；
- 设置老年代的大小为最合适；
- 选择合适的 GC 收集器；

##### 分析系统的运行状况：

- 系统每秒请求数、每个请求创建多少对象，占用多少内存；
- Young GC 触发频率、对象进入老年代的速率；
- 老年代占用内存、Full GC 触发频率、Full GC 触发的原因、长时间 Full GC 的原因；

##### 进行监控和调优的一般步骤为：

1，监控 GC 的状态
	使用各种 JVM 工具，查看当前日志，分析当前JVM参数设置，并且分析当前堆内存快照和 gc 日志，根据实际的各区域内存划分和 GC 执行时间，觉得是否进行优化；

2，分析结果，判断是否需要优化
如果各项参数设置合理，系统没有超时日志出现，GC 频率不高，GC 耗时不高，那么没有必要进行 GC 优化；如果 GC 时间超过 1-3 秒，或者频繁 GC，则必须优化；
如果满足下面的指标，则一般不需要进行 GC：Minor GC 执行时间不到 50 ms；Minor GC 执行不频繁，约10 秒一次；Full GC 执行时间不到1s；Full GC 执行频率不算频繁，不低于 10 分钟 1 次；

3、调整 GC 类型和内存分配
如果内存分配过大或过小，或者采用的 GC 收集器比较慢，则应该优先调整这些参数，并且先找 1 台或几台机器进行对比，然后比较优化过的机器和没有优化的机器的性能对比，并有针对性的做出最后选择；

4、不断的分析和调整并找到最合适的参数—>全面应用参数

### 四、常用工具

##### 1、jstat：jvm 自带命令行工具，可用于统计内存分配速率、GC次数，GC耗时，常用命令格式：

```shell
> jstat -gc <pid> <统计间隔时间>  <统计次数>  # 例如： jstat -gc 32683 1000 10 ，统计 pid= 32683 的进程，每秒统计1次，统计10次
如：
s0 区容量/s1 区容量/s0 区已用内存/s1 区已用内存  之后是Eden、老年代、元空间的容量及其已用内存个  再之后是YGC总次数、YGC总耗时、FGC总次数、FGC总耗时、GC总耗时
S0C S1C S0U S1U EC EU OC OU MC MU YGC YGCT FGC FGCT GCT
```

##### 2、jmap：jvm 自带命令行工具，可用于了解系统运行时的对象分布，常用命令格式：

```shell
# 命令行输出类名、类数量数量，类占用内存大小，
# 按照类占用内存大小降序排列
> jmap -histo <pid>
# 生成堆内存转储快照，在当前目录下导出 dump.hrpof 的二进制文件，
# 可以用 eclipse 的 MAT 图形化工具分析
> jmap -dump:live,format=b,file=dump.hprof <pid>
```

##### 3、jinfo 命令格式：

```shell
> jinfo <pid>
# 用来查看正在运行的 Java 应用程序的扩展参数，包括 Java System 属性和 JVM 命令行参数
```

##### 4、其他GC工具：

- 监控告警系统：Zabbix、Prometheus、Open-Falcon。
- jdk 自动实时内存监控工具：VisualVM。
- 堆外内存监控： Java VisualVM 安装 Buffer Pools 插件、google perf 工具、Java NMT(Native Memory Tracking)工具。
- GC日志分析：GCViewer、gceasy。
- GC参数检查和优化：xxfox.perfma.com/。

### 五、JVM参数解析及调优

比如以下参数示例：

```shell
-Xmx4g –Xms4g –Xmn1200m –Xss512k -XX:NewRatio=4 -XX:SurvivorRatio=8 -XX:PermSize=100m -XX:MaxPermSize=256m -XX:MaxTenuringThreshold=15
# -XX:PermSize=100m -XX:MaxPermSize=256m 在Java 8 中已失效，会有错误警告。
```

#### 参数解析：

- -Xmx4g：可调优，默认(MaxHeapFreeRatio参数可以调整)空余堆内存大于70%时，JVM会减少堆直到-Xms的最小限制。
  	堆内存最大值为4GB。
- -Xms4g：可调优，初始化堆内存大小，默认为物理内存的1/64(小于1GB)。
  	初始化堆内存大小为4GB。
- -Xmn1200m：
  	设置年轻代大小为1200MB。增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。
- -Xss512k：
  	设置每个线程的堆栈大小。JDK5.0 以后每个线程堆栈大小为1MB，以前每个线程堆栈大小为256K。应根据应用线程所需内存大小进行调整。
  	在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在 3000~5000 左右。
- -XX:NewRatio=4：
  	设置年轻代（包括 Eden 和两个 Survivor 区）与年老代的比值（除去持久代）。设置为4，则年轻代与年老代所占比值为1：4，年轻代占整个堆栈的1/5
- -XX:SurvivorRatio=8：
  	设置年轻代中 Eden 区与 Survivor 区的大小比值。设置为8，则两个 Survivor 区与一个 Eden 区的比值为2:8，一个 Survivor 区占整个年轻代的1/10
- -XX:MaxTenuringThreshold=15：
  	设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过 Survivor 区，直接进入年老代。对于年老代比较多的应用，可以提高效率。
  	如果将此值设置为一个较大值，则年轻代对象会在 Survivor 区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概论。
- -XX:MaxDirectMemorySize=1G：可调优
  	直接内存。报java.lang.OutOfMemoryError: Direct buffer memory 异常可以上调这个值。
- -XX:+DisableExplicitGC：
  	禁止运行期显式地调用System.gc()来触发fulll GC。
- -XX:CMSInitiatingOccupancyFraction=60：
  	老年代内存回收阈值，默认值为68。
- -XX:ConcGCThreads=4：
  	CMS垃圾回收器并行线程线，推荐值为CPU核心数。
- -XX:ParallelGCThreads=8：
  	新生代并行收集器的线程数。

### 六、参数

#### 堆参数：

-Xms、-Xmx、-Xmn、-XX:SurvivorRatio、-XX:NewRatio

#### 回收器参数：

- -XX:+UseSerialGC 串行、Y O 都串行化，使用复制算法回收，逻辑简单搞笑，无线程切换开销
- -XX:+UseParallelGC 并行，Y 使用 Parallel Scavenge 回收算法，会产生多个并行回收线程，通过-XX:ParallelGCThreads=n 参数指定线程数，默认是CPU 核数；O 依然单线程
- -XX:+UseParalleOldGC 并行，Y O 都是用多线程收集
- -XX:+UseConcMarpSweepGC CMS短暂并发的收集。Y 可使用普通的或 Parallel 垃圾收集算法，由参数-XX:+UseParNewGC 来控制；O 只能使用CMS
- -XX:+UseG1GC 并行、并发、增量式压缩短暂停顿的垃圾收集器。不区分 Y O；优先收集存活对象少的 Region，因此叫 Garbage First

#### 常用组合：

- -XX:+UseSerialGC： Y Serial O Serial
- -XX:+UseParallelGC 和 -XX:+UseParalleOldGC：Y Parallel Scavenge O Parallel Old/Serial
- -XX:+UseParNewGC 和 -XX:+UseConcMarpSweepGC：Y Seril/Parallel O CMS
- -XX:+UseG1GC：G1

### 七、GC优化案例

#### 数据分析平台系统频繁 Full GC：

平台主要对用户在APP中行为进行定时分析统计，并支持报表导出，使用CMS GC算法。数据分析师在使用中发现系统页面打开经常卡顿，通过 jstat 命令发现系统每次 Young GC 后大约有10%的存活对象进入老年代。
原来是因为 Survivor 区空间设置过小，每次 Young GC 后存活对象在 Survivor 区域放不下，提前进入老年代，通过调大 Survivor区，使得 Survivor 区可以容纳 Young GC 后存活对象，对象在 Survivor 区经历多次
Young GC 达到年龄阈值才进入老年代，调整之后每次 Young GC 后进入老年代的存活对象稳定运行时仅几百 Kb，Full GC 频率大大降低。

#### 业务对接网关OOM：

网关主要消费 Kafka 数据，进行数据处理计算然后转发到另外的 Kafka 队列，系统运行几个小时候出现OOM，重启系统几个小时之后又 OOM，通过 jmap 导出堆内存，
在 eclipse MAT 工具分析才找出原因：代码中将某个业务 Kafka 的 topic 数据进行日志异步打印，该业务数据量较大，大量对象堆积在内存中等待被打印，导致OOM

#### 账号权限管理系统频繁长时间 Full GC：

系统对外提供各种账号鉴权服务，使用时发现系统经常服务不可用，通过 Zabbix 的监控平台监控发现系统频繁发生长时间 Full GC，且触发时老年代的堆内存通常并没有占满，发现原来是业务代码中调用了 System.gc()。

### 新生代和老生代的内存回收策略

##### 新生代垃圾回收：

能与 CMS 搭配使用的新生代垃圾收集器有 Serial 收集器和 ParNew 收集器。这2个收集器都采用标记复制算法，都会触发 STW 事件，停止所有的应用线程。不同之处在于，Serial 是单线程执行，ParNew 是多线程执行。

##### 老年代垃圾回收策略：
见上述分析 G1 和 CMS。
