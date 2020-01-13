### cloneable 接口实现原理，浅拷贝 or 深拷贝

#### 原理

​	一个对象 a 的 clone 就是在堆上分配一个和 a 在堆上所占存储空间一样大的一块地方，然后把 a 的堆上内存的内容复制到这个新分配的内存空间上。

支持克隆的类需要实现 Cloneable 接口，这样才能通过 Object 类的 clone 方法：

```java
protected native Object clone() throws CloneNotSupportedException;
```

否认抛 CloneNotSupportedException 异常。

#### 浅克隆

即很表层的克隆，如果我们要克隆对象，只克隆它自身以及它所包含的所有对象的引用地址。

#### 深克隆

​	克隆除自身对象以外的所有对象，包括自身所包含的所有对象实例及所有的基本数据类型，无论是浅克隆还是深克隆，都会进行原值克隆，毕竟它们都不是对象，不是存储在堆中的。

​	Object 的 clone() 方法，提供的是一种浅克隆的机制，如果想要实现对对象的深克隆，在不引入第三方 jar 包的情况下，可以使用两种办法：

- 先对对象进行序列化，紧接着马上反序列化出；
- 先调用 super.clone() 方法克隆出一个新对象来，然后在子类的 clone() 方法中手动给克隆出来的非基本数据类型（引用类型）赋值，比如 ArrayList 的 clone() 方法；

​	clone 对 string 是深克隆，虽然 clone String 时拷贝其地址引用。但是在修改时，它会从字符串池中重新生成一个新的字符串，原有的对象保持不变。

​	深拷贝时，如果一个对象中有多个属性是其他的对象引用或者引用较深或可能依赖第三方的 jar 包，这时如果通过手动一个个的调用 clone 方法继续拧拷贝时，工作量会比较大，而且可靠性也不能保证。

​	如果是这种情况则建议采用如下的做法，使用序列化 clone：

```java
public class CloneUtils {
    public static <T extends Serializable> T clone(T obj){
        T cloneObj = null;
        try {
            // 写入字节流
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            ObjectOutputStream obs = new ObjectOutputStream(out);
            obs.writeObject(obj);
            obs.close();
            // 分配内存，写入原始对象，生成新对象
            ByteArrayInputStream ios = new ByteArrayInputStream(out.toByteArray());
            ObjectInputStream ois = new ObjectInputStream(ios);
            // 返回生成的新对象
            cloneObj = (T) ois.readObject();
            ois.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return cloneObj;
    }
}
```

#### object clone的实际用途：

​	精心设计一个浅克隆对象被程序缓存，作为功能模块模板；每次有用户调用这个模块则将可变部分替换成用户需要的信息即可。	

​	给同组的用户发送邮件，邮件内容相同（不可变）发送的用户不同（可变）。