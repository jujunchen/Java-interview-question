# LongAdder 源码分析

> LongAdder 的作者是Doug Lea大神，该类原本在Guava工具包中，后来被Java8吸收，以下源码基于Java8

LongAdder 继承自Striped64，并实现了Serializable序列化接口。

Striped64 继承自Number，重写了longValue,intValue,floatValue,doubleValue

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ggh62pe1dgj30ki0mgdhu.jpg" alt="image-20200706130244235" style="zoom:50%;" />

## Striped64

### 设计思路

该类提供一个Cell数组，和一个base字段，在多线程情况，让线程去竞争不同的Cell，而不是像AtomicLong去竞争更新同一个原子量。数组索引使用线程的哈希值

Cell数组的长度根据竞争程度，进行扩容，长度为$2^n$(n为原数组长度)，扩容后不会缩小

通过Celll类的@Contended注解，避免了CPU Cache伪共享问题。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ggh5imvylkj30ui0bgq3i.jpg" alt="image-20200706124326887" style="zoom: 50%;" />

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ggh5mpycljj30q40amwf0.jpg" alt="image-20200706124722352" style="zoom:50%;" />

### Cell

```java
//Contended注解防止CPU 缓存行伪共享
@sun.misc.Contended static final class Cell {
    //存放Cell中的具体值，volatile保证可见性
    volatile long value;
    //Cell构造方法
    Cell(long x) { value = x; }
    //cas更新操作,保证原子性，cmp期望值，val新值
    final boolean cas(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
    }
    private static final sun.misc.Unsafe UNSAFE;
    //value偏移量
    private static final long valueOffset;
    //静态代码块，在Cell加载的时候就加载，用于初始化UNSAFE,valueOffset
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> ak = Cell.class;
            //获取value属性的偏移量
            valueOffset = UNSAFE.objectFieldOffset
                (ak.getDeclaredField("value"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

### Striped64属性

```java
/** 获取CPU数量，后面会根据该数量扩容Cell数组 */
static final int NCPU = Runtime.getRuntime().availableProcessors();

/**
 * Cell数组，称为table，长度为2的幂
 */
transient volatile Cell[] cells;

/**
 * 主要在非竞争情况下使用，同时也充当table初始化期间的竞争后备，通过CAS更新
 */
transient volatile long base;

/**
 * 自旋锁（通过CAS操作）用于cells创建或者调整大小时
 */
transient volatile int cellsBusy;

/**
 * default 构造函数
 */
Striped64() {
}
```

在该类的末尾还有一些属性，主要是获取上面属性的偏移量，偏移量是相对于对象地址的偏移量

```java
private static final sun.misc.Unsafe UNSAFE;
//base属性的偏移量
private static final long BASE;
//cellsBusy属性的偏移量
private static final long CELLSBUSY;
//threadLocalRandomProbe的偏移量
private static final long PROBE;
//静态代码块初始化
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> sk = Striped64.class;
        BASE = UNSAFE.objectFieldOffset
            (sk.getDeclaredField("base"));
        CELLSBUSY = UNSAFE.objectFieldOffset
            (sk.getDeclaredField("cellsBusy"));
        Class<?> tk = Thread.class;
        PROBE = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomProbe"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```

### casBase

```java
/**
 * CAS更新base字段，cmp期望值，val更新值
 * compareAndSwapLong 第一参数表示对象，第二个表示属性偏移量，第三个属性期望值，第四个属性需要更新的值
 */
final boolean casBase(long cmp, long val) {
    return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
}
```

看一下UNSAFE中compareAndSwapLong的代码

```c++
jboolean
sun::misc::Unsafe::compareAndSwapLong (jobject obj, jlong offset,
                       jlong expect, jlong update)
{
    //*addr表示获取属性地址对应的值，对象地址+偏移量
  volatile jlong *addr = (jlong*)((char *) obj + offset);
  return compareAndSwap (addr, expect, update);
}
```

```c++
static inline bool
compareAndSwap (volatile jint *addr, jint old, jint new_val)
{
  jboolean result = false;
  spinlock lock;
  // result=原先指针指向的地址的值(*addr)是否与旧的值(old)相等
  if ((result = (*addr == old)))
    // 如果相等则把内存修改为新值
    *addr = new_val;
  return result;
}
```

### casCellsBusy

```java
/**
 * CAS操作更新cellsBusy属性，从0改为1表示获取锁
 * compareAndSwapInt方法同compareAndSwapLong
 */
final boolean casCellsBusy() {
    return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);
}
```

### getProbe

该方法跟Cell数组长度做&运算，用于确定数组索引

```java
/**
 * 返回当前线程的探针值
 * 这段代码从ThreadLocalRandom中copy过来的
 */
static final int getProbe() {
    return UNSAFE.getInt(Thread.currentThread(), PROBE);
}
```

### advanceProbe

在发生Cell竞争的情况下，会调用该方法，重新计算探针值

该方法同样原来是属于ThreadLocalRandom类，因为包的权限问题，ThreadLocalRandom中该方法没有声明为public，包括getProbe()

```java
static final int advanceProbe(int probe) {
    probe ^= probe << 13;   // xorshift
    probe ^= probe >>> 17;
    probe ^= probe << 5;
    UNSAFE.putInt(Thread.currentThread(), PROBE, probe);
    return probe;
}
```

### longAccumulate

该方法处理初始化，调整大小，创建新的Cell和处理竞争问题

```java
/**
*  x 表示更新的值；fn表示操作函数，在LongAdder中为null，LongAccumulator中为自定义函数；wasUncontended表示CAS是否已*  经更新失败
*/
final void longAccumulate(long x, LongBinaryOperator fn,
                          boolean wasUncontended) {
    int h;
    //如果当前线程的探针值=0，说明还未初始化ThreadLocalRandom，强制初始化
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); 
        h = getProbe();
        wasUncontended = true;
    }
    boolean collide = false; 
    for (;;) {
        Cell[] as; Cell a; int n; long v;
        //cells数组不为null，并且长度大于0，表示已经初始化
        if ((as = cells) != null && (n = as.length) > 0) {
            //当前线程映射的索引处为null,将关联新的Cell
            if ((a = as[(n - 1) & h]) == null) {
                //表示没有其他线程再操作cells
                if (cellsBusy == 0) {  
                    //创建包含更新值x的Cell
                    Cell r = new Cell(x);
                    //再次判断是否有其他线程操作cells
                    if (cellsBusy == 0 && casCellsBusy()) {
                        boolean created = false;
                        try {               // Recheck under lock
                            Cell[] rs; int m, j;
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                         fn.applyAsLong(v, x))))
                break;
            else if (n >= NCPU || cells != as)
                collide = false;            // At max size or stale
            else if (!collide)
                collide = true;
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    if (cells == as) {      // Expand table unless stale
                        Cell[] rs = new Cell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        cells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = advanceProbe(h);
        }
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            boolean init = false;
            try {                           // Initialize table
                if (cells == as) {
                    Cell[] rs = new Cell[2];
                    rs[h & 1] = new Cell(x);
                    cells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        else if (casBase(v = base, ((fn == null) ? v + x :
                                    fn.applyAsLong(v, x))))
            break;                          // Fall back on using base
    }
}
```

## LongAdder

继承了Striped64，分析以下该类的每个方法

### add

增加给定值x

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    //
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```

