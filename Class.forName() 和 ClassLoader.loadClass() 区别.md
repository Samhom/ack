## Class.forName() 和 ClassLoader.loadClass() 区别

#### Class.forName() 和 ClassLoader 都可以对类进行加载。

### 区别：

- Class.forName 除了将类的 .class 文件加载到 jvm 中之外，还会对类进行解释，执行类中的 static 块。
- 而 ClassLoader.loadClass 只干一件事情，就是将 .class 文件加载到 jvm 中，不会执行 static 中的内容，只有在 newInstance 才会去执行 static 块。

​	Class.forName(name,initialize,loader) 带参数也可控制是否加载 static 块。并且只有调用了 newInstance() 方法采用调用构造函数，创建类的对象。

​	ClassLoader 就是遵循双亲委派模型最终调用启动类加载器的类加载器，实现的功能是“通过一个类的全限定名来获取描述此类的二进制字节流”，获取到二进制流后放到 JVM 中。Class.forName() 方法实际上也是调用的CLassLoader 来实现的。

### Java 类装载过程



![img](https:////upload-images.jianshu.io/upload_images/8576307-dfa6742d52a6c3de.png?imageMogr2/auto-orient/strip|imageView2/2/w/1008/format/webp)

- **装载**：通过类的全限定名（com.codetop.***.User）获取二进制字节流，将二进制字节流转换成方法区中的运行时数据结构，在内存中生成 Java.lang.class 对象；

- **链接**：执行下面的校验、准备和解析步骤，其中解析步骤是可以选择的；

  　　**校验**：检查导入类或接口的二进制数据的正确性；（文件格式验证，元数据验证，字节码验证，符号引用验证）

  　　**准备**：给类的静态变量分配并初始化存储空间；

  　　**解析**：将常量池中的符号引用转成直接引用；

- **初始化**：激活类的静态变量的初始化 Java 代码和静态 Java 代码块，并初始化程序员设置的变量值。

​	Class.forName(className)方法，内部实际调用的方法是  Class.forName(className,true,classloader);

第 2 个 boolean 参数表示类是否需要初始化， Class.forName(className) 默认是需要初始化。一旦初始化，就会触发目标对象的 static 块代码执行，static 参数也也会被再次初始化。

​	ClassLoader.loadClass(className)方法，内部实际调用的方法是 ClassLoader.loadClass(className,false);

第 2 个 boolean 参数，表示目标对象是否进行链接，false 表示不进行链接，由上面介绍可以，不进行链接意味着不进行包括初始化等一些列步骤，那么静态块和静态对象就不会得到执行。

#### JDBC Driver 使用 Class.forName(classname) 才能在反射回去类的时候执行 static 块

```java
static {
  try {
	java.sql.DriverManager.registerDriver(new Driver());
  } catch (SQLException E) {
	throw new RuntimeException("Can't register driver!");
  }
}
```

