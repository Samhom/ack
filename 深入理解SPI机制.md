## 深入理解SPI机制

### 1、概念

​	SPI ，全称为 Service Provider Interface，是一种服务发现机制。它通过在 ClassPath 路径下的 META-INF/services 文件夹查找文件，自动加载文件里所定义的类。这一机制为很多框架扩展提供了可能，比如在Dubbo、JDBC 中都使用到了 SPI 机制。

### 2、例子

```java
/**
 * 定义接口
 */
package com.iadfor.cloudinventory.common;
public interface SPIService {
    void execute();
}
/**
 * 定义实现类一
 */
package com.iadfor.cloudinventory.common;
public class SpiImpl1 implements SPIService {
    @Override
    public void execute() {
        System.out.println("SpiImpl1.execute()");
    }
}
/**
 * 定义实现类二
 */
package com.iadfor.cloudinventory.common;
public class SpiImpl2 implements SPIService {
    @Override
    public void execute() {
        System.out.println("SpiImpl1.execute()");
    }
}
// 1、classpath 路径下添加目录结构：META-INF/services
// 2、在该目录下添加文件，文件名：com.iadfor.cloudinventory.common.SPIService
// 3、文件内容为：
// com.iadfor.cloudinventory.common.SpiImpl1
// com.iadfor.cloudinventory.common.SpiImpl2
```

```java
/**
 * 测试类
 */
import sun.misc.Service;
import java.util.Iterator;
import java.util.ServiceLoader;

public class SpiClientTest {
    public static void main(String[] args) {
        Iterator<SPIService> providers = Service.providers(SPIService.class);
        while (providers.hasNext()) {
            SPIService ser = providers.next();
            ser.execute();
        }
        System.out.println("--------------------------------");

        ServiceLoader<SPIService> load = ServiceLoader.load(SPIService.class);
        Iterator<SPIService> iterator = load.iterator();
        while (iterator.hasNext()) {
            SPIService ser = iterator.next();
            ser.execute();
        }
    }
}

// 输出结果如下：
SpiImpl1.execute()
SpiImpl2.execute()
--------------------------------
SpiImpl1.execute()
SpiImpl2.execute()
```

源码不复杂，不再冗述。

### 3、JDBC中的应用

​	较早版本中，JDBC 获取数据库连接的过程，需要先设置数据库驱动的连接，再通过DriverManager.getConnection 获取一个 Connection。

```java
String url = "jdbc:mysql:///consult?serverTimezone=UTC";
String user = "root";
String password = "root";

Class.forName("com.mysql.jdbc.Driver");
Connection connection = DriverManager.getConnection(url, user, password);// 步骤1
```

​	在新版本中设置数据库驱动连接，“步骤1”就不再需要，那么**它是怎么分辨是哪种数据库的呢**？答案就在	SPI。

​	新版本 DriverManager 类中有一个静态代码块，在 JVM 启动时会加载这个类，加载这个类的时候会触发静态代码块的调用，通过 SPI 完成了数据库驱动连接初始化。

```java
static {
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}
```

loadInitialDrivers() 代码如下：

```java
	private static void loadInitialDrivers() {
        String drivers;
        try {
            drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
                public String run() {
                    return System.getProperty("jdbc.drivers");
                }
            });
        } catch (Exception ex) {
            drivers = null;
        }
        // If the driver is packaged as a Service Provider, load it.
        // Get all the drivers through the classloader
        // exposed as a java.sql.Driver.class service.
        // ServiceLoader.load() replaces the sun.misc.Providers()

        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
				// 很明显，它要加载Driver接口的服务类，Driver接口的包为:java.sql.Driver
                // 所以它要找的就是META-INF/services/java.sql.Driver文件
                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();
                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                	// Do nothing
                }
                return null;
            }
        });
    
    	println("DriverManager.initialize: jdbc.drivers = " + drivers);

        if (drivers == null || drivers.equals("")) {
            return;
        }
        String[] driversList = drivers.split(":");
        println("number of Drivers:" + driversList.length);
        for (String aDriver : driversList) {
            try {
                println("DriverManager.Initialize: loading " + aDriver);
                Class.forName(aDriver, true,
                        ClassLoader.getSystemClassLoader());
            } catch (Exception ex) {
                println("DriverManager.Initialize: load failed: " + ex);
            }
        }
    }
```

这个文件在 MySQL 的 jar 包中，文件内容为：`com.mysql.cj.jdbc.Driver`。

上边已经找到了 MySQL 中的 com.mysql.cj.jdbc.Driver 全限定类名，当调用 next 方法时，就会创建这个类的实例。它就完成了一件事，向 DriverManager 注册自身的实例：

```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    static {
        try {
            //注册
            //调用DriverManager类的注册方法
            //往registeredDrivers集合中加入实例
            java.sql.DriverManager.registerDriver(new Driver());
        } catch (SQLException E) {
            throw new RuntimeException("Can't register driver!");
        }
    }
    public Driver() throws SQLException {
        // Required for Class.forName().newInstance()
    }
}
```

在 DriverManager.getConnection() 方法就是创建连接的地方，它通过循环已注册的数据库驱动程序，调用其 connect 方法，获取连接并返回：

```java 
private static Connection getConnection(
        String url, java.util.Properties info, Class<?> caller) throws SQLException {   
    //registeredDrivers中就包含com.mysql.cj.jdbc.Driver实例
    for(DriverInfo aDriver : registeredDrivers) {
        if(isDriverAllowed(aDriver.driver, callerCL)) {
            try {
                //调用connect方法创建连接
                Connection con = aDriver.driver.connect(url, info);
                if (con != null) {
                    return (con);
                }
            }catch (SQLException ex) {
                if (reason == null) {
                    reason = ex;
                }
            }
        } else {
            println("skipping: " + aDriver.getClass().getName());
        }
    }
}
```

自定义实现类MyDriver，先在项目ClassPath下创建文件，文件内容为自定义驱动类。

我们的MyDriver实现类，继承自MySQL中的NonRegisteringDriver，还要实现java.sql.Driver接口。这样，在调用connect方法的时候，就会调用到此类，但实际创建的过程还靠MySQL完成：

```java
package com.viewscenes.netsupervisor.spi

public class MyDriver extends NonRegisteringDriver implements Driver{
    static {
        try {
            java.sql.DriverManager.registerDriver(new MyDriver());
        } catch (SQLException E) {
            throw new RuntimeException("Can't register driver!");
        }
    }
    public MyDriver()throws SQLException {}
    
    public Connection connect(String url, Properties info) throws SQLException {
        System.out.println("准备创建数据库连接.url:"+url);
        System.out.println("JDBC配置信息："+info);
        info.setProperty("user", "root");
        Connection connection =  super.connect(url, info);
        System.out.println("数据库连接创建完成!"+connection.toString());
        return connection;
    }
}
```

