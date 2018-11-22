### mysql死锁问题

https://www.cnblogs.com/Arlen/articles/1756915.html

http://keithlan.github.io/2017/08/17/innodb_locks_deadlock/

http://keithlan.github.io/2017/06/05/innodb_locks_1/

http://yeshaoting.cn/article/database/mysql%20insert%E9%94%81%E6%9C%BA%E5%88%B6/



```
update set time = ***, a = 1 where id = 1;

update set a = 1 where time = 12;
```

上面两条sql 加锁顺序不一致。

注意，自己在尝试重试的时候要防止mysql对查询语句进行优化，走全表而不走索引哦。坑





















