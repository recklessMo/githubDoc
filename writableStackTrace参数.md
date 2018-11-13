### java异常相关，writableStackTrace参数，fillInStackTrace方法



1. 看群里有人说对于每一个业务类别的异常都定义一个新的异常类。这些异常类就不填充异常堆栈，这样printstacktrace就不会打印出异常堆栈，这样可以节约异常的性能。
2. 看Rxjava的源码，发现也利用了这个特点。所以看看，了解一下。



有很多业务会尝试通过异常来控制流程的流转，所以可能希望能不记录堆栈。所以做法就是重写父类的fillinstacktrace方法，让这个方法不做任何动作就可以。



有的业务可能想变更异常堆栈信息，也可以主动调用异常的fillinstacktrace方法来填充堆栈。fillinstacktrace方法会根据当前栈内的栈帧情况来存储异常堆栈信息。





主要是异常的实现机制是怎么样的！每个方法都有一个异常表，通过goto指令来控制流程转义

https://my.oschina.net/suemi/blog/852542，如何实现catch和finally里面的情况。理解异常机制必须从字节码角度来理解。



{"large":"https://cdn-usa.skypixel.com/uploads/usa_files/photo/image/72d95370-c849-4817-bd7d-045b3363cfb7.jpeg@!1920","medium":"https://cdn-usa.skypixel.com/uploads/usa_files/photo/image/72d95370-c849-4817-bd7d-045b3363cfb7.jpeg@!1200","small":"https://cdn-usa.skypixel.com/uploads/usa_files/photo/image/72d95370-c849-4817-bd7d-045b3363cfb7.jpeg@!550"}



https://www.zhihu.com/question/21405047



异常的实现机制，从字节码的角度分析。实际上就是把finally块当成一个subroutine来执行，但是执行的时候会保存return的值到局部变量表。



try和catch的多个exception相当于一个程序执行流程的多个分支。

java在编译的时候，会将所有的分支都加上一个finally块的routine。



https://www.ibm.com/developerworks/cn/java/j-lo-finally/index.html 

核心就是athrow指令。return和throw指令。都是一种控制转义的指令。

把异常存到栈顶，然后athrow出去，就会弹出当前栈帧，然后在上层继续执行一样的逻辑。如果有fill stack track这个逻辑，就会爬取当前的栈帧，将调用数据存储到当前的exception中，实际上爬栈获取异常的堆栈信息是一个基本操作，可以自己实现一个异常，将这个逻辑覆盖掉。

