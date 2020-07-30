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
    static final int CLOSED   = 0x01;  // 设置了，忽略其他值
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

    // Utilities used by SubmissionPublisher

    /**
     * Returns true if closed (consumer task may still be running).
     */
    final boolean isClosed() {
        return (ctl & CLOSED) != 0;
    }

    /**
     * Returns estimated number of buffered items, or negative if
     * closed.
     */
    final int estimateLag() {
        int c = ctl, n = tail - head;
        return ((c & CLOSED) != 0) ? -1 : (n < 0) ? 0 : n;
    }

    // Methods for submitting items

    /**
     * Tries to add item and start consumer task if necessary.
     * @return negative if closed, 0 if saturated, else estimated lag
     */
    final int offer(T item, boolean unowned) {
        Object[] a;
        int stat = 0, cap = ((a = array) == null) ? 0 : a.length;
        int t = tail, i = t & (cap - 1), n = t + 1 - head;
        if (cap > 0) {
            boolean added;
            if (n >= cap && cap < maxCapacity) // resize
                added = growAndOffer(item, a, t);
            else if (n >= cap || unowned)      // need volatile CAS
                added = QA.compareAndSet(a, i, null, item);
            else {                             // can use release mode
                QA.setRelease(a, i, item);
                added = true;
            }
            if (added) {
                tail = t + 1;
                stat = n;
            }
        }
        return startOnOffer(stat);
    }

    /**
     * Tries to expand buffer and add item, returning true on
     * success. Currently fails only if out of memory.
     */
    final boolean growAndOffer(T item, Object[] a, int t) {
        int cap = 0, newCap = 0;
        Object[] newArray = null;
        if (a != null && (cap = a.length) > 0 && (newCap = cap << 1) > 0) {
            try {
                newArray = new Object[newCap];
            } catch (OutOfMemoryError ex) {
            }
        }
        if (newArray == null)
            return false;
        else {                                // take and move items
            int newMask = newCap - 1;
            newArray[t-- & newMask] = item;
            for (int mask = cap - 1, k = mask; k >= 0; --k) {
                Object x = QA.getAndSet(a, t & mask, null);
                if (x == null)
                    break;                    // already consumed
                else
                    newArray[t-- & newMask] = x;
            }
            array = newArray;
            VarHandle.releaseFence();         // release array and slots
            return true;
        }
    }

    /**
     * Version of offer for retries (no resize or bias)
     */
    final int retryOffer(T item) {
        Object[] a;
        int stat = 0, t = tail, h = head, cap;
        if ((a = array) != null && (cap = a.length) > 0 &&
            QA.compareAndSet(a, (cap - 1) & t, null, item))
            stat = (tail = t + 1) - h;
        return startOnOffer(stat);
    }

    /**
     * Tries to start consumer task after offer.
     * @return negative if now closed, else argument
     */
    final int startOnOffer(int stat) {
        int c; // start or keep alive if requests exist and not active
        if (((c = ctl) & (REQS | ACTIVE)) == REQS &&
            ((c = getAndBitwiseOrCtl(RUN | ACTIVE)) & (RUN | CLOSED)) == 0)
            tryStart();
        else if ((c & CLOSED) != 0)
            stat = -1;
        return stat;
    }

    /**
     * Tries to start consumer task. Sets error state on failure.
     */
    final void tryStart() {
        try {
            Executor e;
            ConsumerTask<T> task = new ConsumerTask<T>(this);
            if ((e = executor) != null)   // skip if disabled on error
                e.execute(task);
        } catch (RuntimeException | Error ex) {
            getAndBitwiseOrCtl(ERROR | CLOSED);
            throw ex;
        }
    }

    // Signals to consumer tasks

    /**
     * Sets the given control bits, starting task if not running or closed.
     * @param bits state bits, assumed to include RUN but not CLOSED
     */
    final void startOnSignal(int bits) {
        if ((ctl & bits) != bits &&
            (getAndBitwiseOrCtl(bits) & (RUN | CLOSED)) == 0)
            tryStart();
    }

    final void onSubscribe() {
        startOnSignal(RUN | ACTIVE);
    }

    final void onComplete() {
        startOnSignal(RUN | ACTIVE | COMPLETE);
    }

    final void onError(Throwable ex) {
        int c; Object[] a;      // to null out buffer on async error
        if (ex != null)
            pendingError = ex;  // races are OK
        if (((c = getAndBitwiseOrCtl(ERROR | RUN | ACTIVE)) & CLOSED) == 0) {
            if ((c & RUN) == 0)
                tryStart();
            else if ((a = array) != null)
                Arrays.fill(a, null);
        }
    }

    public final void cancel() {
        onError(null);
    }

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

    // Consumer task actions

    /**
     * Consumer loop, called from ConsumerTask, or indirectly when
     * helping during submit.
     */
    final void consume() {
        Subscriber<? super T> s;
        if ((s = subscriber) != null) {          // hoist checks
            subscribeOnOpen(s);
            long d = demand;
            for (int h = head, t = tail;;) {
                int c, taken; boolean empty;
                if (((c = ctl) & ERROR) != 0) {
                    closeOnError(s, null);
                    break;
                }
                else if ((taken = takeItems(s, d, h)) > 0) {
                    head = h += taken;
                    d = subtractDemand(taken);
                }
                else if ((d = demand) == 0L && (c & REQS) != 0)
                    weakCasCtl(c, c & ~REQS);    // exhausted demand
                else if (d != 0L && (c & REQS) == 0)
                    weakCasCtl(c, c | REQS);     // new demand
                else if (t == (t = tail)) {      // stability check
                    if ((empty = (t == h)) && (c & COMPLETE) != 0) {
                        closeOnComplete(s);      // end of stream
                        break;
                    }
                    else if (empty || d == 0L) {
                        int bit = ((c & ACTIVE) != 0) ? ACTIVE : RUN;
                        if (weakCasCtl(c, c & ~bit) && bit == RUN)
                            break;               // un-keep-alive or exit
                    }
                }
            }
        }
    }

    /**
     * Consumes some items until unavailable or bound or error.
     *
     * @param s subscriber
     * @param d current demand
     * @param h current head
     * @return number taken
     */
    final int takeItems(Subscriber<? super T> s, long d, int h) {
        Object[] a;
        int k = 0, cap;
        if ((a = array) != null && (cap = a.length) > 0) {
            int m = cap - 1, b = (m >>> 3) + 1; // min(1, cap/8)
            int n = (d < (long)b) ? (int)d : b;
            for (; k < n; ++h, ++k) {
                Object x = QA.getAndSet(a, h & m, null);
                if (waiting != 0)
                    signalWaiter();
                if (x == null)
                    break;
                else if (!consumeNext(s, x))
                    break;
            }
        }
        return k;
    }

    final boolean consumeNext(Subscriber<? super T> s, Object x) {
        try {
            @SuppressWarnings("unchecked") T y = (T) x;
            if (s != null)
                s.onNext(y);
            return true;
        } catch (Throwable ex) {
            handleOnNext(s, ex);
            return false;
        }
    }

    /**
     * Processes exception in Subscriber.onNext.
     */
    final void handleOnNext(Subscriber<? super T> s, Throwable ex) {
        BiConsumer<? super Subscriber<? super T>, ? super Throwable> h;
        try {
            if ((h = onNextHandler) != null)
                h.accept(s, ex);
        } catch (Throwable ignore) {
        }
        closeOnError(s, ex);
    }

    /**
     * Issues subscriber.onSubscribe if this is first signal.
     */
    final void subscribeOnOpen(Subscriber<? super T> s) {
        if ((ctl & OPEN) == 0 && (getAndBitwiseOrCtl(OPEN) & OPEN) == 0)
            consumeSubscribe(s);
    }

    final void consumeSubscribe(Subscriber<? super T> s) {
        try {
            if (s != null) // ignore if disabled
                s.onSubscribe(this);
        } catch (Throwable ex) {
            closeOnError(s, ex);
        }
    }

    /**
     * Issues subscriber.onComplete unless already closed.
     */
    final void closeOnComplete(Subscriber<? super T> s) {
        if ((getAndBitwiseOrCtl(CLOSED) & CLOSED) == 0)
            consumeComplete(s);
    }

    final void consumeComplete(Subscriber<? super T> s) {
        try {
            if (s != null)
                s.onComplete();
        } catch (Throwable ignore) {
        }
    }

    /**
     * Issues subscriber.onError, and unblocks producer if needed.
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

    final void consumeError(Subscriber<? super T> s, Throwable ex) {
        try {
            if (ex != null && s != null)
                s.onError(ex);
        } catch (Throwable ignore) {
        }
    }

    // Blocking support

    /**
     * Unblocks waiting producer.
     */
    final void signalWaiter() {
        Thread w;
        waiting = 0;
        if ((w = waiter) != null)
            LockSupport.unpark(w);
    }

    /**
     * Returns true if closed or space available.
     * For ManagedBlocker.
     */
    public final boolean isReleasable() {
        Object[] a; int cap;
        return ((ctl & CLOSED) != 0 ||
                ((a = array) != null && (cap = a.length) > 0 &&
                 QA.getAcquire(a, (cap - 1) & tail) == null));
    }

    /**
     * Helps or blocks until timeout, closed, or space available.
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
     * Blocks until closed, space available or timeout.
     * For ManagedBlocker.
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

    // VarHandle mechanics
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

        // Reduce the risk of rare disastrous classloading in first call to
        // LockSupport.park: https://bugs.openjdk.java.net/browse/JDK-8074773
        Class<?> ensureLoaded = LockSupport.class;
    }
}
```

#### ThreadPerTaskExecutor

```java
/**  如果ForkJoinPool.commonPool()不支持并发，将使用该类 */
private static final class ThreadPerTaskExecutor implements Executor {
    ThreadPerTaskExecutor() {}      // 禁止使用构造函数创建类
    public void execute(Runnable r) { new Thread(r).start(); }
}
```