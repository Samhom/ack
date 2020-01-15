### HashTable 和 HashMap 的区别

#### 一、不同点：

##### 1、Hashtable 是不允许键或值为 null 的，HashMap 的键值则都可以为 null。Hashtable 是不允许键或值为 null 的，HashMap 的键值则都可以为 null。

hashMap 求hash值代码如，key为null，则hash值为0：

```java 
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

HashTable put方法如下：

```java
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) { // value 不能为 null
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    int hash = key.hashCode(); // key 不能为 null
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }
    addEntry(hash, key, value, index);
    return null;
}
```

#### 2、实现方式不同

Hashtable 继承了 Dictionary类，而 HashMap 继承的是 AbstractMap 类。

#### 3、初始化容量不同

HashMap 的初始容量为：16，Hashtable 初始容量为：11，两者的负载因子默认都是：0.75。

#### 4、扩容机制不同

当现有容量大于总容量 * 负载因子时，HashMap 扩容规则为当前容量翻倍，Hashtable 扩容规则为当前容量翻倍 + 1。

#### 5、迭代器不同

HashMap 中的 Iterator 迭代器是 fail-fast 的，而 Hashtable 的 Enumerator 不是 fail-fast 的。

#### 二、为啥 Hashtable 是不允许 KEY 和 VALUE 为 null？

​	这是因为 Hashtable 使用的是安全失败机制（fail-safe），这种机制会使你此次读到的数据不一定是最新的数据。如果你使用 null 值，就会使得其无法判断对应的 key 是不存在还是为空，因为你无法再调用一次 contain(key）来对 key 是否存在进行判断，ConcurrentHashMap 同理。