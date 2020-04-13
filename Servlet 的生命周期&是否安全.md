# Servlet 的生命周期&是否安全

### 一、Servlet_生命周期

首先加载 servlet 的 class，实例化 servlet，然后初始化 servlet 调用 init() 的方法，接着调用服务的 service 的方法处理 doGet 和 doPost 方法，最后是我的还有容器关闭时候调用 destroy 销毁方法。

1.被创建：执行 init 方法，只执行一次

　　1.1Servlet什么时候被创建？

　　--默认情况下，第一次被访问时，Servlet被创建，然后执行init方法；

　　--可以配置执行Servlet的创建时机；

2.提供服务：执行service方法，执行多次

3.被销毁：当Servlet服务器正常关闭时，执行destroy方法，只执行一次

​	当客户端第一次请求Servlet的时候,tomcat会根据web.xml配置文件实例化servlet，当又有一个客户端访问该servlet的时候，不会再实例化该servlet，也就是多个线程在使用这个实例。

### 二、Servlet线程安全问题

​	当容器收到一个 Servlet 请求，Dispatcher 线程从线程池中选出一个工作组线程，将请求传递给该线程，然后由该线程来执行 Servlet 的 service 方法。

​	当这个线程正在执行的时候，容器收到另一个请求，调度者线程将从线程池中选出另外一个工作组线程来服务则个新的请求。当容器收到对同一个 Servlet 的多个请求的时候，那这个 servlet 的 service 方法将在多线程中并发的执行。

![img](https://img-blog.csdn.net/20130526081435048)

#### 多线程和单线程 Servlet 具体区别

​	多线程下每个线程对局部变量都会有自己的一份 copy，这样对局部变量的修改只会影响到自己的 copy 而不会对别的线程产生影响，线程安全的。但是对于实例变量来说，由于 servlet 在 Tomcat 中是以单例模式存在的，所有的线程共享实例变量。多个线程对共享资源的访问就造成了线程不安全问题。

### 三、变量的线程安全改造

```java
protected void doPost(HttpServletRequest request,
            HttpServletResponse response) throws ServletException, IOException {
        message = request.getParameter("message"); // 直接从参数中获取，还是跟其他的线程共享的
        PrintWriter printWriter = response.getWriter();
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        printWriter.write(message);
    }
    
    
    protected void doPost(HttpServletRequest request,
            HttpServletResponse response) throws ServletException, IOException {
        String message; // 将这类变量参数本地化（方法本地化）
        message = request.getParameter("message");
        PrintWriter printWriter = response.getWriter();
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        printWriter.write(message);
    }
```

