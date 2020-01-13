HashMap 的并发问题

#### 一、多线程的 put 可能导致元素的丢失

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, I;
    // 初始化hash表
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 通过hash值计算在hash表中的位置，并将这个位置上的元素赋值给p，如果是空的则new一个新的node放在这个位置上
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // hash表的当前index已经存在元素，向这个元素后追加链表
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                // 新建节点并追加到链表
                if ((e = p.next) == null) { // #1
                    p.next = newNode(hash, key, value, null); // #2
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
	......
	......
}
```

​	Thread1 和 Thread2 同时执行 put 操作，且两个 key 的 hashcode 相同且对应的在数组下标处已有数据（即链表上有1个或多个数据的时候），正常情况下把这两个线程的操作的 key 依次追加到该链表末尾，但由于是并发执行，且同时执行了 #1 处代码片段，即 (e = p.next) == null 判断都为 true，然后 Thread1 先执行了 #2，接着 Thread2 执行了 #2。结果就是 Thread2 覆盖了 Thread1 的元素。

#### 二、put 和 get 并发时，可能导致 get 为 null

```java
// hash表
transient Node<K,V>[] table;
......
final Node<K,V>[] resize() {
	// 计算新hash表容量大小，begin
	......
	......
	// 计算新hash表容量大小，end
	//@SuppressWarnings({"rawtypes","unchecked”})
	Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; // #1
    table = newTab; // #2
    // rehash begin
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
    ......
	......
}
```

场景：线程 1 执行 put 时，因为元素个数超出 threshold 而导致 rehash，线程 2 此时执行 get，有可能导致这个问题。在代码 #1 位置，用新计算的容量 new 了一个新的 hash 表，#2 将新创建的空 hash 表赋值给实例变量 table。注意此时实例变量 table 是空的。那么，如果此时另一个线程执行 get 时，就会 get 出 null。

#### 三、size 值会不准确

```java
transient int size; //（不参与序列化）在各个线程中的 size 副本不会及时同步，在多个线程操作的时候，size将会被覆盖。
```

