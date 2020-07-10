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

    public boolean tryAdvance(Consumer<? super E> action) {
        Node<E> p;
        if (action == null) throw new NullPointerException();
        final ConcurrentLinkedQueue<E> q = this.queue;
        if (!exhausted &&
            ((p = current) != null || (p = q.first()) != null)) {
            E e;
            do {
                e = p.item;
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

    public long estimateSize() { return Long.MAX_VALUE; }

    public int characteristics() {
        return Spliterator.ORDERED | Spliterator.NONNULL |
            Spliterator.CONCURRENT;
    }
}
```

## 属性

先看下ConcurrentLinkedQueue有哪些基本属性

```java
/**
 * A node from which the first live (non-deleted) node (if any)
 * can be reached in O(1) time.
 * Invariants:
 * - all live nodes are reachable from head via succ()
 * - head != null
 * - (tmp = head).next != tmp || tmp != head
 * Non-invariants:
 * - head.item may or may not be null.
 * - it is permitted for tail to lag behind head, that is, for tail
 *   to not be reachable from head!
 *	头节点
 */
private transient volatile Node<E> head;

/**
 * A node from which the last node on list (that is, the unique
 * node with node.next == null) can be reached in O(1) time.
 * Invariants:
 * - the last node is always reachable from tail via succ()
 * - tail != null
 * Non-invariants:
 * - tail.item may or may not be null.
 * - it is permitted for tail to lag behind head, that is, for tail
 *   to not be reachable from head!
 * - tail.next may or may not be self-pointing to tail.
 *	尾节点
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

![指定集合构造](https://tva1.sinaimg.cn/large/007S8ZIlly1ggl6gmjcj5j30m90ptad9.jpg)

