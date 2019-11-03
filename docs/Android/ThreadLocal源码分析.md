### ThreadLocal源码分析
 简述：相当于线程的一个本地变量，不同线程间不共享
 简单使用:
 ```java
 class ThreadLocalTest{
 	public static void main(String[] args){
    	ThreadLocal<Apple> appleThreadLocal = new ThreadLocal<Apple>();
        appleThreadLocal.set(new ThreadLocal);
        Apple apple = appleThreadLocal.get();
        System.out.println(apple.getName());
    }
 }
 class Apple{
 	private String name;
	public String getName(){
    	return name;
    }
    public void setName(String name){
    	this.name = name;
    }
 }

```
从入口进行分析
```java
	//先看构造方法
    /**
     * Creates a thread local variable.
     * @see #withInitial(java.util.function.Supplier)
     */
    public ThreadLocal() {
    }
    //接着看set函数
	 /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();//通过Thread获取当前线程的对象
        ThreadLocalMap map = getMap(t);//通过当前线程对象获取ThreadLocalmap
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    //接着看getMap方法
     /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    //这里返回的是当前线程里维护的ThreadLocalMap的属性，也就是说，每个Thread都有一个ThreadLocalMap
    //思考一个问题,如果在一个线程里创建多个TheadLocal实例，那么有多少个ThreadLocalMap？question?1

    //Thread类里的ThreadLocalMap属性
    /* ThreadLocal values pertaining to this thread. This map is maintained（维护）
     * by the ThreadLocal class. */
    //ThreadLocal.ThreadLocalMap threadLocals = null;

    //回到set函数，在set函数里，判断获取的map是否存在？存在就设置值，不存在走creatMap(t,value)函数，那么下面继续看createMap函数

    /**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    //创建一个ThreadLocalMap实例，这里要说明下，ThreadLocalMap是ThreadLocal里的一静态内部类(这里可以延展下静态内部类)ext?1
    //继续看ThreadLocalMap的构造函数
     /**
         * Construct a new map initially containing (firstKey, firstValue).
         * ThreadLocalMaps are constructed lazily, so we only create
         * one when we have at least one entry to put in it.
         */
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
	//看到这里有个Entry数组，同样说明下，这个Entry是继承自WeakReference的一个弱引用，而且是ThreadLocalMap里的一个静态内部类(ThreadLocalMap是ThreadLocal里的一个静态内部类)
    //继续看下Entry类
     /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        //软引用引用的当前ThreadLocal对象，具体为什么还要思考下 ext 2
        //继续看ThreadLocalMap的构造函数的关键代码
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        //这个i就是数组的下标,怎么计算单的呢？firstKey.thredLocalHashCode&（INITAL_CAPACITY-1）；
        //首先看firstKey是什么？firstKey就是当前TheadLocal实例对象，那threadLocalHashCode就是实例对象的属性
        //这个属性我们看下。
        /**
     * ThreadLocals rely（依靠） on per-thread linear-probe hash maps attached
     * to each thread (Thread.threadLocals and
     * inheritableThreadLocals).  The ThreadLocal objects act as keys,
     * searched via threadLocalHashCode.  This is a custom hash code
     * (useful only within ThreadLocalMaps) that eliminates（消除） collisions
     * in the common case where consecutively constructed ThreadLocals
     * are used by the same threads, while remaining well-behaved in
     * less common cases.
     */
    private final int threadLocalHashCode = nextHashCode();//调用了一个nextHashCode()；方法
     /**
     * Returns the next hash code.
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);	//继续看nextHashCode是什么
    }
     /**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     */
    private static AtomicInteger nextHashCode = new AtomicInteger();//是一个静态的自增长integer值，也就是说每次创建实例都会增长
    //上面提到的HASH_INCREMENT是什么呢？
     /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    private static final int HASH_INCREMENT = 0x61c88647;//斐波那契数列，综上所述就是每次创建实例都会通过nextHashCode()函数获取到一个相对分散的hashcode值，而且这个是是Entry的下标。

```
简单总结下上面的流程：首先创建ThreadLocal实例，这时候ThreadLocal的threadLocalHashCode会根据斐波那契数列计算出一个新的值，这个值是作为Entry的下标使用的，同时如果是第一次创建实例，ThreadLocalMap为null，那就会创建一个TheadLocalMap对象，并将这个ThreadLocalMap对象赋值给当前线程，也就是每个线程维护一个ThreadLocalMap对象，那在创建ThreadLocalMap对象的时候，在其内部会创建一个Entry数组，这个Entry是继承自弱引用的一个类，主要是弱引用ThreadLocal实例，并且根据threadLocalHashCode把希望存储的值以Entry为实例放到以threadLocalHashCode为下标的Entry数组里。这是以上初始流程，下面分析get()和set()方法

#### set方法
```java
 /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);//首先获取当前线程的ThreadLocalMap实例（每个线程维护一个ThreadLocalMap实例）
        if (map != null)
            map.set(this, value);//分析不为null的时候，通过ThreadLocalMap实例的set方法保存
        else
            createMap(t, value);
    }
    
     /**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;//获取到当前数组的长度
            int i = key.threadLocalHashCode & (len-1);//通过ThreadLocal计算下标(上描述过这个属性)

			//如果这个下标存在就更新数据
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
			//不存在的话就创建Entry实例并放入相应的位置上
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
        //这是ThreadLocalMap类的set方法
        //下面分析get方法
        
        /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    //这个比较简单就是获取当前线程的ThreadLocalMap对象并通过ThreadLocalMap的getEntry获取到Entry对象，并获取到Entry对象的Value
 		/**
         * Get the entry associated with key.  This method
         * itself handles only the fast path: a direct hit of existing
         * key. It otherwise relays to getEntryAfterMiss.  This is
         * designed to maximize performance for direct hits, in part
         * by making this method readily inlinable.
         *
         * @param  key the thread local object
         * @return the entry associated with key, or null if no such
         */
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
        //根据下标获取到Entry对象并返回
    

```
#### 以上就是整个ThreadLocal的分析，有一些具体细节也没有分析到，比如扩容等等

