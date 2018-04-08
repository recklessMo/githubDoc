### java的多态

1. ​

简要说怎么实现的：



Child -> Person -> Object



Object o = new Child();

o.take();



编译成字节码，实际上是invokeVirtual指令，找到o对应的实际对象，查看他的方法表里面寻找take方法，然后查看对应的项，如果该项不为空，就代表Child进行了重写，所以直接调用该方法，如果该项不为空，证明没有进行重写，所以需要按照继承链往上找Person类的方法表，然后继续寻找take方法。如此循环。这样形成了多态。