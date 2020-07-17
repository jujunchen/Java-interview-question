# ArrayBlockingQueue源码分析

> 源码基于open jdk 11

![image-20200716182357663](https://tva1.sinaimg.cn/large/007S8ZIlly1ggszjzgkvwj30s00tmdhd.jpg)

ArrayBlockingQueue是通过有界数组方式实现的阻塞队列 , 通过ReentrantLock实现线程安全，阻塞通过Condition实现，出队和入队使用同一把锁。

有两个内部类Itr 和 Itrs，Itrs内部又实现了Node类，像LinkedBlockingQueue和ConcurrentLinkedQueue都有Node类，但两个类有不同，该类的Node类定义在Itrs的内部类中

先来看下该类哪些属性

## 属性

```java
/** The queued items */
final Object[] items;

/** items index for next take, poll, peek or remove */
int takeIndex;

/** items index for next put, offer, or add */
int putIndex;

/** Number of elements in the queue */
int count;

/*
 * Concurrency control uses the classic two-condition algorithm
 * found in any textbook.
 */

/** Main lock guarding all access */
final ReentrantLock lock;

/** Condition for waiting takes */
private final Condition notEmpty;

/** Condition for waiting puts */
private final Condition notFull;

/**
 * Shared state for currently active iterators, or null if there
 * are known not to be any.  Allows queue operations to update
 * iterator state.
 */
transient Itrs itrs;
```

## 内部类

### Itr

```java
private class Itr implements Iterator<E> {
    /** Index to look for new nextItem; NONE at end */
    private int cursor;

    /** Element to be returned by next call to next(); null if none */
    private E nextItem;

    /** Index of nextItem; NONE if none, REMOVED if removed elsewhere */
    private int nextIndex;

    /** Last element returned; null if none or not detached. */
    private E lastItem;

    /** Index of lastItem, NONE if none, REMOVED if removed elsewhere */
    private int lastRet;

    /** Previous value of takeIndex, or DETACHED when detached */
    private int prevTakeIndex;

    /** Previous value of iters.cycles */
    private int prevCycles;

    /** Special index value indicating "not available" or "undefined" */
    private static final int NONE = -1;

    /**
         * Special index value indicating "removed elsewhere", that is,
         * removed by some operation other than a call to this.remove().
         */
    private static final int REMOVED = -2;

    /** Special value for prevTakeIndex indicating "detached mode" */
    private static final int DETACHED = -3;

    Itr() {
        lastRet = NONE;
        final ReentrantLock lock = ArrayBlockingQueue.this.lock;
        lock.lock();
        try {
            if (count == 0) {
                // assert itrs == null;
                cursor = NONE;
                nextIndex = NONE;
                prevTakeIndex = DETACHED;
            } else {
                final int takeIndex = ArrayBlockingQueue.this.takeIndex;
                prevTakeIndex = takeIndex;
                nextItem = itemAt(nextIndex = takeIndex);
                cursor = incCursor(takeIndex);
                if (itrs == null) {
                    itrs = new Itrs(this);
                } else {
                    itrs.register(this); // in this order
                    itrs.doSomeSweeping(false);
                }
                prevCycles = itrs.cycles;
                // assert takeIndex >= 0;
                // assert prevTakeIndex == takeIndex;
                // assert nextIndex >= 0;
                // assert nextItem != null;
            }
        } finally {
            lock.unlock();
        }
    }

    boolean isDetached() {
        // assert lock.isHeldByCurrentThread();
        return prevTakeIndex < 0;
    }

    private int incCursor(int index) {
        // assert lock.isHeldByCurrentThread();
        if (++index == items.length) index = 0;
        if (index == putIndex) index = NONE;
        return index;
    }

    /**
         * Returns true if index is invalidated by the given number of
         * dequeues, starting from prevTakeIndex.
         */
    private boolean invalidated(int index, int prevTakeIndex,
                                long dequeues, int length) {
        if (index < 0)
            return false;
        int distance = index - prevTakeIndex;
        if (distance < 0)
            distance += length;
        return dequeues > distance;
    }

    /**
         * Adjusts indices to incorporate all dequeues since the last
         * operation on this iterator.  Call only from iterating thread.
         */
    private void incorporateDequeues() {
        // assert lock.isHeldByCurrentThread();
        // assert itrs != null;
        // assert !isDetached();
        // assert count > 0;

        final int cycles = itrs.cycles;
        final int takeIndex = ArrayBlockingQueue.this.takeIndex;
        final int prevCycles = this.prevCycles;
        final int prevTakeIndex = this.prevTakeIndex;

        if (cycles != prevCycles || takeIndex != prevTakeIndex) {
            final int len = items.length;
            // how far takeIndex has advanced since the previous
            // operation of this iterator
            long dequeues = (long) (cycles - prevCycles) * len
                + (takeIndex - prevTakeIndex);

            // Check indices for invalidation
            if (invalidated(lastRet, prevTakeIndex, dequeues, len))
                lastRet = REMOVED;
            if (invalidated(nextIndex, prevTakeIndex, dequeues, len))
                nextIndex = REMOVED;
            if (invalidated(cursor, prevTakeIndex, dequeues, len))
                cursor = takeIndex;

            if (cursor < 0 && nextIndex < 0 && lastRet < 0)
                detach();
            else {
                this.prevCycles = cycles;
                this.prevTakeIndex = takeIndex;
            }
        }
    }

    /**
         * Called when itrs should stop tracking this iterator, either
         * because there are no more indices to update (cursor < 0 &&
         * nextIndex < 0 && lastRet < 0) or as a special exception, when
         * lastRet >= 0, because hasNext() is about to return false for the
         * first time.  Call only from iterating thread.
         */
    private void detach() {
        // Switch to detached mode
        // assert lock.isHeldByCurrentThread();
        // assert cursor == NONE;
        // assert nextIndex < 0;
        // assert lastRet < 0 || nextItem == null;
        // assert lastRet < 0 ^ lastItem != null;
        if (prevTakeIndex >= 0) {
            // assert itrs != null;
            prevTakeIndex = DETACHED;
            // try to unlink from itrs (but not too hard)
            itrs.doSomeSweeping(true);
        }
    }

    /**
         * For performance reasons, we would like not to acquire a lock in
         * hasNext in the common case.  To allow for this, we only access
         * fields (i.e. nextItem) that are not modified by update operations
         * triggered by queue modifications.
         */
    public boolean hasNext() {
        if (nextItem != null)
            return true;
        noNext();
        return false;
    }

    private void noNext() {
        final ReentrantLock lock = ArrayBlockingQueue.this.lock;
        lock.lock();
        try {
            // assert cursor == NONE;
            // assert nextIndex == NONE;
            if (!isDetached()) {
                // assert lastRet >= 0;
                incorporateDequeues(); // might update lastRet
                if (lastRet >= 0) {
                    lastItem = itemAt(lastRet);
                    // assert lastItem != null;
                    detach();
                }
            }
            // assert isDetached();
            // assert lastRet < 0 ^ lastItem != null;
        } finally {
            lock.unlock();
        }
    }

    public E next() {
        final E e = nextItem;
        if (e == null)
            throw new NoSuchElementException();
        final ReentrantLock lock = ArrayBlockingQueue.this.lock;
        lock.lock();
        try {
            if (!isDetached())
                incorporateDequeues();
            // assert nextIndex != NONE;
            // assert lastItem == null;
            lastRet = nextIndex;
            final int cursor = this.cursor;
            if (cursor >= 0) {
                nextItem = itemAt(nextIndex = cursor);
                // assert nextItem != null;
                this.cursor = incCursor(cursor);
            } else {
                nextIndex = NONE;
                nextItem = null;
                if (lastRet == REMOVED) detach();
            }
        } finally {
            lock.unlock();
        }
        return e;
    }

    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        final ReentrantLock lock = ArrayBlockingQueue.this.lock;
        lock.lock();
        try {
            final E e = nextItem;
            if (e == null) return;
            if (!isDetached())
                incorporateDequeues();
            action.accept(e);
            if (isDetached() || cursor < 0) return;
            final Object[] items = ArrayBlockingQueue.this.items;
            for (int i = cursor, end = putIndex,
                 to = (i < end) ? end : items.length;
                 ; i = 0, to = end) {
                for (; i < to; i++)
                    action.accept(itemAt(items, i));
                if (to == end) break;
            }
        } finally {
            // Calling forEachRemaining is a strong hint that this
            // iteration is surely over; supporting remove() after
            // forEachRemaining() is more trouble than it's worth
            cursor = nextIndex = lastRet = NONE;
            nextItem = lastItem = null;
            detach();
            lock.unlock();
        }
    }

    public void remove() {
        final ReentrantLock lock = ArrayBlockingQueue.this.lock;
        lock.lock();
        // assert lock.getHoldCount() == 1;
        try {
            if (!isDetached())
                incorporateDequeues(); // might update lastRet or detach
            final int lastRet = this.lastRet;
            this.lastRet = NONE;
            if (lastRet >= 0) {
                if (!isDetached())
                    removeAt(lastRet);
                else {
                    final E lastItem = this.lastItem;
                    // assert lastItem != null;
                    this.lastItem = null;
                    if (itemAt(lastRet) == lastItem)
                        removeAt(lastRet);
                }
            } else if (lastRet == NONE)
                throw new IllegalStateException();
            // else lastRet == REMOVED and the last returned element was
            // previously asynchronously removed via an operation other
            // than this.remove(), so nothing to do.

            if (cursor < 0 && nextIndex < 0)
                detach();
        } finally {
            lock.unlock();
            // assert lastRet == NONE;
            // assert lastItem == null;
        }
    }

    /**
         * Called to notify the iterator that the queue is empty, or that it
         * has fallen hopelessly behind, so that it should abandon any
         * further iteration, except possibly to return one more element
         * from next(), as promised by returning true from hasNext().
         */
    void shutdown() {
        // assert lock.isHeldByCurrentThread();
        cursor = NONE;
        if (nextIndex >= 0)
            nextIndex = REMOVED;
        if (lastRet >= 0) {
            lastRet = REMOVED;
            lastItem = null;
        }
        prevTakeIndex = DETACHED;
        // Don't set nextItem to null because we must continue to be
        // able to return it on next().
        //
        // Caller will unlink from itrs when convenient.
    }

    private int distance(int index, int prevTakeIndex, int length) {
        int distance = index - prevTakeIndex;
        if (distance < 0)
            distance += length;
        return distance;
    }

    /**
         * Called whenever an interior remove (not at takeIndex) occurred.
         *
         * @return true if this iterator should be unlinked from itrs
         */
    boolean removedAt(int removedIndex) {
        // assert lock.isHeldByCurrentThread();
        if (isDetached())
            return true;

        final int takeIndex = ArrayBlockingQueue.this.takeIndex;
        final int prevTakeIndex = this.prevTakeIndex;
        final int len = items.length;
        // distance from prevTakeIndex to removedIndex
        final int removedDistance =
            len * (itrs.cycles - this.prevCycles
                   + ((removedIndex < takeIndex) ? 1 : 0))
            + (removedIndex - prevTakeIndex);
        // assert itrs.cycles - this.prevCycles >= 0;
        // assert itrs.cycles - this.prevCycles <= 1;
        // assert removedDistance > 0;
        // assert removedIndex != takeIndex;
        int cursor = this.cursor;
        if (cursor >= 0) {
            int x = distance(cursor, prevTakeIndex, len);
            if (x == removedDistance) {
                if (cursor == putIndex)
                    this.cursor = cursor = NONE;
            }
            else if (x > removedDistance) {
                // assert cursor != prevTakeIndex;
                this.cursor = cursor = dec(cursor, len);
            }
        }
        int lastRet = this.lastRet;
        if (lastRet >= 0) {
            int x = distance(lastRet, prevTakeIndex, len);
            if (x == removedDistance)
                this.lastRet = lastRet = REMOVED;
            else if (x > removedDistance)
                this.lastRet = lastRet = dec(lastRet, len);
        }
        int nextIndex = this.nextIndex;
        if (nextIndex >= 0) {
            int x = distance(nextIndex, prevTakeIndex, len);
            if (x == removedDistance)
                this.nextIndex = nextIndex = REMOVED;
            else if (x > removedDistance)
                this.nextIndex = nextIndex = dec(nextIndex, len);
        }
        if (cursor < 0 && nextIndex < 0 && lastRet < 0) {
            this.prevTakeIndex = DETACHED;
            return true;
        }
        return false;
    }

    /**
         * Called whenever takeIndex wraps around to zero.
         *
         * @return true if this iterator should be unlinked from itrs
         */
    boolean takeIndexWrapped() {
        // assert lock.isHeldByCurrentThread();
        if (isDetached())
            return true;
        if (itrs.cycles - prevCycles > 1) {
            // All the elements that existed at the time of the last
            // operation are gone, so abandon further iteration.
            shutdown();
            return true;
        }
        return false;
    }

    //         /** Uncomment for debugging. */
    //         public String toString() {
    //             return ("cursor=" + cursor + " " +
    //                     "nextIndex=" + nextIndex + " " +
    //                     "lastRet=" + lastRet + " " +
    //                     "nextItem=" + nextItem + " " +
    //                     "lastItem=" + lastItem + " " +
    //                     "prevCycles=" + prevCycles + " " +
    //                     "prevTakeIndex=" + prevTakeIndex + " " +
    //                     "size()=" + size() + " " +
    //                     "remainingCapacity()=" + remainingCapacity());
    //         }
}
```

### Itrs

```java
class Itrs {

    /**
     * Node in a linked list of weak iterator references.
     */
    private class Node extends WeakReference<Itr> {
        Node next;

        Node(Itr iterator, Node next) {
            super(iterator);
            this.next = next;
        }
    }

    /** Incremented whenever takeIndex wraps around to 0 */
    int cycles;

    /** Linked list of weak iterator references */
    private Node head;

    /** Used to expunge stale iterators */
    private Node sweeper;

    private static final int SHORT_SWEEP_PROBES = 4;
    private static final int LONG_SWEEP_PROBES = 16;

    Itrs(Itr initial) {
        register(initial);
    }

    /**
     * Sweeps itrs, looking for and expunging stale iterators.
     * If at least one was found, tries harder to find more.
     * Called only from iterating thread.
     *
     * @param tryHarder whether to start in try-harder mode, because
     * there is known to be at least one iterator to collect
     */
    void doSomeSweeping(boolean tryHarder) {
        // assert lock.isHeldByCurrentThread();
        // assert head != null;
        int probes = tryHarder ? LONG_SWEEP_PROBES : SHORT_SWEEP_PROBES;
        Node o, p;
        final Node sweeper = this.sweeper;
        boolean passedGo;   // to limit search to one full sweep

        if (sweeper == null) {
            o = null;
            p = head;
            passedGo = true;
        } else {
            o = sweeper;
            p = o.next;
            passedGo = false;
        }

        for (; probes > 0; probes--) {
            if (p == null) {
                if (passedGo)
                    break;
                o = null;
                p = head;
                passedGo = true;
            }
            final Itr it = p.get();
            final Node next = p.next;
            if (it == null || it.isDetached()) {
                // found a discarded/exhausted iterator
                probes = LONG_SWEEP_PROBES; // "try harder"
                // unlink p
                p.clear();
                p.next = null;
                if (o == null) {
                    head = next;
                    if (next == null) {
                        // We've run out of iterators to track; retire
                        itrs = null;
                        return;
                    }
                }
                else
                    o.next = next;
            } else {
                o = p;
            }
            p = next;
        }

        this.sweeper = (p == null) ? null : o;
    }

    /**
     * Adds a new iterator to the linked list of tracked iterators.
     */
    void register(Itr itr) {
        // assert lock.isHeldByCurrentThread();
        head = new Node(itr, head);
    }

    /**
     * Called whenever takeIndex wraps around to 0.
     *
     * Notifies all iterators, and expunges any that are now stale.
     */
    void takeIndexWrapped() {
        // assert lock.isHeldByCurrentThread();
        cycles++;
        for (Node o = null, p = head; p != null;) {
            final Itr it = p.get();
            final Node next = p.next;
            if (it == null || it.takeIndexWrapped()) {
                // unlink p
                // assert it == null || it.isDetached();
                p.clear();
                p.next = null;
                if (o == null)
                    head = next;
                else
                    o.next = next;
            } else {
                o = p;
            }
            p = next;
        }
        if (head == null)   // no more iterators to track
            itrs = null;
    }

    /**
     * Called whenever an interior remove (not at takeIndex) occurred.
     *
     * Notifies all iterators, and expunges any that are now stale.
     */
    void removedAt(int removedIndex) {
        for (Node o = null, p = head; p != null;) {
            final Itr it = p.get();
            final Node next = p.next;
            if (it == null || it.removedAt(removedIndex)) {
                // unlink p
                // assert it == null || it.isDetached();
                p.clear();
                p.next = null;
                if (o == null)
                    head = next;
                else
                    o.next = next;
            } else {
                o = p;
            }
            p = next;
        }
        if (head == null)   // no more iterators to track
            itrs = null;
    }

    /**
     * Called whenever the queue becomes empty.
     *
     * Notifies all active iterators that the queue is empty,
     * clears all weak refs, and unlinks the itrs datastructure.
     */
    void queueIsEmpty() {
        // assert lock.isHeldByCurrentThread();
        for (Node p = head; p != null; p = p.next) {
            Itr it = p.get();
            if (it != null) {
                p.clear();
                it.shutdown();
            }
        }
        head = null;
        itrs = null;
    }

    /**
     * Called whenever an element has been dequeued (at takeIndex).
     */
    void elementDequeued() {
        // assert lock.isHeldByCurrentThread();
        if (count == 0)
            queueIsEmpty();
        else if (takeIndex == 0)
            takeIndexWrapped();
    }
}
```

