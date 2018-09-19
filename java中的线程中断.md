### java中的线程中断



来源：https://praveer09.github.io/technology/2015/12/06/understanding-thread-interruption-in-java/



##### 需求

我们经常碰到使用异步线程来执行长时间耗时的任务，并且我们也希望能够方便的终止任务。比如我们发现任务运行的时间太长，或者我们想要关闭应用程序的时候。



这篇文章解决两个问题：

1. 怎么发送请求终止一个异步线程？
2. 怎么让异步线程响应这个终止请求？



##### 例子

```java
public static void main(String[] args) throws Exception{
 	Thread taskThread = new Thread(taskTahtFinishesEarlyOnInterruption());
    taskThread.start();
    Thread.sleep(3000);
    taskThread.interrupt();//1
    taskThread.join(1000);
}

private static Runnable taskThatFinishesEarlyOnInterruption(){
    return () -> {
        for(int i = 0; i < 10; i++){
            System.out.print(i);
            try{
                Thread.sleep(1000);
            }catch(InterruptedException e){//2
                Thread.currentThread().interrupt();//3
                break;
            }
        }
        if(!Thread.currentThread.isInterrupted()){ //4
            System.out.println("not interrupted !");
        }
    }
}
```

1. 在java中，一个线程是不能终止另外一个线程的，一个线程只能发送终止请求给另外一个线程。终止请求发送的方式是通过终端的方式。在thread对象上调用interrupt方法可以设置对应thread对象的中断状态为true
2. 在java中，基本上所有的阻塞方法都会响应中断，通过抛出interruptException异常。
3. 在阻塞方法抛出异常前，它会清除掉线程的中断状态，所以我们需要在处理异常的时候重新将线程的中断标记设置为true。这样线程的剩余的执行逻辑才可以感知到线程中断。
4. 可以尝试把3注释掉，然后查看输出





##### 总结



1. 我们通过调用thread对象的interrupt方法来尝试终止一个线程（设置该线程的中断标记）
2. 两种情况，如果代码中有阻塞的代码，那么需要尝试catch住InterruptException，并且处理异常的时候需要重新设置中断标记。如果代码中没有阻塞代码，为了能够顺利终止线程，需要在业务逻辑中加入线程是否被中断的判断。