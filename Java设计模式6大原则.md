## Java设计模式6大原则

### 1、单一职责原则

​	就一个类而言，应该仅有一个引起它变化的原因。通俗的讲就是我们不要让一个承担过多的职责，如果一个类承担的职责过多，就等于把这些职责耦合在一起，一个职责的变化可能会削弱或者抑制这个类完成其他职责的能力。这种耦合会导致脆弱的设计，当变化发生时，设计会遭受到破坏。

### 2、开放封闭原则

​	类，模块，函数等应该是可以扩展的，但是不可以修改。

​	开放封闭有两个含义：一个是对于扩展是开放，另一个是对于修改是封闭的。

​	需求肯定是变化的，但是有新需求，我们就要把类重新改一遍，这显然是令人头痛的，所以我们设计程序时，面对需求的改变要尽可能得保证相对稳定，尽量通过扩展的方式来实现变化，而不是通过修改原有的代码来实现。

​	用开发封闭原则解决就是增加一个抽象的功能类，让添加，删除和查询作为这个抽象功能类的子类。这样如果我们再新增功能，你就会发现自己无须修改原有的类，只需要添加一个功能类的子类实现功能类的方法就可以了。

### 3、里式替换原则

​	所有引用基类(父类)的地方必须能透明的使用其子类的对象。

​	程序中尽量使用基类类型来对对象进行定义，而在运行时在确定其子类类型，用子类对象来替换父类对象。

​	子类的所有方法必须在父类中声明，或子类必须实现父类中声明的所有方法。根据里式替换原则，为了保证系统的扩展性，在程序中通常使用父类来定义。如果一个方法只存于子类中，在父类中不提供相应的声明，则无法在以父类定义的对象中使用该方法。

​	我们运用里式替换原则时，尽量把父类设计为抽象类或接口，让子类继承父类或实现父接口，并实现在父类中声明的方法、运行时，子类实例替换父类实例，我们可以很方便的扩展系统功能，同时无序修改原有子类的代码；增加新的功能可以通过增加一个新的子类来实现。里式替换原则是开放封闭原则的具体实现手段之一。

代码示例：

```java
abstract class Frits {
    abstract void Color();
    abstract void Flavour();
}
 
// color 颜色
// flavour 气味
// 苹果类
class Apple extends Frits {
    public void Color() {
        System.out.println("红色");
    }
 
    public void Flavour() {
        System.out.println("甜");
    }
}
 
//Banana 香蕉类
//Stroe 特有储藏方法
class Banana extends Frits {
    //特有的方法
    public void Store(){
        System.out.println("储藏须知");
    }
 
    @Override
    public void Color() {
        System.out.println("黄色");
    }
 
    @Override
    public void Flavour() {
        System.out.println("香甜");
        Store();
    }
}
 
//Pack -水果包装类
//describe -水果描述
//getMessage -返回具体信息
class Pack {
    private Frits frits;
 
    public void setFrits(Frits frits) {
        this.frits = frits;
    }
 
    void getMessage() {
        frits.Color();
        frits.Flavour();
    }
}
 
 
//buyer 买家-/具体场景
class Buyer {
    public static void main(String[] args) {
        Pack pack = new Pack();
 
        pack.setFrits(new Apple());
        pack.getMessage();
        pack.setFrits(new Banana());
        pack.getMessage();
    }
}
```

##### 里式替换原则通俗来说，子类可以扩展父类的功能，但不能改变父类原有的功能：

- 1.子类可以实现父类的抽象，但是不能覆盖父类的非抽象方法
- 2.子类中可以增加自己特有的方法。
- 3.当子类的方法重载父类的方法时，方法的前置条件要比父类方法的输入更宽松。
- 4.当子类的方法实现父类的抽象方法时，方法的后置条件要比父类更严格。

### 4、依赖倒置原则

- 在Java中，抽象指接口或者抽象类，两者都是不能直接被实例化；
- 细节就是实现类，实现接口或者继承抽象类而产生的就是细节，也就是可以加上一个关键字 new 产生的对象。
- 高层模块就是调用端，低层模块就是具体实现类。

​	依赖倒置原则在 java 中的表现就是，模块间的依赖通过抽象发生，实现类之间不发生直接依赖关系，其依赖关系就是通过接口或者抽象类产生的。如果类与类直接依赖细节，那么就会直接耦合。如此一来，就会同时修改依赖者代码，这样限制了可扩展性。

代码示例：

```java
class Cat {
    void Cry() {
        System.out.println("喵喵");
    }
}
 
class Dog {
    void Cry() {
        System.out.println("旺旺");
    }
}
 
 
class Animal {
    void Cry(Cat cat, Dog dog) {
        cat.Cry();
        dog.Cry();
    }
}
 
class Test{
    public static void main(String[] args) {
        new Animal().Cry(new Cat(),new Dog());
    }
}
// 上面这个代码看起来没什么问题，可是如果我们还有别的动物子类时，就又要去更改 Animal类，将其对象作为参数传入Cry方法。而依赖倒置原则 模块间的依赖通过抽象发生，实现类之间不发生直接的依赖关系。而我们上面这个Demo已经违背了这个原则，下面我们对它进行修改。
```

```java
interface Dongwu{
    void Cry();
}
 
class Cat implements Dongwu{
    public void Cry() {
        System.out.println("喵喵");
    }
}
 
class Dog implements Dongwu{
    public void Cry() {
        System.out.println("旺旺");
    }
}
 
 
class Animal {
    Dongwu dongwu;
 
    public void setDongwu(Dongwu dongwu) {
        this.dongwu = dongwu;
    }
 
    void Cry() {
       dongwu.Cry();
    }
}
 
class Test{
    public static void main(String[] args) {
        Animal animal=new Animal();
        
        animal.setDongwu(new Dog());
        animal.Cry();
        animal.setDongwu(new Cat());
        animal.Cry();
    }
}
// 我们增加了一个接口，里面有一个动物的叫声方法，然后让动物类们实现这个接口，然后通过set方法传递依赖对象，这样，如果我们再新增别的动物类，只需要实现相应的接口，而无需再更改我们的管理类 Animal。
```

### 五、迪米特原则

​	一个软件实体应当少的与其他实体发生相互作用。

​	迪米特原则要求我们在设计系统是，应该尽量减少对象之间的交互。如果两个对象之间不必彼此直接通向，那么这两个对象就不应当发生任何直接的相互作用。如果其中的一个对象需要调用另一个对象的某一个方法，则可以通过第三者转发这个调用。简言之，就是通过引入一个合理的第三者来降低现有对象之间的耦合度。

​	举例：就像租房子一样，我叫老王，准备租房子，然后找中介，中介和房东谈价格，我们和中介谈价格，如果我们想自己和房东联系，这种事情肯定是要通过中介介绍了，或者别的途径。

- 在类的划分上，应当尽量创建松耦合的类。类之间的耦合越低，就越有利于复用。一个处在松耦合中的类一旦被修改，则不会对关联的类造成太大波及。

- 在类的结构上，每一个类都应当尽量降低其成员变量和成员函数的访问权限。

- 在对其他类的引用上，一个对象对其他对象的引用应当降到最低。

### 六、接口隔离原则

​	一个类对另一个类的依赖应该建立在最小的接口上。

​	建立单一接口，不要建立庞大臃肿的接口：尽量细化接口，接口中的方法尽量少。也就是说，我们要为各个类建立专用的接口，而不要试图建立一个一个很庞大的接口供所有依赖他的类调用。

- 接口尽量小，但是要有限度。对接口进行细化可以提高程序设计的灵活性；但是如果过小，则会造成接口数量过多，使设计复杂化。所以，一定要适度。

- 为依赖接口的类定制服务，只暴露给调用的类他需要的方法，他不需要的方法则隐藏起来，只有专注的为一个模块提供定制服务，才能建立最小的依赖关系。

- 为提高内聚，减少对外交互。接口方法尽量少用public 修饰。接口是对外的承诺，承诺越少对系统的开发越有利，变更的风险也越少。