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

从字节码中可以看出，JVM是基于进入和退出Monitor对象来实现方法同步和代码块同步，二者的实现细节不同。synchronized同步代码块时，编译时会生成monitorenter和monitorexit指令，分别插入到同步代码块的开始位置以及方法代码块的结束处和异常处。当且一个monitor被持有后，它将处于锁定状态。而对于synchronized方法，编译时会生成一个ACC_SYNCHRONIZED关键字，JVM在进行方法调用时，识别到该关键字，就会先尝试获得锁。

虽然synchronized方法和synchronized块在实现细节上有一定的区别，但其底层获取锁的逻辑实际是一样的，后面关于锁的具体获取过程只分析synchronized块。

