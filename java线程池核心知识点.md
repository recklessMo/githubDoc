### java线程池核心知识点

1. 实现的层次是：
   1. Executor接口提供了execute方法，可以执行一个runnable对象。
   2. ExecutorService接口扩展了Executor接口，提供了除了执行之外的更多方法，包括shutdown，submit系列方法（可以拿到执行的凭证，可以通过凭证来查询执行的情况）以及invoke系列方法（调用多个任务invokeAll，或者调用多个任务中的一个invokeAny）
   3. AbstractExecutorService实现了ExecutorService接口。实现了submit方法（通过将runnable和callable转换为runnableFuture对象futureTask）。实现了invokeAny，通过ExecutorCompletionService这个结构包装了一下，然后等待队列不为空（同时可以设置超时），这样打到invokeAny的效果。实现了invokeAll，将任务都进行execute，然后等待某个任务完成，需要不断的更新超时时间。
   4. ThreadPoolExecutor实现了AbstrackExecutorService。其实就是核心的execute方法。这里面用到了BlockingQueue（TODO 常用的blockingqueue实现）。里面还用到了ReentrantLock（TODO juc lock分析&condition）。shutdown这个逻辑采用了java的中断（详解java线程中断&best practice）。


1. ​


1. ​