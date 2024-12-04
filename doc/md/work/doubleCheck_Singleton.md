## 单例模式中的双重验证
```java
public class Singleton {
        private volatile static Singleton singleton;

        private Singleton() {
        }
        public static Singleton newInstance() {
            if (singleton == null) {  //检查第一个singleton不为null,则不需要执行下面的加锁动作，极大提高了程序的性能；
                synchronized (Singleton.class) {
                    if (singleton == null) {
                        singleton = new Singleton();
                    }
                }
            }
            return singleton;
        }
    }
```
### 为什么singleton使用volatile关键字？
创建一个对象分为三个步骤：
1. **分配内存空间** (扩展: 新生代分配、TALB、对象过大分配在老年代)
2. **初始化对象**
3. **内存空间的地址赋值给对象的引用**<br>
由于JVM可能会对代码进行重排序，所以2和3可能会倒序执行。造成线程A执行到了new方法中的第3步(第2步未执行)，线程B执行判断singleton == null时为false，
但实际上singleton并没有正真的被初始化，仅仅是当前空间地址指针。造成线程B其实是拿到一个空的对象

### 解决方案
1. 使用volatile防止指针重排序
2. 利用classloder的机制来保证初始化instance时只有一个线程。JVM在类初始化阶段会获取一个锁，这个锁可以同步多个线程对同一个类的初始化

```java
public class Singleton {
    private Singleton(){}
   private static class SingletonHolder{
       public static Singleton singleton = new Singleton();
   }
   public static Singleton getInstance(){
       return SingletonHolder.singleton;
   }
}
```
