### 1. 悲观锁&乐观锁

#### 1.1 悲观锁

总是以悲观的态度来对待资源竞争，每次去获取资源时都会认为其他线程会修改资源，所以在每次获取资源时都会上锁，在该线程释放锁之前任何其他线程都不允许对其资源进行操作，其他线程需要阻塞直到拿到了锁。一般数据库的本身锁的机制都是基于悲观锁的思想实现的，比如行锁，表锁，写锁等；java中ReentrantLock和synchronized都是基于悲观锁实现的。

#### 1.2 乐观锁

以乐观的态度来对待资源竞争，每次去获取资源时都会认为资源不会被修改，所以也不会上锁，只有在需要更新数据的时候才会通过某种机制来判断数据是否存在冲突（一般是通过加版本号，进行版本号的对比来实现）。乐观锁适用于多读的应用，可提高吞吐量。java中的原子包就是通过乐观锁（CAS操作+版本机制）来实现的。

对于多写的场景，如果使用乐观锁，会导致应用上层需要不停的retry，这样反而会降低性能，所以多写的场景更适合使用悲观锁。

#### 1.3 悲观锁和乐观锁的调用示例

悲观锁基本都是在显示的锁定之后再操作同步资源，而乐观锁则直接去操作同步资源。

乐观锁不锁定同步资源也可以正确实现线程同步就依赖于CAS操作了。

```java
//悲观锁
private ReentrantLock lock = new ReentrantLock();
public void setState() {
    lock.lock();
    ......
    lock.unlock();
}
//乐观锁
private AtomicInteger atomicInteger = new AtomicInteger();
atomicInteger.incrementAndGet();
//incrementAndGet的具体执行内容
public final int incrementAndGet() {
    return U.getAndAddInt(this, VALUE, 1) + 1;//U为Unsafe类
}
```

#### 1.4 CAS算法

compare and swap，是一种无锁算法，通过CAS操作实现，即在不使用锁的情况下实现多线程之间的变量同步。

CAS操作是原子操作的一种，可在多线程中实现不被打断的数据交换操作，避免多线程同时改写某一数据时由于执行顺序的不确定性以及中断等导致数据的不一致性问题。

CAS算法涉及三个操作数：需要读写的内存值V，进行比较的值A，要写入的新值B。

当且仅当V的值等于A时，CAS通过原子方式用新值B来更新V的值（比较+更新整体是一个原子操作），否则不会执行任何操作。一般情况下，“更新”是一个不断重试的操作。

##### 1.4.1 CAS操作为何能保证变量同步？

之前一直不能理解，CAS算法明明是比较+修改，是两个操作呀，为啥就能保证变量同步了？

其实，java的CAS操作是通过sun.misc包下的一个类Unsafe来实现，Unsafe底层实际上是调用C代码，使得java语言用于类似C语言指针一样操作内存空间的能力。C代码调用汇编，最后生成一条cmpxchg的CPU指令，一条CPU指令是不会被打断的，所以CAS操作是原子的。

##### 1.4.2 CAS操作存在的问题

CAS操作虽然高效，但也存在三大问题：

1. ABA的问题，该问题可通过版本号来解决。所以最后的乐观锁就是由CAS操作+版本机制来实现的。 jdk从1.5开始提供了AtomicStampedReference类来解决ABA问题，具体操作封装在compareAndSet方法中。compareAndSet方法首先检查当前引用是否等于预期引用，并检查当前标志与预期标志是否相等，都相等则以原子的方式将该引用和该标志的值设置为给定的更新值。
2. 循环时间长开销大，自旋CAS如果长时间不成功，则会给CPU带来非常大的执行开销。
3. 只能保证一个共享变量的原子操作。



### 2. 自旋锁&适应性自旋锁

阻塞或唤醒一个线程需要OS切换CPU状态来完成，这种状态切换需要耗费处理器时间。同步代码块比较简单时，状态切换带来的开销可能还大于代码块的执行开销。

#### 2.1 自旋锁

为减少这部分状态切换带来的开销，会让当前线程进行自旋，如果在自旋过程中持有锁的线程释放了锁，当前线程就可获取锁并访问同步资源，而无需进入阻塞状态，从而避免了线程切换的开销。

自旋等待的过程是要占用处理器时间的，如果锁被占用的时间很短，那么自旋等待的效果就非常好；如果锁被占用的时间很长，那么自旋的线程就会白浪费处理器资源。所有自旋等待的时间、次数必须要有一定的限度，超过这个先对，就应当挂起线程。

自旋锁的实现原理也是CAS，AtomicInteger调用的Unsafe中的getAndAddInt方法就是通过do-while循环来自旋的。

#### 2.2 自适应自旋锁

自适应则说明自旋的时间（次数）非固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果某个锁，自旋很少成功获得过，则以后尝试获取锁时可能省略自旋过程，直接阻塞线程，避免浪费处理器资源。

在自旋锁中 另有三种常见的锁形式:TicketLock、CLHlock和MCSlock。



### 3. 无锁&偏向锁&轻量级锁&重量级锁

synchronized锁性能的优化，根据当前竞争激烈程度进行锁的升级，详细可看synchronized的原理部分。



### 4. 公平锁&非公平锁

公平锁指多个线程按照申请锁的顺序来获取锁，线程直接进入队列中排队等待唤醒。公平锁的优点是等待的线程不会饿死，缺点是整体吞吐效率相对非公平锁要低，CPU唤醒阻塞线程的开销比非公平锁大。

非公平锁是多个线程加锁时直接尝试获取锁，尝试一定次数失败后才到等待队列进入阻塞状态。如果线程在尝试过程中刚好获取到了锁，就不需要CPU去唤醒线程了，减少了唤醒的开销。缺点是等待的线程可能会饿死。

ReentrantLock有一个内部类Sync，Sync继承AQS（AbstractQueuedSynchronizer），添加锁和释放锁的大部分操作实际都是在Sync类中实现的，它有FairSync和NonfairSync两个子类。ReentrantLock默认使用非公平锁。

```Java
//公平锁的获取
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
             compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
//非公平锁的获取
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

ReentrantLock中公平锁和非公平锁获取的唯一区别就在于公平锁在获取同步状态时多了一个限制条件：hasQueuedPredecessors。

hasQueuedPredecessors方法主要做一件事情：主要是判断当前线程是否位于同步队列中的第一个，是则返回true，否则返回false。

```Java
public final boolean hasQueuedPredecessors() {
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
         ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

所以公平锁就是通过同步队列来实现多个线程按照申请锁的顺序来获取锁。



### 5. 可重入锁&不可重入锁

可重入锁，又称递归锁，是指同一个线程在获取锁后，再进入该锁的其他代码块时自动获得锁，而不会因为已获得过锁但还没有释放而阻塞。java的ReentrantLock和synchronized都是可重入锁，可重入锁可在一定程度上避免死锁。

#### 可重入锁为什么在嵌套调用时可获得锁？

以synchronized为例
synchronized在执行monitorenter指令获取锁的过程中，会根据锁对象头、LockRecord等信息判断是否是当前线程持有锁，若是，则会通过添加空的Lock Record或计数标记等记录锁的重入次数。

非可重入锁在重复调用同步资源时会出现死锁
非可重入锁：以前可能有个NonReentrantLock类，存在锁嵌套使用的可能也可理解为非可重入锁。

### 6. 独享锁&共享锁

独享锁也叫排他锁，该锁一次只能被一个线程持有。若线程T对数据A加上排它锁后，则其他线程不能在对A加任何类型的锁。获得排他锁的线程既能读数据又能修改数据。

共享锁指该锁可被多个线程所持有，若线程T对数据A加上共享锁后，则其他线程只能对A加共享锁，不能加排它锁。获得共享锁的线程只能读数据，不能修改数据。

java中的ReentrantReadWriteLock有两把锁：ReadLock和WriteLock。这两个锁的锁主体都是Sync，Sync是AQS的一个子类，读锁和写锁的加锁方式不一样，读锁是共享锁，写锁是独享锁。

独享锁的state值通常为0或1（可重入锁则为重入的次数），在共享锁中state值则为持有锁的线程数量。

ReentrantReadWriteLock中有读、写两把锁，所以需要在一个整型变量state上分别描述读锁和写锁的数量，于是将state变量按位切割切分成了两个部分，高16位表示读锁状态，低16位表示写锁状态。
