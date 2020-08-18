# java.util.concurrent

juc是从jdk1.5引入的并发编程工具包，参照[jenkov大神](http://tutorials.jenkov.com/java-util-concurrent/blockingqueue.html)的博客，学习一下。

### 1.BlockingQueue 

BlockingQueue是个接口，表示一个队列，该队列是线程安全的，可以将元素放入其中，也可以从其中取出元素。换句话说，多个线程可以同时从Java BlockingQueue插入和获取元素，而不会产生任何并发性问题。

阻塞队列这个术语来自于这样一个事实:Java BlockingQueue能够阻塞试图插入或从队列中获取元素的线程。例如，如果一个线程试图获取一个元素，但队列中没有剩下任何元素，那么该线程将被阻塞，直到有一个元素需要获取为止。调用线程是否被阻塞取决于您在BlockingQueue上调用的方法。

阻塞队列的常见工作模型：

![bk](http://tutorials.jenkov.com/images/java-concurrency-utils/blocking-queue.png)





jdk为BlockingQueue提供了五种默认实现：

1. ArrayBlockingQueue

   特点：有界队列，数组实现，构造时必须指定容量，支持fair 与 non-fair模式。

2. DelayQueue

   略。。

3. LinkedBlockingQueue

   特点：无界队列，链表实现，也可以指定容量限制。

4. PriorityBlockingQueue

   特点：队列不是FIFO的，可传入一个comparator作为排序的依据。

5. SynchronousQueue

   略。。







***



### 2.BlockingDeque

线程安全的阻塞的双端队列。



### 3.ConcurrentMap

支持并发操作的map，jdk提供了一个实现ConcurrentHashMap

#### 1.ConcurrentHashMap

有点类似于HashTable，但是比前者的并发性能要好得多。ConcurrentHashMap不会在读取时对Map加锁。此外，ConcurrentHashMap在写入时不会锁定整个Map。它在写的时候只会锁定数组的一部分。

另一个区别是，如果在迭代时修改了ConcurrentHashMap，则ConcurrentHashMap不会抛出ConcurrentModificationException，因为迭代器不是设计为供多个线程使用的。





### 4.ConcurrentNavigableMap

有序、且支持并发操作的map，有几个比较常用的方法，可以直接访问map中的一部分。

```java
NavigableMap<K,V> headMap(K toKey, boolean inclusive);

NavigableMap<K,V> subMap(K fromKey, boolean fromInclusive,
                             K toKey,   boolean toInclusive);
                             
NavigableMap<K,V> tailMap(K fromKey, boolean inclusive);
```

#### 1.ConcurrentSkipListMap



### 5.线程通讯

1、CountDownLatch

2、CyclicBarrier



