### spring bean如何初始化

一直没有很好的理清这块，趁着学习aop的时候进行梳理。



好奇的点:

1. 如何处理循环依赖，（构造器的循环依赖无法解决，因为必须构造器完成之后才能获取对象的引用）
2. 通过transactional标签对对象进行增强之后，是如何注入到容器之中的。
3. spring bean是如何动态注入的，比如mybatis通过mapper接口生成了动态的类，并且可以通过@Resource引入
4. @Resource和@Autowired的区别



https://www.jianshu.com/p/a7b61cf13a8a

















