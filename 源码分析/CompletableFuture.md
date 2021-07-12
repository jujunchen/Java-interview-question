# CompletableFuture 源码分析

> 源码基于open jdk 11

CompletableFuture 是 jdk8 引入的类，主要是对Future的补充。

CompletableFuture类的官方API文档解释：

1. CompletableFuture是一个在完成时可以触发相关方法和操作的Future，并且它可以视作为CompletableStage。
2. 除了直接操作状态和结果的这些方法和相关方法外（CompletableFuture API提供的方法），CompletableFuture还实现了以下的CompletionStage的相关策略：
    - 非异步方法的完成，可以由当前CompletableFuture的线程提供，也可以由其他调用完方法的线程提供。
    -  所有没有显示使用Executor的异步方法，会使用ForkJoinPool.commonPool()（那些并行度小于2的任务会创建一个新线程来运行）。为了简化监视、调试和跟踪异步方法，所有异步任务都被标记为CompletableFuture.AsynchronouseCompletionTask。
    -  所有CompletionStage方法都是独立于其他公共方法实现的，因此一个方法的行为不受子类中其他方法的覆盖影响。
3. CompletableFuture还实现了Future的以下策略
    - 不像FutureTask，因CompletableFuture无法直接控制计算任务的完成，所以CompletableFuture的取消会被视为异常完成。调用cancel()方法会和调用completeExceptionally（）方法一样，具有同样的效果。isCompletedEceptionally()方法可以判断CompletableFuture是否是异常完成。
    -  在调用get()和get(long, TimeUnit)方法时以异常的形式完成，则会抛出ExecutionException,大多数情况下都会使用join()和getNow(T)，它们会抛出CompletionException。

待回答的几个问题：

1、如何实现并发执行任务?

2、并发执行如何获取任务结果?



https://blog.csdn.net/CoderBruis/article/details/103181520?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.control

