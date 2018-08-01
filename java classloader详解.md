### java classloader详解



##### classloader作用

##### classloader继承结构

##### classloader设置，线程上下文classloader

线程上下文类加载器（context class loader）是从 JDK 1.2 开始引入的。类 `java.lang.Thread`中的方法 `getContextClassLoader()`和 `setContextClassLoader(ClassLoader cl)`用来获取和设置线程的上下文类加载器。如果没有通过 `setContextClassLoader(ClassLoader cl)`方法进行设置的话，线程将继承其父线程的上下文类加载器。Java 应用运行的初始线程的上下文类加载器是系统类加载器。在线程中运行的代码可以通过此类加载器来加载类和资源。



https://www.cnblogs.com/faunjoe88/p/8023239.html

https://www.ibm.com/developerworks/cn/java/j-lo-classloader/