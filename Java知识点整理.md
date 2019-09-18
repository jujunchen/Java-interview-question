# Java 面试知识点整理

- [1.1 Java基础](#11-Java基础)
  * [自定义注解](#自定义注解)
- [1.2 常见的集合](#12-常见的集合)
  * [HashMap 1.7 和 1.8 的区别](#HashMap-17-和-18-的区别)
- [1.3 并发](#13-并发)
  * [Synchronized原理](#Synchronized原理)
  * [HashMap在并发下会产生什么问题？有什么替代方案?(HashTable, ConcurrentHashMap)。它们两者的实现原理。](#HashMap在并发下会产生什么问题？有什么替代方案HashTable-ConcurrentHashMap。它们两者的实现原理。)
  * [ThreadLocal 子类及原理, OOM产生原因及防治](#ThreadLocal-子类及原理-OOM产生原因及防治)
  * [Java8新增的原子操作类](#Java8新增的原子操作类)
  * [ConcurrentLinkedQueue、LinkedBlockingQueue、ArrayBlockingQueue、PriorityBlockingQueue、DelayQueue、SynchronousQueue、LinkedBlockingDeque、LinkedTransferQueue](#ConcurrentLinkedQueue、LinkedBlockingQueue、ArrayBlockingQueue、PriorityBlockingQueue、DelayQueue、SynchronousQueue、LinkedBlockingDeque、LinkedTransferQueue)
  * [ThreadPoolExecutor构造函数有哪几个参数，实现原理，创建线程池的方式](#ThreadPoolExecutor构造函数有哪几个参数，实现原理，创建线程池的方式)
  * [为什么建议在不用线程池的时候，关闭线程池，线程池的作用就是复用线程的](#为什么建议在不用线程池的时候，关闭线程池，线程池的作用就是复用线程的)
  * [Timer 和 ScheduledThreadPoolExecutor 区别](#Timer-和-ScheduledThreadPoolExecutor-区别)
  * [CountDownLatch原理、CyclicBarrier原理，两者区别](#CountDownLatch原理、CyclicBarrier原理，两者区别)
  * [Semaphore 原理](#Semaphore-原理)
  * [Exchanger 原理](#Exchanger-原理)
  * [如何实现一个生产者与消费者模型](#如何实现一个生产者与消费者模型)
- [1.4 锁](#14-锁)
  * [自旋锁、自适应自旋、锁消除、锁粗化、轻量级锁、偏向锁](#自旋锁、自适应自旋、锁消除、锁粗化、轻量级锁、偏向锁)
  * [公平锁、非公平锁](#公平锁、非公平锁)
  * [AbstractQueuedSynchronizer的作用](#AbstractQueuedSynchronizer的作用)
  * [JDK8新增的锁](#JDK8新增的锁)
- [1.5 JVM](#15-JVM)
  * [JVM运行时内存区域划分](#JVM运行时内存区域划分)
  * [OOM,及SOE的示例、原因，排查方法](#OOM及SOE的示例、原因，排查方法)
  * [如何判断对象可以回收或存活](#如何判断对象可以回收或存活)
  * [常见的GC算法](#常见的GC算法)
  * [常见的JVM性能监测分析工具](#常见的JVM性能监测分析工具)
  * [JVM优化](#JVM优化)
  * [什么时候会触发FullGC](#什么时候会触发FullGC)
  * [类加载器有几种](#类加载器有几种)
  * [什么是双亲委派模型？双亲委派模型的破坏](#什么是双亲委派模型？双亲委派模型的破坏)
  * [类的生命周期](#类的生命周期)
  * [强引用、软引用、弱引用、虚引用](#强引用、软引用、弱引用、虚引用)
  * [CMS和G1的区别](#CMS和G1的区别)
  * [volatile的特点](#volatile的特点)
- [1.6 设计模式](#16-设计模式)
  * [常见的设计模式](#常见的设计模式)
  * [单例模式代码](#单例模式代码)
- [1.7 数据结构](#17-数据结构)
- [1.8 反射、IO](#18-反射、IO)
  * [反射](#反射)
  * [BIO、NIO区别](#BIO、NIO区别)
  * [Java NIO的原理](#Java-NIO的原理)
- [2.1数据库](#21数据库)
  * [存储引擎的 InnoDB与MyISAM区别，优缺点，使用场景](#存储引擎的-InnoDB与MyISAM区别，优缺点，使用场景)
  * [建立索引的原则](#建立索引的原则)
- [2.2 Redis](#22-Redis)
  * [Redis 为什么是单线程? 为什么单线程还能这么快？](#Redis-为什么是单线程-为什么单线程还能这么快？)
  * [Redis 使用场景](#Redis-使用场景)
  * [Redis 持久化机制](#Redis-持久化机制)
  * [Redis 数据类型](#Redis-数据类型)
  * [Pipeline](#Pipeline)
  * [事务](#事务)
  * [Lua](#Lua)
  * [慢查询分析](#慢查询分析)
  * [Redis键过期删除策略](#Redis键过期删除策略)
  * [Redis高可用方案](#Redis高可用方案)
  * [缓存问题](#缓存问题)
- [2.3 消息队列](#23-消息队列)
  * [消息的幂等性处理思路](#消息的幂等性处理思路)
  * [消息队列如何保证高可用](#消息队列如何保证高可用)
  * [如何保证消息可靠性](#如何保证消息可靠性)
  * [消息积压问题](#消息积压问题)
  * [kafka的分区策略](#kafka的分区策略)
- [2.4 Spring知识点](#24-Spring知识点)
  * [BeanFactory 与ApplicationContext 是干什么的，两者的区别](#BeanFactory-与ApplicationContext-是干什么的，两者的区别)
  * [Spring IOC容器如何实现](#Spring-IOC容器如何实现)
  * [Spring如何解决循环依赖问题](#Spring如何解决循环依赖问题)
  * [Spring Bean 生命周期](#Spring-Bean-生命周期)
  * [Spring Bean的作用域，默认是哪个？](#Spring-Bean的作用域，默认是哪个？)
  * [AOP两种代理方式](#AOP两种代理方式)
  * [AOP 切点函数](#AOP-切点函数)
  * [六种增强类型](#六种增强类型)
  * [Spring AOP实现原理](#Spring-AOP实现原理)
  * [Spring MVC运行流程](#Spring-MVC运行流程)
  * [Spring MVC 启动流程](#Spring-MVC-启动流程)
  * [Spring 事务实现方式、事务的传播机制、默认的事务类别](#Spring-事务实现方式、事务的传播机制、默认的事务类别)
  * [Spring 事务底层原理](#Spring-事务底层原理)
  * [Spring事务失效（事务嵌套), JDK动态代理给Spring事务埋下的坑](#Spring事务失效（事务嵌套-JDK动态代理给Spring事务埋下的坑)
  * [Spring 单例实现原理](#Spring-单例实现原理)
  * [Spring 中有哪些不同类型的事件](#Spring-中有哪些不同类型的事件)
- [2.5 分布式、微服务知识点](#25-分布式、微服务知识点)
  * [领域驱动有了解吗？什么是领域驱动模型？](#领域驱动有了解吗？什么是领域驱动模型？)
  * [JWT有了解吗，什么是JWT](#JWT有了解吗，什么是JWT)
  * [说说如何设计一个良好的 API](#说说如何设计一个良好的-API)
  * [如何保证接口的幂等性](#如何保证接口的幂等性)
  * [说说 CAP 定理、BASE 理论](#说说-CAP-定理、BASE-理论)
  * [说说最终一致性的实现方案](#说说最终一致性的实现方案)
  * [微服务与 SOA 的区别](#微服务与-SOA-的区别)
  * [如何拆分服务、水平分割、垂直分割](#如何拆分服务、水平分割、垂直分割)
  * [如何应对微服务的链式调用异常](#如何应对微服务的链式调用异常)
  * [如何快速追踪与定位问题](#如何快速追踪与定位问题)
  * [如何保证微服务的安全、认证](#如何保证微服务的安全、认证)
- [2.6 Dubbo](#26-Dubbo)
  * [Dubbo SPI的理解](#Dubbo-SPI的理解)
  * [Dubbo 基本原理、执行流程](#Dubbo-基本原理、执行流程)

#### 1.1 Java基础

##### 自定义注解

1. 声明注解的保留期限类型

    @Retention(RetentionPolicy.RUNTIME)表示该注解可以在运行期保留

    保留期限类型：java.lang.annotation.Retention

    SOURCE: 注解信息仅保留在目标类源代码文件中，对应的字节码文件不会保留

    CLASS: 注解信息存在于源代码、字节码文件中，但运行期JVM不能获得该注解信息

    RUNTIME: 注解信息存在于源代码、字节码文件、运行期JVM中，能够通过反射机制获取注解类信息

2. 声明注解的可以使用的目标类型

    @Target(ElementType.METHOD) 表示这个注解只能在方法上使用

    目标类型：java.lang.annotation.ElementType

    TYPE: 类、接口、注解类、Enum

    FIELD: 类成员变量或常量

    METHOD: 方法

    PARAMETER: 参数

    CONSTRUCTOR: 构造器

    LOCAL_VARIABLE: 局部变量

    ANNOTATION_TYPE: 注解

    PACKAGE: 包

3. 使用@interface 修饰类

4. 声明注解成员

    成员无入参、不能抛出异常；

    可以通过default成员指定默认值

    成员类型只能使用String、Class、enums、注解类型，及上述类型的数组类型。如ForumService value()是非法的

    如果注解只有一个成员，则成员名必须取名为value()，再使用时可以忽略成员名和赋值号，如果注解类拥有多个成员时，

    对value成员赋值，可以省略value和赋值号，如果是多个成员赋值，必须使用赋值号

#### 1.2 常见的集合

##### HashMap 1.7 和 1.8 的区别

> 1.7，在发生hash冲突的时候，数据结构只有链表；  
> 1.8，数据结构有链表和红黑树，使用红黑树是为了能够提高查询效率。在链表长度达到7的，并且hash tab[]数组长度大于等于64时，将链表转换成红黑树，如果数组长度小于64，只是对数组进行扩容  
> https://blog.csdn.net/qq_21251983/article/details/90056067

#### 1.3 并发

##### Synchronized原理
> Synchronized可以修饰普通方法、同步方法块、静态方法；普通方法锁是当前实例对象，静态方法锁是当前类的Class对象，同步方法块锁是Synchonized配置的对象；用的锁是存在对象头里的,根据mark word的锁状态来判断锁，如果锁只被同一个线程持有使用的是偏向锁，不同线程互相交替持有锁使用轻量级锁，多线程竞争使用重量级锁。锁会按偏向锁->轻量级锁->重量级锁 升级，称为锁膨胀  
> https://github.com/farmerjohngit/myblog/issues/12

##### HashMap在并发下会产生什么问题？有什么替代方案?(HashTable, ConcurrentHashMap)。它们两者的实现原理。
>HashMap并发下产生问题：死循环，Map.size()不准确，数据丢失  
>https://blog.csdn.net/dianzijinglin/article/details/80839115  
>HashTable: 通过synchronized来修饰，效率低，多线程put的时候，只能有一个线程成功，其他线程都处于阻塞状态    
>ConcurrentHashMap：1.7 采用锁分段技术提高并发访问率  
>1.8 数据依旧是分段存储，但锁采用了synchronized，内部采用Node数组+链表+红黑树的结构存储，当单个链表存储数量达到红黑树阈值8时（此时链表已有元素7），存储结构转换为红黑树来存储    
>https://www.cnblogs.com/banjinbaijiu/p/9147434.html


##### ThreadLocal 子类及原理, OOM产生原因及防治
- InheritableThreadLocal 

     继承了ThreadLocal，并重写childValue、getMap、createMap，对该类的操作实际上是对线程ThreadLocalMap的操作

    子线程能够读取父线程数据，实际原因是新建子线程的时候，会从父线程copy数据

- OOM原因及防治  
 ThreadLocal只是一个工具类，具体存放变量的是线程的threadLocals变量，threadLocals是一个ThreadLocalMap类型的变量，内部是一个Entry数组，Entry继承自WeakReference,Entry内部的value用来存放通过ThreadLocal的set方法传递的值,key是ThreadLocal的弱引用，key虽然会被GC回收，但value不能被回收，这时候ThreadLocalMap中会存在key为null，value不为null的entry项，如果时间长了就会存在大量无用对象，造成OOM。虽然set,get也提供了一些对Entry项清理的时机，但不及时，所以在使用完毕后需要及时调用remove


##### Java8新增的原子操作类
> LongAdder 由于AtomicLong通过CAS提供非阻塞的原子性操作，性能已经很好，在高并发下大量线程竞争更新同一个原子量，但只有一个线程能够更新成功，这就造成大量的CPU资源浪费。  
> LongAdder 通过让多个线程去竞争多个Cell资源，来解决，再很高的并发情况下,线程操作的是Cell数组，并不是base，在cell元素不足时进行2倍扩容，在高并发下性能高于AtomicLong
> LongAccumulator 提供了一个函数式接口，可以自定义运算规则

##### ConcurrentLinkedQueue、LinkedBlockingQueue、ArrayBlockingQueue、PriorityBlockingQueue、DelayQueue、SynchronousQueue、LinkedBlockingDeque、LinkedTransferQueue
> ConcurrentLinkedQueue: 无界非阻塞队列，底层使用单向链表实现，对于出队和入队使用CAS来实现线程安全  
> LinkedBlockingQueue: 有界阻塞队列，使用单向链表实现，通过ReentrantLock实现线程安全  
> ArrayBlockingQueue: 有界数组方式实现的阻塞队列  
> PriorityBlockingQueue: 带优先级的无界阻塞队列，内部使用平衡二叉树堆实现，遍历保证有序需要自定排序  
> DelayQueue: 无界阻塞延迟队列，队列中的每个元素都有个过期时间，当从队列获取元素时，只有过期元素才会出队列，队列头元素时最快要过期的元素  
> SynchronousQueue: 任何一个写需要等待一个读的操作，读操作也必须等待一个写操作，相当于数据交换  https://www.cnblogs.com/dwlsxj/p/Thread.html  
> LinkedBlockingDeque: 由链表组成的双向阻塞队列，可以从队列的两端插入和移除元素  
> LinkedTransferQueue: 由链表组成的无界阻塞队列，多了tryTransfer 和 transfer方法。transfer方法，能够把生产者元素立刻传输给消费者，如果没有消费者在等待，那就会放入队列的tail节点，并阻塞等待元素被消费了返回，可以使用带超时的方法。tryTransfer方法，会在没有消费者等待接收元素的时候马上返回false

##### ThreadPoolExecutor构造函数有哪几个参数，实现原理，创建线程池的方式
> 构造参数：  
> corePoolSize: 线程池核心线程个数  
> workQueue: 用于保存等待执行任务的阻塞队列  
> maximunPoolSize: 线程池最大线程数量  
> ThreadFactory: 创建线程的工厂  
> RejectedExecutionHandler: 队列满，并且线程达到最大线程数量的时候，对新任务的处理策略，AbortPolicy（抛出异常）、CallerRunsPolicy(使用调用者所在线程执行)、DiscardOldestPolicy(调用poll丢弃一个任务，执行当前任务)、DiscardPolicy(默默丢弃、不抛异常)  
> keeyAliveTime: 空闲线程存活时间  
> TimeUnit: 存活时间单位  

>原理：  
>线程池主要是解决两个问题：一个是当执行大量异步任务时能够提供较好的性能，能复用线程处理任务；二是能够对线程池进行资源限制和管理。一个任务提交的线程池，首先会判断核心线程池是否已满，未满就会创建worker线程执行任务，已满判断阻塞队列是否已满，阻塞队列未满加入阻塞队列，已满就判断线程池线程数量是否已经达到最大值，没有就新建线程执行任务，达到最大值的话执行拒绝策略。拒绝策略有：直接抛出异常、使用调用者所在线程执行、丢弃一个旧任务，执行当前任务、直接丢弃什么都不做。  

>创建线程池的方式：直接new ThreadPoolExecutor 或者通过Executors工具类创建 

##### 为什么建议在不用线程池的时候，关闭线程池，线程池的作用就是复用线程的
> 线程池的作用确实是为了减少频繁创建线程，使线程达到复用。但如果在不用线程池的情况下，线程池中的核心线程会一直存在，浪费资源，所以建议在不用的情况下调用shutdown方法关闭线程池。在需要的时候再调用创建线程池。

##### Timer 和 ScheduledThreadPoolExecutor 区别
> Timer是固定的多线程生产者单线程消费，如果其中一个任务报错，其他任务也会失效;但后者是可以配置的，既可以是多线程生产单线程消费也可以是多线程生产多线程消费

##### CountDownLatch原理、CyclicBarrier原理，两者区别
> CountDownLatch: 使用AQS实现，通过AQS的状态变量state来作为计数器值,当多个线程调用countdown方法时实际是原子性递减AQS的状态值，当线程调用await方法后当前线程会被放入AQS阻塞队列等待计数器为0再返回  
> CyclicBarrier:   
> 区别：CountDownLatch计数器是一次性的，变为0后就起不到线程同步的作用了。而CyclicBarrier在计数器变为0后重新开始，并且能在所有线程到达屏障点后统一执行某个任务，再执行完后继续执行子线程


##### Semaphore 原理
> Semaphore 可以用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源,使用AQS实现，AQS的状态变量state做为许可证数量，每次通过acquire()/tryAcquire()，许可证数量通过CAS原子性递减，调用release()释放许可证，原子性递增，只要有许可证就可以重复使用

##### Exchanger 原理
> 用于进行线程间的数据交换，它提供一个同步点，在这个同步点两个线程可以交换批次的数据。如果第一个线程先执行exchange()方法，它会一直等待第二个线程也执行exchange方法，当都达到同步点时，这两个线程可以交换数据。
> https://blog.csdn.net/coslay/article/details/45242581

##### 如何实现一个生产者与消费者模型

>一共5种方法  
>1. 同步对象的 wait() / notify() 方法  
>2. ReetrantLock Condition 的 await() / signal()方法
>3. BlockingQueue阻塞队列 put() 和take方法
>4. Semaphore 基于计数的信号量
>5. PipedInputStream / PipedOutputStream 管道输入输出流
>https://blog.csdn.net/ldx19980108/article/details/81707751


#### 1.4 锁

##### 自旋锁、自适应自旋、锁消除、锁粗化、轻量级锁、偏向锁
>自旋锁：开启线程执行一个忙循环，直到需要更新的值为期待值为止  
>自适应自旋：自旋时间不再固定，由前一次在同一个锁上的自旋时间及锁的拥有者状态来决定，比如在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也很有可能再次成功，进而它将自旋等待更长时间，如果很少成功获得过锁，那很可能会忽略掉自旋过程，以避免CPU资源浪费。  
>锁消除：JIT在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除  
>锁粗化：如果虚拟机探测到有一串零碎的操作都对同一个对象加锁，将会把加锁同步的范围扩展到整个序列的外部  
>轻量级锁：加锁是通过同步对象的对象头进行操作的，首先会在当前线程的栈帧中建立一个名为锁记录的空间，存储锁对象目前的Mark Word拷贝，会加Displaced前缀，然后通过CAS尝试将对象的Mark Word更新为指向Lock Record的指针，如果成功，就获得了该对象的锁，如果失败，会检查Mark Word是否指向当前线程的栈帧，如果是就说明已经获得了锁，如果没有就说明有其他线抢占，轻量锁就会膨胀成重量级锁；解锁也是通过CAS来操作，就是将Mark Word 替换为原来的值    
>偏向锁：锁偏向于第一个获得它的线程，如果在接下来的执行过程中，该锁没有被其他的线程获取，则持有偏向锁的线程将永远不需要再同步。-XX:+UseBiasedLocking 开启偏向锁

##### 公平锁、非公平锁
> 公平锁：根据线程请求锁的顺序来获取锁  
> 非公平锁：抢占式获取锁

##### AbstractQueuedSynchronizer的作用
> 抽象同步队列简称AQS,是实现同步器的基础组件，并发包中的锁都是基于其实现的

##### JDK8新增的锁
> StampedLock 提供了三种模式的读写控制，当调用获取锁的系列函数时，会返回一个long型变量，支持在一定条件下三种模式的相互转换  
> 写锁writeLock: 一个排它锁或者独占锁，并且写锁不可重入  
> 悲观读锁readLock: 共享锁，在没有线程独占获取写锁的情况下，多个线程可以同时获取该锁，如果已经有其他线程持有写锁，则其他线程请求读锁会被阻塞  
> 乐观读锁tryOptimisticRead: 在操作数据前并没有通过CAS设置锁的状态，仅通过位运算测试

#### 1.5 JVM
##### JVM运行时内存区域划分
> 线程独享区域：程序计数器，本地方法栈，虚拟机栈  
> 线程共享区域：元空间(<=1.7方法区), 堆  
>
> 程序计数器：线程私有，是一块较小的内存空间，可以看做是当前线程执行的字节码指示器，也是唯一的没有定义OOM的区块  
> 本地方法栈： 用于执行Native 方法时使用  
> 虚拟机栈：用于存储局部变量，操作数栈，动态链接，方法出口等信息  
> 元空间(方法区)：存储已被虚拟机加载的类信息，常量，静态变量，即时编译器编译后的代码等数据  
> 堆：存储对象实例

##### OOM,及SOE的示例、原因，排查方法
```java
//OOM -Xmx20m -Xms20m -XX:+HeapDumpOnOutOfMemoryError
public class OOMTest {
    public static void main(String[] args) {
        List<Object> objList = new ArrayList();
        while(true) {
            objList.add(new Object());
        }
    }
}

//SOE栈异常 -Xss125k
public class SOETest() {
    static int count = 0;
    public static void main(String[] args) {
        try {
            stackMethod();
        } catch(Error err) {
            err.printStackTrace();
            System.out.println("执行count=" + count);
        }
    }
    private static void stackMethod() {
        count ++;
        stackMethod();
    }
}
```
> OOM排查：如果能看到日志，可以从打印的日志中获取到发送异常的代码行，再去代码中查找具体哪块的代码有问题。如果没有记录日志，通过设置的 -XX:+HeapDumpOnOutOfMemoryError 在发生OOM的时候生成.hprof文件，再导入JProfiler能够看到是由于哪个对象造成的OOM，再通过这个对象去代码中寻找  
> SOE排查：栈的深度一般为1000-2000深度,超过了深度或者超过了栈大小就会导致SOE，通过打印的日志定位错误代码位置，检测是否有无限递归，发生了死循环等情况，修改代码

##### 如何判断对象可以回收或存活
> 判断是否可以回收，或者存活主要是看：  
> 堆中是否存在该实例  
> 加载该类的classloader是否已经被回收  
> 该类的java.lang.Class对象在任何地方没有被引用，也就是不能够通过反射方法获取该类信息

##### 常见的GC算法
> 1. 标记-清除算法：先标记出需要回收的对象，再清除这些被标记了的对象，缺点是：标记清除过程效率不高；会产生内存碎片  
> 2. 复制算法：将内存划分成同等大小两块，只使用其中一块内存，当这一块的内存快用完后，将已存活的对象复制到另外一块内存，再对已使用的内存空间进行一次清理
> 3. 标记-整理算法：标记出已存活的对象，将对象移动到内存一端，再对端以外的内存进行清理回收
> 4. 分代收集算法：年轻代使用==复制算法==,永久代使用 ==标记-清理或者标记-整理算法==

##### 常见的JVM性能监测分析工具
> 1. jps  
> 能够查看正在运行的虚拟机进程，并显示虚拟机的执行主类及进程ID
> 2. jstat [option vmid [interval[s|ms] [count]] ]
> 可以显示本地或者远程虚拟机中的类装载、内存、垃圾收集、JIT编译等运行数据
> 3. jinfo  
> 实时查看和调整虚拟机的各项参数
> 4. jmap 
> 生成堆转储快照
> 5. jstack  
> 用于生成虚拟机当前时刻的线程快照

##### JVM优化
> 1. 响应时间优先：年轻代设的大些，直到接近系统的最低响应时间限制。年轻代设大，可以减少到达年老代的对象。对于永久代的设置需要参考：永久代并发收集的次数、年轻代和永久代回收时间比例，调整达到一个合适的值
> 2. 吞吐量优先：年轻设的大些，永久代较小

##### 什么时候会触发FullGC
> 永久代空间不足  
> 手动调用触发gc

##### 类加载器有几种
> 1. Bootstrap ClassLoader  
> 负责加载JDK自带的rt.jar包中的类文件，它是所有类加载器的父加载器，Bootstrap ClassLoader没有任何父类加载器。
> 2. Extension ClassLoader负责加载Java的扩展类库，也就是从jre/lib/ext目录下或者java.ext.dirs系统属性指定的目录下加载类。  
> 3. System ClassLoader负责从classpath环境变量中加载类文件，classpath环境变量通常由"-classpath" 或 "-cp" 命令行选项来定义，或是由 jar中 Manifest文件的classpath属性指定，System ClassLoader是Extension ClassLoader的子加载器
> 4. 自定义加载器

##### 什么是双亲委派模型？双亲委派模型的破坏
> 一个类在加载的时候，首先会将加载请求委派给父加载器，只有当父加载器反馈无法加载完成这个请求时，子加载器才会尝试自己加载  
> 双亲委派模型的破坏指的是不按照双亲委派模型来加载类，比如JNDI，它的代码由启动类加载器加载,但JDNI需要调用部署在ClassPath的JNDI接口，但启动类加载器是不知道这些代码的，所以就有了线程上下文类加载器(Thread Context ClassLoader)，可以通过java.lang.Thread类setContextClassLoader设置类加载器，通过这个加载器父加载器就可以请求子类加载器完成类加载的动作。

##### 类的生命周期
> 类的生命周期一个有7个阶段：加载、验证、准备、解析、初始化、使用、卸载

>加载:  
>加载阶段，虚拟机需要完成以下3件事
>
>1. 通过类的全限定名来获取此类的二进制字节流
>2. 将字节流所代表的静态存储结构转化为方法区的运行时数据结构
>3. 在内存中生成代表这个类的java.lang.Class对象，作为方法区这个类的各种数据访问入口  

>验证：分4个验证
>1. 文件格式验证，验证是否符合Class文件格式的规范，并且能被当前版本的虚拟机处理
>2. 元数据验证，对字节码描述的信息进行语义分析，以保证其描述的信息符合Java语言规范的要求
>3. 字节码验证,通过数据流和控制流分析，确定程序语义是否合法、符合逻辑
>4. 符合引用验证，是对类自身以外的信息进行匹配性校验(常量池中各种符合引用)

>准备：正式为类变量分配内存并设置初始值的阶段，这里设置初始值是数据类型的默认值

>解析：虚拟机将常量池中的符号引用替换为直接引用的过程

>初始化：执行类构造器的过程

##### 强引用、软引用、弱引用、虚引用
> 1. 强引用：大部分使用都是强引用，当内存不足时，会OOM，程序异常终止，也不会随意回收具有强引用的对象  
> 2. 软引用：内存足够时，不会清除对象，在内存不足时就会回收这些对象  
> 3. 弱引用：弱引用的对象，在发生GC的时候，就会被回收
> 4. 虚引用：虚引用主要用来跟踪垃圾回收的活动，虚引用必须和引用队列联合使用。

-编译器会对指令做哪些优化？
>编译器优化分编译期和运行期  
>编译期：  
>1.标注检查，检查变量使用前是否已被声明、变量与赋值之间的数据类型是否能够匹配，对常量进行折叠  
>2.数据及控制流分析，检查诸如程序局部变量在使用前是否有赋值、是否所有的受检异常都被正确处理等问题  
>3.将语法糖还原为基础的语法结构  
>4.生成字节码  
>运行期：
>即时编译器JIT会把运行频繁的代码编译成与本地平台相关的机器码，并进行各种层次的优化   
>Client Compiler: 会进行局部性的优化，分三阶段：第一阶段，一个平台独立的前端将字节码构造成一种高级中间代码表示HIR，HIR使用静态单分配（SSA）的形式来代表代码值，在字节码上做方法内联，常量传播等基础优化；第二阶段，从HIR中产生低级中间代码，在这之前会做空值检查消除，范围检查消除等。第三阶段，在LIR上分配寄存器，并在LIR上做窥孔优化，最后产生机器码  
>Server Compiler: 会执行无用代码消除、循环展开、循环表达式外提、消除公共子表达式、常量传播、基本块重排序、范围检测消除、空值检查消除，另外还能根据解释器或Client Compiler提供的性能监控信息，进行一些不稳定的激进优化，比如守护内联、分支频率预测等

>几种经典的优化技术：
>1. 公共子表达式消除  
>如果一个表达式E已经计算过，并且从先前的计算到现在E中所有变量的值都没有发生变化，那么E的这次出现就成为公共子表达式
>2. 数组范围检查消除  
>编译期就判断数组是否在合理的范围内，如果在，那就可以在循环中把数组的上下界检查消除。另外还有隐式异常处理，虚拟机会注册一个Segment Fault信号的异常处理器，但如果代码经常为空，消耗时间比判空慢，但虚拟机会根据运行期收集到的信息选择使用判空还是隐式异常处理
>3. 方法内联  
>一可以给“公共子表达式消除”等其他优化技术提供基础。虚方法即多态情况下，如果要确定是否能内联，虚拟机需要向“类型继承关系分析”器查询，如果只有一个版本，那可以进行内联，但还会留一个“逃生门”，这种内联称为“守护内联”，如果继承关系发生变化，虚拟机会通过“逃生门”退回到解释状态执行，或重新编译。  
>如果是多版本方法，虚拟机会通过“内联缓存”，在第一次调用的时候将目标方法版本缓存起来，下次调用的时候检查版本是否一致，如果不一致就会取消内联，查找虚拟方法表进行方法分派
>4. 逃逸分析  
>分析对象动态作用域，对象是否作为调用参数传递到其他方法中（方法逃逸），是否有被其他线程访问(线程逃逸)。如果没有以上情况，虚拟机会做一些高效优化：栈上分配、同步消除（去掉同步措施）、标量替换（将对象成员变量恢复到原始类型）

##### CMS和G1的区别
>CMS 基于“标记-清除”算法，一共分初始标记、并发标记、重新标记、并发清除4个阶段，在初始标记、重新标记阶段需要STW，并且CMS收集器占用CPU资源较多，无法处理浮动垃圾（-XX:CMSInitiatingOccupancyFraction 调整老年代占用多少触发回收；-XX:+UseCMSCompactAtFullCollection 默认开启，在即将触发FullGC前对内存碎片进行整理；-XX:CMSFullGCsBeforeCompaction设置执行多少次不压缩的FullGC后，来一次带压缩的的碎片整理）  
>G1 可以跟用户程序并发进行垃圾收集；分代收集，将堆划分成多个大小相等的独立Region区域；空间整合，默认就会进行内存整理；可预测的停顿，G1跟踪各个Region的回收获得的空间大小和回收所需要的经验值，维护一个优先列表；

##### volatile的特点
> 修改可见性，通过volatile修饰的变量，线程对变量的修改对另一个线程即时可见，因此另外一个线程在使用前会去判断该变量是否有修改；  
> 禁止指令重排序  
> ==long 和double的读写操作分为两次32位操作进行，其他基本数据类型的读写操作都是原子性的==  
> 实现：加入volatile关键字的代码生成汇编代码，会发现多了一个lock前缀指令，这个指令相当于一个内存屏障，可以防止重排序，通过空的写入操作将变量的修改写入到主内存中，这就实现了可见性

#### 1.6 设计模式
##### 常见的设计模式
> 工厂模式、策略模式、适配器模式、模版模式、装饰模式、代理模式、单例模式等

##### 单例模式代码
```java
//基于类静态变量
public class InstanceFactory {
    private static final Single instance = new Single();
    
    public static Single getInstance() {
        return this.instance;
    }
}

//基于双重校验锁
public class InstanceFactory {
    private static Single instance;
    private static final Object object = new Object();
    
    public static Single getInstance() {
        if(instance == null) {
            synchronized(object) {
                if (instance == null) {
                    instance = new Single();
                }
            }
        }
        return instance;
    }
}
```

#### 1.7 数据结构

#### 1.8 反射、IO
##### 反射
>Class类： 反射的核心类、可以获取类的属性，方法等信息  
>Field类： Java.lang.reflec包中的类，表示类的成员变量，可以用来获取和设置类之中的属性值  
>Method类：Java.lang.reflec包中的类，表示类的方法，可以用来获取类中的方法信息或者执行方法  
>Constructor类：Java.lang.reflec包中的类，表示类的构造方法  

>获取Class对象的3种方法：  
```java
//1
Person p = new Person();
Class clazz = p.getClass;

//2
Class clazz = Person.class;

//3
Class clazz = Class.forName("类的全路径");
```
>创建对象的两种方法
>1.使用Class对象的newInstance()，这种方法需要Class对象对应的类有默认的空构造器  
>2.调用Constructor对象的newInstance(),先通过Class对象获取构造器对象，再通过构造器对象的newInstance()创建

##### BIO、NIO区别
> BIO(Block IO): jkd1.4以前的IO模型，它是一种阻塞IO  
> NIO（NoN-Block IO）：JDK1.4以后才有的IO模型，提高了程序的性能，借鉴比较先进的设计思想，linux多路复用技术，轮询机制  
> AIO（Asynchronous IO）:JDK1.7以后才有的IO模型，相当于NIO2，相当于NIO2，学习Linux epoll模式  

##### Java NIO的原理
>1.多路复用技术：建立连接—发送数据—服务端处理—反馈  
>2.轮询机制（Select模式）  
>3.SelectionKey: 牌号，时间的标识，唯一的身份标识  
>4.Buffer(数据缓冲区)  

>Channel实现：FileChannel(文件IO)、DatagramChannel（UDP）、SocketChannel（Client）、ServerSocketChannel（Server）
```java
//实例
try(ServerSocketChannel serverSocketChannel = ServerSocketChannel.open()) {
            serverSocketChannel.socket().bind(new InetSocketAddress(3388));

            Selector selector = Selector.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            System.out.println("服务器准备就绪，开始监听，端口3388");

            while (true) {
                int wait = selector.select();
                if (wait == 0)
                    continue;

                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = keys.iterator();
                ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

                while (iterator.hasNext()) {
                    SelectionKey key = iterator.next();
                    if (key.isAcceptable()) {
                        //do something
                    } else if (key.isReadable()) {
                        //do something
                    } else if (key.isWritable()) {
                        //do something
                    }

                    iterator.remove();
                }

            }
        } catch (Exception e) {
            e.printStackTrace();
        }
```


#### 2.1数据库
##### 存储引擎的 InnoDB与MyISAM区别，优缺点，使用场景

| 存储引擎 | InnoDB                      | MyISAM                                            |
| -------- | --------------------------- | ------------------------------------------------- |
| 存储文件 | .frm表定义文件 .ibd数据文件 | .frm表定义文件<br> .myd数据文件<br> .myi 索引文件 |
| 锁       | 表锁，行锁                  | 表锁                                              |
| 事务     | ACID                        | 不支持                                            |
| CRUD     | 读写                        | 读多                                              |
| count    | 扫表                        | 专门存储的地方                                    |
| 索引结构 | B+Tree                      | B+Tree                                            |

##### 建立索引的原则
> 1. 最左匹配原则，直到遇到范围查询(>, <, between, like)就停止，比如a = 1 and b = 2 and c >3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，abd的顺序可以任意调整
> 2. = 和 in可以乱序，比如a = 1 and b =2 and c = 3建立(a, b, c) 索引可以任意顺序，mysql查询优化器会帮你优化
> 3. 尽量选择区分度高的索引，区分度公式count(distinct col)/count(*) ，表示字段不重复的比例，比例越大我们的扫描记录越少,比例一般是需要join的字段要求是0.1以上，即平均1条扫描10条记录
> 4. 索引不能参与计算，比如from_unixtime(create_time) = '2014-05-29' 就不能使用到索引，因为b+tree中存的都是数据表中的字段值，但进行检索时，需要把素有元素都应用到函数才能比较，成本大，应该改成create_time = unix_timestamp('2014-05-29')
> 5. 尽量扩展索引，不要新建索引，比如表中已经有a索引，现在要加(a,b)索引，只需要修改原来的索引即可

#### 2.2 Redis

> 建议阅读《Redis开发与运维》《Redis设计与实现》《Redis深度历险：核心原理和应用实践》

##### Redis 为什么是单线程? 为什么单线程还能这么快？
> 单线程能够避免线程切换和竞态产生的消耗，而且单线程可以简化数据结构和算法的实现  
> 至于单线程还快，是因为Redis是基于内存的数据库，内存响应速度是很快的，并且采用epoll作为I/O多路复用技术，再加上Redis自身的事件处理模型将epoll中的连接、读写、关闭都转换为事件，不在网络I/O上浪费过多时间  

>epoll是为了解决Linux内核处理大量文件描述符提出的方案，属于Linux下多路I/O复用接口中select/poll的增强,经常用于Linux下高并发服务型程序，特别是大量并发连接中只有少部分处于活跃下的情况，能提高CPU利用率。  
>epoll采用事件驱动，只需要遍历那些被内核IO事件异步唤醒之后加入到就绪队列并返回到用户空间的描述符集合  
>epoll提供两种触发模式，水平触发（LT）和边沿触发（ET）,目前效率最高的IO操作方案是：epoll+ET+非阻塞IO模型

##### Redis 使用场景
> 最多的应用于缓存，其他可以用于排行榜、计数器、消息队列等

##### Redis 持久化机制
1.RDB持久化：
把当前进程数据生成快照保存到硬盘的过程，触发方式有手动触发和自动触发
手动触发命令：save和bgsave命令
- save命令：阻塞当前Redis服务，直到RDB过程完成为止，不建议线上环境使用
- bgsave命令：Redis进程执行fork操作创建子进程，RDB持久化过程由子进程负责，完成后自动结束，阻塞只发生在fork阶段，一般时间很短
自动触发：
- 使用save相关配置，如"save m n"。表示m秒内数据集存在n次修改时，自动触发
- 如果从节点执行全量复制操作，主节点自动执行bgsave 生成RDB文件并发送给从节点
- 执行debug reload命令重新加载Redis时，也会自动触发save操作
- 默认情况下执行shutdown命令时，如果没有开始AOF持久化则自动执行bgsave

RDB文件保存在dir配置指定的目录下，文件名通过dbfilename配置指定，可以通过执行config set dir {newDir} 和 config set dbfilename {newFileName} 运行期动态执行，当下次运行时RDB文件会保存到新目录

- 遇到磁盘损坏或写满时，可以通过config set dir 在线修改文件路径到可用的磁盘路径，之后执行bgsave进行磁盘切换
- redis默认采用LZF算法对生成的RDB文件进行压缩，压缩后文件元小于内存大小，默认开启，可以通过参数config set rdbcompression {yes|no}动态修改

RDB优点：
- RDB是一个紧凑的二进制文件，代表Redis在某个时间点上的数据快照，适合备份，全量复制等场景
- Redis加载RDB恢复数据远远快于AOF方式   

RDB缺点：
- 没办法做到实时持久化/秒级持久化
- RDB文件使用特定二进制保存，存在兼容问题

2.AOF持久化
以独立日志的方式记录每次写命令，写入的内容直接是文本协议格式，重启时再重新执行AOF文件中的命令达到恢复数据的目的，解决了数据持久化实时性问题
开启AOF：appendonly yes,默认不开启
AOF文件名： appendfilename 配置，默认appendonly.aof
保存路径：同RDB，通过dir配置

AOF工作流程：
1. 所有的写入命令追加到aof_buf(缓冲区)中
2. AOF缓冲区根据对应的策略向硬盘做同步操作
3. 随着AOF文件越来越大，需要定期对AOF文件进行重写，压缩，父进程执行fork创建子进程，由子进程根据内存快照执行AOF重写，父进行继续响应后面的命令，在子进程完成重写后，父进程再把新增的写入命令写入到新的AOF文件中
4. Redis服务重启，加载AOF文件进行数据恢复

AOF为什么直接采用文本协议格式？
- 文本协议具有良好的兼容性
- 开启AOF后，所有写入命令都包含追加操作，直接采用协议格式，避免二次处理开销
- 文本协议具有可读性，方便直接修改和处理

AOF 为什么把命令追加到aof_buf中
- 写入缓存区aof_buf中，能提高性能，并且Redis提供了多种缓存区同步硬盘策略

重写后的AOF文件为什么可以变小？
- 进程内已经超时的数据不再写入文件
- 旧的AOF文件含有无效命令，重写使用进程内数据直接生成，新的AOF文件只保留最终数据的写入命令
- 多条写命令可以合并为一个，为了防止溢出，以64个元素为界拆分为多条

AOF缓冲区同步策略，通过参数appendfsync控制

|可配置值| 说明|其他|
| --- | --- | --- |
|always| 命令写入aof_buf后调用系统fsync操作同步到AOF文件，fsync完成后线程返回| 每次写入都要同步AOF文件，在一般的SATA硬盘很难达到高性能|
|everysec| 命令写入aof_buf后调用系统write操作，write完成后线程返回。fsync同步操作由线程每秒调用一次（建议策略）| 默认同步策略|
|no| 命令写入aof_buf后调用系统write操作，不对AOF文件做fsync同步，同步硬盘操作由操作系统负责，通常同步周期最长30秒| 操作系统每次同步AOF文件的周期不可控，而且会加大每次同步硬盘的数据量，虽然提升了性能，但数据安全性无法保证|

AOF手动触发：调用bgrewriteaof命令
自动触发：有两个参数
- auto-aof-rewrite-min-size: 表示运行时AOF重写时文件最小体积，默认64MB
- auto-aof-rewrite-percentage: 代表当前AOF文件空间（aof_current_size）和上一次重写后AOF文件空间(aof_base_size) 的比值
自动触发时机=aof_current_size > auto-aof-rewrite-min-size && (aof_current_size -aof_base_size) / aof_base_size >= auto-aof-rewrite-percentage

RDB和AOF同时开启，并且AOF文件存在，优先加载AOF文件

AOF文件错误，可以通过redis-check-aof-fix 修复

##### Redis 数据类型
> 字符串  
> key是字符串类型，字符串类型的值可以是字符串、数字、二进制（最大不能超过512MB)，是动态字符串，内部通过预分配冗余空间的方式来减少内存的频繁分配  

|命令|解释|备注|
| --- | --- | --- |
| set | 设置值 | key<br> value<br> [ex seconds] 为键设置秒级过期时间<br> [px milliseconds] 为键设置毫秒级别的过期时间 <br> [nx\|xx] nx键必须不存在才可以设置成功 xx键必须存在才可以设置成功 |
| mset | 批量设置值 | key<br> value<br> [key value ...]|
|mget| 批量获取值 | key<br> [key ...]|
|incr| 对值做自增操作 | 值不是整数，返回错误<br> 值是整数，返回自增后的结果<br> 键不存在，按照值为0自增，返回结果为1|
|decr| 自减 | key |
|incrby| 自增指定数字| key increment |
|decrby| 自减指定数字| key decrement|
|incrbyfloat| 自增浮点数| key increment|
|append| 追加值 | key value|
|strlen| 字符串长度| key|
|getset| 设置并返回原值 | key value|
|setrange| 设置指定位置的字符 | key offeset value|
|getrange| 获取部分字符串 | key start end|

> 哈希  
> 一个键值对结构, 内部结构同Java的 HashMap, 数组+链表的结构，值只能存储字符串。另外在rehash的时候，采用定时任务渐进式迁移内容

|命令| 解释| 备注|
|---|---|---|
|hset| 设置值| key field value|
|hsetnx| 类似setnx|-|
|hget| 获取值| key field|
|hdel| 删除一个或多个field| key field [field ...]|
|hlen| 计算field个数| -|
|hmget| 批量获取| key field [field ...]|
|hmset| 批量设置| key field value [field value ...]|
|hexists| 判断field是否存在| key field|
|hkeys| 获取所有field| key|
|hvals| 获取所有value| key|
|hgetall| 获取所有的field-value| key<br> 如果元素很多，会造成阻塞，建议使用hscan|
|hincrby hincrbyfloat| 类似incrby / incrbyfloat| key field increment|
|hstrlen| 计算value字符串长度| key field|

> 列表  
> 用来存储多个有序的字符串，一个列表最多可以存储2<sup>32</sup> - 1 个元素,列表中的元素可以是重复的，相当于Java中的LinkedList, 是一个链表而不是数组, 底层是采用quicklist结构，在数据量少的时候会使用ziplist压缩列表，数据量多的时候才使用quicklist

|命令| 解释| 备注|
|-|-|-|
|rpush| 从右边插入元素| key value [value ...]|
|lpush| 从左边插入元素| key value [value ...]|
|linsert| 向某个元素前或者后插入元素| key before\|after pivot value<br> 从列表中找到等于pivot元素，在其前before 或者after 插入一个新的元素value|
|lrange| 获取指定范围内的元素列表| key start end<br> 索引下标从左到右是0到N-1, 从右到左是-1到-N; end包含自身|
|lindex| 获取列表指定索引下标的元素| key index|
|llen| 获取列表长度| key|
|lpop| 从列表左侧弹出元素| key|
|rpop| 从列表右侧弹出元素| key|
|lrem| 删除指定元素| key count value<br> 从列表中找到等于value的元素进行删除，count>0, 从左到右，删除最多count个元素。count<0, 从右到左，删除最多count绝对值个元素。count=0,删除所有|
|ltrim| 按照索引范围修剪列表| key start end|
|lset| 修改指定索引下标的元素| key index newValue|
|blpop<br> brpop| 弹出元素阻塞版| key [key ...] 多个键<br> timeount 阻塞时间(秒)|

> 集合(set)   
> 用来保存多个的字符串的元素，但和列表元素不一样的是，集合中不允许有重复元素，并且集合中的元素是无序的，不能通过索引下标获取元素，一个集合最多可以存储2<sup>32</sup> - 1个元素
> 当集合元素个数都是整数并且元素个数小于set-max-intset-entries配置(默认512个)时，使用intset编码减少内存使用，否则使用hashtable编码
>
> 相当于Java中的HashSet, 所有的value都是一个值NULL

|命令| 解释 | 备注|
|-|  -|  -|
|sadd| 添加元素| key element [element ...]|
|srem| 删除元素| key element [element ...]|
|scard| 计算元素个数| key|
|sismember| 判断元素是否在集合中| key element|
|srandmember| 随机从集合返回指定个数元素| key [count]|
|spop| 从集合随机弹出元素,会删除元素| key [count]|
|smembers| 获取所有元素| key|
|sinter| 求多个集合的交集| key [key ...]|
|sunion| 求多个集合的并集| key [key ...]|
|sdiff| 求多个集合的差集| key [key ...]|
|sinterstore<br>suionstore<br>sdiffstore|将交集、并集、差集的结果保存| destination key [key...]<br> destination 目标集合名称<br>集合间的运算在元素比较多的情况下会比较耗时|

> 有序集合  
> 集合不能重复，但集合中的元素可以根据score分数排序,排序从0开始
> 当有序集合的元素个数小于zset-max-ziplist-entries(默认128个)，同时每个元素的值都小于zset-max-ziplist-value配置（默认64字节）时，使用ziplist来作为内部编码，否则使用skiplist作为内部编码
>
> 其相当于Java的SortedSet 和 HashMap结合体，内部实现’跳跃表‘ 的数据结构

|命令| 解释| 备注|
|-| -| -|
|zadd|添加成员| key [NX\|XX] [CH] [INCR]score member [score member ...]<br> [NX\|XX] NX: 只有在元素不存在时候添加新元素; XX：更新已经存在的元素，不添加元素<br> CH 返回有序集合元素和分数发生变化的个数  incr 对score做增加，相当于zincrby|
|zcard| 计算成员个数| key|
|zscore| 计算某个成员的分数| key member|
|zrank| 计算成员排名,分数从低到高| key member|
|zrevrank| 计算成员排名，分数从高到低| key member|
|zrem| 删除成员| key member [member ...]|
|zincrby| 增加成员分数| key increment member|
|zrange<br>zrevrange| 返回指定排名范围的成员| key start end [withscores] 显示分数|
|zrangebyscore<br>zrevrangebyscore| 返回指定分数范围的成员| key min max [withscores] [limit offset count]<br> key max min [withscores] [limit offset count]<br>min和max 支持开区间（小括号） 和闭区间（中括号），-inf 和 +inf 无限小和无限大士大|
|zcount| 返回指定分数范围成员个数| key min max|
|zremrangebyrank| 删除指定排名内的升序元素| key start end|
|zremrangebyscore| 删除指定分数范围的成员| key min max|
|zinterstore| 交集| destination numkeys key [key ...] [weights weight [weight ...]] [aggregate sum\|min\|max]<br> destination 交集计算结果保存到这个键<br> numkeys 需要做交集计算键的个数<br> key[key...] 需要做交集计算的键<br> weights weight[weight...] 每个键的权重，在做交集计算时，每个键中的每个member会将自己分数乘以这个权重，每个键的权重默认为1<br> aggregate sum\|min\|max  计算成员交集后，分值可以按照sum、min、max 做汇总，默认是sum|
|zunionstroe| 并集| 参数同zinterstore|

内部编码表

![e5c2bf73](https://tva1.sinaimg.cn/large/006y8mN6ly1g6lnmw7glhj31240u0aou.jpg)


> 键相关命令
> 字符串类型的键，执行set命令会清除过期时间

|命令|解释|备注|
|-|-|-|
|rename| 键重命名| key newkey<br>如果键已经存在会把value也覆盖，如果不想被覆盖，可以使用renamenx,重名键期间会执行del命令删除旧的键，如果键值比较大，会造成阻塞|
|randomkey| 随机返回一个键|
|expire<br>pexpire| 键在seconds秒后过期,后者毫秒| key seconds|
|expireat<br>pexpireat| 键在秒级时间戳后过期，后者毫秒| key timestamp|
|ttl<br>pttl| 查询键剩余过期时间，pttl毫秒级别| -1 键没有设置过期时间<br> -2 键不存在|
|persist| 清除键的过期时间| key|

> 键迁移
> move key db 用于在Redis内部多数据库之间进行迁移
> dump+restore 不同的实例之间进行数据迁移
> migrate host port key\|"" destination-db timeout [copy] [replace] [keys key...] 实际上是dump、restore、del是三个命令的组合，具有原子性
> host 目标redis的Ip地址
> port 目标redis的端口
> key|"" 表示迁移多个键
> destination-db 目标redis数据库索引
> timeout 迁移的超时时间(毫秒)
> [copy] 迁移后并不删除源键
> [replace]  对目标redis的键进行覆盖
> [keys key...] 需要迁移的键

>遍历键
>keys pattern
>\* 代表匹配任意字符
>. 代表匹配一个字符
>[] 代表匹配部分字符，如[1,3]匹配1，3 [1-10]匹配1到10任意数字
>\x用来做转义，如匹配问号需要转义
>该命令会大数量下会造成阻塞，建议使用scan

> scan cursor [match pattern] [count number]
> cursor 必需参数，从0开始的游标
> match pattern 可选，模式匹配
> count number 可选，每次要遍历的键个数，默认是10
> 其他类型命令： 哈希hscan 集合sscan 有序集合zscan 用法与scan相同
> 问题：渐进式遍历中，如果有新增或删除键，就可能会没有完整遍历出所有的键

Bitmaps
>其并不是一种数据结构，实际上就是字符串，可以对字符串的位进行操作，内部使用二进制存储,在存储超大上亿数据的时候，能节约很多内存空间

|命令| 说明| 备注|
|-|-|-|
|setbit| 设置值| key offset value<br>offset 偏移量，可以理解为索引|
|getbit| 获取值| key offset|
|bitcount| 获取指定范围值为1的个数| [start] [end] <br> [start]和[end]代表起始和结束字节数|
|bitop| 多个bitmaps操作| op destkey key[key ...]<br>  and(交集)、or(并集)、not(非)、xor（异或）<br> destkey 结果保存的key|
|bitpos| 计算第一个值为targetBit的偏移量| key targetBit [start] [end]|

HyperLogLog
> 其实际类型是字符串类型，是一种基数算法，可以利用极小的内存空间完成独立总数的统计
> HyperLogLog内存占用小，但存在一定错误率

|命令| 说明| 备注|
|-|-|-|
|pfadd| 添加| key element [element ...]|
|pfcount| 计算独立用户数| key [key ...]|
|pfmerge| 合并| destkey sourcekey [sourcekey ...]|

##### Pipeline
Pipeline(流水线) 机制能将一组Redis命令进行组装，通过一次RTT传输给Redis,再将这组Redis命令的执行结果按照顺序返回给客户端

原生批量命令与Pipeline对比：
*   原生批量命令是原子的，Pipeline是非原子的
*   原生批量命令是一个命令对应多个key,Pipeline支持多个命令
*   原生批量命令是Redis服务端支持实现的，而Pipeline需要服务端和客户端共同实现

##### 事务
redis 简单事务功能，将一组需要一起执行的命令放到multi和exec两个命令之间。multi命令代表事务开始，exec命令代表事务结束，他们之间的命令是原子顺序执行的。如果要停止事务的执行，用discard命令代替exec命令

watch 命令 用来确保事务中的key没有被其他客户端修改过，才执行事务，否则不执行（类似乐观锁）

##### Lua
两种方法：
- eval 
eval 脚本内容 key个数 key列表 参数列表
```lua
eval 'return "helo "..KEYS[1].. ARGV[1]' 1 redis world
```
还可以使用redis-cli --eval 直接执行文件

- evalsha 
evalsha 脚本sha1值 key个数 key列表 参数列表
将Lua脚本加载到Redis服务端，得到该脚本的SHA1校验和，evalsha使用SHA1作为参数执行对应的Lua脚本，避免每次发送脚本，脚本常驻与内存中

加载脚本到Redis:  redis-cli script load "$(cat ~/lua_test.lua)"

##### 慢查询分析
> 配置文件配置: slowlog-log-slower-than 超过多少阈值（微秒，默认10毫秒）；slowlog-max-len 慢日志最大长度
> 命令修改： config set 和  config rewrite（写入配置文件）
> config set slowlog-log-slower-than 100
> config set slowlog-max-len
- 获取慢查询日志： 
 慢查询只记录命令执行时间，不包括命令排队和网络传输时间
 slowlog get [n] n指定条数

- 日志数据结构
> 3  1) (integer) 3    （id）
> 2) (integer) 1567317373   (发送时间戳)
> 3) (integer) 21  (命令消耗时间)
> 4) 1) "keys"  (命令和参数)
>    2) "*"
>
> 慢查询日志重置：
> slowlog reset

##### Redis键过期删除策略
键过期，内部保存在过期字典中，Redis采用惰性删除和定时任务删除机制；
惰性删除用于在客户端读取带有超时属性的键时，如果已经超过设置的过期时间，会执行删除并返回空，但这种方式如果键过期，而且一直没有被重新访问，键一直存在
定时任务删除：能够解决惰性删除问题，Redis内部维护一个定时任务，每秒运行10次。根据键的过期比例，使用快慢两种速率模式回收键

##### Redis高可用方案
1. 主从模式：一主二从
配置redis.conf , 从节点配置 slaveof 127.0.0.1 6379 
确认主从关系： redis-cli -h 127.0.0.1 -p 6379 info replication

2. 哨兵模式：
配置 redis-sentinel.conf ,sentinel monitor mymaster 127.0.0.1 6379 1
最后一位是选举master需要的票数
启动哨兵：
redis-sentinel redis-sentinel.conf
或者 redis-server redis-sentinel.conf --sentinel

3. redis-cluster:
  每个节点保存数据和整个集群状态，每个节点都和其他节点连接。采用哈希函数把数据映射到一个固定范围的整数集合中，整数定义为槽。所有键根据哈希函数映射到0~16383整数槽内，公式 slot = CRC16(key) & 16383

4. Codis

  https://juejin.im/post/5c132b076fb9a04a08218eef

- 集群限制：

1. key批量操作支持限制，目前只支持具有相同slot值的key执行批量操作

2. key事务操作支持有限
3. key作为数据分区最小粒度，不能将大的键值对象hash、list等映射到不同节点
4. 只能使用0号数据库
5. 从节点只能复制主节点，不支持嵌套树状复制结构

##### 缓存问题

- 缓存穿透

    缓存穿透是指，缓存中不存在该key的数据，于是就是去数据库中查询，如果流量一大，就会打趴数据库

    优化:

    1. 缓存空对象

        对于不存在的数据，依旧将空值缓存起来。但这会造成内存空间的浪费，可以针对这类数据加一个过期时间。对于缓存和存储层数据的一致性，可以在过期的时候，请求存储层，或者通过消息系统更新缓存

    2. 布隆过滤器

        将所有存在的数据做成布隆过滤器，可以使用Bitmaps实现，其在大数据量下空间占用小

        Redis4.0以上采用插件集成，https://github.com/RedisBloom/RedisBloom

        原理：

        在redis中是一个大型的位数组，通过计算key的hash然后对数组长度取模得到一个位置，进行写入；在判断是否存在时，判断位数组中几个位置是否都为1，只要一个位为0，就说明这个key不存在。布隆过滤器也会存在一定的误判，如果位数组比较稀疏，概率就会很大，否则就会降低。

- 缓存雪崩

    指缓存由于大量请求，造成缓存挂掉，大量请求直接打到存储层，造成存储层挂机

    优化：

    1. 使用主从，哨兵，集群模式保证缓存高可用
    2. 依赖隔离组件为后端限流并降级
    3. 提前演练，做好后备方案

- 缓存更新方式

    同步更新，先写入数据库，写入成功后，再更新缓存。异步更新，通过消息中间件进行触发更新。失效更新，在取不到缓存的时候，从数据库取数据，再更新缓存。定时更新，通过定时任务来更新缓存

- 缓存不一致

    通过增加重试机制，补偿任务，达到最终一致

- 热点key重建

    优化

    1. 使用互斥锁，只允许一个线程重建缓存
    2. 使用逻辑缓存过期，在value中存一个key过期时间，在获取key的时候通过逻辑时间进行判断



#### 2.3 消息队列
##### 消息的幂等性处理思路
> 主要是防止消息重复消费，通过业务方去保证消息的幂等性，每条消息设置一个唯一Id，数据库的话通过唯一索引。

##### 消息队列如何保证高可用    
> kafka可以有多个Borker，一个topic会将数据存储在不同的partiton上, 并且有多个副本来同步数据，但只会有一个leader，数据以Log的形式存储在硬盘中，并且记录了消费的offset。如果leader挂掉，会从ISR集合中的副本选出一个做为leader。

##### 如何保证消息可靠性
> kafka保证消息可靠性，可以通过如下几个配置:  
> 1.生产者配置 acks = all (-1) ISR中的所有副本都写入成功，才代表该消息发送成功
> 2. min.insync.replicas默认为1，指定了ISR集合中最小的副本数，不满足条件就会抛出NotEnoughReplicasException 或 NotEnoughReplicasAfterAppendException,也就是必须保证有一个副本同步数据跟得上leader
> 3. unclean.leader.election.enable 默认为false, 当leader下线的时候可以从非ISR集合中选举新的leader，这样能提高可用性，但会丢数据
> 4. 配置log.flush.interval.messages 和 log.flush.interval.ms 同步刷盘策略
> 5. 消费端手动提交位移，enable.auto.commit 设置为false，自动提交会带来重复消费和消息丢失的问题，客户端如果消息消费失败应该放入死信队列，以便后期排除故障
> 6. 回溯消费功能，消息貌似已经成功消费，但实际消息失败了，不知道是什么错误造成的，就可以通过回溯消费补偿，或者复现 “丢失”，seek()方法

##### 消息积压问题
> 如果消息积压了太多，一直消费不了，需要检查是不是consumer有问题，或者服务端磁盘是否快满了。
> 1. consumer有问题就修复问题。但由于积压的数据太多，用原程序消费还是太慢。就需要扩容，新建临时topic,将分区改为原来的10倍，写程序将原来积压的消息发送到新建的topic中，启动10倍的机器来消息这些数据
> 2. 服务端磁盘满，就只能扩容服务端磁盘，再采用第一种办法来修复问题

##### kafka的分区策略
> 消费者客服端参数partition.assignment.strategy 来配置消费分区策略
> 1. RangeAssignor 默认分配策略 通过  分区数/消费者总数 来获得一个跨度进行分配
> 2. RoundRobinAssignor 轮询分配策略
> 3. StickyAssignor 能够使分区的分配尽可能与上一次保持一致，避免过度重分配
> 4. 自定义分配，实现PartitionAssignor接口

#### 2.4 Spring知识点

> 《精通Spring 4.x 企业应用开发实战》、《Spring技术内幕：深入Spring架构与设计原理》

##### BeanFactory 与ApplicationContext 是干什么的，两者的区别

BeanFactory、ApplicationContext都代表容器，BeanFactory是一个基础接口，实现了容器基础的功能，ApplicationContext是容器的高级形态，增加了许多了特性，顶级父类是BeanFactory

##### Spring IOC容器如何实现

Spring为提供了各种各样的容器，有DefaultListableBeanFactory、FileSystemXmlApplicationContext等，这些容器都是基于BeanFactory，其中Spring通过定义BeanDefinition来管理应用的各种对象及依赖关系，其是容器实现依赖反转功能的核心数据结构

##### Spring如何解决循环依赖问题

比如A依赖B, B依赖A.

创建A的时候，会把A对应的ObjectFactory放入缓存中，当注入的时候发现需要B, 就会去调用B对象，B对象会先从singletonObjects 查找，没有再从earlySingletonObjects找，还没有就会调用singletonFactory创建对象B,B对象也是先从singletonObjects，earlySingletonObjects，singletonFactories三个缓存中搜索，只要找到就返回，相关方法`AbstractBeanFactory.doGetBean()`

##### Spring Bean 生命周期

1. Bean实例的创建
2. 为Bean实例设置属性
3. 调用Bean的初始化方法
4. 应用可以通过IOC容器使用Bean
5. 当容器关闭时，调用Bean的销毁方法

##### Spring Bean的作用域，默认是哪个？

Singleton: 单例模式，IOC容器中只会存在一个共享的Bean实例，是Spring的默认模式； 

prototype: 原型模式，每次从IOC容器获取Bean的时候，都会创建一个新的Bean实例；

 request: 每次请求都生成一个实例；

 session: 在一次请求会话中，容器返回该Bean的同一实例，不同的Session请求不同的实例，实例仅在该Session内有效，请求结束，则实例销毁；

 globalsession:  全局的session中，容器返回该Bean的同一个实例,仅在portlet context 有效

##### AOP两种代理方式

默认策略是如果目标类是接口，则使用JDK动态代理技术，否则使用Cglib来生成代理

1. JDK 动态代理

    主要涉及java.lang.reflect中的两个类，Proxy 和 InvocationHandler

    InvocationHandler是一个接口，通过实现该接口定义横切逻辑，并通过反射机制调用目标类的代码，动态将横切逻辑和业务逻辑编辑在一起。只能为实现接口的类创建代理。

2. Cglib 代理

    是一个强大的高性能，高质量的代码生成类库，可以在运行期扩展Java类与实现Java接口，Cglib封装了asm,可以在运行期动态生成新的class。可以是普通类，也可以是实现接口的类

##### AOP 切点函数

| 类别               | 函数          | 入参           | 说明                                                         |
| ------------------ | ------------- | -------------- | ------------------------------------------------------------ |
| 方法切入点函数     | execution()   | 方法匹配模式串 | 满足某一匹配模式的所有目标类方法连接点。<br/>如execution(* greetTo(..)) 表示所有目标类中的greetTo()方法 |
|                    | @annotation() | 方法注解类名   | 标注了特定注解的目标类方法连接点。<br/>如@annotation(com.smart.anno.NeedTest)表示任何标注了@NeedTest注解的目标类方法 |
| 方法入参切入点函数 | args()        | 类名           | 通过判断目标类方法运行时入参对象的类型定义指定连接点。<br/>如args(com.smart.Waiter)表示所有有且仅有一个按类型匹配于Waiter入参的方法 |
|                    | @args()       | 类型注解类名   | 通过判断目标类方法运行时入参对象的类是否标注特定注解来指定连接点。<br/>如@args(com.smart.Monitorable)表示任何这样的一个目标方法：它有一个入参且`入参对象的类`标注@Monitorable注解 |
| 目标类切点函数     | within()      | 类名匹配串     | 表示特定域下的所有连接点。<br>如within(com.smart.service.\*) 表示com.smart.service 包中的所有连接点，即包中所有类的所有方法；<br>而within(com.smart.service.\*Service)表示在com.smart.service包中所有以Service结尾的类的所有连接点 |
|                    | target()      | 类名           | 假如目标类按类型匹配于指定类，则目标类的所有连接点匹配这个切点<br>如通过target(com.smart.Waiter)，Waiter及Waiter实现类NaiveWaiter中的所有连接点都匹配该切点 |
|                    | @within()     | 类型注解类名   | 假如目标类型按类型匹配于某个类A, 且类A标注了特定注解，则目标类的所有连接点匹配该切点<br>如@within(com.smart.Monitorable) 假如Waiter类标注了@Monitorable注解，则Waiter的所有连接点都匹配该切点，`说是这个注解也会匹配Waiter的子类，但试了后并没有用，Spring 5.1` |
|                    | @target       | 类型注解类名   | 假如目标类标注了特定注解，则目标类的所有连接点都匹配该切点。<br>如@target(com.smart.Monitorable),假如NaiveWaiter标注了@Monitorable,则NaiveWaiter的所有连接点都匹配这个切点 |
| 代理类切点函数     | this()        | 类名           | 代理类按类型匹配于指定类，则被代理的目标类的所有连接点都匹配该切点。<br>如this(com.smart.Seller) 匹配任何运行期对象为Seller类型的类 |



#####  六种增强类型

1. @Before 前置增强，相当于BeforeAdvice
2. @AfterReturning 后置增强，相当于AfterReturningAdvice
3. @Around 环绕增强，相当于MethodInterceptor
4. @AfterThrowing 抛出增强，相当于ThrowsAdvice
5. @After Final增强，不管抛出异常还是正常退出，都会执行，没有对应的增强接口，一般用于释放资源
6. @DeclareParents 引介增强，相当于IntroductionInterceptor

##### Spring AOP实现原理

通过JDK代理，和CGLIB代理两种方式生成动态代理，构造不同的回调方法来对拦截器链的调用，比如JdkDynamicAopProxy的invoke方法，Cglib2AopProxy中的DynamicAdvisedInterceptor的intercept方法，他们最终都是通过ReflectiveMethodInvocation的proceed方法实现对拦截器链的调用, 首先需要根据配置来对拦截器进行匹配，匹配成功后，拦截器发挥作用，在对拦截器调用完成后，再对目标对象的方法调用，这样一个普通的Java对象的功能就得到了增强

##### Spring MVC运行流程

![image-20190910155238902](https://tva1.sinaimg.cn/large/006y8mN6ly1g6uh335ksuj31au0m8nhh.jpg)

1. 客户端请求到DispatcherServlet
2. DispatcherServlet根据请求地址查询映射处理器HandleMapping，获取Handler
3. 请求HandlerAdatper执行Handler
4. 执行相应的Controller方法，执行完毕返回ModelAndView
5. 通过ViewResolver解析视图，返回View
6. 渲染视图，将Model数据转换为Response响应
7. 将结果返回给客户端

`2，3 两步都在DispatcherServlet -> doDispatch中进行处理`



##### Spring MVC 启动流程

1. 在Tomcat启动的时候，ServletContext 会根据web.xml加载ContextLoaderListener，继而通过ContextLoaderListener 载入IOC容器，具体过程有ContextLoader完成，这个IOC容器是在Web环境在使用的WebApplicationContext, 这个容器在后面的DispatcherServlet中作为双亲根上下文来使用
2. IOC容器加载完成后，开始加载DIspatcherServlet，这是Spring MVC的核心，由`HttpServletBean -> initServeltBean`启动（HttpServletBean是DispatcherServlet的父类，HttpServletBean继承了HttpServlet），最终调用`DispatcherServlet -> initStrategies` 方法对HandlerMapping、ViewResolver等进行初始化，至此，DispatcherServelt就初始化完成了，它持有一个第一步完成的上下文作为根上下文，以自己的Servlet名称命名的IOC容器，这个容器是一个WebApplicationContext对象。

##### Spring 事务实现方式、事务的传播机制、默认的事务类别

1. 事务实现方式

    - 声明式，在xml文件中通过tx:advice来配置事务
    - 注解式，在xml文件中定一个事务管理对象（DataSourceTransactionManager），然后加入\<tx:annotation-driven/>, 这样就可以使用@Transactional注解配置事务

2. 事务的传播机制

    一共7种事务传播行为，相关code: `AbstractPlatformTransactionManager -> getTransaction`

    - PROPAGATION_REQUIRED

        如果当前没有事务，则新建一个事务；如果已经存在一个事务，则加入到这个事务中，这也是默认事务类别

    - PROPAGATION_SUPPORTS

        支持当前事务。如果当前没有事务，则以非事务方式执行

    - PROPAGATION_MANDATORY

        使用当前事务。如果当前没有事务，则抛出异常

    - PROPAGATION_REQUIRES_NEW

        新建事务。如果当前存在事务，则把当前事务挂起

    - PROPAGATION_NOT_SUPPORTED

        以非事务方式执行操作。如果当前存在事务，则把当前事务挂起

    - PROPAGATION_NEVER

        以非事务方式执行。如果当前存在事务，则抛出异常

    - PROPAGATION_NESTED

        如果当前存在事务，则在嵌套事务内执行；如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作

##### Spring 事务底层原理

1. 事务的准备

    在声明式事务处理中，需要Ioc容器配置TransactionProxyFactoryBean,其父类AbstractSingletonProxyFactoryBean实现了InitializingBeean接口,因此在初始化过程中会调用afterPropertiesSet方法，这个方法实例化了ProxyFactory, 并为其设置了通知，目标对象后，最终返回Proxy代理对象，对象建立起来后，在调用其代理方法的时候，会调用相应的TransactionInterceptor拦截器，在这个调用中，会根据TransactionAttribute配置的事务属性进行配置，为事务处理做好准备

2. 事务拦截器实现

    经过TransactionProxyFactoryBean的AOP包装后，此时如果对目标对象进行方法调用，实际上起作用的是一个Proxy代理对象，拦截器会拦截其中的事务处理，在调用Proxy对象的代理方法时会触发invoke回调,其中会根据事务属性配置决定具体用哪一个PlatformTransactionManager来完成事务操作

##### Spring事务失效（事务嵌套), JDK动态代理给Spring事务埋下的坑

https://blog.csdn.net/bntx2jsqfehy7/article/details/79040349

##### Spring 单例实现原理

在创建Bean的时候`AbstractAutowireCapableBeanFactory -> doCreateBea0` 通过BeanDefinition 设置的是否单例属性，来判断该bean是否是单例，如果是单例就 根据Bean的名称删除bean缓存中同名称的bean,再在后面重新创建bean.

##### Spring 中有哪些不同类型的事件

对于ApplicationEvent 类和在ApplicatinContext接口中处理的事件，如果一个Bean实现了ApplicationListener接口，但一个ApplicationEvent发布后，Bean会自动被通知

```java
public class AllApplicationEventListener implements ApplicationListener<ApplicationEvent> {
    @Override
    public void onApplicationEvent(ApplicationEvent applicationEvent) {
        
    }
}
```

Spring 提供了5种标准的事件：

1. 上下文更新事件（ContextRefreshedEvent）：该事件会在ApplicationContext被初始化或者更新时发布。也可以在ConfigurableApplicationContext 接口中的refresh()方法时触发
2. 上下文开始事件（ContextStartedEvent）：当容器调用ConfigurableApplicationContext的Start()方法开始或重新开始容器时触发该事件
3. 上下文停止事件（ContextStoppedEvent）：当容器调用ConfigurableApplicationContext的Stop()方法停止容器时触发该事件
4. 上下文关闭事件（ContextCloseEvent）：当ApplicationContext被关闭时触发该事件。容器被关闭时，其管理的所有单例Bean都被销毁
5. 请求处理事件（RequestHandledEvent）：在Web应用中，当一个Http请求（Request）结束时触发该事件
6. 自定义事件继承ApplicationEvent, 在通过ApplicationContext 接口的publishEvent()方法发布事件

#### 2.5 分布式、微服务知识点

> 参考《高可用可伸缩微服务架构》

#####  领域驱动有了解吗？什么是领域驱动模型？

##### JWT有了解吗，什么是JWT

##### 说说如何设计一个良好的 API

##### 如何保证接口的幂等性

##### 说说 CAP 定理、BASE 理论
##### 说说最终一致性的实现方案
##### 微服务与 SOA 的区别

SOA即面向服务架构，关注点是服务，现有的分布式服务化技术有Dubbo等

1. 微服务是一种经过改良架构设计的SOA解决方案，是面向服务的交互方案
2. 微服务更趋向于以自治的方式产生价值
3. 微服务与敏捷开发的思想高度结合在一起，服务的定义更加清晰，同时减少了企业ESB开发的复杂性
4. 微服务是SOA思想的一种提炼
5. SOA是重ESB,微服务是轻网关

##### 如何拆分服务、水平分割、垂直分割
##### 如何应对微服务的链式调用异常
##### 如何快速追踪与定位问题
##### 如何保证微服务的安全、认证

#### 2.6 Dubbo

> 最好的文档：http://dubbo.apache.org/zh-cn/docs/user/quick-start.html

##### Dubbo SPI的理解

Dubbo SPI 跟Java SPI很相似，Java SPI是Java内置的一种服务提供发现功能，一种动态替换发现机制。

Java  SPI使用方法：

1. 在META-INF/services 目录下放置配置文件，文件名是接口全路径名，文件内部是要实现接口的实现类全路径名，编码用UTF-8
2. 使用ServiceLoad.load(xx.class)调用

Dubbo比Java 增加了：

1. 可以方便地获取某一个想要的扩展实现
2. 对于扩展实现IOC依赖注入功能
3. @SPI声明一个扩展接口，@Adaptive用在方法上，表示自动生成和编译一个动态的Adaptive类，如果用在类上表示一个装饰模式的类

Dubbo  通过ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()方法进行加载，每个SPI接口(@SPI注解的接口)都会产生一个ExtensionLoader扩展加载器实例，保存在名为EXTENSION_LOADERS的ConcureentMap中，通过扩展加载器调用getAdaptiveExtension()方法来获得自适应对象，并注入对象依赖属性



##### Dubbo 基本原理、执行流程

1. 服务容器Container负责启动加载运行服务提供者Provider，根据Provider配置文件根据协议发布服务，完成服务初始化
2. 在Provider（服务提供者）启动时，根据配置中的Registry地址连接注册中心，将Provider的服务信息发布到注册中心
3. Consumer（消费者）启动时，根据消费者配置文件中的服务引用信息，连接到注册中心，向注册中心订阅自己所需的服务
4. 注册中心根据服务订阅关系，返回Provider地址列表给Consumer，如果有变更，注册中心会推送最新的服务地址信息给Consumer
5. Consumer调用远程服务时，根据路由策略，先从缓存的Provider地址列表中选择一台进行，跨进程调用服务，假如调用失败，再重新选择其他调用
6. 服务Provider和Consumer,会在内存中记录调用次数和调用时间，每分钟发送一次统计数据到Monitor



##### Dubbo 负载均衡策略、集群策略

负责均衡策略：

1. RoundRobinLoadBalance 权重轮询算法，按照公约后的权重设置轮询比例，把来自用户的请求轮流分配给内部的服务器
2. LeastActiveLoadBalance 最少活跃调用数均衡算法，活跃数是指调用前后计数差，使慢的机器收到的更少
3. ConsistenHashLoadBalance 一致性Hash算法，相同参数的请求总是发到同一个提供者
4. RandomLoadBlance 随机均衡算法，按权重设置随机概率，如果每个提供者的权重都相同，那么随机选一个，如果权重不同，则先累加权重值，从0~累加权重值选一个随机数，判断该随机数落在哪个提供者上

集群策略：

1. FailoverCluster:  失败转移，当出现失败时，重试其他服务器，通常用于读操作，但重试会带来更长延迟，默认集群策略
2. FailfastCluster: 快速失败，只发起一次调用，失败立即报错，通常用于非幂等性操作
3. FailbackCluster: 失败自动恢复， 对于Invoker调用失败，后台记录失败请求，任务定时重发，通常用于通知
4. BroadcastCluster: 广播调用，遍历所有Invokers,如果调用其中某个invoker报错，则catch住异常，这样就不影响其他invoker调用
5. AvailableCluster: 获取可用调用，遍历所有Invokers并判断Invoker.isAvalible,只要有一个为true就直接调用返回，不管成不成功
6. FailsafeCluster: 失败安全，出现异常时，直接忽略，通常用于写入审计日志等操作
7. ForkingCluster: 并且调用，只要一个成功即返回，通常用于实时性要求较高的操作，但需要更多的服务资源
8. MergeableCluster: 分组聚合，按组合并返回结果，比如某个服务接口有多种实现，可以用group区分，调用者调用多种实现并将得到的结果合并

##### 注册中心挂了还可以通信吗？

可以。对于正在运行的 Consumer 调用 Provider 是不需要经过注册中心，所以不受影响。并且，Consumer 进程中，内存已经缓存了 Provider 列表。

那么，此时 Provider 如果下线呢？如果 Provider 是**正常关闭**，它会主动且直接对和其处于连接中的 Consumer 们，发送一条“我要关闭”了的消息。那么，Consumer 们就不会调用该 Provider ，而调用其它的 Provider 。

另外，因为 Consumer 也会持久化 Provider 列表到本地文件。所以，此处如果 Consumer 重启，依然能够通过本地缓存的文件，获得到 Provider 列表。

再另外，一般情况下，注册中心是一个集群，如果一个节点挂了，Dubbo Consumer 和 Provider 将自动切换到集群的另外一个节点上。

