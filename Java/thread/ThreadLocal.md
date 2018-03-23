# ThreadLocal

# Thread类
 Thread类的公共构造函数都会调用到create函数
```
	private void create(ThreadGroup group, Runnable runnable, String threadName, long stackSize){
		Thread currentThread = Thread.currentThread();
        if (group == null) {
            group = currentThread.getThreadGroup();
        }
        ...
        // Transfer over InheritableThreadLocals.
        if (currentThread.inheritableValues != null) {
            inheritableValues = new ThreadLocal.Values(currentThread.inheritableValues);
        }

        // add ourselves to our ThreadGroup of choice
        this.group.addThread(this);

	}
```
 create函数接收四个参数，group, runnable, threadName, stackSize。

 然后检查当前线程的inheritableValues是否为空，如果为空则new一个ThreadLocal.Values。这是第一次发现ThreadLocal的地方，然后调用group的addThread函数。

 group的addThread函数是一个同步方法。把当前线程添加到group里的一个threadRefs里。threadRefs是一个ArrayList。`private final List<WeakReference<Thread>> threadRefs = new ArrayList<WeakReference<Thread>>(5);`

 然后构造方法结束了。。。

 再看成员变量，发现有两个ThreadLocal.Values类型的成员变量：
```
	/**
     * Normal thread local values.
     */
    ThreadLocal.Values localValues;

    /**
     * Inheritable thread local values.
     */
    ThreadLocal.Values inheritableValues;
```
 看ThreadLocal类

# ThreadLocal类
 ThreadLocal类还是比较简单的，包含：

* 空的构造函数
* public方法有：get、set、remove
* 一些私有或包访问权限的方法和变量
* 一个Values静态内部类

# ThreadLocal.Values类
 ThreadLocal.Values类有两个构造函数：
```
	/**
         * Constructs a new, empty instance.
         */
        Values() {
            initializeTable(INITIAL_SIZE);
            this.size = 0;
            this.tombstones = 0;
        }

        /**
         * Used for InheritableThreadLocals.
         */
        Values(Values fromParent) {
            this.table = fromParent.table.clone();
            this.mask = fromParent.mask;
            this.size = fromParent.size;
            this.tombstones = fromParent.tombstones;
            this.maximumLoad = fromParent.maximumLoad;
            this.clean = fromParent.clean;
            inheritValues(fromParent);
        }
```
 INITIAL_SIZE的值是16.
 从前面知道new Thread的时候会把当前线程的inheritableValues传进来，在这里可以看到把传进来的Values的值都赋值给了当前Values。实际上Values也就只有这么多成员变量。
- table：一个Object数组，存储map entries，包含了key和values，长度总是2的幂数。
- mask：用来将hash转换成索引
- size：有效的entry数量
- tombstones：tombstone的数量
- maximumLoad：有效entry和tombstone中的较大者
- clean：下一个将要clean的位置


