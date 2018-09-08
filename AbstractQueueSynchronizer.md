# AbstractQueuedSynchronizer
[toc]
## 数据结构

```
AbstractQueuedSynchronizer类底层的数据结构是双向链表，是队列的一种实现。
其中Sync queue（同步队列）是双向链表，
包括head结点和tail结点，head结点主要用作后续的调度。
而Condition queue不是必须的，其是一个单向链表，
只有当使用Condition时，才会存在此单向链表。并且可能会有多个Condition queue。
```

## 主要源码分析

### 类的继承关系

```
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable//可序列化，只是一个接口
    
AbstractOwnableSynchronizer类

public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {
    //版本号
    private static final long serialVersionUID = 3737899427754241961L;
    protected AbstractOwnableSynchronizer() { }

    //独占模式下的线程，不进行序列化
    private transient Thread exclusiveOwnerThread;
    //独占线程的set方法
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
    //独占线程的get方法
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}


```

### 内部类

> Node类

```
static final class Node {
        //共享模式
        static final Node SHARED = new Node();
        //独占模式
        static final Node EXCLUSIVE = null;

        // 结点状态
        // CANCELLED，值为1，表示当前的线程被取消
        static final int CANCELLED =  1;
        // SIGNAL，值为-1，表示当前节点的后继节点包含的线程需要运行，也就是unpark
        static final int SIGNAL    = -1;
        // CONDITION，值为-2，表示当前节点在等待condition，也就是在condition队列中
        static final int CONDITION = -2;
        // PROPAGATE，值为-3，表示当前场景下后续的acquireShared能够得以执行
        static final int PROPAGATE = -3;
        // 值为0，表示当前节点在sync队列中，等待着获取锁
        volatile int waitStatus;
        //前驱结点
        volatile Node prev;
        //后继结点
        volatile Node next;
        //结点所对应的线程
        volatile Thread thread;
        //下一个等待者
        Node nextWaiter;
        //是否是共享模式
        final boolean isShared() {
            return nextWaiter == SHARED;
        }
        //获取前驱结点，null则抛出空指针异常
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
        //无参
        Node() {    // Used to establish initial head or SHARED marker
        }
        //当前线程与下一个等待线程
        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }
        //当前线程及其状态
        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

>  ConditionObject类

```
public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        //condition队列的头节点与尾节点，均不可序列化
        private transient Node firstWaiter;
        private transient Node lastWaiter;
        //空构造函数
        public ConditionObject() { }
        // Internal methods

        //添加新的waiter到队列
        private Node addConditionWaiter() {
            Node t = lastWaiter;//获取尾节点
            //尾节点不为null，并且状态不为condition
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();//清除等待状态的结点
                t = lastWaiter;//并将最后一个结点赋值给t
            }
            //新建一个结点
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                //尾节点为null，则将其赋值给头节点
                firstWaiter = node;
            else//否则将其赋值给尾节点的下一个结点
                t.nextWaiter = node;
            lastWaiter = node;//更新尾节点
            return node;
        }

        //从等待队列移动到拥有锁的等待队列。
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)//若头结点的下一个结点为null，则将头尾结点设为null
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
                    //将结点从condition队列转移到sync队列失败并且condition队列中的头结点不为空，一直循环
        }

        //将所有线程从这个条件的等待队列移动到拥有锁的等待队列。 
        private void doSignalAll(Node first) {
        //将condition队列的头尾结点都设置为null
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;//获取first结点的nextWaiter域的结点
                first.nextWaiter = null;//设置first结点的nextWaiter域为空
                transferForSignal(first);//将first结点从condition队列转移到sync队列
                first = next;//重新设置first
            } while (first != null);
        }
        //从condition队列中清除状态为cancel的结点
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;//保存condition队列的头节点
            Node trail = null;
            while (t != null) {//当前结点不为null
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {//当前节点的状态不为condition状态
                    t.nextWaiter = null;//将当前结点的nextWaiter域为空
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else//t结点的状态为condition状态
                    trail = t;
                t = next;
            }
        }
        //唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒。在await返回之前，该线程必须重新获取锁
        public final void signal() {
            if (!isHeldExclusively())//不被当前线程独占，抛出异常
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;//保存condition队列头节点
            if (first != null)//头节点不为null
                doSignal(first);//唤醒一个等待线程
        }
        //唤醒所有等待线程。在从await返回之前，每个线程都必须重新获取锁。
        public final void signalAll() {
            if (!isHeldExclusively())//不被当前线程独占，抛出异常
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;//保存队列头节点
            if (first != null)
                doSignalAll(first);
        }
        //等待，当前线程在接到信号之前一直处于等待状态，不响应中断
        public final void awaitUninterruptibly() {
            Node node = addConditionWaiter();//添加一个结点到等待队列
            int savedState = fullyRelease(node);//获取释放的状态
            boolean interrupted = false;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);//阻塞当前线程
                if (Thread.interrupted())
                    interrupted = true;//设置interrupted状态
            }
            if (acquireQueued(node, savedState) || interrupted)
                selfInterrupt();
        }


}

```

### 类属性

```
    //头结点，不可序列化
    private transient volatile Node head;
    //尾节点
    private transient volatile Node tail;
    //状态
    private volatile int state;
    //自旋时间
    static final long spinForTimeoutThreshold = 1000L;
    //Unsafe类的实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    //个属性的偏移地址内存偏移地址
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long waitStatusOffset;
    private static final long nextOffset;
    //进行赋值
    static {
        try {
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
            waitStatusOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("waitStatus"));
            nextOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("next"));

        } catch (Exception ex) { throw new Error(ex); }
    }
```

### 类的构造函数

```
默认的空构造函数
```

### 主要方法

> acquire方法

```
    //以独占模式获取资源，忽略中断
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    //addWaiter方法：将线程封装成结点放入sync queue
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;//保存尾节点
        if (pred != null) {//尾节点不为null
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {//判断pred是否为尾节点
                pred.next = node;//将结点放到尾节点后
                return node;
            }
        }
        //尾结点为空(即还没有被初始化过)，或者是compareAndSetTail操作失败，则入队列
        enq(node); 
        return node;
    }
    //enq方法：将结点插入队列中；无限循环确保结点的插入
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // 尾节点为null则还未被初始化
                if (compareAndSetHead(new Node()))//头节点为null
                    tail = head;//直接设置头尾结点为新生成结点
            } else {//尾节点不为null
                node.prev = t;//将结点接到尾节点尾部
                if (compareAndSetTail(t, node)) {//比较结点t是否为尾结点，若是则将尾结点设置为node
                    t.next = node;
                    return t;
                }
            }
        }
    }
    //sync队列中在独占且忽略中断的情况下，是否可以获取到资源
    final boolean acquireQueued(final Node node, int arg) {
        //失败标志
        boolean failed = true;
        try {
            //中断标志
            boolean interrupted = false;
            //无限循环，直到满足某一条件
            for (;;) {
                final Node p = node.predecessor();//获取node结点的前驱结点
                if (p == head && tryAcquire(arg)) {//前驱为头节点，并且成功获取锁
                    setHead(node);//设置头节点
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())//获取失败，检查并更新结点状态
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    
```

> release方法

```
    //以独占模式释放对象
    public final boolean release(int arg) {
        if (tryRelease(arg)) {//释放成功
            Node h = head;//保存头结点
            if (h != null && h.waitStatus != 0)//头节点不为null，且头结点状态不为0；
                unparkSuccessor(h);//
            return true;
        }
        return false;
    }
```

