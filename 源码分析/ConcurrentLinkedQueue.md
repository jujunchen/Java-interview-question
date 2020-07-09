# ConcurrentLinkedQueue 源码分析



无界非阻塞队列，底层使用单向链表实现，对于出队和入队使用CAS来实现线程安全。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ggkv4ab1rgj30em0me75m.jpg" alt="111" style="zoom: 67%;" />

看一下ConcurrentLinkedQueue的UML类图结构

AbstractQueue 实现了Queue接口的基本方法

Node 静态内部类，表示队列的节点

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

## Node

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
            //设置原node节点的next变量
            //lazySetNext使用了UNSAFE.putOrderedObject能够防止重排序
            t.lazySetNext(newNode);
            //将t指向最后一个node 节点
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

![image-20200709233242307](https://tva1.sinaimg.cn/large/007S8ZIlly1ggl5545wyyj30hw05mt95.jpg)

**根据给定的集合顺序创建队列对象，给定集合不是空集合**

