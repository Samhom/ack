## Object 类

```java
Object类（7个native方法、3个wait()方法、个一个静态方法块、一个equals、一个finalize）：
	public class Object {

    private static native void registerNatives();
    static {
        registerNatives();
    }
    /**
     * 用来获取运行时的对象
     */
    public final native Class<?> getClass();

    public native int hashCode();

    public boolean equals(Object obj) {
        return (this == obj);
    }

    /**
     * 浅拷贝 和 深拷贝
     */
    protected native Object clone() throws CloneNotSupportedException;

	/**
     * 该字符串由类名（对象是该类的一个实例）、标记符“@”和此对象哈希码的无符号十六进制表示组成。
     */
    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }

    /*唤醒在此对象监视器上等待的单个线程。*/
    public final native void notify();

    /*唤醒在此对象监视器上等待的所有线程。*/
    public final native void notifyAll();

    public final native void wait(long timeout) throws InterruptedException;

    public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(nanosecond timeout value out of range");
        }
        if (nanos > 0) {
            timeout++;
        }
        wait(timeout);
    }

    public final void wait() throws InterruptedException {
        wait(0);
    }

    /**
     * 垃圾回收器准备释放内存的时候，会先调用finalize()。
     */
    protected void finalize() throws Throwable { }
}
```