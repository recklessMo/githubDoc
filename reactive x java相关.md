### reactive x java相关



观察者模式observer  observable



An advantage of this approach is that when you have a bunch of tasks that are not dependent on each other, you can start them all at the same time rather than waiting for each one to finish before starting the next one — that way, your entire bundle of tasks only takes as long to complete as the longest task in the bundle. 好处就是你可以同时开启多个任务。因为每个任务都异步的执行，执行完毕之后执行callback就行了。







