### ThreadLocalRandom的坑

```java
	static ThreadLocalRandom random = ThreadLocalRandom.current();

	public static void main(String[] args) throws Exception {
//		SpringApplication.run(MoneyApplication.class, args);


		new Thread(() -> {
//			ThreadLocalRandom.current();
			for(int i = 0; i < 10; i++){
				System.out.print(ThreadLocalRandom.current().nextLong(0, 1000));
				System.out.print(" ");
			}
			System.out.println("");
		}).start();

		Thread.sleep(1000);

		new Thread(() -> {
//			ThreadLocalRandom.current();
			for(int i = 0; i < 10; i++){
				System.out.print(ThreadLocalRandom.current().nextLong(0, 1000));
				System.out.print(" ");
			}
		}).start();
     }
```





```java

	static ThreadLocalRandom random = ThreadLocalRandom.current();

	public static void main(String[] args) throws Exception {
//		SpringApplication.run(MoneyApplication.class, args);


		new Thread(() -> {
			ThreadLocalRandom.current();
			for(int i = 0; i < 10; i++){
				System.out.print(random.nextLong(0, 1000));
				System.out.print(" ");
			}
			System.out.println("");
		}).start();

		Thread.sleep(1000);

		new Thread(() -> {
			ThreadLocalRandom.current();
			for(int i = 0; i < 10; i++){
				System.out.print(random.nextLong(0, 1000));
				System.out.print(" ");
			}
		}).start();
	}
```



这样就好理解了，把ThreadLocalRandom理解为一个工具类，实际上ThreadLocalRandom.current是初始化种子的类，所以第一种写法的时候，而且种子是存在Thread类里面的，如果直接用random的话，相当于只是有了一个工具类，但是这个工具类使用的种子还是没有被初始化过的，所以在线程执行的过程中调用了ThreadLocalRandom来进行初始化就行了，就是第二种方式。