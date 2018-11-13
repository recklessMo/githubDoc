### java类加载机制





1. 双亲委派模型，避免类的重复加载，而且避免jdk的核心类被业务代码覆盖。
2. jvm在判定两个class是否相同时，除了比较class的内容和包名之外，还需要检测这两个class是否被同一个classloader加载了，如果是不同的classloader那么就是不同的。
3. bootstrap(jre/lib/rt.jar)->extension(jre/lib/ext/*.jar)->app(classpath下的jar)->custom(webapp classloader)
4. java提供的classloader只加载指定目录下的jar和class，如果想加载其它位置的类和jar时，比如网络等。
5. 主要有loadclass和findclass两个类型的方法，loadclass是递归的向上搜索，findclass是在当前classloader的路径下查找类。所以实现自己的classloader一般需要覆盖findclass方法。
6. 用tomcat举例，tomcat定义了自己独立的classloader用于加载webapp下的jar



某个类在引用另外一个类的时候，会通过当前类所在的classloader去加载，在加载的时候就会采用双亲委托模型。

































