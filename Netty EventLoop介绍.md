### Netty EventLoop介绍

这篇文章在上一篇EventExecutor的基础上介绍一下EventLoop，EventLoop在EventExecutor的基础上增加了Loop的功能，这里的Loop就是指事件循环，以NIO为例，我们写NIO程序都会在有一个while循环，不断的监听channel上的事件，所以EventLoop是具有事件循环的功能的EventExecutor，既可以执行事件循环监听，也可以执行runnable任务！

照例先介绍一下组件：

1. EventLoop，前面说过了
2. EventLoopGroup，EventLoop的集合，可以类比EventExecutor和EventExecutorGroup的关系！

我们以NioEventLoop开始，先看继承类图



我们从EventLoop接口开始讲起，可以看到EventLoop继承自EventLoopGroup，就像EventExecutor继承自EventExecutorGroup一样。

```java
public interface EventLoopGroup extends EventExecutorGroup {
    /**
     * Return the next {@link EventLoop} to use
     */
    @Override
    EventLoop next();

    /**
     * Register a {@link Channel} with this {@link EventLoop}. The returned {@link ChannelFuture}
     * will get notified once the registration was complete.
     */
    ChannelFuture register(Channel channel);

    /**
     * Register a {@link Channel} with this {@link EventLoop} using a {@link ChannelFuture}. The passed
     * {@link ChannelFuture} will get notified once the registration was complete and also will get returned.
     */
    ChannelFuture register(ChannelPromise promise);

    /**
     * Register a {@link Channel} with this {@link EventLoop}. The passed {@link ChannelFuture}
     * will get notified once the registration was complete and also will get returned.
     *
     * @deprecated Use {@link #register(ChannelPromise)} instead.
     */
    @Deprecated
    ChannelFuture register(Channel channel, ChannelPromise promise);
}

```

发现EventLoopGroup实际上在EventExcutorGroup的基础上增加了几个register方法。register方法是干什么用的呢？我们说EventLoop是事件循环，它是轮询channel上的数据，所以我们需要有register方法来把channel注册到对应的EventLoop上，这样EventLoop就能监听channel上的事件进行轮询！

下面再看SingleThreadEventLoop类，它继承了SingleThreadEventExecutor并且实现了EventLoop接口，那么需要增加哪些功能呢？

```java
public abstract class SingleThreadEventLoop extends SingleThreadEventExecutor implements EventLoop {

    @Override
    public ChannelFuture register(Channel channel) {
        return register(new DefaultChannelPromise(channel, this));
    }

    @Override
    public ChannelFuture register(final ChannelPromise promise) {
        ObjectUtil.checkNotNull(promise, "promise");
        promise.channel().unsafe().register(this, promise);
        return promise;
    }

    @Deprecated
    @Override
    public ChannelFuture register(final Channel channel, final ChannelPromise promise) {
        if (channel == null) {
            throw new NullPointerException("channel");
        }
        if (promise == null) {
            throw new NullPointerException("promise");
        }

        channel.unsafe().register(this, promise);
        return promise;
    }

}
```

实际上就是实现了这个register方法，但是具体的register调用并没有封装在eventLoop里面，而是交给了channel去处理。这也算是netty的一个策略，将属于channel的方法比如connect，register，read，write这些方法都交给channel自己来处理，职责划分很清楚！

既然上面已经提供了注册功能，那么注册完毕了就可以进行轮询了，我们从前一篇文章的分析知道SingleThreadEventExecutor的子类只需要实现run方法就可以正常运行了，但是SingleThreadEventLoop并没有实现run方法，而是交给了子类执行，所以下面看看NioEventLoop的核心run方法

```java
    @Override
    protected void run() {
        for (;;) {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));

                        // 'wakenUp.compareAndSet(false, true)' is always evaluated
                        // before calling 'selector.wakeup()' to reduce the wake-up
                        // overhead. (Selector.wakeup() is an expensive operation.)
                        //
                        // However, there is a race condition in this approach.
                        // The race condition is triggered when 'wakenUp' is set to
                        // true too early.
                        //
                        // 'wakenUp' is set to true too early if:
                        // 1) Selector is waken up between 'wakenUp.set(false)' and
                        //    'selector.select(...)'. (BAD)
                        // 2) Selector is waken up between 'selector.select(...)' and
                        //    'if (wakenUp.get()) { ... }'. (OK)
                        //
                        // In the first case, 'wakenUp' is set to true and the
                        // following 'selector.select(...)' will wake up immediately.
                        // Until 'wakenUp' is set to false again in the next round,
                        // 'wakenUp.compareAndSet(false, true)' will fail, and therefore
                        // any attempt to wake up the Selector will fail, too, causing
                        // the following 'selector.select(...)' call to block
                        // unnecessarily.
                        //
                        // To fix this problem, we wake up the selector again if wakenUp
                        // is true immediately after selector.select(...).
                        // It is inefficient in that it wakes up the selector for both
                        // the first case (BAD - wake-up required) and the second case
                        // (OK - no wake-up required).

                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                        // fall through
                    default:
                }

                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        runAllTasks();
                    }
                } else {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
```

前面说了很多次EventLoop是带事件循环的EventExecutor，那么它是如何协调两者的执行呢？毕竟两个都是要循环的执行的，loop需要循环执行channel的事件，executor需要循环执行提交到其上的task，核心就是先执行在其上产生的事件，然后根据执行时间和分配比例来确定执行task的时间。使两者的执行时间比率达到ioRatio的值！



NioEventLoop可以说是整个netty里面特别重要的部分了，它是整个netty的核心所在，它轮询所有注册在其上的channel，并且处理channel上的读写事件，并且它还解决了java NIO的bug！



### 经过上面的分析，了解到其实就是在EventExecutor上增加了Loop的功能