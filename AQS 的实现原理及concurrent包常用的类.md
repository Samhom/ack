## AQS 的实现原理&concurrent包常用类举例

### 一、AQS 的实现原理

​	AbstractQueuedSynchronizer（其中的部分核心方法可结合下边的 CountDownLatch 进行分析）。AQS 是一个 FIFO 的双向队列，其内部通过节点 head 和 tail 记录队首和队尾元素，队列元素的类型为 Node。其中 Node 中的 thread 变量用来存放进入 AQS 队列里面的线程

##### 节点状态

- Node 节点内部的 SHARED 用来标记该线程是获取共享资源时被阻塞挂起后放入 AQS 队列的。
- EXCLUSIVE 用来标记线程是获取独占资源时被挂起后放入AQS 队列的 。

##### Node 中 waitStatus 字段记录当前线程等待状态

- CANCELLED （线程被取消了）
- SIGNAL（ 线程需要被唤醒）
- CONDITION（线程在条件队列里面等待）
- PROPAGATE（释放共享资源时需要通知其他节点）

Node.prev 记录当前节点的前驱节点， Node.next 记录当前节点的后继节点。

##### 独占模式和共享模式

​	AQS提供了两种工作模式：独占(exclusive)模式和共享(shared)模式。它的所有子类中，要么实现并使用了它独占功能的 API，要么使用了共享功能的API，而不会同时使用两套 API，即便是它最有名的子类 
ReentrantReadWriteLock，也是通过两个内部类：读锁和写锁，分别实现的两套 API 来实现的。独占模式即当锁被某个线程成功获取时，其他线程无法获取到该锁，共享模式即当锁被某个线程成功获取时，其他线程仍然可能获取到该锁。

##### AQS 内部类 ConditionObject

​	用来结合锁实现线程同步。ConditionObject 可以直接访问 AQS 对象内部的变量，比如 state 状态值和 AQS 队列。 ConditionObject 是条件变量，每个条件变量对应一个条件队列（单向链表队列），其用来存放调用条件变量的 await 方法后被阻塞的线程，这个条件队列的头、尾元素分别为为 firstWaiter 和 lastWaiter。

​	对于 AQS 来说，线程同步的关键是对状态值 state 进行操作。根据 state 是否属于一个线程，操作 state 的方式分为独占方式和共享方式：

```java
// 在独占方式下获取和释放资源的方法为
void acquire(int arg);         
void acquirelnterruptibly(int arg);         
boolean release(int arg)。
// 在共享方式下获取和释放资源的方法为
void acquireShared(int arg);   
void acquireSharedInterruptibly(int arg);   
boolean reaseShared(int arg)。
// tryAcquire 等以 try 开头的方法，会直接返回不会阻塞
// 不带 Intenuptibly 关键字的方法的意思是不对中断进行响应，也就是线程在调用不带 Interruptibly 关键字的方法获取资源时或者获取资源失败被挂起时，其他线程中断了该线程，那么该线程不会因为被中断而抛出异常，它还是继续获取资源或者被挂起，也就是说不对中断进行响应，忽略中断
```

##### AQS 提供的队列，主要看入队操作

​	当一个线程获取锁失败后该线程会被转换为 Node 节点，然后就会使用 enq(final Node node）方法将该节点插入到 AQS 的阻塞队列：第一次循环 head 与 tail 指向一个哨兵节点。第二次循环将当前节点插入队尾。
​	notify 和 wait，是配合 synchronized 内置锁实现线程间同步的基础设施一样，条件变量的 signal 和 await 方法也是用来配合锁（使用 AQS 实现的锁〉实现线程间同步的基础设施。AQS 的一个锁可以对应多个条件变量。

比如基于 AQS 实现的 ReentrantLock 的例子：

```java
ReentrantLock lock = new ReentrantLock(); // (l)
Condition condition = lock.newCondition(); // (2)
lock.lock(); // (3) 阻塞，当多个线程同时调用 lock.lock（）方法获取锁时，只有一个线程获取到了锁，其他线程会被转换为 Node 节点插入到 lock 锁对应的 AQS 阻塞队列里面，并做自旋 CAS 尝试获取锁
try {
    System.out.println("begin wait");
    condition.await(); // (4) 放入条件队列末尾，并阻塞在这里。这时候因为调用 lock.lock（）方法被阻塞到 AQS 队列里面的一个线程会获取到被释放的锁
    System.out.println("end wait");
} catch (Exception e) {
    e.printStackTrace();
} finally {
    lock.unlock(); // (5)
}
lock.lock(); // (6)
try {
    System.out.println("begin signal");
    condition.signal(); // (7) 当另外一个线程调用条件变量的 signal或 signalAll 方法时（必须先调用锁的 lock()方法获取锁），在内部会把条件队列里面队头的一个或全部线程节点从条件队列里面移除并放入 AQS 的阻塞队列里面，然后激活这个线程
    System.out.println("ed signal");
} catch (Exception e) {
    e.printStackTrace();
} finally {
    lock.unlock();  // (8)
}
```

#### LockSupport 工具

主要作用是挂起和唤醒线程，该工具类是创建锁和其他同步类的基础。AQS 底层就是通过 LockSupport 实现的。

LockSupport 是使用 Unsafe 类实现的，下面介绍 LockSupport 中的几个主要函数：

##### 1、void park() 方法

​	如果调用 park 方法的线程已经拿到了与 LockSupport 关联的许可证，则调用 Locksupport.park（）时会马上返回，否则调用线程会被禁止参与线程的调度，也就是会被阻塞挂起。

```java
public static void main(String[] args) throws InterruptedException {
    System.out.println("child begin park");
    LockSupport.park(); // 会永远阻塞在这里
    System.out.println("child  end  park");
}
```

​	其他线程调用 unpark(Thread thread）方法并且将当前线程作为参数时，调用 park 方法而被阻塞的线程会返回。另外，如果其他线程调用了阻塞线程的 interrupt()方法，设置了中断标志或者线程被虚假唤醒，则阻塞线程也会返回。所以在调用 park 方法时最好也使用循环条件判断方式。需要注意的是，因调用 park() 方法而被阻塞的线程被其他线程中断而返回时并不会抛出 InterruptedException 异常。

##### 2、void unpark(Thread thread）方法

​	当一个线程调用 unpark 时，如果参数 thread 线程没有持有 thread 与 LockSupport 类关联的许可证，则让 thread 线程持有。 如果 thread 之前因调用 park（）而被挂起，则调用unpark 后，该线程会被唤醒。 如果 thread 之前没有调用 park，则调用 unpark 方法后，再调用 park 方法，其会立刻返回。

```java
public static void main(String[] args) throws InterruptedException {
    System.out.println("child begin park");
    LockSupport.unpark(Thread.currentThread());
    LockSupport.park(); // 该处会直接返回
    System.out.println("child  end  park");
}
```

示例代码：

```java
// 这个例子看起来没什么问题，“但是”来了，park 方法返回时不会告诉你因何种原因返回，所以调用者需要根据之前调用 park 方法的原因，再次检查条件是否满足，如果不满足则还需要再次调用 park 方法。
Thread thread = new Thread(() -> {
    System.out.println("child begin park");
    LockSupport.park();
    System.out.println("child  end  park");
});
thread.start();
Thread.sleep(5000);
System.out.println("main begin unPark");
LockSupport.unpark(thread);
```

所以可以有下边的优化：

```JAVA
Thread thread = new Thread(() -> {
    System.out.println("child begin park");
    while (!Thread.currentThread().isInterrupted()) {// 根据调用前后中断状态的对比就可以判断是不是因为被中断才返回的
        LockSupport.park();
    }
    System.out.println("child  end  park");
});
thread.start();
Thread.sleep(5000);
System.out.println("main begin unPark");
LockSupport.unpark(thread); // 只有中断子线程，子线程才会运行结束，如果子线程不被中断，即使调用 unpark(thread）方法子线程也不会结束
System.out.println("child unPark preparing...");
Thread.sleep(5000);
System.out.println("child unPark begin");
thread.interrupt(); // 其他线程调用了阻塞线程的 interrupt()方法，设置了中断标志或者线程被虚假唤醒，则阻塞线程也会返回。
```

##### 3、void parkNanos(long nanos）方法

如果没有拿到许可证，则调用线程会被挂起 nanos 时间后修改为自动返回。

##### 4、void park(Object blocker）方法

```java
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);
    setBlocker(t, null);
}
private static void setBlocker(Thread t, Object arg) {
    UNSAFE.putObject(t, parkBlockerOffset, arg);
}
```

​	当线程在没有持有许可证的情况下调用 park 方法而被阻塞挂起时，这个 blocker 对象会被记录到该线程内部。
使用诊断工具可以观察线程被阻塞的原因，诊断工具是通过调用 getBlocker(Thread thread) 方法来获取 blocker 对象的，所以 JDK 推荐我们使用带有 blocker 参数的 park 方法，并且blocker 被设置为 this，这样当在打印线程堆横排查问题时就能知道是哪个类被阻塞了。使用带 blocker 参数的 park 方法，线程堆栈可以提供更多有关阻塞对象的信息。Thread 类里面有个变量 volatile Object parkBlocker，用来存放 park 方法传递的 blocker 对象，也就是把 blocker 变量存放到了调用 park 方法的线程的成员变量里面。

### 二、concurrent 包中常用类

#### 1、ThreadLocalRandom

​	每个 Random 实例里面都有一个原子性的种子变量用来记录当前的种子值，当要生成新的随机数时需要根据当前种子计算新的种子并更新回原子变量。

在多线程下使用单个 Random 实例生成随机数时，当多个线程同时计算随机数来计算新的种子时，多个线程会竞争同一个原子变量的更新操作，由于原子变量的更新是 CAS 操作，同时只有一个线程会成功，所以会造成大量线程进行自旋重试，这会降低并发性能，所以 ThreadLocalRandom 应运而生。

​	ThreadLocalRandom 类继承了 Random 类并重写了 nextlnt 方法，在 ThreadLocalRandom 类中并没有使用继承自 Random 类的原子性种子变量。在 ThreadLocalRandom 中并没有存放具体的种子，具体的种子存放在具体的调用线程的 threadLocalRandomSeed 变量里面。ThreadLocalRandom 类似于 ThreadLocal 类 ，就是个工具类。 当线程调用 ThreadLocalRandom 的 current 方法时， ThreadLocalRandom 负责初始化调用线程的 threadLocalRandomSeed 变量，也就是初始化种子。当调用 ThreadLocalRandom 的 nextInt 方法时，实际上是获取当前线程的 threadLocalRandomSeed 变量作为当前种子来计算新的种子，然后更新新的种子到当前线程的 threadLocalRandomSeed 变量，而后再根据新种子并使用具体算法计算随机数。这里需要注意的是，threadLocalRandomSeed 变量就是 Thread 类里面的一个普通 long 变量，它并不是原子性变量。 其实道理很简单，因为这个变量是线程级别的，所以根本不需要使用原子性变量。

#### 2、LongAdder

​	使用 AtomicLong 时，在高并发下大量线程会同时去竞争更新同一个原子变量，但是由于同时只有一个线程的 CAS 操作会成功，这就造成了大量线程竞争失败后，会通过无限循环不断进行自旋尝试 CAS 的操作，而这会白白浪费 CPU 资源。

​	JDK 8 新增了一个原子性递增或者递减类 LongAdder 用来克服在高并发下使用 AtomicLong 的缺点。AtomicLong 的性能瓶颈是由于过多线程同时去竞争一个变量的更新而产生的，那么如果把一个变量分解为多个变量，让同样多的线程去竞争多个资源，就是 LongAddr 的解决思路。

#### 3、CopyOnWriteArrayList（无界）

​	CopyOnWrite 容器即**写时复制的容器**。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行 Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，
再将原容器的引用指向新的容器。

​	这样做的好处是我们可以对 CopyOnWrite 容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以 CopyOnWrite 容器也是一种读写分离的思想，读和写不同的容器。

​	public boolean add(E e) {...} 由于加了锁（RetreenLock 独占锁），所以整个 add 过程是个原子性操作。需要注意的是，在添加元素时，首先复制了一个快照，然后在快照上进行添加，而不是直接在原来数组上进行。

##### 写时复制策略会产生的弱一致性问题 

​	在通过 get(int index) 获取指定下标下的元素时，该方法并没有加锁，也就是获取队列数组内存指向空间和从队列指定下标处取出指定元素两个步骤之间并不是原子性操作。如果这两个步骤之间别的线程通过 remove(先获取独占锁再拷贝一份原有数据，并在拷贝的数据的内存地址上进行删除操作)进行删除当前线程要获取的数据，并将队列元素指向地址重新指向了这儿新的内存地址。但是当前线程访问的还是旧的内存指向空间，还是能继续根据数组下标访问到要访问的元素。遍历列表元素可以使用法代器，也存在上述的弱一致性问题。

​	CopyOnWriteArrayList 使用写时复制的策略来保证 list 的一致性，而获取一修改一写入三步操作并不是原子性的，所以在增删改的过程中都使用了独占锁，来保证在某个时间只有一个线程能对 list 数组进行修改。 

​	另外 CopyOnWriteArrayList 提供了弱一致性的法代器，从而保证在获取迭代器后，其他线程对 list 的修改是不可见的，迭代器遍历的数组是一个快照。 另外，CopyOnWriteArraySet 的底层就是使用它实现的。

​	另外一个问题就是内存占用问题。

#### 4、ReentrantLock

​	ReentrantLock 的底层是使用 AQS 实现的可重入独占锁。 在这里 AQS 状态值为 0 表示当前锁空闲，为大于等于 1 的值则说明该锁己经被占用。 该锁内部有公平与非公平实现，默认情况下是非公平的实现。 

另外，由于该锁是独占锁，所以某时只有一个线程可以获取该锁。

##### 所谓的公平与非公平是指

​		非公平是说先尝试获取锁的线程并不一定比后尝试获取锁的线程优先获取锁。这里假设线程 A 调用 lock() 方法时执行到 nonfairTryAcquire 发现当前状态值不为 0，然后发现当前线程不是线程持有者，则返回 false，然后当前线程被放入 AQS 阻塞队列。这时候线程 B 也调用了 lock() 方法执行到 nonfairTryAcquire，发现当前状态值为 0 了 （ 假设占有该锁的其他线程释放了该锁），所以通过 CAS 设置获取到了该锁。明明是线程 A 先请求获取该锁呀，这就是非公平的体现。这里线程 B 在获取锁前并没有查看当前 AQS 队列里面是否有比自己更早请求该锁的线程 ， 而是使用了抢夺策略。

​		公平的实现是核心代码是 hasQueuedPredecessors 方法，也就是设置队列的状态 state 值时，会先调用这个方法，即先判断当前节点是否有前驱节点。

#### 5、ReentrantReadWritelock

​	采用读写分离的策略，允许多个线程可以同时获取读锁。

​	读写锁的内部维护了一个 ReadLock 和一个 WriteLock，它们依赖 Sync 实现具体功能。而 Sync 继承自 AQS，并且也提供了公平和非公平的实现。

​	ReentrantReadWriteLock 巧妙地使用 state 的高 16 位表示读状态，也就是获取到读锁的次数；使用低 16 位表示获取到写锁的线程的可重入次数。

#### a、ConcurrentLinkedQueue（多线程同时操作一个 collection 时使用）

​	**CAS** 算法实现的线程安全的**无界非阻塞队列**，其底层数据结构使用单向链表实现，对于入队和出队操作使用 CAS 来实现线程安全。按FIFO原则进行排序。

​	入队、出队都是操作使用 volatile 修饰的 tail、 head 节点，要保证在多线程下出入队线程安全，只需要保证这两个 Node 操作的可见性和原子性即可。

​	由于 volatile 本身可以保证可见性，所以只需要保证对两个变量操作的原子性即可。

#### b、LinkedBlockingQueue（一般用于生产者消费者场景）

​	独占锁实现的单向链表(head为哨兵节点)阻塞队列，AutomicInteger 原子变量记录队列元素个数，两个 ReentrantLock 的实例，分别用来控制元素入队和出队的原子性，notEmpty 和 notFull 是条件变量，它们内部都有一个条件队列用来存放进队和出队时被阻塞的线程，其实这是生产者一消费者模型。默认队列容量0x7fffffff，用户也可以自己指定容量，所以从一定程度上可以说 LinkedBlockingQueue 是有界阻塞队列。

#### c、ArrayBlockingQueue

​	有界数组方式实现的阻塞队列，putIndex 变量表示入队元素下标，takeIndex 是出队下标，count 统计队列元素个数。 从定义可知，这些变量并没有使用 volatile 修饰，这是因为访问这些变量都是在锁块内，而加锁己经保证了锁块内变量的内存可见性了。 

​	另外有个独占锁 lock 用来保证出、入队操作的原子性，这保证了同时只有一个线程可以进行入队、出队操作。 另外，notEmpty、 notFull 条件变量用来进行出、入队的同步。

#### d、PriorityBlockingQueue

​	带优先级的无界阻塞队列，每次出队都返回优先级最高或者最低的元素。其内部是使用**平衡二叉树堆**实现的，所以直接遍历队列元素不保证有序。默认使用对象的 compareTo 方法提供比较规则，如果需要自定义比较规则则可以自定义 comparators。

​	PriorityBlockingQueue 类似于 ArrayBlockingQueue，在内部使用一个独占锁来控制同时只有一个线程可以进行入队和出队操作。另外，前者只使用了一个 notEmpty 条件变量而没有使用 notFull，这是因为前者是无界队列，执行 put 操作时永远不会处于 await 状态，所以也不需要被唤醒。而 take 方法是阻塞方法，并且是可被中断的。当需要存放有优先级的元素时该队列比较有用。

#### e、DelayQueue

​	DelayQueue 并发队列是一个无界阻塞延迟队列，队列中的每个元素都有个过期时间，当从队列获取元素时，只有过期元素才会出队列。队列头元素是最快要过期的元素。

​	其内部使用 PriorityQueue 存放数据，使用 ReentrantLock 实现线程同步。另外队列里面的元素要实现 Delayed 接口，其中一个是获取当前元素到过期时间剩余时间的接口，在出队时判断元素是否过期了，一个是元素之间比较的接口，因为这是一个有优先级的队列。

