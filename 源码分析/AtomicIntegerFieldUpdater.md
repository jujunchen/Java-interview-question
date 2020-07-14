#  AtomicIntegerFieldUpdater源码分析

> 源码基于Java8

这个类是一个基于反射的使用的工具类，可以对指定类的指定的被volatile修饰的int型字段进行原子更新。

![image-20200708232748537](https://tva1.sinaimg.cn/large/007S8ZIlly1ggjzdqqif2j30ys0gkwfy.jpg)

## 创建方法

```java
//创建并返回一个Updater，
//要求给定字段在目标对象中必须存在，
//Class类型的参数用于检查反射类型和泛型之间是否匹配
//tclass 持有给定字段的目标对象的类类型
//fieldName 要更新的字段的名字，字符串类型
//泛型参数U代表了目标类型

//如果目标字段不是volatile描述的整形，会抛出IllegalArgumentException
//如果目标字段基于访问控制不允许访问，
//或者目标类型中不含有这个字段
//或者类型不匹配，可能会出现反射的异常，
//会在捕获之后抛出RuntimeException
//此处有个CallerSensitive注解，这个注解具体的内容在对应的地方说
//对于这个注解此处要明白它在这主要是配合Reflection.getCallerClass()
//作用是避免自己写Reflection.getCallerClass()的参数
//增加这个特性主要还是为了修复一个jdk中的利用双重反射的越权漏洞
@CallerSensitive
public static <U> AtomicIntegerFieldUpdater<U> newUpdater(Class<U> tclass,
                                                          String fieldName) {
    //这个实现类在下面，是个内部类
    //通过反射Reflection类实现对调用者的反射
    //Reflection属于sun包下的，
    //Oracle曾多次表示sun包不安全，
    //或者说要废弃，
    //所以自己使用的时候要谨慎
    //关于sun包的事情后面单独再说
    return new AtomicIntegerFieldUpdaterImpl<U>
        (tclass, fieldName, Reflection.getCallerClass());
}
    
//构造方法protected，
//用于保证子类可以调用一个无任何作用的构造方法
protected AtomicIntegerFieldUpdater() {
}
    
```

## 抽象方法

这些抽象方法有子类AtomicIntegerFieldUpdaterImpl实现

```java

//如果当前值==预期值，
//那么就原子的将当前Updater管理的给定的对象的字段设置为给定的更新值
//这个方法只对compareAndSet和set提供原子保证
//但是对于字段的其他修改不一定能够提供保证
//obj 是要进行设置的目标对象
//expect 是期待的目标对象
//update 要更新设置的值
//如果返回true说明设置成功了
//如果obj不是构造方法里给出的类型的实例，这个方法可能会抛出ClassCastException
//注： 按照package中对于false的定义，在期待值和当前值不同时，返回false
public abstract boolean compareAndSet(T obj, int expect, int update);

//方法的基本描述跟上面的一样
//但是略有不同的地方是
//这个方法可能会由于错误而失败，
//并且不提供顺序保证，
//因此很少用做compareAndSet的替换方法
//参数以及返回和异常与上一个方法一样
public abstract boolean weakCompareAndSet(T obj, int expect, int update);

//将这个Updater所管理的给定对象的字段设置为给定的新元素
//这个操作相对于compareAndSet提供了操作的保证
public abstract void set(T obj, int newValue);

//可以保证最终会把目标中的被Updater管理的元素更新为给定元素
public abstract void lazySet(T obj, int newValue);

//读取被当前Updater管理的字段的当前值
public abstract int get(T obj);

```

## 其他方法

```java
//原子的被当前Updater管理的目标对象中的字段更新为指定值
//同时返回原来的元素值
public int getAndSet(T obj, int newValue) {
    int prev;
    do {
        prev = get(obj);
    } while (!compareAndSet(obj, prev, newValue));
    return prev;
}


//用于将当前Updater管理的目标对象中的字段更新为+1后的值
//返回原来的值
public int getAndIncrement(T obj)  {
    int prev, next;
    do {
        prev = get(obj);
        next = prev + 1;
    } while (!compareAndSet(obj, prev, next));
    return prev;
}

//跟上一个方法相似，区别在于这个方法适用于-1
public int getAndDecrement(T obj) {
    int prev, next;
    do {
        prev = get(obj);
        next = prev - 1;
    } while (!compareAndSet(obj, prev, next));
    return prev;
}

//用于将当前Updater管理的目标对象中的字段更新为+delta后的值
//返回原来的值
public int getAndAdd(T obj, int delta) {
    int prev, next;
    do {
        prev = get(obj);
        next = prev + delta;
    } while (!compareAndSet(obj, prev, next));
    return prev;
}

//用于将当前Updater管理的目标对象中的字段更新为+1后的值
//返回新值
public int incrementAndGet(T obj) {
    int prev, next;
    do {
        prev = get(obj);
        next = prev + 1;
    } while (!compareAndSet(obj, prev, next));
    return next;
}

//跟上一个方法相似，区别在于这个方法-1
public int decrementAndGet(T obj) {
    int prev, next;
    do {
        prev = get(obj);
        next = prev - 1;
    } while (!compareAndSet(obj, prev, next));
    return next;
}

//用于将当前Updater管理的目标对象中的字段更新为+delta后的值
//返回新值
public int addAndGet(T obj, int delta) {
    int prev, next;
    do {
        prev = get(obj);
        next = prev + delta;
    } while (!compareAndSet(obj, prev, next));
    return next;
}
```

## Java8后增加的方法

```java
//返回旧元素
public final int getAndUpdate(T obj, IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get(obj);
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(obj, prev, next));
    return prev;
}

//返回新元素
public final int updateAndGet(T obj, IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get(obj);
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(obj, prev, next));
    return next;
}

//执行方法的时候传递的第一个参数是旧元素，第二个参数是x
//返回旧元素
public final int getAndAccumulate(T obj, int x,
                                  IntBinaryOperator accumulatorFunction) {
    int prev, next;
    do {
        prev = get(obj);
        next = accumulatorFunction.applyAsInt(prev, x);
    } while (!compareAndSet(obj, prev, next));
    return prev;
}

//跟上一个方法相似
//返回新元素
public final int accumulateAndGet(T obj, int x,
                                  IntBinaryOperator accumulatorFunction) {
    int prev, next;
    do {
        prev = get(obj);
        next = accumulatorFunction.applyAsInt(prev, x);
    } while (!compareAndSet(obj, prev, next));
    return next;
}
    
```

## AtomicIntegerFieldUpdaterImpl

该类为AtomicIntegerFieldUpdater的内部子类

```java
private static final class AtomicIntegerFieldUpdaterImpl<T>
    extends AtomicIntegerFieldUpdater<T> {
    //Unsafe实例
    private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
    //偏移量记录
    private final long offset;
    //如果字段受保护，子类构造这个更新器，否则和tclass一样
    //cclass记录的是子类类名
    private final Class<?> cclass;
    //含有目标字段的类的类类型
    //cclass tclass两个字段主要用于校验是否能够进行更新
    private final Class<T> tclass;

    //构造方法
    //tclass：含有目标字段的目标类类型
    //caller：调用者的类类型
    AtomicIntegerFieldUpdaterImpl(final Class<T> tclass,
                                  final String fieldName,
                                  final Class<?> caller) {
        final Field field;
        final int modifiers;
        try {
            //这里有个AccessController，这个属于Java的Security的一部分
            //包括AccessController，SecurityManager
            //这部分相关内容在对应的部分说
            //在这doPrivileged方法主要是用于越过权限检查
            //在特权块内的代码会越过特权检查，
            //这之后其他的类就可以越过检查执行这段代码
            field = AccessController.doPrivileged(
                //PrivilegedExceptionAction接口中含有一个run方法
                //当做一个doPrivileged调用时，
                //一个PrivilegedAction实现的实例被传递给它。
                //doPrivileged方法在使特权生效后，
                //从PrivilegedAction实现中调用run方法，
                //并返回run方法的返回值以作为doPrivileged的返回值
                //PrivilegedAction和PrivilegedExceptionAction区别在于
                //如果特权块中可能异常，就用PrivilegedExceptionActio
                //否则使用PrivilegedAction
                new PrivilegedExceptionAction<Field>() {
                    public Field run() throws NoSuchFieldException {
                        return tclass.getDeclaredField(fieldName);
                    }
                });
            //读取字段修饰符（此处返回的是int类型）
            //    PUBLIC: 1
            //    PRIVATE: 2
            //    PROTECTED: 4
            //    STATIC: 8
            //    FINAL: 16
            //    SYNCHRONIZED: 32
            //    VOLATILE: 64
            //    TRANSIENT: 128
            //    NATIVE: 256
            //    INTERFACE: 512
            //    ABSTRACT: 1024
            //    STRICT: 2048
            //返回值是各项修饰符的加和
            //自己做判断可能有点麻烦
            //可以通过Modifier.toString(int mod)方法来进行解析
            //或者通过Modifier里面的isXXX相关方法判断
            //Modifier.toString方法放到Modifier里面说
            modifiers = field.getModifiers();
            //这个方法也在sun包下面
            //这个方法主要用于校验给定的调用者（第一个参数）
            //第二个参数是目标的类类型
            //第三个参数在静态的情况系可以是null
            //第四个参数是访问修饰
            //判断是否可以访问目标对象中构造方法、字段、普通方法等        
            //如果没有访问权限会抛出IllegalAccessException
            sun.reflect.misc.ReflectUtil.ensureMemberAccess(
                caller, tclass, null, modifiers);
            //目标类的类加载器  
            ClassLoader cl = tclass.getClassLoader();
            //调用者的类加载器
            ClassLoader ccl = caller.getClassLoader();
            //这个判断条件要求满足目标类和调用者不能使同一个类加载器
            //调用者不能是根加载器
            //目标类加载器是根加载器或者调用者类加载器和目标类加载器的关系不满足isAncestor（没有委托链关系）
            if ((ccl != null) && (ccl != cl) &&
                ((cl == null) || !isAncestor(cl, ccl))) {
                //sun包下面
                //校验目标类的包的可访问性
                sun.reflect.misc.ReflectUtil.checkPackageAccess(tclass);
            }
        } catch (PrivilegedActionException pae) {
            throw new RuntimeException(pae.getException());
        } catch (Exception ex) {
            throw new RuntimeException(ex);
        }

        //类型校验
        if (field.getType() != int.class)
            throw new IllegalArgumentException("Must be integer type");

        //判断是否有volatile修饰
        if (!Modifier.isVolatile(modifiers))
            throw new IllegalArgumentException("Must be volatile type");

        //访问protected字段需要是同包下或者有父子关系的情况
        //这个判断条件指出，如果是
        //在字段为Protected，目标类是调用类的父类，调用类和目标类不是同一个包下时
        //cclass存储调用者的类
        //上面条件任何一个不满足都指向tclass
        //包括不是保护域，没有父子关系，在同一个包下面
        this.cclass = (Modifier.isProtected(modifiers) &&
                       tclass.isAssignableFrom(caller) &&
                       !isSamePackage(tclass, caller))
            ? caller : tclass;

        //记录目标class           
        this.tclass = tclass;
        //记录偏移量
        this.offset = U.objectFieldOffset(field);
    }


    //如果第二个加载器可以再第一个加载器的委托链中找到返回True
    //等价于调用 first.isAncestor(second).
    private static boolean isAncestor(ClassLoader first, ClassLoader second) {
        ClassLoader acl = first;
        do {
            acl = acl.getParent();
            if (second == acl) {
                return true;
            }
        } while (acl != null);
        return false;
    }

    //如果两个类具有相同的类加载器和包限定符，则返回true
    private static boolean isSamePackage(Class<?> class1, Class<?> class2) {
        return class1.getClassLoader() == class2.getClassLoader()
            //Objects是一个工具类，具体的放到对应的类讲
            && Objects.equals(getPackageName(class1), getPackageName(class2));
    }

    //获取给定类的包名
    private static String getPackageName(Class<?> cls) {
        //这个方法返回虚拟机中Class对象的表示
        //例如String[]类型的表示为: [Ljava.lang.String
        String cn = cls.getName();
        //定位到最后一个点的位置
        int dot = cn.lastIndexOf('.');
        //返回类名的截取
        return (dot != -1) ? cn.substring(0, dot) : "";
    }

    //检查目标参数是否为cclass的实例。失败时，抛出异常原因
    private final void accessCheck(T obj) {
        if (!cclass.isInstance(obj))
            throwAccessCheckException(obj);
    }

    //如果由于受保护的访问而导致访问检查失败，
    //则引发访问异常，
    //否则将引发ClassCastException。
    //实际上当cclass和tclass如果相等，会抛出ClassCastException
    //也就是说在上面初始化过程中出现了对于受保护字段的访问检查失败
    private final void throwAccessCheckException(T obj) {
        if (cclass == tclass)
            throw new ClassCastException();
        else
            throw new RuntimeException(
            new IllegalAccessException(
                "Class " +
                cclass.getName() +
                " can not access a protected member of class " +
                tclass.getName() +
                " using an instance of " +
                obj.getClass().getName()));
    }

    //**************************************************************

    //下面终于到了这个类应该做的正经事了
    //由于上面有过叙述，就不进行冗余的叙述了
    //每个方法都是先进行对可访问性的校验

    //CAS操作
    public final boolean compareAndSet(T obj, int expect, int update) {
        accessCheck(obj);
        return U.compareAndSwapInt(obj, offset, expect, update);
    }

    //弱CAS操作
    public final boolean weakCompareAndSet(T obj, int expect, int update) {
        accessCheck(obj);
        return U.compareAndSwapInt(obj, offset, expect, update);
    }

    //set
    public final void set(T obj, int newValue) {
        accessCheck(obj);
        U.putIntVolatile(obj, offset, newValue);
    }

    //lazySet
    public final void lazySet(T obj, int newValue) {
        accessCheck(obj);
        U.putOrderedInt(obj, offset, newValue);
    }

    //读取
    public final int get(T obj) {
        accessCheck(obj);
        return U.getIntVolatile(obj, offset);
    }

    //设置并返回原来的值
    public final int getAndSet(T obj, int newValue) {
        accessCheck(obj);
        return U.getAndSetInt(obj, offset, newValue);
    }

    //增加delta后返回原来的元素
    public final int getAndAdd(T obj, int delta) {
        accessCheck(obj);
        return U.getAndAddInt(obj, offset, delta);
    }

    //+1然后返回原来的元素，实际调用的是getAndAdd
    public final int getAndIncrement(T obj) {
        return getAndAdd(obj, 1);
    }

    //-1然后返回原来的元素，实际调用的是getAndAdd
    public final int getAndDecrement(T obj) {
        return getAndAdd(obj, -1);
    }

    //+1然后返回新元素，实际调用的是getAndAdd
    public final int incrementAndGet(T obj) {
        return getAndAdd(obj, 1) + 1;
    }

    //-1然后返回新元素，实际调用的是getAndAdd
    public final int decrementAndGet(T obj) {
        return getAndAdd(obj, -1) - 1;
    }

    //加delta然后返回新元素，实际调用的是getAndAdd
    public final int addAndGet(T obj, int delta) {
        return getAndAdd(obj, delta) + delta;
    }

}
```