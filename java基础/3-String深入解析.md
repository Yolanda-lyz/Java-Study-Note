### 1. 基本认知

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

### 2. String对象的不可变

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

### 3. 字符串常量池

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

### 4. String对象的创建

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

另外，如果直接或间接以字符数组作为参数来创建String对象，如String类的构造方法：``Sring(char value[])``、``String(byte ascii[])``、``String(StringBuilder builder)``等，则不会在常量池中查找或创建String对象，仅在堆中创建新的String对象返回。

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

### 5. 字符串拼接

#### 5.1 StringBuilder、StringBuffer

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

##### 5.1.1 StringBuilder和StringBuffer的区别

根据文档的注释，StringBuilder的出现，是为了在单线程使用中替代StringBuffer的，StringBuilder相较于StringBuffer有执行速度优势，但是是非线程安全的。在多线程环境下必须使用StringBuffer。

StringBuffer的线程安全在于其内部的方法增加了``synchronized``修饰，从接口调用上保证线程安全，也导致它的效率较低。

#### 5.2 '+' 拼接

字符串拼接最常用的就是直接使用加号，如下实例：

```java
String s = "";
for(int i = 0; i < 10; i++) {
    s += " good";  //（1）
}
String str = "study";
String slogan = "good" + " good" +" " + str + "day" +" day" + " up";  //（2）
```

- 执行语句（1），通过这个for循环的反编译字节码可知，实际上``s += " good";``的执行，等价于``s = (new StringBuilder(s)).append(" good").toString();``(PS：字节码中执行了两次append，第一次是在StringBuilder构造方法中执行)
  每一次循环都会创建一个新的StringBuilder对象，这也是循环内部不要使用加号拼接字符串的原因。
- 执行语句（2），根据反编译字节码可看出，对于加号连接操作，编译器会尽可能多的将字符串常量连接起来，所以其执行逻辑等价于``slogan = (new StringBuilder("good good ")).append(str).append("day day up").toString();``。

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

 ### 6. Android SDK对String类的变更

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
> 参考：[JDK8 String类知识总结](https://www.cnblogs.com/Createsequence/p/13477439.html)、[字符串常量池深入解析](https://blog.csdn.net/weixin_40304387/article/details/81071816)、[深入理解java中的String](https://www.cnblogs.com/xiaoxi/p/6036701.html)
