### 简述
![垃圾收集器](https://pic.yupoo.com/crowhawk/56a02e55/3b3c42d2.jpg)
<br>上图展示了7种作用于不同分代的收集器，如果两个收集器之间存在连线，就说明它们可以搭配使用。虚拟机所处的区域，则表示它是属于新生代收集器还是老年代收集器。Hotspot实现了如此多的收集器，正是因为目前并无完美的收集器出现，只是选择对具体应用最适合的收集器。
## 新生代收集器
### Serial
**Serial（串行）收集器是最基本、发展历史最悠久的收集器，采用复制算法的新生代收集器。一个单线程收集器，只会使用一个CPU或一条收集线程去完成垃圾收集工作，更重要的是在进行垃圾收集时，必须暂停其他所有的工作线程，直至Serial收集器收集结束为止（“Stop The World”）**
**HotSpot虚拟机运行在Client模式下的默认的新生代收集器**。  -XX:+UseSerialGC：添加该参数来显式的使用串行垃圾收集器

![Serial](https://pic.yupoo.com/crowhawk/6b90388c/6c281cf0.png)

特点：
 - 简单而高效（与其他收集器的单线程相比）
 - 对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得更高的单线程收集效率
 - 在用户的桌面应用场景中，可用内存一般不大（几十M至一两百M），可以在较短时间内完成垃圾收集（几十MS至一百多MS）,只要不频繁发生，这是可以接受的
 
### ParNew
**ParNew收集器就是Serial收集器的多线程版本，它也是一个新生代收集器。除了使用多线程进行垃圾收集外，其余行为包括Serial收集器可用的所有控制参数、收集算法（复制算法）、Stop The World、对象分配规则、回收策略等与Serial收集器完全相同**
除了Serial收集器外，目前只有它能和CMS收集器（Concurrent Mark Sweep）配合工作

![ParNew](https://pic.yupoo.com/crowhawk/605f57b5/75122b84.png)

**设置参数**
- -XX:+UseConcMarkSweepGC：指定使用CMS后，会默认使用
- -XX:+UseParNewGC：强制指定使用ParNew
- -XX:ParallelGCThreads：指定垃圾收集的线程数量，ParNew默认开启的收集线程与CPU的数量相同；

### Parallel Scavenge
**Parallel Scavenge收集器也是一个并行的多线程新生代收集器，使用复制算法**。目标是达到一个**可控制的吞吐量（Throughput）**。即减少垃圾收集时间，让用户代码获得更长的运行时间<br>
当应用程序运行在具有多个CPU上，对暂停时间没有特别高的要求时，即程序主要在后台进行计算，而不需要与用户进行太多交互。<br>
**设置参数**
- -XX:MaxGCPauseMillis：**控制最大垃圾收集停顿时间，大于0的毫秒数**。MaxGCPauseMillis设置得稍小，停顿时间可能会缩短，但也可能会使得吞吐量下降；可能导致垃圾收集发生得更频繁
- -XX:GCTimeRatio：**设置垃圾收集时间占总时间的比率，0<n<100的整数**。
GCTimeRatio相当于设置吞吐量大小;垃圾收集执行时间占应用程序执行时间的比例的计算方法是：1 / (1 + n);例如，选项-XX:GCTimeRatio=19，设置了垃圾收集时间占总时间的5%  1/(1+19)；默认值是1%  1/(1+99)，即n=99；
- -XX:+UseAdptiveSizePolicy：开启这个参数后，就不用手工指定一些细节参数，如：新生代的大小（-Xmn）、Eden与Survivor区的比例（-XX:SurvivorRation）、晋升老年代的对象年龄（-XX:PretenureSizeThreshold）等；会根据当前系统运行情况收集性能监控信息，**动态调整**这些参数，以提供最合适的停顿时间或最大的吞吐量，这种调节方式称为**GC自适应的调节策略**

## 老年代收集器
### Serial Old
Serial Old 是 Serial收集器的老年代版本，它同样是一个单线程收集器，使用**标记-整理**算法
### Parallel Old
Parallel Old收集器是Parallel Scavenge收集器的老年代版本，使用多线程和**标记-整理**算法
### CMS
**CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间**为目标的收集器,使用**标记-清除**算法<br>
步骤
- 初始标记（CMS initial mark）：仅仅只是标记一下GC Roots能直接关联到的对象，速度很快，需要“Stop The World”。
- 并发标记（CMS concurrent mark）：进行GC Roots Tracing的过程，在整个过程中耗时最长。
- 重新标记（CMS remark）：为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短。此阶段也需要“Stop The World”。
- 并发清除（CMS concurrent sweep）

