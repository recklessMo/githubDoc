### spring的事务处理

1. 在配置事务的时候发现了一个bug，applicationContext和servletContext之间有了冲突，dataSource相关的配置放到了applicationContext里面，但是service这层却放到了servletContext里面，所以说出现了transaction这个标签不生效的情况，经过stackoverflow解决了。解决方法两种，一种是定义都挪到servletContext中，还有就是把<tx:annotation-driven/>这个标签放到servlet的配置里面去，因为是在tx:annotation-driven的时候进行标签的解析和绑定的，所以这个放到servlet层去做就可以生效了。



https://stackoverflow.com/questions/10019426/spring-transactional-not-working

https://stackoverflow.com/questions/11708967/what-is-the-difference-between-applicationcontext-and-webapplicationcontext-in-s

https://stackoverflow.com/questions/29743320/how-exactly-works-the-spring-bean-post-processor

https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop



2



