# java.util.concurrent
## JUC介绍
目的就是为了更好的支持高并发任务，让开发者利用这个包进行的多线程编程时可以有效的减少竞争条件和死锁线程。
## JUC结构
![JUC结构](https://img-blog.csdn.net/20151222151848290) 

## AQS概念
AQS 的全称为（AbstractQueuedSynchronizer），这个类在 java.util.concurrent.locks 包下面。  
是一个用来构建锁和同步器的框架，使用 AQS 能简单且高效地构造出应用广泛的大量的同步器，  
比如 ReentrantLock，Semaphore，其他的诸如 ReentrantReadWriteLock，SynchronousQueue，FutureTask(jdk1.7) 等等皆是基于 AQS 的。  
![AQS](https://camo.githubusercontent.com/e7314fa1abf26ccbb14aafc2cb02ccec0965b9e8/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f4a6176612532302545372541382538422545352542412538462545352539312539382545352542462538352545352541342538372545462542432539412545352542392542362545352538462539312545372539462541352545382541462538362545372542332542422545372542422539462545362538302542422545372542422539332f4151532e706e67)  

## AQS原理
> 核心思想：
>> **如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。  
>> 如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，  
>> 这个机制 AQS 是用 CLH 队列锁实现的，即将暂时获取不到锁的线程加入到队列中。**  
>>> CLH(Craig,Landin,and Hagersten)队列是一个虚拟的双向队列（虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系）。 
>>> AQS 是将每条请求共享资源的线程封装成一个 CLH 锁队列的一个结点（Node）来实现锁的分配。 
>>> ![CLH](https://camo.githubusercontent.com/55090fdc22963d41a3e56d5dadb17dfa7ad6379f/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f4a6176612532302545372541382538422545352542412538462545352539312539382545352542462538352545352541342538372545462542432539412545352542392542362545352538462539312545372539462541352545382541462538362545372542332542422545372542422539462545362538302542422545372542422539332f434c482e706e67)  

> 底层代码：
 >> **使用一个volatile关键字修饰的 int 成员变量来表示同步状态，通过内置的 FIFO 队列来完成获取资源线程的排队工作，使用 CAS 对该同步状态进行原子操作实现对其值的修改。**  
 >>> 核心成员和内部类  
 ```java 
 // 部分源码
 public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer implements java.io.Serializable {
  
  // 头结点，直接把它当做 当前持有锁的线程 是最好理解的
  private transient volatile Node head;
  // 阻塞的尾节点，每个新的节点进来，都插入到最后，也就形成了一个隐视的链表
  private transient volatile Node tail;
  // 共享变量，使用volatile修饰保证线程可见性，0代表没有被占用，大于0代表有线程持有当前锁
  // 之所以说大于0，而不是等于1，是因为锁可以重入，每次重入都加上1
  private volatile int state;
 
  // 返回同步状态的当前值
  protected final int getState() {
        return state;
  }
  // 设置同步状态的值
  protected final void setState(int newState) {
        state = newState;
  }
  // 原子地（CAS操作）将同步状态值设置为给定值update如果当前同步状态的值等于expect（期望值）
  protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
  }
 }
 ```  
 
 >>> AQS通过维护一个等待获取锁的线程队列来管理（获取资源失败入队/唤醒出队）抢占锁的线程，这个队列是一种CLH队列的变体
 ```java
/** 队列中的结点,每个等待中的结点可能会处于几种不同的等待状态 **/
static final class Node{
    static final Node SHARED = new Node(); //表示结点对应线程想共享地抢占锁
    static final Node EXCLUSIVE = null;    //表示结点对应线程想独占地抢占锁
    volatile int waitStatus;   //结点的等待状态，CLH队列中初始默认为0,Condition队列中初始默认为-2
    static final int CANCELLED = 1; // 结点已被取消,表示线程放弃抢锁,结点状态以后不再变直到GC回收它
    static final int SIGNAL = -1;//结点的后继已经或很快就阻塞,在结点释放锁或被取消时要唤醒其后面第1个非CANCELLED结点
    
    /** Condition队列中结点的状态,CLH队列中结点没有该状态,当Condition的signal方法被调用,
    Condition队列中的结点被转移进CLH队列并且状态变为0 **/
    static final int CONDITION = -2;
    
    //与共享模式相关,当线程以共享模式去获取或释放锁时,对后续线程的释放动作需要不断往后传播
    static final int PROGAGATE = -3;
    
    volatile Node prev;  //指向结点在队列中的前驱
    volatile Node next;  //指向结点在队列中的后继
    volatile Thread thread;  //使当前结点进队的线程（与当前结点关联的线程）
    Node nextWaiter;//Condition队列中指向结点在队列中的后继;在CLH队列中共享模式下值取SHARED,独占模式下为null
    final boolean isShared() {  //若结点在CLH队列中以共享模式等待则返回true
        return nextWaiter == SHARED;
    }
    final Node predecessor() throws NullPointerException {  //返回结点前驱
        Node p = prev;
        if (p == null) throw new NullPointerException();
        else return p;
    }
    Node() {}  // Used to establish initial head or SHARED marker
    Node(Thread thread, Node mode) {  //往CLH队列中添加结点时调用此构造器构造结点
        this.nextWaiter = mode;
        this.thread = thread;
    }
    Node(Thread thread, int waitStatus) { //往Condition队列中添加结点时调用此构造器构造结点
        this.waitStatus = waitStatus; //传入的waitStatus为CONDITION
        this.thread = thread;
    }
}
 ```
 >> AQS核心模版方法  
 **AQS使用模板方法模式**，如果需要自定义同步器一般的方式是这样：
 >>+ 使用者继承 AbstractQueuedSynchronizer 并重写指定的方法。  
 >>++ isHeldExclusively()//该线程是否正在独占资源。只有用到condition才需要去实现它。  
 >>++ tryAcquire(int)//独占方式。尝试获取资源，成功则返回true，失败则返回false。  
 >>++ tryRelease(int)//独占方式。尝试释放资源，成功则返回true，失败则返回false。  
 >>++ tryAcquireShared(int)//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。  
 >>++ tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。  
 >>+ 将 AQS 组合在自定义同步组件的实现中，并调用其模板方法，而这些模板方法会调用使用者重写的方法。  
 
 >>> Exclusive独占
 
 >>> acquire(int arg)方法-->独占模式下获取锁  
 >>> 线程获取锁，成功直接返回，失败则进入等待队列中排队获取锁。  
 >>> 在获取锁的过程中不响应发生的中断而是记录下来，最后检查是否中断过，如果中断过再将中断标记补上。  
 ```java
 public final void acquire(int arg) {  //独占模式获取锁的模板方法
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
//线程具体的获取锁方法,此方法由具体同步器(即AQS子类)实现,获取锁成功时要返回true,失败返回false
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
/**
 * 根据当前线程的获取锁模式创建一个结点并加入队列中
 * @param mode Node.EXCLUSIVE表示独占模式, Node.SHARED表示共享模式
 * @param return 返回创建的新结点
**/
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    //先在当前方法中用CAS进队试一次,不成功则进入enq()方法中反复尝试直到进队成功
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;  //注意这里先置了node的前驱
        if (compareAndSetTail(pred, node)) { //CAS操作尝试原子地将tail置为指向当前新建结点
           pred.next = node; //成功说明tail已指向当前结点,则给当前结点前驱的next指针赋值
           return node;
        }
    }
    enq(node);  //失败进入enq()方法反复尝试直到成功
    return node;
}
//反复尝试直到结点进入等待队列成为队尾,注意该方法返回输入结点的前驱结点
private Node enq(final Node node) {
    for (;;) {   //经典“CAS + 失败重试”
        Node t = tail;
        if (t == null) { //需要初始化等待队列
           if (compareAndSetHead(new Node()))
                tail = head;
        } else {    //下面这部分和addWaiter方法中一样
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
//已经进入等待队列的线程在队列中独占（且不响应中断）地获取锁的行为
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;  //获取失败标志,初值为true
    try {
        boolean interrupted = false; //记录线程在队列中获取锁的过程中是否发生过中断
        //死循环,在循环中线程可能会被阻塞0次,1次,或多次,直到获取锁成功才跳出循环,方法返回
        for (;;) {
            final Node p = node.predecessor(); //获取当前结点的前驱
            //只有当前结点的前驱是头结点,当前线程才被允许尝试获取锁;只有获取锁成功才会跳出循环方法返回
            if (p == head && tryAcquire(arg)) {
                setHead(node);  //获取锁成功会将当前结点设为头结点
                p.next = null; // help GC
                failed = false;
                return interrupted;  //返回是否发生过中断
            }
            //线程不被允许获取锁或获取失败都会进入下面的方法检查是否自己可以阻塞;被唤醒后记录是否是被中断唤醒的
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;  //如果是被中断唤醒的,会记录下来被中断过
        }
    } finally {
        if (failed)  //线程发生异常则取消获取锁行为
            cancelAcquire(node);
    }
}
//等待队列中的线程不被允许获取锁或尝试获取锁失败后调用,检查自己是否可以阻塞
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus; //获取前驱的等待状态
    if (ws == Node.SIGNAL)  //前驱的等待状态已经是SIGNAL,则当前线程可以放心阻塞
        return true;  //表示要阻塞
   if (ws > 0) {  //前驱等待状态为CANCELLED,说明前驱已无效
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);//不断向前寻找状态不为CANCELLED的结点,同时将无效结点链成一个不可达的环,便于GC
        pred.next = node;  //找到状态不为CANCELLED的结点
    } else {//前驱状态是PROGAGATE或0时,将其前驱的状态设为SIGNAL,在再次尝试失败后才阻塞(?)
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;  //表示还要再尝试
}
//线程阻塞,被唤醒时会返回是否是被中断唤醒的
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
 ```  
 
 >>> release(int arg)方法-->独占模式下释放锁  
 >>> 调用tryRelease尝试释放锁，如果成功了，需要查看head的waitStatus状态，如果是0，表示CLH队列中没有后继节点了，不需要唤醒后继；  
 >>> 否则调用unparkSuccessor唤醒后继。而unparkSuccessor唤醒后继的原理是：找到node后面的第一个非cancelled结点进行唤醒。   
 ```java
 public final boolean release(int arg) {
        if (tryRelease(arg)) {  //尝试释放锁
            Node h = head;
            //如果head的waitStatus为0说明没有后继了,因为如果有后继,它的后继在阻塞前一定会把它的waitStatus设为SIGNAL
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h); //唤醒后继
            return true;
        }
        return false;
}
//唤醒后继
private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;  //node是获取了锁的结点
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);  //唤醒后继前,重置waitStatus为0
        Node s = node.next;  //node的next指针不一定指向其后继,当node的状态为cancelled的时候,其next指向自己
        if (s == null || s.waitStatus > 0) {  //这里的s == null的条件判断不理解(?)
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)  //从后往前找node的后继中第一个没有被取消的结点
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);  //唤醒该结点线程
}
 ```
 >>> Share共享  
 
 >>> 在共享资源允许范围内会有多个线程同时共享该锁，剩下的线程就被加入到CLH等待队列中排队阻塞等待；  
 >>> 当持有锁的线程释放锁时，它会唤醒在队列中等待的后继，而这个后继在获取锁之后会继续检查资源的剩余量，如果还有剩余，它会接着唤醒自己的后继。  
 >>> **共享模式下，线程无论是在获取锁或者释放锁的时候，都可能会唤醒其后继，而且在共享资源允许的条件下会引起多个线程被连续唤醒。  
 >>> 如果有多个线程同时获取了共享锁，则head指向的那个是CLH队列中最后一个持有锁的线程，其他的都已经出队了。**  
 
 >>> acquireShared(int arg)方法--->共享模式下获取锁  
 ```java
 public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)  //尝试获取锁,成功返回值大于等于0,失败返回值小于0
        doAcquireShared(arg); //如果失败,则调用doAcquireShared方法获取锁
}
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED); //线程入队
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor(); //取当前结点的前驱
            if (p == head) {  //前驱为头结点才允许尝试获取锁,这里体现了入队之后获取资源的顺序性,只要入队,就是顺序的了
                int r = tryAcquireShared(arg); 
                if (r >= 0) { //获取锁成功
                    setHeadAndPropagate(node, r); //将当前线程设为头,然后可能执行对后继SHARED结点的连续唤醒
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            //获取锁失败,设置前驱waitStatus为SIGNAL,然后阻塞,这个过程与独占模式相同
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)  //获取锁过程中发生异常造成未成功获取,则取消获取
            cancelAcquire(node);
    }
}
private void setHeadAndPropagate(Node node, int propagate) { //propagate是资源剩余量,从上面的调用中可以看到
    Node h = head;  //将旧的头结点先记录下来
    setHead(node);  //将当前node线程设为头结点,node已经获取了锁
    //如果资源有剩余量,或者原来的头结点的waitStatus小于0,进一步检查node的后继是否也是共享模式
    if (propagate > 0 || h == null || h.waitStatus < 0 || (h = head) == null || h.waitStatus < 0) {
        Node s = node.next; //得到node的后继
        if (s == null || s.isShared())  //如果后继是共享模式或者现在还看不到后继的状态,则都继续唤醒后继线程
            doReleaseShared();
    }
}
private void doReleaseShared() {
    for (;;) {
        Node h = head;  //记录下当前的head
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {  //如果head的waitStatus为SIGNAL,一定是它的后继设的,共享模式下要唤醒它的后继
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) //先将head的waitStatus设置为0,成功后唤醒其后继
                    continue;        // loop to recheck cases
                unparkSuccessor(h); //关键,若成功唤醒了它的后继,它的后继就会去获取锁,如果获取成功,会造成head的改变
            } else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) //没有后继结点,设为PROPAGATE
                continue;           // loop on failed CAS
        }
        if (h == head) //若head发生改变,说明后继成功获取了锁,此时要检查新head的waitStatus,判断是否继续唤醒(下次循环)
            break; //head没有发生改变则停止持续唤醒
    }
}
 ```
 
 >>> releaseShared(int arg)方法--->共享模式下释放锁  
 ```java
 public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {  //如果释放锁成功
        doReleaseShared();  //启动对后继的持续唤醒
        return true;
    }
    return false;
}
private void doReleaseShared() {
    for (;;) {
        Node h = head;  //记录下当前的head
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {  //如果head的waitStatus为SIGNAL,一定是它的后继设的,共享模式下要唤醒它的后继
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) //先将head的waitStatus设置为0,成功后唤醒其后继
                    continue;        // loop to recheck cases
                unparkSuccessor(h); //关键,若成功唤醒了它的后继,它的后继就会去获取锁,如果获取成功,会造成head的改变
            } else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) //没有后继结点,设为PROPAGATE
                continue;           // loop on failed CAS
        }
        if (h == head) //若head发生改变,说明后继成功获取了锁,此时要检查新head的waitStatus,判断是否继续唤醒(下次循环)
            break; //head没有发生改变则停止持续唤醒
    }
}
 ```
