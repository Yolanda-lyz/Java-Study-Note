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

### 4. java对象头

synchronized使用的锁是存在java对象头里的。

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

其中，类型指针是指向方法区中的目标类的类型信息，通过该指针可以确定对象的具体类型；Mark Word默认存储对象的hashCode、GC分代年龄和锁标记位，32位JVM的Mark Word的默认存储结构如下所示：

|  锁状态  |     25bit      |     4bit     | 1bit是否是偏向锁 | 2bit锁标志位 |
| :------: | :------------: | :----------: | :--------------- | :----------- |
| 无锁状态 | 对象的hashCode | 对象分代年龄 | 0                | 01           |

在运行期间，Mark Word里存储的数据会随着锁标志位的变化而变化，Mark Word可能变化为存储以下四种数据：

[后续补图]

这四种数据分别对应锁的四种状态，级别从低到高：无锁状态->偏向锁状态->轻量级锁状态->重量级锁状态，这几个状态会随着竞争情况逐渐升级。

在hotspot jdk8以前，锁只能升级，不能降级；而在jdk8开始引入了锁的降级机制。

synchronized用的锁是存在java对象头里的，由于java任意对象都可用做锁，因此必定要有一个映射关系，存储该对象以及其对应的锁信息（比如当前哪个线程持有锁，哪些线程在等待），而对象头本身就有一些hashCode、GC相关的信息，所以将这个映射关系存储在对象头中是一个合理的选择。



以下源码来源于[版本为jdk8u的hotspot虚拟机](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/741cd0f77fac)

### 5. 偏向锁

偏向锁出现的原因：大多数情况下，锁不仅不存在竞争，而且总是有同一线程多次获得，为使线程获得锁的代价更低而引入了偏向锁。

synchronized代码块锁的获取是从monitorenter指令开始的，所以就要找到解析这条指令的地方。hotspot的解释器模块有两个实现：基于C++的解释器bytecodeInterpreter和基于汇编的模板解释器templateTable，hotspot默认使用汇编的模板解释器。关于模板解释器的实现可参考：https://zhuanlan.zhihu.com/p/33886967

模板解释器的调用路径：templateTable_x86_64.cpp#monitorenter->interp_masm_x86_64.cpp#biased_locking_enter->macroAssembler_x86.cpp#biased_locking_enter，在macroAssembler文件中就可以看到对应生成的汇编代码。

实际上C++解释器和模板解释器的执行逻辑大体上是一致的，所以还是通过C++解释器来继续探索。

#### 5.1 偏向锁的获取

monitorenter指令的执行逻辑如代码中注释：

```java
void BytecodeInterpreter::run(interpreterState istate) {   
      ......
 
      CASE(_monitorenter): {
        //lockee即为锁对象
        oop lockee = STACK_OBJECT(-1);
        // derefing's lockee ought to provoke implicit null check
        CHECK_NULL(lockee);
        // find a free monitor or one already allocated for this object
        // if we find a matching object then we need a new monitor
        // since this is recursive enter
        //1：找到一个空闲的Lock Record
        BasicObjectLock* limit = istate->monitor_base();
        BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
        BasicObjectLock* entry = NULL;
        while (most_recent != limit ) {
          if (most_recent->obj() == NULL) entry = most_recent;
          else if (most_recent->obj() == lockee) break;
          most_recent++;
        }
        if (entry != NULL) {
          //2：把锁对象赋给获取到的Lock Record的obj字段
          entry->set_obj(lockee);
          int success = false;
          uintptr_t epoch_mask_in_place = (uintptr_t)markOopDesc::epoch_mask_in_place;
          //markOop即为对象头的Mark Word字段
          markOop mark = lockee->mark();
          intptr_t hash = (intptr_t) markOopDesc::no_hash;
          //3：如果锁对象的Mark Word字段是偏向模式
          if (mark->has_bias_pattern()) {
            uintptr_t thread_ident;
            uintptr_t anticipated_bias_locking_value;
            thread_ident = (uintptr_t)istate->thread();
            anticipated_bias_locking_value =
              //将线程id和类的prototype_header(epoch+分代年龄+偏向模式+锁标志)相或
              (((uintptr_t)lockee->klass()->prototype_header() | thread_ident)
              //再与锁对象的Mark Word字段异或，只留下不相等部分
               ^ (uintptr_t)mark)
              //markOopDesc::age_mask_in_place为...0001111000，取反后为...1110000111，
              //除中间分代年龄4位，其他位均为1，
              //将上面所得结果与该值相与，即可将上面获取到的结果中忽略掉分代年龄4位
               & ~((uintptr_t) markOopDesc::age_mask_in_place);
 
 
            //所得结果为0，说明线程id及其他字段都与锁对象头的Mark Word各部分相同，
            //当前线程即为锁的偏向线程，无需再进一步处理
            if  (anticipated_bias_locking_value == 0) {
              // already biased towards this thread, nothing to do
              if (PrintBiasedLockingStatistics) {
                (* BiasedLocking::biased_lock_entry_count_addr())++;
              }
              success = true;
            }
            //4：如果偏向模式关闭，则需尝试撤销偏向锁
            else if ((anticipated_bias_locking_value & markOopDesc::biased_lock_mask_in_place) != 0) {
              // try revoke bias
              markOop header = lockee->klass()->prototype_header();
              if (hash != markOopDesc::no_hash) {
                header = header->copy_set_hash(hash);
              }
              //利用CAS操作将锁对象头的Mark Word替换为class的Mark Word
              if (Atomic::cmpxchg_ptr(header, lockee->mark_addr(), mark) == mark) {
                if (PrintBiasedLockingStatistics)
                  (*BiasedLocking::revoked_lock_entry_count_addr())++;
              }
            }
            //5：如果epoch不等于class的epoch，则尝试重偏向
            else if ((anticipated_bias_locking_value & epoch_mask_in_place) !=0) {
              // try rebias
              //构造一个偏向当前线程的Mark Word
              markOop new_header = (markOop) ( (intptr_t) lockee->klass()->prototype_header() | thread_ident);
              if (hash != markOopDesc::no_hash) {
                new_header = new_header->copy_set_hash(hash);
              }
              //通过CAS操作替换锁对象头的Mark Word
              if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), mark) == mark) {
                if (PrintBiasedLockingStatistics)
                  (* BiasedLocking::rebiased_lock_entry_count_addr())++;
              }
              else {
                //重偏向失败，代表存在多线程竞争，则调用InterpreterRuntime::monitorenter进行锁升级
                CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
              }
              success = true;
            }
            //6：走到此处说明当前要么偏向别的线程，要么是匿名偏向（没有偏向任何线程）
            else {
              // try to bias towards thread in case object is anonymously biased
              //构造一个匿名偏向的Mark Word
              markOop header = (markOop) ((uintptr_t) mark & ((uintptr_t)markOopDesc::biased_lock_mask_in_place |
                                                              (uintptr_t)markOopDesc::age_mask_in_place |
                                                              epoch_mask_in_place));
              if (hash != markOopDesc::no_hash) {
                header = header->copy_set_hash(hash);
              }
              //待替换的Mark Word偏向当前线程
              markOop new_header = (markOop) ((uintptr_t) header | thread_ident);
              // debugging hint
              DEBUG_ONLY(entry->lock()->set_displaced_header((markOop) (uintptr_t) 0xdeaddead);)
              //使用CAS操作替换锁对象头的Mark Word
              if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), header) == header) {
                if (PrintBiasedLockingStatistics)
                  (* BiasedLocking::anonymously_biased_lock_entry_count_addr())++;
              }
              else {
                //替换失败，说明存在多线程竞争，进入InterpreterRuntime::monitorenter方法
                CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
              }
              success = true;
            }
          }
 
          // traditional lightweight locking
          //7：偏向模式未开启，以及上面的3，会进入这里，走轻量级锁的逻辑
          if (!success) {
            //把锁对象头的Mark word置为无锁状态，并将Lock Record的lock指向它
            markOop displaced = lockee->mark()->set_unlocked();
            entry->lock()->set_displaced_header(displaced);
            //call_vm为true时，表示直接使用重量级锁
            bool call_vm = UseHeavyMonitors;
            //通过CAS尝试将锁对象头的Mark Word替换为指向Lock Record的指针
            if (call_vm || Atomic::cmpxchg_ptr(entry, lockee->mark_addr(), displaced) != displaced) {
              // Is it simple recursive case?
              //若是锁重入，将Displaced Mark Word置空
              if (!call_vm && THREAD->is_lock_owned((address) displaced->clear_lock_bits())) {
                entry->lock()->set_displaced_header(NULL);
              } else {
                //替换失败或重量级锁走这里
                CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
              }
            }
          }
          UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
        } else {
          istate->set_msg(more_monitors);
          UPDATE_PC_AND_RETURN(0); // Re-execute
        }
      }
      ......
}
```

上面代码中的lockee->klass()->prototype_header()表示的是锁对象的Klass Word字段中的prototype_header，它类似于Mark Word，JVM中每个类都有一个，用来标记该class的epoch和偏向开关等信息。

monitorenter指令的具体执行逻辑如下图：
[后续补图]

monitorenter指令处理中需要注意的点：

1）首先是从当前线程栈的低地址开始，查找最高地址的空闲可用的Lock Record，这里的Lock Record在代码中为BasicObjectLock对象。然后给获取到Lock Record的obj字段赋值，指向锁对象。

2）代码中anticipated_bias_locking_value值：首先将线程id和Klass Word字段中的prototype_header相或，得到的实际就是线程id和Klass Word字段中的prototype_header的组合（即为线程id+epoch+分代年龄+偏向模式+锁标志），而偏向模式下锁对象头的Mark Word字段实际上就是这种结构，所以把两者除分代年龄外的其他部分进行异或，所得结果若非零，说明这其中某个部分肯定不同，再根据进一步判断做不同的处理。

#### 5.2 偏向锁的撤销
--待续
