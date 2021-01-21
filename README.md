# Java-Study-Note
秉着未雨绸缪，时刻为面试做准备的想法来写这个系列，顺便夯实一下我还不太扎实的java基础。通过参考[敖丙](https://github.com/AobingJava/JavaFamily)、[三歪](https://github.com/ZhongFuCheng3y/3y)和其他[面试问题集合](https://github.com/Moosphan/Android-Daily-Interview)等整理了一些主流知识点，后续会慢慢将各个知识点填坑。
## 知识点目录
- java基础知识
  - :small_orange_diamond: [OOP：继承、封装和多态](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%9F%BA%E7%A1%80/1-%E7%BB%A7%E6%89%BF%E3%80%81%E5%B0%81%E8%A3%85%E5%92%8C%E5%A4%9A%E6%80%81.md)
  - :small_orange_diamond: [java数据类型](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%9F%BA%E7%A1%80/2-java%E5%9F%BA%E6%9C%AC%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B.md)
    - :small_orange_diamond: [包装类](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%9F%BA%E7%A1%80/2-java%E5%9F%BA%E6%9C%AC%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B.md#2-%E5%8C%85%E8%A3%85%E7%B1%BB)
  - :small_orange_diamond: [深入理解String](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%9F%BA%E7%A1%80/3-String%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90.md)
  - :small_orange_diamond: [关键字：修饰符、static、final](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%9F%BA%E7%A1%80/4-%E5%85%B3%E9%94%AE%E5%AD%97%EF%BC%9A%E4%BF%AE%E9%A5%B0%E7%AC%A6%E3%80%81static%E3%80%81final.md)
  - :small_orange_diamond: [你知道枚举是如何实现的吗？](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%9F%BA%E7%A1%80/5-%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%E6%9E%9A%E4%B8%BE.md)
  - :small_orange_diamond: [反射的使用及实现原理](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%9F%BA%E7%A1%80/7-%E5%8F%8D%E5%B0%84.md)
  - :small_orange_diamond: [单例模式的写法及线程安全问题分析](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%9F%BA%E7%A1%80/8-%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F.md)
  - :small_orange_diamond: [泛型--深挖那些被忽略的知识点](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%9F%BA%E7%A1%80/6-%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%E6%B3%9B%E5%9E%8B.md)
- java集合
  - :small_orange_diamond: [java面试的常客-HashMap深入解析](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E9%9B%86%E5%90%88/1-java%E9%9D%A2%E8%AF%95%E5%B8%B8%E5%AE%A2%EF%BC%9AHashMap%E8%A7%A3%E6%9E%90.md)
  - :small_orange_diamond: [逃不过的ConcurrentHashMap](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E9%9B%86%E5%90%88/2-%E5%95%83%E4%B8%8BConcurrentHashMap%E8%BF%99%E5%9D%97%E7%A1%AC%E9%AA%A8%E5%A4%B4.md)
  - :white_circle: ArrayList
  - :white_circle: CopyOnWriteArrayList
  - :small_orange_diamond: [ConcurrentModificationException的产生原因](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E9%9B%86%E5%90%88/3-ConcurrentModificationException%E5%BC%82%E5%B8%B8%E4%BA%A7%E7%94%9F%E5%8E%9F%E5%9B%A0.md)
- 并发编程
  - :white_circle: java内存模型
  - :white_circle: 双重检查锁定
  - :small_orange_diamond: [java的各种锁，你知道几个？](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/java%E7%9A%84%E5%90%84%E7%A7%8D%E9%94%81.md)
  - :white_circle: volatile
  - :white_circle: synchronized
  - :white_circle: 阻塞队列
  - :white_circle: Fork/Join框架
  - :small_orange_diamond: [并发工具类：CountDownLatch、CyclicBarrier和Semaphore](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/java%E5%B9%B6%E5%8F%91%E5%B7%A5%E5%85%B7%E7%B1%BB.md)
  - :white_circle: 线程池
  - :white_circle: 原子操作类
  - :small_orange_diamond: [ThreadLocal的实现原理及基本使用](https://github.com/Yolanda-lyz/Java-Study-Note/blob/main/java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/ThreadLocal%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E5%8F%8A%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8.md)
- JVM核心知识点
  - :white_circle: java的四种引用及其区别
  - :white_circle: 垃圾回收机制
  - :white_circle: 类加载机制
