### netty EventExecutor介绍

前面介绍过netty中的pipeline，了解了pipeline，我们就理解了netty的数据流。今天我们介绍eventExecutor，了解了eventExecutor就可以了解netty的线程模型，这样就能够全方位的了解netty ！

#### 基本组件概念

理解这篇文章是明白netty线程模型的一个很关键的点！netty没有采用jdk的线程池，而是自己实现了一套线程池的模型。

这篇主要是介绍一下netty的eventExecutor，先说下netty这个部分的几个组件的介绍：

1. eventExecutorGroup： netty的线程池，netty没有采用jdk自带的线程池实现，而是在此基础上自己实现了一套，如下

   ```java
   public interface EventExecutorGroup extends ScheduledExecutorService, Iterable<EventExecutor>
   ```

   可以看到实现了可执行定时任务的线程池接口，而且里面的单个线程模型是EventExecutor，并且实现了iterable接口！

2. eventExecutor： netty中的线程，实质就是对单个线程进行的封装

   ```java
   public interface EventExecutor extends EventExecutorGroup 
   ```

   看到这里可能有点懵，eventExecutor又实现了EventExecutorGroup，实际上netty这里是把EventExecutor当成了一种特殊的EventExecutorGroup来对待，eventExecutor是只有一个线程的eventExecutorGroup。所以EventExecutorGroup和EventExecutor实际上可以认为是一样的功能组件，只不过一个是多线程，一个是单线程。其实EventExecutorGroup的执行也是交给内部的一个EventExecutor去执行，所以本质上在执行工作的是eventExecutor。eventExecutorGroup就是包工头，eventExecutor就是小工！

#### eventExecutorGroup和eventExecutor

###### EventExecutorGroup

先介绍一下eventExecutorGroup，前面说过就是线程池嘛。线程池如果不熟悉的话，可以先去复习一下jdk的线程池，了解一下threadPoolExecutor！我们以默认实现DefaultEventExecutorGroup来一步步讲

结构如下图：



1. 其实从继承类图可以看出EventExecutorGroup实现了ExecutorService接口，EventExecutorGroup是一个线程池，这个线程池并且可以执行定时任务。但是这只是一个接口，所以下一步就是需要有一个实现

2. AbstractEventExecutorGroup实现了这个接口，但是实际上只是对这些方法进行了简单的实现，从下面的代码可以看出eventExecutorGroup就是简单的选择了一个eventExecutor来执行任务。具体选择就是通过next方法。

   ```java
   public abstract class AbstractEventExecutorGroup implements EventExecutorGroup {
       @Override
       public Future<?> submit(Runnable task) {
           return next().submit(task);
       }

       @Override
       public <T> Future<T> submit(Runnable task, T result) {
           return next().submit(task, result);
       }

       @Override
       public <T> Future<T> submit(Callable<T> task) {
           return next().submit(task);
       }

       @Override
       public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
           return next().schedule(command, delay, unit);
       }

       @Override
       public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit) {
           return next().schedule(callable, delay, unit);
       }

       @Override
       public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
           return next().scheduleAtFixedRate(command, initialDelay, period, unit);
       }

       @Override
       public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) {
           return next().scheduleWithFixedDelay(command, initialDelay, delay, unit);
       }

       @Override
       public Future<?> shutdownGracefully() {
           return shutdownGracefully(DEFAULT_SHUTDOWN_QUIET_PERIOD, DEFAULT_SHUTDOWN_TIMEOUT, TimeUnit.SECONDS);
       }

       /**
        * @deprecated {@link #shutdownGracefully(long, long, TimeUnit)} or {@link #shutdownGracefully()} instead.
        */
       @Override
       @Deprecated
       public abstract void shutdown();

       /**
        * @deprecated {@link #shutdownGracefully(long, long, TimeUnit)} or {@link #shutdownGracefully()} instead.
        */
       @Override
       @Deprecated
       public List<Runnable> shutdownNow() {
           shutdown();
           return Collections.emptyList();
       }

       @Override
       public <T> List<java.util.concurrent.Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
               throws InterruptedException {
           return next().invokeAll(tasks);
       }

       @Override
       public <T> List<java.util.concurrent.Future<T>> invokeAll(
               Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException {
           return next().invokeAll(tasks, timeout, unit);
       }

       @Override
       public <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException {
           return next().invokeAny(tasks);
       }

       @Override
       public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
               throws InterruptedException, ExecutionException, TimeoutException {
           return next().invokeAny(tasks, timeout, unit);
       }

       @Override
       public void execute(Runnable command) {
           next().execute(command);
       }
   }
   ```

3. 上面抽象的实现只是提供了一些方法，并没有具体的实现，下面就是一个带有字段的抽象实现了，MultithreadEventExecutorGroup

   ```java
   public abstract class MultithreadEventExecutorGroup extends AbstractEventExecutorGroup {

       private final EventExecutor[] children;//1
       private final Set<EventExecutor> readonlyChildren;
       private final AtomicInteger terminatedChildren = new AtomicInteger();
       private final Promise<?> terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);
       private final EventExecutorChooserFactory.EventExecutorChooser chooser;//2
       
     	protected abstract EventExecutor newChild(Executor executor, Object... args) throws Exception;//3

   }
   ```

   1. 如何管理麾下所有的EventExecutor呢，一个数组。
   2. 当提交任务的时候，如何选择一个EventExecutor来执行任务呢？提供了一个chooser类来进行选择，一般情况下只要依次选择就行了。

4. 有了上面的抽象实现，我们还缺点啥？还缺如何建立一个EventExecutor，下面看看最后一个default实现类

   ```java
   /**
    * Default implementation of {@link MultithreadEventExecutorGroup} which will use {@link DefaultEventExecutor} instances
    * to handle the tasks.
    */
   public class DefaultEventExecutorGroup extends MultithreadEventExecutorGroup {
       /**
        * @see #DefaultEventExecutorGroup(int, ThreadFactory)
        */
       public DefaultEventExecutorGroup(int nThreads) {
           this(nThreads, null);
       }

       /**
        * Create a new instance.
        *
        * @param nThreads          the number of threads that will be used by this instance.
        * @param threadFactory     the ThreadFactory to use, or {@code null} if the default should be used.
        */
       public DefaultEventExecutorGroup(int nThreads, ThreadFactory threadFactory) {
           this(nThreads, threadFactory, SingleThreadEventExecutor.DEFAULT_MAX_PENDING_EXECUTOR_TASKS,
                   RejectedExecutionHandlers.reject());
       }

       /**
        * Create a new instance.
        *
        * @param nThreads          the number of threads that will be used by this instance.
        * @param threadFactory     the ThreadFactory to use, or {@code null} if the default should be used.
        * @param maxPendingTasks   the maximum number of pending tasks before new tasks will be rejected.
        * @param rejectedHandler   the {@link RejectedExecutionHandler} to use.
        */
       public DefaultEventExecutorGroup(int nThreads, ThreadFactory threadFactory, int maxPendingTasks,
                                        RejectedExecutionHandler rejectedHandler) {
           super(nThreads, threadFactory, maxPendingTasks, rejectedHandler);
       }

       @Override
       protected EventExecutor newChild(Executor executor, Object... args) throws Exception {
           return new DefaultEventExecutor(this, executor, (Integer) args[0], (RejectedExecutionHandler) args[1]);
       }
   }
   ```

   可以看到这个类实现了netChild方法，创建了一个DefaultEventExecutor实例！

上面的例子说明了EventExecutorGroup的结构，其实eventExecutor并没有什么特别的地方，只是做了一个EventExecutor的管理和分配，下面说一下核心的EventExecutor。

###### EventExecutor

再来看看EventExecutor的类图，同样的我们以最终的实现DefaultEventExecutor来讲



我们从上往下看，最上面就是EventExecutor的接口，由于EventExecutor是特殊的EventExecutorGroup，所以

这个接口方法不表

```java
public abstract class AbstractEventExecutor extends AbstractExecutorService implements EventExecutor {
    private static final InternalLogger logger = InternalLoggerFactory.getInstance(AbstractEventExecutor.class);

    static final long DEFAULT_SHUTDOWN_QUIET_PERIOD = 2;
    static final long DEFAULT_SHUTDOWN_TIMEOUT = 15;

    private final EventExecutorGroup parent;//1
    private final Collection<EventExecutor> selfCollection = Collections.<EventExecutor>singleton(this);

    protected AbstractEventExecutor() {
        this(null);
    }

    protected AbstractEventExecutor(EventExecutorGroup parent) {
        this.parent = parent;
    }

    @Override
    public EventExecutorGroup parent() {
        return parent;
    }

    @Override
    public EventExecutor next() {
        return this;//2
    }
}
```

1. 可以看到这个抽象的实现有一个parent的字段，这个字段就是指向它所归属的eventExecutorGroup
2. 看到这个next方法返回的是自己，也就是说默认这个eventExecutor是一个只有一个eventExecutor的eventExecutorGroup。
3. 这个类还是继承了AbstractExecutorService类，可以利用到jdk给我提供的submit方法，省去了自己去写的麻烦。java的线程池我们一般都是调用submit方法来提交任务的，同时返回一个future对象，如果不需要future对象的话可以直接调用execute方法。future对象用来获取runnable任务执行的结果！

实际上到了这个类之后，我们的eventExecutor有了submit的方法，但是还没有具体的实现，因为submit方法内部实际上是调用的execute方法，所以我们接着往下看。

接下来我们看它的下一个子类。AbstractScheduledEventExecutor

```java
public abstract class AbstractScheduledEventExecutor extends AbstractEventExecutor {

    Queue<ScheduledFutureTask<?>> scheduledTaskQueue;//1
    
    @Override
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
        ObjectUtil.checkNotNull(command, "command");
        ObjectUtil.checkNotNull(unit, "unit");
        if (initialDelay < 0) {
            throw new IllegalArgumentException(
                    String.format("initialDelay: %d (expected: >= 0)", initialDelay));
        }
        if (period <= 0) {
            throw new IllegalArgumentException(
                    String.format("period: %d (expected: > 0)", period));
        }

        return schedule(new ScheduledFutureTask<Void>(
                this, Executors.<Void>callable(command, null),
                ScheduledFutureTask.deadlineNanos(unit.toNanos(initialDelay)), unit.toNanos(period)));
    }

    @Override
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) {
        ObjectUtil.checkNotNull(command, "command");
        ObjectUtil.checkNotNull(unit, "unit");
        if (initialDelay < 0) {
            throw new IllegalArgumentException(
                    String.format("initialDelay: %d (expected: >= 0)", initialDelay));
        }
        if (delay <= 0) {
            throw new IllegalArgumentException(
                    String.format("delay: %d (expected: > 0)", delay));
        }

        return schedule(new ScheduledFutureTask<Void>(
                this, Executors.<Void>callable(command, null),
                ScheduledFutureTask.deadlineNanos(unit.toNanos(initialDelay)), -unit.toNanos(delay)));
    }

}
```

1. 这个类加了一个queue，用来实现定时的执行
2. 这个类锦上添花了，在AbstractEventExecutor的submit的方法基础上添加了scheduleAtFixedRate和scheduleAtFixedDelay，延迟执行和定时执行的功能。

也就是说我们现在eventExecutor越来越完善了，但是还是没有用，因为还是没有具体的实现，所以接下来看！

下一个类是：SingleThreadEventExecutor，看看关键代码

```java
public abstract class SingleThreadEventExecutor extends AbstractScheduledEventExecutor implements OrderedEventExecutor {

    static final int DEFAULT_MAX_PENDING_EXECUTOR_TASKS = Math.max(16,
            SystemPropertyUtil.getInt("io.netty.eventexecutor.maxPendingTasks", Integer.MAX_VALUE));

    private static final InternalLogger logger =
            InternalLoggerFactory.getInstance(SingleThreadEventExecutor.class);

    private static final int ST_NOT_STARTED = 1;
    private static final int ST_STARTED = 2;
    private static final int ST_SHUTTING_DOWN = 3;
    private static final int ST_SHUTDOWN = 4;
    private static final int ST_TERMINATED = 5;

    private static final Runnable WAKEUP_TASK = new Runnable() {
        @Override
        public void run() {
            // Do nothing.
        }
    };
    private static final Runnable NOOP_TASK = new Runnable() {
        @Override
        public void run() {
            // Do nothing.
        }
    };

    private static final AtomicIntegerFieldUpdater<SingleThreadEventExecutor> STATE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(SingleThreadEventExecutor.class, "state");
    private static final AtomicReferenceFieldUpdater<SingleThreadEventExecutor, ThreadProperties> PROPERTIES_UPDATER =
            AtomicReferenceFieldUpdater.newUpdater(
                    SingleThreadEventExecutor.class, ThreadProperties.class, "threadProperties");

    private final Queue<Runnable> taskQueue;//运行的任务列表

    private volatile Thread thread;
    @SuppressWarnings("unused")
    private volatile ThreadProperties threadProperties;
    private final Executor executor;
    private volatile boolean interrupted;

    private final Semaphore threadLock = new Semaphore(0);
    private final Set<Runnable> shutdownHooks = new LinkedHashSet<Runnable>();
    private final boolean addTaskWakesUp;
    private final int maxPendingTasks;
    private final RejectedExecutionHandler rejectedExecutionHandler;

    private long lastExecutionTime;

    @SuppressWarnings({ "FieldMayBeFinal", "unused" })
    private volatile int state = ST_NOT_STARTED;

    private volatile long gracefulShutdownQuietPeriod;
    private volatile long gracefulShutdownTimeout;
    private long gracefulShutdownStartTime;

    private final Promise<?> terminationFuture = new DefaultPromise<Void>(GlobalEventExecutor.INSTANCE);
    
    
    @Override
    public void execute(Runnable task) {//1
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        if (inEventLoop) {//2
            addTask(task);
        } else {
            startThread();//3
            addTask(task);//4
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
    
  	
    private void doStartThread() {//3
        assert thread == null;
        executor.execute(new Runnable() {//6
            @Override
            public void run() {
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }

                boolean success = false;
                updateLastExecutionTime();
                try {
                    SingleThreadEventExecutor.this.run();//5
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    for (;;) {
                        int oldState = state;
                        if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                                SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                            break;
                        }
                    }

                    // Check if confirmShutdown() was called at the end of the loop.
                    if (success && gracefulShutdownStartTime == 0) {
                        logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must be called " +
                                "before run() implementation terminates.");
                    }

                    try {
                        // Run all remaining tasks and shutdown hooks.
                        for (;;) {
                            if (confirmShutdown()) {
                                break;
                            }
                        }
                    } finally {
                        try {
                            cleanup();
                        } finally {
                            STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                            threadLock.release();
                            if (!taskQueue.isEmpty()) {
                                logger.warn(
                                        "An event executor terminated with " +
                                                "non-empty task queue (" + taskQueue.size() + ')');
                            }

                            terminationFuture.setSuccess(null);
                        }
                    }
                }
            }
        });
    }
    
}
```

如果对线程池不了解的朋友可以看看这篇文章介绍。在线程池实现中，最关键的就是execute方法，其余的比如submit方法，是在execute方法基础上封装了一下，提供了返回future的功能，再比如invokeAll或者invokeAny也是在此基础上提供的一些辅助功能。所以基于jdk Executor接口实现的所有线程池核心就是execute方法！下面我们来看execute方法，这就是看线程池代码的入口，前面两个类的最终实现都是会调用这个方法！

1. execute方法，整个线程池的核心入口

2. 判断一下是不是在eventLoop里面，这个怎么理解？我们可能从另外一个线程调用execute方法，那么此时inEventLoop这个值就是false的，但是我们可能提交了一个task，这个task在eventExecutor里面执行的时候又调用了execute方法，这个时候inEventLoop就是true。

3. startThread方法，最终会调用到上面的doStartThread方法，最终调用到了代码5处的run方法，这个方法是一个抽象方法，需要实现！注意代码6处的一个executor，如果追踪这个来源，这个executor是java自带的方式实现的一个executor，就是一个用来启动线程并且执行的，这样的好处是不用自己去new thread再start。直接调用它的executor方法。实际上跟进去代码如下

   ```java
       @Override
       public void execute(Runnable command) {
           threadFactory.newThread(command).start();
       }
   ```

   可以看到就是帮我新建thread然后调用runnable。

到了上面一步的SingleThreadEventExecutor之后，我们有了一个完成的流程了，给我们一个eventExecutor对象，不管是直接调用execute，还是通过submit等方法调用，我们的流程都能走的通了。但是现在唯一缺一点，就是我们缺一个run方法，缺一个执行的逻辑，怎么样去执行我们的task呢？

这就需要子类DefaultEventExecutor要做的了。代码如下：

```java
public final class DefaultEventExecutor extends SingleThreadEventExecutor {

    public DefaultEventExecutor() {
        this((EventExecutorGroup) null);
    }

    public DefaultEventExecutor(ThreadFactory threadFactory) {
        this(null, threadFactory);
    }

    public DefaultEventExecutor(Executor executor) {
        this(null, executor);
    }

    public DefaultEventExecutor(EventExecutorGroup parent) {
        this(parent, new DefaultThreadFactory(DefaultEventExecutor.class));
    }

    public DefaultEventExecutor(EventExecutorGroup parent, ThreadFactory threadFactory) {
        super(parent, threadFactory, true);
    }

    public DefaultEventExecutor(EventExecutorGroup parent, Executor executor) {
        super(parent, executor, true);
    }

    public DefaultEventExecutor(EventExecutorGroup parent, ThreadFactory threadFactory, int maxPendingTasks,
                                RejectedExecutionHandler rejectedExecutionHandler) {
        super(parent, threadFactory, true, maxPendingTasks, rejectedExecutionHandler);
    }

    public DefaultEventExecutor(EventExecutorGroup parent, Executor executor, int maxPendingTasks,
                                RejectedExecutionHandler rejectedExecutionHandler) {
        super(parent, executor, true, maxPendingTasks, rejectedExecutionHandler);
    }

    @Override
    protected void run() {//1
        for (;;) {
            Runnable task = takeTask();
            if (task != null) {
                task.run();
                updateLastExecutionTime();
            }

            if (confirmShutdown()) {
                break;
            }
        }
    }
}
```

1. 果然这个类帮我们实现了run方法的具体逻辑，具体逻辑很简单，for循环，一直在调用takeTask，这里提一下，taketask，polltask等这些内容都是在父类里面实现的，因为父类中定义了taskqueue，所以相关的操作都放到了父类。

所以到这里，我们整个的执行流程就通了，前面一直帮我做准备工作，提供了submit，invokeAll很多方法的实现（这些方法内部都必须调用execute方法），所以就进一步实现了execute方法，在execute方法中启动了线程来执行逻辑，这个具体逻辑放到子类里面去实现，就是不断的去取任务，执行任务。

于是，我们得到了一个不断进行循环执行任务的单线程的线程池。 这就是netty中的EventExecutor！



###下一节介绍一下netty里面的eventLoop，先理解eventExecutor再来理解eventLoop会方便很多！看到这里你会不会想平时写程序的时候把DefaultEventExecutor拿来用用呢，netty提供的很多组件都可以拿过来自己使用



