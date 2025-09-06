# Striped64、LongAdder、LongAccumulator 源码分析

[TOC]
> LongAdder 的作者是Doug Lea大神，该类原本在Guava工具包中，后来被Java8吸收，以下源码基于Java8

LongAdder 继承自Striped64，并实现了Serializable序列化接口。

Striped64 继承自Number，重写了longValue,intValue,floatValue,doubleValue

![111](https://tva1.sinaimg.cn/large/007S8ZIlly1ggiavs03zjj30nd0d8jse.jpg)

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

看一下UNSAFE中compareAndSwapLong的C++代码

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
*  x 表示更新的值；fn表示操作函数，在LongAdder中为null，LongAccumulator中为自定义函数；wasUncontended表示CAS是否
*  已经更新失败,false表示失败，说明有竞争，true表示成功
*/
final void longAccumulate(long x, LongBinaryOperator fn,
                          boolean wasUncontended) {
    int h;
    //如果当前线程的探针值=0，说明还未初始化ThreadLocalRandom，强制初始化
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); 
        h = getProbe();
        //true表示没有竞争
        wasUncontended = true;
    }
    //一个碰撞标记，false表示没有发生碰撞
    boolean collide = false; 
    for (;;) {
        Cell[] as; Cell a; int n; long v;
        //cells数组不为null，并且长度大于0，表示已经初始化
        //如下else if语句 如果其中一个为true，即会跳到advanceProbe()进行重写hash计算
        if ((as = cells) != null && (n = as.length) > 0) {
            //当前线程映射的索引处为null,将关联新的Cell
            if ((a = as[(n - 1) & h]) == null) {
                //表示没有其他线程再操作cells
                if (cellsBusy == 0) {  
                    //创建包含更新值x的Cell
                    Cell r = new Cell(x);
                    //再次判断是否有其他线程操作cells，没有的话，CAS将cellsBusy从0修改为1，成功表示获取到锁
                    if (cellsBusy == 0 && casCellsBusy()) {
                        //是否已成功关联的标识
                        boolean created = false;
                        try {
                            Cell[] rs; int m, j;
                            //再次检查索引处的槽是否为null
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                //关联新的Cell到该空闲的槽
                                rs[j] = r;
                                //成功关联
                                created = true;
                            }
                        } finally {
                            //释放锁
                            cellsBusy = 0;
                        }
                        //成功关联了，就跳出循环结束
                        if (created)
                            break;
                        //继续尝试
                        continue;           
                    }
                }
                collide = false;
            }
            //之前CAS操作已经失败，wasUncontended=false，false说明存在竞争，true说明不存在竞争
            else if (!wasUncontended)
                //重新rehash后继续
                wasUncontended = true;     
            //CAS更新当前槽，fn==null只做加法，否则运用fn函数计算更新值
            else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                         fn.applyAsLong(v, x))))
                //更新成功跳出循环，结束
                break;
            //cells长度大于等于CPU数量，cells!=as 表示已经发生了扩容
            else if (n >= NCPU || cells != as)
                collide = false; 
            //如果不存在冲突，则设置为存在冲突
            //为什么要这么做？看前面的代码collide=false时，一种是cellsBusy==1时，表示有其他线程在操作Cell，
            //一种是n>= NCPU || cells != as 两种情况都需要进行rehash,不需要再继续执行扩容，只需要再次重试
            else if (!collide)
                collide = true;
            //扩容
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    //cells的地址没有改变，表示没有进行过扩容
                    if (cells == as) {
                        //新数组长度为n*2
                        Cell[] rs = new Cell[n << 1];
                        //赋值元素
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        //关联地址
                        cells = rs;
                    }
                } finally {
                    //释放锁
                    cellsBusy = 0;
                }
                collide = false;
                //扩容后，重试
                continue;                   
            }
            //重新rehash
            h = advanceProbe(h);
        }
        //这种情况是，表还没有初始化
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            boolean init = false;
            try {                           // Initialize table
                if (cells == as) {
                    //初始长度为2
                    Cell[] rs = new Cell[2];
                    rs[h & 1] = new Cell(x);
                    cells = rs;
                    init = true;
                }
            } finally {
                //释放锁
                cellsBusy = 0;
            }
            //初始化完，跳出循环，结束
            if (init)
                break;
        }
        //后备操作，获取锁失败，使用base操作
        else if (casBase(v = base, ((fn == null) ? v + x :
                                    fn.applyAsLong(v, x))))
            break;                          
    }
}
```

## LongAdder

### add

增加给定值x，首先会尝试更新base字段，如果更新失败，更新线程映射槽Cell的value字段

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    //如果cells = null, 就会执行casBase
    //如果cells不为空，或者base变量CAS更新失败，说明产生竞争
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        //as == null || (m = as.length - 1) < 0 此时说明还未初始化，需要初始化
        //(a = as[getProbe() & m]) == null 此时说明当前线程映射的槽为空，需要创建新的Cell并关联
        //!(uncontended = a.cas(v = a.value, v + x)) 此时说明该位置的槽不为空，但value更新失败，说明有竞争
        //需要换槽
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            //做初始化，扩容，创建新的Cell和处理竞争问题
            longAccumulate(x, null, uncontended);
    }
}
```

### increment、decrement

```java
/**
 * 递增1，相当于add(1)
 */
public void increment() {
    add(1L);
}

/**
 * 递减-1，相当于add(-1)
 */
public void decrement() {
    add(-1L);
}
```

### sum

获取累加值，base+所有Cell的value值。由于在做累加的时候，没有加锁，可能期间其他线程对Cell中的value 进行修改，所以累加的值是一个原子快照值

```java
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;
    if (as != null) {
        //循环cells数组，累加value
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

### reset

重置base，Cell中的value值为0

```java
public void reset() {
    Cell[] as = cells; Cell a;
    base = 0L;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                a.value = 0L;
        }
    }
}
```

### sumThenReset

累加值，并重置base，即Cell中的值为0。在多线程下有问题，可能第一个线程做累加，并重置为0，第二个线程累加调用的值都变为0

```java
public long sumThenReset() {
    Cell[] as = cells; Cell a;
    long sum = base;
    base = 0L;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null) {
                sum += a.value;
                a.value = 0L;
            }
        }
    }
    return sum;
}
```

### toString、longValue、intValue、floatValue、doubleValue

这些方法都调用sum()方法，返回累加值

```java
/**
 * Returns the String representation of the {@link #sum}.
 * @return the String representation of the {@link #sum}
 */
public String toString() {
    return Long.toString(sum());
}

/**
 * Equivalent to {@link #sum}.
 *
 * @return the sum
 */
public long longValue() {
    return sum();
}

/**
 * Returns the {@link #sum} as an {@code int} after a narrowing
 * primitive conversion.
 */
public int intValue() {
    return (int)sum();
}

/**
 * Returns the {@link #sum} as a {@code float}
 * after a widening primitive conversion.
 */
public float floatValue() {
    return (float)sum();
}

/**
 * Returns the {@link #sum} as a {@code double} after a widening
 * primitive conversion.
 */
public double doubleValue() {
    return (double)sum();
}
```

### SerializationProxy

序列化代理类，其可以阻止伪字节流的攻击，以及内部域的盗用攻击

详见：[考虑用序列化代理代替序列化实例](https://blog.csdn.net/xujiajun945/article/details/84245471)

## LongAccumulator

该类同样继承了Striped64,实现了序列化接口

LongAccumulator 相比LongAdder,可以为累加器提供非0的初始值，后者只能提供默认为0的值，另外前者可以提供自定义函数，指定运算规则，后者只能进行累加运算。

LongAdder就相当于用LongAccumulator这样写，初始值为0，两数累加

`LongAccumulator longAccumulator1 = new LongAccumulator((x,y) -> x + y, 0);`

### 构造函数，属性

```java
//函数式接口
private final LongBinaryOperator function;
//初始值
private final long identity;

/**
 * 使用给定的函数接口，和初始值创建实例
 */
public LongAccumulator(LongBinaryOperator accumulatorFunction,
                       long identity) {
    this.function = accumulatorFunction;
    base = this.identity = identity;
}
```

```java
@FunctionalInterface
public interface LongBinaryOperator {

    /**
     * 根据两个参数计算并返回结果
     *
     * @param left the first operand
     * @param right the second operand
     * @return the operator result
     */
    long applyAsLong(long left, long right);
}
```

### accumulate

根据给定值x做计算，该方法类似[Longadder#add](#add)

```java
public void accumulate(long x) {
    Cell[] as; long b, v, r; int m; Cell a;
    //1.先求(r = function.applyAsLong(b = base, x)) != b && !casBase(b, r) 这部分，
    //  1.1 表示base与给定值x使用函数计算后不等于原来的base值，值改变了，那么使用计算后的值r通过CAS操作更新base值，如果
    //      更新失败继续后面的操作
    //  1.2 如果使用函数计算后的值没有改变，该部分表达式为false，无需在执行casBase，接着判断(as = cells) != null
    //      如果(as = cells) != null 为false，结束，如果是true，表示cells之前已经初始化，发生过竞争，需要进一步操作
    if ((as = cells) != null ||
        (r = function.applyAsLong(b = base, x)) != b && !casBase(b, r)) {
        boolean uncontended = true;
        //as == null || (m = as.length - 1) < 0 此时说明还未初始化，需要初始化
        //(a = as[getProbe() & m]) == null 此时说明当前线程映射的槽为空，需要创建新的Cell并关联
        //!(uncontended =(r = function.applyAsLong(v = a.value, x)) == v || a.cas(v, r)) 此时说明该位置的槽
        //    不为空，但value更新失败，说明有竞争，需要换槽
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended =
              (r = function.applyAsLong(v = a.value, x)) == v ||
              a.cas(v, r)))
            longAccumulate(x, function, uncontended);
    }
}
```

### get

获取函数计算后的值，将所有的Cell通过函数计算后返回

同样，在多线程下有问题，计算不精确

```java
public long get() {
    Cell[] as = cells; Cell a;
    long result = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                result = function.applyAsLong(result, a.value);
        }
    }
    return result;
}
```

### reset

将Cell的value值都重置为初始值

```java
public void reset() {
    Cell[] as = cells; Cell a;
    base = identity;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                a.value = identity;
        }
    }
}
```

### getThenReset

将所有Cell的value值通过函数计算后，返回结果，并重置Cell的value值为初始值

同样，在多线程下有问题，计算不精确

```java
public long getThenReset() {
    Cell[] as = cells; Cell a;
    long result = base;
    base = identity;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null) {
                long v = a.value;
                a.value = identity;
                result = function.applyAsLong(result, v);
            }
        }
    }
    return result;
}
```

该类的其他方法类同LongAdder

## DoubleAdder、DoubleAccumulator

这两个类同样继承自Striped64，只是Double的表现形式，内部实现同LongAdder、LongAccumulator

