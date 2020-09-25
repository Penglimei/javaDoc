# synchronized
> synchronized 关键字`解决的是多个线程之间访问资源的同步性问题`，synchronized关键字可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行。在 Java 早期版本中，synchronized 属于 重量级锁，效率低下。`Java 6 之后 Java 官方对从 JVM 层面对 synchronized 较大优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销`,所以`目前的话，不论是各种开源框架还是 JDK 源码都大量使用了 synchronized 关键字`。  
>+ synchronized 关键字最主要的三种使用方式：  
>>+ 修饰实例方法  
>>+ 修饰静态方法  
>>+ 修饰代码块  
## 对象锁
>+ 在 Java 中，每个对象都会有一个 monitor 对象，这个对象其实就是 Java 对象的锁，通常会被称为“内置锁”或“对象锁”。类的对象可以有多个，所以`每个对象有其独立的对象锁，互不干扰`。  

>+ synchronized(this)  
```java
public class Main {
    static class SyncTest implements Runnable{

        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName()+" start test "+System.currentTimeMillis());
            synchronized (this){
                System.out.println(Thread.currentThread().getName()+" synchronized(this) "+System.currentTimeMillis());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println(Thread.currentThread().getName()+" end test "+System.currentTimeMillis());

        }
    }

    public static void main(String[] args) {
        SyncTest syncTest = new SyncTest();
        Thread threadA = new Thread(new SyncTest(),"ThreadA");
        Thread threadB = new Thread(new SyncTest(),"ThreadB");
        Thread threadC = new Thread(syncTest,"ThreadC");
        Thread threadD = new Thread(syncTest,"ThreadD");
        threadA.start();
        threadB.start();
        threadC.start();
        threadD.start();
    }
}

/**
 * 执行结果：
 *  ThreadA start test 1601001373679
 *  ThreadD start test 1601001373679
 *  ThreadC start test 1601001373679
 *  ThreadB start test 1601001373679
 *  ThreadD synchronized(this) 1601001373679
 *  ThreadA synchronized(this) 1601001373679
 *  ThreadB synchronized(this) 1601001373679
 *  ThreadD end test 1601001374683
 *  ThreadB end test 1601001374683
 *  ThreadC synchronized(this) 1601001374683
 *  ThreadA end test 1601001374683
 *  ThreadC end test 1601001375688
 */

```
>>+ ThreadA、ThreadB、ThreadD同时获得且释放同步锁，ThreadC后于ThreadD获取锁释放锁，
>>+ 是因为ThreadA、ThreadB、ThreadD使用的是三把非同一对象实例的对象锁，而ThreadC、ThreadD获取的是同一实例对象syncTest的同步锁所以会阻塞。  

>+ 修饰非静态方法  
## 类锁
>+ 在 Java 中，每个类有一个锁，可以称为“类锁”，类锁实际上是通过对象锁实现的，每个类只有一个 Class 对象，所以每个类只有一个类锁，`同一个类的所有实例对象共用同一个类锁`。

>+ synchronized(类.class)  
```java
public class Main {
    static class SyncTest implements Runnable{

        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName()+" start test "+System.currentTimeMillis());
            synchronized (SyncTest.class){
                System.out.println(Thread.currentThread().getName()+" synchronized(this) "+System.currentTimeMillis());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println(Thread.currentThread().getName()+" end test "+System.currentTimeMillis());

        }
    }

    public static void main(String[] args) {
        SyncTest syncTest = new SyncTest();
        Thread threadA = new Thread(new SyncTest(),"ThreadA");
        Thread threadB = new Thread(new SyncTest(),"ThreadB");
        Thread threadC = new Thread(syncTest,"ThreadC");
        Thread threadD = new Thread(syncTest,"ThreadD");
        threadA.start();
        threadB.start();
        threadC.start();
        threadD.start();
    }
}

/**
 * 执行结果：
 *  ThreadA start test 1601001894531
 *  ThreadD start test 1601001894531
 *  ThreadC start test 1601001894531
 *  ThreadB start test 1601001894531
 *  ThreadA synchronized(this) 1601001894531
 *  ThreadA end test 1601001895536
 *  ThreadB synchronized(this) 1601001895536
 *  ThreadB end test 1601001896539
 *  ThreadC synchronized(this) 1601001896539
 *  ThreadC end test 1601001897543
 *  ThreadD synchronized(this) 1601001897543
 *  ThreadD end test 1601001898547
 */
```
>+ ThreadA、ThreadB、ThreadC、ThreadD使用的都是SyncTest.class的对象实例同步锁，所以会阻塞，只有当一个线程释放锁后被阻塞的线程才能尝试获取同步锁。

>+ 修饰静态方法
