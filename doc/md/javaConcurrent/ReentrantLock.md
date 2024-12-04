### 使用
```java
//两种构造器，不传参默认非公平锁
public ReentrantLock() { 
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```
当调用 lock.lock()方法时：
ReetrantLock.lock -> sync.lock(); 公平锁与非公平锁实现lock()区别在于:lock时就会尝试更改stat值。
Sync继承**AQS**类，acquire()方法就是获取资源方法，能成功拿到资源代码正常运行。资源获取失败会阻塞当前线程（LockSupport.park()）
```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    final void lock() {
        acquire(1);
    }
}    
static final class NonfairSync extends Sync {
     private static final long serialVersionUID = 7316153563782823691L;
     final void lock() {
         if (compareAndSetState(0, 1)) //非公平锁尝试更改state值
             setExclusiveOwnerThread(Thread.currentThread());
         else
             acquire(1);
     }
 }
```