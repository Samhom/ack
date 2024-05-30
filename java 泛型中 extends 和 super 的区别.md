# java 泛型中 extends 和 super 的区别

## 关键字说明

### ? 通配符类型

- <? extends T> 表示类型的上界，表示参数化类型的可能是T 或是 T的子类
- <? super T> 表示类型下界（Java Core中叫超类型限定），表示参数化类型是此类型的超类型（父类型），直至Object

### 可以这么理解：

- List<? extends T> 是说 这个list放的是T或者T的子类型的对象，但是不能确定具体是什么类型，所以可以get（），不能add（）（可以add null值）
- List<? super T> 是说这个list放的是至少是T类型的对象，所以我可以add T或者T的子类型，但是get得到的类型不确定，所以不能get

## extends 示例

```scala
static class Food{}
static class Fruit extends Food{}
static class Apple extends Fruit{}
static class RedApple extends Apple{}

List<? extends Fruit> flist = new ArrayList<Apple>();
// complie error:
// flist.add(new Apple());
// flist.add(new Fruit());
// flist.add(new Object());
flist.add(null); // only work for null 
```

`List<? extends Frut>` 表示 “具有任何从 Fruit 继承类型的列表”，编译器无法确定 List 所持有的类型，所以无法安全的向其中添加对象。可以添加 null,因为 null 可以表示任何类型。所以 List 的 add 方法不能添加任何有意义的元素，但是可以接受现有的子类型List`<Apple>` 赋值。

```csharp
Fruit fruit = flist.get(0);
Apple apple = (Apple)flist.get(0);
```

由于，其中放置是从Fruit中继承的类型，所以可以安全地取出Fruit类型。

```cpp
flist.contains(new Fruit());
flist.contains(new Apple());
```

在使用Collection中的contains 方法时，接受Object 参数类型，可以不涉及任何通配符，编译器也允许这么调用。

## super 示例

```csharp
List<? super Fruit> flist = new ArrayList<Fruit>();
flist.add(new Fruit());
flist.add(new Apple());
flist.add(new RedApple());

// compile error:
List<? super Fruit> flist = new ArrayList<Apple>();
```

`List<? super Fruit>` 表示“具有任何Fruit超类型的列表”，列表的类型至少是一个 Fruit 类型，因此可以安全的向其中添加Fruit 及其子类型。由于List<? super Fruit>中的类型可能是任何Fruit 的超类型，无法赋值为Fruit的子类型Apple的`List<Apple>`.

```csharp
// compile error:
Fruit item = flist.get(0);
```

因为，`List<? super Fruit>`中的类型可能是任何Fruit 的超类型，所以编译器无法确定get返回的对象类型是Fruit,还是Fruit的父类Food 或 Object.

## 小结

extends 可用于返回类型限定，不能用于参数类型限定。

super 可用于参数类型限定，不能用于返回类型限定。

带有super超类型限定的通配符可以向泛型对易用写入，带有extends子类型限定的通配符可以向泛型对象读取。
 super限定类型安全时方法参数用,extends限定类型安全时返回值用。

这里 ，PECS原则 如下：泛型读用？ extends T（T{T类型或者T的子类型}类型不确定，写入的时候编译器不知道具体的类型所以会报编译器错误）{上届}，写用？ super T（T{T父类型直至Object}，读出来的是object）{下届}，泛型就是为了类型安全。

如果要从集合中读取类型T的数据，并且不能写入，可以使用 ? extends 通配符；(Producer Extends)。

如果要从集合中写入类型T的数据，并且不需要读取，可以使用 ? super 通配符；(Consumer Super)。

如果既要存又要取，那么就不要使用任何通配符。