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

```
	private void initializeTable(int capacity){
		this.table = new Object[capacity * 2];
		this.mask = table.length -1;
		this.clean = 0;
		this.maximumLoad = capacity * 2 / 3;
	}
```

 initializeTable会新生成一个Object类型的数组赋值给table，大小是传尽来参数的2倍。和其他的一些用来管理数组的变量的初始化工作。

```
	private void inheritValues(Values fromParent){
		//从父线程的value转换到子线程的values
		Object[] table = this.table;
		for(int i = table.length -2; i >= 0; i -= 2){
			Object k = table[i];
			if(k== null || k == TOMBSTONE){
				//跳过
				continue;
			}
			//数组只能包含null，tombstones，references
			Reference<InheritableThreadLocal<?>> reference = (Reference<InheritableThreadLocal<?>>) k;
			InheritableThreadLocal key = reference.get();
			if(key != null){
				table[i + 1] = key.childValue(fromParent.table[i + 1]);
			}else{
				table[i] = TOMBSTONE;//TOMESTONE就是一个Object对象
				table[i + 1] = null;
				fromParent.table[i] = TOMESTONE;
				fromParent.table[i + 1] = null;

				tombstones++;
				fromParent.tombstones++;

				size--;
				fromParent.size--;
			}
		}
	}
```
 从代码中可以看出来，table数组在存值的时候是key和value存储在数组相邻的两个地方。从后往前依次访问key和value值，整理数组内的数据，最终整个函数达到所有的数据都有效的目的。

 Values还有其他的一些函数，如cleanup，resize，put，remove等，都是为了维护这个Object数组，并提供方法供ThreadLocal类调用。
 
# 总结

 至此我们了解到了ThreadLocal里面有一个静态内部类Values，这两个都是纯Java类。Values内部通过Object数组通过在2n位置存Key，2n+1位置存Value的方式保存数据，自动扩展和收缩数组大小，并提供初始化，存取方法供ThreadLocal调用。而TreadLocal类主要是封装Values，并维护Thread内部的数据。

 ThreadLocal里并没有native的函数，也没有特别复杂的逻辑，和刚开始的猜想有点不一样，也可能是因为我是根据android4.4.2在Eclipse反编译下看到的缘故，google对ThreadLocal进行了简化修改吧。总之第一次分析源码，感觉还不错:)



