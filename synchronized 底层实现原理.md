## synchronized 底层实现原理

### 一、对于 synchronized (this) 反编译字节码后结果如下：

![img](https://images2015.cnblogs.com/blog/820406/201604/820406-20160414215316020-1963237484.png)

##### monitorenter：

​	每个对象有一个监视器锁（monitor）。当monitor 被占用时就会处于锁定状态，线程执行 monitorenter 指令时尝试获取 monitor 的所有权，过程如下：

- 如果 monitor 的进入数为 0，则该线程进入 monitor，然后将进入数设置为 1，该线程即为 monitor（**一个对象都有且仅有一个与之对应的`monitor`对象**） 的所有者。
- 如果线程已经占有该 monitor，只是重新进入，则进入 monitor 的进入数加 1。
- 如果其他线程已经占用了 monitor，则该线程进入阻塞状态，直到 monitor 的进入数为 0，再重新尝试获取 monitor 的所有权。

##### monitorexit：

​	执行 monitorexit 的线程必须是 objectref 所对应的 monitor 的所有者。指令执行时，monitor 的进入数减 1，如果减 1 后进入数为 0，那线程退出 monitor，不再是这个 monitor 的所有者。其他被这个 monitor 阻塞的线程可以尝试去获取这个 monitor 的所有权。 

​	wait/notify 等方法也依赖于 monitor 对象，这就是为什么只有在同步的块或者方法中才能调用 wait/notify 等方法，否则会抛出 java.lang.IllegalMonitorStateException 的异常的原因。

### 二、对于 public synchronized void method() 反编译字节码结果如下：

![img](https://images2015.cnblogs.com/blog/820406/201604/820406-20160418202553429-1642545018.png)

​	从反编译的结果来看，方法的同步并没有通过指令 monitorenter 和 monitorexit 来完成（理论上其实也可以通过这两条指令来实现），不过相对于普通方法，其**常量池中**多了 ACC_SYNCHRONIZED 标示符。

​	当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取 monitor，获取成功之后才能执行方法体，方法执行完后再释放 monitor。在方法执行期间，其他任何线程都无法再获得同一个 monitor 对象。 其实本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。

### 三、从对象头的角度继续剖析

​	同步和`monitor`有关，而`monitor`则和对象头有关。在JVM中，对象是分成三部分存在的：对象头、实例数据、对其填充。

​	对象头是 synchronized 实现锁的基础，synchronized申请锁、上锁、释放锁都与对象头有关。

​	对象头主要结构是由`Mark Word` 和 `Class Metadata Address`组成，**其中** Mark Word 存储对象的hashCode、锁信息、分代年龄、GC 标志等信息 ， Class Metadata Address 是类型指针指向对象的类元数据，JVM 通过该指针确定该对象是哪个类的实例 。

​	每一个锁都对应一个 monitor 对象，在 HotSpot 虚拟机中它是由 ObjectMonitor 实现的（C++实现）。每个对象都存在着一个 monitor 与之关联。

```java
ObjectMonitor() {
    _header       = NULL;
    _count        = 0;  // 锁计数器
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; // 处于 wait 状态的线程，会被加入到 _WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; // 处于等待锁 block 状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

​	ObjectMonitor 中有两个队列 WaitSet 和 EntryList，用来保存 ObjectWaiter 对象列表(每个等待锁的线程都会被封装 ObjectWaiter 对象)，owner 指向持有 ObjectMonitor 对象的线程。

​	当多个线程同时访问一段同步代码时，首先会进入 EntryList 集合，当线程获取到对象的 monitor 后把 monitor 中的 owner 变量设置为当前线程同时 monitor 中的计数器 count 加 1。

​	若线程调用 wait() 方法，将释放当前持有的 monitor，owner 变量恢复为 null，count 自减 1，同时该线程进入 WaitSet 集合中等待被唤醒。若当前线程执行完毕也将释放 monitor (锁)并复位变量的值，以便其他线程进入获取 monitor(锁)。  

​	monitor 对象存在于每个 Java 对象的对象头中(存储的指针的指向)，synchronized 锁便是通过这种方式获取锁的，也是为什么 Java 中任意对象可以作为锁的原因，同时也是 notify/notifyAll/wait 等方法存在于顶级对象 Object 中的原因。