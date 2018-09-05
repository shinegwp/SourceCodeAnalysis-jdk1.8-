## ReentrantLock
[toc]
### 属性

```
private final Sync sync;
```

### 内部类

```
FairSync 与 NonfairSync 都是 AQS 的子类, lock的获取, 
释放主逻辑都是交由 AQS来完成, 则子类实现模版方法(也就是模版模式)

abstract static class Sync extends AbstractQueuedSynchronizer
static final class NonfairSync extends Sync
static final class FairSync extends Sync
```

### 构造方法
``` 
    //默认为非公平
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

### 主要方法

> 加锁lock()

```
    
    public void lock() {
        sync.lock();
    }
    //非公平
    final void lock() {
        if(compareAndSetState(0, 1))  // 先cas改变一下 state 成功就表示获取
            setExclusiveOwnerThread(Thread.currentThread()); // 获取成功设置 exclusiveOwnerThread
        else
            acquire(1); // 获取不成功, 调用 AQS 的 acquire 进行获取
    }
    
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();//获取当前线程
        int c = getState();//获取线程状态
        if (c == 0) {//还未获取到锁
            if (compareAndSetState(0, acquires)) {//直接CAS获取lock
            setExclusiveOwnerThread(current);//获取成功设置当前线程
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {//以获取到锁是否是当前线程
            int nextc = c + acquires;//锁的重入
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    
    //公平
    final void lock() {
        acquire(1);//调用AQS的acquire进行获取
    }
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //先判断AQS Sync Queue里面是否有线程等待获取锁,若没有直接CAS获取lock
            if (!hasQueuedPredecessors() &&compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```

> 释放锁unlock

```
    public void unlock() {
        sync.release(1);
    }
    //aqs中
    public final boolean release(int arg) {
        if (tryRelease(arg)) {//调用子类，完全释放返回true
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);//有后继结点，将其唤醒
            return true;
        }
        return false;
    }
    //reentrantlock$sync
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;//释放一个锁
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
```
