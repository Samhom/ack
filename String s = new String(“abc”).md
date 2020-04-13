# String s = new String(“abc”)

### 字符串常量池

​	由于字符串在Java中被大量使用，为了避免每次都创建相同的字符串对象（这样就意味着占据更多内存），JVM对字符串对象做了一定的优化，有一块专门的区域来存储字符串常量，该区域就是字符串常量池。常量池(constant pool)指的是在编译期被确定，并被保存在已编译的.class文件中的一些数据。它包括了关于类、方法、接口等中的常量，也包括字符串常量。

​	在JDK1.6即以前，常量池位于JVM的方法区中，在JDK7即以后，常量池放在堆中。

```java
public class StringDemo {
	public static void main(String[] args) {
		String str1 = "abc";
		String str2 = "abc";
		String str3 = new String("abc");
        
		System.out.println(str1==str2); //第七行
		System.out.println(str2==str3);//第八行
	}
}
// 第7行输出true，第8行输出false
```

#### 执行过程：

执行String str1 = “abc”;时，JVM常量池中并没有"abc"这个字符串，此时会在常量池中创建这个对象，然后将引用赋给str1`。

执行String str2 = "abc";时，JVM在常量池中找到"abc"这个字符串了，所以直接将应用附给它。

执行String str3 = new String("abc");时，JVM会在堆中创建一个String对象，然后将传进去的常量池String对象中保存的char[]数组，赋给堆中的String。

### String s=new String(“abc”) 到底创建了几个对象？

​	如果在执行 String s = new String(“abc”) 这句话之前，常量池中并没有"abc"，那么在创建new String()时会先在常量池中创建字符串常量，然后通过这个字符串常量，在堆中创建一个新的字符串，但是这两个字符串对象（常量池中的"abc"和堆中的"abc"）底层保存字符的数组都是一个。

  如果在执行之前就有了"abc"这个字符串常量了（例如上面的代码），在执行String s=new String(“abc”)这句话时，也就只在堆中创建一个对象了。

### 验证常量池中的"abc"和堆中的"abc"底层保存字符的数组都是一个

​	采用new关键字在堆中创建的`"abc"`和常量池中`"abc"`虽然对象不是一个，但是它们两个对象底层指向的数组是一个，那我们就通过代码验证这个结论。

​	整体思路是这样的：我们通过反射，修改常量词中字符串对象底层数组的值，看堆中的String对象的值是否跟着改变：

```java
@Test
public void test() throws Exception {
	//在常量池中创建一个"abc"
	String str = "abc";
	//通过常量词中的"abc"在堆中创建一个String对象
	String str2 = new String("abc");
	
	//获取String类中的value字段
	Field field = String.class.getDeclaredField("value");
	//将字段设置为可访问的
	field.setAccessible(true);
	
	//获取str对象上的value属性的值
	char[] arr = (char[]) field.get(str);
	arr[2]='1';
        
	System.out.println(str);//输出：ab1
	System.out.println(str2);//输出：ab1
}
```

