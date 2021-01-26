> 从本篇开始及后续的系列解析，源码来源于[版本为jdk8u的hotspot虚拟机](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/741cd0f77fac)


偏向锁出现的原因：大多数情况下，锁不仅不存在竞争，而且总是有同一线程多次获得，为使线程获得锁的代价更低而引入了偏向锁。

synchronized代码块锁的获取是从monitorenter指令开始的，所以就要找到解析这条指令的地方。hotspot的``解释器模块``有两个实现：基于C++的解释器``bytecodeInterpreter``和基于汇编的模板解释器``templateTable``，hotspot默认使用汇编的模板解释器。关于模板解释器的实现可参考：https://zhuanlan.zhihu.com/p/33886967

模板解释器的调用路径：templateTable_x86_64.cpp#monitorenter->interp_masm_x86_64.cpp#biased_locking_enter->macroAssembler_x86.cpp#biased_locking_enter，在macroAssembler文件中就可以看到对应生成的汇编代码。

实际上C++解释器和模板解释器的执行逻辑大体上是一致的，所以我们还是通过C++解释器来详细分析。
