> 不管是Executors提供的几种线程池，还是Spring提供的线程池，你会发现阻塞队列用的都是LinkedBlockingQueue，而不是用的ArrayBlockingQueue

#### LinkedBlockingQueue

使用单链表实现，提供3种构造函数

1. LinkedBlockingQueue() 无参构造函数，链表长度为Integer.MAX_VALUE
2. LinkedBlockingQueue(int capacity) 指定capacity长度
3. LinkedBlockingQueue(Collection<? extends E> c)  不指定长度，即默认长度为Integer.MAX_VALUE，提供初始化元素

链表节点由Node对象组成，每个Node有item变量用于存储元素，next变量指向下一个节点

执行put的时候，将元素放到链表尾部节点；take的时候从头部取元素

两种操作分别有一个锁putLock, takeLock,互不影响,可以同时进行

```java
/** Lock held by take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();

/** Lock held by take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();

//put
 public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
     //...
 }

//take
public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
    //...
}
```

#### ArrayBlockingQueue

使用数组实现，3种构造函数

1. ArrayBlockingQueue(int capacity) 指定长度
2. ArrayBlockingQueue(int capacity, boolean fair) 指定长度，及指定是否使用FIFO顺序进出队列
3. ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c) 指定长度，进行队列顺序，初始元素

从构造函数看出，ArrayBlockingQueue必须指定初始化长度，如果线程池使用该队列，指定长度大了浪费内存，长度小队列并发性不高，在数组满的时候，put操作只能阻塞等待，或者返回false

ArrayBlockingQueue 只定义了一个Lock，put和take使用同一锁，不能同时进行

```java
/** Main lock guarding all access */
    final ReentrantLock lock;
```



#### 总结

1. LinkedBlockingQueue 无须指定长度，放入和取出元素使用不同的锁，互不影响，效率高，通用性强
2. ArrayBlockingQueue 必须指定长度，大了浪费内存，小了性能不高，使用同一把锁，效率低



