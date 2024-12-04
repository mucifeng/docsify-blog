## 基本使用
```java
public class SemaphoreTest implements Runnable{
    final Semaphore semaphore = new Semaphore(5);
    @Override
    public void run() {
        try {
            semaphore.acquire();
            Thread.sleep(2000);
            System.out.println(Thread.currentThread().getName() + ":done");
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            semaphore.release();
        }
    }
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(20);
        SemaphoreTest test = new SemaphoreTest();
        for (int i=0; i<20; i++){
            executorService.submit(test);
        }
    }
}
//程序线程大致是按照5个一组的方式批量运行
```
## 假如在使用Semaphore.acquire()时，先运行了Semaphore.release(n)。会是什么情况？
```java
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(20);
        SemaphoreTest test = new SemaphoreTest();
        test.semaphore.release(15);
        for (int i=0; i<20; i++){
            executorService.submit(test);
        }
    }
```
其实可以观察到，20个线程几乎是一起同时运行的，也就是说Semaphore的通行证增加到了20个
### 为什么会发生这样的情况？
首先确定Semaphore内部是使用Sync extends AbstractQueuedSynchronizer。仔细追究release()方法，可以看到
Sync对象重写的tryReleaseShared(int releases)方法。在方法体中并没有指定Semaphore必须先有acquire()调用才能执行release().
这就导致AQS中的state值增加，使得更多的线程执行acquire()通过。
```java
    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) 
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))
                return true;
        }
    }
```
