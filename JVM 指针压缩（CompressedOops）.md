## JVM 指针压缩（CompressedOops）

​	对于32位机器，进程能使用的最大内存是4G。如果进程需要使用更多的内存，需要使用64位机器。

​	对于Java进程，在oop只有32位时，只能引用4G内存。因此，如果需要使用更大的堆内存，需要部署64位JVM。这样，oop为64位，可引用的堆内存就更大了。

​	**注：oop（ordinary object pointer），即普通对象指针，是JVM中用于代表引用对象的句柄。**

​	在堆中，32位的对象引用占4个字节，而64位的对象引用占8个字节。也就是说，64位的对象引用大小是32位的2倍。

#### 64位JVM在支持更大堆的同时，由于对象引用变大却带来了性能问题：

- 增加了GC开销：

  64位对象引用需要占用更多的堆空间，留给其他数据的空间将会减少，从而加快了GC的发生，更频繁的进行GC。

- 降低CPU缓存命中率：

  64位对象引用增大了，CPU能缓存的oop将会更少，从而降低了CPU缓存的效率。

​	为了能够保持32位的性能，oop必须保留32位。那么，如何用32位oop来引用更大的堆内存呢？答案是压缩指针（CompressedOops）。