### java jmm & volatile关键字



参考资料：

https://juejin.im/post/5a2b53b7f265da432a7b821c

https://blog.csdn.net/ns_code/article/details/17348313

https://www.jianshu.com/p/b4d4506d3585

https://crowhawk.github.io/2018/02/10/volatile/  volatile底层是如何实现的。

https://www.ibm.com/developerworks/java/library/j-jtp06197/ volatile的一些总结

https://javarevisited.blogspot.com/2014/07/top-50-java-multithreading-interview-questions-answers.html

https://javarevisited.blogspot.com/2017/01/can-we-make-array-volatile-in-java.html

https://monkeysayhi.github.io/2017/12/28/%E4%B8%80%E6%96%87%E8%A7%A3%E5%86%B3%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C/  内存屏障

https://segmentfault.com/a/1190000014315651 内存屏障

https://www.infoq.cn/article/ftf-java-volatile 



####java内存模型概念

1. 有主内存和工作内存的概念。每个线程都有自己的工作内存（工作内存可以看做是对缓存寄存器等的一种抽象概念）。cpu速度很快，内存操作相对来说很慢，之所以可以利用缓存和寄存器来加快cpu的执行速度是基于一个原理：cpu访问过的内存地址，在短时间内很可能会再次被访问（缓存的原理是通用的）
2. 内存模型描述了（给你一个程序和一个程序的执行流，如何描述是否是一个合法的执行流）。对于java来说，内存模型就是定义了一系列happen-before规则，来规范哪些写操作可以立马对后续的读操作可见。规避可见性和有序性的问题。来自umd的slide（https://www.cs.umd.edu/~pugh/java/memoryModel/jsr133.pdf）



####先描述一下多线程并发存在的一些问题

java多线程并发编程的问题：可见性，有序性，

######可见性-可以理解为一致性

1. jmm在工作的时候读数据会从主内存中加载到工作内存中，写数据的时候会写到工作内存中，工作内存的内容在适当的时候被刷新到主内存中。由于存在工作内存，所以就势必有数据一致性的问题，数据什么时候从工作内存刷新到主内存，如果刷新过于频繁会对性能有影响，不刷新的话又可能读到脏数据。所以可见性问题是jmm里面无法避免的情况。

2. 多个线程在操作同一个共享的变量（静态变量，实例变量）的时候就会存在可见性的问题，一个线程的更改怎么被另外一个线程捕捉到。

   ```java
   private static boolean stop = false;

   public void thread_a(){
       while(!stop){
           System.out.println("still running");
       }
   }

   public void thread_b(){
       stop = true;
   }
   ```

   在java中，当thread b运行完成的时候，thread a不一定能够停下来。这就是因为内存可见性导致的，thread a看到的stop值还是false，并没有同步到thread b的最新修改值true。



######重排序

1. java里面有重排序，包括编译重排序和处理器重排序。编译重排序的目的是可以把对同一个变量的操作尽可能放在一起，这样可以尽可能的利用cpu的缓存或者寄存器等，不用来回加载数据。处理器重排序是可以让处理器流水线更加的饱和，现在处理器都是采用指令集并行技术。总而言之，重排序的目的是为了性能。

2. 语句之间存在数据依赖性，如果重排序两条具有数据依赖性的语句，那么将会导致结果发生改变。所以编译器和处理器在重排序的时候都需要遵守as-if-serial语义：也就是重排序不能改变程序的执行结果。也正是因为有as-if-serial语义的存在，我们感觉单线程执行的时候代码是按照编写的顺序来执行的（实际上不是，实际上编译器和处理器都会做很多的优化工作）

3. 重排序在单线程看来是没有问题的，但是在多线程环境下，重排序可能会产生意想不到的情况，可能造成很费解的情况比如下面的代码：

   ```java
   int a = 0;
   boolean flag = false;
   public void thread_a(){
   	a = 1;//1
   	flag = true;//2
   }

   public void thread_b(){
       if(flag){
           System.out.println(a);
       }
   }
   ```

   如果没有重排序的情况下，thread b的打印值只可能是1，但是在有重排序的情况下：2可能被提前到1之前进行执行，那么当flag为true的情况下，a可能还是0.

   ​

####如何解决这些问题呢

从上面描述可以看到，可见性和重排序对我们程序逻辑有很大的影响，可能会造成一些意想不到的结果。所以需要程序员主动来解决这个问题。主要目的就是为了保持可见性和进制重排序

######happen-before原则-一种偏序关系

我们需要jmm来对可见性和重排序进行一些限制，于是引出了happen-before规则，happen-before规则详情如下：如何理解happen-before原则呢？我们说A happen-before B，就是在发生操作B的时候，操作A产生的影响都能被B观察到。影响包括内存中共享变量的修改等。

1. 程序内顺序原则：在单线程内，按照程序代码的执行流（注意是执行流，不是书写流）先执行的操作happen-before后执行的操作。
2. volatile变量原则：你对一个volatile变量的写happen-before与任意后续对这个volatile的读。在读volatile变量的时候，在时间上之前对volatile变量的写都能观察到。
3. 监视器锁原则：对一个监视器的解锁happen-before对这个监视器的加锁。也就是加锁的时候，前面应该有解锁操作了。
4. 传递性规则：a如果happen-before b，b如果happen-before c，那么a就happen-before c。
5. 线程启动原则：thread对象start方法happen-before此线程的任何一个动作，不可能没启动就执行
6. 线程终止原则：线程的所有操作都happen-before对此线程的终止检测，比如thread.join，thread.isAlive等
7. 线程中断原则：线程interrupt方法的调用happen-before于发生检测到中断事件。
8. 对象终结原则：一个对象初始化完成（构造函数执行完毕）happen-before它的finalize方法。

可以总结：在时间上先后发生+某些必要条件（比如锁，volatile等）可以构成happen-before规则，这样就可以规避掉可见性和重排序给我们带来的影响。



#### 利用volatile的happen-before原则

###### 内存屏障-mfence

1. 首先cpu多核之间本来就存在着缓存可见性问题，core0更新了变量值，core1如何能感知到最新的值。这就引出了一个MESI协议，专门为cpu来解决可见性问题。
2. 虽然在cpu层面解决了这个可见性问题，但是还存在重排序的问题，比如编译器优化重排序以及处理器重排序问题。所以还需要解决这个重排序和可见性的问题。
3. 在jvm层面上volatile标记可以解决编译器层面的可见性与重排序问题，而内存屏障则解决了硬件层面的可见性和重排序，不同架构下内存屏障的实现不一样。
4. 内存屏障分为loadload，storestore，loadstore，storeload等四种类型barrier。
5. 内存屏障的性质就是禁止一切跨barrier的重排序并且对数据进行刷新。



###### volatile实现原理

根据原则2，我们可以选择volatile关键字来应用happen-before规则。

1. 在对volatile变量的写之后插入一个store barrier，禁止一切跨store barrier的重排序并且保证store写入主存，在读volatile变量之前插入一个load barrier，禁止一切跨load barrier的重排序并且保证从主内存读到最新的数据。
2. 在x86架构下，barrier的实现是通过lock汇编指令来操作的，lock指令的作用是将cpu的cache写到到内存，同时使别的cpu的cache失效。这样修改就可以对别的可见了。



###### volatile常见的问题

1. volatile 修饰数组？https://stackoverflow.com/questions/5173614/java-volatile-array，实际上只是修饰了数组这个引用，数组的内容并不遵守happen-before规则，可以使用AtomicIntegerArray，AtomicLongArray，AtomicReferenceArray等。volatile语义只保证引用变量本身具有happen-before规则，对引用进行更改可以立马可见。但是对引用对应的对象进行内容的更改是不满足happen-before规则的。
2. volatile reference变量和atomicReference的区别。https://stackoverflow.com/questions/281132/java-volatile-reference-vs-atomicreference，atomicReference相对于volatile reference来说，提供了getAndSet等方法，提供了一些cas操作。



###### volatile用法场景

1. status flag：用于标记

   ```java
   volatile boolean shutdownRequested;
    
    
   public void shutdown() { shutdownRequested = true; }
    
   public void doWork() { 
       while (!shutdownRequested) { 
           // do stuff
       }
   }
   ```

   算是一个共享变量很常见的场景，比如标识状态，结束一个线程等，很多框架都有应用

2. 对象发布的场景，类似的有一个著名的double-checked-locking。

   ```java
   public class BackgroundFloobleLoader {
       public volatile Flooble theFlooble;
    
       public void initInBackground() {
           // do lots of stuff
           theFlooble = new Flooble();  // 1
       }
   }
    
   public class SomeOtherClass {
       public void doWork() {
           while (true) { 
               // do some stuff...
               // use the Flooble, but only if it is ready
               if (floobleLoader.theFlooble != null) 
                   doSomething(floobleLoader.theFlooble);
           }
       }
   }
   ```

   1. 在1这个地方，实际上分为三部分，new一个对象，初始化这个对象，赋值给变量。如果发生了重排序，赋值给变量被提前到了第二步，那么别的线程拿到这个对象后操作就会出问题。所以需要将这个变量定义为volatile类型，这样就可以禁止重排序。

3. 读多写少的单变量更新场景

   ```java
   @ThreadSafe
   public class CheesyCounter {
       // Employs the cheap read-write lock trick
       // All mutative operations MUST be done with the 'this' lock held
       @GuardedBy("this") private volatile int value;
    
       public int getValue() { return value; }
    
       public synchronized int increment() {
           return value++;
       }
   }
   ```

   通过volatile变量保证读的可见性，通过synchronized来保证更新的原子性。读的时候不加锁，而是简单的通过volatile保证可见性即可。可以省掉一些加锁的开销。



































1. ​