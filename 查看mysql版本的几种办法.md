## 查看mysql版本的几种办法
标签：mysql

1）使用命令行模式进入mysql会看到最开始的提示符
```
Your MySQL connection id is 2
Server version: 5.6.28 Source distribution
```
2）登录mysql后，命令行中使用status可以看到
```
status;
//mysql  Ver 14.14 Distrib 5.6.28, for Linux (x86_64) using  EditLine wrapper
```
3）使用系统函数
```
select version();
```
4）在bin目录下执行
```
mysql -V
//mysql  Ver 14.14 Distrib 5.6.28, for Linux (x86_64) using  EditLine wrapper
```
5）在bin目录下执行
```
mysql --help | grep Distrib
//mysql  Ver 14.14 Distrib 5.6.28, for Linux (x86_64) using  EditLine wrapper
```