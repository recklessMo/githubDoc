###netty的pipeline

这篇主要讲一下netty里面的pipeline的创建和工作流程。

##### pipeline结构



​					----------------------channelPipeline-----------------------

----channelHandlerContext #1----channelHandlerContext #2 ----channelHandlerContext #3



channelPipeline里面的每个节点是channelHandlerContext结构，每个channelHandlerContext里面包含了next和prev指针和channelHandler结构



##### 创建pipeline

每个channel都有自己的pipeline对象，比如我们上篇文章讲过的netty启动里面，我们创建了服务端NioServerSocketChannel对象，这个对象在创建的时候，如果跟踪它的代码，它在初始化的时候会调用下面的方法创建一个pipeline

```java
    /**
     * Returns a new {@link DefaultChannelPipeline} instance.
     */
    protected DefaultChannelPipeline newChannelPipeline() {
        return new DefaultChannelPipeline(this);
    }
```

上面的代码将pipeline和channel串联到了一起，下面具体看defaultPipeline

```java
    protected DefaultChannelPipeline(Channel channel) {
        this.channel = ObjectUtil.checkNotNull(channel, "channel");
        succeededFuture = new SucceededChannelFuture(channel, null);
        voidPromise =  new VoidChannelPromise(channel, true);

        tail = new TailContext(this);
        head = new HeadContext(this);

        head.next = tail;
        tail.prev = head;
    }
```

上面的代码看出，建立一个空链，head和tail是两个预先定义好的channelHandlerContext，一会分析具体内容。

下面再看pipeline的常用操作，比如添加删除等等，先看addFirst方法

```java
 @Override
    public final ChannelPipeline addFirst(EventExecutorGroup group, String name, ChannelHandler handler) {//1
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
            checkMultiplicity(handler);
            name = filterName(name, handler);//2

            newCtx = newContext(group, name, handler);//3

            addFirst0(newCtx);//4

            // If the registered is false it means that the channel was not registered on an eventloop yet.
            // In this case we add the context to the pipeline and add a task that will call
            // ChannelHandler.handlerAdded(...) once the channel is registered.
            if (!registered) {//5
                newCtx.setAddPending();
                callHandlerCallbackLater(newCtx, true);
                return this;
            }

            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {
                newCtx.setAddPending();
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        callHandlerAdded0(newCtx);
                    }
                });
                return this;
            }
        }
        callHandlerAdded0(newCtx);
        return this;
    }
```

1. 为什么有个executorGroup参数呢？实际上是因为channelPipeline允许在不同的线程里面执行handler，所以可以提供一个executorGroup参数
2. 检查一下名字是否有重复
3. 新建一个channelHandlerContext，下面说
4. 新添加到链表的头部，没啥可说的

下面接着详细说一下3中的newContext方法

```java

    private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
        return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
    }
```

可以看到实际上就是新建了一个defaultChannelHandlerContext对象。这个对象包含了handler这个对象。

到这里我们看到pipeline的创建就完成了，我们通过addFirst这个例子明白了整个pipeline是怎么被一步步的构建起来的。

下面我们来看看如何使用这些pipeline。

##### pipeline的使用

pipeline的作用是为了组合多个channelHandler形成一条责任链，pipeline上的每个节点都负责某一部分的功能，比如log，codec等等

我们看看如何触发pipeline的流转

```java
public interface ChannelPipeline
        extends ChannelInboundInvoker, ChannelOutboundInvoker, Iterable<Entry<String, ChannelHandler>> {}
```

可以看到channelPipeline这个接口是继承了ChannelInboundInvoker和ChannelOutBoundInvoker的，也就是说channelPipeline具有发起inbound事件和outbound事件的能力！

下面分别就inbound事件和outbound事件进行举例。

###### inbound事件

什么是inbound事件呢？简单来说，就是底层socket发生了变化，需要通知到业务处理程序，就会产生一个inbound事件，通过pipeline由下至上传递到业务处理程序中。这就是inbound事件，比如register成功，读取到了数据等等，比如

```java
    @Override
    public final ChannelPipeline fireChannelRegistered() {
        AbstractChannelHandlerContext.invokeChannelRegistered(head);
        return this;
    }
```

可以看到实际上调用的静态方法，如下： 也就是说inbound事件从head开始沿着pipeline一直调用，

```java
    static void invokeChannelRegistered(final AbstractChannelHandlerContext next) {
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelRegistered();
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelRegistered();
                }
            });
        }
    }

    private void invokeChannelRegistered() {
        if (invokeHandler()) {
            try {
                ((ChannelInboundHandler) handler()).channelRegistered(this);
            } catch (Throwable t) {
                notifyHandlerException(t);
            }
        } else {
            fireChannelRegistered();
        }
    }
```

上面可以看到，需要判断executor.inEventLoop()，如果是在当前的eventLoop中就直接调用方法，否则提交一个任务让executor去执行。

可以看到，直接从pipeline上调用fireChannelRegistered方法的时候，是会从head开始一直沿着整个pipeline去调用的，如果想从中间来调用，可以直接调用某个channelHandlerContext的方法。channelHandlerContext也是可以触发inbound和outbound事件的

```java
abstract class AbstractChannelHandlerContext extends DefaultAttributeMap
        implements ChannelHandlerContext, ResourceLeakHint {}

public interface ChannelHandlerContext extends AttributeMap, ChannelInboundInvoker, ChannelOutboundInvoker {}

```

所以可以调用channelHandlerContext的fireChannelRegistered方法，这样可以灵活的在整个pipeline上进行控制，可以控制从哪个channelHandlerContext开始往后调用！

```java
    @Override
    public ChannelHandlerContext fireChannelRegistered() {
        invokeChannelRegistered(findContextInbound());
        return this;
    }
```



##### pipeline交互

上面介绍了pipeline内部的实现机理，如何创建，以及如何流转inbound事件和outbound事件。下面需要介绍pipeline如何和别的组件进行交互了

上面说到，pipeline的一端是底层的channel，另外一端是业务逻辑部分。那么pipeline是如何与这两部分进行交互的呢

实际上就是通过pipeline预先定义好的两个channelHandlerContext，分别是headContext和tailContext。

###### headContext

```java

    final class HeadContext extends AbstractChannelHandlerContext
            implements ChannelOutboundHandler, ChannelInboundHandler {

        private final Unsafe unsafe;//1 unsafe是把channel里面的操作封装了一下的结构

        HeadContext(DefaultChannelPipeline pipeline) {
            super(pipeline, null, HEAD_NAME, false, true);
            unsafe = pipeline.channel().unsafe();
            setAddComplete();
        }

        @Override
        public ChannelHandler handler() {
            return this;
        }

        @Override
        public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
            // NOOP
        }

        @Override
        public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
            // NOOP
        }

        @Override
        public void bind(
                ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)
                throws Exception {
            unsafe.bind(localAddress, promise);
        }

        @Override
        public void connect(
                ChannelHandlerContext ctx,
                SocketAddress remoteAddress, SocketAddress localAddress,
                ChannelPromise promise) throws Exception {
            unsafe.connect(remoteAddress, localAddress, promise);
        }

        @Override
        public void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
            unsafe.disconnect(promise);
        }

        @Override
        public void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
            unsafe.close(promise);
        }

        @Override
        public void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
            unsafe.deregister(promise);
        }

        @Override
        public void read(ChannelHandlerContext ctx) {
            unsafe.beginRead();
        }

        @Override
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
            unsafe.write(msg, promise);
        }

        @Override
        public void flush(ChannelHandlerContext ctx) throws Exception {
            unsafe.flush();
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            ctx.fireExceptionCaught(cause);
        }

        @Override
        public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
            invokeHandlerAddedIfNeeded();
            ctx.fireChannelRegistered();
        }

        @Override
        public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelUnregistered();

            // Remove all handlers sequentially if channel is closed and unregistered.
            if (!channel.isOpen()) {
                destroy();
            }
        }

        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelActive();

            readIfIsAutoRead();
        }

        @Override
        public void channelInactive(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelInactive();
        }

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            ctx.fireChannelRead(msg);
        }

        @Override
        public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelReadComplete();

            readIfIsAutoRead();
        }

        private void readIfIsAutoRead() {
            if (channel.config().isAutoRead()) {
                channel.read();
            }
        }

        @Override
        public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
            ctx.fireUserEventTriggered(evt);
        }

        @Override
        public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelWritabilityChanged();
        }
    }
    
```

1. 前面说到headContext是一个和channel进行交互的context，所以对于headContext来说，outbound事件的话就可以直接调用headContext的unsafe来执行
2. inbound事件直接沿着pipeline进行传递就可以了，可以看到直接调用fire***方法

下面来看看tailContext，tailContext简单很多，直接忽略所有的inbound事件，为什么呢？因为在netty里面，我们的业务逻辑也是通过channelHandler来完成的，而我们的channelHandler也是要插入到pipeline里面去的，并且在tailContext的前面，所以对于inbound事件来说tailContext并不需要做什么处理。相反，对于outbound事件，tailContext却需要传递下去



```java
final class TailContext extends AbstractChannelHandlerContext implements ChannelInboundHandler {

        TailContext(DefaultChannelPipeline pipeline) {
            super(pipeline, null, TAIL_NAME, true, false);
            setAddComplete();
        }

        @Override
        public ChannelHandler handler() {
            return this;
        }

        @Override
        public void channelRegistered(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void channelUnregistered(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void channelInactive(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void handlerAdded(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void handlerRemoved(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
            // This may not be a configuration error and so don't log anything.
            // The event may be superfluous for the current pipeline configuration.
            ReferenceCountUtil.release(evt);
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            onUnhandledInboundException(cause);
        }

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            onUnhandledInboundMessage(msg);
        }

        @Override
        public void channelReadComplete(ChannelHandlerContext ctx) throws Exception { }
    }


```

所以说pipeline和channel的交互通过headContext来完成，和业务逻辑的交互就是通过把业务逻辑封装成channelHandler并且插入到pipeline中来进行！



#### 总结一下，pipeline的创建，使用和交互都是很重要的一部分内容，理解了pipeline就理解了整个netty的数据流！











