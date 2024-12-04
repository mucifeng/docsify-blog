# 使用
AQS的核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，
并将共享资源设置为锁定状态，如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，
这个机制AQS是用CLH队列锁实现的，即将暂时获取不到锁的线程加入到队列中。<br>
简单点来解释就是：多个线程同时获取AQS中的资源,肯定存在不能获取资源的线程，这类线程封装进入AQS内部队列中，等待其他成功获得资源的线程
同步代码执行完之后唤醒。AQS就提供这样一种功能框架，AQS内部使用 state记录资源数，同时继承AbstractOwnableSynchronizer类
## 独占锁机制
* 当**state=0**时，表示无锁状态
* 当**state>0**时，表示已经有线程获得了锁，也就是state=1，但是因为ReentrantLock允许重入，所以同一个线程多次获得同步锁的时候，state会递增，比如重入5次，那么state=5。 而在释放锁的时候，同样需要释放5次直到state=0其他线程才有资格获得锁

### 获取锁
以ReentrantLock深入理解AQS执行流程。（公平锁代码）
```java
public class ReentrantLock {
        public void lock() { sync.lock(); }
}
abstract static class Sync extends AbstractQueuedSynchronizer{}
static final class FairSync extends Sync{
    final void lock() { acquire(1); }
}
```
调用链为：ReentrantLock.lock() -> FairSync.lock() -> AQS.acquire(1);
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
tryAcquire(arg) 是AQS提供的一个钩子方法，需要AQS子类自己根据实际情况实现。用于尝试获取锁

```java
//添加Node到等待队列中。
private Node addWaiter(Node mode) { 
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) { //初始状态，队列尾节点tail=null。若tail ！= null 表示队列已经初始化
        node.prev = pred;
        if (compareAndSetTail(pred, node)) { //尝试将当前线程节点加入到队尾。可能失败，会在方法enq(node)中重新加入队尾
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // 队列初始化，发生竞争时，队列会进行初始化操作。初始化完成后，由于for(;;)，线程会再次运行else中的代码块
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) { 
                //t此时指向tail,所以可以CAS成功，将tail重新指向Node。
                // 此时t为更新前的tail的值，即指向空的头结点，t.next=node。返回当前node的prev
                t.next = node;
                return t;
            }
        }
    }
}
```
由以上代码可以知道：
1. 当只有一个线程使用lock(底层AQS)时，在tryAcquire(arg)方法中就应该修改state的值成功，并且进入到同步方法，不需要执行后续的addWaiter()方法。
查看ReentrantLock中的方法，可以看到出了修改state值外，还setExclusiveOwnerThread(currentThread);将AQS中的exclusiveOwnerThread指向了当前线程。**类比synchronized中的偏向锁**
2. 当有多个线程同时调用lock()方法，AQS中的队列将会进行初始化并且将多个线程封装成Node类，CAS操作依次放入队列中

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) { //只有node的前驱是head，才有资格竞争尝试再次竞争资源
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&  //如果获取锁失败，则根据节点的waitStatus决定是否需要挂起线程
                parkAndCheckInterrupt()) 
                interrupted = true;
        }
    } finally {
        if (failed) // 在for(;;)中如果抛出异常那么failed=true,就会执行取消操作
            cancelAcquire(node);
    }
}
private void setHead(Node node) { //setHead方法中可以看到仅仅是保证当前node为head，同时将前驱和线程都等于null
    head = node;
    node.thread = null;
    node.prev = null;
}
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL) //node前驱是SIGNAL状态,那么node中线程肯定是需要被挂起
       return true;
    //如果前节点的状态大于0，即为CANCELLED状态时，则会从前节点开始逐步循环找到一个没有被“CANCELLED”节点设置为当前节点的前节点
    // 在下次循环执行shouldParkAfterFailedAcquire时，返回true。这个操作实际是把队列中CANCELLED的节点移除
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else { // 如果前继节点为“0”或者“共享锁”状态，则设置前继节点为SIGNAL状态。
        //仅仅是更改节点的状态，重新进入acquireQueued()中的循环体中，再次判断能否获得到独占锁。尽可能的不使用park()挂起线程
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
private final boolean parkAndCheckInterrupt() { //调用park()方法挂起线程，
    LockSupport.park(this);
    return Thread.interrupted();
}
```
1. 首先进入acquireQueued()方法中后，先判断node的前驱是不是head节点。只有前驱节点是head的才有资格尝试获取锁
2. 没有获取到资源的线程开始查看前驱状态，根据前驱状态进行一次自旋或者直接进入park()挂起状态

### 释放锁
调用lock.unlock()方法的时候，底层调用AQS release方法：释放锁、唤醒park线程
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) { //tryRelease方法由AQS子类实现，主要是释放锁资源
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
private void unparkSuccessor(Node node) { //释放锁流程中传入的节点是head节点
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0); 
    Node s = node.next;
    //判断后继节点是否为空或者是否是取消状态
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0) 
                s = t;
    }
// 节点唤醒后继续执行parkAndCheckInterrupt()的方法后续
    if (s != null)
        LockSupport.unpark(s.thread); //释放许可
}
```
## 共享锁
以 CountDownLatch 为基础了解AQS共享锁代码,以下是CountDownLatch部分代码
```java
CountDownLatch latch = new CountDownLatch(3);
public CountDownLatch(int count) { this.sync = new Sync(count);}
private static final class Sync extends AbstractQueuedSynchronizer {
    Sync(int count) {
        setState(count);
    }
｝
```
1. 底层调用AQS setState方法为当前共享锁能同时共享线程数量

### 获取锁
```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0) //尝试获取共享锁，但是不一定能获得
        doAcquireShared(arg);
}
private void doAcquireShared(int arg) { 
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
共享锁的实现大致和独占锁一致，当线程没有获得锁的时候，会将当前线程挂起

### 释放锁
```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) { //尝试释放共享锁
        doReleaseShared();
        return true;
    }
    return false;
}

private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h); //唤醒后继节点
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                
        }
        if (h == head)            
            break;
    }
}
```

**参考文章:**<br>
[深入分析AQS实现原理](https://segmentfault.com/a/1190000017372067)<br>
[Java并发之AQS详解](https://www.cnblogs.com/waterystone/p/4920797.html)<br>
