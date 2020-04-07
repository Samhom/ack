## HashMap 的 hash() 函数

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

- ##### 异或运算符是用符号“^”表示的，其运算规律是：两个操作数的位中，相同则结果为0，不同则结果为1。

- ##### 与运算符用符号“&”表示，其使用规律如下：两个操作数中位都为1，结果才为1，否则结果为0。

#### 现在问题是：为什么要有 HashMap 的 hash() 方法，难道不能直接使用 KV 中 K 原有的 hash 值吗？

​	从上面的代码可以看到 key 的 hash 值的计算方法。key 的 hash 值高 16 位不变，低 16 位与高 16 位异或作为 key 的最终 hash 值。（h >>> 16，表示无符号右移 16 位，高位补 0，任何数跟 0 异或都是其本身，因此 key 的 hash 值高 16 位不变） 

![这里写图片描述](http://img.blog.csdn.net/20160408155045341)

##### 这么做的目的是什么？

其实这个与 HashMap 中 table 下标的计算有关

```java
n = table.length;
index = (n-1) & hash;
```

​	因为，table 的长度都是 2 的幂，因此 index 仅与 hash 值的低 n 位有关，hash 值的高位都被与操作置为 0 了。 
假设  table.length = 2 ^ 4 = 16。

![这里写图片描述](http://img.blog.csdn.net/20160408155102734)

​	由上图可以看到，只有 hash 值的低 4 位参与了运算。 

​	这样做很容易产生碰撞。设计者权衡了 speed, utility, and quality，将高 16 位与低 16 位异或来减少这种影响。设计者考虑到现在的 hashCode 分布的已经很不错了，而且当发生较大碰撞时也用树形存储降低了冲突。仅仅异或一下，既减少了系统的开销，也不会造成的因为高位没有参与下标的计算(table 长度比较小时)，从而引起的碰撞。

### HashMap#tableSizeFor()

```java 
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity); // 在这里
    }


// 或运算符用符号“|”表示，其运算规律如下：两个位只要有一个为1，那么结果就是1，否则就为0。
static final int tableSizeFor(int cap) {
    	// cap 做减 1 操作的目的是：防止cap已经是2的幂。如果cap已经是2的幂， 
        // 又没有执行这个减1操作，则执行完后面的几条无符号右移操作之后，返回的capacity将是这个cap的2倍。
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

```

​	由此可以看到，当在实例化 HashMap 实例时，如果给定了 initialCapacity，由于 HashMap 的 capacity 都是 2的幂，因此这个方法用于找到大于等于 initialCapacity 的最小的 2 的幂（initialCapacity 如果就是 2 的幂，则返回的还是这个数）。 

#### 下面看看这几个无符号右移操作： 

​	如果n这时为0了（经过了cap-1之后），则经过后面的几次无符号右移依然是0，最后返回的capacity是1（最后有个n+1的操作）。 这里只讨论n不等于0的情况。 

**第一次右移**

```
n |= n >>> 1;
```

​	由于n不等于0，则n的二进制表示中总会有一bit为1，这时考虑最高位的1。通过无符号右移1位，则将最高位的1右移了1位，再做或操作，使得n的二进制表示中与最高位的1紧邻的右边一位也为1，如000011xxxxxx。 

**第二次右移**

```
n |= n >>> 2;
```

​	注意，这个n已经经过了`n |= n >>> 1;` 操作。假设此时n为000011xxxxxx ，则n无符号右移两位，会将最高位两个连续的1右移两位，然后再与原来的n做或操作，这样n的二进制表示的高位中会有4个连续的1。如00001111xxxxxx 。 

**第三次右移**

```
n |= n >>> 4;
```

​	这次把已经有的高位中的连续的4个1，右移4位，再做或操作，这样n的二进制表示的高位中会有8个连续的1。如00001111 1111xxxxxx 。 

**以此类推** 

​	注意，容量最大也就是32bit的正数，因此最后`n |= n >>> 16;` ，最多也就32个1，但是这时已经大于了`MAXIMUM_CAPACITY` ，所以取值到`MAXIMUM_CAPACITY` 。 

##### 举一个例子说明下吧：

![这里写图片描述](http://img.blog.csdn.net/20160408183651111)

### 附：

#### HashMap 中 capacity、loadFactor、threshold、size 等概念的解释

- size 表示 HashMap 中存放KV的数量（为链表和树中的KV的总和）。
- capacity 译为容量。capacity就是指HashMap中桶的数量。默认值为16。
- loadFactor译为装载因子。装载因子用来衡量HashMap满的程度。loadFactor的默认值为0.75f。计算HashMap的实时装载因子的方法为：size/capacity，而不是占用桶的数量去除以capacity。
- threshold表示当HashMap的size大于threshold时会执行resize操作。 
  threshold=capacity*loadFactor