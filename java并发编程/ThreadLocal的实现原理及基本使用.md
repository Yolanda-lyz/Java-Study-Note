ThreadLocal，顾名思义，肯定是与Thread类有关系的，所以先从Thread类入手。

### 1. Thread和ThreadLocal的关联

Thread类中有两个类型为ThreadLocal.ThreadLocalMap的成员变量threadLocals和inheritableThreadLocals，根据注释可知，这两个成员变量是ThreadLocal同当前线程的关联，且是由ThreadLocal类（InheritableThreadLocal类是ThreadLocal类的派生类）维护的。

```java
//Thread
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

### 2. 变量的初始化

那么这两个成员变量是什么时候被初始化的？前面说这两个map是由ThreadLocal维护的，说明初始化可能就在ThreadLocal类中。
当线程第一次调用ThreadLocal的set方法或get方法时发现线程的threadLocals变量仍为空，此时就会对其进行赋值，指向一个新建的ThreadLocalMap对象。

```java
public class ThreadLocal<T> {
    ......
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    ......
}
```

ThreadLocal的内部类ThreadLocalMap的结构也是比较简单，其内部维护了一个Entry数组，用于存储ThreadLocal和value值的键值对。Entry继承了ThreadLocal的弱引用，同时内部保存了需要存储的value值。
这里关注一下table表中首次插入索引值的取值方式，取键值的hashCode与table表的容量-1的值相与，得到的即为索引值。

```java
public class ThreadLocal<T> {
    ......
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        ......
        private Entry[] table;
        ......
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
    }
    ......
}
```

而inheritableThreadLocals的初始化则在Thread类的构造函数中，它指向的是一个新建的ThreadLocalMap对象，其内部封装了父线程的inheritableThreadLocals成员变量。
inheritableThreadLocals存储的是父线程中可向子线程中传递的ThreadLocal.ThreadLocalMap。在构造新的ThreadLocalMap对象时，子线程会将parentMap中的所有记录逐一复制到自身的线程。

```java
//Thread
    Thread(ThreadGroup group, String name, int priority, boolean daemon) {
        ......
        init2(currentThread());
        ......
    }

    private void init2(Thread parent) {
        ......
        if (parent.inheritableThreadLocals != null) {
            this.inheritableThreadLocals = ThreadLocal.createInheritedMap(
                    parent.inheritableThreadLocals);
        }
    }
//ThreadLocal
    static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
        return new ThreadLocalMap(parentMap);
    }
```

结合上面的源码，就能理解《java并发编程的艺术》中关于ThreadLocal的解析：

>ThreadLocal，即线程变量，是一个以ThreadLocal对象为键、任意对象为值的存储结构，这个结构被附带在线程上，也就是说一个线程可以根据一个ThreadLocal对象查询绑定在这个线程上的一个值。

### 3. ThreadLocal的作用

根据以上的存储结构的设计，可以发现：ThreadLocal只是这个存储结构的入口，数据真正是保存在Thread类中的，每个Thread类都指向自己的ThreadLocalMap，从而在线程上实现了数据隔离。
我们可以通过创建不同类型的ThreadLocal对象，通过ThreadLocal的set方法将当前ThreadLocal对象以及要存储的对象打包保存到当前线程中，从而通过不同的ThreadLocal对象以及不同线程的将存储对象隔离开来。

ThreadLocal是为了解决对象不能被多线程共享访问的问题，每个线程使用保存在自身的对象实例，彼此相互隔离，互不影响。
同同步机制对比，同步机制就是通过控制线程访问同一个共享对象的顺序，而ThreadLocal则是每个线程都有一个各自的实例对象，各用各的互不影响，ThreadLocal也可理解为一种“空间换时间”的方式，而同步机制则是“时间换空间”。

### 4. ThreadLocal的内存泄露问题

>https://www.cnblogs.com/windpoplar/p/11869661.html：
>ThreadLocal的实现是这样的：每个Thread 维护一个 ThreadLocalMap 映射表，这个映射表的 key 是 ThreadLocal实例本身，value 是真正需要存储的 Object。
>也就是说 ThreadLocal 本身并不存储值，它只是作为一个 key 来让线程从 ThreadLocalMap 获取 value。
>值得注意的是图中的虚线，表示 ThreadLocalMap 是使用 ThreadLocal 的弱引用作为 Key 的，弱引用的对象在 GC 时会被回收。
>ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用来引用它，那么系统 GC 的时候，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value永远无法回收，造成内存泄漏。

既然泄漏是因为使用了弱引用导致的，那为何还要使用弱引用？
若ThreadLocal使用的是强引用，当ThreadLocal外部强引用被清除时，ThreadLocalMap内部的Entry仍强引用ThreadLocal对象，导致GC回收时ThreadLocal仍有强引用，导致无法清除。
所以每次使用完ThreadLocal，调用其remove方法及时清除关联的数据。

#### 5. ThreadLocalMap的插入（冲突的解决以及脏数据处理）

当ThreadLocalMap非空后，就会调用其set方法来插入新的Entry对象。这里关注一下当冲突发生的解决方式：当通过键值的hashCode与容量-1的值相与得到的索引位置非空时，如果该索引对应的Entry对象的键值与当前键值相同，则直接替换value值，若Entry对象的键值为空，则调用replaceStaleEntry方法进行替换（这里即为脏数据的处理过程）；否则则当前索引值加1，重复判断，或新的索引位置为空，则直接插入。
显然，一个ThreadLocal对象在一个ThreadLocalMap中只能对应一个值。

```java
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

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

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }

        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

```

上面提到replaceStaleEntry方法是用于处理脏数据的，当查找到键值为null的Entry时，说明已经有过GC了，此时会通过replaceStaleEntry方法及时清理部分键值为null的数据，该方法也是解决内存泄漏的一种优化。
具体的处理细节可以参考[这篇文章](https://www.jianshu.com/p/dde92ec37bd1)，里面结合图及例子讲得挺全的。
