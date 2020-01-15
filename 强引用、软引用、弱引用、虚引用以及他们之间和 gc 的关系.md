## 强引用、软引用、弱引用、虚引用以及他们之间和 gc 的关系

#### 强引用

​	如果一个对象具有强引用，那就类似于必不可少的物品，不会被垃圾回收器回收。当内存空间不足，Java虚拟机宁愿抛出 OutOfMemoryError 错误，使程序异常终止，也不回收这种对象。不过要注意的是，当 method1 运行完之后，object 都已经不存在了，所以它们指向的对象都会被 JVM 回收。如果想中断强引用和某个对象之间的关联，可以显示地将引用赋值为 null，这样一来的话，JVM 在合适的时间就会回收该对象。比如 ArraryList 类的 clear() 方法中就是通过将引用赋值为 null 来实现清理工作的。

#### 软引用

​	软引用是用来描述一些有用但并不是必需的对象，在 Java 中用 java.lang.ref.SoftReference 类来表示。对于软引用关联着的对象，只有在内存不足的时候 JVM 才会回收该对象。因此，这一点可以很好地用来解决 OOM 的问题，软引用非常适合于创建缓存。当系统内存不足的时候，缓存中的内容是可以被释放的。

​	redisson 的 cache 、guava 的 LocalCache、Histrix 的 HystrixTimer 都继承了 SoftReference。

​	软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被 JVM 回收，这个软引用就会被加入到与之关联的引用队列中。

使用例子：

```java
// 注意：wrf这个引用也是强引用，它是指向SoftReference这个对象的，
// 这里的软引用指的是指向new Obj()的引用，也就是SoftReference类中T
SoftReference<String> wrf = new SoftReference<String>(new Obj());

public class ImageData {
    private String path;
    private SoftReference<byte[]> dataRef;
    public ImageData(String path) {
        this.path = path;
        dataRef = new SoftReference<byte[]>(new byte[0]);
    }
    private byte[] readImage() {
        return new byte[1024 * 1024]; //省略了读取文件的操作
    }
    public byte[] getData() {
        byte[] dataArray = dataRef.get();
        if (dataArray == null || dataArray.length == 0) {
            dataArray = readImage();
            dataRef = new SoftReference<byte[]>(dataArray);
        }
        return dataArray;
    }
}
```

#### 弱引用

​	弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。

​	在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。

​	不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。

使用例子：

```java
String str = new String("abc");
WeakReference abcWeakRef = new WeakReference(str);
// 当垃圾回收器进行扫描回收时等价于：str = null; System.gc();
str = null;
```

​	弱引用最常见的用处是在集合类中，尤其在哈希表中。哈希表的接口允许使用任何 Java 对象作为键来使用。当一个键值对被放入到哈希表中之后，哈希表对象本身就有了对这些键和值对象的引用。如果这种引用是强引用的话，那么只要哈希表对象本身还存活，其中所包含的键和值对象是不会被回收的。如果某个存活时间很长的哈希表中包含的键值对很多，最终就有可能消耗掉 JVM 中全部的内存【ThreadLocal】。

​	对于这种情况的解决办法就是使用弱引用来引用这些对象，这样哈希表中的键和值对象都能被垃圾回收。Java 中提供了 WeakHashMap 来满足这一常见需求。

##### WeakHashMap：

```java
// entry:
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    V value;
    final int hash;
    Entry<K,V> next;
    Entry(Object key, V value, ReferenceQueue<Object> queue, int hash, Entry<K,V> next) {
        super(key, queue);
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }
    ...
}
```

```java
expungeStaleEntries() {...} //（剔除失效的Entry）:
// 当 key 失效的时候 gc 会自动把对应的Entry添加到这个引用队列中；
// 所有对 map 的操作都会直接或间接地调用到这个方法先移除失效的 Entry，比如 getTable()、size()、resize()；
// 这个方法的目的就是遍历引用队列，并把其中保存的 Entry 从 map 中移除掉；
// 从这里可以看到移除 Entry 的同时把 value 也一并置为 null 帮助 gc 清理元素，防御性编程。
```

##### 总结：

- （1）WeakHashMap 使用（数组 + 链表）存储结构；
- （2）WeakHashMap 中的 key 是弱引用，gc 的时候会被清除；
- （3）每次对 map 的操作都会剔除失效 key 对应的 Entry；
- （4）使用 String 作为 key 时，一定要使用 new String() 这样的方式声明 key，才会失效，其它的基本类型的包装类型是一样的；因为通过 new String() 声明的变量才是弱引用，比如使用"6"这种声明方式会一直存在于常量池中，不会被清理，所以"6"这个元素会一直在 map 里面，其它的元素随着 gc 都会被清理掉。
- （5）WeakHashMap 常用来作为缓存使用；

#### 虚引用

​	在介绍幽灵引用之前，要先介绍 Java 提供的对象终止化机制（finalization）。在 Object 类里面有个 finalize 方法，其设计的初衷是在一个对象被真正回收之前，可以用来执行一些清理的工作。因为 Java 并没有提供类似 C++ 的析构函数一样的机制，就通过 finalize 方法来实现。但是问题在于垃圾回收器的运行时间是不固定的，所以这些清理工作的实际运行时间也是不能预知的。幽灵引用（phantom reference）可以解决这个问题。在创建幽灵引用PhantomReference 的时候必须要指定一个引用队列。

​	当一个对象的 finalize 方法已经被调用了之后，这个对象的幽灵引用会被加入到队列中。通过检查该队列里面的内容就知道一个对象是不是已经准备要被回收了。

​	幽灵引用及其队列的使用情况并不多见，主要用来实现比较精细的内存使用控制，这对于移动设备来说是很有意义的。程序可以在确定一个对象要被回收之后，再申请内存创建新的对象。通过这种方式可以使得程序所消耗的内存维持在一个相对较低的数量。