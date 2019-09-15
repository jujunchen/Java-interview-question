# Java-Interview-Question
#### Java面试题整理

##### 1.1 Java基础

自定义注解

##### 1.2 常见的集合

HashMap 1.7 和 1.8 的区别
##### 1.3 并发
Synchronized原理
HashMap在并发下会产生什么问题？有什么替代方案?(HashTable, ConcurrentHashMap)。它们两者的实现原理。
ThreadLocal 子类及原理, OOM产生原因及防治
Java8新增的原子操作类
ConcurrentLinkedQueue、LinkedBlockingQueue、ArrayBlockingQueue、PriorityBlockingQueue、DelayQueue、SynchronousQueue、LinkedBlockingDeque、LinkedTransferQueue
ThreadPoolExecutor构造函数有哪几个参数，实现原理，创建线程池的方式
为什么建议在不用线程池的时候，关闭线程池，线程池的作用就是复用线程的
Timer 和 ScheduledThreadPoolExecutor 区别
CountDownLatch原理、CyclicBarrier原理，两者区别
Semaphore 原理
Exchanger 原理
如何实现一个生产者与消费者模型
##### 1.4 锁
自旋锁、自适应自旋、锁消除、锁粗化、轻量级锁、偏向锁
公平锁、非公平锁
AbstractQueuedSynchronizer的作用
JDK8新增的锁
##### 1.5 JVM
JVM运行时内存区域划分
OOM,及SOE的示例、原因，排查方法
如何判断对象可以回收或存活
常见的GC算法
常见的JVM性能监测分析工具
JVM优化
什么时候会触发FullGC
类加载器有几种
什么是双亲委派模型？双亲委派模型的破坏
类的生命周期
强引用、软引用、弱引用、虚引用
CMS和G1的区别
volatile的特点
##### 1.6 设计模式
常见的设计模式
单例模式代码
##### 1.7 数据结构
##### 1.8 反射、IO
反射
BIO、NIO区别
Java NIO的原理
##### 2.1数据库
存储引擎的 InnoDB与MyISAM区别，优缺点，使用场景
建立索引的原则
##### 2.2 Redis
Redis 为什么是单线程? 为什么单线程还能这么快？
Redis 使用场景
Redis 持久化机制
Redis 数据类型
Pipeline
事务
Lua
慢查询分析
Redis键过期删除策略
Redis高可用方案
缓存问题
##### 2.3 消息队列
消息的幂等性处理思路
消息队列如何保证高可用    
如何保证消息可靠性
消息积压问题
kafka的分区策略
##### 2.4 Spring知识点
BeanFactory 与ApplicationContext 是干什么的，两者的区别
Spring IOC容器如何实现
Spring如何解决循环依赖问题
Spring Bean 生命周期
Spring Bean的作用域，默认是哪个？
AOP两种代理方式
AOP 切点函数
六种增强类型
Spring AOP实现原理
Spring MVC运行流程
Spring MVC 启动流程
Spring 事务实现方式、事务的传播机制、默认的事务类别
Spring 事务底层原理
Spring事务失效（事务嵌套), JDK动态代理给Spring事务埋下的坑
Spring 单例实现原理
Spring 中有哪些不同类型的事件
##### 2.5 分布式、微服务知识点
领域驱动有了解吗？什么是领域驱动模型？
JWT有了解吗，什么是JWT
说说如何设计一个良好的 API
如何保证接口的幂等性
说说 CAP 定理、BASE 理论
说说最终一致性的实现方案
微服务与 SOA 的区别
如何拆分服务、水平分割、垂直分割
如何应对微服务的链式调用异常
如何快速追踪与定位问题
如何保证微服务的安全、认证

##### 2.6 Dubbo
Dubbo SPI的理解
Dubbo 基本原理、执行流程