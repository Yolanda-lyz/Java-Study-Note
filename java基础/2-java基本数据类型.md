java的数据类型分为两种：基本数据类型和引用数据类型。

java共有8种基本类型，具体的类型及存储占位空间定义如下，每种类型的取值范围都定义在包装类的常量MIN_VALUE和MAX_VALUE中。

| 分类   | 数据类型 | 存储            | 包装类    |
| ------ | -------- | --------------- | --------- |
| 整型   | int      | 32 bit（4字节） | Integer   |
| 整型   | short    | 16 bit（2字节） | Short     |
| 整型   | long     | 64 bit（8字节） | Long      |
| 整型   | byte     | 8 bit（1字节）  | Byte      |
| 浮点型 | float    | 32 bit（4字节） | Float     |
| 浮点型 | double   | 64 bit（8字节） | Double    |
| 字符型 | char     | 16 bit（2字节） | Character |
| 逻辑型 | boolean  | 与平台有关      | Boolean   |

每种基本数据类型都有一个对应的包装类，便于基本数据类型和引用类型之间的相互转换。其中，基本数据类型存放在栈中，包装类在堆中给对象分配空间，栈中存放对象的引用地址。因此，包装类的效率会比基本数据类型低。

### 1. 数值自动转换

- char可以转int，但Character不能转Integer，只能安全的转型为它实现的接口或父类。
- 自动转换：从低精度到高精度（byte, short, char -> int -> long -> float -> double）的顺序转换。
- 强制转换：在代码中显式声明，从高到低的转换，精度会丢失。

### 2. 包装类

- int->Integer：装箱，如``Interger i = 2;``，实际上会自动调用``Integer.valueOf(int i)``方法，该方法会返回一个Integer对象。当传入的参数在-128~127之间时，返回``IntegerCache``中``cache数组``的Integer对象，否则会``new``一个Integer对象返回。

  ```java
  public static Integer valueOf(int i) {
      if (i >= IntegerCache.low && i <= IntegerCache.high)
          return IntegerCache.cache[i + (-IntegerCache.low)];
      return new Integer(i);
  }
  ```

- Integer->int：拆箱，如``Integer i = 2; int j = i;``,``int j = i;``的赋值过程实际上会自动调用``Integer.intValue()``方法，返回Integer对象内部的成员变量``value``。

- int与Integer的对比

  - Integer变量实际上是对一个Integer对象的引用，所以两个new出来的Integer变量不相等，故（1）为false。
  - Integer变量与int变量比较时，实际上是``Integer.intValue()``与int变量的比较，隐含了拆箱动作，是对两个基本数据类型的比较，故（2）为true。
  - 非new生成的Integer变量，当int变量范围为-128~127时，指向的是存储在静态常量池中IntegerCache的cache数组的Integer对象，当int变量超出该范围，则指向的是重新new的Integer对象。基于此，也就不难理解（3）（4）（5）的输出结果了。

  ```java
  public class testInteger {
      public static void main(String[] args) {
          Integer a = new Integer(10);
          Integer b = new Integer(10);
          System.out.println(a == b); // (1)false 
          int c = 10;
          System.out.println(a == c); // (2)true
          Integer e = 10;
          Integer f = 10;
          System.out.println(a == e); // (3)false
          System.out.println(e == f); // (4)true
          Integer g = 128;
          Integer h = 128;
          System.out.println(g == f); // (5)false
      }
  }
  ```

- Integer内部的IntegerCache的设计模式，称为享元模式（Flyweight Pattern），主要用于减少创建对象的数量，以减少内存占用和提高性能。缓存在第一次使用时初始化，high属性可以通过参数配置，否则默认设为127，创建[-128, 127]范围的Integer对象，存储在其内部的cache数组中，从而实现对象的复用。

```java
public final class Integer ...{
        private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
}
```

- 注意包装类隐式拆箱可能导致的空指针问题。以下归纳了几种因包装类对象为空而导致NE的场景：
  - Integer->int；
  - 包装类对象与基本数据类型变量进行【==】、【!=】、【>】、【+】等操作；
  - 包装类对象作为函数的返回值，而函数的返回类型声明为基本数据类型，也包含了拆箱逻辑；
  - if(Boolean)会转化为if(boolean)，Boolean为null，也会导致NE。
