### Arrays.sort 实现原理和 Collections 实现原理

#### Arrays.sort 实现原理：

​	Arrays.sort 底层通过 DualPivotQuicksort.sort (双轴快速排序) 实现，即快速排序。与归并排序比较适合大规模的数据排序。

​	数组小于 47 时 使用插入排序，小于 286 时使用双轴快排（切片-整合，改进版快排）。大于 286 时评估数组的无序程度，大的话使用改进版归并排序，小的话使双轴快排。

#### Collections 实现原理：

```java
public static <T extends Comparable<? super T>> void sort(List<T> list) { // T 需要实现 Comparable
    list.sort(null);
}
public static <T> void sort(List<T> list, Comparator<? super T> c) { // 需要自定义比较 工具类
    list.sort(c);
}
// 底层通过 List 的方法 default void sort(Comparator<? super E> c) {...} 实现，
// 而 List 的 该方法通过 Arrays 的 public static <T> void sort(T[] a, Comparator<? super T> c) {...} 实现。
public static <T> void sort(T[] a, Comparator<? super T> c) {
	if (c == null) {
        sort(a);
    } else {
        if (LegacyMergeSort.userRequested)
        	// 使用归并排序
            legacyMergeSort(a, c);
        else
        	// 
            TimSort.sort(a, 0, a.length, c, null, 0, 0);
    }
}
```

#### foreach 和 while 的区别(编译之后)

##### foreach 其实是通过迭代器 Iterator 来实现的，编译后：

```java
ArrayList<String> list = new ArrayList();
Iterator var2 = list.iterator();
while(var2.hasNext()) {
	String item = (String)var2.next();
	System.out.println(item);
}
```

for循环再某些方面要更优一些如无限循环 while(true) for(;;)

```java
编译前              编译后
while (1)；        mov eax,1
test eax,eax
je foo+23h
jmp foo+18h

编译前              编译后
for (；；)；        jmp foo+23h

从编译后的结果我们可以看出for的指令少，而且没有判断，显然更快.
```

