### java中锁的整理



https://zhuanlan.zhihu.com/p/50098743?utm_source=qq&utm_medium=social&utm_oi=38285766295552



1. 读写锁的实现
2. 锁的实现





1. 悲观锁和乐观锁，悲观锁是在执行的时候都会锁住共享资源，乐观锁是在更改的时候比较预期值和内存值是否一致，如果一致的话把目标值设置进去。乐观锁在java中是通过unsafe的cas操作实现的，具体就是一个cpu指令cmpxchg，属于原子操作
   1. 原子类的自增就是通过cas自旋+volatile实现的
   2. cas操作存在ABA问题，可以通过版本号来解决，也可以通过AtomicStampedReference来解决
   3. cas操作可能会长时间自旋
   4. 基础的cas只能保证对一个变量的原子修改，所以可以通过AtomicReference来解决。
2. ​

