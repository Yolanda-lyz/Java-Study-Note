### 1. OOP（面向对象编程）

#### 1.1 继承、封装和多态

- 继承：子类继承父类的特征和行为，使得子类对象具有父类的实例域和方法。（父类private修饰的属性不能被继承）继承提高了代码的复用性。每个子类只能有一个父类，允许多级继承。子类只能重写父类的非静态方法，父类的静态方法会被隐藏。

- 封装：尽可能地隐藏对象内部的实现细节，向外提供简单的编程接口。既可减少代码间的耦合，也能提高程序的安全性。封装一般通过使用private修饰符，控制对象内部变量的访问权限，对外提供get和set接口。

- 多态：不同子类型的对象对同一消息作出不同的响应。多态性分为编译时的多态和运行时的多态。方法重载（overload）属于编译时的多态，而方法重写（overwrite）属于运行时的多态。

  多态主要体现在：一是父类引用指向子类，这样同样的引用调用同样的方法时就会根据子类对象的不同而表现出不同的行为；二是方法重写，子类继承父类并重写父类中已有的或抽象的方法。

> 参考：https://www.cnblogs.com/fenjyang/p/11462278.html

#### 1.2 重写与重载

重载指一个类中方法名相同，参数列表不同的多个函数，是类内部的多态体现，常见的有类构造器的重载；而重写是指子类继承父类时，对父类已有的或抽象的方法进行重新定义，子类对象在调用该方法时，会自动调用子类中的定义。

#### 1.3 接口和抽象类

共同点：都不能实例化，但可以定义抽象类和接口类型的引用。继承了抽象类和接口的子类，要么实现所有的抽象方法，要么仍声明为抽象类。

不同点：

- 接口：方法的集合，默认属性为public final static，接口中定义的成员变量均为常量。jdk 7及之前只包含抽象方法，jdk 8之后可以有默认方法和静态方法，jdk 9允许有私有方法。

  接口的抽象方法：abstract修饰，可省略，没有方法体，供子类实现使用。

  接口的默认方法：default修饰，不可省略，供子类调用或子类重写。

  接口的静态方法：static修饰，供接口直接调用。

  接口的私有方法：private修饰，供接口中的默认方法或者静态方法调用。

- 抽象类：允许定义构造器和成员变量，但由于不能实例化，其构造器必须被子类继承之后才有意义。抽象类中可以没有抽象方法，而内部有抽象方法的一定是抽象类。抽象类内部成员可以由private、default、protected、public修饰。

#### 1.4 多态的实现原理

待完成


---
### 2. 数据类型
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

#### 2.1 数值自动转换

- char可以转int，但Character不能转Integer，只能安全的转型为它实现的接口或父类。
- 自动转换：从低精度到高精度（byte, short, char -> int -> long -> float -> double）的顺序转换。
- 强制转换：在代码中显式声明，从高到低的转换，精度会丢失。

#### 2.2 包装类

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

---
### 3. 字符串

#### 3.1 基本认知

- String文档中的注释：Strings是常量，其值在实例创建后就不能被修改，字符串缓冲区支持可变字符串，因为缓冲区中的不可变字符串对象可以被共享。

- String类通过final修饰，不可被继承，同时其成员变量字符数组``value[]``也是被final修饰（final修饰基本数据类型变量，则变量的值被初始化后就不能更改），字符数组初始化后不可变，故String是不可变的。

  成员变量``hash``是由``value[]``计算得来，由于``value[]``不可变，则``hash``值也是唯一的。

  CharSequence是一个接口类，提供了关于字符串操作的接口，如``int length();``、``char charAt(int index)``等；

  ```java
  public final class String
      implements java.io.Serializable, Comparable<String>, CharSequence {
      /** The value is used for character storage. */
      private final char value[];
  
      /** Cache the hash code for the string */
      private int hash; // Default to 0
      ......
  }
  ```

- String对象初始化

  ```java
  public String() {
      this.value = "".value;
  }
  
  public String(String original) {
      this.value = original.value;
      this.hash = original.hash;
  }
  
  public String(char value[]) {
      this.value = Arrays.copyOf(value, value.length);
  }
  ```

#### 3.2 String对象的不可变

- 不可变的原因：String对象内部的字符数组``value[]``由final修饰，一旦被初始化后，字符数组的值不能被修改。

- 代码中关于String对象可变的假象：

  形如``String s = "aaa"; s = "bbb";``，实际是创建了两个String对象，只是将``s``的指向从"aaa"的String对象变更为"bbb"的String对象。

  String类内部的``subString()``、``trim()``等接口，也并不是在当前String对象上做修改，而是构造了一个新的String对象并返回。

  ```java
  public String substring(int beginIndex) {
      if (beginIndex < 0) {
          throw new StringIndexOutOfBoundsException(beginIndex);
      }
      int subLen = value.length - beginIndex;
      if (subLen < 0) {
          throw new StringIndexOutOfBoundsException(subLen);
      }
      return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
  }
  ```

- String对象设计成不可变的原因

  一是字符串常量池设计的需要。当创建String对象时，若字符串已经存在常量池中，则直接引用已经存在的String对象。如果允许改变，会导致各种逻辑错误，一个对象的改变会影响其他引用对象的逻辑。

  二是字符串的不变性保证了其hash码的唯一性，从而可以缓存hash码，避免重复计算。

  三是程序安全的要求。String作为各种java类的参数，若可变，会引起各种严重错误和安全漏洞；类加载器也需使用字符串，不可变性保证了安全性；不可变的字符串保证了线程安全。

#### 3.3 字符串常量池

在本地方法区存在一块特殊的内存区域，用于记录字符串常量，称为字符串常量池（jdk 8之后分离到了堆中），通过这个常量池来实现字符串的共享。常量池是在编译期被确定，并保存在已编译的.class文件中的一些数据。

- 对于形如``String str = "aaa"``，JVM会先检查常量池中是否存在相同的字符串，若已存在则直接返回已有的字符串实例地址，否则创建一个新的String对象并存入常量池。

- 对于形如``String str = new String("aaa")``，实际上等价于``String temp = "aaa"; String str = new String(temp);``，仍会先在常量池中获取或创建``"aaa"``String对象，然后将其作为参数调用``public String(String original)``构造方法，在堆中创建一个新的String对象且独立于常量池。

- 存储在堆中的String对象可以通过调用``intern()``方法手动入池。如果常量池已经存在该字符串，则``intern()``方法返回的是常量池中已存在的字符串的地址；如果常量池中不存在该字符串，jdk 7以后常量池不需要存储该String对象，而是直接存在该字符串在堆中的引用。

```java
public class testString {
    public static void main(String args[]) {
        String a = "a" + "b" + 1;
        String b = "ab1";
        System.out.println(a == b);  //（1）true
        String c = new String("ab1");
        String d = new String("ab1");
        System.out.println(c == b);  //（2）false
        System.out.println(c == d);  //（3）false
        System.out.println(c == c.intern());  //（4）false，"ab1"已在常量池，故c.intern()返回的是常量池中的String地址
    }
}
```

#### 3.4 String对象的创建

String对象的创建方式有两种：

- 以双引号形式创建，形如``String str = "aaa"``，JVM会检查该字符串是否已存在于常量池，若已存在，则直接返回已有String对象的地址；否则，会先在堆中创建一个String对象，并将对象移入常量池，再将地址返回赋给str。

- 以new形式创建，形如``String str = new String("aaa")``，实际上等价于``String temp = "aaa"; String str = new String(temp);``，同样会在常量池中检查是否已存在字符串"aaa"：

  - 若已存在，获取常量池中该字符创的地址，并将该地址作为参数在堆中创建新的String对象，其value指向的是常量池中对应String对象的value数组。
  - 若不存在，则先在常量池中创建一个新的String对象，并将其地址作为参数在堆中创建新的String对象，同样其value指向常量池中对应String对象的value数组。

  ```java
  public String(String original) {
      this.value = original.value;
      this.hash = original.hash;
  }
  ```

另外，如果直接以字符数组作为参数来创建String对象，则不会在常量池中查找或创建String对象，仅在堆中创建新的String对象返回。

```java
public class testString {
    public static void main(String args[]) {
        char[] value = new char[]{'g','o','o','d'};
        String a = new String(value, 0, 4);
        System.out.println(a == a.intern());  //true
    }
}
public final class String ... {
    public String(char value[], int offset, int count) {
        ......
        this.value = Arrays.copyOfRange(value, offset, offset+count);
    }
}
```

#### 3.5 字符串拼接

##### StringBuilder、StringBuffer

java的字符串类包括String、StringBuilder和StringBuffer，后两者与String的区别在于他们是``可变的字符序列``，每次变更都是针对对象本身，而不是像String一样直接创建新的对象。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence { ... }
    
public final class StringBuilder extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence { ... }
    
 public final class StringBuffer extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence { ... }
     
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    /**
     * The value is used for character storage.
     */
    char[] value;

    /**
     * The count is the number of characters used.
     */
    int count;
    
    AbstractStringBuilder(int capacity) {
        value = new char[capacity];
    }
    ...
}
```

可以看出，StringBuilder和StringBuffer的``可变性``就在于它们继承了``AbstractStringBuilder``抽象类，该类内部维护的char数组没有final修饰，因此是可变的。

value数组初始化时会给定一个初始容量值，当append的字符串长度超过数组容量时，会对数组进行扩容，反之，也会进行缩容。

###### StringBuilder和StringBuffer的区别

根据文档的注释，StringBuilder的出现，是为了在单线程使用中替代StringBuffer的，StringBuilder相较于StringBuffer有执行速度优势，但是是非线程安全的。在多线程环境下必须使用StringBuffer。

StringBuffer的线程安全在于其内部的方法增加了``synchronized``修饰，从接口调用上保证线程安全，也导致它的效率较低。

##### '+' 拼接

字符串拼接最常用的就是直接使用加号，如下实例：

```java
String s = "good";
for(int i = 0; i < 10; i++) {
    s += " good";
}
```

通过这个for循环的反编译字节码可知，实际上``s += " good";``的执行，等价于``s = (new StringBuilder(s)).append(" good").toString();``(PS：字节码中执行了两次append，第一次是在StringBuilder构造方法中执行)
每一次循环都会创建一个新的StringBuilder对象，这也是循环内部不要使用加号拼接字符串的原因。

[字节码截图待补]

``append``方法会将字符串保存到StringBuilder内部的value数组中。``StringBuilder.toString()``会构造一个新的String对象返回。

```java
public final class StringBuilder ... {
    @Override
    public String toString() {
        return new String(value, 0, count);
    }
}
```

对加号拼接的执行逻辑有了基本认识后，我们分析一下以下程序的执行逻辑。

1. ``new String("1") + new String("1")``，第一个new String("1")，会分别在常量池和堆中创建String对象，第二个则仅在堆中创建一个String对象；两个String对象的加号拼接，等价于``(new StringBuilder(string1)).append(string1).toString()``，最终的``toString()``是以char数组作为参数来创建String对象的，不会在常量池中查找或创建字符串，仅在堆中创建一个新的String对象。故常量池当前仍只有字符串``"1"``。
2. ``s3.intern()``，String对象"11"手动入池，jdk 7之后并不需要将String对象存储在常量池中，只需存储待入池的String对象在堆中的地址，故s3入池后返回的地址仍为s3在堆中的地址，即``s5 == s3``为true。
3. ``s4 = "11"``，s3手动入池后，常量池中已存在字符串"11"，故直接返回存储在池中的字符串"11"的地址。
4. 第二次的执行，由于是同一段程序，常量池唯一，所以一开始常量池就残留了第一次执行存储的字符串"1"和"11"，所以s3是新创建的String对象的地址，而因常量池已存在字符串"11"，``s3.intern()``返回的是存储在常量池中的第一次执行时创建的s3的地址，``s5 == s3``为false，``s5 == s4``为true。

  ```java
  public class testString {
      public static void main(String args[]) {
          {
              String s3 = new String("1") + new String("1");
              String s5 = s3.intern();
              String s4 = "11";
              System.out.println(s5 == s3);  //（1）true
              System.out.println(s5 == s4);  //（2）true
              System.out.println(s3 == s4);  //（3）true
          }
  
          System.out.println("======================");
  
          String s3 = new String("1") + new String("1");
          String s5 = s3.intern();
          String s4 = "11";
          System.out.println(s5 == s3);  //（4）false
          System.out.println(s5 == s4);  //（5）true
          System.out.println(s3 == s4);  //（6）false
      }
  }
  ```

 #### 3.6 Android SDK对String类的变更

Android SDK的String类有所变更，其内部并没有value数组成员，而是维护了一个int变量count。字符数组交由runtime运行时库来管理，而所有需要访问到字符数组的都替换成一个用于访问运行时库的native接口。

```java
public final class String implements ... {
    // BEGIN Android-changed: The character data is managed by the runtime.
    // private final char value[];
    //
    // If STRING_COMPRESSION_ENABLED, count stores the length shifted one bit to the left with the
    // lowest bit used to indicate whether or not the bytes are compressed (see GetFlaggedCount in
    // the native code).
    private final int count;
    // END Android-changed: The character data is managed by the runtime.
    
    /** Cache the hash code for the string */
    private int hash; // Default to 0
    ......
    // BEGIN Android-changed: Replace with implementation in runtime to access chars (see above).
    @FastNative
    public native char charAt(int index);
    // END Android-changed: Replace with implementation in runtime to access chars (see above).
    ......
    // BEGIN Android-changed: Replace with implementation in runtime to access chars (see above).
    @FastNative
    public native int compareTo(String anotherString);
    // END Android-changed: Replace with implementation in runtime to access chars (see above).
    ......
    // BEGIN Android-changed: Replace with implementation in runtime to access chars (see above).
    @FastNative
    public native String concat(String str);
    // END Android-changed: Replace with implementation in runtime to access chars (see above).
    ......
}
```
