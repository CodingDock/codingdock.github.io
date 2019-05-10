# ArrayBlockingQueue



## 特性功能

基于**数组**构建的阻塞队列，队列的元素存储在数组中。它的大小需在创建时指定，因此它的 **容量是有限** 的。它还提供一种能使插入或删除时阻塞的线程的队列访问按FIFO顺序处理的 “**公平模式**“ 。

 

## 源码分析

从构造器开始

```java
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}

public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);//公平与否就是这个锁是否公平锁
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}

public ArrayBlockingQueue(int capacity, boolean fair,
            Collection<? extends E> c) {
    this(capacity, fair);

    final ReentrantLock lock = this.lock;
    lock.lock(); // 锁仅用于保证可见性，而不是互斥
    try {
        int i = 0;
        try {
            for (E e : c) {
                checkNotNull(e);
                items[i++] = e;
            }
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();
        }
        count = i;
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        lock.unlock();
    }
}

```

引出几个中要的属性：

- `final Object[] items`：用于存储队列元素
- `final ReentrantLock lock`：控制所有访问的锁
- `private final Condition notEmpty`：出队操作阻塞条件
- `private final Condition notFull`：入队操作的阻塞条件

一目了然不多解释。下面详解`offer(E e, long timeout, TimeUnit unit)`方法。

```java
/**
 * 将指定的元素插入此队列的尾部，等待指定的等待时间，以便在队列已满时空间可用。
 *
 */
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    
    checkNotNull(e);//检查是不是null，是null直接抛空指针异常
    long nanos = unit.toNanos(timeout);//转换为纳秒
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();//可响应中断地获取锁
    try {
        while (count == items.length) {//判断是不是队列满了，注意这里是while循环！
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);//队列满了，阻塞当前入队操作线程
        }
        enqueue(e);//入队
        return true;
    } finally {
        lock.unlock();
    }
}

/**
 * Inserts element at current put position, advances, and signals.
 * Call only when holding lock.
 */
private void enqueue(E x) {//这是私有方法，并且只能在持有锁的时候调用。
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)//判断队列是否满了，putIndex到达队尾
        putIndex = 0;//重置到队头
    count++;
    notEmpty.signal();//注意这里，唤起被阻塞的出队操作线程
}
```

方法逻辑都简单，这里有两个要点和一个疑问：

> **`notFull`**上面阻塞的都是 **入队** 操作
>
> **`notEmpty`** 上阻塞的都是 **出队** 操作
>
> 为什么第13行判断语句是 **`while`** 而不是 **`if`** ？



使用while语句是为了保证让恢复运行的线程再进行一次状态检查，因为状态可能被其他线程改变了。举一个不是很符合业务规范的例子：队列满了，此时有一个线程阻塞在入队操作，假设无限期等待。这时，来了另一个出队线程做了出队操作。队列有了空位，那么入队线程被唤醒。此时又来了新的入队线程，两个入队线程争夺锁资源，新线程获得锁，入队成功队列满了。而后老线程也获得了锁，此时，老线程应该再次检查队列状态才能保证不出问题。所以这里要将等待操作放到 `while()` 里，这样保证线程苏醒后必须再次确认执行条件。

还有一个原因是为了避免虚假唤醒。因此唤醒的线程做重复检查是必要的。




再看`poll(long timeout, TimeUnit unit)` 方法：

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();//可响应中断地获取锁
    try {
        while (count == 0) {//判断队列是不是空了。依然是while语句哟
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);//队列空，阻塞出队操作线程
        }
        return dequeue();//出队
    } finally {
        lock.unlock();
    }
}

/**
 * Extracts element at current take position, advances, and signals.
 * Call only when holding lock.
 */
private E dequeue() {//这是私有方法，并且只能在持有锁的时候调用。
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)//判断队列是不是空了，takeIndex到达队尾
        takeIndex = 0;//重置takeIndex到队头
    count--;
    if (itrs != null)//不管他
        itrs.elementDequeued();
    notFull.signal();//队列有了空位，唤起被阻塞的入队操作线程
    return x;
}
```



## 使用案例

`ArrayBlockingQueue` 一般用来实现生产者消费者模式，通过指定队列容量来限制生产者生产速度。大致工作流程如下：多个生产者负责生产`产品` 放入队列中，如果队列满了，生产者阻塞。消费者从队列取出 `产品` 消费，如果队列空了，消费者阻塞。生产者和消费者就这样通过 `队列` 完成 `产品` 的生产与消费。

上代码：

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

/**
 * ArrayBlockingQueue的
 * 生产者消费者 示例
 */
public class ABQDemo {

    public static void main(String[] args) {

        BlockingQueue<String> queue = new ArrayBlockingQueue<>(50);

        new Thread(new Producer(queue),"p_1").start();
        new Thread(new Producer(queue),"p_2").start();
        
        new Thread(new Consumer(queue),"c_1").start();
        new Thread(new Consumer(queue),"c_2").start();
        new Thread(new Consumer(queue),"c_3").start();
        
    }
}

class Producer implements Runnable{

    private final BlockingQueue<String> queue;

    Producer(BlockingQueue<String> queue) {
        this.queue = queue;
    }
    
    @Override
    public void run() {
        int i = 0;
        //每隔一秒，往队列里放一个“产品”
        while(true){
            try {
                Thread.sleep(1000);
                String _p = Thread.currentThread().getName()+"_"+"_产品::"+(++i);
                //最多阻塞两秒
                if(!queue.offer(_p,2, TimeUnit.SECONDS)){
                    System.out.println(Thread.currentThread().getName()+"::超时，产品丢弃，"+_p);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

class Consumer implements Runnable{
    
    private final BlockingQueue<String> queue;

    Consumer(BlockingQueue<String> queue) {
        this.queue = queue;
    }
    
    @Override
    public void run() {
        //每隔一秒，往队列取出一个“产品”
        while(true){
            try {
                Thread.sleep(1000);
                //最多阻塞两秒
                String _p = queue.poll(2, TimeUnit.SECONDS);
                if(null != _p){
                    System.out.println(Thread.currentThread().getName()+"::取出产品：："+_p);
                }else {
                    System.out.println(Thread.currentThread().getName()+"::啥也没拿到");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```



## 引用与参考

- `ArrayBlockingQueue` 的 [JavaDoc](<https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ArrayBlockingQueue.html>)

- 方腾飞等：[《Java并发编程的艺术》](<https://www.amazon.cn/dp/B012NDCEA0>)，机械工业出版社2015年7月第一版，第6章第3节。

- Joshua Bloch：[《Effective Java中文版》](<https://www.amazon.cn/dp/B07MP7K1NW>)，机械工业出版社2019年1月第1版，第11章第81条

- [对条件变量(condition variable)的讨论](<https://blog.csdn.net/nhn_devlab/article/details/6117239>)

- [虚假唤醒（spurious wakeup）](<https://www.jianshu.com/p/0eff666a4875>)





