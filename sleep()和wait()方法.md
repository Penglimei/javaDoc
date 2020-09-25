# sleep()和wait()方法
## 区别
>+ 两者都可以暂停线程的执行,wait()通常被用于线程间交互/通信，sleep()通常被用于暂停执行;  
>+ sleep()方法是Thread类的静态方法,wait()是Object超类的成员方法;  
>+ sleep()方法导致了程序暂停执行指定的时间，让出cpu该其他线程，但是他的监控状态依然保持者，当指定的时间到了又会自动恢复运行状态,在调用sleep()方法的过程中，线程`不会释放已持有的对象锁`;  
>+ 调用wait()方法的时候，线程`会放弃已持有的对象锁`，进入此对象的等待锁定池，只有针对此对象调用notify()、notifyAll()方法后本线程才进入对象锁定池准备，或者可以使用 wait(long timeout)超时后线程会自动苏醒;  
## 使用
### sleep()方法
> 执行结果解释：  
> Thread A 先于Thread B 获得CPU时间片，Thread A 进入count()方法后，获取synchronized同步锁，`调用Thread.sleep()方法暂停执行，但是并不会释放已获取的同步锁sybchronized`，
Thread B 获取CPU时间片执行到count()方法后，由于`synchronized同步锁被Thread A 获取且未释放`，所以Thread B 被阻塞，
当Thread A 睡眠时间达到后，自动苏醒进入就入就绪态继而获取CPU时间片继续后序执行，`当Thread A 释放同步锁Synchronized后，Thread B 获取同步锁继续后序操作`。
```java
public class Main {
    
    public static class ThreadSleep implements Runnable{
        int count = 0;
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName()+" sleep...");
            count();
        }

        private void count() {
            while (count < 5){
                synchronized (this){
                    System.out.println(Thread.currentThread().getName()+" count is "+count);
                    count++;
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }

    }

    public static void main(String[] args) {
        System.out.println("begin test");
        ThreadSleep threadSleep = new ThreadSleep();
        Thread thread1 = new Thread(threadSleep,"Thread A ");
        Thread thread2 = new Thread(threadSleep,"Thread B ");
        thread1.start();
        thread2.start();
        System.out.println("end test");
    }
}


/**
 * 执行结果：
 *  begin test
 *  end test
 *  Thread A  sleep...
 *  Thread B  sleep...
 *  Thread A  count is 0
 *  Thread A  count is 1
 *  Thread A  count is 2
 *  Thread A  count is 3
 *  Thread A  count is 4
 *  Thread B  count is 5
 */

```

### wait()方法
>+ 在调用wait方法之前，线程必须获得该对象的对象级别锁，即`只能在同步方法或同步块中调用wait()方法`。  
>+ `notify()同wait()方法一样，也需要在同步方法或同步块中调用`，即在调用前，线程也必须获得`该对象的对象级别锁`。  
>+ wait和notify调用时，如果没有持有适当的锁，将会抛出IllegalMonitorStateException的异常，它是一个RuntimeException的子类。  
>>+ wait/notify等方法也依赖于monitor对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因  

#### wait(long timeout)
>+ Thread B线程调用wait()方法时，会释放已有的同步锁，因此Thread A 线程可以获取`同一个对象的同步锁synchronized`，执行后续操作。  
```java
public class Main {

    public static class ThreadWait implements Runnable{
        int count = 0;
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName()+" sleep...");
            count();
        }

        private void count() {
            while (count < 5){
                synchronized (this){
                    System.out.println(Thread.currentThread().getName()+" count is "+count);
                    count++;
                    try {
                        this.wait(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }

    }

    public static void main(String[] args) {
        System.out.println("begin test");
        ThreadWait threadSleep = new ThreadWait();
        Thread thread1 = new Thread(threadSleep,"Thread A ");
        Thread thread2 = new Thread(threadSleep,"Thread B ");
        thread1.start();
        thread2.start();
        System.out.println("end test");
    }
}
/**
 * 执行结果：
 *  begin test
 *  end test
 *  Thread B  sleep...
 *  Thread A  sleep...
 *  Thread B  count is 0
 *  Thread A  count is 1
 *  Thread A  count is 2
 *  Thread B  count is 3
 *  Thread A  count is 4
 *  Thread B  count is 5
 */
```

#### wait()、notify()
>+ `必须是同一个对象的同步锁`。  
> waitThread和notifyThread获取的都是同一个对象lock的同步锁。  
```java
public class Main {
    public static class notifyThread extends Thread{
        private final Object lock;

        public notifyThread(Object lock) {
            super();
            this.lock = lock;
        }

        @Override
        public void run() {
            synchronized (lock){
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName()+" notify start "+System.currentTimeMillis());
                lock.notify();
                System.out.println(Thread.currentThread().getName()+" notify end "+System.currentTimeMillis());
            }
        }
    }

    public static class waitThread extends Thread{
        private final Object lock;

        public waitThread(Object lock) {
            super();
            this.lock = lock;
        }

        @Override
        public void run() {
            synchronized (lock){
                System.out.println(Thread.currentThread().getName()+" wait start "+System.currentTimeMillis());
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName()+" wait end "+System.currentTimeMillis());
            }
        }
    }

    public static void main(String[] args) {
        System.out.println("begin test");
        Object lock = new Object();
        waitThread waitThread = new waitThread(lock);
        waitThread.start();
        notifyThread notifyThread = new notifyThread(lock);
        notifyThread.start();
        System.out.println("end test");
    }
}


/**
 * 执行结果：
 *  begin test
 *  Thread-0 wait start 1600937628595
 *  end test
 *  Thread-1 notify start 1600937629599
 *  Thread-1 notify end 1600937629599
 *  Thread-0 wait end 1600937629599
 */
```
>+ 如果wait()、notify()调用不同对象的同步锁  
> notifyThread对象获取对象lock2的同步锁后又释放掉了同步锁，但是waitThread对象一直等待的是对象lock1的同步锁，所以waitThread对象一直处于等待状态。  
```java
public class Main {
    public static class notifyThread extends Thread{
        private final Object lock;

        public notifyThread(Object lock) {
            super();
            this.lock = lock;
        }

        @Override
        public void run() {
            synchronized (lock){
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName()+" notify start "+System.currentTimeMillis());
                lock.notify();
                System.out.println(Thread.currentThread().getName()+" notify end "+System.currentTimeMillis());
            }
        }
    }

    public static class waitThread extends Thread{
        private final Object lock;

        public waitThread(Object lock) {
            super();
            this.lock = lock;
        }

        @Override
        public void run() {
            synchronized (lock){
                System.out.println(Thread.currentThread().getName()+" wait start "+System.currentTimeMillis());
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName()+" wait end "+System.currentTimeMillis());
            }
        }
    }

    public static void main(String[] args) {
        System.out.println("begin test");
        Object lock1 = new Object();
        Object lock2 = new Object();
        waitThread waitThread = new waitThread(lock1);
        waitThread.start();
        notifyThread notifyThread = new notifyThread(lock2);
        notifyThread.start();
        System.out.println("end test");
    }
}


/**
 * 执行结果：
 *  begin test
 *  Thread-0 wait start 1600937916517
 *  end test
 *  Thread-1 notify start 1600937917520
 *  Thread-1 notify end 1600937917520
 *  ...
 */
```
