## volatile 原理

#### volatile 两大作用

保证内存可见性、防止指令重拍（但不保证原子性，也就是它修饰的变量不一定是线程安全的）。

#### JVM 内存模型

主内存和线程独立的工作内存。

#### Java 内存模型规定

​	对于多个线程共享的变量，存储在主内存当中，每个线程都有自己独立的工作内存（比如 CPU 的寄存器），线程只能访问自己的工作内存，不可以访问其它线程的工作内存。

##### 工作内存与主内存之间交互的协议，定义了8种原子操作：

- lock: 将主内存中的变量锁定，为一个线程所独占
- unclock: 将 lock 加的锁定解除，此时其它的线程可以有机会访问此变量
- read: 将主内存中的变量值读到工作内存当中
- load: 将 read 读取的值保存到工作内存中的变量副本中
- use: 将值传递给线程的代码执行引擎
- assign: 将执行引擎处理返回的值重新赋值给变量副本
- store: 将变量副本的值存储到主内存中
- write: 将 store 存储的值写入到主内存的共享变量当中

#### 指令重排导致单例模式失效

懒加载方式的双重判断单例模式：

```java
public class Singleton {
    private static Singleton instance = null;
    private Singleton() {
    }

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    //非原子操作
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

instance = new Singleton() 可以抽象为下面几条 JVM 指令：

```java
memory = allocate();   // 1：分配对象的内存空间 
ctorInstance(memory);  // 2：初始化对象 
instance = memory;     // 3：设置 instance 指向刚分配的内存地址
```

上面操作 2 依赖于操作 1，但是操作 3 并不依赖于操作 2，所以 JVM 是可以针对它们进行指令的优化重排序的，经过重排序后如下：

```java
memory = allocate();   // 1：分配对象的内存空间 
instance = memory;     // 3：设置 instance 指向刚分配的内存地址
ctorInstance(memory);  // 2：初始化对象 
```

​	可以看到指令重排之后，instance 指向分配好的内存放在了前面，而这段内存的初始化被排在了后面。在线程A执行这段赋值语句，在初始化分配对象之前就已经将其赋值给 instance 引用，恰好另一个线程进入方法判instance 引用不为 null，然后就将其返回使用，导致出错。上述案例中使用关键字 volatile 对变量 instance 修饰后可避免可能出现的出错的问题。

#### volatile 原理

​	java 内存模型中讲到的 volatile 是基于 Memory Barrier 实现的，volatile 关键字通过提供“内存屏障”的方式来防止指令被重排序，为了实现 volatile 的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

​	编译器和 CPU 能够重排序指令，保证最终相同的结果，尝试优化性能。插入一条 Memory Barrier 会告诉编译器和 CPU：不管什么指令都不能和这条 Memory Barrier 指令重排序。

​	Memory Barrier 所做的另外一件事是强制刷出各种 CPU cache，如一个 Write-Barrier（写入屏障）将刷出所有在 Barrier 之前写入 cache 的数据，因此，任何 CPU 上的线程都能读取到这些数据的最新版本。

##### JMM内存屏障插入策略：

- 在每个 volatile 写操作的前面插入一个 StoreStore 屏障。
- 在每个 volatile 写操作的后面插入一个 StoreLoad 屏障。
- 在每个 volatile 读操作的后面插入一个 LoadLoad 屏障。
- 在每个 volatile 读操作的后面插入一个 LoadStore 屏障。

#### volatile 和 synchronized 区别

- volatile 无法同时保证内存可见性和原子性。
- volatile 本质是在告诉 jvm 当前变量在寄存器（工作内存）中的值是不确定的，需要从主存中读取；synchronized 则是锁定当前变量，只有当前线程可以访问该变量，其他线程被阻塞住。
- 作用域不同，volatile 仅能使用在变量级别；synchronized 则可以使用在变量、方法、和类级别的。
- volatile 不会造成线程的阻塞；synchronized 可能会造成线程的阻塞。
- volatile 标记的变量不会被编译器优化；synchronized 标记的变量可以被编译器优化。