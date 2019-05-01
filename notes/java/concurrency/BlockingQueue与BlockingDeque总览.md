# BlockingQueue与BlockingDeque总览

## BlockingQueue

### 特性与功能

单向队列，在队列**空**时**阻塞获取**元素操作，在队列**满**时**阻塞存储**元素操作。一般这个接口的实现类是**线程安全的**。批量收集操作`addAll`，`containsAll`，`retainAll`和`removeAll`**不一定**以原子方式执行，除非在实现中另有说明。`BlockingQueue`**不接受null元素**。

### 源码分析

参见java源码，`java.util.concurrent.BlockingQueue`。

### 使用案例

这个接口定义的操作根据等待与否大致分四类：操作失败抛异常；操作失败返回特定值；无限期阻塞；有等待时间的阻塞。且不同的实现类这些操作的表现和使用后果也不尽相同，使用时注意根据业务场景仔细选择。细则如下：

|          |  抛异常   | 返回特定值 |      阻塞      |       阻塞超时       |
| :------- | :-------: | :--------: | :------------: | :------------------: |
| 插入操作 |  add(e)   |  offer(e)  |     put(e)     | offer(e, time, unit) |
| 移除操作 | remove()  |   poll()   |     take()     |   poll(time, unit)   |
| 检查操作 | element() |   peek()   | not applicable |    not applicable    |



## BlockingDeque

### 特性与功能

双向队列，是`BlockingQueue`的拓展，功能和`BlockingQueue`大体一致。所以它的实现类可能兼有`BlockingQueue`的方法。

### 源码分析

参见java源码，`java.util.concurrent.BlockingDeque`。

### 使用案例

操作方法分类大致和`BlockingQueue`一致，不过因为是双向队列，因此分别有针对队头和队尾的操作，细则如下：

队头操作

|          |    抛异常     |  返回特定值   |      阻塞      |         阻塞超时          |
| :------- | :-----------: | :-----------: | :------------: | :-----------------------: |
| 插入操作 |  addFirst(e)  | offerFirst(e) |  putFirst(e)   | offerFirst(e, time, unit) |
| 移除操作 | removeFirst() |  pollFirst()  |  takeFirst()   |   pollFirst(time, unit)   |
| 检查操作 |  getFirst()   |  peekFirst()  | not applicable |      not applicable       |

队尾操作

|          |    抛异常    |  返回特定值  |      阻塞      |         阻塞超时         |
| :------- | :----------: | :----------: | :------------: | :----------------------: |
| 插入操作 |  addLast(e)  | offerLast(e) |   putLast(e)   | offerLast(e, time, unit) |
| 移除操作 | removeLast() |  pollLast()  |   takeLast()   |   pollLast(time, unit)   |
| 检查操作 |  getLast()   |  peekLast()  | not applicable |      not applicable      |

显而易见，

> 单向队列的 **插入** 操作等同于双向队列的 **插入队尾** 操作。
>
> 单向队列的 **取出** 操作等同于双向队列的 **取出队头** 操作。



## 参考与引用

[BlockingQueue](<https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/BlockingQueue.html>)

[BlockingDeque](<https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/BlockingDeque.html>)

