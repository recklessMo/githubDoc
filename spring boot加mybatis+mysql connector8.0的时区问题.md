### spring boot加mybatis+mysql connector8.0的时区问题



https://juejin.im/post/5902e087da2f60005df05c3d



场景发现存到数据库的时间相差了14个小时。发现是时区问题。归根结底是因为数据库设置的时区是cst

cst这个时间的话在java 新版驱动里面可能会被当成是-06：00时区。而数据库和web服务器都是+08：00时区。

所以java就主动把时间转成了数据库的时间存进去了。这样就导致了不对应。所以两个解决方案。



1. 数据库时区改成0800
2. 在连接url上加上serverTimeZone参数Hongkong或者Asia/shanghai，表明数据库的时区也是东八区，这样java就不会再对时间进行转化了。

