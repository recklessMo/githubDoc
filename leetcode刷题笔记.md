### leetcode刷题笔记

1. ​
2. 2
3. 3
4. 34
5. 6
6. 7
7. 和取模息息相关的是除法，因为y = a*x+b，有三种常见的除法，分别是向上取整，向下取整，向0取整，java采用向0取整，所以在取模的时候可以根据y - (a * x)来计算。比如-3 % 10 -> -3 - (-3 / 10 * 10)以及如何check overflow，check方式可以采用逆向操作来判断原值是否相等，比如sum_new = sum * 10。如何判断sum_new发生了溢出呢？可以采用判断sum_new / 10 是否等于sum来进行判断。切勿不可采用更高级数据类型。也尽量避免采用string。同时可以看看Integer的toString方法，有很多奇怪的优化~
8. 由第8题又知道了一种判断溢出的方法。对于正数，(Integer.MAX_VALUE -  single) /10 < sum，对于负数，(Integer.MIN_VALUE - single / 10) > sum.