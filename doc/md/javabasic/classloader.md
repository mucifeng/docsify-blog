## 基本介绍
### Java程序退出
- 执行了System。exit()方法
- 程序正常执行结束
- 程序在执行过程中遇到了异常或者错误而异常终止
- 由于操作系统出现错误而导致Java虚拟机进程终止
### 类的加载执行与初始化
- 加载：查找并加载类的二进制数据
    - 从本地系统中直接加载
    - 通过网络下载.class文件
    - 从zip，jar等归档文件中加载.class文件
    - 从专有数据库中提取.class文件
    - 将java源文件动态编译为.class文件（将JAVA源文件动态编译这种情况会在动态代理和web开发中jsp转换成Servlet）
      
- 连接
    - 验证：确保被加载的类的正确性
    - 准备：为类的静态变量分配内存，并将其初始化为默认值
    - 解析：把类中的符号引用转换为直接引用
 - 初始化：为类的静态变量赋予正确的初始值
 
**准备阶段即使我们为静态变量赋值为任意的数值，但是该静态变量还是会被初始化为他的默认值，最后的初始化时才会把我们赋予的值设为该静态变量的值**

### 类加载器描述
- java.lang.ClassLoader的子类
- 用户可以定制类的加载方式

根类加载器–>扩展类加载器–>系统应用类加载器–>自定义类加载器
类加载器并不需要等到某个类被“首次主动使用”时再加载它

JVM规范允许类加载器在预料某个类将要被使用时就预先加载它，如果在预先加载的过程中遇到了.class文件缺失或存在错误，类加载器必须在**程序首次主动**使用该类才报告错误（LinkageError错误），如果这个类没有被程序主动使用，那么类加载器就不会报告错误。

类加载器用来把类加载到java虚拟机中。从JDK1.2版本开始，类的加载过程采用父亲委托机制，这种机制能更好地保证Java平台的安全。在此委托机制中，除了java虚拟机自带的根类加载器以外，其余的类加载器都有且只有一个父加载器。当java程序请求加载器loader1加载Sample类时，loader1首先委托自己的父加载器去加载Sample类，若父加载器能加载，则有父加载器完成加载任务，否则才由加载器loader1本身加载Sample类。

类被加载后，就进入连接阶段。连接阶段就是将已经读入到内存的类的二进制数据合并到虚拟机的运行时环境中去。

- 类的连接-验证
    1. 类文件的结构检查
    2. 语义检查
    3. 字节码验证
    4. 二进制兼容性的验证
- 类的连接-准备
  在准备阶段，java虚拟机为类的静态变量分配内存，并设置默认的初始值。例如对于以下Sample类，在准备阶段，将为int类型的静态变量a分配4个字节的内存空间，并且赋予默认值0，为long类型的静态变量b分配8个字节的内存空间，并且赋予默认值0
  ```java
      public class Sample{
          private static int a=1;
          public  static long b;
          public  static long c;
          static {
              b=2;
          }
      }
  
  ```
  
## 代码用例
```java
  public static void main(String[] args) {System.out.println(Child.str1);}
  class Parent {
      static String str1 = "welcome Parent";
      static { System.out.println("from Parent class");}
  }
  class Child extends Parent {
      static String str2 = "welcome Child";
      static { System.out.println("from Child class");}
  }
  //输出内容：
  from Parent class
  welcome Parent
```  

**对于静态字段来说，只有直接定义了该字段的类才会被初始化,当一个类在初始化时，要求父类全部都已经初始化完毕**,例子中及时使用**Child.str1**也没有Child类的初始化过程
```java
public static void main(String[] args) {  
    System.out.println(Student.m); 
    System.out.println(Student2.str); 
}
class Student {
    static final int m = 6;
    static {System.out.println("Student init");  }
}
class Student2 {
    static final String str = UUID.randomUUID().toString();
    static {System.out.println("Student2 init");  }
}
//输出内容：
6
Student2 init
随机字符串
```
**常量在编译阶段会存入到调用这个常量的方法所在的类的常量池中**,使用时直接从常量池中拿，不会初始化Student类。
**当一个常量的值并非编译期间可以确定的，那么其值就不会放到调用类的常量池中**

```java
  public static void main(String[] args) {
      System.out.println(Singleton.a);//1
      System.out.println(Singleton.b);//0
  }
  class Singleton {
      public static int a;
      private static Singleton instance = new Singleton();
      private Singleton() {
          a++;
          b++;
          System.out.println(a);//1
          System.out.println(b);//1
      }
      public static int b = 0;
  }
  //输出
  1 1 1 0
```

```java
System.out.println(System.getProperty("sun.boot.class.path"));//根加载器路径
System.out.println(System.getProperty("java.ext.dirs"));//扩展类加载器路径
System.out.println(System.getProperty("java.class.path"));//应用类加载器路径
```