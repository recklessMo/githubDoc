### netty服务端启动

##### netty名词

先对netty中出现的几个组件做一个简要的解释：首先必须尝试理解这几个名词，要不然后面容易读不懂

1. channel：底层socket的抽象，代表一个连接的一端
2. channelHandler：对channel上的事件进行处理的对象，比如LogHandler等，当channel上有事件到来的时候就调用channelHandler里面的方法进行处理！
3. pipeline：netty将channelHandler按照双链表的形式组织起来形成pipeline，每个channel都有一个pipeline，处理数据的时候按顺序在channelHandler之间流转！原因是一个channel在收到数据之后，可能需要多种处理，比如记录log，比如数据解码，比如数据过滤等等，所以需要多个channelHandler。
4. channelHandlerContext：3中说道，pipeline把channelHandler组织成双链表，那么肯定有next和prev，netty将这些和业务不相关的字段和channelHandler一起包装成了channelHandlerContext结构。这就是一个辅助结构
5. future和promise：future是异步操作返回的凭证，可以通过future去查询操作是否成功，后续会单独讲解，暂时先当成一个异步操作的返回凭证，凭这个凭证可以去控制异步操作。
6. eventExecutor：一个类似于threadPoolExecutor的线程池，用来执行runnable任务
7. eventLoop：eventLoop是一个特殊的eventExecutor，除了可以执行runnable任务之外，还可以为其注册channel，轮询其上绑定的channel的事件，两者可以灵活调节执行占比。eventLoop比eventExecutor的功能更加复杂



##### netty整体结构图

首先需要了解一下的netty的整体结构图，了解一下netty的整个数据流动的过程

先看看netty的模型如下图：包括了数据流模型和线程模型！



##### netty服务端概述

netty服务端主要有三个职责，一个职责是接受客户端的连接，另外一个职责是接收客户端发送的数据，最后一个是往客户端里面写数据。每个职责netty都采用线程池来进行处理，并且netty将对客户端的读和写这两个职责采用同一个线程池。也就是netty有两个组件来处理三个职责，分别是bossEventLoop和workEventLoop，这两个eventLoop大家可以理解成一个线程池，线程池里面的线程都在不断的死循环进行事件处理，同时由于是eventLoop，所以也有处理事件的能力。具体的分工如下：

1. bossEventLoop里面的线程不断的死循环监听serverSocketChannel上的连接事件，连接到来之后将接收到的socketchannel交给workEventLoop来进行注册

2. workEventLoop里面的线程不断的死循环监听和处理其上所有的socketchannel上的读写事件




##### netty服务端启动流程

netty服务端启动流程可以分为三个流程：根据下面的流程来看代码就很容易理解了。

1. init，创建并且初始化服务端的serverSocketChannel！初始化包括pipeline的设置，还有将channelHandler设置到pipeline上去，这样让pipeline能够根据channel上的事件来进行业务逻辑。
2. register，将ServerSocketChannel注册到bossEventloop上！让boss可以开始处理ServerSocketChannel上的事件，但是此时是没有事件进行处理的，因为还没有给ServerSocketChannel绑定端口
3. bind，将ServerSocketChannel绑定到相应的端口上，只有绑定了才能开始接收相应的连接事件！

所以根据以上三个流程我们看看代码的流程图： 以官方的discardServer为例

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);//bossGroup建立了，用于监听连接
EventLoopGroup workerGroup = new NioEventLoopGroup();//workerGroup用于处理接收到的socketChannel
try {
  ServerBootstrap b = new ServerBootstrap();//Server启动的助手
  b.group(bossGroup, workerGroup)//将两个group设置进去
    .channel(NioServerSocketChannel.class)//设置一下ServerSocketChannel的类型，这里是NIO
    .handler(new LoggingHandler(LogLevel.INFO))//给channel加上一个handler
    .childHandler(new ChannelInitializer<SocketChannel>() {//设置socketChannel的handler
      @Override
      public void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();
        p.addLast(new DiscardServerHandler());
      }
    });

  // Bind and start to accept incoming connections.
  ChannelFuture f = b.bind(PORT).sync();//绑定到本地端口，进行sync等待！

  // Wait until the server socket is closed.
  // In this example, this does not happen, but you can do that to gracefully
  // shut down your server.
  f.channel().closeFuture().sync();//等待关闭
} finally {
  workerGroup.shutdownGracefully();
  bossGroup.shutdownGracefully();
}
```

childHandler的是ServerSocketChannel接收到的socketChannel的handler，childHandler会被插入到socketChannel的pipeline中，前面说过每个channel都有一个pipeline来用于处理其上的各种读写事件！ServerSocketChannel和SocketChannel都有自己的pipeline和handler，所以启动的核心就是正确设置这些参数。

启动的时候主要就是通过创建bootstrap对象来设置其中的一些参数，然后真正开始进行逻辑处理是调用boostrap的doBind方法，如下：

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
        final ChannelFuture regFuture = initAndRegister();//1
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        if (regFuture.isDone()) {//2
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);//4
            return promise;
        } else {//3
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();

                        doBind0(regFuture, channel, localAddress, promise);//4
                    }
                }
            });
            return promise;
        }
    }
```

上面的代码就对应到了我前面说的流程，init & register & bind

1. 上面的流程就是先调用initAndRegister，正好对应前面我说的流程，在1处调用返回的是一个future。这里需要说下，由于netty的设计哲学里面大多都是异步处理，也就是说initAndRegister实际上是交给bossGroup，也就是另外一个线程去执行的，而不是我们现在所在的main线程，netty将一个channel上所有的处理都交给了这个channel归属的线程去执行，这样就不会出现竞争条件。这里channel是对应的NioServerSocketChannel，所以这里对应的group是bossGroup，因为bossGroup负责轮询ServerSocketChannel上的事件，而workGroup负责轮询由ServerSocketChannel创建的socketChannel上的事件！

2. 在main线程中，我们就需要根据1中返回的future来进行判断，判断future是否成功！如果future已经成功了那么我们就直接调用bind流程。

3. 如果future返回还没有成功，说明bossGroup还没来得及对ServerSocketChannel进行register，由于netty的设计原则是对一个channel上的操作要让同一个线程来进行处理，所以就需要采用回调函数的方式，也就是给future设置一个callback，当register对应的future完成之后自动帮我们进行调用bind，这样我们的main线程就可以空出手来继续执行别的请求，而且我们的register和bind就都由对应的bossGroup进行串行处理了。很巧妙。

   下面看下initRegister方法的具体处理过程：

```java
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();//1
            init(channel);//2
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        ChannelFuture regFuture = config().group().register(channel);//3
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;
    }
```

1. 还记得前面DiscardServer中我们给bootstrap里面设置的是ServerSocketChannel对应的class，所以第一步就需要创建ServerSocketChannel。

2. init方法是一个抽象方法，实际是在子类ServerBootstrap中实现的。目的就是初始化对应的ServerSocketChannel，设置pipeline等，后面会分析如何建立一个责任链模式的pipeline。

3. ServerSocketChannel既然都已经准备好了，那么就可以register到线程了，记得我们将group设置成了bossGroup，所以我们就是将ServerSocketChannel注册到了bossGroup上，根据netty能异步就异步的特点，这里就返回了future，代表这个事情是异步操作，操作完成后再设置回调！

   ​

下面看一下init方法流程：这个方法很重要，因为这个方法是设置ServerSocketChannel，只有ServerSocketChannel正确的设置，后续才能处理读写事件，才能正确的创建socketChannel。

```java
    @Override
    void init(Channel channel) throws Exception {
        final Map<ChannelOption<?>, Object> options = options0();
        synchronized (options) {
            setChannelOptions(channel, options, logger);//1
        }

        final Map<AttributeKey<?>, Object> attrs = attrs0();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                channel.attr(key).set(e.getValue());//2
            }
        }

        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
        }
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
        }

        p.addLast(new ChannelInitializer<Channel>() {//3
            @Override
            public void initChannel(final Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```

1. 1处主要是设置一些ServerSocketChannel的属性，例如tcpnodelay，buffer这些，一般跟底层的socket的选项差不多。
2. 设置attribute，实际上就是ServerSocketChannel对象里面有一个map，设置一些属性进去，用的比较少！
3. 然后开始了重要的一环，设置pipeline，前面根据数据流来看，channel收到数据是由下向上由pipline进行处理的，channel写数据则是由上到下由pipline进行处理。这样设计的好处是我们很容易自己组合自己需要的功能。我们只需要将合适的handler注入到pipeline中即可。在代码3里，这里给pipeline添加了一个比较特殊的类，叫ChannelInitializer，这看起来不像是一个handler，但是实际上就是一个特殊的handler，下面具体分析。

```java
public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter 

public class ChannelInboundHandlerAdapter extends ChannelHandlerAdapter implements ChannelInboundHandler {

```

可以看到ChannelInitializer确实是一个handler

那么它的initChannel方法是什么时候被调用的呢？实际上是在ChannelInitializer的channelRegistered方法中调用。这个channelRegistered会在register注册成功之后调用到。

```java
    @Override
    @SuppressWarnings("unchecked")
    public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        // Normally this method will never be called as handlerAdded(...) should call initChannel(...) and remove
        // the handler.
        if (initChannel(ctx)) {//1
            // we called initChannel(...) so we need to call now pipeline.fireChannelRegistered() to ensure we not
            // miss an event.
            ctx.pipeline().fireChannelRegistered();//2
        } else {
            // Called initChannel(...) before which is the expected behavior, so just forward the event.
            ctx.fireChannelRegistered();//3
        }
    }

```

1. 在代码1处调用initChannel方法。而这个channelRegistered方法是被谁来调用呢？ 这需要详细解释一下pipeline的工作机制了，前面我们说到在pipeline中会有很多个事件流转，比如read事件，write事件，register的事件等等。pipeline实际上是以双链表的形式来将多个handler组织在一起的。那么就存在handler如何调用下一个handler的问题了
2. 在netty中实际上是通过2处的代码来进行调用下一个hanlder的方法的。也就是说在上一个handler执行完毕之后需要主动的去调用下一个handler的相应方法。仔细一看，其实这里有两种调用方式，2处的代码是通过pipeline来调用的，而3处的代码是通过channelHandlerContext来调用的，这两者有什么区别呢？这里我们用一个图来说明者这几个类的关系

图

首先我需要说一下为什么我们需要ChannelHandlerContext这个东西，前面说过。pipeline将所有的handler组织成了一个双链表，那么也就是说上一个handler需要有个next指针指向下一个handler，下一个handler需要有个prev指针指向上一个handler，我们平时自己写双链表，可能会把next和prev直接放到hanlder里面，但是netty没有，为了保证handler职责的单一性，handler只需要处理相应的事件就行。具体的next和prev，netty在外面给你包一层，这个思想在linux里面也有。也是写双链表一个比较常见的方式。看看代码

```java
abstract class AbstractChannelHandlerContext extends DefaultAttributeMap
        implements ChannelHandlerContext, ResourceLeakHint {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(AbstractChannelHandlerContext.class);
    volatile AbstractChannelHandlerContext next;//1
    volatile AbstractChannelHandlerContext prev;//2

    private static final AtomicIntegerFieldUpdater<AbstractChannelHandlerContext> HANDLER_STATE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(AbstractChannelHandlerContext.class, "handlerState");

    private static final int ADD_PENDING = 1;
    private static final int ADD_COMPLETE = 2;
    private static final int REMOVE_COMPLETE = 3;
    private static final int INIT = 0;
    private final boolean inbound;//3
    private final boolean outbound;//4
    private final DefaultChannelPipeline pipeline;//5
    private final String name;//6
    private final boolean ordered;//7
    final EventExecutor executor;//8
    private ChannelFuture succeededFuture;
   }
```

1. next，指向下一个context
2. prev，指向上一个context
3. pipline里面有两个方向的数据流，但是netty将他们放在了一个链表里面，所以我们需要inbound和outbound来进行区分哪些处理出方向，哪些处理入方向。
4. 同3
5. 属于哪个pipline
6. handler名称
7. 顺序
8. 这个比较重要。正常情况下一个handler调用下一个handler，都是顺序的由一个线程来执行的。但是netty可以给channelhandlercontext设置一个executor，也就是设置一个线程。这样当上一个handler调用下一个handler时，如果这个handler设置了executor，那么将由这个executor来执行对应的handler方法！具体代码如下：

```java
    static void invokeChannelRegistered(final AbstractChannelHandlerContext next) {
        EventExecutor executor = next.executor();//1
        if (executor.inEventLoop()) {//2
            next.invokeChannelRegistered();
        } else {
            executor.execute(new Runnable() {//3
                @Override
                public void run() {
                    next.invokeChannelRegistered();
                }
            });
        }
    }
```

1. next就是下一个handler
2. 如果当前handler的线程和下一个handler的线程是同一个的话，那么直接执行就可以了。
3. 如果线程不是同一个，那么就需要提交一个runnable，异步执行。读到这里，可能大家对executor比较好奇，netty有自己的一套实现机制，但是目前我们就先把executor简单的当成一个异步线程，runnable直接就交给了异步线程来执行。后续再详细介绍netty线程池机制！

OK，明白了channel，channelHandlerContext和channel pipeline的关系，那么我们回到ChannelInitializer上去，ChannelInitializer是个handler，这个handler会在channelRegistered方法里面被调用。调用的时候呢会调用我们提供的initChannel方法，在initChannel方法里面，我们给pipeline增加了一些handler。然后我们调用了pipeline的fireChannelRegistered方法，这个方法我前面分析过，它会从pipeline的第一个handler开始，依次顺序的调用pipeline里面所有相关handler的channelRegistered方法，那么我们新增加的handler的channelRegistered方法就会被调用一遍，就和我们从一开始就添加了这些handler一样！这样我们通过这种ChannelInitializer的方式，就可以动态的配置pipeline里面的handler。这就是好处！

可能有人问那ChannelInitializer这个handler就一直在这里吗，对，没有移除，只是给它加了一个执行标记，保证只执行一遍，也就是保证handler只被添加一遍！

注意在ChannelInitializer最后，在链表的最尾部添加了一个handler，如下

```java
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                ch.eventLoop().execute(new Runnable() {//1
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs)); //2
                    }
                });
            }
        });
```

1. 在代码1处为什么要用异步呢？同步其实也能正常工作吧，因为netty设计者考虑到，config.handler可能被用户设置成一个ChannelInitializer，那么就会造成ServerBootstrapAcceptor不是双链表的最后一个handler了，这样就会出现问题！一定要保证ServerBootstrapAcceptor是双链表的最后一个handler。
2. 在代码1处添加了一个ServerBootstrapAcceptor的类，这个类是什么作用呢？ 如下：

```java
    private static class ServerBootstrapAcceptor extends ChannelInboundHandlerAdapter {

        private final EventLoopGroup childGroup;
        private final ChannelHandler childHandler;
        private final Entry<ChannelOption<?>, Object>[] childOptions;
        private final Entry<AttributeKey<?>, Object>[] childAttrs;
        private final Runnable enableAutoReadTask;

        ServerBootstrapAcceptor(
                final Channel channel, EventLoopGroup childGroup, ChannelHandler childHandler,
                Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
            this.childGroup = childGroup;
            this.childHandler = childHandler;
            this.childOptions = childOptions;
            this.childAttrs = childAttrs;

            // Task which is scheduled to re-enable auto-read.
            // It's important to create this Runnable before we try to submit it as otherwise the URLClassLoader may
            // not be able to load the class because of the file limit it already reached.
            //
            // See https://github.com/netty/netty/issues/1328
            enableAutoReadTask = new Runnable() {
                @Override
                public void run() {
                    channel.config().setAutoRead(true);
                }
            };
        }

        @Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {//1
            final Channel child = (Channel) msg;

            child.pipeline().addLast(childHandler);

            setChannelOptions(child, childOptions, logger);

            for (Entry<AttributeKey<?>, Object> e: childAttrs) {
                child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }

            try {
                childGroup.register(child).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }

        private static void forceClose(Channel child, Throwable t) {
            child.unsafe().closeForcibly();
            logger.warn("Failed to register an accepted channel: {}", child, t);
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            final ChannelConfig config = ctx.channel().config();
            if (config.isAutoRead()) {
                // stop accept new connections for 1 second to allow the channel to recover
                // See https://github.com/netty/netty/issues/1328
                config.setAutoRead(false);
                ctx.channel().eventLoop().schedule(enableAutoReadTask, 1, TimeUnit.SECONDS);
            }
            // still let the exceptionCaught event flow through the pipeline to give the user
            // a chance to do something with it
            ctx.fireExceptionCaught(cause);
        }
    }
```

1. 注意在代码1处，channelRead方法，这个handler是属于ServerSocketChannel的handler，因此这个handler的channelRead方法处理的是ServerSocketChannel上的读事件，ServerSocketChannel上的读事件那就只有连接建立的事件了，也就是这时候读取到的实际上就是一个新的socketChannel，那么实际上这个handler就是来处理新建立的socketChannel，包括初始化，设置参数，然后交给workGroup去接管！至于底层read事件过来的处理流程，是如何产生这个SocketChannel的，这个后续我们再说。

在前面主要分析了init和register的流程，现在的状态是ServerSocketChannel已经成功建立，初始化，并且注册到了eventloop上，已经可以成功接收处理事件了，并且接收到连接事件后怎么处理接收到的socketChannel也已经写好了，但是此时还接收不到事件，因为我们没有给ServerSocketChannel绑定端口，下一步就是绑定端口。

```java
    private static void doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {

        // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
        // the pipeline in its channelRegistered() implementation.
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                    promise.setFailure(regFuture.cause());
                }
            }
        });
    }
```

同样，绑定端口也是交给了eventLoop去做，保证channel上几个操作都是串行进行的。



经过前面的分析，整个server的启动流程就确定了，实际上就是围绕着ServerSocketChannel的init，register和bind来进行的。而且由于netty是异步进行执行操作，所以netty需要保证这些操作的顺序，也就是你必须得先init和register没问题之后再bind，否则先bind得话ServerSocketChannel还没准备好，没法处理请求。

## 这篇文章主要分析了，netty是如何进行服务端的启动的，但是还有很多细节需要下一步进行分析