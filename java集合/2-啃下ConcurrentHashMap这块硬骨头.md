ConcurrentHashMap是线程安全且高效的HashMap，学习ConcurrentHashMap之前可以先了解[HashMap的基本原理](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E9%9B%86%E5%90%88/1-java%E9%9D%A2%E8%AF%95%E5%B8%B8%E5%AE%A2%EF%BC%9AHashMap%E8%A7%A3%E6%9E%90.md)。

HashMap是非线程安全的，在多线程环境下会导致死循环、数据覆盖等问题，因此在多线程环境下，一般会选择线程安全的集合来替代其使用：

- 调用Collections.synchronizedMap来创建线程安全的map集合；
- hashtable
- ConcurrentHashMap

Collections.synchronizedMap(Map<K, V> m)返回一个SynchronizedMap类对象，SynchronizedMap类是Collections的内部类，其内部维护了一个Map对象和一个锁对象mutex，mutex默认情况下会被赋值为this，也可由传入的mutex参数指定。之后由SynchronizedMap对象间接操作map，SynchronizedMap各方法内部通过锁定mutex对象来实现同步。

HashTable内部则是对put、get等方法加synchronized，在同一时刻只能有一个线程对其进行读写，在多线程竞争激烈时会导致效率低下。

### 1. jdk 1.7的ConcurrentHashMap

jdk1.7的ConcurrentHashMap采用的是分段锁的机制，实现并发更新，底层采用数组+链表的存储结构。包含两个核心静态内部类：Segment和HashEntry。

#### 1.1 数据结构

ConcurrentHashMap维护了一个Segment数组，每个Segment内部类对象又维护了一个声明为```transient volatile```的HashEntry成员变量。
HashEntry类内部及ConcurrentHashMap类都维护了一个UnSafe成员变量，Unsafe类底层实际上是调用C代码，使得java语言拥有类似C语言指针操作内存空间的能力。C代码调用汇编，生成CPU指令，使得对应的操作是原子的。

```java
public class ConcurrentHashMap<K, V> extends ... {
    ...
    final Segment<K,V>[] segments;
    
    static final class Segment<K,V> extends ReentrantLock implements Serializable {
        ...
        //每个segment会维护一个HashEntry数组，是真正存放数据的桶结构
        transient volatile HashEntry<K,V>[] table;
        //计数器，只能在锁内或volatile读中访问
        transient int count;
        //用于快速失败安全机制
        transient int modCount;
    }
    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        //volatile修饰value和next节点
        volatile V value;
        volatile HashEntry<K,V> next;
        
        final void setNext(HashEntry<K,V> n) {
            UNSAFE.putOrderedObject(this, nextOffset, n);
        }
        // Unsafe mechanics
        static final sun.misc.Unsafe UNSAFE;
        ...
    }
}
```

Segment数组的作用就是把一个大的table分割成多个小table来加锁，在每个Segment中，存储的是数组+链表的结构。
![ConcurrentHashMap数据结构](https://img-blog.csdnimg.cn/20200929172150845.png?size_16,color_FFFFFF,t_70#pic_center)


#### 1.2 数组长度的定义

Segment数组的长度ssize是通过concurrencyLevel计算出来的，默认情况下传入的concurrencyLevel为DEFAULT_CONCURRENCY_LEVEL=16。ssize由1开始循环左移，最终得到一个大于等于concurrencyLevel的最小的2的N次幂值。

参数initialOneCapacity是ConcurrentHashMap的初始容量，默认情况下传入的是DEFAULT_INITIAL_CAPACITY=16。由initialOneCapacity/concurrencyLevel计算出Segment中HashEntry数组的大小，同样，HashEntry数组大小最小为2，当initialOneCapacity/concurrencyLevel大于2时，则从2开始循环左移，最终得到一个大于等于initialOneCapacity/concurrencyLevel且最小为2的2的N次幂值。

数组长度都须保证为2的N次幂是为了能通过按位与的散列算法来定位数组的索引。

全局变量segmentShift和segmentMask是在散列算法中使用，sshift记录ssize由1左移的次数。segmentShift为32-sshift，用于定位参与散列运算的位数，32是因为key的hashCode是32位的int类型。segmentMask为ssize-1，ssize是2的N次幂，所得结果的二进制值各位均为1，作为散列运算的掩码。

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    //循环，ssize从1开始左移直到满足while条件
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    //segmentShift用于定位参与散列运算的位数
    this.segmentShift = 32 - sshift;
    //segmentMask是散列运算的掩码，掩码的各位都是1
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    //计算Segment对象的HashEntry数组长度，最小为2
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    //创建Segment实例作为segments[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    //创建segment数组
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    //通过Unsafe将segments[0]保存到segment数组
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```

#### 1.3 put操作

执行put操作时，会进行第一次key的hash来定位Segment的位置，如果该Segment还未初始化，则通过CAS操作进行赋值。
```java
public V put(K key, V value) {
    Segment<K,V> s;
    //value为null时会抛异常，故不能存放null值
    if (value == null)
        throw new NullPointerException();
    //计算出需存放在segments数组的索引
    int hash = hash(key.hashCode());
    int j = (hash >>> segmentShift) & segmentMask;
    //获取到对应的Segment
    if ((s = (Segment<K,V>)UNSAFE.getObject       //nonvolatile;recheck
         (segments, (j << SSHIFT) + SBASE)) == null) // in ensureSegment
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```
##### 1.3.1 定位Segment

hash(key.hashCode())，传入的key.hashCode是32位的int类型，把得到的hashCode进行一次再散列，目的是减少散列冲突，使元素能均匀地分步在不同的Segment上。
```java
private static int hash(int h) {
    // Spread bits to regularize both segment and index locations,
    // using variant of single-word Wang/Jenkins hash.
    h += (h <<  15) ^ 0xffffcd7d;
    h ^= (h >>> 10);
    h += (h <<   3);
    h ^= (h >>>  6);
    h += (h <<   2) + (h << 14);
    return h ^ (h >>> 16);
}
```
通过以上的再散列过程，可以将每一位的数据都散列开，将高位和低位的特征混合起来，让每一位都参加到散列运算中，从而减少散列冲突。
再散列得到的hash执行``(hash >>> segmentShift) & segmentMask``，把hash的二进制值无符号右移segmentShift位，再与segmentMask相与，得到的结果实际就是保存了hash的高N位。
通过Unsafe.getObject可以获取到对应索引的Segment，UNSAFE.getObject传入的第二个参数``(j << SSHIFT) + SBASE``，SSHIFT和SBASE分别通过Unsafe的native方法arrayIndexScale和arrayBaseOffset得到，这两个方法分别返回数组中元素的增量地址及数组的基地址，把arrayBaseOffset与arrayIndexScale配合使用，可以定位数组中每个元素在内存中的位置。
如果获取到的segment为null，说明还未创建，则调用ensureSegment方法创建Segment对象并保存到Segment数组。

##### 1.3.2 Segment的put方法

Segment继承了ReentrantLock，也就带有了锁的功能，调用Segment的put方法时，就会利用锁的属性来尝试锁定具体的put操作。
第一步调用tryLock方法尝试获取锁，如果获取失败说明存在竞争，则利用scanAndLcokForPut方法来自旋获取锁，自旋超过指定次数就调用阻塞锁获取``lock()``，保证锁能获取成功。
第二步进行第二次hash操作``(tab.length - 1) & hash``，保留hash的低N位即为索引值，通过Unsafe到内存获取对应索引位置的HashEntry，若HashEntry还未创建则返回null。
若是调用了scanAndLockForPut方法，该方法会先通过Unsafe到内存获取对应索引位置的HashEntry，若HashEntry还未创建则先创建一个HashEntry对象，并在最终获取到锁时返回。

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    //将当前Segment中的table通过key的hashCode定位到HashEntry
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        //获取存放在HashEntry数组的索引位置
        int index = (tab.length - 1) & hash;
        //Unsafe获取到对应的HashEntry
        HashEntry<K,V> first = entryAt(tab, index);
        //遍历链表
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                //存在键值相同的元素，则更新其value值
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            //HashEntry仍未创建，也可能遍历完链表找不到键值相同的元素
            else {
                //在scanAndLockForPut方法中已创建
                if (node != null)
                    node.setNext(first);
                //创建新的HashEntry，头插法插入链表
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                //Segment的元素个数超过了threshold，需扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                //把新的HashEntry保存到table数组中
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

ConcurrentHashMap的写操作只对元素所在的Segment进行加锁，不会影响到其他Segment，而Segment的写操作以外则通过Unsafe类进行操作来保证线程安全。所以在最理想的情况下，ConcurrentHashMap可以最高同时支持Segment数量大小的写操作。

##### 1.3.3 Segment中HashEntry数组的扩容

上面的put操作中，在将新的HashEntry保存到数组前，会先判断Segment中HashEntry的数量是否超过了threshold，是则调用rehash方法进行扩容。扩容后的新数组长度为原数组的两倍。
```java
private void rehash(HashEntry<K,V> node) { 
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    //新数组容量扩大为原来的2倍
    int newCapacity = oldCapacity << 1;
    //threshold等于容量*负载因子
    threshold = (int)(newCapacity * loadFactor);
    //创建新数组
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    //遍历旧数组
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;
            //单元素节点，直接保存到新数组
            if (next == null)   //  Single node on list
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                //遍历链表，找到最后一个同上一个索引值不同的节点
                //即如果链表后半截的新索引都相同，则整段挪到新数组位置即可
                for (HashEntry<K,V> last = next;
                    last != null;
                    last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
                //遍历链表还没挪动的前半截，单个移到新数组对应位置
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    //把put操作的待插入/待更新节点插入/更新
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```

为了高效，ConcurrentHashMap不会对整个容器进行扩容，而只会对某个segment进行扩容。

#### 1.4 get操作

ConcurrentHashMap的get操作比较简单，只需通过Unsafe类的方法获取到目标Segment，并遍历其HashEntry数组即可。

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    //定位Segment索引位置
    int h = hash(key.hashCode());
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    //获取目标Segment
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        //遍历Segment的HashEntry数组
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

#### 1.5 size操作

size操作需要统计整个ConcurrentHashMap中元素的个数，即要统计所有Segment中元素的个数之和。
Segment的全局变量count是Segment内部的元素计数器，虽然count声明为volatile，但多线程环境下，可能在count累加前使用了count，导致最终统计结果不准确。
最安全的做法当然是锁住所有的写操作，但这种方法比较低效。

因为在累加count操作过程中，之前累加过的count发生变化的几率非常小，所以 ConcurrentHashMap的做法是迭代1次+重试2次通过不锁住Segment的方式来统计各个Segment大小。这三次无锁的迭代统计中，如果连续两次迭代统计过程modCount没有发生变化，则说明在统计过程中容器的count没有发生变化，所得的size即为准确的size，即可返回结果；如果统计的过程中，前后两次得到的modCount不同，则容器的count发生了变化，所得size无效，重试两次都失败则采用加锁的方式来统计所有Segment的大小。

```java
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            //已经重试了两次（第一次迭代不算重试），则需采用加锁的方式了
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    //Segment[j]为空时强制创建
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            //遍历Segment数组
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    //统计modCount
                    sum += seg.modCount;
                    int c = seg.count;
                    //统计size
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            //前后两次的modCount统计结果相同，说明所得size有效，直接break返回
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```

### 2. jdk 1.8的ConcurrentHashMap

jdk1.8的ConcurrentHashMap抛弃了分段锁Segment的设计，是在jdk1.8的HashMap的基础上增加``CAS + synchronized``来实现线程安全的，其底层数据结构也是数组+链表和红黑树。
synchronized锁的粒度为每个数组的元素Node。
ConcurrentHashMap的全局变量都被声明为``volatile``，保证可见性。
```java
transient volatile Node<K,V>[] table;
//扩容时的新数组，在非扩容时为null
private transient volatile Node<K,V>[] nextTable;
//用于记录table中的元素个数
private transient volatile long baseCount;
//用于table数组的初始化和扩容控制
//为负值时，表示table正在初始化或正在扩容：-1表示正初始化，-(1+N)表示有N个线程正在执行扩容操作
//其他情况：table未初始化时，保存table的初始化容量或默认为0；table初始化后，保存table的容量，默认是table大小的0.75倍
private transient volatile int sizeCtl;
//多线程竞争时，CAS操作尝试累加baseCount失败时，会将累加量存储到counterCells中
private transient volatile CounterCell[] counterCells;
```
#### 2.1 put操作

jdk1.8的ConcurrentHashMap元素的键值都不能为null，为null时抛空指针异常。
- 第一次调用put方法时，table还未初始化，此时才调用initTable方法初始化table数组，在创建table数组前会通过CAS操作将sizeCtl更新为-1。
- 如果对应索引位置的Node节点的hash值为MOVED，为-1，则需调用helpTransfer方法进行扩容。
- 如果对应索引位置的Node节点既不为空也无需扩容，则利用synchronized锁住该Node节点，执行节点value更新或新节点尾插法插入链表或插入红黑树。插入链表后，若链表长度超过阈值，则需转换为红黑树。
- ps：table是由volatile修饰的，为什么还需通过Unsafe类获取其中的元素节点？
java的内存模型中，每个线程都有一个工作内存，里面存储着table的副本，只能保证table引用的可见性，并不能保证table中每个元素的可见性，所以若直接通过table[index]无法保证线程每次拿到的都是table中的最新元素。Unsafe类可直接获取指定内存的数据，保证每次拿到的数据都是最新的。
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //键值为空时，抛空指针异常
    if (key == null || value == null) throw new NullPointerException();
    //调用spread方法进行再hash
    int hash = spread(key.hashCode());
    int binCount = 0;
    //for循环直至元素插入成功
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //table还未初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //Unsafe类获取索引位置对应的Node，若为空则创建新的Node并CAS更新
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;              // no lock when adding to empty bin
        }
        //若当前索引位置的hash == MOVED，说明当前有线程正在扩容
        else if ((fh = f.hash) == MOVED)
            //helpTransfer方法会检查是否需要当前线程帮忙扩容，是则调用transfer方法帮忙执行扩容数组转移
            tab = helpTransfer(tab, f);
        //获取到的Node非null且不处于扩容转移状态，则需对获取到的Node加锁
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                   //当前是链表（红黑树根节点的hash值为-2）
                   if (fh >= 0) {
                        binCount = 1;
                        //遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //找到键值相同的节点，则更新其value
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            //遍历到链表尾，则尾插法插入新的Node结点
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //当前是红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                      value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (binCount != 0) {
                //若链表长度超过TREEIFY_THRESHOLD，则需转换为红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //更新baseCount值，同时会检查是否需要扩容
    addCount(1L, binCount);
    return null;
}
```

##### 2.1.1 table元素定位

ConcurrentHashMap通过调用``spread``方法把获取到的``key.hashCode``进行再hash，spread方法的执行逻辑比较简单，就是将``key.hashCode``的高16位和低16位相与，再取其低31位即为再hash结果。
把再hash结果同table数组长度进行取模运算：``(n - 1) & hash``，n是table数组的长度，是2的N次幂，n-1的二进制值各位均为1，所以实际是保留了hash的后N位。
```java
static final int HASH_BITS = 0x7fffffff; //usable bits of normal node hash
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
##### 2.1.2 扩容

什么情况下会触发扩容？在put方法最后，会调用addCount方法来更新baseCount的值，猜测扩容在该方法中触发。
新增节点之后，会调用addCount方法记录元素个数，并检查是否需扩容，当数组元素个数达到阈值时，会触发transfer方法，重新调整节点的位置。

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    //若as非空或CAS更新baseCount失败，则会把待增加的x值保存到CounterCell中
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        //as为空或CAS更新as数组某个对象的value值失败，则调用fullAddCount存储待增加的值
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    //check >= 0表示需要进行扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        //while循环，直至扩容到满足s的要求
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null && (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            //sc为负值，说明正在初始化或有线程正在扩容
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || (nt = nextTable) == null || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            //sc非负，则CAS操作更新sizeCtl的值，更新成功后，当前线程调用transfer发起扩容操作
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                            (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```
另外，在链表转换为红黑树的treeifyBin方法中，会先检查当前数组长度是否超过64，若还未超过，则先调用tryPresize方法，该方法会调用transfer方法进行扩容。

###### 2.1.2.1 sizeCtl的含义

这里可能有人会有疑问，前面不是提到sizeCtl的值为``-(N+1)``表示有N个线程在参与扩容吗？那addCount方法中调用transfer方法前更新sizeCtl值为什么是``sc+1``而不是``sc-1``呢？
对于这个问题，我们先看一下sizeCtl非负时，线程调用transfer方法前将sizeCtl更新为``rs << RESIZE_STAMP_SHIFT) + 2``，其中，``RESIZE_STAMP_SHIFT``及``rs``的值定义如下：

```java
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
private static final int RESIZE_STAMP_BITS = 16;
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

假设当前数组长度n为16，Integer为32位，``Integer.numberOfLeadingZeros(n)``表示16的二进制值前面有27个0，27对应二进制为``11011``，``(1 << (RESIZE_STAMP_BITS - 1)``把1左移15位，所以``rs = resizeStamp(16)``最终所得二进制值为``1000 0000 0001 1011``，再把rs左移16位，得``1000 0000 0001 1011 0000 0000 0000 0000``，即把低16位都置为0，再+2，最终更新的sizeCtl的二进制值即为``1000 0000 0001 1011 0000 0000 0000 0010``，如果只取sizeCtl的二进制值的低16位的数值M，此时扩容线程确实是``M-1=1``个线程在扩容。
sizeCtl值为负时，我们假设当前sizeCtl值为上面的``1000 0000 0001 1011 0000 0000 0000 0010``，更新sizeCtl的值为sizeCtl+1，得二进制值``1000 0000 0001 1011 0000 0000 0000 0011``，只取sizeCtl的二进制的低16位的数值M，此时扩容线程是``M-1=2``个线程在扩容。
所以，准确来讲，sizeCtl为负时，应将其二进制值分成两部分，低16位的数值M，表示当前有``M-1``个线程在执行扩容，高16位除符号位的数值与数组长度n相关，其值等于``Integer.numberOfLeadingZeros(n)``。

###### 2.1.2.2 transfer方法

transfer方法执行具体的扩容操作，整个过程有点复杂，主要涉及到多线程并发扩容的处理，主要执行流程如下：
- 创建新的扩容后的数组nextTable，创建ForwardingNode对象，该对象是控制并发扩容的核心，一旦旧数组的某个索引位置的节点已被成功转移到新数组，则会将旧数组的该位置替换成ForwardingNode对象，其他线程访问到ForwardingNode对象，就可知道该位置节点已被转移。
- for循环遍历旧数组，其中while循环用于确定待处理节点的索引，主要用于提高扩容数组转移的并发度，比如，旧数组的长度为32，则第一个线程会从索引位置31开始递减遍历，而第二个线程会从索引位置15开始递减遍历。
- 检查索引``i``的合法性，若非法，判断扩容操作是否已执行完成。
- 获取索引位置``i``的对应节点，分三种情况：
  1. 为空时则直接将ForwardingNode对象插入旧数组的该位置，通知其他线程该位置已被处理；
  2. 已被替换为ForwardingNode节点，表示该节点已被处理；
  3. 非空且非ForwardingNode，则需上锁处理。遍历该位置的链表或红黑树，将其中节点分为两部分，再分别将两部分保存到新数组的对应索引位置，最后将旧数组的对应索引位置替换成ForwardingNode对象。

<font color="#b22222">处理链表或红黑树时为什么需要拆分成两部分？</font>
我们知道，一个Node在table数组下标位置的确定是通过``(n - 1) & hash``，hash是由``spread(key.hashCode())``计算出来的，扩容前后都是一样的，而``n-1``的二进制值在扩容后高位多了一个1，比如旧数组长度为16，扩容后新数组长度即为32，``n-1``的二进制值由``01111``变成``11111``，扩容后的``(n - 1) & hash``结果也由保留低四位变成保留低五位。
如果``hash``的二进制值的左数第五位为0，则``(n - 1) & hash``结果同扩容前相同；若为1，则``(n - 1) & hash``的结果比扩容前大了``10000``，即大了16，即为扩容前的数组长度。
所以，节点在旧数组的索引位置为``i``，那么扩容后该节点在新数组索引位置要么为``i``，要么为``i+旧数组长度``。

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //当前可用处理器中，每核处理的量小于16，则强制赋值16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            //创建nextTab数组，容量为原来的两倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        //nextTable指向新数组
        nextTable = nextTab;
        //transferIndex初始值为原数组长度
        transferIndex = n;
    }
    int nextn = nextTab.length;
    //创建ForwardingNode对象，其中，fwd.hash=-1，fwd.nextTable=nextTab
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    //advance为true，表明该节点已经处理过了
    boolean advance = true;
    //finishing为true，表示扩容完成
    boolean finishing = false; // to ensure sweep before committing nextTab
    //遍历旧数组，bound表示需要处理的数组边界
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        //advance默认为true，while循环CAS操作尝试设置transferIndex，bound和i的值，选取要处理的节点
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //CAS操作设置transferIndex的值为transferIndex-stride
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                //假设旧数组长度为16，stride也为16，
                //则bound被设为0，i为15，表示先处理数组下标为15的节点
                //下一次再进入while循环，会满足第一个if条件，所以i会递减，15-->14-->13-->...依次处理
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        //索引值i非法，可能已处理完成？
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //已经完成所有节点的赋值了，更新全局变量后return结束
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            //CAS操作更新sizeCtl的值，sizectl值减一，该线程完成扩容任务的执行，扩容执行线程数减1
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        //遍历到的节点为null，则直接将旧数组对应索引位置替换为ForwardingNode，告诉其他线程该节点已处理
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        //遍历到的节点为ForwardingNode，表示该节点已被处理过
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        //遍历到的节点既不为null也非ForwardingNode，则需上锁处理
        else {
            //锁住该节点
            synchronized (f) {
                //先检查索引i位置节点是否仍为前面拿到的节点，若不是，说明可能已被处理或被修改，需重新获取节点
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    //节点为链表节点
                    if (fh >= 0) {
                        //runBit只有两种可能：0或n
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        //遍历链表，最终把链表分成两截，后半截从lastRun开始，且节点的hash&n值相同
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        //ln保存hash&n==0的节点，hn保存hash&n==n的节点
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        //再遍历链表的前半截，分别链入到ln和hn
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //ln链表节点在新数组的索引位置仍为i
                        setTabAt(nextTab, i, ln);
                        //nh链表节点在新数组的索引位置为i+n
                        setTabAt(nextTab, i + n, hn);
                        //节点处理完成，把旧数组对应索引位置替换为ForwardingNode，通知其他线程该节点已处理
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    //节点为红黑树节点
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        //遍历树节点，同样拆分成两棵树
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        //树节点小于阈值时则转换为链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        //节点插入到新数组
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        //插入ForwardingNode，表示节点处理完成
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```
#### 2.2 size操作

ConcurrentHashMap中有一个声明为volatile的全局变量baseCount记录元素的个数 ，当插入新数据或删除数据时，会通过addCount方法更新baseCount的值。

在addCount方法中会通过CAS操作尝试更新baseCount的值，若存在竞争，CAS操作失败，则会初始化一个长度为2的CounterCell数组，用于存储没能保存到baseCount的值。CounterCell数组也会扩容，每次扩容为原来的两倍，从而提高CAS操作执行的成功率，提高并发性。

所以CounterCell数组就是用于多线程更新baseCount时，保存部分元素的变化个数。所以通过累加baseCount和CounterCell数组中的数量，即可得到元素的总个数。

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
         (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
         (int)n);
}
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

