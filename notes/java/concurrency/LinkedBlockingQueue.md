# LinkedBlockingQueue

## 特性功能

基于**链表**构建的阻塞队列，队列的元素存储在链表中。它在创建时可以不指定容量大小，因此它的 **容量是无限** 的（并不是无限，最大为`Integer.MAX_VALUE` = 2^31^-1）。它对于元素的操作也是FIFO的。一般它的吞吐量高于ArrayBlockingQueue，不过实际使用上可能性能也没有预期的高[^1]。

## 源码分析

从构造器开始：

```java
public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
}

public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null);
}

public LinkedBlockingQueue(Collection<? extends E> c) {
    this(Integer.MAX_VALUE);
    final ReentrantLock putLock = this.putLock;
    putLock.lock(); // Never contended, but necessary for visibility
    try {
        int n = 0;
        for (E e : c) {
            if (e == null)
                throw new NullPointerException();
            if (n == capacity)
                throw new IllegalStateException("Queue full");
            enqueue(new Node<E>(e));
            ++n;
        }
        count.set(n);
    } finally {
        putLock.unlock();
    }
}

//入队
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}
```

引出几个属性：

```java
/**
 * Linked list node class
 */
static class Node<E> {
    E item;

    Node<E> next;

    Node(E x) { item = x; }
}

/** The capacity bound, or Integer.MAX_VALUE if none */
private final int capacity;

/** Current number of elements */
private final AtomicInteger count = new AtomicInteger();

/**
 * 队尾
 */
transient Node<E> head;

/**
 * 队头
 */
private transient Node<E> last;

/** 出队操作锁 */
private final ReentrantLock takeLock = new ReentrantLock();

/** 出队等待条件 */
private final Condition notEmpty = takeLock.newCondition();

/** 入队操作锁 */
private final ReentrantLock putLock = new ReentrantLock();

/** 入队等待条件 */
private final Condition notFull = putLock.newCondition();
```

目前看来，`LinkedBlockingQueue`与`ArrayBlockingQueue`的差别除了队列元素存储形式外，`ArrayBlockingQueue`中读写共用一把锁，而`LinkedBlockingQueue`是分离的。

再看具体的插入操作：

```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    if (e == null) throw new NullPointerException();
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;//入队锁
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity) {//队列满，要阻塞入队操作
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(new Node<E>(e));//入队操作
        c = count.getAndIncrement();//计数器加一，c是旧值。
        if (c + 1 < capacity)//队列没满，唤醒阻塞的入队操作线程
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)//队列不空，唤醒阻塞的出队操作。这里先记下来。
        signalNotEmpty();
    return true;
}
   
//入队
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}

/**
 * 唤醒一个出队操作，仅由自put/offer方法调用（通常他们不会持有takeLock锁）
 */
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}

```

再开取出操作：

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout);
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;//出队锁
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {//队列空，阻塞出队操作
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        x = dequeue();//出队
        c = count.getAndDecrement();//计数器减一，c是旧值
        if (c > 1)//还有元素，唤醒其他出队操作线程
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)//队列有空位，唤醒入队操作。这里也记下来。
        signalNotFull();
    return x;
}

//出队操作
private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    Node<E> h = head;
    Node<E> first = h.next;
    h.next = h; // help GC
    head = first;
    E x = first.item;
    first.item = null;
    return x;
}

/**
 * 唤醒阻塞的入队操作. 仅仅由 take/poll方法调用.
 */
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```

这里需要关注的地方：

- `LinkedBlockingQueue`入队和出队由两把锁控制，因为对于链表的实现而言，入队仅操作队尾，出队仅操作队头，不存在数据共享的问题。而`ArrayBlockingQueue`由于采用数组实现，读写会出现操作同一个数组位置的情况，因此ABQ不能“读写分离“。所以说`LBQ`性能更好。

- `AtomicInteger.getAndIncrement()`和`AtomicInteger.getAndDecrement()`两个方法功能是对`AtomicInteger`加一再返回原值。搞清这个上面的两个方法逻辑就能搞通。

  

## 使用案例

略





## 引用与参考

[^1]: https://docs.oracle.com/javase/8/docs/api/index.html