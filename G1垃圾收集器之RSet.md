# G1垃圾收集器之RSet

### 堆内存

​	在 G1 的垃圾回收算法中，堆内存采用了另外一种完全不同的方式进行组织，被划分为多个（默认2000多个）大小相同的内存块（Region），每个Region是逻辑连续的一段内存，在被使用时都充当一种角色，如下图：

![img](https:////upload-images.jianshu.io/upload_images/2184951-256a9ccb6e51be85.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

​	G1堆内存的相关实现位于`g1CollectedHeap.cpp`类中.

### Region

​	每个 Region 被标记了 E、S、O 和 H，其中 H 是以往算法中没有的，它代表 Humongous，表示这些 Region 存储的是巨型对象（humongous object，H-obj），当新建对象大小超过 Region 大小一半时，直接在新的一个或多个连续 Region 中分配，并标记为 H。

​	Region的相关实现位于`heapRegion.cpp`类中，当堆内存初始化时，G1CollectorPolicy 调用`HeapRegion::setup_heap_region_size`方法根据最小堆设置每个 Region 大小。

```php
// ------- g1CollectorPolicy.cpp
// Set up the region size and associated fields. Given that the
// policy is created before the heap, we have to set this up here,
// so it's done as soon as possible.
HeapRegion::setup_heap_region_size(Arguments::min_heap_size());
```

​	Region 的大小可以通过`-XX:G1HeapRegionSize`参数指定，如果没有显示设置，则根据如下逻辑计算出一个合理的大小。

![img](https:////upload-images.jianshu.io/upload_images/2184951-336641e50b6c29fc.png?imageMogr2/auto-orient/strip|imageView2/2/w/1102/format/webp)

​	Region 的大小只能是 1M、2M、4M、8M、16M 或 32M，比如`-Xmx16g -Xms16g`，G1就会采用 16G / 2048 = 8M 的 Region.

### RSet

​	每个 Region 初始化时，会初始化一个 remembered set（已记忆集合），这个翻译有点拗口，以下简称 RSet，该集合用来记录并跟踪其它 Region 指向该 Region 中对象的引用，每个 Region 默认按照 512Kb 划分成多个 Card，所以 RSet 需要记录的东西应该是 xx Region 的 xx Card。

![img](https:////upload-images.jianshu.io/upload_images/2184951-bd04a968d1c8c895.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

​	Region1 和 Region3 中有对象引用了 Region2 的对象，则在 Region2 的 Rset 中记录了这些引用。

#### RSet 实现过程

​	为了维护这些 RSet，如果每次给引用类型的字段赋值都要更新 RSet，这带来的额外开销实在太大，G1 中采用post-write barrier 和 concurrent refinement threads 实现了 RSet 的更新。

```dart
// 假设对象 young 和 old 分别在不同的 Region 中
Object young = new Object();
old.p = young;
```

​	java 层面给 old 对象的 p 字段赋值 young 对象之后，jvm 底层会执行 oop_store 方法，实现位于`oop.inline.hpp`类中。

![img](https:////upload-images.jianshu.io/upload_images/2184951-e8370b0bbb1f6eaa.png?imageMogr2/auto-orient/strip|imageView2/2/w/1126/format/webp)

​	在赋值动作的前后，JVM 插入一个 pre-write barrier 和 post-write barrier，其中 post-write barrier 的最终动作如下：

![img](https:////upload-images.jianshu.io/upload_images/2184951-a2b508e065fc62c4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1010/format/webp)

1. 找到该字段所在的位置(Card)，并设置为 dirty_card
2. 如果当前是应用线程，每个 Java 线程有一个 dirty card queue，把该 card 插入队列
3. 除了每个线程自带的 dirty card queue，还有一个全局共享的 queue

​	赋值动作到此结束，接下来的 RSet 更新操作交由多个 ConcurrentG1RefineThread 并发完成，每当全局队列集合超过一定阈值后，ConcurrentG1RefineThread 会取出若干个队列，遍历每个队列中记录的 card，并进行处理，位于`G1RemSet::refine_card`方法，大概实现逻辑如下：

 1、根据card的地址，计算出card所在的Region

 2、如果Region不存在，或者Region是Young区，或者该Region在回收集合中，则不进行处理

 3、最终使用闭合函数`G1UpdateRSOrPushRefOopClosure::do_oop_nv()`的处理该card中的对象

![img](https:////upload-images.jianshu.io/upload_images/2184951-742ff1083c0bd4ef.png?imageMogr2/auto-orient/strip|imageView2/2/w/1154/format/webp)

​	其中 _from 是持有引用的对象所在的 Region，to 是引用对象所在的 Region，通过 add_reference 方法加入到 RSet 中，更细节的实现在`OtherRegionsTable::add_reference`方法中，有兴趣的同学可以继续研究，比如 RSet 的存储结构。

#### RSet 有什么好处？

​	进行垃圾回收时，如果 Region1 有根对象 A 引用了 Region2 的对象 B，显然对象 B 是活的，如果没有 Rset，就需要扫描整个 Region1 或者其它 Region，才能确定对象 B 是活跃的，有了 Rset 可以避免对整个堆进行扫描。

#### RSet 有什么风险？

​	通过对 RSet 实现过程的研究，我们得知应用线程只负责把更新字段所在的 Card 插入到 dirty card queue 中，然后由后台线程 refinement threads 负责 RSet 的更新操作，如果应用线程插入速度过快，refinement threads 来不及处理，那么应用线程将接管 RSet 更新的任务，这是必须要避免的。

​	refinement threads 线程数量可以通过`-XX:G1ConcRefinementThreads`或`-XX:ParallelGCThreads`参数设置。