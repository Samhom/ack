## jps、jinfo、jstack、jmap、jhat、jstat、hprof 

### 一、jps(Java Virtual Machine Process Status Tool)

jps主要用来输出JVM中运行的进程状态信息。语法格式如下：

```properties
jps [options] [hostid]
```

如果不指定 hostid 就默认为当前主机或服务器。

命令行参数选项说明如下：

```properties
-q  不输出类名、Jar名和传入main方法的参数
-m  输出传入main方法的参数
-l  输出main类或Jar的全限名
-v  输出传入JVM的参数
```

比如下面：

```shell
root@ubuntu:/# jps -m -l
2458 org.artifactory.standalone.main.Main /usr/local/artifactory-2.2.5/etc/jetty.xml
29920 com.sun.tools.hat.Main -port 9998 /tmp/dump.dat
3149 org.apache.catalina.startup.Bootstrap start
30972 sun.tools.jps.Jps -m -l
8247 org.apache.catalina.startup.Bootstrap start
25687 com.sun.tools.hat.Main -port 9999 dump.dat
21711 mrf-center.jar
```

### 二、jstack

jstack主要用来查看某个Java进程内的线程堆栈信息。语法格式如下：

```properties
jstack [option] pid
jstack [option] executable core
jstack [option] [server-id@]remote-hostname-or-ip
```

命令行参数选项说明如下：

```properties
-l   会打印出额外的锁信息，在发生死锁时可以用 jstack -l pid 来观察锁持有情况
-m   不仅会输出 Java 堆栈信息，还会输出 C/C++ 堆栈信息（比如 Native 方法）
```

​	jstack 可以定位到线程堆栈，根据堆栈信息我们可以定位到具体代码，所以它在JVM性能调优中使用得非常多。下面我们来一个实例找出某个 Java 进程中最耗费 CPU 的 Java 线程并定位堆栈信息，用到的命令有 ps、top、printf、jstack、grep。

第一步先找出 Java 进程 ID，我部署在服务器上的 Java 应用名称为 mrf-center：

```text
root@ubuntu:/# ps -ef | grep mrf-center | grep -v grep
root     21711     1  1 14:47 pts/3    00:02:10 java -jar mrf-center.jar
```

得到进程 ID 为 21711，第二步找出该进程内最耗费 CPU 的线程，可以使用 ps -Lfp pid 或者 ps -mp pid -o THREAD, tid, time 或者 top -Hp pid，我这里用第三个，输出如下：

![img](https://pic4.zhimg.com/80/v2-504463fa918e5b7708df826500752a4f_720w.jpg)

TIME 列就是各个 Java 线程耗费的 CPU 时间，CPU 时间最长的是线程 ID 为 21742 的线程，用

```shell
printf "%x\n" 21742
```

得到 21742 的十六进制值为 54ee，下面会用到。

OK，下一步终于轮到 jstack 上场了，它用来输出进程 21711 的堆栈信息，然后根据线程 ID 的十六进制值 grep，如下：

```shell
root@ubuntu:/# jstack 21711 | grep 54ee
"PollIntervalRetrySchedulerThread" prio=10 tid=0x00007f950043e000 nid=0x54ee in Object.wait() [0x00007f94c6eda000]
```

可以看到 CPU 消耗在 PollIntervalRetrySchedulerThread 这个类的 Object.wait()，我找了下我的代码，定位到下面的代码：

```java
// Idle wait
getLog().info("Thread [" + getName() + "] is idle waiting...");
schedulerThreadState = PollTaskSchedulerThreadState.IdleWaiting;
long now = System.currentTimeMillis();
long waitTime = now + getIdleWaitTime();
long timeUntilContinue = waitTime - now;
synchronized(sigLock) {    
    try {
        if(!halted.get()) {
            sigLock.wait(timeUntilContinue);
        }
    } catch (InterruptedException ignore) {
        
    }
}
```

它是轮询任务的空闲等待代码，上面的sigLock.wait(timeUntilContinue)就对应了前面的Object.wait()。

### 三、jmap（Memory Map）和 jhat（Java Heap Analysis Tool）

jmap 命令是一个可以输出所有内存中对象的工具，甚至可以将 VM 中的 heap，以二进制输出成文本。

jmap 用来查看堆内存使用状况，一般结合 jhat 使用。

jmap 语法格式如下：

```properties
jmap [option] pid        ## 连接到正在运行的进程
jmap [option] executable core      ## 连接到核心文件
jmap [option] [server-id@]remote-hostname-or-ip      ## 连接到远程调试服务
```

options： 

```shell
>    pid:    目标进程的PID，进程编号，可以采用ps -ef | grep java 查看java进程的PID;
>    executable:     产生 core dump 的 java 可执行程序;
>    core:     将被打印信息的 core dump 文件;
>    remote-hostname-or-IP:     远程 debug 服务的主机名或 ip;
>    server-id:     唯一 id,假如一台主机上多个远程 debug 服务;
```

如果运行在64位JVM上，可能需要指定 -J-d64 命令选项参数。

#### 1、使用 hprof 二进制形式输出 jvm 的 heap 内容到文件：

```shell
## options 可替换为 -dump:[live,]format=b,file=<filename> live子选项是可选的，假如指定live选项,那么只输出活的对象到文件. 
## 将进程的内存heap输出出来到dump文件里，可再配合MAT（内存分析工具）。
root@ubuntu:/# jmap -dump:live,format=b,file=/tmp/dump.dat 21711
Dumping heap to /tmp/dump.dat ...
Heap dump file created
```

dump出来的文件可以用MAT、VisualVM等工具查看，这里用jhat查看：

```text
root@ubuntu:/# jhat -port 9998 /tmp/dump.dat
Reading from /tmp/dump.dat...
Dump file created Tue Jan 28 17:46:14 CST 2014Snapshot read, resolving...
Resolving 132207 objects...
Chasing references, expect 26 dots..........................
Eliminating duplicate references..........................
Snapshot resolved.
Started HTTP server on port 9998Server is ready.
```

​	注意如果 Dump 文件太大，可能需要加上 -J-Xmx512m 这种参数指定最大堆内存，即 jhat -J-Xmx512m -port 9998 /tmp/dump.dat。然后就可以在浏览器中输入主机地址:9998查看了：

![img](https://pic3.zhimg.com/80/v2-f0a572038bad53263108b85793b5e636_720w.jpg)

上面红线框出来的部分大家可以自己去摸索下，最后一项支持OQL（对象查询语言）。

#### 2、打印正等候回收的对象的信息

```shell
## -finalizerinfo
> jmap -finalizerinfo 3772
Attaching to process ID 19570, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.80-b11
Number of objects pending for finalization: 0 (等候回收的对象为0个)
```

#### 3、打印 heap 的概要信息，GC 使用的算法，heap（堆）的配置及 JVM 堆内存的使用情况

```shell
> jmap -heap 19570
...
虚拟机的一些基本信息
...
using parallel threads in the new generation.  ## 新生代采用的是并行线程处理方式
using thread-local object allocation.   
Concurrent Mark-Sweep GC   ## 同步并行垃圾回收

Heap Configuration:  ## 堆配置情况，也就是JVM参数配置的结果[平常说的tomcat配置JVM参数，就是在配置这些]
   MinHeapFreeRatio = 40 ##最小堆使用比例
   MaxHeapFreeRatio = 70 ##最大堆可用比例
   MaxHeapSize      = 2147483648 (2048.0MB) ##最大堆空间大小
   NewSize          = 268435456 (256.0MB) ##新生代分配大小
   MaxNewSize       = 268435456 (256.0MB) ##最大可新生代分配大小
   OldSize          = 5439488 (5.1875MB) ##老年代大小
   NewRatio         = 2  ##新生代比例
   SurvivorRatio    = 8  ##新生代与suvivor的比例
   PermSize         = 134217728 (128.0MB) ##perm区 永久代大小
   MaxPermSize      = 134217728 (128.0MB) ##最大可分配perm区 也就是永久代大小

Heap Usage: ##堆使用情况【堆内存实际的使用情况】
New Generation (Eden + 1 Survivor Space):  ##新生代（伊甸区Eden区 + 幸存区survior(1+2)空间）
   capacity = 241631232 (230.4375MB)  ##伊甸区容量
   used     = 77776272 (74.17323303222656MB) ##已经使用大小
   free     = 163854960 (156.26426696777344MB) ##剩余容量
   32.188004570534986% used ##使用比例
Eden Space:  ##伊甸区
   capacity = 214827008 (204.875MB) ##伊甸区容量
   used     = 74442288 (70.99369812011719MB) ##伊甸区使用
   free     = 140384720 (133.8813018798828MB) ##伊甸区当前剩余容量
   34.65220164496263% used ##伊甸区使用情况
From Space: ##survior1区
   capacity = 26804224 (25.5625MB) ##survior1区容量
   used     = 3333984 (3.179534912109375MB) ##surviror1区已使用情况
   free     = 23470240 (22.382965087890625MB) ##surviror1区剩余容量
   12.43827838477995% used ##survior1区使用比例
To Space: ##survior2 区
   capacity = 26804224 (25.5625MB) ##survior2区容量
   used     = 0 (0.0MB) ##survior2区已使用情况
   free     = 26804224 (25.5625MB) ##survior2区剩余容量
   0.0% used ## survior2区使用比例
PS Old  Generation: ##老年代使用情况
   capacity = 1879048192 (1792.0MB) ##老年代容量
   used     = 30847928 (29.41887664794922MB) ##老年代已使用容量
   free     = 1848200264 (1762.5811233520508MB) ##老年代剩余容量
   1.6416783843721663% used ##老年代使用比例
Perm Generation: ##永久代使用情况
   capacity = 134217728 (128.0MB) ##perm区容量
   used     = 47303016 (45.111671447753906MB) ##perm区已使用容量
   free     = 86914712 (82.8883285522461MB) ##perm区剩余容量
   35.24349331855774% used ##perm区使用比例
```

#### 4、打印每个 class 的实例数目,内存占用,类全名信息

VM的内部类名字开头会加上前缀”*”. 如果live子参数加上后,只统计活的对象数量. 

```shell
> jmap -histo:live 19570
num     #instances（实例）   #bytes（字节大小）  class name（类名）
----------------------------------------------
   1:         65220        9755240  　　　　<constMethodKlass>
   2:         65220        8880384  　　　　<methodKlass>
   3:         11721        8252112  　　　　[B
   4:          6300        6784040  　　　　<constantPoolKlass>
   5:         75224        6218208  　　　　[C
   6:         93969        5163280  　　　　<symbolKlass>
   7:          6300        4854440  　　　　<instanceKlassKlass>
   8:          5482        4203152  　　　　<constantPoolCacheKlass>
   9:         72097        2307104  　　　　java.lang.String
  10:         15102        2289912  　　　　[I
  11:          4089        2227728  　　　　<methodDataKlass>
  12:         28887        1386576 　　　　 org.apache.velocity.runtime.parser.Token
  13:          6792         706368          java.lang.Class
  14:          7445         638312          [Ljava.util.HashMap$Entry;

4380:             1             16          com.sun.proxy.$Proxy208
4381:             1             16          sun.reflect.GeneratedMethodAccessor198
4382:             1             16          com.sun.proxy.$Proxy46
4383:             1             16          org.apache.ibatis.ognl.SetPropertyAccessor
4384:             1             16          oracle.jdbc.driver.OracleDriver
4385:             1             16          com.sun.proxy.$Proxy181
4386:             1             16          com.sun.proxy.$Proxy82
4387:             1             16          java.util.jar.JavaUtilJarAccessImpl
4388:             1             16          com.sun.proxy.$Proxy171
4389:             1             16          sun.reflect.GeneratedMethodAccessor136
4390:             1             16            sun.reflect.GeneratedConstructorAccessor22
4391:             1             16          org.elasticsearch.action.search.SearchAction
4392:             1             16          org.springframework.core.annotation.AnnotationAwareOrderComparator
Total       1756265      162523736
```

#### 5、打印 classload 和 jvm heap 长久层的信息

​	打印 classload 和 jvm heap 长久层的信息，包含每个 classloader 的名字,活泼性,地址,父 classloader 和加载的 class 数量. 另外,内部 String 的数量和占用内存数也会打印出来.。

```shell
> jmap -permstat pid
Attaching to process ID 19570, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.80-b11
finding class loader instances ..done.
computing per loader stat ..done.
please wait.. computing liveness.liveness analysis may be inaccurate ...
class_loader    classes    bytes    parent_loader    alive?    type

<bootstrap>    2538    14654264      null      live    <internal>
0x000000070af968c8    63    399160    0x0000000707db1788    dead    org/apache/catalina/loader/WebappClassLoader@0x000000070367d2a8
0x000000070cba7b08    1    3064    0x0000000707e709a8    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070cba6a38    28    221344    0x0000000707e709a8    dead    org/apache/jasperrvlet/JasperLoader@0x0000000705b11878
0x000000070baed8b8    32    297296    0x0000000707e709a8    dead    org/apache/jasperrvlet/JasperLoader@0x0000000705b11878
0x000000070919a610    1    3056    0x0000000707e709a8    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070bdd1e18    1    3064    0x0000000707e709a8    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x0000000707f1d480    1    3072      null      dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070cba7f48    1    1912    0x0000000707e709a8    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070cba8508    1    3064    0x0000000707e709a8    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070cba6c40    1    3064    0x0000000707e709a8    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070bd4c6a0    1    3056      null      dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070cba62b0    28    235472    0x0000000707e709a8    dead    org/apache/jasperrvlet/JasperLoader@0x0000000705b11878
0x000000070cba77c8    1    3064    0x0000000707e709a8    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070cba7388    1    3064    0x0000000707e709a8    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070cba8148    1    3064    0x0000000707e709a8    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070afd8b60    1    6704    0x0000000707db1788    dead    org/apache/catalina/loader/WebappClassLoader@0x000000070367d2a8
0x000000070919a410    1    1888      null      dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070bdd05b0    1    1912      null      dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070bc848b8    1    3088    0x0000000707db1788    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070cba64e8    1    1888    0x0000000707e709a8    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x0000000707f1d2c0    1    3064    0x0000000707db1788    dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070be82e38    1    1912      null      dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
0x000000070cba7908    1    3208      null      dead    sun/reflect/DelegatingClassLoader@0x0000000702a50b98
.........

total = 273    12995    87547304        N/A        alive=1, dead=272        N/A
```

### 四、jstat（JVM统计监测工具）

语法格式如下：

```shell
jstat [ generalOption | outputOptions vmid [interval[s|ms] [count]] ]
```

vmid 是 Java 虚拟机 ID，在 Linux/Unix 系统上一般就是进程 ID。interval 是采样时间间隔。count 是采样数目。比如下面输出的是 GC 信息，采样时间间隔为 250ms，采样数为 4：

```shell
root@ubuntu:/# jstat -gc 21711 250 4
S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT
192.0  192.0   64.0   0.0    6144.0   1854.9   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649
192.0  192.0   64.0   0.0    6144.0   1972.2   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649
192.0  192.0   64.0   0.0    6144.0   1972.2   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649
192.0  192.0   64.0   0.0    6144.0   2109.7   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649
```

要明白上面各列的意义，先看JVM堆内存布局：

![img](https://pic1.zhimg.com/80/v2-d682034813378ef949a927d015d4e05c_720w.jpg)

可以看出：

```properties
堆内存 = 年轻代 + 年老代 + 永久代
年轻代 = Eden区 + 两个Survivor区（From和To）
```

现在来解释各列含义：

```text
S0C、S1C、S0U、S1U：Survivor 0/1区容量（Capacity）和使用量（Used）
EC、EU：Eden区容量和使用量
OC、OU：年老代容量和使用量
PC、PU：永久代容量和使用量
YGC、YGT：年轻代GC次数和GC耗时
FGC、FGCT：Full GC次数和Full GC耗时
GCT：GC总耗时
```

### 五、hprof（Heap/CPU Profiling Tool）

hprof 能够展现 CPU 使用率，统计堆内存使用情况。

语法格式如下：

```properties
java -agentlib:hprof[=options] ToBeProfiledClass
java -Xrunprof[:options] ToBeProfiledClass
javac -J-agentlib:hprof[=options] ToBeProfiledClass
```

完整的命令选项如下：

```properties
Option Name and Value  Description                    Default
---------------------  -----------                    -------
heap=dump|sites|all    heap profiling                 all
cpu=samples|times|old  CPU usage                      off
monitor=y|n            monitor contention             n
format=a|b             text(txt) or binary output     a
file=<file>            write data to file             java.hprof[.txt]
net=<host>:<port>      send data over a socket        off
depth=<size>           stack trace depth              4
interval=<ms>          sample interval in ms          10
cutoff=<value>         output cutoff point            0.0001
lineno=y|n             line number in traces?         y
thread=y|n             thread in traces?              n
doe=y|n                dump on exit?                  y
msa=y|n                Solaris micro state accounting n
force=y|n              force output to <file>         y
verbose=y|n            print messages about dumps     y
```

来几个官方指南上的实例。

CPU Usage Sampling Profiling(cpu=samples)的例子：

```shell
java -agentlib:hprof=cpu=samples,interval=20,depth=3 Hello
```

上面每隔20毫秒采样CPU消耗信息，堆栈深度为3，生成的profile文件名称是java.hprof.txt，在当前目录。

CPU Usage Times Profiling(cpu=times)的例子，它相对于CPU Usage Sampling Profile能够获得更加细粒度的CPU消耗信息，能够细到每个方法调用的开始和结束，它的实现使用了字节码注入技术（BCI）：

```text
javac -J-agentlib:hprof=cpu=times Hello.java
```

Heap Allocation Profiling(heap=sites)的例子：

```text
javac -J-agentlib:hprof=heap=sites Hello.java
```

Heap Dump(heap=dump)的例子，它比上面的Heap Allocation Profiling能生成更详细的Heap Dump信息：

```text
javac -J-agentlib:hprof=heap=dump Hello.java
```

**虽然在JVM启动参数中加入-Xrunprof:heap=sites参数可以生成CPU/Heap Profile文件，但对JVM性能影响非常大，不建议在线上服务器环境使用。**