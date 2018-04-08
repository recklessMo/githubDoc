### java中的日志

一直没有理清slf4j和log4j之间的关系

commons logging和slf4j都是java的日志门面。log4j和logback是底层实现

https://my.oschina.net/u/2245029/blog/507761



slf4j如果使用log4j来实现的话还需要加入一个适配器进行适配。



如果使用tomcat的话，那么在filter里面产生的异常如果没catch得话就抛到了tomcat去了，tomcat不一定能打印完整的stack trace，tomcat有自己的日志框架。所以最好的方式是将filter里面的异常自己来进行catch控制打印。使用自己的日志框架