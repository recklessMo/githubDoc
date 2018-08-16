

channelInboundHandlerAdapter和channelOutboundHandlerAdapter主要是用来封装netty为我们提供的事件，可以具体展开来各种事件，是如何抽象的！具体的比如exceptionCaught需要特别说明一下，需要找例子说明每个handler的方法具体作用是什么！



byteBuf的分配，释放，回收，缓存机制！为什么要release而不是直接交给gc进行回收



serverBootstrap和bootstrap两个help类是如何帮助启动的，这个必须得在整个模型之后进行解释。



handler如何半包和粘包的情况呢？byteToMessageDecoder（cumulative buffer）



netty中的异步思想如何思想：future 和 promise，这里面的泛型需要好好理解一下、









