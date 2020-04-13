## java 序列化&反序列化

### JDK 实现序列化方式

```java
public class Demo4 {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        BF bf = new BF("haha", 128);
        // 序列化到文件中
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("D://tmpla"));
        oos.writeObject(bf);
        oos.close();
        // 从文件中反序列化出来
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(new File("D://tmpla")));
        BF newO = (BF) ois.readObject();
        System.out.println(newO);
    }

    protected static class BF implements Serializable{
        private String name;
        private Integer age;
        public BF(String name, Integer age) {
            this.name = name;
            this.age = age;
        }
        public String getName() {
            return name;
        }
        public Integer getAge() {
            return age;
        }

        @Override
        public String toString() {
            return "BF{name='" + name + '\'' +
                    ", age=" + age +
                    '}';
        }
    }
}
```

### Thrift 序列化方式

// TODO

### ProtoBuf 序列化方式

// TODO

### kyro 序列化方式

// TODO