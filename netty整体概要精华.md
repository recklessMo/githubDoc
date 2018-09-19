### netty整体概要精华

代码结构来看包括几个部分：

buffer：提供buffer的操作，主要是一些buffer内容

codec：提供常见协议的编码解码操作

common：提供一些工具类，比如executorGroup和fastthreadlocal

handler：提供了一些常见的handler，比如日志，流控，超时等等

microbench：提供了jmh的测试

transport：传输层的一些实现，底层的socket那些，pipline，channel，channelHandlerContext





channelInboundHandlerAdapter和channelOutboundHandlerAdapter主要是用来封装netty为我们提供的事件，可以具体展开来各种事件，是如何抽象的！具体的比如exceptionCaught需要特别说明一下，需要找例子说明handler的每个方法具体作用是什么！handler单独一篇



channelHandlerContext和channelPipeline一篇。



byteBuf的分配，释放，回收，缓存机制！为什么要release而不是直接交给gc进行回收



serverBootstrap和bootstrap两个help类是如何帮助启动的，这个必须得在整个模型之后进行解释。



handler如何半包和粘包的情况呢？byteToMessageDecoder（cumulative buffer）



netty中的异步思想如何思想：future 和 promise，这里面的泛型需要好好理解一下（ok）



netty中的fastThreadLocal，引出threadLocal的实现原理，然后里面的weakreference的原理，引出到各种reference的实现



netty中自己实现了一套线程池，而且分配任务的时候是通过绑定的方式来的，这套线程池比较适合现实生活中需要通过key保证顺序的场景！— 同时还可以扩展到java的线程池实现，如何实现定时任务的。



netty中的小技巧：

1. 如何打乱一个list，让其概率相等。





