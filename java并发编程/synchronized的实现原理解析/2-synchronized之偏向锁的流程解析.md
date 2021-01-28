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
撤销是指在获取偏向锁的过程因不满足条件导致要将锁对象改成非偏向锁的状态。

根据上面的代码逻辑，撤销偏向锁的逻辑主要在``InterpreterRuntime#monitorenter``，可以看到，如果开启了JVM偏向锁，则会进入``ObjectSynchronizer#fast_enter``方法中。

```c++
//InterpreterRuntime#monitorenter
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
  ......
  if (UseBiasedLocking) {
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
    ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
  } else {
    ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
  }
  ......
IRT_END
```

正常的java线程，会进入到``BiasedLocking#revoke_and_rebias``方法中，VM线程则会走下面的else分支。

```c++
//ObjectSynchronizer#fast_enter
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
 if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
 }
 slow_enter (obj, lock, THREAD) ;
}
```

``BiasedLocking#revoke_and_rebias``方法用于撤销或重偏向，走到该方法的逻辑有很多，我们这里就以常见的情况为例分析：锁已偏向线程A，此时线程B尝试获得锁。

B线程最终会走到撤销单个线程的逻辑处，若要撤销的锁偏向的是当前线程则直接调用``revoke_bias``撤销偏向锁，否则会push到VM线程``等待safepoint到来``再执行。

JVM中有个专门的``VM Thread``，该线程会不断从``VMOperationQueue``中取出请求，比如GC请求，对于需要safepoint的操作必须等到所有的java线程进入到safepoint才开始执行。

```C++
//BiasedLocking#revoke_and_rebias
BiasedLocking::Condition BiasedLocking::revoke_and_rebias(Handle obj, bool attempt_rebias, TRAPS) {
  ...
  markOop mark = obj->mark();
  //如果是匿名偏向且不重偏向则走这里
  if (mark->is_biased_anonymously() && !attempt_rebias) {
    ......
  }
  //锁对象开启了偏向模式会走这里
  else if (mark->has_bias_pattern()) {
    Klass* k = obj->klass();
    markOop prototype_header = k->prototype_header();
    //若class的prototype_header关闭了偏向模式
    if (!prototype_header->has_bias_pattern()) {
      ......
    }
    //若epoch过期
    else if (prototype_header->bias_epoch() != mark->bias_epoch()) {
      ......
    }
  }
  //根据update_heuristics返回值决定是单个还是批量
  HeuristicsResult heuristics = update_heuristics(obj(), attempt_rebias);
  if (heuristics == HR_NOT_BIASED) {
    return NOT_BIASED;
  }
  //撤销单个线程
  else if (heuristics == HR_SINGLE_REVOKE) {
    Klass *k = obj->klass();
    markOop prototype_header = k->prototype_header();
    if (mark->biased_locker() == THREAD &&
        prototype_header->bias_epoch() == mark->bias_epoch()) {     
      //走到这里说明需要撤销的是偏向当前线程的锁，当调用Object#hashcode方法时会走到这一步
      //由于只需遍历当前线程的栈即可，故不需要等到safepoint再撤销
      ResourceMark rm;
      ...
      EventBiasedLockSelfRevocation event;
      BiasedLocking::Condition cond = revoke_bias(obj(), false, false, (JavaThread*) THREAD, NULL);
      ...
      if (event.should_commit()) {
        ...
      }
      return cond;
    } else {
      //这里最终会在VM线程中的safepoint调用revoke_bias方法
      EventBiasedLockRevocation event;
      VM_RevokeBias revoke(&obj, (JavaThread*) THREAD);
      VMThread::execute(&revoke);
      if (event.should_commit() && (revoke.status_code() != NOT_BIASED)) {
        ...
      }
      return revoke.status_code();
    }
  }
 
  EventBiasedLockClassRevocation event;
  //批量撤销、批量重偏向的逻辑
  VM_BulkRevokeBias bulk_revoke(&obj, (JavaThread*) THREAD,
                                (heuristics == HR_BULK_REBIAS),
                                attempt_rebias);
  VMThread::execute(&bulk_revoke);
  if (event.should_commit()) {
    ...
  }
  return bulk_revoke.status_code();
}
```

继续分析撤销线程调用的``revoke_bias``方法，其具体执行逻辑如代码中注释：

```c++
//BiasedLocking#revoke_bias
static BiasedLocking::Condition revoke_bias(oop obj, bool allow_rebias, bool is_bulk, JavaThread* requesting_thread, JavaThread** biased_locker) {
  markOop mark = obj->mark();
  //如果没有开启偏向模式，则直接返回
  if (!mark->has_bias_pattern()) {
    ...
    return BiasedLocking::NOT_BIASED;
  }
 
  uint age = mark->age();
  //构建两个Mark Word，一个是匿名偏向模式（101），一个是无锁模式（001）
  markOop   biased_prototype = markOopDesc::biased_locking_prototype()->set_age(age);
  markOop unbiased_prototype = markOopDesc::prototype()->set_age(age);
  ...
 
  JavaThread* biased_thread = mark->biased_locker();
  //匿名偏向，当调用锁对象的hashCode方法可能会走这个逻辑
  if (biased_thread == NULL) {
    // Object is anonymously biased. We can get here if, for
    // example, we revoke the bias due to an identity hash code
    // being computed for an object.
    //如果不允许重偏向，则将对象头的Mark Word设置为无锁模式
    if (!allow_rebias) {
      obj->set_mark(unbiased_prototype);
    }
    ...
    return BiasedLocking::BIAS_REVOKED;
  }
 
  // Handle case where the thread toward which the object was biased has exited
  //该标志用于标识当前的偏向线程是否还存活
  bool thread_is_alive = false;
  //如果当前线程就是偏向线程
  if (requesting_thread == biased_thread) {
    thread_is_alive = true;
  } else {
    //遍历jvm的所有线程，如果找到，证明当前的偏向线程存活
    for (JavaThread* cur_thread = Threads::first(); cur_thread != NULL; cur_thread = cur_thread->next()) {
      if (cur_thread == biased_thread) {
        thread_is_alive = true;
        break;
      }
    }
  }
  //当前偏向线程已不存活了
  if (!thread_is_alive) {
    //若允许重偏向，则将对象头的Mark Word设置为匿名偏向状态
    if (allow_rebias) {
      obj->set_mark(biased_prototype);
    }
    //不允许重偏向则设置为无锁模式
    else {
      obj->set_mark(unbiased_prototype);
    }
    ...
    return BiasedLocking::BIAS_REVOKED;
  }
 
  // Thread owning bias is alive.
  // Check to see whether it currently owns the lock and, if so,
  // write down the needed displaced headers to the thread's stack.
  // Otherwise, restore the object's header either to the unlocked
  // or unbiased state.
 
  //接着是当前偏向线程仍存活的执行逻辑
  GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(biased_thread);
  BasicLock* highest_lock = NULL;
  //遍历线程栈中的所有Lock Record
  for (int i = 0; i < cached_monitor_info->length(); i++) {
    MonitorInfo* mon_info = cached_monitor_info->at(i);
    //如果找到对应的Lock Record说明偏向的线程还在执行同步代码块中的代码
    if (mon_info->owner() == obj) {
      ...
      // Assume recursive case and fix up highest lock later
      //需要升级为轻量级锁，直接修改偏向线程栈中的Lock Record
      //为了处理锁重入的case，在这里将Lock Record的Displaced Mark Word设置为null，第一个Lock Record会在下面的代码中再处理
      markOop mark = markOopDesc::encode((BasicLock*) NULL);
      highest_lock = mon_info->lock();
      highest_lock->set_displaced_header(mark);
    } else {
      ...
    }
  }
  //修改第一个Lock Record为无锁状态，并将obj的Mark Word设置为指向该Lock Record的指针
  if (highest_lock != NULL) {
    // Fix up highest lock to contain displaced header and point
    // object at it
    highest_lock->set_displaced_header(unbiased_prototype);
    // Reset object header to point to displaced mark.
    // Must release storing the lock address for platforms without TSO
    // ordering (e.g. ppc).
    obj->release_set_mark(markOopDesc::encode(highest_lock));
    ...
  }
  //走这里说明找不到对应的Lock Record，即偏向线程已不在同步代码块中
  else {
    ...
    //允许重偏向则设为匿名偏向状态
    if (allow_rebias) {
      obj->set_mark(biased_prototype);
    } else {
      // Store the unlocked value into the object's header.
      //否则设为无锁状态
      obj->set_mark(unbiased_prototype);
    }
  }
```

``revoke_bias``方法中有几个需要注意的点：

- 当调用锁对象的``Object#hash``或``System.identityHashCode``方法导致该对象的偏向锁或轻量级锁升级，因为java中一个对象的hashCode是在调用这两个方法时才生成的，若是无锁状态则存放在Mark Word中，若是重量级锁则存放在对应的monitor中，若是偏向锁则没有地方存放该信息，必须进行锁升级。
- 通过遍历栈的``Lock Record``可判断线程是否还在同步块中。前面执行``monitorenter指令``时，会找到``最高地址的空闲``Lock Record，并将其obj字段指向锁对象；执行``monitorexit指令``时，会找到``最低地址``一个相关的Lock Record并做相关清理动作。
- 若当前的偏向线程非存活或不在同步代码块中，则直接将锁设置为匿名偏向状态或无锁状态，从而完成偏向锁的撤销。
- 若当前的偏向线程仍存活且仍在同步代码块中，则需将偏向线程的所有相关Lock Record的Displaced Mark Word设置为null，并将最高位的Lock Record的Displaced Mark Word设置为无锁状态，然后将锁对象头Mark Word字段指向最高位的Lock Record（因为这里是在safepoint中，所以不需要CAS指令），执行完后，原偏向线程的所有Lock Record都变成轻量级锁的状态。
- revoke_bias执行完之后会回到前面的fast_enter方法中，再调用slow_enter方法，将置为无锁状态的锁升级为轻量级锁，否则会膨胀成重量级锁，所以4中的情况最终会膨胀成重量级锁。
