### 线程池的优势
1. 降低系统资源消耗，通过重用已存在的线程，降低线程创建和销毁造成的消耗；
2. 提高系统响应速度，当有任务到达时，通过复用已存在的线程，无需等待新线程的创建便能立即执行；
3. 方便线程并发数的管控。因为线程若是无限制的创建，可能会导致内存占用过多而产生OOM，并且会造成cpu过度切
4. 提供更强大的功能，延时定时线程池。

### 参数
```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {}
```
- corePoolSize（线程池基本大小）：当向线程池提交一个任务时，若线程池已创建的线程数小于corePoolSize，即便此时存在空闲线程，也会通过创建一个新线程来执行该任务，直到已创建的线程数大于或等于corePoolSize时，（除了利用提交新任务来创建和启动线程（按需构造），也可以通过 prestartCoreThread() 或 prestartAllCoreThreads() 方法来提前启动线程池中的基本线程。）
- maximumPoolSize（线程池最大大小）：线程池所允许的最大线程个数。当队列满了，且已创建的线程数小于maximumPoolSize，则线程池会创建新的线程来执行任务。另外，对于无界队列，可忽略该参数。
- keepAliveTime（线程存活保持时间）当线程池中线程数大于核心线程数时，线程的空闲时间如果超过线程存活时间，那么这个线程就会被销毁，直到线程池中的线程数小于等于核心线程数。
- workQueue（任务队列）：用于传输和保存等待执行任务的阻塞队列。
- threadFactory（线程工厂）：用于创建新线程。threadFactory创建的线程也是采用new Thread()方式，threadFactory创建的线程名都具有统一的风格：pool-m-thread-n（m为线程池的编号，n为线程池内的线程编号），工作中常用语修改线程名字，与功能点相契合的名字
- handler（线程饱和策略）：当线程池和队列都满了，再加入线程会执行此策略。

### 执行流程
1. 判断核心线程池是否已满，没满则创建一个新的工作线程来执行任务。已满则将任务添加进入任务队列
2. 判断任务队列是否已满，没满则将新提交的任务添加在工作队列，已满且线程数小于maximumPoolSize则创建新线程
3. 判断整个线程池是否已满,线程数量已经等于maximumPoolSize，并且队列也满了。则执行handler拒绝方法（饱和策略）

### 线程池的位运算
**ThreadPoolExecutor 用一个 int 类型来表示当前线程池的 运行状态 和 线程有效数量**
不同平台的 int 类型的范围不一样，假定用 32 位的二进制表示 int 类型：
- 高 3 位表示 线程池运行状态
- 低 29 位表示 线程有效数量

```java

 private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//线程个数的表示 位数。不管什么平台，高三位始终是状态
private static final int COUNT_BITS = Integer.SIZE - 3;
// 线程最大数量
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

//高三位表示 线程池状态 
private static final int RUNNING    = -1 << COUNT_BITS;  //高三位 111 
private static final int SHUTDOWN   =  0 << COUNT_BITS; // 高三位为 000
private static final int STOP       =  1 << COUNT_BITS; //高三位为 001
private static final int TIDYING    =  2 << COUNT_BITS; //高三位为 010
private static final int TERMINATED =  3 << COUNT_BITS; //高三位为 011
// 获取线程池状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 获取线程池有效线程数量
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

**参考文章:**<br>
[Java线程池详解](https://www.jianshu.com/p/7726c70cdc40)<br>
[Java8线程池源码](https://www.jianshu.com/p/a60d40b0e4e9)<br>

