### LinkedHashMap 的应用

重点在于：插入有序和访问有序。

##### 原理简介

​	LinkedHashMap 拥有与 HashMap 相同的底层哈希表结构，即数组 + 单链表 + 红黑树，也拥有相同的扩容机制。LinkedHashMap 相比 HashMap 的拉链式存储结构，内部额外通过 Entry （extends HashMap.Node<K,V>） 维护了一个双向链表。HashMap 元素的遍历顺序不一定与元素的插入顺序相同，而 LinkedHashMap 则通过遍历双向链表来获取元素，所以遍历顺序在一定条件下等于插入顺序。LinkedHashMap 可以通过构造参数 accessOrder 来指定双向链表是否在元素被访问后改变其在双向链表中的位置。

#### 1、首先分析 put 函数，并没有重写 hashmap 的 put 函数：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table为空，则通过扩容来创建，后面在看扩容函数
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 根据key的hash值 与 数组长度进行取模来得到数组索引    
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 空链表，创建节点
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 不为空，则判断是否与当前节点一样，一样就进行覆盖
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            // 不存在重复节点，则判断是否属于树节点，如果属于树节点，则通过树的特性去添加节点
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 该链为链表
            for (int binCount = 0; ; ++binCount) {
                // 当链表遍历到尾节点时，则插入到最后 -> 尾插法
                if ((e = p.next) == null) {
                	========== 重写 1 ==========
                    p.next = newNode(hash, key, value, null);
                    // 检测是否该从链表变成树（注意：这里是先插入节点，没有增加binCount,所以判断条件是大于等于阈值-1）
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 满足则树形化
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // 存在相同的key, 则替换value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value；
            ========== 重写 2 ==========
            // 注意这里，这里是供子类LinkedHashMap实现    
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 注意细节：先加入节点，再加长度与阈值进行判断，是否需要扩容。
    if (++size > threshold)
        resize();
    ========== 重写 3 ==========
    // 注意这里，这里是供子类LinkedHashMap实现        
    afterNodeInsertion(evict);
    return null;
}
```

##### 重写 1：

每次插入都会调用 newNode 函数创建一个新节点，对于 LinkeHashMap 来说有重写该函数。

```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    // 调用父类创建节点， 没什么区别。
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    // 新加的方法    
    linkNodeLast(p);
    return p;
}
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    // 如果双向链表为空，则当前节点是第一个节点
    if (last == null)
        head = p;
    else {
        // 将新创建的节点添加至双向链表的尾部。
        p.before = last;
        last.after = p;
    }
}
```

##### 重写 2：

​	当存在相同 key 替换 value 后，会调用 afterNodeAccess 函数，这函数在 HashMap 中是没有任何实现的，主要是供子类 LinkeHashMap 来实现。当访问到双向链表存在的值时，如果开启访问有序的开关，则会将访问到的节点移至到双向链表的尾部。见下边代码片段分析：

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    // 如果accessOrder=true,即访问有序，且双向链表不止一个节点的时候，进行下面操作：
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        // 将p的后置指针置为null     
        p.after = null; 
        // 如果e的前置指针没有元素, 则直接将双向链表的头节点指向它。
        if (b == null)
            head = a;
        else
            // e的前置指针存在元素, 则将e的前置指针指向节点的后置指针指向其后置指针指向的的节点。
            b.after = a;
        // e的后置指针存在元素, 则将e的后置指针指向节点的前置指针指向e前置指针指向的节点    
        if (a != null)
            a.before = b;
        else
            // 否则将尾节点指向e的前置节点
            last = b;
        // 上面步骤主要是将e节点从链表中移除，然后添加到链表尾部    
        if (last == null)
            head = p;
        else {
            // 添加置链表尾部
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

另外get函数也会调用这个函数，所以从源码的角度去看问题很清晰：

```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    // 如果存在节点且开启了访问有序的开关，则会将当前节点移至双向链表尾部    
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

##### 重写 3：

​	当扩容完后，会调用 afterNodeInsertion 函数，同理这个函数也是供子类 LinkeHashMap 来实现的。该函数表示是否需要删除最年长的节点：

```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first；
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        // 获取头节点：头节点表示最近很久没有访问的元素
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
// 返回false, 所以LinkedHashMap不会有删除年长节点的行为，但其子类可以继承重写该函数。
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

##### forEach：

```java
public void forEach(BiConsumer<? super K, ? super V> action) {
    if (action == null)
        throw new NullPointerException();
    int mc = modCount;
    // 遍历的是双向链表。所以我们看到的就是插入的顺序
    for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after)
        action.accept(e.key, e.value);
    if (modCount != mc)
        throw new ConcurrentModificationException();
}
```

