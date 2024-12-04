## Object o = new Object()占用空间
JavaVersion 1.8、64位计算机、工具使用jol
```
compile group: 'org.openjdk.jol', name: 'jol-core', version: '0.9'
```
```java
public class AObjectSpace {
    public static void main(String[] args) {
        Object o = new Object();
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
    }
}
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

占用16bytes,但是存在4bytes的填充对齐
```
### UseCompressedOops 与 UseCompressedClassPointers 
- UseCompressedOops 普通对象指针压缩
- UseCompressedClassPointers 类指针压缩
- 开启UseCompressedOops，默认会开启UseCompressedClassPointers，会压缩klass pointer。
- 要使UseCompressedClassPointers起作用，得先开启UseCompressedOops，并且开启UseCompressedOops 也默认强制开启UseCompressedClassPointers，关闭UseCompressedOops 默认关闭UseCompressedClassPointers。
### 指针压缩测试
```java
public class AObjectSpace {
    static class T{ Object o;}

    public static void main(String[] args) {
        T o = new T();
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
    }
}
```

```
-XX:+UseCompressedOops -XX:-UseCompressedClassPointers
开启普通对象指针压缩，关闭类指针压缩后
Object header占用16byte空间，内部属性o指针占用空间 4byte，总体占用24bytes。存在补充对齐4bytes

com.mouxf.work.AObjectSpace$T object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)                           10 35 c0 1b (00010000 00110101 11000000 00011011) (465581328)
     12     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
     16     4   java.lang.Object T.o                                       null
     20     4                    (loss due to the next object alignment)  补充对齐
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
```
-XX:+UseCompressedOops -XX:+UseCompressedClassPointers
开启普通对象指针压缩，关闭类指针压缩后
Object header占用12byte空间，内部属性o指针占用空间 4byte，总体占用16bytes

com.mouxf.work.AObjectSpace$T object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4   java.lang.Object T.o                                       null
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```
```
-XX:-UseCompressedOops -XX:-UseCompressedClassPointers
关闭普通对象指针压缩，关闭类指针压缩后
Object header占用16byte空间，内部属性o指针占用空间 8byte，总体占用24bytes。不存在对齐字段

com.mouxf.work.AObjectSpace$T object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)                           10 35 bd 1b (00010000 00110101 10111101 00011011) (465384720)
     12     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
     16     8   java.lang.Object T.o                                       null
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

```
```
-XX:-UseCompressedOops -XX:+UseCompressedClassPointers
com.mouxf.work.AObjectSpace$T object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)                           10 35 6f 1c (00010000 00110101 01101111 00011100) (477050128)
     12     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
     16     8   java.lang.Object T.o                                       null
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

Java HotSpot(TM) 64-Bit Server VM warning: UseCompressedClassPointers requires UseCompressedOops
报错信息，开启UseCompressedClassPointers前提是UseCompressedOops也必须开启
```
### 指针压缩
在堆中，32位的对象引用（指针）占4个字节，而64位的对象引用占8个字节。也就是说，64位的对象引用大小是32位的2倍。
64位JVM在支持更大堆的同时，由于对象引用变大却带来了性能问题：
- 增加了GC开销：64位对象引用需要占用更多的堆空间，留给其他数据的空间将会减少，从而加快了GC的发生，更频繁的进行GC。
- 降低CPU缓存命中率：64位对象引用增大了，CPU能缓存的oop将会更少，从而降低了CPU缓存的效率

### 指针压缩一定生效吗？
- CompressedOops原理：
64位地址分为堆的基地址+偏移量，当堆内存<32GB时候，在压缩过程中，把偏移量/8后保存到32位地址。在解压再把32位地址放大8倍，所以启用CompressedOops的条件是堆内存要在4GB*8=32GB以内。
- 零基压缩优化(Zero Based Compressd Oops)
零基压缩是针对压解压动作的进一步优化。 它通过改变正常指针的随机地址分配特性，强制堆地址从零开始分配（需要OS支持），进一步提高了压解压效率。要启用零基压缩，你分配给JVM的内存大小必须控制在4G以上，32G以下。
1. 如果GC堆大小在4G以下，直接砍掉高32位，避免了编码解码过程；
2. 如果GC堆大小在4G以上32G以下，则启用UseCompressedOop；
3. 如果GC堆大小大于32G，压缩指针失效，使用原来的64位

## 总结
new Object()无论怎样开启指针压缩都只是会占用16Bytes的内存，但是如果一个对象里面包含其他属性，则
- 考虑该属性本身占用空间大小
- 是否开启了普通对象指针压缩 UseCompressedOops
- 是否开启了类指针压缩 UseCompressedClassPointers
- 内存大小情况分析



