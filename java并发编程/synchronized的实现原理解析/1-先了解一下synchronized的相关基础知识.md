### 1. synchronized实现同步的基础

- 每个java对象都可以作为锁。
- java是基于Hoare的[监视器](https://en.wikipedia.org/wiki/Monitor_(synchronization))的思想，每个java对象都有一个监视器。

### 2. 关于synchronized作用的对象

无论synchronized关键字加在方法上还是对象上，如果synchronized作用的对象是非静态的，则它取得的锁是对象，如果synchronized作用的对象是一个静态方法或一个类，则它取得的锁是类，该类所有的对象同一把锁。

### 3. synchronized字节码

java中synchronized用于实现同步代码块或同步方法，基本使用如下实例：

```java
public class TestSynchronized {
    Integer mutex = 0;
    public void syncBlock() {
        synchronized (mutex) {
            mutex = 1;
        }
    }
    public synchronized void syncMethod() {
        mutex = 2;
    }
}
```

使用javap -v来反编译经javac编译出来的class文件，得到的JVM字节码信息如下：

```java
public class TestSynchronized
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   ......
{
  java.lang.Integer mutex;
    descriptor: Ljava/lang/Integer;
    flags:
 
  public TestSynchronized();
    ......
 
  public void syncBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: getfield      #3                  // Field mutex:Ljava/lang/Integer;
         4: dup
         5: astore_1
         6: monitorenter                //monitorenter指令进入同步块
         7: aload_0
         8: iconst_1
         9: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        12: putfield      #3                  // Field mutex:Ljava/lang/Integer;
        15: aload_1
        16: monitorexit                 //monitorexit指令退出同步块
        17: goto          25
        20: astore_2
        21: aload_1
        22: monitorexit                 //执行异常时会走这里，monitorexit指令退出同步块
        23: aload_2
        24: athrow
        25: return
      Exception table:
         from    to  target type
             7    17    20   any
            20    23    20   any
      LineNumberTable:
        line 5: 0
        line 6: 7
        line 7: 15
        line 8: 25
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 20
          locals = [ class TestSynchronized, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
 
  public synchronized void syncMethod();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED       //添加ACC_SYNCHRONIZED标记
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: iconst_2
         2: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
         5: putfield      #3                  // Field mutex:Ljava/lang/Integer;
         8: return
      LineNumberTable:
        line 11: 0
        line 12: 8
}
```

从字节码中可以看出，JVM是``基于进入和退出Monitor对象``来实现方法同步和代码块同步，二者的实现细节不同。synchronized同步代码块时，编译时会生成``monitorenter和monitorexit指令``，分别插入到同步代码块的开始位置以及方法代码块的结束处和异常处。当且一个monitor被持有后，它将处于锁定状态。而对于synchronized方法，编译时会生成一个``ACC_SYNCHRONIZED``关键字，JVM在进行方法调用时，识别到该关键字，就会先尝试获得锁。

虽然synchronized方法和synchronized块在实现细节上有一定的区别，但其底层获取锁的逻辑实际是一样的，后面关于锁的具体获取过程只分析synchronized块。

### 4. java对象头

synchronized使用的``锁是存放在java对象头``里的。

java面向对象的思想，在jvm中需大量存储对象，存储时为了实现一些额外的功能，需要在对象中添加一些标记字段用于增强对象功能，这些标记字段组成了对象头。

如果对象是数组类型，虚拟机使用3个字宽来存储对象头，如果对象是非数组类型，则用2字宽存储对象头。对象头的形式如下所示：

```java
|--------------------------------------------------------------------------------------------|
|                                         Object Header                                      |
|-----------------------------------|--------------------------|-----------------------------|
|        Mark Word(32/64bits)       |    Klass Word(32/64bits) |  [array length(32/64bits)]  |
|-----------------------------------|--------------------------|-----------------------------|
```

普通对象的对象头中包括Mark Word和类型指针，而数组对象还会有一个字宽来记录数组长度。

其中，类型指针是指向方法区中的``目标类的类型信息``，通过该指针可以确定对象的具体类型；Mark Word默认存储对象的hashCode、GC分代年龄和锁标记位，32位JVM的Mark Word的默认存储结构如下所示：

|  锁状态  |     25bit      |     4bit     | 1bit是否是偏向锁 | 2bit锁标志位 |
| :------: | :------------: | :----------: | :--------------- | :----------- |
| 无锁状态 | 对象的hashCode | 对象分代年龄 | 0                | 01           |

在运行期间，Mark Word里存储的数据会随着锁标志位的变化而变化，Mark Word可能变化为存储以下四种数据：

【待后续补图】

这四种数据分别对应锁的四种状态，级别从低到高：无锁状态->偏向锁状态->轻量级锁状态->重量级锁状态，这几个状态会随着竞争情况逐渐升级。

在hotspot jdk8以前，锁只能升级，不能降级；而在jdk8开始引入了锁的降级机制。

synchronized用的锁是存在java对象头里的，由于java任意对象都可用做锁，因此必定要有一个映射关系，存储该对象以及其对应的锁信息（比如当前哪个线程持有锁，哪些线程在等待），而对象头本身就有一些hashCode、GC相关的信息，所以将这个映射关系存储在对象头中是一个合理的选择。


