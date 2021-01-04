### 1. HashMap内部数据结构
jdk1.8的HashMap内部使用的数据结构是数组+链表/红黑树：


![HashMap数据结构](https://img-blog.csdnimg.cn/20200912144948473.png)

TREEIFY_THRESHOLD表示链表转红黑树的阈值，当链表的长度超过该值时，就将链表转换为红黑树。
UNTREEIFY_THRESHOLD表示红黑树转链表的阈值，该值为6而不是8，这样设计的原因是``为避免hash碰撞次数在8左右徘徊时，频繁地在链表和红黑树之间转换``。
```java
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;

transient Node<K,V>[] table;
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    ......
}
```
#### 1.1 为什么链表转红黑树的阈值为8？
这个问题在源码中其实是有解释：``TreeNode占用空间是普通nodes的两倍``，所以我们只在bins中包含了足够多的节点时才转换成TreeNodes，当bins中节点数变少时，又会转换成普通nodes。
如果hashCode的分散性很好，使用到TreeNodes的概率会很小，因为元素分散均匀，几乎不会有bin中的链表长度达到阈值。理想情况下随机hashCode算法下所有bin中节点的分布频率会遵循泊松分步，一个bin中元素个数达到8的概率是0.00000006，小于百万分之一，bin中元素个数超过8的概率小于千万分之一，几乎都是不可能事件。
所以，阈值设为8的原因可总结为：在hashCode分散均匀时，单个bin元素个数几乎不可能达到8，也就不会触发转换为空间更大的TreeNode；而hashCode分散不均匀时，出于查找时间的考虑，才将单个bin元素个数达到8的转换为红黑树。``说白了，其实就是对空间和时间的一种权衡策略``。
```java
    /*
     * Because TreeNodes are about twice the size of regular nodes, we
     * use them only when bins contain enough nodes to warrant use
     * (see TREEIFY_THRESHOLD). And when they become too small (due to
     * removal or resizing) they are converted back to plain bins.  In
     * usages with well-distributed user hashCodes, tree bins are
     * rarely used.  Ideally, under random hashCodes, the frequency of
     * nodes in bins follows a Poisson distribution
     * (http://en.wikipedia.org/wiki/Poisson_distribution) with a
     * parameter of about 0.5 on average for the default resizing
     * threshold of 0.75, although with a large variance because of
     * resizing granularity. Ignoring variance, the expected
     * occurrences of list size k are (exp(-0.5) * pow(0.5, k) /
     * factorial(k)). The first values are:
     *
     * 0:    0.60653066
     * 1:    0.30326533
     * 2:    0.07581633
     * 3:    0.01263606
     * 4:    0.00157952
     * 5:    0.00015795
     * 6:    0.00001316
     * 7:    0.00000094
     * 8:    0.00000006
     * more: less than 1 in ten million
     */
```
无参创建HashMap时，默认的初始化容量为16，负载因子为0.75。无参构造函数的容量值只有在第一次调用put方法时才会被赋值。
```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```
若创建HashMap时传入了自定义的容量值cap，则初始容量值会被调整为大于cap的2的整数次方，比如传入10，通过调用tableSizeFor函数将10调整为16赋为容量初始值。
```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
#### 1.2 容量值的调整过程
假设我们传入的初始容量值为80（二进制为0101 0000），调用tableSizeFor方法的计算过程如下：
| 计算过程 | 计算结果 |
| --- | --- |
| 0101 0000 >>>1（逻辑右移1位） | 0010 1000 |
| 0101 0000<br>&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#124;=位或<br>0010 1000 | 0111 1000 |
| 0111 1000 >>>2（逻辑右移2位） | 0011 1100 |
| 0111 1000<br>&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#124;=位或<br>0011 1100 | 0111 1100 |
| 0111 1100 >>>4（逻辑右移4位） | 0000 0111 |
| 0111 1100<br>&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#124;=位或<br>0000 0111 | 0111 1111 |
| 0111 1111 >>>8（逻辑右移8位） | 0000 0000 |
| 0000 0000<br>&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#8195;&#124;=位或<br>0111 1111 | 0111 1111 |
由上面的计算过程可以看出，就是不断将高位的1不断右移，再同自身位或，把后面的0都变成1，全1的二进制再加上1，得到的即为2的整数次幂。

### 2. put方法
接着简单分析一下put方法的执行过程：
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //无参创建HashMap，第一次put，table为null，调用resize方法，创建一个默认容量的Node数组
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //(n-1)&hash即为待插入的数组下标，该位置为null时则直接创建新的Node结点并保存到数组中
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    //该位置非空，表示发生hash碰撞
    else {
        Node<K,V> e; K k;
        //若p结点的hash值及key值相等，则更新p结点的value值
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //若p结点为树结点，表示当前为红黑树则调用putTreeVal方法继续执行
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //当前仍为链表    
        else {
            //遍历链表
            for (int binCount = 0; ; ++binCount) {
                //遍历完链表，没找到key值相同的结点，则尾插法将其插入链表
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //如果当前链表的结点数超过了阈值，则调用treeifyBin方法将链表转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //找到key值相同的结点，则直接更新该结点对应的value值
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //检查是否需要扩容，是则调用resize方法
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
ps：在treeifyBin方法中，还会做一次判断，如果table数组的长度小于64，则会进行扩容。所以链表转换为红黑树有两个条件：一是链表长度超过阈值8，二是table数组的长度大于等于64。
### 3. HashMap的hash方法
HashMap中是通过```(n - 1) &  hash```确定待插入元素在table数组中存放的位置下标，hash是调用HashMap的哈希函数```hash(key)```返回的hash值。
```hash(key)```是将拿到的32位int类型的```key.hashCode```的高16位和低16位进行异或操作得到的。这个hash方法也叫扰动函数，这么设计的主要原因就是要使拿到的hash值尽可能的分散，尽可能的降低hash碰撞。
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
32位int型的散列值，只要哈希函数映射得均匀，一般是很难出现碰撞的。但由于数组长度有限，需要对数组长度进行取模运算，这里的取模运算就是```(n - 1) &  hash```，```&```运算的运算速度更快。
而HashMap的数组长度均为2的整数次幂，```(n-1)```即为后几位都是1，所以```&```运算的结果就是只保留了散列值的后几个低位值，即为存放元素的数组下标。
只保留了散列值的后几个低位值，如果散列函数正好让后几个低位值呈现规律性的重复，hash碰撞就变得十分严重。所以把hashCode的高16位和低16位相异或，把其高16位和低16混合起来，低位中保存了高位的部分特征，从而加大了低位的随机性，降低hash碰撞的发生。

### 4. HashMap的扩容
在将元素存放到对应位置后，会检查是否需要扩容，若是，调用```resize```方法进行扩容。
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    //当前table容量大于0，说明是要进行扩容的，否则，说明是table数组还未初始化
    if (oldCap > 0) {
        //当前容量已达最大容量值
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //当前容量值右移一位，即新容量值为旧容量值的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //创建HashMap时若指定了容量值，则会把调整后的容量值保存在threshold中
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //无参创建HashMap，则设为默认容量值
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    //threshold存储的是容量值*负载因子
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    //创建新的Node数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    //保存到全局变量table中
    table = newTab;
    if (oldTab != null) {
        //遍历旧table
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //单元素结点，直接保存到新数组
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                //树节点，当前是红黑树，则调用TreeNode.split
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //链表
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    //遍历链表，链表中元素对应新的数组下标有两种可能，故拆分成两个链表
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //把拆分成的两个链表保存到对应的数组位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```
resize方法进行扩容，新数组容量是旧数组的2倍，元素在数组中下标位置的确定是通过```hash(key) & (n-1)```，由于扩容后的新数组长度为原来的2倍，所以对应的n-1就是高位比原先多了一个1，```hash(key) & (n-1)```在扩容后的结果就是保留的位数多了一位，比如原数组长度为16，扩容后即为32位，```hash(key) & (n-1)```的结果由保留低四位变成了保留低5位，如果hash(key)的低5位为0，则```hash(key) & (n-1)```的结果同扩容前一致，如果hash(key)的低5位为1，则```hash(key) & (n-1)```的结果比扩容前大了1 0000，即为16，旧容量值。
所以原数组元素在新数组中的位置，要么仍为原索引值，要么是原索引值+旧数组容量值。

### 5. jdk1.8 HashMap的优化
jdk1.8对比jdk1.7，HashMap所做的优化：

- 由数组+链表改成了数组+链表/红黑树，红黑树的时间复杂度为O(logn)；
- 元素插入链表时，由头插法改成了尾插法，解决了多线程竞争可能导致链表死循环的问题；
- 由先扩容再插入改为插入之后判断是否需扩容；
- jdk1.7扩容后需对原数组的元素进行重新hash定位在新数组中的位置，而jdk1.8则直接判断是原索引值还是原索引值+旧数组容量值。

jdk1.8的HashMap仍是非线程安全的，在多线程环境下仍有数据覆盖、死循环等问题。
jdk1.8的HashMap虽使用尾插法解决了链表出现环的死循环问题，但死循环问题依然存在，只不过该问题是出现在红黑树上，测试时发现多线程环境下有3个地方会出现死循环（测试覆盖不全，可能还有其他情况会导致死循环），这三种死循环出现的原因都是：多线程插入树节点导致树节点之间相互引用，遍历时出现死循环。

- 一是调用putTreeVal方法把节点插入红黑树时，调用root方法导致死循环；
  ![treeify](https://img-blog.csdnimg.cn/20200914152750184.jpg)

- 二是扩容时调用TreeNode的split->treeify导致死循环；![balanceInsertion](https://img-blog.csdnimg.cn/20200914152750180.jpg )

- 三是扩容时调用TreeNode的split->treeify->balanceInsertion导致死循环。![root](https://img-blog.csdnimg.cn/20200914152750142.jpg)

>参考： [一个HashMap跟面试官扯了半个小时](https://mp.weixin.qq.com/s?__biz=MzI3ODA0ODkwNA==&mid=2247483674&idx=1&sn=fe1a7f6652e44a60444424207b87cc93&scene=21#wechat_redirect)、[HashMap常见问题](https://blog.csdn.net/z1c5809145294zv/article/details/105726296?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.channel_param)、[HashMap链表转红黑树的阈值是8？](https://blog.csdn.net/wo1901446409/article/details/97388971)
