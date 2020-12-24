### 1. CountDownLatch

CountDownLatch可用于某个线程等待其他一个或多个线程完成操作后再继续执行，它的作用类似于线程的join。
join用于让当前执行线程等待join线程执行结束，其实现原理就是不停检查join线程是否存活，如果仍存活则当前线程继续等待，否则会调用线程的notifyAll方法唤醒其他线程。

CountDownLatch类的主要方法：

```java
//参数count即为计数值，对应countDown方法需要被调用的次数
public CountDownLatch(int count) {...}
//调用await方法的线程会进入阻塞状态，直到count值为0时才被唤醒继续执行
public void await() throws InterruptedException {...}
//有等待时限的await方法
public boolean await(long timeout, TimeUnit unit) {...}
//每次调用countDown方法，count计数值-1
public void countDown() {...}
```

CountDownLatch各方法的具体实现是通过其静态内部类Sync，该类继承于AQS类，其内部是通过CAS操作及自旋实现线程安全。
await方法会调用LockSupport.park方法挂起当前线程。

CountDownLatch的两种典型用法：

- 某个线程等待其他若干个线程任务执行完成后，再继续运行；
- 多个线程并行执行，需等到所有线程都到达某个同步点后才能继续往下执行，类似于同步屏障的作用。

这两种用法，可通过百米赛跑来理解。在比赛开始前，我们需要等待所有参赛选手都到达起跑线做好起跑准备，此时先到的选手都调用了await方法进入阻塞状态。发令枪一响，调用了countDown方法，所有选手都被唤醒，继续往下执行，每个选手达到终点时，调用countDown方法，直到所有选手都到达终点，count值为0，唤醒了主线程，宣布比赛结束。

```java
public class CDLDemo {
    public static void main(String[] args) {
        ExecutorService players = Executors.newCachedThreadPool();
        final CountDownLatch playerPrepare = new CountDownLatch(1);
        final CountDownLatch contestComplete = new CountDownLatch(6);

        for (int i = 0; i < 6; i++) {
            Runnable runnable = new Runnable() {
                @Override
                public void run() {
                    try {
                        System.out.println("选手" + Thread.currentThread().getName() + "：已候场做准备");
                        playerPrepare.await();
                        System.out.println("选手" + Thread.currentThread().getName() + "：已开跑");
                        Thread.sleep((long)(Math.random() * 10000));
                        System.out.println("选手" + Thread.currentThread().getName() + "：到达终点");
                        contestComplete.countDown();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            };
            players.execute(runnable);
        }

        try {
            System.out.println("裁判" + Thread.currentThread().getName() + "：请所有选手入场，做好准备！");
            Thread.sleep((long)(Math.random() * 10000));
            playerPrepare.countDown();
            System.out.println("裁判" + Thread.currentThread().getName() + "：预备，跑！");
            contestComplete.await();
            System.out.println("所有选手到达终点，比赛结束");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        players.shutdown();
    }
}
```

一般需使用带有时限的await方法，避免某个线程执行异常导致无法执行countDown，使得线程无法被唤醒。
一个CountDownLatch只能使用一次，不能重新初始化或修改CountDownLatch对象内部计数器的值。

### 2. CyclicBarrier

字面意思，可循环使用的屏障，一组线程到达一个屏障时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。
CyclicBarrier的默认构造函数为CyclicBarrier(int parties)，参数parties表示屏障拦截的线程数，每个线程通过调用await方法通知CyclicBarrier已到达屏障，然后该线程被阻塞。
参数barrierAction用于在所有线程到达屏障时，在唤醒其他线程前执行的操作，方便处理较复杂的业务场景。
每次调用await方法，全局变量计数器count-1，当count为0时，则会调用barrierCommand.run，再唤醒其他阻塞的线程。

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

CyclicBarrier的计数器可通过reset方法重置。
CyclicBarrier内部维护了一个Generation类对象，用于标记屏障的每次使用，每次reset或屏障打开都会创建新的Generation对象。
CyclicBarrier内部通过ReentrantLock来实现屏障使用的同步，而线程的阻塞和唤醒是通过Condition对象的await和signalAll方法实现。

### 3. Semaphore

信号量，用来控制同时访问特定资源的线程数量。可通过设定一个阈值，多个线程竞争获取许可信号，完成处理后归还，超过阈值后，申请信号的线程会进入阻塞状态。
Semaphore内部是通过继承AQS的静态抽象类Sync的成员变量来实现主要功能的，Sync类有两个具体实现类：NonfairSync和FairSync，默认情况下创建NonFairSync实例对象。

线程通过调用acquire方法申请信号量，非公平模式下，若当前的可用信号量满足申请，则通过CAS操作更新分配之后的可用信号量，否则，线程将被挂起。公平模式下则会先判断等待队列内是否存在线程在等待申请信号量，有则将该线程入队，否则判断是否可分配并通过CAS操作更新信号量。

线程完成处理后，通过调用release方法归还信号量，for循环自旋执行CAS操作更新归还后的可用信号量。

Semaphore也提供了tryAcquire(long timeout, TimeUnit unit)、tryAcquire()等在限制时间内阻塞或非阻塞的实现方式。tryAcquire()、tryAcquire(int permits)会直接调用sync.nonfairTryAcquireShared方法，若在公平模式下调用会破坏公平性，所以公平模式下应调用tryAcquire(long timeout, TimeUnit unit)、tryAcquire(int permits, long timeout, TimeUnit unit)来保证公平性。

### 4. Exchanger

用于线程间协作的工具，提供了在线程间交换数据的手段。提供了一个同步点，在这个同步点，两个线程可交换彼此的数据。
如果第一个线程先执行exchange方法，会一直等待第二个线程也执行exchange方法，当两个线程都到达同步点时，这两个线程就可以交换数据。如果两个线程有一个没有执行exchange方法，就会一直等待。
