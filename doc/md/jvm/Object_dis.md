## 对象分配策略
```
java -version
java version "1.8.0_172"
Java(TM) SE Runtime Environment (build 1.8.0_172-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.172-b11, mixed mode)
```

### 当Eden区中没有足够的空间进行分配时，虚拟机将发起一次Minor GC
```java
/**
 * -verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
 */
public class MinorGCTest {
    private static final int _1MB = 1024 * 1024;
    byte[] allocation1,allocation2,allocation3,allocation4;
    public static void testAllocation(){
        allocation1 = new byte[2 * _1MB];
        allocation2 = new byte[2 * _1MB];
        allocation3 = new byte[2 * _1MB];
        allocation4 = new byte[4 * _1MB];
    }
    public static void main(String[] args) {
        MinorGCTest test = new 
        testAllocation();
    }
}
//基本信息  2 * _1MB = 2048K  4 * _1MB=4096K    6 * _1MB = 6144K  9 * _1MB = 9216k
//GC打印信息
[GC (Allocation Failure) [PSYoungGen: 6246K->760K(9216K)] 6246K->4864K(19456K), 0.0472815 secs] [Times: user=0.00 sys=0.00, real=0.05 secs] 
Heap
 PSYoungGen      total 9216K, used 7225K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 78% used [0x00000000ff600000,0x00000000ffc50670,0x00000000ffe00000)
  from space 1024K, 74% used [0x00000000ffe00000,0x00000000ffebe010,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 4104K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 40% used [0x00000000fec00000,0x00000000ff002020,0x00000000ff600000)
 Metaspace       used 3432K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 374K, capacity 388K, committed 512K, reserved 1048576K

```
