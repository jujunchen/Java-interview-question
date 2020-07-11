# ConcurrentLinkedQueue 源码分析



无界非阻塞队列，底层使用单向链表实现，对于出队和入队使用CAS来实现线程安全。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ggkv4ab1rgj30em0me75m.jpg" alt="111" style="zoom: 67%;" />

看一下ConcurrentLinkedQueue的UML类图结构

AbstractQueue 实现了Queue接口的基本方法

Node 静态内部类，表示队列的节点

## 内部类

### Node

队列节点对象，ConcurrentLinkedQueue的静态内部类

```java
private static class Node<E> {
    //节点值
    volatile E item;
    //指向下一个节点
    volatile Node<E> next;

    /**
     * node构造函数，将item值写入内存，宽松写入，因为只有在casNext成功后才可见
     */
    Node(E item) {
        UNSAFE.putObject(this, itemOffset, item);
    }

    //cas更新item值
    boolean casItem(E cmp, E val) {
        return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
    }

    //防重排序更新next值
    void lazySetNext(Node<E> val) {
        UNSAFE.putOrderedObject(this, nextOffset, val);
    }

    //cas更新next值
    boolean casNext(Node<E> cmp, Node<E> val) {
        return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
    }

    private static final sun.misc.Unsafe UNSAFE;
    //item属性的偏移量
    private static final long itemOffset
    //next属性的偏移量
    private static final long nextOffset;

    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = Node.class;
            itemOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("item"));
            nextOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("next"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

### CLQSpliterator

可拆分迭代器

```java
static final class CLQSpliterator<E> implements Spliterator<E> {
    static final int MAX_BATCH = 1 << 25;  //最大批处理大小
    final ConcurrentLinkedQueue<E> queue; 
    Node<E> current;    //当前节点，初始化之前都为null
    int batch;          //分割批次的大小
    boolean exhausted;  //如果没有元素了，为true 
    CLQSpliterator(ConcurrentLinkedQueue<E> queue) {
        this.queue = queue;
    }

    //分割队列
    public Spliterator<E> trySplit() {
        Node<E> p;
        final ConcurrentLinkedQueue<E> q = this.queue;
        int b = batch;
        //n表示数组长度
        //分割批次长度<=0，数组长度为1；
        //分割批次长度>=最大批处理大小，数组长度为最大批处理大小；否则为分割批次长度+1
        int n = (b <= 0) ? 1 : (b >= MAX_BATCH) ? MAX_BATCH : b + 1;
        //!exhausted 为true表示队列还有元素
        //((p = current) != null || (p = q.first()) != null) 如果前部分为true表示已经运行过一次，后部分为true表		  //示是第一次运行，将队列的头节点赋值给p
        //p.next != null 表示队列后面还有元素,p表示的节点不是最后一个
        if (!exhausted &&
            ((p = current) != null || (p = q.first()) != null) &&
            p.next != null) {
            //临时存储需分割的元素
            Object[] a = new Object[n];
            //已分割的元素计数，只计数item不为null的元素
            //对于ConcurrentLinkedQueue来说，这里不会有item值为null的情况
            int i = 0;
            do {
                //节点item值赋值临时数组
                if ((a[i] = p.item) != null)
                    //计数
                    ++i;
                //当next也指向本身p的时候为true,也就是自引用，重新找头节点
                if (p == (p = p.next))
                    p = q.first();
            } while (p != null && i < n);  //直到取完需要的元素数量
            //如果p等于null了就说明没有元素了
            if ((current = p) == null)
                exhausted = true;
            //有不为null的元素
            if (i > 0) {
                batch = i;
                //分割数组从0-i不包括i，指定了迭代器的特性
                return Spliterators.spliterator
                    (a, 0, i, Spliterator.ORDERED | Spliterator.NONNULL |
                     Spliterator.CONCURRENT);
            }
        }
        return null;
    }

    //循环每个节点并执行指定函数
    public void forEachRemaining(Consumer<? super E> action) {
        Node<E> p;
        if (action == null) throw new NullPointerException();
        final ConcurrentLinkedQueue<E> q = this.queue;
        if (!exhausted &&
            ((p = current) != null || (p = q.first()) != null)) {
            exhausted = true;
            do {
                E e = p.item;
                if (p == (p = p.next))
                    p = q.first();
                if (e != null)
                    action.accept(e);
            } while (p != null); //循环每个节点，直到没有节点
        }
    }

    //获取一个有效的节点，执行指定函数
    public boolean tryAdvance(Consumer<? super E> action) {
        Node<E> p;
        if (action == null) throw new NullPointerException();
        final ConcurrentLinkedQueue<E> q = this.queue;
        if (!exhausted &&
            ((p = current) != null || (p = q.first()) != null)) {
            E e;
            //通过循环的方式直到节点item值不为null
            //item值为null时通过p.next继续寻找
            do {
                e = p.item;
                //同样如果p是自引用就重新寻找头节点
                if (p == (p = p.next))
                    p = q.first();
            } while (e == null && p != null); 
            if ((current = p) == null)
                exhausted = true;
            if (e != null) {
                action.accept(e);
                return true;
            }
        }
        return false;
    }

    //返回估计的元素数量，因为ConcurrentLinkedQueue是无限队列，返回MAX_VALUE
    public long estimateSize() { return Long.MAX_VALUE; }

    //返回特征值
    public int characteristics() {
        return Spliterator.ORDERED | Spliterator.NONNULL |
            Spliterator.CONCURRENT;
    }
}
```

### Itr

迭代器，从头节点到尾节点进行迭代，迭代器是弱一致性的，只是某一时刻的快照，因为同时有可能其他线程还在对队列进行修改

```java
private class Itr implements Iterator<E> {
    /**
     * 返回下个项目的节点
     */
    private Node<E> nextNode;

    /**
     * 保存下一个节点的item字段
     */
    private E nextItem;

    /**
     * 最后一个项目节点，支持删除
     */
    private Node<E> lastRet;

    Itr() {
        advance();
    }

    /**
     * 移动到下一个有效节点，通过next()返回item或者null
     */
    private E advance() {
        lastRet = nextNode;
        E x = nextItem;

        Node<E> pred, p;
        //nextNode为null是说明是第一次运行
        if (nextNode == null) {
            //获取头节点
            p = first();
            pred = null;
        } else {
            //将上次获取的节点保存为此次的上一个节点
            pred = nextNode;
            //获取下一个节点
            p = succ(nextNode);
        }

        for (;;) {
            //p == null 的情况，可能是哨兵节点时，p=first()返回null
            //也可能是 p=succ(nextNode)到最后一个节点时，返回null
            if (p == null) {
                nextNode = null;
                nextItem = null;
                return x;
            }
            E item = p.item;
            if (item != null) {
                nextNode = p;
                nextItem = item;
                return x;
            } else {
                // 如果item=null时，即节点已被其他线程删除，那么继续获取下一个节点
                Node<E> next = succ(p);
                if (pred != null && next != null)
                    //跳过null节点，将上一个节点的next指向新的节点
                    pred.casNext(p, next);
                	//将p赋值新节点，继续循环
                	p = next;
            }
        }
    }

    //通过下个项目节点不为null来判断是否还有下一个
    public boolean hasNext() {
        return nextNode != null;
    }

    //下一个项目
    public E next() {
        if (nextNode == null) throw new NoSuchElementException();
        return advance();
    }

    public void remove() {
        Node<E> l = lastRet;
        if (l == null) throw new IllegalStateException();
        //将最后一个项目节点设置为null，在下一次执行advance时，该节点将被重新连接
        l.item = null;
        lastRet = null;
    }
}
```

## 属性

先看下ConcurrentLinkedQueue有哪些基本属性

```java
/**
 * 头节点(未删除的)，到达时间复杂度O(1)
 * 不变性:
 * - 所有有效节点能够从头节点通过succ()方法到达
 * - head != null
 * - (tmp = head).next != tmp || tmp != head，就是头节点不会自引用
 * 可变性:
 * - head.item 可能是null也可能不是null
 * - 允许tail滞后于head,也就是不能从头节点通过succ()到达
 *	
 */
private transient volatile Node<E> head;

/**
 * 尾节点，唯一一个node.next==null的节点，到达时间复杂度O(1)
 * 不变性:
 * - tail 节点通过succ()方法一定到达队列中的最后一个节点
 * - tail != null
 * 可变性:
 * - tail.item 可能是null也可能不是null.
 * - 允许tail滞后于head,也就是不能从头节点通过succ()到达
 * - tail.next 可以指向自己，也可以不指向自己
 *	
 */
private transient volatile Node<E> tail;
```
Unsafe相关
```java
private static final sun.misc.Unsafe UNSAFE;
//头节点相对于对象的偏移量
private static final long headOffset;
//尾节点相对于对象的偏移量
private static final long tailOffset;
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> k = ConcurrentLinkedQueue.class;
        //使用Unsafe获取head属性相对于ConcurrentLinkedQueue对象的偏移量，可以使用该偏移量获取head属性的值
        headOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("head"));
        tailOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("tail"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```

## 构造函数

```java
/**
 * 创建一个item,next都为null的Node节点
 * head tail都指向该节点
 */
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>(null);
}

/**
 * 根据给定的集合顺序创建队列对象
 *
 * @param c 给定集合
 * @throws 如果给定的集合为null 或者其中任何一个元素为null将抛出NullPointerException
 */
public ConcurrentLinkedQueue(Collection<? extends E> c) {
    //定义头尾节点局部变量，h头节点变量，t尾节点变量
    Node<E> h = null, t = null;
    for (E e : c) {
        //检查集合元素是否是null，否则将抛出NPE
        checkNotNull(e);
        //构造节点对象
        Node<E> newNode = new Node<E>(e);
        //如果t指向null，则h,t都指向第一个创建的node节点
        if (h == null)
            h = t = newNode;
        //否则说明已经存在头节点
        else {
            //①设置原node节点的next变量
            //lazySetNext使用了UNSAFE.putOrderedObject能够防止重排序
            t.lazySetNext(newNode);
            //②将t指向最后一个node 节点
            t = newNode;
        }
    }
    //当传入的集合为空集合时，局部变量h依然是null，将h,t指向item，next都为null的节点
    if (h == null)
        h = t = new Node<E>(null);
    //更新全局头节点变量，尾节点变量
    head = h;
    tail = t;
}
```

**无参构造函数创建对象后的节点图**

![无参构造](https://tva1.sinaimg.cn/large/007S8ZIlly1ggl6cs8yyuj30e607mgm1.jpg)

**根据给定的集合顺序创建队列对象，给定集合不是空集合**

![未命名绘图](https://tva1.sinaimg.cn/large/007S8ZIlly1ggmr3uhg8vj30yi0kb778.jpg)

