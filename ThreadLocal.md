# ThreadLocal 
## 概念  
>+ ThreadLocal类主要解决的就是，对于共享资源变量让`每个线程绑定属于自己的这个变量的本地副本`，
`可以使用 get()和 set()方法来获取默认值或将其值更改为当前线程所存的副本的值，从而避免了线程安全问题。`  
>+ `一个ThreadLocal类的实例对象只能保证一个共享变量的线程安全问题，需要记录多个共享数据，可以创建多个ThreadLocal对象，或者将这些数据进行封装，将封装后的数据对象存入ThreadLocal对象中`。  
```java
public class Main {

    static class threadLocalDemo implements Runnable{
        // SimpleDateFormat 不是线程安全的，所以每个线程都要有自己独立的副本
        private static final ThreadLocal<SimpleDateFormat> formatter =
                ThreadLocal.withInitial(()->new SimpleDateFormat("yyyMMdd HHmm"));

        @Override
        public void run() {
            System.out.println("Thread Name = "+Thread.currentThread().getName()+
                    " default Formatter = "+formatter.get().toPattern());
            try {
                Thread.sleep(new Random().nextInt(1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // 修改为原始默认的日期格式
            formatter.set(new SimpleDateFormat());

            System.out.println("Thread Name = "+Thread.currentThread().getName()+
                    " alter Formatter = "+formatter.get().toPattern());
        }
    }

    public static void main(String[] args) {
        threadLocalDemo threadLocalDemo = new threadLocalDemo();
        for (int i = 0; i < 5; i++) {
            try {
                Thread thread = new Thread(threadLocalDemo,"Thread"+i);
                Thread.sleep(new Random().nextInt(1000));
                thread.start();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

/**
 * 执行结果：
 *  Thread Name = Thread0 default Formatter = yyyMMdd HHmm
 *  Thread Name = Thread0 alter Formatter = M/d/yy h:mm a
 *  Thread Name = Thread1 default Formatter = yyyMMdd HHmm
 *  Thread Name = Thread2 default Formatter = yyyMMdd HHmm
 *  Thread Name = Thread2 alter Formatter = M/d/yy h:mm a
 *  Thread Name = Thread3 default Formatter = yyyMMdd HHmm
 *  Thread Name = Thread1 alter Formatter = M/d/yy h:mm a
 *  Thread Name = Thread3 alter Formatter = M/d/yy h:mm a
 *  Thread Name = Thread4 default Formatter = yyyMMdd HHmm
 *  Thread Name = Thread4 alter Formatter = M/d/yy h:mm a
 *
 */

```
>+ Thread0修改了默认的日期格式后，其余的线程Thread1等的默认日期格式并未受到影响。  
## 原理
>+ Thread 类中有一个 threadLocals 和 一个 inheritableThreadLocals 变量，它们`都是 ThreadLocalMap 类型的变量`，
`*ThreadLocalMap可以存储以ThreadLocal为 key ，Object 对象为 value 的键值对*`，
`默认情况下这两个变量都是 null，只有当前线程调用 ThreadLocal 类的 set或get方法时才创建它们`，
实际上调用这两个方法的时候，我们调用的是ThreadLocalMap类对应的 get()、set()方法。  
```java
public class Thread implements Runnable {
 ......
//与此线程有关的ThreadLocal值。由ThreadLocal类维护
ThreadLocal.ThreadLocalMap threadLocals = null;

//与此线程有关的InheritableThreadLocal值。由InheritableThreadLocal类维护
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
 ......
}
```

>+ ThreadLocalMap是ThreadLocal的静态内部类  
> ![ThreadLocalMap是ThreadLocal的静态内部类](https://github.com/Snailclimb/JavaGuide/raw/master/docs/java/Multithread/images/ThreadLocal%E5%86%85%E9%83%A8%E7%B1%BB.png)  

>+ ThreadLocal类的set()方法  
>>+ ThrealLocal 类中可以通过Thread.currentThread()获取到当前线程对象后，直接通过getMap(Thread t)可以访问到该线程的ThreadLocalMap对象，
```java
public void set(T value) {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
  return t.threadLocals;
}
```

>+ ThreadLocalMap的Hash算法  
>>+ ThreadLocalMap中hash算法，这里i就是当前key在散列表中对应的数组下标位置。  
>>+ 每当创建一个ThreadLocal对象，这个``ThreadLocal.nextHashCode 这个值就`会增长 0x61c88647 `，
这个值很特殊，`它是斐波那契数也叫 黄金分割数`。hash增量为 这个数字，带来的`好处就是 hash 分布非常均匀`。  
```java
int i = key.threadLocalHashCode & (len-1);

public class ThreadLocal<T> {
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode = new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
    
    static class `ThreadLocalMap` {
        `ThreadLocalMap`(`ThreadLocal`<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);

            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
    }
}
```


>+ ThreadLocalMap的 Hash冲突   
>>+ 虽然ThreadLocalMap中使用了黄金分隔数来作为hash计算因子，大大减少了Hash冲突的概率，但是仍然会存在冲突。  
>>+ 当存入值位置已经存在一个Entry `产生冲突，的就会线性向后查找，一直找到Entry为null的槽位才会停止查找，将当前元素放入此槽`，还存在多种情况讨论。 

>>+ `什么情况下桶才是可以使用的呢？`  
>>>+ k = key 说明是替换操作，可以使用；  
>>>+ 碰到一个过期的桶，执行替换逻辑，占用过期桶；  
>>>+ 查找过程中，碰到桶中Entry=null的情况，直接使用。  
>>+ [ThreadLocalMap的 Hash冲突](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/Multithread/ThreadLocal.md)  

>+ ThreadLocal类的get()方法  
```java
public T get() {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null) {  
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  return setInitialValue();
}
```
## 内存泄漏问题 
>+ ThreadLocalMap 中使用的 `key 为 ThreadLocal 的弱引用,而 value 是强引用`。
所以，如果 ThreadLocal 没有被外部强引用的情况下，在垃圾回收的时候，key 会被清理掉，而 value 不会被清理掉。
这样一来，`ThreadLocalMap 中就会出现 key 为 null 的 Entry`。  
>+ 假如我们不做任何措施的话，`value 永远无法被 GC 回收，这个时候就可能会产生内存泄露`。  
>+ ThreadLocalMap 实现中已经考虑了这种情况，`在调用 set()、get()、remove() 方法的时候，会清理掉 key 为 null 的记录`。  
>+ `使用完 ThreadLocal方法后 最好手动调用remove()方法`。   
```java
static class ThreadLocalMap {
  static class Entry extends WeakReference<ThreadLocal<?>> {
      /** The value associated with this ThreadLocal. */
      Object value;

      Entry(ThreadLocal<?> k, Object v) {
          super(k);
          value = v;
      }
  }
}
```
>>+ `ThreadLocalMap 被static 修饰`  
>>>+ 对于static 修饰的变量在类装载时被加载，卸载时才会被释放，如果大量使用不带static 的对象会`造成内存的浪费`。  
### 什么时候易发生内存泄漏
>+ 在ThreadLocal和线程池一起使用的时候，因为线程池中的线程用完之后只是放回线程池，而不是关闭，那么static化的ThreadLocal就会一直存在，
ThreadLocalMap中的value就会一直存在。所以，`在使用连接池的时候，线程close的同时，ThreaddLocal.remove()`。  
