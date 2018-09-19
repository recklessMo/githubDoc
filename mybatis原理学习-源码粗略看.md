### mybatis原理学习-源码粗略看



1. 启动方式，如何让一个框架和spring进行耦合
2. typeAlias原理
3. typeHandler原理
4. 关于反射和类引用相关
5. 整个执行流程是啥？对于mapper接口上定义的各种方法，会通过动态代理动态生成mapper对应的类，然后最终都会调用sqlsession里面的方法。

