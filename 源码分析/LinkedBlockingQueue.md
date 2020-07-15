# LinkedBlockingQueue 源码分析

> 源码基于open-jdk 11

有界阻塞队列，使用单向链表实现，通过ReentrantLock实现线程安全，阻塞通过Condition实现，出队和入队各一把锁，不存在互相竞争 

![image-20200714154317899](https://tva1.sinaimg.cn/large/007S8ZIlly1ggqjo9fousj30yg0rkdhb.jpg)

## 内部类

### Node

节点对象

```java
static class Node<E> {
    //值
    E item;

    /**
     * 表示有效的后续节点
     * null 表示最后一个节点
     */
    Node<E> next;

    Node(E x) { item = x; }
}
```

### Itr

迭代器

```java
private class Itr implements Iterator<E> {
    private Node<E> next;           //保存下一个节点
    private E nextItem;             // 保存下一个节点的item值
    private Node<E> lastRet;		//最近返回的项目节点
    private Node<E> ancestor;       //remove()中用于断开连接

    Itr() {
        //获取写锁和取锁，takeLock 和 putLock
        fullyLock();
        try {
            //从头节点开始，如果头节点的下一个节点存在
            //为什么不直接用head呢？因为head最开始是指向一个哨兵节点,item=next=null
            if ((next = head.next) != null)
                //就将下一个节点的item值保存
                nextItem = next.item;
        } finally {
            //释放写锁和取锁
            fullyUnlock();
        }
    }

    //通过next判断是否有下一个节点
    public boolean hasNext() {
        return next != null;
    }

    //获取下一个节点
    public E next() {
        Node<E> p;
        if ((p = next) == null)
            throw new NoSuchElementException();
        //p = next != null，将节点保存在lastRet中
        lastRet = p;
        E x = nextItem;
        //获取写锁和取锁
        fullyLock();
        try {
            E e = null;
            //获取有效的节点，直到节点的item值不为null
            for (p = p.next; p != null && (e = p.item) == null; )
                //获取下一个节点
                p = succ(p);
            next = p;
            nextItem = e;
        } finally {
            //释放锁
            fullyUnlock();
        }
        return x;
    }

    //循环对队列中的元素应用指定操作
    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        Node<E> p;
        if ((p = next) == null) return;
        lastRet = p;
        next = null;
        final int batchSize = 64;
        Object[] es = null;
        int n, len = 1;
        do {
            //获取写锁和读锁
            fullyLock();
            try {
                //这是时候进行初始化，获取数组长度，创建数组
                if (es == null) {
                    p = p.next;
                    for (Node<E> q = p; q != null; q = succ(q))
                        if (q.item != null && ++len == batchSize)
                            break;
                    es = new Object[len];
                    es[0] = nextItem;
                    nextItem = null;
                    n = 1;
                } else
                    n = 0;
                //开始循环赋值
                for (; p != null && n < len; p = succ(p))
                    if ((es[n] = p.item) != null) {
                        lastRet = p;
                        n++;
                    }
            } finally {
                //释放锁
                fullyUnlock();
            }
            //循环执行指定操作
            for (int i = 0; i < n; i++) {
                @SuppressWarnings("unchecked") E e = (E) es[i];
                action.accept(e);
            }
        } while (n > 0 && p != null);
    }

    public void remove() {
        Node<E> p = lastRet;
        if (p == null)
            throw new IllegalStateException();
        lastRet = null;
        fullyLock();
        try {
            if (p.item != null) {
                //第一次的时候从头部开始
                if (ancestor == null)
                    ancestor = head;
                //查找当前节点p的上一个节点,findPred通过循环比较的方式
                ancestor = findPred(p, ancestor);
                //断开连接
                unlink(p, ancestor);
            }
        } finally {
            //释放锁
            fullyUnlock();
        }
    }
}
```

## LBQSpliterator

可拆分迭代器

```java
private final class LBQSpliterator implements Spliterator<E> {
    static final int MAX_BATCH = 1 << 25;  //处理数组最大长度
    Node<E> current;    // 当前节点，初始化前为null
    int batch;          // 拆分处理的大小
    boolean exhausted;  // true表示没有更多的节点
    long est = size();  // 预估大小,size()获取的大小，并不准确

    LBQSpliterator() {}

    public long estimateSize() { return est; }

    //分割队列
    public Spliterator<E> trySplit() {
        Node<E> h;
        //下面的表达式true，表示还有元素
        if (!exhausted &&
            ((h = current) != null || (h = head.next) != null)
            && h.next != null) {
            //获取批量处理的最小值，每次执行trySplit，batch都+1
            int n = batch = Math.min(batch + 1, MAX_BATCH);
            Object[] a = new Object[n];
            int i = 0;
            Node<E> p = current;
            //获取读锁和写锁
            fullyLock();
            try {
                if (p != null || (p = head.next) != null)
                    //循环赋值到数组a，直到n个元素
                    for (; p != null && i < n; p = succ(p))
                        if ((a[i] = p.item) != null)
                            i++;
            } finally {
                fullyUnlock();
            }
            //表示没有元素
            if ((current = p) == null) {
                est = 0L;
                exhausted = true;
            }
            else if ((est -= i) < 0L)
                //迭代器大小-已拆分的长度 < 0，将est赋值为0，防止发生负数
                est = 0L;
            if (i > 0)
                //创建Spliterator，指定迭代器的特性
                return Spliterators.spliterator
                    (a, 0, i, (Spliterator.ORDERED |
                               Spliterator.NONNULL |
                               Spliterator.CONCURRENT));
        }
        return null;
    }

    //获取有效节点，并执行指定函数
    public boolean tryAdvance(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        //有元素才继续执行
        if (!exhausted) {
            E e = null;
            //获取写锁和读锁
            fullyLock();
            try {
                Node<E> p;
                //当前节点p不为null执行，如果为null获取头节点下一个节点，直到获取不为null的节点
                if ((p = current) != null || (p = head.next) != null)
                    do {
                        e = p.item;
                        p = succ(p);
                    } while (e == null && p != null);//获取一个节点值不为null的节点
                //这种情况说明，已经没有可获取的节点了
                if ((current = p) == null)
                    exhausted = true;
            } finally {
                //释放锁
                fullyUnlock();
            }
            if (e != null) {
                //应用函数
                action.accept(e);
                return true;
            }
        }
        return false;
    }

    //循环执行指定函数
    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        if (!exhausted) {
            exhausted = true;
            Node<E> p = current;
            current = null;
            forEachFrom(action, p);
        }
    }

    public int characteristics() {
        return (Spliterator.ORDERED |
                Spliterator.NONNULL |
                Spliterator.CONCURRENT);
    }
}

void forEachFrom(Consumer<? super E> action, Node<E> p) {
        // Extract batches of elements while holding the lock; then
        // run the action on the elements while not
        final int batchSize = 64;       // max number of elements per batch
        Object[] es = null;             // container for batch of elements
        int n, len = 0;
        do {
            fullyLock();
            try {
                if (es == null) {
                    if (p == null) p = head.next;
                    //先获取所需数组的长度，并创建数组
                    for (Node<E> q = p; q != null; q = succ(q))
                        if (q.item != null && ++len == batchSize)
                            break;
                    es = new Object[len];
                }
                //将队列节点的值赋值给数组
                for (n = 0; p != null && n < len; p = succ(p))
                    if ((es[n] = p.item) != null)
                        n++;
            } finally {
                //释放锁
                fullyUnlock();
            }
            //给每个元素执行指定函数
            for (int i = 0; i < n; i++) {
                @SuppressWarnings("unchecked") E e = (E) es[i];
                action.accept(e);
            }
        } while (n > 0 && p != null); //这里有必要用循环吗？
    }
```

## 属性

```java
/**队列大小，最大为Integer.MAX_VALUE */
private final int capacity;

/**队列当前节点数量*/
private final AtomicInteger count = new AtomicInteger();

/**
 * 头部节点
 * 不变性: head.item == null
 */
transient Node<E> head;

/**
 * 尾部节点
 * 不变变性: last.next == null
*/
private transient Node<E> last;

/**take,poll等要获取的锁*/
private final ReentrantLock takeLock = new ReentrantLock();

/**当队列为空时，出队操作的线程会阻塞放入该队列中等待*/
private final Condition notEmpty = takeLock.newCondition();

/**put,offer等要获取的锁*/
private final ReentrantLock putLock = new ReentrantLock();

/**当队列满时，入队操作的线程会放入该队列中等待*/
private final Condition notFull = putLock.newCondition();
```

## 构造函数

```java
//无参构造函数，长度默认为最大长度
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}

//提供容量参数
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    //构造一个哨兵节点
    last = head = new Node<E>(null);
}

//从给定的集合中创建队列
public LinkedBlockingQueue(Collection<? extends E> c) {
    //首先创建一个默认队列
    this(Integer.MAX_VALUE);
    //获取写锁
    final ReentrantLock putLock = this.putLock;
    putLock.lock(); //虽然不会发生竞争，但保证可见性是必要的
    try {
        int n = 0;
        for (E e : c) {
            if (e == null)
                throw new NullPointerException();
            if (n == capacity)
                throw new IllegalStateException("Queue full");
            //入队，将元素加入队列末尾
            enqueue(new Node<E>(e));
            ++n;
        }
        //设置节点数量
        count.set(n);
    } finally {
        //释放写锁
        putLock.unlock();
    }
}
```

## size、remainingCapacity

```java
//获取队列长度
public int size() {
    return count.get();
}

//队列剩余容量
public int remainingCapacity() {
    return capacity - count.get();
}
```

## put

在尾部插入指定元素

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    final int c;
    final Node<E> node = new Node<E>(e);
    //获取写锁对象
    final ReentrantLock putLock = this.putLock;
    //获取队列当前数量
    final AtomicInteger count = this.count;
    //加锁
    putLock.lockInterruptibly();
    try {
        //如果队列数量 已经达到队列的最大容量，则将该线程放入等待队列
        while (count.get() == capacity) {
            notFull.await();
        }
        //队列未满，插入队列
        enqueue(node);
        //队列数量+1，返回原来的队列数量
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            //说明队列未满，唤醒原来阻塞等待插入的线程
            notFull.signal();
    } finally {
        //释放写锁
        putLock.unlock();
    }
    //c == 0 说明第一次插入
    if (c == 0)
        //唤醒原来阻塞等待出队的线程
        signalNotEmpty();
}

//链接到末尾节点
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}

//唤醒原来阻塞等待出队的线程，该方法在put/offer中调用
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

## offer

在队列的尾部插入元素，该方法提供超时版本，非一直阻塞

```java
//插队元素，超时版本
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
	//插入元素不能为null
    if (e == null) throw new NullPointerException();
    //将时间转换为纳秒
    long nanos = unit.toNanos(timeout);
    final int c;
    //写锁
    final ReentrantLock putLock = this.putLock;
    //队列数量
    final AtomicInteger count = this.count;
    //加锁，可以被中断
    putLock.lockInterruptibly();
    try {
        //如果队列满了
        while (count.get() == capacity) {
            if (nanos <= 0L)
                return false;
            //将该线程阻塞等待指定时间
            nanos = notFull.awaitNanos(nanos);
        }
        //入队
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        //表示没满，唤醒原来入队阻塞等待的线程
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        //释放锁
        putLock.unlock();
    }
    if (c == 0)
        //至少有一个元素，队列不为空，唤醒原来出队阻塞的线程
        signalNotEmpty();
    return true;
}

//入队，不带超时
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    //队列满了，不等待，直接返回false
    if (count.get() == capacity)
        return false;
    final int c;
    final Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        //再次检查是否已满
        if (count.get() == capacity)
            return false;
        //入队
        enqueue(node);
        c = count.getAndIncrement();
        //为true说明队列未满，还可以继续插入
        if (c + 1 < capacity)
            //唤醒原来阻塞等待的入队线程
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return true;
}
```

## take

取元素，阻塞版本，直到有元素为止

```java
public E take() throws InterruptedException {
    final E x;
    final int c;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        //没有元素，将线程放入出队列表中
        while (count.get() == 0) {
            notEmpty.await();
        }
        //出队
        x = dequeue();
        //队列数量减一，返回原队列数量
        c = count.getAndDecrement();
        //为true 说明队列还有元素，唤醒之前阻塞的出队线程
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    //为true说明之前队列是满的，现在已经至少有一个空闲位置了
    if (c == capacity)
        //唤醒之前阻塞的入队线程
        signalNotFull();
    return x;
}

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

## poll、peek

pool  弹出元素，不阻塞线程，或者阻塞有超时时间

peek 只获取队列的头部元素，不阻塞线程

```java
//弹出元素的超时版本
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    final E x;
    final int c;
    long nanos = unit.toNanos(timeout);
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    //可以被中断
    takeLock.lockInterruptibly();
    try {
        //为true说明队列是空的
        while (count.get() == 0) {
            if (nanos <= 0L)
                return null;
            //阻塞等待nanos，直到超时
            nanos = notEmpty.awaitNanos(nanos);
        }
        //出队
        x = dequeue();
        //队列数量减一，返回原来的数量
        c = count.getAndDecrement();
        //为true说明队列还有元素
        if (c > 1)
            //唤醒原来阻塞出队的线程
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    //为true，说明现在已经有空闲的位置
    if (c == capacity)
        //唤醒原来阻塞的入队线程
        signalNotFull();
    return x;
}

//弹出元素
public E poll() {
    final AtomicInteger count = this.count;
    //队列为空，直接返回null
    if (count.get() == 0)
        return null;
    final E x;
    final int c;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        //再次检查队列是否为空
        if (count.get() == 0)
            return null;
        //出队
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

```java
//获取头部元素
public E peek() {
    final AtomicInteger count = this.count;
    //队列是否为空
    if (count.get() == 0)
        return null;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        //队列不为空，返回头部元素,head.next，因为还有一个item=null的哨兵节点，比如poll形成的
        return (count.get() > 0) ? head.next.item : null;
    } finally {
        takeLock.unlock();
    }
}
```



```java
//出队，head元素的item始终等于null
private E dequeue() {
    Node<E> h = head; //①
    Node<E> first = h.next; //②
    //变为自引用，等GC回收
    h.next = h; //③  
    head = first; //④
    E x = first.item; //⑤
    first.item = null; //⑥
    return x;
}
```

![出队动画演示](https://tva1.sinaimg.cn/large/007S8ZIlly1ggs2u3xfo6g30hs0b4tbn.gif)

## findPred

查找当前节点的上一个节点

```java
Node<E> findPred(Node<E> p, Node<E> ancestor) {
    //如果ancestor为空，从头节点开始
    if (ancestor.item == null)
        ancestor = head;
    //直到找到与当前节点p相同，返回上一个节点
    for (Node<E> q; (q = ancestor.next) != p; )
        ancestor = q;
    return ancestor;
}
```

## unlink

断开连接

```java
//p为当前节点，pred为上一个节点
void unlink(Node<E> p, Node<E> pred) {
    p.item = null;
    // 将当前节点p的下一个节点赋值给上一个节点pred的next上
    pred.next = p.next;
    //如果当前节点为最后一个节点
    if (last == p)
        //将上一个节点赋值给last，因为当前节点将被删除
        last = pred;
    //count获取数量，并减一
    //判断之前是否是满的，因为删除了一个后，有空闲，唤醒原来在notFull阻塞的线程
    if (count.getAndDecrement() == capacity)
        notFull.signal();
}
```

## 默认线程池阻塞队列为什么用LinkedBlockingQueue

>  不管是Executors提供的几种线程池，还是Spring提供的线程池，你会发现阻塞队列用的都是LinkedBlockingQueue，而不是用的ArrayBlockingQueue。
>
>  源码基于Java8

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



