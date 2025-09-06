# Flow、SubmissionPubliser源码分析

> 基于open jdk 11

Flow 、SubmissionPubliser类是 java9中新增的类，都被放在JUC包中

## Flow

定义了一种生产者和消费者(订阅者)模型的接口，可以用于流式控制中

### Publisher

```java
//流式接口
//定义生产者
@FunctionalInterface
public static interface Publisher<T> {
    /**
     * 增加订阅者
     *
     * @param subscriber 订阅者
     * @throws NullPointerException if subscriber is null
     */
    public void subscribe(Subscriber<? super T> subscriber);
}
```

### Subscriber

```java
//订阅者
public static interface Subscriber<T> {
    /**
     * 在Publisher接收一个新的订阅者时，会调用该方法
     *
     * @param subscription 订阅包，提供了获取下一个元素和取消获取操作的方法
     */
    public void onSubscribe(Subscription subscription);

    /**
     * 获取一个新元素，每次有新元素的时候会被调用
     *
     * @param item the item
     */
    public void onNext(T item);

    /**
     * 发送异常时会被调用
     *
     * @param throwable the exception
     */
    public void onError(Throwable throwable);

    /**
     * 当数据接收完成后调用
     */
    public void onComplete();
}
```

### Subscription

```java
//订阅包
public static interface Subscription {
    /**
     * 发起请求，获取元素
     *
     * @param n 获取的增量，Long.MAX_VALUE表示无限的
     */
    public void request(long n);

    /**
     * 取消消息接收
     */
    public void cancel();
}
```

### Processor

```java
//既可以充当生产者，也可以充当订阅者
public static interface Processor<T,R> extends Subscriber<T>, Publisher<R> {
}

static final int DEFAULT_BUFFER_SIZE = 256;

/**
 * 定义的默认缓存区，256
 *
 * @return the buffer size value
 */
public static int defaultBufferSize() {
    return DEFAULT_BUFFER_SIZE;
}
```

## SubmissionPublisher

FLow 类的一种实现，可以在并发环境下使用

JDK中的说明：

SubmissionPublisher提供了使用Executor的构造函数，如果生产者是在独立线程中运行，并且能估计消费者数量，就使用Executors.newFixedThreadPool(int)固定线程数量的线程池，否则使用默认，ForkJoinPool.commonPool()

SubmissionPublisher提供缓冲功能，能够使生产者和消费者以不同的速率运行，每个消费者独立使用一个缓冲区，缓冲区在首次使用的时候创建，提供了一个默认值256，并会根据需要扩大到最大值，容量通常扩大到最近的2的次幂或者支持的最大值

SubmissionPublisher可以在多个线程之间共享，会在发布项目之前执行操作或者会发出一个happen-before信号给每个对应的消费者

发布方法支持设置在缓冲区满时如何处理的不同策略，submit(Object)方法会阻塞直到有可用资源，这种用法最简单，但速度慢；offer方法虽然会丢弃项目，但提供了插入动作重试的机会

如果任何Subscriber方法抛出异常，在其订阅将被取消

方法consume(Consumer)简化了对常见情况的支持，其中订阅者的唯一操作是使用supplied的函数请求和处理所有项目

此类还可以作为生成元素的子类的基类，并使用此类中的方法发布。例如，下面是一个周期性发布supplier生成元素的类。 （实际上，您可以添加独立启动和停止生成的方法，在发布者之间共享Executor等，或者使用SubmissionPublisher作为组件而不是超类。）

```java
class PeriodicPublisher<T> extends SubmissionPublisher<T> {
   final ScheduledFuture<?> periodicTask;
   final ScheduledExecutorService scheduler;
   PeriodicPublisher(Executor executor, int maxBufferCapacity,
                     Supplier<? extends T> supplier,
                     long period, TimeUnit unit) {
     super(executor, maxBufferCapacity);
     scheduler = new ScheduledThreadPoolExecutor(1);
     periodicTask = scheduler.scheduleAtFixedRate(
       () -> submit(supplier.get()), 0, period, unit);
   }
   public void close() {
     periodicTask.cancel(false);
     scheduler.shutdown();
     super.close();
   }
 }
```

以下是Flow.Processor实现的示例，为了简化说明，使用单步向publisher发起请求，更合适的版本可以使用submit方法或者其他实用方法来监控流量

```java
class TransformProcessor<S,T> extends SubmissionPublisher<T>
    implements Flow.Processor<S,T> {
    final Function<? super S, ? extends T> function;
    Flow.Subscription subscription;
    TransformProcessor(Executor executor, int maxBufferCapacity,
                       Function<? super S, ? extends T> function) {
        super(executor, maxBufferCapacity);
        this.function = function;
    }
    public void onSubscribe(Flow.Subscription subscription) {
        (this.subscription = subscription).request(1);
    }
    public void onNext(S item) {
        subscription.request(1);
        submit(function.apply(item));
    }
    public void onError(Throwable ex) { closeExceptionally(ex); }
    public void onComplete() { close(); }
}
```

### 使用示例

```java
@Test
public void test1() throws Exception {
    SubmissionPublisher<Integer> submissionPublisher = new SubmissionPublisher<>();
    submissionPublisher.consume(System.out::println);
    submissionPublisher.submit(1);
    submissionPublisher.submit(2);
    submissionPublisher.offer(3, (x, y) -> {
        System.out.println("xxx");
        return false;
    });
}
```

> 使用二进制来表示状态码可以参考：https://blog.csdn.net/qq_36631014/article/details/79675924

### 基本属性

```java
//缓存区最大值，1073741824
static final int BUFFER_CAPACITY_LIMIT = 1 << 30;
//初始的最大缓存区，缓存区大小必须是2的幂次
static final int INITIAL_CAPACITY = 32;
/**
* BufferedSubscription通过next字段维护了一个链表，这种结构对循环发布非常有用
* 需要遍历O(n)次来检查重复的消费者，但是预计消费比发布少的多
* 取消订阅只发生在遍历循环期间，如果BufferedSubscription方法返回负值表示他们已经关闭
* 为了减少head-of-line阻塞，submit和offer方法首先调用BufferedSubscription的offer方法
* 在满的时候提供了重试
*/
BufferedSubscription<T> clients;
/** 运行状态，只在锁内更新*/
volatile boolean closed;
/** 在第一次调用subscribe的时候，设置为true，初始化消费的所有者线程*/
boolean subscribed;
/** 第一次调用subscribe的线程，如果线程曾经改变过，则为null */
Thread owner;
/** 如果不为null，说明在closeExceptionally中生成了异常 */
volatile Throwable closedException;

// 用于构造BufferedSubscriptions的参数
final Executor executor;
//onNext中发生异常时的处理函数
final BiConsumer<? super Subscriber<? super T>, ? super Throwable> onNextHandler;
//表示缓冲区最大容量
final int maxBufferCapacity;
```

### 构造函数

```java
//默认构造函数，默认使用ForkJoinPool的公共线程池异步运行subscriber，除非并发级别不支持，则使用普通的线程池
//ThreadPerTaskExecutor
//subscriber的最大缓冲区为256
//Subscriber的异常处理器为null
public SubmissionPublisher() {
    this(ASYNC_POOL, Flow.defaultBufferSize(), null);
}
//使用指定的线程池，指定的缓冲区容量创建
public SubmissionPublisher(Executor executor, int maxBufferCapacity) {
    this(executor, maxBufferCapacity, null);
}

public SubmissionPublisher(Executor executor, int maxBufferCapacity,
                           BiConsumer<? super Subscriber<? super T>, ? super Throwable> handler) {
    if (executor == null)
        throw new NullPointerException();
    if (maxBufferCapacity <= 0)
        throw new IllegalArgumentException("capacity must be positive");
    this.executor = executor;
    this.onNextHandler = handler;
    //最大容量为离2的幂次最近的值，跟hashMap的计算最大容量算法一样
    this.maxBufferCapacity = roundCapacity(maxBufferCapacity);
}
```

### 内部类

#### ConsumerSubscriber

```java
/** 订阅者,供内部使用*/
static final class ConsumerSubscriber<T> implements Subscriber<T> {
    final CompletableFuture<Void> status;
    final Consumer<? super T> consumer;
    Subscription subscription;
    ConsumerSubscriber(CompletableFuture<Void> status,
                       Consumer<? super T> consumer) {
        this.status = status; this.consumer = consumer;
    }
    public final void onSubscribe(Subscription subscription) {
        this.subscription = subscription;
        //如果元素已经消费完成，那么取消订阅
        status.whenComplete((v, e) -> subscription.cancel());
        //没有完成继续请求
        if (!status.isDone())
            subscription.request(Long.MAX_VALUE);
    }
    //发送错误时候的处理
    public final void onError(Throwable ex) {
        status.completeExceptionally(ex);
    }
    //元素消费完成时候的处理
    public final void onComplete() {
        status.complete(null);
    }
    //有新元素的时候会被调用
    public final void onNext(T item) {
        try {
            //执行Consumer函数
            consumer.accept(item);
        } catch (Throwable ex) {
            subscription.cancel();
            status.completeExceptionally(ex);
        }
    }
}
```

#### ConsumerTask

```java
//订阅者的消费任务
static final class ConsumerTask<T> extends ForkJoinTask<Void>
    implements Runnable, CompletableFuture.AsynchronousCompletionTask {
    final BufferedSubscription<T> consumer;
    ConsumerTask(BufferedSubscription<T> consumer) {
        this.consumer = consumer;
    }
    public final Void getRawResult() { return null; }
    public final void setRawResult(Void v) {}
    public final boolean exec() { consumer.consume(); return false; }
    public final void run() { consumer.consume(); }
}
```

#### BufferedSubscription

```java
//基于数组的可扩展的环形缓冲区，通过CAS原子操作来实现插入和获取元素
//在任何时间内最多只有一个活动的消费者任务
//publisher通过锁来保证单个生产者
//生成者和消费者之间的同步依赖于volatile修饰的ctl、demand、waiting变量
//使用ctl管理状态
//缓存区开始很小，只在需要的时候扩容
//使用了Contended类避免CPU伪共享问题
//当生成者和消费者以不同的速度运行时，会出现失衡，通过人为细分一些消费者方法，包括隔离所有subscriber回调
@jdk.internal.vm.annotation.Contended
static final class BufferedSubscription<T>
    implements Subscription, ForkJoinPool.ManagedBlocker {
    long timeout;                      // Long.MAX_VALUE 表示一直等待
    int head;                          // 下一个获取的位置
    int tail;                          // 下一个插入的位置
    final int maxCapacity;             // 最大缓冲大小
    volatile int ctl;                  // 以volatile方式运行状态标志
    Object[] array;                    // 缓冲数组
    final Subscriber<? super T> subscriber; //消费者
    final BiConsumer<? super Subscriber<? super T>, ? super Throwable> onNextHandler; //用于处理异常
    Executor executor;                 // 线程池，发生错误为null
    Thread waiter;                     // 生成者线程被阻塞
    Throwable pendingError;            // onError发生时会赋值该变量
    BufferedSubscription<T> next;      // publisher使用
    BufferedSubscription<T> nextRetry; // publisher使用

    @jdk.internal.vm.annotation.Contended("c") // 隔离CPU缓存行
    volatile long demand;              // 未获取的请求
    @jdk.internal.vm.annotation.Contended("c")
    volatile int waiting;              // 如果生产者被阻塞，则不为零

    // ctl 16进制值
    static final int CLOSED   = 0x01;  // 设置了，忽略其他位
    static final int ACTIVE   = 0x02;  // 保存消费者任务一直存活
    static final int REQS     = 0x04;  // 非零请求
    static final int ERROR    = 0x08;  // 发生错误时，调用onError
    static final int COMPLETE = 0x10;  // 当完成时，调用onComplete
    static final int RUN      = 0x20;  // 任务正在跑或刚开始跑
    static final int OPEN     = 0x40;  // 订阅后位true

    static final long INTERRUPTED = -1L; // 超时或者中断标志

    BufferedSubscription(Subscriber<? super T> subscriber,
                         Executor executor,
                         BiConsumer<? super Subscriber<? super T>,
                         ? super Throwable> onNextHandler,
                         Object[] array,
                         int maxBufferCapacity) {
        this.subscriber = subscriber;
        this.executor = executor;
        this.onNextHandler = onNextHandler;
        this.array = array;
        this.maxCapacity = maxBufferCapacity;
    }

    // Wrappers for some VarHandle methods

    //cas更新ctl变量
    final boolean weakCasCtl(int cmp, int val) {
        return CTL.weakCompareAndSet(this, cmp, val);
    }

    //进行或计算
    final int getAndBitwiseOrCtl(int bits) {
        return (int)CTL.getAndBitwiseOr(this, bits);
    }

    //这段代码会将demand的值减去指定的k
    final long subtractDemand(int k) {
        long n = (long)(-k);
        //为什么还要再加n呢？
        return n + (long)DEMAND.getAndAdd(this, n);
    }

    //cas设置demand值
    final boolean casDemand(long cmp, long val) {
        return DEMAND.compareAndSet(this, cmp, val);
    }

    // SubmissionPublisher实用方法

    /**
     * 如果关闭了返回true (消费任务可能还在运行中).
     */
    final boolean isClosed() {
        return (ctl & CLOSED) != 0;
    }

    /**
     * 返回估计的元素数量，如果关闭了返回-1
     */
    final int estimateLag() {
        int c = ctl, n = tail - head;
        return ((c & CLOSED) != 0) ? -1 : (n < 0) ? 0 : n;
    }

    // 提交元素的方法

    /**
     * 尝试插入元素，并启动消费任务
     * @return 如果关闭返回-1，0表示满了
     */
    final int offer(T item, boolean unowned) {
        //存放元素的数组
        Object[] a;
        //cap 表示数组的容量
        int stat = 0, cap = ((a = array) == null) ? 0 : a.length; 
        //i 表示要插入的索引位置；n表示已有的元素
        int t = tail, i = t & (cap - 1), n = t + 1 - head;
        if (cap > 0) {
            boolean added;
            if (n >= cap && cap < maxCapacity) // 发生扩容
                added = growAndOffer(item, a, t);
            else if (n >= cap || unowned)      // 如果不是自身的线程，就说明有竞争，需要使用CAS来更新
                added = QA.compareAndSet(a, i, null, item);
            else {                             // 使用release 模式插入元素到数组中
                QA.setRelease(a, i, item);
                added = true;
            }
            //插入成功，tail后移一位
            if (added) {
                tail = t + 1;
                stat = n;
            }
        }
        return startOnOffer(stat);  //尝试唤醒消费线程
    }

    /**
     * 尝试扩展缓冲区，并插入元素，成功返回true。如果内存溢出就会失败
     */
    final boolean growAndOffer(T item, Object[] a, int t) {
        int cap = 0, newCap = 0;
        Object[] newArray = null;
        if (a != null && (cap = a.length) > 0 && (newCap = cap << 1) > 0) { //进行两倍扩容
            try {
                newArray = new Object[newCap];
            } catch (OutOfMemoryError ex) {
            }
        }
        if (newArray == null)
            return false;
        else {
            //从原数组中取值，并插入新数组中
            int newMask = newCap - 1;
            newArray[t-- & newMask] = item;
            for (int mask = cap - 1, k = mask; k >= 0; --k) {
                Object x = QA.getAndSet(a, t & mask, null);
                if (x == null) //x为null说明已经获取
                    break;                    
                else
                    newArray[t-- & newMask] = x;
            }
            array = newArray;
            VarHandle.releaseFence();  //释放空数组
            return true;
        }
    }

    /**
     * 重试版本，没有扩容或者偏置
     */
    final int retryOffer(T item) {
        Object[] a;
        int stat = 0, t = tail, h = head, cap;
        if ((a = array) != null && (cap = a.length) > 0 &&
            QA.compareAndSet(a, (cap - 1) & t, null, item))
            //将tail后移1位
            stat = (tail = t + 1) - h;
        return startOnOffer(stat);
    }

    /**
     * 唤醒消费者任务
     * @return 如果是关闭的返回负数
     */
    final int startOnOffer(int stat) {
        int c; // 如果存在请求且未激活，则启动或保持活动状态
        if (((c = ctl) & (REQS | ACTIVE)) == REQS &&
            ((c = getAndBitwiseOrCtl(RUN | ACTIVE)) & (RUN | CLOSED)) == 0)
            //启动消费者线程
            tryStart();
        else if ((c & CLOSED) != 0)
            stat = -1;
        return stat;
    }

    /**
     * 启动消费的任务。 失败设置错误状态。
     */
    final void tryStart() {
        try {
            Executor e;
            //消费者任务
            ConsumerTask<T> task = new ConsumerTask<T>(this);
            if ((e = executor) != null)   // 发生错误就跳过
                e.execute(task);
        } catch (RuntimeException | Error ex) {
            //设置ctl状态
            getAndBitwiseOrCtl(ERROR | CLOSED);
            throw ex;
        }
    }

    // 向消费者任务发信号

    /**
     * 设置给定的控制位，启动任务，如果任务没有运行或者没有关闭
     * @param 运行状态位
     */
    final void startOnSignal(int bits) {
        if ((ctl & bits) != bits &&
            (getAndBitwiseOrCtl(bits) & (RUN | CLOSED)) == 0)
            tryStart();
    }

    //订阅中
    final void onSubscribe() {
        startOnSignal(RUN | ACTIVE);
    }

    //已完成
    final void onComplete() {
        startOnSignal(RUN | ACTIVE | COMPLETE);
    }

    //发生错误
    final void onError(Throwable ex) {
        int c; Object[] a;      //发生异步错误了清空缓冲区
        if (ex != null)
            pendingError = ex; 
        if (((c = getAndBitwiseOrCtl(ERROR | RUN | ACTIVE)) & CLOSED) == 0) {  //还没有关闭
            if ((c & RUN) == 0)  //没有运行，尝试重试
                tryStart();
            else if ((a = array) != null)  //清空缓冲区
                Arrays.fill(a, null);
        }
    }

    //取消
    public final void cancel() {
        onError(null);
    }

    //请求为获取的数据
    public final void request(long n) {
        if (n > 0L) {
            for (;;) {
                long p = demand, d = p + n;  // saturate
                if (casDemand(p, d < p ? Long.MAX_VALUE : d))
                    break;
            }
            startOnSignal(RUN | ACTIVE | REQS);
        }
        else
            onError(new IllegalArgumentException(
                        "non-positive subscription request"));
    }

    // 消费者的方法

    /**
     * ConsumerTask调用，循环消费元素，或者在提交元素的时候，间接触发
     */
    final void consume() {
        Subscriber<? super T> s;
        if ((s = subscriber) != null) {          // hoist checks
            //触发消费者，请求更多的数据
            subscribeOnOpen(s);
            long d = demand;
            //从数组头到尾进行循环
            for (int h = head, t = tail;;) {
                int c, taken; boolean empty;
                //如果发生了错误，中断消费
                if (((c = ctl) & ERROR) != 0) { 
                    closeOnError(s, null);
                    break;
                }
                //一直获取元素，直到消费完或者发生错误
                else if ((taken = takeItems(s, d, h)) > 0) {
                    //头指针后移
                    head = h += taken;
                    //修改未获取数据的局部变量demand
                    d = subtractDemand(taken);
                }
                else if ((d = demand) == 0L && (c & REQS) != 0)
                    weakCasCtl(c, c & ~REQS);    // 已经消费完，删除ctl中的请求标志位
                else if (d != 0L && (c & REQS) == 0)
                    weakCasCtl(c, c | REQS);     // 有新元素需要消费，增加ctl中的请求标志位
                else if (t == (t = tail)) {      // 稳定性检查
                    if ((empty = (t == h)) && (c & COMPLETE) != 0) {  //已经消费完了，但缓存区还没关闭
                        closeOnComplete(s);      //关闭缓存区
                        break;
                    }
                    else if (empty || d == 0L) {
                        //判断ctl运行标志是否有ACTIVE标志，有bit=ACTIVE
                        //没有bit=RUN
                        int bit = ((c & ACTIVE) != 0) ? ACTIVE : RUN;
                        //删除相应的标志位，并更新ctl值，如果是RUN状态，需要退出消费任务
                        if (weakCasCtl(c, c & ~bit) && bit == RUN)
                            break;               // 取消激活或者退出
                    }
                }
            }
        }
    }

    /**
     * 一直消费元素直到 不可用 或者 达到边界 或者 发生错误
     *
     * @param s subscriber 消费者
     * @param d demand 当前需要获取的数量
     * @param h current head 当前的头指针位置
     * @return number taken   获取的数量
     */
    final int takeItems(Subscriber<? super T> s, long d, int h) {
        Object[] a;
        int k = 0, cap;
        if ((a = array) != null && (cap = a.length) > 0) {
            int m = cap - 1, b = (m >>> 3) + 1; // min(1, cap/8)
            int n = (d < (long)b) ? (int)d : b;
            for (; k < n; ++h, ++k) {
                //获取元素，并置为null
                Object x = QA.getAndSet(a, h & m, null);
                if (waiting != 0)
                    //唤醒被阻塞的生成者
                    signalWaiter();
                if (x == null)
                    break;
                else if (!consumeNext(s, x))  //调用消费者的onNext方法，来处理元素
                    break;
            }
        }
        return k;
    }

    //调用消费者的onNext方法，来处理元素
    final boolean consumeNext(Subscriber<? super T> s, Object x) {
        try {
            @SuppressWarnings("unchecked") T y = (T) x;
            if (s != null)
                s.onNext(y);
            return true;
        } catch (Throwable ex) {
            //处理异常
            handleOnNext(s, ex);
            return false;
        }
    }

    /**
     * 处理 Subscriber.onNext中的异常
     */
    final void handleOnNext(Subscriber<? super T> s, Throwable ex) {
        BiConsumer<? super Subscriber<? super T>, ? super Throwable> h;
        try {
            if ((h = onNextHandler) != null)
                //调用自定义的BiConsumer函数处理异常
                h.accept(s, ex);
        } catch (Throwable ignore) {
        }
        //关闭缓冲并调用消费者的onError方法
        closeOnError(s, ex);
    }

    /**
     * 如果是第一次执行，那么执行subscriber.onSubscribe，用于请求更多数据
     */
    final void subscribeOnOpen(Subscriber<? super T> s) {
        if ((ctl & OPEN) == 0 && (getAndBitwiseOrCtl(OPEN) & OPEN) == 0)
            consumeSubscribe(s);
    }

    //执行subscriber.onSubscribe，用于请求更多数据
    final void consumeSubscribe(Subscriber<? super T> s) {
        try {
            if (s != null) // ignore if disabled
                s.onSubscribe(this);
        } catch (Throwable ex) {
            closeOnError(s, ex);
        }
    }

    /**
     * 缓冲区还没关闭，就执行subscriber.onComplete
     */
    final void closeOnComplete(Subscriber<? super T> s) {
        if ((getAndBitwiseOrCtl(CLOSED) & CLOSED) == 0)
            consumeComplete(s);
    }

    //执行subscriber.onComplete
    final void consumeComplete(Subscriber<? super T> s) {
        try {
            if (s != null)
                s.onComplete();
        } catch (Throwable ignore) {
        }
    }

    /**
     * 发生错误，执行subscriber.onError,并且唤醒阻塞的生产者
     */
    final void closeOnError(Subscriber<? super T> s, Throwable ex) {
        if ((getAndBitwiseOrCtl(ERROR | CLOSED) & CLOSED) == 0) {
            if (ex == null)
                ex = pendingError;
            pendingError = null;  // detach
            executor = null;      // suppress racing start calls
            signalWaiter();
            consumeError(s, ex);
        }
    }

    //执行subscriber.onError
    final void consumeError(Subscriber<? super T> s, Throwable ex) {
        try {
            if (ex != null && s != null)
                s.onError(ex);
        } catch (Throwable ignore) {
        }
    }

    // 阻塞支持

    /**
     * 唤醒等待的生产者
     */
    final void signalWaiter() {
        Thread w;
        waiting = 0;
        if ((w = waiter) != null)
            LockSupport.unpark(w);
    }

    /**
     * 如果缓冲区关闭了，或者有可用空间，返回true
     */
    public final boolean isReleasable() {
        Object[] a; int cap;
        return ((ctl & CLOSED) != 0 ||
                ((a = array) != null && (cap = a.length) > 0 &&
                 QA.getAcquire(a, (cap - 1) & tail) == null));
    }

    /**
     * 一直阻塞知道 timeout, closed,或者有可用空间
     */
    final void awaitSpace(long nanos) {
        if (!isReleasable()) {
            ForkJoinPool.helpAsyncBlocker(executor, this);
            if (!isReleasable()) {
                timeout = nanos;
                try {
                    ForkJoinPool.managedBlock(this);
                } catch (InterruptedException ie) {
                    timeout = INTERRUPTED;
                }
                if (timeout == INTERRUPTED)
                    Thread.currentThread().interrupt();
            }
        }
    }

    /**
     * 给 ManagedBlocker提供的用于阻塞的方法
     */
    public final boolean block() {
        long nanos = timeout;
        boolean timed = (nanos < Long.MAX_VALUE);
        long deadline = timed ? System.nanoTime() + nanos : 0L;
        while (!isReleasable()) {
            if (Thread.interrupted()) {
                timeout = INTERRUPTED;
                if (timed)
                    break;
            }
            else if (timed && (nanos = deadline - System.nanoTime()) <= 0L)
                break;
            else if (waiter == null)
                waiter = Thread.currentThread();
            else if (waiting == 0)
                waiting = 1;
            else if (timed)
                LockSupport.parkNanos(this, nanos);
            else
                LockSupport.park(this);
        }
        waiter = null;
        waiting = 0;
        return true;
    }

    // VarHandle mechanics 一些变量的封装对象，相当于以前调用UNSAFE的方法
    static final VarHandle CTL;
    static final VarHandle DEMAND;
    static final VarHandle QA;

    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            CTL = l.findVarHandle(BufferedSubscription.class, "ctl",
                                  int.class);
            DEMAND = l.findVarHandle(BufferedSubscription.class, "demand",
                                     long.class);
            QA = MethodHandles.arrayElementVarHandle(Object[].class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }

        // 减少首次加载时候的风险
        // LockSupport.park: https://bugs.openjdk.java.net/browse/JDK-8074773
        Class<?> ensureLoaded = LockSupport.class;
    }
}
```

#### ThreadPerTaskExecutor

```java
/**  如果ForkJoinPool.commonPool()不支持并发(公共线程池的并发级别小于1)，将使用该类*/
private static final class ThreadPerTaskExecutor implements Executor {
    ThreadPerTaskExecutor() {}      // 禁止使用构造函数创建类
    public void execute(Runnable r) { new Thread(r).start(); }
}
```

### 基本方法

#### subscribe

```java
//给传入的订阅者订阅，如果已经订阅，将调用订阅者的onError方法抛出IllegalStateException异常
//如果订阅成功，则会异步调用订阅者的onSubscribe方法，如果其中抛出异常，订阅将被取消
//如果SubmissionPublisher被异常关闭，那么订阅者的onError方法会被调用
//如果没有异常被关闭了，就会调用订阅者的onComplete方法
//通过调用Subscription的request方法能够接收其他更多数据
//通过调用Subscription的cancel方法取消订阅
public void subscribe(Subscriber<? super T> subscriber) {
    if (subscriber == null) throw new NullPointerException();
    // 分配初始数组
    int max = maxBufferCapacity; 
    Object[] array = new Object[max < INITIAL_CAPACITY ?
                                max : INITIAL_CAPACITY];
    BufferedSubscription<T> subscription =
        new BufferedSubscription<T>(subscriber, executor, onNextHandler,
                                    array, max);
    //同步解决了一些变量可见性问题
    synchronized (this) {
        if (!subscribed) {
            subscribed = true;
            owner = Thread.currentThread();
        }
        //循环缓冲区链表，并处理订阅
        for (BufferedSubscription<T> b = clients, pred = null;;) {
            //新的订阅者，始终会被放到链表的最后,b的变量会在循环末尾被next赋值，最后一个缓冲区的next值为null
            if (b == null) {
                Throwable ex;
                //执行消费者任务
                subscription.onSubscribe();
                if ((ex = closedException) != null)
                    //异常处理
                    subscription.onError(ex);
                else if (closed)
                    //关闭完成处理
                    subscription.onComplete();
                else if (pred == null)
                    //首次订阅
                    clients = subscription;
                else
                    //后面的订阅者缓冲区通过next连接成链表
                    pred.next = subscription;
                break;
            }
            BufferedSubscription<T> next = b.next;
            if (b.isClosed()) {   // 缓冲区关闭
                b.next = null;    // 断开该缓存区的连接
                if (pred == null)
                    clients = next;
                else
                    pred.next = next;
            }
            else if (subscriber.equals(b.subscriber)) { //重复的订阅者，抛出异常
                b.onError(new IllegalStateException("Duplicate subscribe"));
                break;
            }
            else
                pred = b;
            b = next;
        }
    }
}
```

#### submit

```java
//将元素发布给每个订阅者，如果缓冲区已满，也会一直等待
public int submit(T item) {
    return doOffer(item, Long.MAX_VALUE, null);
}
```

#### offer

```java
//提供了自定义BiPredicate用于判断，参数是否满足指定的要求，如果满足将有一次重试的机会
//该方法如果缓存区已满，不会一直等待
public int offer(T item,
                 BiPredicate<Subscriber<? super T>, ? super T> onDrop) {
    return doOffer(item, 0L, onDrop);
}

//跟前者比，多提供了等待超时时间，和时间单位
public int offer(T item, long timeout, TimeUnit unit,
                 BiPredicate<Subscriber<? super T>, ? super T> onDrop) {
    long nanos = unit.toNanos(timeout);
    // distinguishes from untimed (only wrt interrupt policy)
    if (nanos == Long.MAX_VALUE) --nanos;
    return doOffer(item, nanos, onDrop);
}
```

#### close

```java
//给当前的订阅者发布onComplete信号
//并禁止后面的发布任务
//该方法无法说明所有的订阅者已经完成
public void close() {
    if (!closed) {
        BufferedSubscription<T> b;
        synchronized (this) {
            // no need to re-check closed here
            b = clients;
            clients = null;
            owner = null;
            closed = true;
        }
        //循环处理所有的缓冲区，断开连接，完成消费
        while (b != null) {
            BufferedSubscription<T> next = b.next;
            b.next = null;
            b.onComplete();
            b = next;
        }
    }
}
```

#### closeExceptionally

```java
//给当前的订阅者发送指定的错误信号，并禁止后续发布
//该方法无法说明订阅者是否已经完成
public void closeExceptionally(Throwable error) {
    if (error == null)
        throw new NullPointerException();
    if (!closed) {
        BufferedSubscription<T> b;
        synchronized (this) {
            b = clients;
            if (!closed) {  //再次检查关闭状态，因为有可能其他线程会调用close()
                closedException = error;
                clients = null;
                owner = null;
                closed = true;
            }
        }
        while (b != null) {
            BufferedSubscription<T> next = b.next;
            b.next = null;
            b.onError(error);
            b = next;
        }
    }
}
```

#### isClosed

```java
//是否关闭
public boolean isClosed() {
    return closed;
}
```

#### getClosedException

```java
//获取closed异常
public Throwable getClosedException() {
    return closedException;
}
```

#### hasSubscribers

```java
//判断是否有订阅者，只要有一个订阅者就返回true
//如果有缓冲区已经被关闭，会清理这些缓冲区
public boolean hasSubscribers() {
    boolean nonEmpty = false;
    synchronized (this) {
        for (BufferedSubscription<T> b = clients; b != null;) {
            BufferedSubscription<T> next = b.next;
            if (b.isClosed()) {
                b.next = null;
                b = clients = next;
            }
            else {
                nonEmpty = true;
                break;
            }
        }
    }
    return nonEmpty;
}
```

#### getNumberOfSubscribers

```java
//通过缓冲区来计算还订阅的订阅者数量
public int getNumberOfSubscribers() {
    synchronized (this) {
        return cleanAndCount();
    }
}
```

#### getExecutor

```java
//获取线程池
public Executor getExecutor() {
    return executor;
}
```

#### getMaxBufferCapacity

```java
//获取缓冲区最大容量
public int getMaxBufferCapacity() {
    return maxBufferCapacity;
}
```

#### getSubscribers

```java
//获取还订阅的订阅者列表，并清理缓冲区链表
public List<Subscriber<? super T>> getSubscribers() {
    ArrayList<Subscriber<? super T>> subs = new ArrayList<>();
    synchronized (this) {
        BufferedSubscription<T> pred = null, next;
        for (BufferedSubscription<T> b = clients; b != null; b = next) {
            next = b.next;
            if (b.isClosed()) {
                b.next = null;
                if (pred == null)
                    clients = next;
                else
                    pred.next = next;
            }
            else {
                subs.add(b.subscriber);
                pred = b;
            }
        }
    }
    return subs;
}
```

#### isSubscribed

```java
//通过equals判断，指定的订阅者是否还在订阅
public boolean isSubscribed(Subscriber<? super T> subscriber) {
    if (subscriber == null) throw new NullPointerException();
    if (!closed) {
        synchronized (this) {
            BufferedSubscription<T> pred = null, next;
            for (BufferedSubscription<T> b = clients; b != null; b = next) {
                next = b.next;
                if (b.isClosed()) {
                    b.next = null;
                    if (pred == null)
                        clients = next;
                    else
                        pred.next = next;
                }
                else if (subscriber.equals(b.subscriber))
                    return true;
                else
                    pred = b;
            }
        }
    }
    return false;
}
```

#### estimateMinimumDemand

```java
//返回所有的订阅者中，还没有消费的最少元素数量
public long estimateMinimumDemand() {
    long min = Long.MAX_VALUE;
    boolean nonEmpty = false;
    synchronized (this) {
        BufferedSubscription<T> pred = null, next;
        for (BufferedSubscription<T> b = clients; b != null; b = next) {
            int n; long d;
            next = b.next;
            if ((n = b.estimateLag()) < 0) {
                b.next = null;
                if (pred == null)
                    clients = next;
                else
                    pred.next = next;
            }
            else {
                if ((d = b.demand - n) < min)
                    min = d;
                nonEmpty = true;
                pred = b;
            }
        }
    }
    return nonEmpty ? min : 0;
}
```

#### estimateMaximumLag

```java
//返回所有的订阅者中，还没有消费的最多元素数量
public int estimateMaximumLag() {
    int max = 0;
    synchronized (this) {
        BufferedSubscription<T> pred = null, next;
        for (BufferedSubscription<T> b = clients; b != null; b = next) {
            int n;
            next = b.next;
            if ((n = b.estimateLag()) < 0) {
                b.next = null;
                if (pred == null)
                    clients = next;
                else
                    pred.next = next;
            }
            else {
                if (n > max)
                    max = n;
                pred = b;
            }
        }
    }
    return max;
}
```

#### consume

```java
//公开的消费方法
public CompletableFuture<Void> consume(Consumer<? super T> consumer) {
    if (consumer == null)
        throw new NullPointerException();
    CompletableFuture<Void> status = new CompletableFuture<>();
    subscribe(new ConsumerSubscriber<T>(status, consumer));
    return status;
}
```

#### roundCapacity

```java
//计算最大容量的算法
static final int roundCapacity(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n <= 0) ? 1 : // at least 1
        (n >= BUFFER_CAPACITY_LIMIT) ? BUFFER_CAPACITY_LIMIT : n + 1;
}
```

#### doOffer

```java
//submit 和offer的公共实现
//submit方法nanos == Long.MAX_VALUE
private int doOffer(T item, long nanos,
                    BiPredicate<Subscriber<? super T>, ? super T> onDrop) {
    if (item == null) throw new NullPointerException();
    int lag = 0;
    boolean complete, unowned;
    synchronized (this) {
        Thread t = Thread.currentThread(), o;
        BufferedSubscription<T> b = clients;
        if ((unowned = ((o = owner) != t)) && o != null)
            owner = null;                     // disable bias
        if (b == null)
            complete = closed;
        else {
            complete = false;
            boolean cleanMe = false;
            BufferedSubscription<T> retries = null, rtail = null, next;
            do {
                next = b.next;
                int stat = b.offer(item, unowned);
                if (stat == 0) {              // 缓冲区已满，加入重试列表
                    b.nextRetry = null;       // 先重置原来的nextRetry变量
                    if (rtail == null)
                        retries = b;
                    else
                        rtail.nextRetry = b;
                    rtail = b;
                }
                else if (stat < 0)            // 缓存区已经关闭
                    cleanMe = true;           // 标记清理，将在后面清理
                else if (stat > lag)
                    lag = stat;
            } while ((b = next) != null);  //循环处理，直到处理完成

            if (retries != null || cleanMe)  //重试列表不为空，或者清理标志为true
                //重试发布
                lag = retryOffer(item, nanos, onDrop, retries, lag, cleanMe);
        }
    }
    //已经关闭抛出异常Closed
    if (complete)
        throw new IllegalStateException("Closed");
    else
        return lag;
}
```

#### retryOffer

```java
//重试发布元素
private int retryOffer(T item, long nanos,
                       BiPredicate<Subscriber<? super T>, ? super T> onDrop,
                       BufferedSubscription<T> retries, int lag,
                       boolean cleanMe) {
    for (BufferedSubscription<T> r = retries; r != null;) {
        BufferedSubscription<T> nextRetry = r.nextRetry;
        //将缓冲区存入临时变量后，及时断开连接
        r.nextRetry = null;
        if (nanos > 0L)
            //阻塞等待
            r.awaitSpace(nanos);
        //重试发布
        int stat = r.retryOffer(item);
        //如果还是发布失败，但自定义了BiPredicate函数，会根据自定义规则进行判断，为true会再重试一次
        if (stat == 0 && onDrop != null && onDrop.test(r.subscriber, item))
            stat = r.retryOffer(item);
        if (stat == 0) //缓冲区满了
            lag = (lag >= 0) ? -1 : lag - 1;   //这里有点奇怪？
        else if (stat < 0)  //关闭了，标志为清理
            cleanMe = true;
        else if (lag >= 0 && stat > lag)
            lag = stat;
        r = nextRetry;
    }
    if (cleanMe)
        cleanAndCount();
    return lag;
}
```

#### cleanAndCount

```java
//清理缓存区链表并计算还在订阅的订阅者数量
private int cleanAndCount() {
    int count = 0;
    BufferedSubscription<T> pred = null, next;
    for (BufferedSubscription<T> b = clients; b != null; b = next) {
        next = b.next;
        if (b.isClosed()) {
            b.next = null;
            if (pred == null)
                clients = next;
            else
                pred.next = next;
        }
        else {
            pred = b;
            ++count;
        }
    }
    return count;
}
```

