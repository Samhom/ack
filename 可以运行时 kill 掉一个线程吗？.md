## 可以运行时 kill 掉一个线程吗？

### 1、使用共享变量的方式

```java
public class ThreadFlag extends Thread { 
    public volatile boolean exit = false; 
    public void run() { 
        while (!exit); 
    } 

    public static void main(String[] args) throws Exception { 
        ThreadFlag thread = new ThreadFlag(); 
        thread.start(); 
        sleep(3000); // 主线程延迟3秒 
        thread.exit = true;  // 终止线程thread 
        thread.join(); 
        System.out.println("线程退出!"); 
    } 
} 
```

### 2、使用 interrupt 方法终止线程

```java
public class MyThread1 extends Thread {

    volatile boolean stop = false;

    public void run() {
        while (!stop) {
            System.out.println(getName() + " is running");
            try {
                sleep(1000);
            } catch (InterruptedException e) {
                System.out.println("week up from blcok...");
                stop = true; // 在异常处理代码中修改共享变量的状态
            }
        }
        System.out.println(getName() + " is exiting...");
    }

    public static void main(String[] args) throws InterruptedException {
        MyThread1 m1 = new MyThread1();
        System.out.println("Starting thread...");
        m1.start();

        Thread.sleep(3000);
        System.out.println("Interrupt thread...: " + m1.getName());
        m1.stop = true; // 设置共享变量为true
        m1.interrupt(); // 阻塞时退出阻塞状态
        Thread.sleep(3000); // 主线程休眠3秒以便观察线程m1的中断情况
        System.out.println("Stopping application...");
    }
}
```

