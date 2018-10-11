### java线程池核心知识点

1. 实现的层次是：
   1. Executor接口提供了execute方法，可以执行一个runnable对象。
   2. ExecutorService接口扩展了Executor接口，提供了除了执行之外的更多方法，包括shutdown，submit系列方法（可以拿到执行的凭证，可以通过凭证来查询执行的情况）以及invoke系列方法（调用多个任务invokeAll，或者调用多个任务中的一个invokeAny）
   3. AbstractExecutorService实现了ExecutorService接口。实现了submit方法（通过将runnable和callable转换为runnableFuture对象futureTask）。实现了invokeAny，通过ExecutorCompletionService这个结构包装了一下，然后等待队列不为空（同时可以设置超时），这样打到invokeAny的效果。实现了invokeAll，将任务都进行execute，然后等待某个任务完成，需要不断的更新超时时间。
   4. ThreadPoolExecutor实现了AbstractExecutorService。其实就是核心的execute方法。这里面用到了BlockingQueue（TODO 常用的blockingqueue实现）。里面还用到了ReentrantLock（TODO juc lock分析&condition）。shutdown这个逻辑采用了java的中断（详解java线程中断&best practice）。如果线程池的执行逻辑抛出了异常，那么这个线程会终止，线程池会新建一个线程来替补。
   5. ScheduledExecutorService接口扩展了AbstractExecutorService这个接口。提供了schedule系列方法，可以周期和间隔的执行任务。
   6. ScheduledThreadPoolExecutor实现了ScheduledExecutorService接口，提供了具体的实现，其中线程的最大size是最大整数，所以相当于coresize来决定线程池大小。基本思想是将任务包装成一个ScheduledFutureTask，然后交给线程池执行，执行完毕之后重新计算执行时间并将task入队列。其中delayqueue和delayworkqueue都是基于堆结构来进行实现的（手写）。注意ScheduledFutureTask里面如果任务抛出了异常，那么就不会再进行下一次的调度。
2. 常用的应用场景&best practice：


1. ​


1. ​