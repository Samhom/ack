## Thread 相关

### 1、三种线程创建方式

分别为实现 Runnable 接口的 run 方法，继承 Thread 类并重写 run 的方法，使用 FutureTask 方式 。

##### 继承 Thread 类并重写 run 的方法

​	调用了 start 方法后才真正启动了线程。其实调用 start 方法后线程并没有马上执行而是处于就绪状态，这个就绪状态是指该线程已经获取了除 CPU 资源外的其他资源，等待获取 CPU 资源后才会真正处于运行状态。

​	一旦 run 方法执行完毕，该线程就处于终止状态。不好的地方是 Java 不支持多继承，如果继承了 Thread 类，那么就不能再继承其他类。另外任务与代码没有分离，当多个线程执行一样的任务时需要多份任务代码。

##### 实现 Runnable 接口

​	两个线程共用一个 task 代码逻辑，如果需要，可以给 RunableTask 添加参数进行任务区分。另外，RunableTask 可以继承其他类。但是上面介绍的两种方式都有一个缺点，就是任务没有返回值。

##### 使用 FutureTask 的方式

​	实现 Callable 接口的 call() 方法。在 main 函数内首先创建一个 FutrueTask 对象（构造函数为 CallerTask 的实例），然后使用创建的 FutrueTask 对象作为任务创建了一个线程并且启动它，最后通过 futureTask.get(）等待任务执行完毕并返回结果。

### 2、线程中断

​	线程中断是一种线程间的协作模式，通过设置线程的中断标志并不能直接终止该线程的执行，而是被中断的线程根据中断状态自行处理。

##### void interrupt() 方法

​	中断线程。 例如，当线程 A 运行时，线程 B 可以调用钱程 A 的 interrupt() 方法来设置线程 A 的中断标志为 true 并立即返回。设置标志仅仅是设置标志，线程 A 实际并没有被中断，它会继续往下执行。如果线程 A 因为调用了 wait 系列函数、 join 方法或者 sleep 方法而被阻塞挂起，这时候若线程 B 调用线程A 的 interrupt（） 方法，线程 A 会在调用这些方法的地方抛出 InterruptedException 异常而返回。

##### boolean isInterrupte() 方法

检测当前线程是否被中断。如果是返回 true，否则返回 false。

```java
public boolean isInterrupted(){
	//传递 false ，说明不清除中断标志
	return isInterrupted(false) ;
}
```

##### boolean interrupted() 方法

由于它是静态方法，因此不能在特定的线程上使用，只能报告调用它的线程的中断状态；如果该方法被调用两次，则第二次一般是返回false，如果线程不存活，则返回false。

​	检测当前线程是否被中断，如果是返回 true，否则返回 false。与 isInterrupted 不同的是，该方法如果发现当前线程被中断，则会清除中断标志，并且该方法是 static 方法，可以通过 Thread 类直接调用。另外从下面的代码可以知道，在 interrupted() 内部是获取当前调用线程的中断标志而不是调用 interrupted() 方法的实例对象的中断标志。

```java
public static boolean interrupted() {
	// 清除 中断标志
	return currentThread().isinterrupted(true);
}
```

```java
// 线程退出条件
while (!Thread.currentThread().isInterrupted() && more work to do) {
	// do more work;	
}
```

### 3、如何避免线程死锁

​	要想避免死锁，只需要破坏掉至少一个构造死锁的必要条件即可，只有请求并持有和环路等待条件是可以被破坏的。造成死锁的原因其实和申请资源的顺序有很大关系，使用资源申请的有序性原则就可以避免死锁。

### 4、守护线程与用户线程

​	区别之一是当最后一个非守护线程结束时，JVM 会正常退出，而不管当前是否有守护线程，也就是说守护线程是否结束并不影响 JVM 的退出。言外之意，只要有一个用户线程还没结束，正常情况下 JVM 就不会退出。父线程结束后，子线程还是可以继续存在的，也就是子线程的生命周期并不受父线程的影响。

### 5、ThreadLocal、ThreadLocalMap、Thread、InheritableThreadLocal

​	Thread 类中有一个 threadLocals 和一个 inheritableThreadLocals，它们都是 ThreadLocalMap 类型的变量，而 ThreadLocalMap 是一个定制化的 Hashmap。 在默认情况下，每个线程中的这两个变量都为 null，只有当前线程第一次调用 ThreadLocal 的 set 或者 get 方法时才会创建它们。 

​	其实每个线程的本地变量不是存放在 ThreadLocal 实例里面，而是存放在调用线程的 threadLocals 变量里面。 也就是说，ThreadLocal 类型的本地变量存放在具体的线程内存空间中。ThreadLocal 就是一个工具壳，它通过 set 方法把 value 值放入调用线程的 threadLocals 里面并存放起来，当调用线程调用它的 get 方法时，再从当前线程的 threadLocals 变量里面将其拿出来使用。 如果调用线程一直不终止，那么这个本地变量会一直存放在调用线程的 threadLocals 变量里面，所以当不需要使用本地变量时可以通过调用 ThreadLocal 变量的 remove 方法，从当前线程的 threadLocals 里面删除该本地变量。另外，Thread 里面的 threadLocals 为何被设计为 map 结构？很明显是因为每个线程可以关联多个 ThreadLocal 变量。

但是使用ThreadLocal和ThreadLocalRandom的工具类的需要特别注意：

##### 造成内存溢出

​	每个线程的本地变量存放在线程自己的内存变量 threadLocals 中，如果当前线程一直不消亡，那么这些本地变量会一直存在，所以可能会造成内存溢出，因此使用完毕后要记得调用 ThreadLocal 的 remove 方法删除对应线程的 threadLocals 中的本地变量。在高级篇要讲解的只 JUC 包里面的 ThreadLocalRandom，就是借鉴 ThreadLocal 的思想实现的。

##### Threadlocal 不支持继承性

​	同一个 ThreadLocal 变量在父线程中被设置值后，在子线程中是获取不到的。因为在子线程 thread 里面调用 get 方法时当前线程为 thread 线程，而调用 set 方法设置线程变量的是父线程，两者是不同的线程，自然子线程访问时返回 null。那么有没有办法让子线程能访问到父线程中的值？ 

​	答案是有，那就是 InheritableThreadLocal。

##### InheritableThreadLocal 实现原理大致如下

​	在创建子线程的时候，会把父线程的 inheritableThreadLocals 变量中的值拷贝一份到子线程的 inheritableThreadLocals 变量，也就是说当父线程创建子线程时，构造函数会把父线程中 inheritableThreadLocals 变量里面的本地变量复制一份保存到子线程的 inheritableThreadLocals 变量里面。
