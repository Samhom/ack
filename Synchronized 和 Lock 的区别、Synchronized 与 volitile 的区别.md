## Synchronized 和 Lock 的区别、Synchronized 与 volitile 的区别

### 1、Synchronized 和 Lock 的区别

- Lock 是一个接口，而 synchronized 是 Java 中的关键字，synchronized 是内置的语言实现；
- synchronized 在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而 Lock 在发生异常时，如果没有主动通过 unLock() 去释放锁，则很可能造成死锁现象，因此使用 Lock 时需要在 finally 块中释放锁；这是因为 synchronized 编译成字节码指令后，会对应一个 monitorenter 和两个 monitorexit 一个在流程正常结束时调用，一个在异常发生时调用。
- Lock 可以让等待锁的线程响应中断，而 synchronized 却不行，使用 synchronized 时，等待的线程会一直等待下去，不能够响应中断；
- 通过 Lock 可以知道有没有成功获取锁，而 synchronized 却无法办到。
- 性能上来说，在资源竞争不激烈的情形下，Lock 性能稍微比 synchronized 差点（编译程序通常会尽可能的进行优化 synchronized ）。但是当同步非常激烈的时候，synchronized 的性能一下子能下降好几十倍。而 ReentrantLock 还能维持常态。

### 2、Synchronized 与 volitile 的区别

##### synchronized 的内存语义

​	这个内存语义就可以解决共享变量内存可见性问题。进入 synchronized 块的内存语义是把在 synchronized 块内使用到的变量从线程的工作内存中清除，这样在 synchronized 块内使用到该变量时就不会从线程的工作内存中获取，而是直接从主内存中获取。 退出 synchronized 块的内存语义是把在 synchronized 块内对共享变量的修改刷新到主内存。其实这也是加锁和释放锁的语义，当获取锁后会清空锁块内本地内存中将会被用到的共享变量，在使用这些共享变量时从主内存进行加载，在释放锁时将本地内存中修改的共享变量刷新到主内存。除可以解决共享变量内存可见性问题外，synchronized 经常被用来实现原子性操作。

​	另外请注意，synchronized 关键字会引起线程上下文切换并带来线程调度开销。

##### volatile 关键字

​	当一个变量被声明为 volatile 时，线程在写入变量时不会把值缓存在寄存器或者其他地方，而是会把值刷新回主内存。 当其他线程读取该共享变量时，会从主内存重新获取最新值，而不是使用当前线程的工作内存中的值。 

​	volatile 的内存语义和 synchronized 有**相似之处**，具体来说就是，当线程写入了 volatile 变量值时就等价于线程退出 synchronized 同步块（把写入工作内存的变量值同步到主内存），读取 volatile 变量值时就相当于
进入同步块（先清空本地内存变量值，再从主内存获取最新值）。

​	另外，通过把变量声明为 volatile 可以避免指令重排序问题。写 volatile 变量时，可以确保 volatile 写之前的操作不会被编译器重排序到 volatile 写之后。 读 volatile 变量时，可以确保 volatile 读之后的操作不会被编译器重排序到 volatile 读之前。

​	但是，volatile 虽然提供了可见性保证，但并不保证操作的原子性。

​	写入变量值不依赖、变量的当前值时、读写变量值时没有加锁时候可以使用 volatile 对变量就行修饰。

### 3、synchronized 方法和 synchronized 块：

可同时参考：https://github.com/farmerjohngit/myblog/issues/12

这时两种 Java 中实现同步的基础语义。

```java
public class SyncTest {
    public void syncBlock() {
    	// synchronized 块
        synchronized (this) {
            System.out.println("hello block");
        }
    }
    // synchronized 方法
    public synchronized void syncMethod() {
        System.out.println("hello method");
    }
}
```

当 SyncTest.java 被编译成 class 文件的时候，synchronized 关键字和 synchronized 方法的字节码略有不同，在JVM底层，对于这两种 synchronized 语义的实现大致相同。我们可以用 javap -v 命令查看 class 文件对应的JVM字节码信息，部分信息如下：

```java
{
  public void syncBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter				 	  // monitorenter指令进入同步块
         4: getstatic     #2    // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3                  // String hello block
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit						  // monitorexit指令退出同步块
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit		// monitorexit指令退出同步块 两个monitorexit指令的原因是：为了保证抛异常的情况下也能释放锁，所以javac为同步代码块添加了一个隐式的try-finally，在finally中会调用monitorexit命令释放锁
        20: aload_2
        21: athrow
        22: return
      Exception table:
         from    to  target type
             4    14    17   any
            17    20    17   any
 
  public synchronized void syncMethod();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED      //添加了ACC_SYNCHRONIZED标记 在JVM进行方法调用时，发现调用的方法被 ACC_SYNCHRONIZED 修饰，则会先尝试获得锁
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5                  // String hello method
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
}
```

​	JDK 1.6 引入了两种新型锁机制：偏向锁和轻量级锁，它们的引入是为了解决在没有多线程竞争或基本没有竞争的场景下因使用传统锁机制带来的性能开销问题。

​	对象头，它是实现多种锁机制的基础，因为在 Java 中任意对象都可以用作锁，因此必定要有一个映射关系来存储该对象以及其对应的锁信息（比如当前哪个线程持有锁，哪些线程在等待）。

​	在 JVM 中，对象在内存中除了本身的数据外还会有个对象头，对于普通对象而言，其对象头中有两类信息：**mark word** 和类型指针。另外对于数组而言还会有一份记录数组长度的数据。

​	类型指针是指向该对象所属类对象的指针，mark word 用于存储对象的 HashCode、GC分代年龄、锁状态等信息。在 32 位系统上 mark word 长度为 32 bit，64 位系统上长度为 64 bit。

![img](https://camo.githubusercontent.com/ba0c739510c9092e06a37903670441072239d7c7/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f31312f32382f313637353964643162306239363236383f773d37323026683d32353026663d6a70656726733d3337323831)

- 当对象状态为偏向锁（biasable）时，mark word 存储的是偏向的线程ID
- 当状态为轻量级锁（lightweight locked）时，mark word 存储的是指向线程栈中 Lock Record 的指针
- 当状态为重量级锁（inflated）时，为指向堆中的 monitor 对象的指针

### 4、synchronize 性能优化：

​	JDK 1.6中对 synchronize 的实现进行了各种优化，使得它显得不是那么重了，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。

​	锁主要存在四种状态，依次是：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态。他们会随着竞争的激烈而逐渐升级。注意锁可以升级不可降级，这种策略是为了提高获得锁和释放锁的效率。

#### 优化技术：

- 自旋锁：
  ​	线程的阻塞和唤醒需要 CPU 从用户态转为核心态，频繁的阻塞和唤醒对 CPU 来说是一件负担很重的工作，势必会给系统的并发性能带来很大的压力。同时我们发现在许多应用上面，对象锁的锁状态只会持续很短一段时间，为了这一段很短的时间频繁地阻塞和唤醒线程是非常不值得的。所谓自旋锁，就是让该线程等待一段时间，不会被立即挂起，看持有锁的线程是否会很快释放锁。 由于 Java 中的线程是与操作系统中的线程一一对应的，所以当一个线程在获取锁（比如独占锁）失败后，会被切换到内核状态而被挂起。当该线程获取到锁时又需要将其切换到内核状态而唤醒该线程。 而从用户状态切换到内核状态的开销是比较大的，在一定程度上会影响并发性能。自旋锁则是，当前线程在获取锁时，如果发现锁已经被其他线程占有，它不马上阻塞自己，在不放弃 CPU 使用权的情况下，多次尝试获取（默认次数是 10，可以使用 -XX :PreBlockSpinsh 参数设置该值），很有可能在后面几次尝试中其他线程己经释放了锁。如果尝试指定的次数后仍没有获取到锁则当前线程才会被阻塞挂起。 由此看来自旋锁是使用 CPU 时间换取线程阻塞与调度的开销，但是很有可能这些 CPU 时间白白浪费了。

- 适应自旋锁：
  所谓自适应就意味着自旋的次数不再是固定的，它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。反之，如果对于某个锁，很少有自旋能够成功的，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。有了自适应自旋锁，随着程序运行和性能监控信息的不断完善，虚拟机对程序锁的状况预测会越来越准确，虚拟机会变得越来越聪明。

- 锁消除：
  JVM 检测到不可能存在共享数据竞争，这时 JVM 会对这些同步锁进行锁消除。锁消除的依据是逃逸分析的数据支持。如 StringBuffer、Vector、HashTable 等，这个时候会存在隐形的加锁操作：

  ```java
  public void vectorTest() {
      Vector<String> vector = new Vector<String>();
      for(int i = 0 ; i < 10 ; i++) {
          vector.add(i + "");
      }
      System.out.println(vector);
  }
  ```

  在运行这段代码时，JVM 可以明显检测到变量 vector 没有逃逸出方法 vectorTest() 之外，所以 JVM 可以大胆地将 vector 内部的加锁操作消除。

- 锁粗化：
  如果一系列的连续加锁解锁操作，可能会导致不必要的性能损耗，所以引入锁粗化的概念。将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁。如上面实例：vector 每次 add 的时候都需要加锁操作，JVM检测到对同一个对象（vector）连续加锁、解锁操作，会合并一个更大范围的加锁、解锁操作，即加锁解锁操作会移到 for 循环之外。

#### 5、锁状态（无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态 4 种）：

##### 偏向锁：

​	通俗的讲，偏向锁就是在运行过程中，对象的锁偏向某个线程。即在开启偏向锁机制的情况下，某个线程获得锁，当该线程下次再想要获得锁时，不需要再获得锁（即忽略 synchronized 关键词），直接就可以执行同步代码，比较适合竞争较少的情况。

偏向锁的获取流程：

- （1）查看 Mark Word 中偏向锁的标识以及锁标志位，若偏向锁标识为 1 且锁标志位为 01，则该锁为可偏向状态。
- （2）若为可偏向状态，则测试 Mark Word 中的线程 ID 是否与当前线程相同，若相同，则直接执行同步代码，否则进入下一步。
- （3）当前线程通过 CAS 操作竞争锁，若竞争成功，则将 Mark Word 中线程 ID 设置为当前线程ID，然后执行同步代码，若竞争失败，进入下一步。
- （4）当前线程通过 CAS 竞争锁失败的情况下，说明有竞争。当到达全局安全点时之前获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。

偏向锁的释放流程：

​	偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁状态的线程才会释放锁，线程不会主动去释放偏向锁。偏向锁的撤销需要等待全局安全点（即没有字节码正在执行），它会暂停拥有偏向锁的线程，撤销后偏向锁恢复到未锁定状态或轻量级锁状态。

​	偏向锁对 synchronized 关键词的优化在单个线程中或竞争较少的线程中是很成功的。但是在多线程竞争十分频繁的情况下，偏向锁不仅不能提高效率，反而会因为不断地重新设置偏向线程ID等其他消耗而降低效率。

代码示例：

```java
/**
 * 开启偏向锁参数：-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0  耗时 2400ms 左右
 * 禁用偏向锁参数：-XX:-UseBiasedLocking  耗时800ms左右
 */
public static void main(String[] args) throws InterruptedException {
    long time1 = System.currentTimeMillis();
    Vector<Integer> vector = new Vector<Integer>();
    for (int i = 0; i < 100000000; i++) {
        vector.add(100);// add 是 synchronized 操作
    }
    System.out.println(System.currentTimeMillis() - time1);
}
```

##### 轻量级锁：

​	很多情况下，在 Java 程序运行时，同步块中的代码都是不存在竞争的，不同的线程交替的执行同步块中的代码。这种情况下，用重量级锁是没必要的。因此JVM引入了轻量级锁的概念。

​	轻量级锁不是用来替代传统的重量级锁的，而是在没有多线程竞争的情况下，使用轻量级锁能够减少性能消耗，但是当多个线程同时竞争锁时，轻量级锁会膨胀为重量级锁。

​	线程在执行同步块之前，JVM  会先在当前的线程的栈帧中创建一个 Lock Record，其包括一个用于存储对象头中的 mark word（官方称之为 Displaced Mark Word ）以及一个指向对象的指针。

<https://camo.githubusercontent.com/3579362e569b9ea6046cf34702dba32eceb212bb/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f31312f32382f313637353964643162323461633733643f773d38363926683d33353126663d706e6726733d3331313531>

轻量级锁的加锁过程：

- 1.在线程栈中创建一个 Lock Record，将其 obj（即上图的Object reference）字段指向锁对象。
- 2.直接通过CAS指令将Lock Record的地址存储在对象头的mark word中，如果对象处于无锁状态则修改成功，代表该线程获得了轻量级锁。如果失败，进入到步骤3。
- 3.如果是当前线程已经持有该锁了，代表这是一次锁重入。设置Lock Record第一部分（Displaced Mark Word）为null，起到了一个重入计数器的作用。然后结束。
- 4.走到这一步说明发生了竞争，需要膨胀为重量级锁。

解锁过程：

- 1.遍历线程栈,找到所有obj字段等于当前锁对象的Lock Record。
- 2.如果Lock Record的Displaced Mark Word为null，代表这是一次重入，将obj设置为null后continue。
- 3.如果Lock Record的Displaced Mark Word不为null，则利用CAS指令将对象头的mark word恢复成为Displaced Mark Word。如果成功，则continue，否则膨胀为重量级锁。

### 6、悲观锁和乐观锁、公平锁与非公平锁、独占锁与共享锁、可重入锁

##### a、悲观锁和乐观锁

​	乐观锁直到提交时才锁定，所以不会产生任何死锁。

##### b、公平锁与非公平锁

​	ReentrantLock 提供了公平和非公平锁的实现。

##### c、独占锁与共享锁

独占锁是一种悲观锁，共享锁则是一种乐观锁。

根据锁只能被单个线程持有还是能被多个线程共同持有，锁可以分为独占锁和共享锁。

独占锁保证任何时候都只有一个线程能得到锁，ReentrantLock 就是以独占方式实现的。

共享锁则可以同时由多个线程持有，例如 ReadWriteLock 读写锁，它允许一个资源可以被多线程同时进行读操作。

##### d、可重入锁

​	当一个线程再次获取它自己己经获取的锁时如果不被阻塞，那么我们说该锁是可重入的，也就是只要该线程获取了该锁，那么可以无限次数地进入被该锁锁住的代码。

​	实际上， synchronized 内部锁是可重入锁。 可重入锁的原理是在锁内部维护一个线程标示，用来标示该锁目前被哪个线程占用，然后关联一个计数器。一开始计数器值为 0,说明该锁没有被任何线程占用。 

​	当一个钱程获取了该锁时，计数器的值会变成 1 ，这时其他线程再来获取该锁时会发现锁的所有者不是自己而被阻塞挂起。但是当获取了该锁的线程再次获取锁时发现锁拥有者是自己，就会把计数器值加加 1,当释放锁后计数器值减 1。 当计数器值为 0 时，锁里面的线程标示被重置为 null，这时候被阻塞的线程会被唤醒来竞争获取该锁。