# SynchronousQueue原理详解-公平模式

## 一、介绍

SynchronousQueue是一个双栈双队列算法，无空间的队列或栈，任何一个对SynchronousQueue写需要等到一个对SynchronousQueue的读操作，反之亦然。一个读操作需要等待一个写操作，相当于是交换通道，提供者和消费者是需要组队完成工作，缺少一个将会阻塞线程，知道等到配对为止。

SynchronousQueue是一个队列和栈算法实现，在SynchronousQueue中双队列FIFO提供公平模式，而双栈LIFO提供的则是非公平模式。

对于SynchronousQueue来说，他的put方法和take方法都被抽象成统一方法来进行操作，通过抽象出内部类Transferer，来实现不同的操作。

> 注意事项：本文分析主要是针对jdk1.8的版本进行分析，下面的代码中的线程执行顺序可能并不能完全保证顺序性，执行时间比较短，所以暂且认定有序执行。
>
> 约定：图片中以Reference-开头的代表对象的引用地址，通过箭头方式进行引用对象。

Transferer.transfer方法主要介绍如下所示：

```java
abstract static class Transferer<E> {
    /**
     * 执行put和take方法.
     *
     * @param e 非空时,表示这个元素要传递给消费者（提供者-put）;
     *          为空时, 则表示当前操作要请求消费一个数据（消费者-take）。
     *          offered by producer.
     * @param timed 决定是否存在timeout时间。
     * @param nanos 超时时长。
     * @return 如果返回非空, 代表数据已经被消费或者正常提供; 如果为空,
     *         则表示由于超时或中断导致失败。可通过Thread.interrupted来检查是那种。
     */
    abstract E transfer(E e, boolean timed, long nanos);
}
```

接下来看一下SynchronousQueue的字段信息：

```java
/** CPU数量 */
static final int NCPUS = Runtime.getRuntime().availableProcessors();

/**
 * 自旋次数，如果transfer指定了timeout时间，则使用maxTimeSpins,如果CPU数量小于2则自旋次数为0，否则为32
 * 此值为经验值，不随CPU数量增加而变化，这里只是个常量。
 */
static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;

/**
 * 自旋次数，如果没有指定时间设置，则使用maxUntimedSpins。如果NCPUS数量大于等于2则设定为为32*16，否则为0；
 */
static final int maxUntimedSpins = maxTimedSpins * 16;

/**
 * The number of nanoseconds for which it is faster to spin
 * rather than to use timed park. A rough estimate suffices.
 */
static final long spinForTimeoutThreshold = 1000L;
```

- NCPUS：代表CPU的数量
- maxTimedSpins：自旋次数，如果transfer指定了timeout时间，则使用maxTimeSpins,如果CPU数量小于2则自旋次数为0，否则为32，此值为经验值，不随CPU数量增加而变化，这里只是个常量。
- maxUntimedSpins：自旋次数，如果没有指定时间设置，则使用maxUntimedSpins。如果NCPUS数量大于等于2则设定为为32*16，否则为0；
- spinForTimeoutThreshold：为了防止自定义的时间限过长，而设置的，如果设置的时间限长于这个值则取这个spinForTimeoutThreshold 为时间限。这是为了优化而考虑的。这个的单位为纳秒。

## 公平模式-TransferQueue

TransferQueue内部是如何进行工作的，这里先大致讲解下，队列采用了互补模式进行等待，QNode中有一个字段是isData，如果模式相同或空队列时进行等待操作，互补的情况下就进行消费操作。

入队操作相同模式
![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511194700294-1968231289.png)

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511194800943-86356229.png)

不同模式时进行出队列操作：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511194811557-2047294276.png)

这时候来了一个isData=false的互补模式，队列就会变成如下状态：
![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511194824692-1695599974.png)

TransferQueue继承自Transferer抽象类，并且实现了transfer方法，它主要包含以下内容:

### QNode

代表队列中的节点元素，它内部包含以下字段信息：

1. 字段信息描述

| 字段   | 描述           | 类型    |
| ------ | -------------- | ------- |
| next   | 下一个节点     | QNode   |
| item   | 元素信息       | Object  |
| waiter | 当前等待的线程 | Thread  |
| isData | 是否是数据     | boolean |

1. 方法信息描述

| 方法        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| casNext     | 替换当前节点的next节点                                       |
| casItem     | 替换当前节点的item数据                                       |
| tryCancel   | 取消当前操作，将当前item赋值为this(当前QNode节点)            |
| isCancelled | 如果item是this(当前QNode节点)的话就返回true，反之返回false   |
| isOffList   | 如果已知此节点离队列，判断next节点是不是为this，则返回true，因为由于* advanceHead操作而忘记了其下一个指针。 |

```java
E transfer(E e, boolean timed, long nanos) {
    /* Basic algorithm is to loop trying to take either of
     * two actions:
     *
     * 1. If queue apparently empty or holding same-mode nodes,
     *    try to add node to queue of waiters, wait to be
     *    fulfilled (or cancelled) and return matching item.
     *
     * 2. If queue apparently contains waiting items, and this
     *    call is of complementary mode, try to fulfill by CAS'ing
     *    item field of waiting node and dequeuing it, and then
     *    returning matching item.
     *
     * In each case, along the way, check for and try to help
     * advance head and tail on behalf of other stalled/slow
     * threads.
     *
     * The loop starts off with a null check guarding against
     * seeing uninitialized head or tail values. This never
     * happens in current SynchronousQueue, but could if
     * callers held non-volatile/final ref to the
     * transferer. The check is here anyway because it places
     * null checks at top of loop, which is usually faster
     * than having them implicitly interspersed.
     */

    QNode s = null; // constructed/reused as needed
    // 分为两种状态1.有数据=true 2.无数据=false
    boolean isData = (e != null);
    // 循环内容
    for (;;) {
        // 尾部节点。
        QNode t = tail;
        // 头部节点。
        QNode h = head;
        // 判断头部和尾部如果有一个为null则自旋转。
        if (t == null || h == null)         // 还未进行初始化的值。
            continue;                       // 自旋
        // 头结点和尾节点相同或者尾节点的模式和当前节点模式相同。
        if (h == t || t.isData == isData) { // 空或同模式。
            // tn为尾节点的下一个节点信息。
            QNode tn = t.next;
            // 这里我认为是阅读不一致，原因是当前线程还没有阻塞的时候其他线程已经修改了尾节点tail会导致当前线程的tail节点不一致。
            if (t != tail)                  // inconsistent read
                continue;
            if (tn != null) {               // lagging tail
                advanceTail(t, tn);
                continue;
            }
            if (timed && nanos <= 0)        // 这里如果指定timed判断时间小于等于0直接返回。
                return null;
            // 判断新增节点是否为null,为null直接构建新节点。
            if (s == null)
                s = new QNode(e, isData);
            if (!t.casNext(null, s))        // 如果next节点不为null说明已经有其他线程进行tail操作
                continue;
            // 将t节点替换为s节点
            advanceTail(t, s);
            // 等待有消费者消费线程。
            Object x = awaitFulfill(s, e, timed, nanos);
            // 如果返回的x，指的是s.item,如果s.item指向自己的话清除操作。
            if (x == s) {
                clean(t, s);
                return null;
            }
            // 如果没有取消联系
            if (!s.isOffList()) {
                // 将当前节点替换头结点
                advanceHead(t, s);          // unlink if head
                if (x != null)              // 取消item值，这里是take方法时会进行item赋值为this
                    s.item = s;
                // 将等待线程设置为null
                s.waiter = null;
            }
            return (x != null) ? (E)x : e;

        } else {                            // complementary-mode
            // 获取头结点下一个节点
            QNode m = h.next;               // node to fulfill
            // 如果当前线程尾节点和全局尾节点不一致,重新开始
            // 头结点的next节点为空，代表无下一个节点，则重新开始，
            // 当前线程头结点和全局头结点不相等，则重新开始
            if (t != tail || m == null || h != head)
                continue;                   // inconsistent read

            Object x = m.item;
            if (isData == (x != null) ||    // m already fulfilled
                x == m ||                   // m cancelled
                !m.casItem(x, e)) {         // lost CAS
                advanceHead(h, m);          // dequeue and retry
                continue;
            }

            advanceHead(h, m);              // successfully fulfilled
            LockSupport.unpark(m.waiter);
            return (x != null) ? (E)x : e;
        }
    }
}
```

我们来看一下awaitFulfill方法内容：

```java
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
    // 如果指定了timed则为System.nanoTime() + nanos，反之为0。
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    // 获取当前线程。
    Thread w = Thread.currentThread();
    // 如果头节点下一个节点是当前s节点(以防止其他线程已经修改了head节点)
    // 则运算(timed ? maxTimedSpins : maxUntimedSpins)，否则直接返回。
    // 指定了timed则使用maxTimedSpins，反之使用maxUntimedSpins
    int spins = ((head.next == s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    // 自旋
    for (;;) {
        // 判断是否已经被中断。
        if (w.isInterrupted())
            //尝试取消，将当前节点的item修改为当前节点(this)。
            s.tryCancel(e);
        // 获取当前节点内容。
        Object x = s.item;
        // 判断当前值和节点值不相同是返回，因为弹出时会将item值赋值为null。
        if (x != e)
            return x;
        if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                s.tryCancel(e);
                continue;
            }
        }
        if (spins > 0)![](https://img2018.cnblogs.com/blog/458325/201905/458325-20190511194850882-1013581623.png)

            --spins;
        else if (s.waiter == null)
            s.waiter = w;
        else if (!timed)
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}
```

1. 首先先判断有没有被中断，如果被中断则取消本次操作，将当前节点的item内容赋值为当前节点。
2. 判断当前节点和节点值不相同是返回
3. 将当前线程赋值给当前节点
4. 自旋，如果指定了timed则使用`LockSupport.parkNanos(this, nanos);`，如果没有指定则使用`LockSupport.park(this);`。
5. 中断相应是在下次才能被执行。

通过上面源码分析我们这里做出简单的示例代码演示一下put操作和take操作是如何进行运作的，首先看一下示例代码，如下所示：

```java
/**
 * SynchronousQueue进行put和take操作。
 *
 * @author battleheart
 */
public class SynchronousQueueDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        SynchronousQueue<Integer> queue = new SynchronousQueue<>(true);
        Thread thread1 = new Thread(() -> {
            try {
                queue.put(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread1.start();
        Thread.sleep(2000);
        Thread thread2 = new Thread(() -> {
            try {
                queue.put(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread2.start();
        Thread.sleep(10000);
        Thread thread3 = new Thread(() -> {
            try {
                System.out.println(queue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread3.start();
    }
}
```

首先上来之后进行的是两次put操作，然后再take操作，默认队列上来会进行初始化，初始化的内容如下代码所示:

```java
TransferQueue() {
    QNode h = new QNode(null, false); // initialize to dummy node.
    head = h;
    tail = h;
}
```

初始化后队列的状态如下图所示：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195104140-1116474850.png)

当线程1执行put操作时，来分析下代码：

```java
QNode t = tail;
QNode h = head;
if (t == null || h == null)         // saw uninitialized value
    continue;
```

首先执行局部变量t代表队尾指针，h代表队头指针，判断队头和队尾不为空则进行下面的操作，接下来是if…else语句这里是分水岭，当相同模式操作的时候执行if语句，当进行不同模式操作时执行的是else语句，程序是如何控制这样的操作的呢？接下来我们慢慢分析一下：

```java
if (h == t || t.isData == isData) { // 队列为空或者模式相同时进行if语句
    QNode tn = t.next;
    if (t != tail)                  // 判断t是否是队尾，不是则重新循环。
        continue;
    if (tn != null) {               // tn是队尾的下个节点，如果tn有内容则将队尾更换为tn，并且重新循环操作。
        advanceTail(t, tn);
        continue;
    }
    if (timed && nanos <= 0)        // 如果指定了timed并且延时时间用尽则直接返回空，这里操作主要是offer操作时，因为队列无存储空间的当offer时不允许插入。
        return null;
    if (s == null)                  // 这里是新节点生成。
        s = new QNode(e, isData);
    if (!t.casNext(null, s))        // 将尾节点的next节点修改为当前节点。
        continue;

    advanceTail(t, s);              // 队尾移动
    Object x = awaitFulfill(s, e, timed, nanos);	//自旋并且设置线程。
    if (x == s) {                   // wait was cancelled
        clean(t, s);
        return null;
    }

    if (!s.isOffList()) {           // not already unlinked
        advanceHead(t, s);          // unlink if head
        if (x != null)              // and forget fields
            s.item = s;
        s.waiter = null;
    }
    return (x != null) ? (E)x : e;

}
```

上面代码是if语句中的内容，进入到if语句中的判断是如果头结点和尾节点相等代表队列为空，并没有元素所有要进行插入队列的操作，或者是队尾的节点的isData标志和当前操作的节点的类型一样时，会进行入队操作，isData标识当前元素是否是数据，如果为true代表是数据，如果为false则代表不是数据，换句话说只有模式相同的时候才会往队列中存放，如果不是模式相同的时候则代表互补模式，就不走if语句了，而是走了else语句，上面代码中做有注释讲解，下面看一下这里：

```java
if (s == null)                  // 这里是新节点生成。
    s = new QNode(e, isData);
if (!t.casNext(null, s))        // 将尾节点的next节点修改为当前节点。
    continue
```

当执行上面代码后，队列的情况如下图所示：(这里视为`插入第一个元素`图，方便下面的引用)

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195115791-1877855953.png)

接下来执行这段代码：

```java
 advanceTail(t, s);              // 队尾移动
```

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195122314-69757535.png)

修改了tail节点后，这时候就需要进行自旋操作，并且设置QNode的waiter等待线程，并且将线程等待，等到唤醒线程进行唤醒操作

```java
 Object x = awaitFulfill(s, e, timed, nanos);	//自旋并且设置线程。
```

方法内部分析局部内容，上面已经全部内容的分析：

```java
if (spins > 0)
    --spins;
else if (s.waiter == null)
    s.waiter = w;
else if (!timed)
    LockSupport.park(this);
else if (nanos > spinForTimeoutThreshold)
    LockSupport.parkNanos(this, nanos);
```

如果自旋时间spins还有则进行循环递减操作，接下来判断如果当前节点的waiter是空则价格当前线程赋值给waiter，上图中显然是为空的所以会把当前线程进行赋值给我waiter，接下来就是等待操作了。

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195128586-1381252368.png)

上面线程则处于等待状态，接下来是线程二进行操作，这里不进行重复进行，插入第二个元素队列的状况，此时线程二也处于等待状态。

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195133984-1507606809.png)

上面的主要是put了两次操作后队列的情况，接下来分析一下take操作时又是如何进行操作的，当take操作时，isData为false，而队尾的isData为true两个不相等，所以不会进入到if语句，而是进入到了else语句

```java
} else {                            // 互补模式
    QNode m = h.next;               // 获取头结点的下一个节点，进行互补操作。
    if (t != tail || m == null || h != head)
        continue;                   // 这里就是为了防止阅读不一致的问题

    Object x = m.item;
    if (isData == (x != null) ||    // 如果x=null说明已经被读取了。
        x == m ||                   // x节点和m节点相等说明被中断操作，被取消操作了。
        !m.casItem(x, e)) {         // 这里是将item值设置为null
        advanceHead(h, m);          // 移动头结点到头结点的下一个节点
        continue;
    }

    advanceHead(h, m);              // successfully fulfilled
    LockSupport.unpark(m.waiter);
    return (x != null) ? (E)x : e;
}
```

首先获取头结点的下一个节点用于互补操作，也就是take操作，接下来进行阅读不一致的判断，防止其他线程进行了阅读操作，接下来获取需要弹出内容x=1，首先进行判断节点内容是不是已经被消费了，节点内容为null时则代表被消费了，接下来判断节点的item值是不是和本身相等如果相等话说明节点被取消了或者被中断了，然后移动头结点到下一个节点上，然后将`refenrence-715`的item值修改为null，`至于为什么修改为null这里留下一个悬念，这里还是比较重要的，大家看到这里的时候需要注意下`，显然这些都不会成立，所以if语句中内容不会被执行，接下来的队列的状态是是这个样子的：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195139214-877626357.png)

OK，接下来就开始移动队头head了，将head移动到m节点上，执行代码如下所示：

```java
advanceHead(h, m);
```

此时队列的状态是这个样子的：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195144711-804621885.png)

```java
LockSupport.unpark(m.waiter);
return (x != null) ? (E)x : e;
```

接下来将执行唤醒被等待的线程，也就是thread-0，然后返回获取item值1，take方法结束，但是这里并没有结束，因为唤醒了put的线程，此时会切换到put方法中，这时候线程唤醒后会执行`awaitFulfill`方法，此时循环时，有与item值修改为null则直接返回内容。

```java
Object x = s.item;
if (x != e)
    return x;
```

这里的代码我们可以对照`插入第一个元素`图，s节点也就是当前m节点，获取值得时候已经修改为null，但是当时插入的值时1，所以两个不想等了，则直接返回null值。

```java
Object x = awaitFulfill(s, e, timed, nanos);
if (x == s) {                   // wait was cancelled
    clean(t, s);
    return null;
}

if (!s.isOffList()) {           // not already unlinked
    advanceHead(t, s);          // unlink if head
    if (x != null)              // and forget fields
        s.item = s;
    s.waiter = null;
}
return (x != null) ? (E)x : e;
```

又返回到了transfer方法的if语句中，此时x和s并不相等所以不用进行clean操作，首先判断s节点是否已经离队了，显然并没有进行离队操作，`advanceHead(t, s);`操作不会被执行因为上面已近将头节点修改了，但是第一次插入的时候头结点还是`reference-716`，此时已经是`reference-715`,而t节点的引用地址是`reference-716`，所以不会操作，接下来就是将waiter设置为null，也就是忘记掉等待的线程。

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195151368-1904763115.png)

分析了正常的take和put操作，接下来分析下中断操作，由于中断相应后，会被执行`if(w.isInterrupted())`这段代码，它会执行`s.tryCancel(e)`方法，这个方法的作用的是将QNode节点的item节点赋值为当前QNode，这时候x和e值就不相等了（`if (x != e)`），x的值是s.item，则为当前QNode，而e的值是用户指定的值，这时候返回x(s.item)。返回到函数调用地方`transfer`中，这时候要执行下面语句：

```java
if (x == s) {
    clean(t, s);
    return null;
}
```

进入到clean方法执行清理当前节点，下面是方法clean代码：

```java
/**
 * Gets rid of cancelled node s with original predecessor pred.
 */
void clean(QNode pred, QNode s) {
    s.waiter = null; // forget thread
    /*
     * At any given time, exactly one node on list cannot be
     * deleted -- the last inserted node. To accommodate this,
     * if we cannot delete s, we save its predecessor as
     * "cleanMe", deleting the previously saved version
     * first. At least one of node s or the node previously
     * saved can always be deleted, so this always terminates.
     */
    while (pred.next == s) { // Return early if already unlinked
        QNode h = head;
        QNode hn = h.next;   // Absorb cancelled first node as head
        if (hn != null && hn.isCancelled()) {
            advanceHead(h, hn);
            continue;
        }
        QNode t = tail;      // Ensure consistent read for tail
        if (t == h)
            return;
        QNode tn = t.next;
        // 判断现在的t是不是末尾节点，可能其他线程插入了内容导致不是最后的节点。
        if (t != tail)
            continue;
        // 如果不是最后节点的话将其现在t.next节点作为tail尾节点。
        if (tn != null) {
            advanceTail(t, tn);
            continue;
        }
        // 如果当前节点不是尾节点进入到这里面。
        if (s != t) {        // If not tail, try to unsplice
            // 获取当前节点（被取消的节点）的下一个节点。
            QNode sn = s.next;
            // 修改上一个节点的next(下一个)元素为下下个节点。
            if (sn == s || pred.casNext(s, sn))
                //返回。
                return;
        }
        QNode dp = cleanMe;
        if (dp != null) {    // 尝试清除上一个标记为清除的节点。
            QNode d = dp.next;	//1.获取要被清除的节点
            QNode dn;
            if (d == null ||               // 被清除节点不为空
                d == dp ||                 // 被清除节点已经离队
                !d.isCancelled() ||        // 被清除节点是标记为Cancel状态的。
                (d != t &&                 // 被清除节点不是尾节点
                 (dn = d.next) != null &&  // 被清除节点下一个节点不为null
                 dn != d &&                //   that is on list
                 dp.casNext(d, dn)))       // 将被清除的节点的前一个节点的下一个节点修改为被清除节点的下一个节点。
                casCleanMe(dp, null);      // 清空cleanMe节点。
            if (dp == pred)
                return;      // s is already saved node
        } else if (casCleanMe(null, pred)) // 这里将上一个节点标记为被清除操作，但是其实要操作的是下一个节点。
            return;          // Postpone cleaning s
    }
}
```

1. 如果节点中取消的头结点的下一个节点，只需要移动当前head节点到下一个节点即可。
2. 如果取消的是中间的节点，则将当前节点next节点修改为下下个节点。
3. 如果修改为末尾的节点，则将当前节点放入到QNode的clearMe中，等待有内容进来之后下一次进行清除操作。

**实例一**：清除头结点下一个节点，下面是实例代码进行讲解：

```java
/**
 * 清除头结点的下一个节点实例代码。
 *
 * @author battleheart
 */
public class SynchronousQueueDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        SynchronousQueue<Integer> queue = new SynchronousQueue<>(true);
        AtomicInteger atomicInteger = new AtomicInteger(0);

        Thread thread1 = new Thread(() -> {
            try {
                queue.put(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread1.start();
        Thread.sleep(200);

        Thread thread2 = new Thread(() -> {
            try {
                queue.put(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread2.start();
      	Thread.sleep(2000);
        thread1.interrupt();

    }
}
```

上面例子说明我们启动了两个线程，分别向SynchronousQueue队列中添加了元素1和元素2，添加成功之后的，让主线程休眠一会，然后将第一个线程进行中断操作，添加两个元素后节点所处在的状态为下图所示：
![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511194913685-639275710.png)

当我们调用`thread1.interrupt`时，此时线程1等待的消费操作将被终止，会相应上面`awaitFulfill`方法，该方法会运行下面代码：

```java
if (w.isInterrupted())
    //尝试取消，将当前节点的item修改为当前节点(this)。
    s.tryCancel(e);
// 获取当前节点内容。
Object x = s.item;
// 判断当前值和节点值不相同是返回，因为弹出时会将item值赋值为null。
if (x != e)
    return x;
```

首先上来现将s节点(上图中的Reference-715引用对象)的item节点设置为当前节点引用(Reference-715引用对象)，所以s节点和e=1不相等则直接返回，此时节点的状态变化如下所示：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511194923006-1133504150.png)

退出`awaitFulfill`并且返回的是s节点内容（实际上返回的就是s节点），接下来返回到调用`awaitFulfill`的方法`transfer`方法中

```java
Object x = awaitFulfill(s, e, timed, nanos);
if (x == s) {                   // 是否是被取消了
    clean(t, s);
    return null;
}
```

首先判断的事x节点和s节点是否相等，上面我们也说了明显是相等的所以这里会进入到clean方法中，`clean(QNode pred, QNode s)`clean方法一个是前节点，一个是当前被取消的节点，也就是当前s节点的前节点是head节点，接下来我们一步一步的分析代码：

```java
s.waiter = null; // 删除等待的线程。
```

进入到方法体之后首先先进行的是将当前节点的等待线程删除，如下图所示：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511194930509-319081924.png)

接下来进入while循环，循环内容时`pred.next == s`如果不是则表示已经移除了节点，反之还在队列中，则进行下面的操作：

```java
QNode h = head;
QNode hn = h.next;   // 如果取消的是第一个节点则进入下面语句
if (hn != null && hn.isCancelled()) {
    advanceHead(h, hn);
    continue;
}
```

可以看到首先h节点为head节点，hn为头结点的下一个节点，在进行判断头结点的下一个节点不为空并且头结点下一个节点是被中断的节点(取消的节点)，则进入到if语句中，if语句其实也很简单就是将头结点修改为头结点的下一个节点(s节点，别取消节点，并且将前节点的next节点修改为自己，也就是移除了之前的节点，我们看下advanceHead方法：

```java
void advanceHead(QNode h, QNode nh) {
    if (h == head &&
        UNSAFE.compareAndSwapObject(this, headOffset, h, nh))
        h.next = h; // forget old next
}
```

首先上来先进行CAS移动头结点，再讲原来头结点h的next节点修改为自己(h)，为什么这样做呢？因为上面进行`advanceHead`之后并没有退出循环，是进行continue操作，也就是它并没有跳出while循环，他还会循环一次prev.next此时已经不能等于s所以退出循环，如下图所示：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511194937772-952186611.png)

**实例二**：清除中间的节点

```java
/**
 * SynchronousQueue实例二，清除中间的节点。
 *
 * @author battleheart
 */
public class SynchronousQueueDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        SynchronousQueue<Integer> queue = new SynchronousQueue<>(true);
        AtomicInteger atomicInteger = new AtomicInteger(0);

        Thread thread1 = new Thread(() -> {
            try {
                queue.put(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread1.start();
        //休眠一会。
        Thread.sleep(200);
        Thread thread2 = new Thread(() -> {
            try {
                queue.put(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread2.start();
        //休眠一会。
        Thread.sleep(200);
        Thread thread3 = new Thread(() -> {
            try {
                queue.put(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread3.start();
        //休眠一会。
        Thread.sleep(10000);
        thread2.interrupt();


    }
}
```

看上面例子，首先先进行put操作三次，也就是入队3条数据，分别是整型值1，整型值2，整型值3，然后将当前线程休眠一下，对中间线程进行中断操作，通过让主线程休眠一会保证线程执行顺序性(当然上面线程不一定能保证执行顺序，因为put操作一下子就执行完了所以这点时间是可以的)，此时队列所处的状态来看一下下图：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511194945160-1193138044.png)

当休眠一会之后，进入到threa2进行中断操作，目前上图中表示`Reference-723`被中断操作，此时也会进入到`awaitFulfill`方法中，将`Reference-723`的item节点修改为当前节点，如下图所示：
![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511194951281-1862925421.png)

进入到clear方法中此时的prev节点为`Reference-715`，s节点是被清除节点，还是首先进入clear方法中先将waiter设置为null，取消当前线程内容，如下图所示：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511194955786-1500616605.png)

接下来进入到循环中，进行下面处理

```java
QNode h = head;
QNode hn = h.next;   // Absorb cancelled first node as head
if (hn != null && hn.isCancelled()) {
    advanceHead(h, hn);
    continue;
}
QNode t = tail;      // Ensure consistent read for tail
if (t == h)
    return;
QNode tn = t.next;
if (t != tail)
    continue;
if (tn != null) {
    advanceTail(t, tn);
    continue;
}
if (s != t) {        // If not tail, try to unsplice
    QNode sn = s.next;
    if (sn == s || pred.casNext(s, sn))
        return;
}
```

第一个if语句已经分析过了所以说这里不会进入到里面去，接下来是进行尾节点t是否是等于head节点如果相等则代表没有元素，在判断当前方法的t尾节点是不是真正的尾节点tail如果不是则进行修改尾节点，先来看一下现在的状态：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195001272-912303463.png)

`tn != null`判断如果tn不是尾节点，则将tn作为尾节点处理，如果处理之后还不是尾节点还会进行处理直到tail是尾节点未知，我们现在这个是尾节点所以跳过这段代码。`s != t`通过上图可以看到s节点是被清除节点，并不是尾节点所以进入到循环中：

```java
if (s != t) {        // If not tail, try to unsplice
    QNode sn = s.next;
    if (sn == s || pred.casNext(s, sn))
        return;
}
```

首先获取的s节点的下一个节点，上图中表示`Reference-725`节点，判断sn是都等于当前节点显然这一条不成立，pred节点为`Reference-715`节点，将715节点的next节点变成`Reference-725`节点，这里就将原来的节点清理出去了，现在的状态如下所示：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195006304-825474069.png)

**实例三**：删除的节点是尾节点

```java
/**
 * SynchronousQueue实例三，删除的节点为尾节点
 *
 * @author battleheart
 */
public class SynchronousQueueDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        SynchronousQueue<Integer> queue = new SynchronousQueue<>(true);
        AtomicInteger atomicInteger = new AtomicInteger(0);

        Thread thread1 = new Thread(() -> {
            try {
                queue.put(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread1.start();

        Thread thread2 = new Thread(() -> {
            try {
                queue.put(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread2.start();

        Thread.sleep(10000);
        thread2.interrupt();

        Thread.sleep(10000);

        Thread thread3 = new Thread(() -> {
            try {
                queue.put(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread3.start();

        Thread.sleep(10000);
        thread3.interrupt();
    }
}
```

该例子主要说明一个问题就是删除的节点如果是末尾节点的话，`clear`方法又是如何处理的，首先启动了三个线程其中主线程休眠了一会，为了能让插入的顺序保持线程1，线程2，线程3这样子，启动第二个线程后，又将第二个线程中断，这是第二个线程插入的节点为尾节点，然后再启动第三个节点插入值，再中断了第三个节点末尾节点，说一下为啥这样操作，因为当清除尾节点时，并不是直接移除当前节点，而是将被清除的节点的前节点设置到QNode的CleanMe中，等待下次clear方法时进行清除上次保存在CleanMe的节点，然后再处理当前被中断节点，将新的被清理的节点prev设置为cleanMe当中，等待下次进行处理，接下来一步一步分析，首先我们先来看一下第二个线程启动后节点的状态。

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195014952-2096559474.png)

此时运行`thread2.interrupt();`将第二个线程中断，这时候会进入到clear方法中，前面的代码都不会被返回，会执行下面的语句：

```java
QNode dp = cleanMe;
if (dp != null) {    // Try unlinking previous cancelled node
    QNode d = dp.next;
    QNode dn;
    if (d == null ||               // d is gone or
        d == dp ||                 // d is off list or
        !d.isCancelled() ||        // d not cancelled or
        (d != t &&                 // d not tail and
         (dn = d.next) != null &&  //   has successor
         dn != d &&                //   that is on list
         dp.casNext(d, dn)))       // d unspliced
        casCleanMe(dp, null);
    if (dp == pred)
        return;      // s is already saved node
} else if (casCleanMe(null, pred))
    return;
```

首先获得TransferQueue当中cleanMe节点，此时获取的为null，当判断dp!=null时就会被跳过，直接执行

`casCleanMe(null, pred)`此时pred传入的值时t节点指向的内容，也就是当前节点的上一个节点，它会被标记为清除操作节点(其实并不清楚它而是清除它下一个节点，也就是说item=this的节点)，此时看一下节点状态为下图所示：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195020658-676114858.png)

接下来第三个线程启动了这时候又往队列中添加了元素3，此时队列的状况如下图所示：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195025439-1111265737.png)

此时thread3也被中断操作了，这时候还是运行上面的代码，但是这次不同的点在于cleanMe已经不是空值，是有内容的，首先获取的是cleanMe的下一个节点（d），然我来把变量标记在图上然后看起来好分析一些，如下图所示：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195029976-1603958797.png)

dp表示d节点的前一个pred节点，dn表示d节点的next节点，主要逻辑在这里：

```java
if (d == null ||               // d is gone or
    d == dp ||                 // d is off list or
    !d.isCancelled() ||        // d not cancelled or
    (d != t &&                 // d not tail and
     (dn = d.next) != null &&  //   has successor
     dn != d &&                //   that is on list
     dp.casNext(d, dn)))       // d unspliced
    casCleanMe(dp, null);
if (dp == pred)
    return;      // s
```

首先判断d节点是不是为null，如果d节点为null代表已经清除掉了，如果cleanMe节点的下一个节点和自己相等，说明需要清除的节点已经离队了，判断下个节点是不是需要被清除的节点，目前看d节点是被清除的节点，然后就将被清除的节点的下一个节点赋值给dn并且判断d节点是不是末尾节点，如果不是末尾节点则进行`dp.casNext`方法，这个地方是关键点，它将被清除节点d的前节点的next节点修改为被清除节点d的后面节点dn，然后调用caseCleanMe将TransferQueue中的cleanMe节点清空，此时节点的内容如下所示：
![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195037209-1474107924.png)

可以看出将上一次标记为清除的节点清除了队列中，清除完了就完事儿？那这次的怎么弄呢?因为现在运行的是thread3的中断程序，所以上面并没有退出，而是再次进入循环，循环之后发现dp为null则会运行`casCleanMe(null, pred)`，此时当前节点s的前一个节点已经被清除队列，但是并不影响后续的清除操作，因为前节点的next节点还在维护中，也是前节点的next指向还是`reference-725`,如下图所示：

![img](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/458325-20190511195042216-2002082895.png)

就此分析完毕如果有不正确的地方请指正。



> 原文：[图解SynchronousQueue原理-公平模式 - BattleHeart - 博客园 (cnblogs.com)](https://www.cnblogs.com/dwlsxj/p/Thread.html)