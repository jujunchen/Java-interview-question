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

## 属性

```java
/**队列大小，最大为Integer.MAX_VALUE */
private final int capacity;

/**队列当前节点数量*/
private final AtomicInteger count = new AtomicInteger();

/**
 * 头部节点
 * Invariant: head.item == null
 */
transient Node<E> head;

/**
 * 尾部节点
 * Invariant: last.next == null
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



