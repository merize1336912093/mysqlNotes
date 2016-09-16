### MySQL开启慢查询日志

标签：mysql

http://jingyan.baidu.com/article/0aa223755476db88cc0d6492.html

MySQL慢查询日志对于跟踪有问题的查询非常有用,可以分析出当前程序里有很耗费资源的sql语句,那如何开启MySQL慢查询日志呢?
1)
在MySQL客户端中输入命令：
show variables like '%quer%';
其中红框标注的选项是：
-slow_query_log是否记录慢查询。用long_query_time变量的值来确定“慢查询”。
-slow_query_log_file慢日志文件路径
-long_query_time慢日志执行时长（秒），超过设定的时间才会记日志
2)
在/etc/my.cnf配置文件的[mysqld]选项下增加：
slow_query_log=TRUE
slow_query_log_file=/usr/local/mysql/slow_query_log.txt
long_query_time=3
Windows：
在my.ini配置文件的[mysqld]选项下增加：
slow_query_log=TRUE
slow_query_log_file=c:/slow_query_log.txt
long_query_time=3
3)
重启MySQL后，可发现已经开启慢查询日志