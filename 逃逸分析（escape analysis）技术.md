### 逃逸分析（escape analysis）技术

#### 一、VM优化之逃逸分析与分配消除

​	逃逸分析是 JVM 的一项自动分析变量作用域的技术，它可以用来实现某些特殊的优化。要了解逃逸分析背后的基本原理，我们先来看下这段有问题的 C 代码——当然这个是没法用 Java 来写的：

```java
int * get_the_int() {
	int i = 42;
	return &amp;i;
}
```

​	这段 C 代码在栈上创建了一个 int 类型的变量，然后把它的指针作为函数的返回值返回了。这样做是有问题的，因为当 get_the_int() 函数返回的时候，int 所在的栈帧就已经被销毁了，后面你再去访问这个地址的话，就不知道里面存储的到底是什么了。

​	Java 平台设计的一个主要目标就是要消除这种类型的 bug。从设计上，JVM 就不具备这种低级的“根据位置索引来读内存”的能力。这类操作对应的 Java 字节码是 putfield 和 getfield。

来看下这段Java代码：

```java
public class TestDemo {
    public static void main(String[] args) throws SQLException {
        Random random = new Random();
        int sameArea = 0;
        for (int i = 0; i < 100000000; i++) {
            Rect r1 = new Rect(random.nextInt(5), random.nextInt(5));
            Rect r2 = new Rect(random.nextInt(5), random.nextInt(5));
            if (r1.sameArea(r2)) {
                sameArea++;
            }
        }
        System.out.println(sameArea);
    }
    static class Rect {
        private int v;
        private int h;
        public Rect(int v, int h) {
            this.v = v;
            this.h = h;
        }
        public int area() {
            return v * h;
        }
        public boolean sameArea(Rect targer) {
            return this.area() == targer.area();
        }
    }
}
```

那么 main 方法里会创建 2 亿个 Rect 对象：一亿个 r1，一亿个 r2，真是这样么？

​	不过，如果某个对象只是在方法内部创建并使用的话，也就是说，它不会传递到另一个方法中或者作为返回值返回，那么运行时程序就还能做得更聪明一些。

​	你可以说这个对象是没有逃逸出去的，因此运行时（其实就是 JIT 编译器）做的这个分析又叫做逃逸分析。
如果一个对象没有逃逸出去，那也就是说 JVM 可以针对这个对象做一些类似“栈自动分配”的事情。在这个例子当中，这个对象不会从堆上分配空间，因此它也不需要垃圾回收器来回收。

​	一旦使用这个“栈分配（stack-allocated）”对象的方法返回了，这个对象所占用的内存也就自动被释放掉了。
事实上，HotSpot VM 的 C2 编译器做的事情要比栈分配要复杂得多：

```c++
typedef enum {
	// 这个对象可以用标量来代替。这种分配消除技术叫标量替换（scalar replacement）。这意味着这个对象会被拆解成它的构成字段，这就相当于分配对象的操作变成了在方法内部创建多个局部变量。
	// 完成这个之后，另一项 HotSpot VM 的 JIT 技术会参与进来，它会将这些字段（事实上已经是局部变量了）存储到 CPU 的寄存器中（如果有必要就存储在栈上）。
	NoEscape = 1;
	//
	ArgEscape = 2;
	//
	GlobalEscape = 3;
}
```

​	在上述例子中，如果只看源代码，你会认为 r1 对象是不会逃逸出 main 方法外的，但 r2 会作为参数传给 r1 的sameArea 方法，因此它逃逸出了 main 方法外。

​	根据上面的分类，乍一看的话 r1 应该归类为 NoEscape，而 r2 应该归为 ArgEscape。不过这个结论是错误的，原因有几点：

​	Java 中的方法调用最终会通过编译器替换为字节码 invoke。它会把调用目标（也就是接收对象，注：即要调用的对象）和入参填充到栈中，然后查找到这个方法，再分发给它（也就是执行这个方法）。这意味着接收对象也被传入了调用的方法中（它就是调用的方法里的 this 对象）。因此接收对象也逃逸出了当前域；在这个例子中，这意味着如果逃逸分析分析完这段 Java 代码，r1 和 r2 都会归类为 ArgEscape。

​	如果就只是这样的话，那么分配消除的使用场景就很有限了。所幸的是，HotSpot VM 能做得更好，sameArea() 方法很小（只有17个字节的字节码），在本例中也会被频繁调用，因此它是方法内联（method inlined）的一个理想对象。 

​	area() 方法的调用的确被内联进了调用方 sameArea() 方法里，而 sameArea() 又被内联到了 main() 方法的循环体中。方法内联是最早的优化，因为它首先把相关联的代码都聚合在了一起，为其它优化打开了大门。现在 sameArea() 方法和 area() 方法都被内联进来了，方法域的问题不复存在，所有的变量都只在 main() 方法的作用域内了。也就是说逃逸分析不会再把 r1 和 r2 视作 ArgEscape 类型：方法内联之后，它们现在都被归类为 NoEscape。这个结果看起来可能有悖常理，不过需要记住的是 JIT 编译器并不是通过原始代码来进行优化的。

​	上面的例子中，这些对象的分配都不会在堆上进行了，会把它们的字段拆解成独立的值。寄存器分配器通常会把拆解出来的字段直接放到寄存器中，不过如果没有足够可用的寄存器，那剩下的字段会被存储到栈上。这种情况被称为栈溢出（stack spill，注：和stack overflow不同）。

​	现代 JVM 中逃逸分析是默认开启的，如果针对上述代码进行 GC 日志分析，可以看到根本没有发生 GC 事件，只是在进程退出时往日志里记录了下堆的摘要信息。通过 JVM 参数 -XX:-DoEscapeAnalysis 来关掉它后发现，由于 Eden 区空间满了，导致了内存分配失败、需要进行垃圾回收，因此触发了GC事件。

#### 二、JVM优化之逃逸分析及锁消除

​	如果能确认某个加锁的对象不会逃逸出局部作用域，就可以进行锁删除。这意味着这个对象同时只可能被一个线程访问，因此也就没有必要防止其它线程对它进行访问了。这样的话这个锁就是可以删除的。这个便叫做锁消除。

​	StringBuffer 是一个使用同步方法的线程安全的类，它可以用来很好地诠释锁消除。StringBuffer 是 Java1.0 的时候开始引入的，可以用来高效地拼接不可变的字符串对象。它对所有 append 方法都进行了同步操作，以确保当多个线程同时写入同一个 StringBuffer 对象的时候也能够保证构造中的字符串可以安全地创建出来：

```java
@Override
public synchronized StringBuffer append(String str) {
    toStringCache = null;
    super.append(str);
    return this;
}
// StringBuilder：
@Override
public StringBuilder append(String str) {
    super.append(str);
    return this;
}
```

​	调用 StringBuffer 的 append 方法的线程，必须得获取到这个对象的内部锁（也叫监视器锁）才能进入到方法内部，在退出方法前也必须要释放掉这个锁。不过在 HotSpot 虚拟机引入了逃逸分析之后，在调用像 StringBuffer 这样的对象的同步方法时，就能够自动地把锁消除掉了。这只会出现在方法域内部所创建的对象上，只有这样才能保证不会发生逃逸。在 Java 8 中它是默认开启的，不过你也可以通过 -XX:-DoEscapeAnalysis 这个 VM 参数来关掉它，这样可以看下优化的效果。

##### 锁粗化：

​	当连续获取同一个对象的锁时，HotSpot 虚拟机会去检查多个锁区域是否能合并成一个更大的锁区域。这种聚合被称作锁粗化，它能够减少加锁和解锁的消耗。当 HotSpot JVM 发现需要加锁时，它会尝试往前查找同一个对象的解锁操作。如果能匹配上，它会考虑是否要将两个锁区域作合并，并删除一组 解锁/加锁 操作。

​	锁粗化是默认开启的，不过也可以通过启动参数 -XX:-EliminateLocks 来关掉它。

##### 嵌套锁：

​	同步块可能会一个嵌套一个，进而两个块使用同一个对象的监视器锁来进行同步也是很有可能的。这种情况我们称之为嵌套锁，HotSpot 虚拟机是可以识别出来并删除掉内部块中的锁的。

​	Java 8 中的嵌套锁删除只有在锁被声明为 static final 或者锁的是 this 对象时才可能发生。

​	嵌套锁优化是默认开启的，不过也可以通过启动参数 -XX:-EliminateNestedLocks 来关掉它。

##### 数组及逃逸分析：

​	非堆上分配的空间要么存储在栈上，要么就在 CPU 寄存器中，这些都是相对稀缺的资源，因此逃逸分析和其它优化一样，（在实现上）肯定会面临妥协。HotSpot JVM 上的一个默认限制是大于 64 个元素的数组不会进行逃逸分析优化。这个大小可以通过启动参数 -XX:EliminateAllocationArraySizeLimit=n 来进行控制，n 是数组的大小。

​	假设有段热点代码，它会去分配一个临时数组用于从缓存中读取数据。如果逃逸分析发现这个数组的作用域没有逃逸出方法体外，便不会在堆上分配内存。不过如果数组大小超过64的话（哪怕并不是全都用到）便仍会存储到堆里。这样数组的逃逸分析优化便不会起作用，也仍会从堆内分配内存。

#### 三、是不是所有的对象和数组都会在堆内存分配空间

​	有一种特殊情况，那就是如果经过逃逸分析后发现，一个对象并没有逃逸出方法的话，那么就可能被优化成栈上分配。这样就无需在堆上分配内存，也无须进行垃圾回收了。