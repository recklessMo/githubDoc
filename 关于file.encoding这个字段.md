### 关于file.encoding这个字段





这个字段是有缓存的，并不是每次都实时读取。所以必须在启动的时候设置



使用idea这个ide的时候可以设置project.encoding这个字段，这个字段就会设置启动的时候对应的file.encoding



这个字段的意义



javac -encoding gbk x.java

java -Dfile.encoding=gbk x





jcmd命令

