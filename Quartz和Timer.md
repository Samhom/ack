## Quartz和Timer

### 一、Java定时器之Quartz	

​	Quartz 是一个功能丰富的开源作业调度库，它可以集成在几乎任何Java应用程序中——从最小的独立应用程序到最大的电子商务系统。Quartz 可以用来创建简单或复杂的调度，用于执行数十、数百甚至数万个作业；任务被定义为标准的 Java 组件，实际上可以执行任何可以编程的任务。Quartz 调度器包含许多企业级功能，例如对 JTA事务和集群的支持。

#### demo

引入Quartz的jar包

```java 
		<dependency>
            <groupId>org.quartz-scheduler</groupId>
            <artifactId>quartz</artifactId>
            <version>2.3.0</version>
        </dependency>
```

创建相关测试类

```java 
public class QuartzTest {

    public static class QuartzTestJob implements Job {

        public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
            System.out.println("Quartz 测试任务被执行中...");
        }
    }

    /**
      * @Description 调度器（Simple Triggers）
      */
    public static void mySchedule() {
        try {
            //创建scheduler（调度器）实例
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();

            //创建JobDetail实例，创建要执行的job
            JobDetail jobDetail = JobBuilder.newJob(QuartzTestJob.class)
                    .withIdentity("job1", "group1").build();

            //构建Trigger（触发器）实例,每隔5s执行一次
            Trigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity("trigger1", "group1")
                    //立即生效
                    .startNow()
                    .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                            //每隔5s执行一次
                            .withIntervalInSeconds(5)
                            //一直执行
                            .repeatForever())
                    .build();

            //调度执行任务
            scheduler.scheduleJob(jobDetail, trigger);
            //启动
            scheduler.start();

            //睡眠
            //Thread.sleep(6000);

            //停止
            //scheduler.shutdown();
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

    /**
      * @Description 调度器（Cron Triggers）
      */
    public static void myCronSchedule() {
        try {
            //创建调度器实例scheduler
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();

            JobDetail job = JobBuilder.newJob(QuartzTestJob.class)
                    .withIdentity("job1", "group1")
                    .build();

            CronTrigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity("trigger1", "group1")
                    //20秒执行一次
                    .withSchedule(CronScheduleBuilder.cronSchedule("0/20 * * * * ?"))
                    .build();

            scheduler.scheduleJob(job, trigger);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        new QuartzTest().mySchedule();
    }
}
```

以上两种触发器的形式调度任务，其中 Cron Triggers需要使用到 cron 表达式。

##### Quartzs结构说明

- Job： 是一个接口，只定义一个方法 execute(JobExecutionContext context)，在实现接口的 execute 方法中编写所需要定时执行的 Job (任务)， JobExecutionContext 类提供了调度应用的一些信息。
- JobDetail： Quartz 每次调度 Job 时， 都重新创建一个 Job 实例， 所以它不直接接受一个 Job 的实例，相反它接收一个 Job 实现类。
- Trigger：是一个类，描述触发 Job 执行的时间触发规则。主要有 SimpleTrigger 和 CronTrigger 这两个子类。
- Scheduler：代表一个 Quartz 的独立运行容器， Trigger 和 JobDetail 可以注册到 Scheduler 中，拥有各自的组及名称，组及名称必须唯一（但可以和 Trigger 的组和名称相同，因为它们是不同类型的）。Scheduler 定义了多个接口方法， 允许外部通过组及名称访问和控制容器中 Trigger 和 JobDetail。

### 二、Java定时器之Timer

​	Timer 对象是在Java编程中常用到的定时计划任务功能，它在内部使用多线程的方式进行处理，所以它和多线程技术还是有非常大的关联的。

##### 在 JDK 中 Timer 类主要负责计划任务的功能，也就是在指定的时间开始执行某一个任务

```java
public class Timer {
    private final TaskQueue queue = new TaskQueue();

    private final TimerThread thread = new TimerThread(queue);

    private final Object threadReaper = new Object() {
        protected void finalize() throws Throwable {
            synchronized(queue) {
                thread.newTasksMayBeScheduled = false;
                queue.notify(); // In case queue is empty.
            }
        }
    };

    private final static AtomicInteger nextSerialNumber = new AtomicInteger(0);
    private static int serialNumber() {
        return nextSerialNumber.getAndIncrement();
    }

    public Timer() {
        this("Timer-" + serialNumber());
    }
    。。。省略部分构造方法

    /**
     * 在指定的时间开始调度指定的任务且只执行一次
       task：被调度执行的任务
       delay：延迟调度的时长（毫秒）
     */
    public void schedule(TimerTask task, long delay) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        sched(task, System.currentTimeMillis()+delay, 0);
    }

    /**
     * 在指定的时间开始调度指定的任务
       task：被调度执行的任务
       time：任务开始调度执行的时间
     */
    public void schedule(TimerTask task, Date time) {
        sched(task, time.getTime(), 0);
    }
    
    /**
     * 设定延迟调度任务及连续执行的时长
       delay：延迟调度的时长（毫秒）
       period：任务执行时间间隔（毫秒）
     */
	public void schedule(TimerTask task, long delay, long period) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, System.currentTimeMillis()+delay, -period);
    }
    
    /**
     * 设定延迟调度任务及连续执行的时长
       firstTime：任务第一次开始调度执行的时间
       period：任务执行时间间隔（毫秒）
     */
    public void schedule(TimerTask task, Date firstTime, long period) {
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, firstTime.getTime(), -period);
    }

    /**
     * 设定延迟调度任务及连续连续间隔执行的时常不同与schedule方法
       该方法的任务调度中每两个连续的任务的时间间隔是在第一个任务开始执行时就确认好的，
       比如第一次任务会在早上10点执行，连续任务时间间隔是1小时，那么以后的每次任务都是
       11点、12点、13点等准时执行，不管遇到什么情况（比如连续任务有时间上的间隔）
       firstTime：任务第一次开始调度执行的时间
       period：任务执行时间间隔（毫秒）
     */
    public void scheduleAtFixedRate(TimerTask task, long delay, long period) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, System.currentTimeMillis()+delay, period);
    }

    /**
     * 在指定的时间开始调度指定的任务
     */
    public void scheduleAtFixedRate(TimerTask task, Date firstTime, long period) {
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, firstTime.getTime(), period);
    }

    private void sched(TimerTask task, long time, long period) {
        if (time < 0)
            throw new IllegalArgumentException("Illegal execution time.");
        f (Math.abs(period) > (Long.MAX_VALUE >> 1))
            period >>= 1;
        synchronized(queue) {
            if (!thread.newTasksMayBeScheduled)
                throw new IllegalStateException("Timer already cancelled.");
            synchronized(task.lock) {
                if (task.state != TimerTask.VIRGIN)
                    throw new IllegalStateException(
                        "Task already scheduled or cancelled");
                task.nextExecutionTime = time;
                task.period = period;
                task.state = TimerTask.SCHEDULED;
            }
            queue.add(task);
            if (queue.getMin() == task)
                queue.notify();
        }
    }

    public void cancel() {
        synchronized(queue) {
            thread.newTasksMayBeScheduled = false;
            queue.clear();
            queue.notify();  // In case queue was already empty.
        }
    }

    public int purge() {
         int result = 0;
         synchronized(queue) {
             for (int i = queue.size(); i > 0; i--) {
                 if (queue.get(i).state == TimerTask.CANCELLED) {
                     queue.quickRemove(i);
                     result++;
                 }
             }
             if (result != 0)
                 queue.heapify();
         }
         return result;
     }
}
```

##### Timer内部维护了一个TaskQueue实例，TaskQueue实例对TimerTask的封装

```java
class TaskQueue {
    // 维护一个 TimerTask 数组
    private TimerTask[] queue = new TimerTask[128];
    private int size = 0;

    // 当前 TimerTask 数组中的 TimerTask 的数量 
    int size() {
        return size;
    }

    void add(TimerTask task) {
        // 扩容操作
        if (size + 1 == queue.length)
            queue = Arrays.copyOf(queue, 2*queue.length);
        queue[++size] = task;
        fixUp(size);
    }

    TimerTask getMin() {
        return queue[1];
    }

    TimerTask get(int i) {
        return queue[i];
    }

    void removeMin() {
        queue[1] = queue[size];
        // 预防内存泄露
        queue[size--] = null;
        fixDown(1);
    }

    void quickRemove(int i) {
        assert i <= size;
        queue[i] = queue[size];
        // 预防内存泄露
        queue[size--] = null;
    }

    void rescheduleMin(long newTime) {
        queue[1].nextExecutionTime = newTime;
        fixDown(1);
    }

    boolean isEmpty() {
        return size==0;
    }

    void clear() {
        for (int i=1; i<=size; i++)
            queue[i] = null;

        size = 0;
    }

    private void fixUp(int k) {
        while (k > 1) {
            int j = k >> 1;
            if (queue[j].nextExecutionTime <= queue[k].nextExecutionTime)
                break;
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }

    private void fixDown(int k) {
        int j;
        while ((j = k << 1) <= size && j > 0) {
            if (j < size &&
                queue[j].nextExecutionTime > queue[j+1].nextExecutionTime)
                j++; // j indexes smallest kid
            if (queue[k].nextExecutionTime <= queue[j].nextExecutionTime)
                break;
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }

    void heapify() {
        for (int i = size/2; i >= 1; i--)
            fixDown(i);
    }
}
```

##### 封装任务的类却是 TimerTask 类，TimerTask 是实现接口 Runnable 的一个抽象类，即任务调度时是启动一个新的线程来执行的

```java
public abstract class TimerTask implements Runnable {
    final Object lock = new Object();

    int state = VIRGIN;
    static final int VIRGIN = 0;
    static final int SCHEDULED   = 1;
    static final int EXECUTED    = 2;
    static final int CANCELLED   = 3;

    long nextExecutionTime;
    // 重复任务的周期（以毫秒为单位）。正值表示固定速率执行。负值表示固定延迟执行。值为0表示非重复任务。
    long period = 0;

    protected TimerTask() {
    }

    public abstract void run();

    public boolean cancel() {
        synchronized(lock) {
            boolean result = (state == SCHEDULED);
            state = CANCELLED;
            return result;
        }
    }

    public long scheduledExecutionTime() {
        synchronized(lock) {
            return (period < 0 ? nextExecutionTime + period
                               : nextExecutionTime - period);
        }
    }
}
```

#### demo

```java
public class TimerTest {

    public static class MyTimer extends TimerTask {
        public void run() {
            //任务执行的逻辑代码块
            System.out.println("任务被调度执行...");
        }
    }

    public static void main(String[] args) {
        Timer t = new Timer();
        MyTimer mt = new MyTimer();
        //立即执行
        //t.schedule(mt, new Date());
        
        //5s后执行
        //t.schedule(mt, 5000);

        //5s 后执行，每2s执行一次
        //t.schedule(mt, 5000, 2000);

        //每2s执行一次
        t.schedule(mt, new Date(), 2000);
    }
}
```

#### 3、Quartz和Timer对比

- ##### Quartz

  - 非常灵活，可以使用多种形式的定时方式
  - 非常轻量级，而且只需要很少的设置/配置
  - 是容错的，在系统重启后记住以前的记录
  - 可以参与事务

- ##### Timer

  - 定时器没有持久性机制
  - 创建方便简单，不用另外引入jar包