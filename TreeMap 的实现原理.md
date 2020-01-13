## TreeMap 的实现原理

可直接参考：https://www.jianshu.com/p/fc5e16b5c674

​	TreeMap 基于红黑树实现的，TreeMap 是有序的。同时红黑树更是一颗自平衡的排序二叉树。因为设计到 key 之间的比较进行排序，所以 key 不允许为 null。

​	TreeMap 的基本操作 containsKey、get、put 和 remove 的时间复杂度是 log(n) 。

​	另外，TreeMap 是非同步的。 它的 iterator 方法返回的迭代器是 fail-fast 的。	

首先分析一下红黑树的特性：

- 每个节点都只能是红色或者黑色
- 根节点是黑色
- 每个叶节点（NIL节点，空节点）是黑色的
- 如果一个结点是红的，则它两个子节点都是黑的。也就是说在一条路径上不能出现相邻的两个红色结点
- 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点

​	这些约束强制了红黑树的关键性质: 从根到叶子的最长的可能路径不多于最短的可能路径的两倍长。因为操作比如插入、删除和查找某个值的最坏情况时间都要求与树的高度成比例，这个在高度上的理论上限允许红黑树在最坏情况下都是高效的，而不同于普通的二叉查找树。所以红黑树它是复杂而高效的，其检索效率O(log n)。

### 主要函数解析

#### Entry 树的节点类:

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    // 左孩子节点
    Entry<K,V> left = null;
    // 右孩子节点
    Entry<K,V> right = null;
    // 父节点
    Entry<K,V> parent;
    // 红黑树用来表示节点颜色的属性，默认为黑色
    boolean color = BLACK;

    /**
     * 用key，value和父节点构造一个Entry，默认为黑色
     */
    Entry(K key, V value, Entry<K,V> parent) {
        this.key = key;
        this.value = value;
        this.parent = parent;
    }
    ...
    // 重写了 equals 和 hashCode 方法，利于比较是否相等
    public boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>)o;

        return valEquals( key,e.getKey()) && valEquals( value,e.getValue());
    }

    public int hashCode() {
        int keyHash = (key ==null ? 0 : key.hashCode());
        int valueHash = (value ==null ? 0 : value.hashCode());
        return keyHash ^ valueHash;
    }
    ...
}
```

#### 1、put

##### 一、将一个节点添加到红黑树中，通常需要下面几个步骤：

- a.将红黑树当成一颗二叉查找树，将节点插入。因为红黑树本身就是一个二叉查找树。
- b.将新插入的节点设置为红色
  		为什么新插入的节点一定要是红色的，因为新插入节点为红色，不会违背红黑规则第（5）条，少违背一条就少处理一种情况。
- c.通过旋转和着色，使它恢复平衡，重新变成一颗符合规则的红黑树。

##### 二、对比下红黑树的规则和新插入节点后的情况，看下新插入节点会违背哪些规则：

- 节点是红色或黑色：NO
- 根节点是黑色：NO
- 每个叶节点（NIL节点，空节点）是黑色的：NO
- 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点)：YES
  这一点是有可能违背的，我们将新插入的节点都设置成红色，如果其父节点也是红色的话，那就产生冲突了。
- 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点：NO

##### 三、添加新节点的过程有哪几种情况：

- 新插入节点为根节点。这种情况直接将新插入节点设置为根节点即可，无需进行后续的旋转和着色处理。
- 新插入节点的父节点是黑色。这种情况直接将新节点插入即可，不会违背规则（4）。
- 新插入节点的父节点是红色。这种情况会违背规则（4），而这种情况又分为了以下三种：
  aaaa.  新插入节点N的父节点P和叔叔节点U都是红色。方法是：将祖父节点G设置为红色，父节点P和叔叔节点U设置为黑色，这时候就看似平衡了。但是，如果祖父节点G的父节点也是红色，这时候又违背规则（4）了，怎么办？
  方法是：将GPUN这一组看成一个新的节点，按照前面的方案递归；又但是根节点为红就违反规则（2）了，怎么办，方法是直接将根节点设置为黑色（两个连续黑色是没问题的）。
  bbbb.  新插入节点N的父节点P是红色，叔叔节点U是黑色或者缺少，且新节点N是P的右孩子。方法是：左旋父节点P。左旋后N和P角色互换，但是P和N还是连续的两个红色节点，还没有平衡，怎么办，看第三种情况。
  cccc.  新插入节点N的父节点P是红色，叔叔节点U是黑色或者缺少，且新节点N是P的左孩子。方法是：右旋祖父节点G，然后将P设置为黑色，G设置为红色，达到平衡。此时父节点P是黑色，所以不用担心P的父节点是红色。
  当然上面说的三种情况都是基于一个前提：新插入节点N的父节点P是祖父节点G的左孩子，如果P是G的右孩子又是什么情况呢？其实情况和上面是相似的，只需要调整左旋还是右旋。

##### TreeMap 实现

```java
public V put(K key, V value) {
    // 根节点
    Entry<K,V> t = root;
    // 如果根节点为空，则直接创建一个根节点，返回
    if (t == null) {
        root = new Entry<K,V>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    // 记录比较结果
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    // 当前使用的比较器
    Comparator<? super K> cpr = comparator ;
    // 如果比较器不为空，就是用指定的比较器来维护TreeMap的元素顺序
    if (cpr != null) {
         // do while循环，查找key要插入的位置（也就是新节点的父节点是谁）
        do {
            // 记录上次循环的节点t
            parent = t;
            // 比较当前节点的key和新插入的key的大小
            cmp = cpr.compare(key, t. key);
             // 新插入的key小的话，则以当前节点的左孩子节点为新的比较节点
            if (cmp < 0)
                t = t. left;
            // 新插入的key大的话，则以当前节点的右孩子节点为新的比较节点
            else if (cmp > 0)
                t = t. right;
            else
          // 如果当前节点的key和新插入的key想的的话，则覆盖map的value，返回
                return t.setValue(value);
        // 只有当t为null，也就是没有要比较节点的时候，代表已经找到新节点要插入的位置
        } while (t != null);
    }
    else {
        // 如果比较器为空，则使用key作为比较器进行比较
        // 这里要求key不能为空，并且必须实现Comparable接口
        if (key == null)
            throw new NullPointerException();
        Comparable<? super K> k = (Comparable<? super K>) key;
        // 和上面一样，喜欢查找新节点要插入的位置
        do {
            parent = t;
            cmp = k.compareTo(t. key);
            if (cmp < 0)
                t = t. left;
            else if (cmp > 0)
                t = t. right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    // 找到新节点的父节点后，创建节点对象
    Entry<K,V> e = new Entry<K,V>(key, value, parent);
    // 如果新节点key的值小于父节点key的值，则插在父节点的左侧
    if (cmp < 0)
        parent. left = e;
    // 如果新节点key的值大于父节点key的值，则插在父节点的右侧
    else
        parent. right = e;
    // 插入新的节点后，为了保持红黑树平衡，对红黑树进行调整
    fixAfterInsertion(e);
    // map元素个数+1
    size++;
    modCount++;
    return null;
}

private static <K, V> void setColor(TreeMap.Entry<K, V> p, boolean c) {
    if (p != null)
        p.color = c;
}

private static <K, V> TreeMap.Entry<K, V> parentOf(TreeMap.Entry<K, V> p) {
    return (p == null ? null : p.parent);
}

private static <K, V> TreeMap.Entry<K, V> leftOf(TreeMap.Entry<K, V> p) {
    return (p == null) ? null : p.left;
}

private static <K, V> TreeMap.Entry<K, V> rightOf(TreeMap.Entry<K, V> p) {
    return (p == null) ? null : p.right;
}

private static <K, V> boolean colorOf(TreeMap.Entry<K, V> p) {
    return (p == null ? BLACK : p.color);
}
```

```java
/**
 * 对红黑树的节点(x)进行左旋转
 *
 * 左旋示意图(对节点x进行左旋)：
 *      px                              px
 *     /                               /
 *    x                               y               
 *   /  \      --(左旋)--           / \                
 *  lx   y                          x  ry    
 *     /   \                       /  \
 *    ly   ry                     lx  ly 
 *
 */
private void rotateLeft(Entry<K, V> p) {
    if (p != null) {
        // 取得要选择节点p的右孩子
        Entry<K, V> r = p.right;
        // "p"和"r的左孩子"的相互指向...
        // 将"r的左孩子"设为"p的右孩子"
        p.right = r.left;
        // 如果r的左孩子非空，将"p"设为"r的左孩子的父亲"
        if (r.left != null)
            r.left.parent = p;

        // "p的父亲"和"r"的相互指向...
        // 将"p的父亲"设为"y的父亲"
        r.parent = p.parent;
        // 如果"p的父亲"是空节点，则将r设为根节点
        if (p.parent == null)
            root = r;
            // 如果p是它父节点的左孩子，则将r设为"p的父节点的左孩子"
        else if (p.parent.left == p)
            p.parent.left = r;
        else
            // 如果p是它父节点的左孩子，则将r设为"p的父节点的左孩子"
            p.parent.right = r;
        // "p"和"r"的相互指向...
        // 将"p"设为"r的左孩子"
        r.left = p;
        // 将"p的父节点"设为"r"
        p.parent = r;
    }
}
```

```java
/**
 * 对红黑树的节点进行右旋转
 *
 * 右旋示意图(对节点y进行右旋)：
 *            py                               py
 *           /                                /
 *          y                                x                 
 *         /  \      --(右旋)--            /  \                     
 *        x   ry                           lx   y 
 *       / \                                   / \                   
 *      lx  rx                                rx  ry
 *
 */
private void rotateRight(Entry<K, V> p) {
    if (p != null) {
        // 取得要选择节点p的左孩子
        Entry<K, V> l = p.left;
        // 将"l的右孩子"设为"p的左孩子"
        p.left = l.right;
        // 如果"l的右孩子"不为空的话，将"p"设为"l的右孩子的父亲"
        if (l.right != null) l.right.parent = p;
        // 将"p的父亲"设为"l的父亲"
        l.parent = p.parent;
        // 如果"p的父亲"是空节点，则将l设为根节点
        if (p.parent == null)
            root = l;
            // 如果p是它父节点的右孩子，则将l设为"p的父节点的右孩子"
        else if (p.parent.right == p)
            p.parent.right = l;
            //如果p是它父节点的左孩子，将l设为"p的父节点的左孩子"
        else p.parent.left = l;
        // 将"p"设为"l的右孩子"
        l.right = p;
        // 将"l"设为"p父节点"
        p.parent = l;
    }
}
```

```java
/**
 * 新增节点后对红黑树的调整方法
 */
private void fixAfterInsertion(Entry<K, V> x) {
    // 将新插入节点的颜色设置为红色
    x.color = RED;
    // while循环，保证新插入节点x不是根节点或者新插入节点x的父节点不是红色（这两种情况不需要调整）
    while (x != null && x != root && x.parent.color == RED) {
        // 如果新插入节点x的父节点是祖父节点的左孩子
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            // 取得新插入节点x的叔叔节点
            Entry<K, V> y = rightOf(parentOf(parentOf(x)));
            // 如果新插入x的叔叔节点是红色-------------------①
            if (colorOf(y) == RED) {
                // 将x的父节点设置为黑色
                setColor(parentOf(x), BLACK);
                // 将x的叔叔节点设置为黑色
                setColor(y, BLACK);
                // 将x的祖父节点设置为红色
                setColor(parentOf(parentOf(x)), RED);
                // 将x指向祖父节点，如果x的祖父节点的父节点是红色，按照上面的步奏继续循环
                x = parentOf(parentOf(x));
            } else {
                // 如果新插入x的叔叔节点是黑色或缺少，且x的父节点是祖父节点的右孩子-------------------②
                if (x == rightOf(parentOf(x))) {
                    // 左旋父节点
                    x = parentOf(x);
                    rotateLeft(x);
                }
                // 如果新插入x的叔叔节点是黑色或缺少，且x的父节点是祖父节点的左孩子-------------------③
                // 将x的父节点设置为黑色
                setColor(parentOf(x), BLACK);
                // 将x的祖父节点设置为红色
                setColor(parentOf(parentOf(x)), RED);
                // 右旋x的祖父节点
                rotateRight(parentOf(parentOf(x)));
            }
        } else { // 如果新插入节点x的父节点是祖父节点的右孩子，下面的步奏和上面的相似，只不过左旋右旋的区分，不在细讲
            Entry<K, V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    // 最后将根节点设置为黑色，不管当前是不是红色，反正根节点必须是黑色
    root.color = BLACK;
}
```

#### 2、remove

##### 相比添加，红黑树的删除显得更加复杂了。看下红黑树的删除需要哪几个步奏：

- a.将红黑树当成一颗二叉查找树，将节点删除
- b.通过旋转和着色，使它恢复平衡，重新变成一颗符合规则的红黑树

##### 删除节点的关键是：

- aa.如果删除的是红色节点，不会违背红黑树的规则
- bb.如果删除的是黑色节点，那么这个路径上就少了一个黑色节点，则违背了红黑树的规则

##### 来看下红黑树删除节点会有哪几种情况：

- a.被删除的节点没有孩子节点，即叶子节点。可直接删除。
- b.被删除的节点只有一个孩子节点，那么直接删除该节点，然后用它的孩子节点顶替它的位置。
- c.被删除的节点有两个孩子节点。这种情况二叉树的删除有一个技巧，就是查找到要删除的节点X，接着我们找到它左子树的最大元素M，或者它右子树的最小元素M，交换X和M的值，然后删除节点M。
  	此时M就最多只有一个子节点N(若是左子树则没有右子节点，若是右子树则没有左子节点)，若M没有孩子则进入(1)的情况，否则进入(2)的情况。



​	红黑树的删除遇到的主要问题就是被删除路径上的黑色节点减少，于是需要进行一系列旋转和着色，比如节点X（需要删除的节点）和M节点的值互换后，将M点删除，如果M是黑色，则红黑树路径上就少了一个黑色节点，这违背了第五条：从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点。

​	当然上面的情况是基于M是X右子树的最小元素，而M如果是X左子树的最大元素和上面的情况是相似的。

​	所以就需要在此基础上继续根据兄弟节点、兄弟节点的孩子节点及父节点的红黑色情况做进一步的颜色替换和左旋或右旋操作，来使整棵树重新达到平衡。

##### TreeMap 实现：

```java
public V remove(Object key) {
    // 根据key查找到对应的节点对象
    Entry<K,V> p = getEntry(key);
    if (p == null)
        return null;

    // 记录key对应的value，供返回使用
    V oldValue = p. value;
    // 删除节点
    deleteEntry(p);
    return oldValue;
}
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    // map容器的元素个数减一
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.
    // 如果被删除的节点p的左孩子和右孩子都不为空，则查找其替代节点-----------这里表示要删除的节点有两个孩子（3）
    if (p.left != null && p. right != null) {
        // 查找p的替代节点
        Entry<K,V> s = successor (p);
        p. key = s.key ;
        p. value = s.value ;
        // 将p指向替代节点，※※※※※※从此之后的p不再是原先要删除的节点p，而是替代者p（就是图解里面讲到的M） ※※※※※※
        p = s;
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    // replacement为替代节点p的继承者（就是图解里面讲到的N），p的左孩子存在则用p的左孩子替代，否则用p的右孩子
    Entry<K,V> replacement = (p. left != null ? p.left : p. right);

    if (replacement != null) { // 如果上面的if有两个孩子不通过--------------这里表示要删除的节点只有一个孩子（2）
        // Link replacement to parent
        // 将p的父节点拷贝给替代节点
        replacement. parent = p.parent ;
        // 如果替代节点p的父节点为空，也就是p为跟节点，则将replacement设置为根节点
        if (p.parent == null)
            root = replacement;
        // 如果替代节点p是其父节点的左孩子，则将replacement设置为其父节点的左孩子
        else if (p == p.parent. left)
            p. parent.left   = replacement;
        // 如果替代节点p是其父节点的左孩子，则将replacement设置为其父节点的右孩子
        else
            p. parent.right = replacement;

        // Null out links so they are OK to use by fixAfterDeletion.
        // 将替代节点p的left、right、parent的指针都指向空，即解除前后引用关系（相当于将p从树种摘除），使得gc可以回收
        p. left = p.right = p.parent = null;

        // Fix replacement
        // 如果替代节点p的颜色是黑色，则需要调整红黑树以保持其平衡
        if (p.color == BLACK)
            fixAfterDeletion(replacement);
    } else if (p.parent == null) { // return if we are the only node.
        // 如果要替代节点p没有父节点，代表p为根节点，直接删除即可
        root = null;
    } else { //  No children. Use self as phantom replacement and unlink.
        // 判断进入这里说明替代节点p没有孩子--------------这里表示没有孩子则直接删除（1）
        // 如果p的颜色是黑色，则调整红黑树
        if (p.color == BLACK)
            fixAfterDeletion(p);
        // 下面删除替代节点p
        if (p.parent != null) {
            // 解除p的父节点对p的引用
            if (p == p.parent .left)
                p. parent.left = null;
            else if (p == p.parent. right)
                p. parent.right = null;
            // 解除p对p父节点的引用
            p. parent = null;
        }
    }
}
```

```java
/**
 * 查找要删除节点的替代节点
 */
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    if (t == null)
        return null;
    // 查找右子树的最左孩子
    else if (t.right != null) {
        Entry<K,V> p = t. right;
        while (p.left != null)
            p = p. left;
        return p;
    } else { // 查找左子树的最右孩子
        Entry<K,V> p = t. parent;
        Entry<K,V> ch = t;
        while (p != null && ch == p. right) {
            ch = p;
            p = p. parent;
        }
        return p;
    }
}

```

```java
/** From CLR */
private void fixAfterDeletion(Entry<K,V> x) {
    // while循环，保证要删除节点x不是跟节点，并且是黑色（根节点和红色不需要调整）
    while (x != root && colorOf (x) == BLACK) {
        // 如果要删除节点x是其父亲的左孩子
        if (x == leftOf( parentOf(x))) {
            // 取出要删除节点x的兄弟节点
            Entry<K,V> sib = rightOf(parentOf (x));

            // 如果删除节点x的兄弟节点是红色---------------------------①
            if (colorOf(sib) == RED) {
                // 将x的兄弟节点颜色设置为黑色
                setColor(sib, BLACK);
                // 将x的父节点颜色设置为红色
                setColor(parentOf (x), RED);
                // 左旋x的父节点
                rotateLeft( parentOf(x));
                // 将sib重新指向旋转后x的兄弟节点 ，进入else的步奏③
                sib = rightOf(parentOf (x));
            }

            // 如果x的兄弟节点的两个孩子都是黑色-------------------------③
            if (colorOf(leftOf(sib))  == BLACK &&
                colorOf(rightOf (sib)) == BLACK) {
                // 将兄弟节点的颜色设置为红色
                setColor(sib, RED);
                // 将x的父节点指向x，如果x的父节点是黑色，需要将x的父节点整天看做一个节点继续调整-------------------------②
                x = parentOf(x);
            } else {
                // 如果x的兄弟节点右孩子是黑色，左孩子是红色-------------------------④
                if (colorOf(rightOf(sib)) == BLACK) {
                    // 将x的兄弟节点的左孩子设置为黑色
                    setColor(leftOf (sib), BLACK);
                    // 将x的兄弟节点设置为红色
                    setColor(sib, RED);
                    // 右旋x的兄弟节点
                    rotateRight(sib);
                    // 将sib重新指向旋转后x的兄弟节点，进入步奏⑤
                    sib = rightOf(parentOf (x));
                }
                // 如果x的兄弟节点右孩子是红色-------------------------⑤
                setColor(sib, colorOf (parentOf(x)));
                // 将x的父节点设置为黑色
                setColor(parentOf (x), BLACK);
                // 将x的兄弟节点的右孩子设置为黑色
                setColor(rightOf (sib), BLACK);
                // 左旋x的父节点
                rotateLeft( parentOf(x));
                // 达到平衡，将x指向root，退出循环
                x = root;
            }
        } else { // symmetric // 如果要删除节点x是其父亲的右孩子，和上面情况一样，这里不再细讲
            Entry<K,V> sib = leftOf(parentOf (x));

            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf (x), RED);
                rotateRight( parentOf(x));
                sib = leftOf(parentOf (x));
            }

            if (colorOf(rightOf(sib)) == BLACK &&
                colorOf(leftOf (sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(leftOf(sib)) == BLACK) {
                    setColor(rightOf (sib), BLACK);
                    setColor(sib, RED);
                    rotateLeft(sib);
                    sib = leftOf(parentOf (x));
                }
                setColor(sib, colorOf (parentOf(x)));
                setColor(parentOf (x), BLACK);
                setColor(leftOf (sib), BLACK);
                rotateRight( parentOf(x));
                x = root;
            }
        }
    }

    setColor(x, BLACK);
}
```

#### 3、get

红黑树的查询，逻辑相对简单：

```java
public V get(Object key) {
    Entry<K, V> p = getEntry(key);
    return (p == null ? null : p.value);
}

final Entry<K, V> getEntry(Object key) {
    if (comparator != null)
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K, V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}

final Entry<K, V> getEntryUsingComparator(Object key) {
    K k = (K) key;
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        Entry<K, V> p = root;
        while (p != null) {
            int cmp = cpr.compare(k, p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
    }
    return null;
}
```

#### 4、Ite

##### 通过Iterator遍历key-value：

```
Iterator iter = tmap.entrySet().iterator();
while (iter.hasNext()) {
    Map.Entry entry = (Map.Entry) iter.next();
    System.out.printf("next : %s - %s\n", entry.getKey(), entry.getValue());
}
```

##### 清空 TreeMap：

```
public void clear() {
    modCount++;
    size = 0;
    root = null;
}
```

##### TreeMap 的子 Map 函数：

```java
tmap.headMap("c")
tmap.headMap("c", true)
tmap.headMap("c", false)
tmap.tailMap("c")
tmap.tailMap("c", true)
tmap.tailMap("c", false)
tmap.subMap("a", "c")
tmap.subMap("a", true, "c", true)
tmap.subMap("a", true, "c", false)
tmap.subMap("a", false, "c", true)
tmap.subMap("a", false, "c", false)
tmap.navigableKeySet()
tmap.descendingKeySet()
tmap.firstKey()
tmap.firstEntry()
tMap.lastKey()
tMap.lastEntry()
tMap.floorKey("bbb")	// 获取“小于/等于bbb”的最大键值对
tMap.lowerKey("bbb")	// 获取“小于bbb”的最大键值对
tMap.ceilingKey("ccc")  // 获取“大于/等于bbb”的最小键值对
tMap.higherKey("ccc")   // 获取“大于bbb”的最小键值对
```