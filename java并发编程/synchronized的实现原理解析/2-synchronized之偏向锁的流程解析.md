> 从本篇开始及后续的系列解析，源码来源于[版本为jdk8u的hotspot虚拟机](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/741cd0f77fac)


偏向锁出现的原因：大多数情况下，锁不仅不存在竞争，而且总是有同一线程多次获得，为使线程获得锁的代价更低而引入了偏向锁。

synchronized代码块锁的获取是从monitorenter指令开始的，所以就要找到解析这条指令的地方。hotspot的``解释器模块``有两个实现：基于C++的解释器``bytecodeInterpreter``和基于汇编的模板解释器``templateTable``，hotspot默认使用汇编的模板解释器。关于模板解释器的实现可参考：https://zhuanlan.zhihu.com/p/33886967

模板解释器的调用路径：templateTable_x86_64.cpp#monitorenter->interp_masm_x86_64.cpp#biased_locking_enter->macroAssembler_x86.cpp#biased_locking_enter，在macroAssembler文件中就可以看到对应生成的汇编代码。

实际上C++解释器和模板解释器的执行逻辑大体上是一致的，所以我们还是通过C++解释器来详细分析。

### 1. 偏向锁的获取

``monitorenter指令``的执行逻辑如代码中注释：

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

上面代码中的``lockee->klass()->prototype_header()``表示的是锁对象的Klass Word字段中的``prototype_header``，它类似于Mark Word，JVM中每个类都有一个，用来``标记该class的epoch和偏向开关``等信息。

monitorenter指令的具体执行逻辑如下图：

![mmonitorenter指令执行逻辑](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a192aaab45454ad8a7d8eb204dca01f0~tplv-k3u1fbpfcp-zoom-1.image)

monitorenter指令处理中需要注意的点：
- 首先是从当前线程栈的低地址开始，查找最高地址的空闲可用的Lock Record，这里的Lock Record在代码中为``BasicObjectLock``对象。然后给获取到Lock Record的``obj字段``赋值，``指向锁对象``。
- 代码中``anticipated_bias_locking_value``值：首先将线程id和Klass Word字段中的prototype_header相或，得到的实际就是线程id和Klass Word字段中的prototype_header的组合（即为``线程id+epoch+分代年龄+偏向模式+锁标志``），而偏向模式下锁对象头的Mark Word字段实际上就是这种结构，所以把两者除分代年龄外的其他部分进行异或，所得结果若非零，说明这其中某个部分肯定不同，再根据进一步判断做不同的处理。

### 2. 偏向锁的撤销
