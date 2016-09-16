# mysql主从复制配置实验

标签（空格分隔）： mysql

---
http://www.cnblogs.com/lyhabc/p/3888702.html

MYSQL 从3.25.15版本开始提供数据库复制功能（replication）。mysql复制是指从一个mysql主服务器（MASTER）将数据复制到另一台或多台mysql从服务器（SLAVE）的过程，将主数据库的DDL和DML操作通过二进制日志传到复制服务器上，
然后在从服务器上对这些日志重新执行，从而使从服务器的数据保持同步。

在mysql中，复制操作是异步进行的，slave服务器不需要持续的保持连接接收master服务器的数据


mysql支持一台主服务器同时向多台从服务器进行复制操作，从服务器同时可以作为其他从服务器的主服务器，如果mysql主服务器访问量大，可以通过复制数据，然后在从服务器上进行查询操作，从而降低主服务器的访问压力（读写分离），同时从服务器作为主服务器的备份，可以避免主服务器因为故障数据丢失的问题。

mysql数据库复制操作大致可以分为三个步骤
1）主服务器将数据的改变记录到二进制日志（binlog）中。
2）从服务器将主服务器的binary log events复制到他的中继日志（relay log）中。
3）从服务器做中继日志中的事件，将数据的改变与从服务器保持同步。

首先，主服务器会记录二进制日志，每个事务更新完毕数据之前，主服务器将这些操作的信息记录在二进制日志里面，在事件写入二进制日志完成后，主服务器通知存储引擎提交事务。
SLAVE上面的I/O进程连接上MASTER，并发出日志请求，MASTER接收到来自SLAVE的I/O进程的请求后，通过负责复制的I/O进程根据请求信息读取指定日志位置之后的日志信息，返回给SLAVE的I/O进程。返回信息中除了日志所包含的信息之外，还包括本次返回的信息已经到MASTER端的binlog文件的名称以及binlog的位置。

SLAVE的I/O进程接收到信息后，将接收到的日志内容依次添加到SLAVE端的relay-log文件的最末端，并将读取到的MASTER端的binlog文件名和位置记录到master-Info文件中，SLAVE的SQL进程检测到relay-log中新增了内容后，会马上解析relay-log的内容成为在master端真实执行时候的那些可执行内容，并在自身执行mysql复制环境，90%以上都是一个master带一个或者多个slave的架构模式。如果master和slave压力不是太大的话，异步复制的延时一般都很少。尤其是slave端的复制方式改成两个进程处理之后，更是减少了slave端的延时

提示：对于数据实时性要求不是特别严格的应用，只需要通过廉价的电脑服务器来扩展slave的数量，将读压力分散到多台slave的机器上面即可解决数据库端的读压力瓶颈。这在很大程度上解决了目前很多中小型网站的数据库压力瓶颈问题，甚至有些大型网站也在使用类似方案解决数据库瓶颈问题。

---
虚拟机上做的实验（centos6.6-64,mysql5.6.28）
复制前的准备工作:
```
主 192.168.83.234 datadir=/data/mysql/  5.6.28  3306
从 192.168.83.250  datadir=/data/mysql/   5.6.28 3306 
```
1、配置好两台主机的ip地址，实现两台计算机可以网络连通
2、配置master的相关配置信息，在master主机上开启binlog日志，首先，看下datadir的具体路径
```
show variables  LIKE '%datadir%'
```
3、此时需要打开在mysql路径下的配置文件my.cnf，添加如下代码，开启binlog功能
```
//记得创建对应的目录
log-bin=/data/mysql/mysql-bin
expire_logs_days=10
max_binlog_size=100M
```
提示：此事我们需要创建mysql-bin文件夹，binlog日志记录在该文件夹里面，该配置文件中的其他参数如下所示
expire_logs_days：表示二进制日志文件删除的天数
max_binlog_size：表示二进制日志文件最大的大小

4、登录mysql后，可以执行show VARIABLES LIKE '%log_bin%'命令来测试下log_bin是否成功开启
```
show VARIABLES LIKE '%log_bin%';
```
如果log_bin参数是ON的话，那么表示二进制日志文件已经成功开启，如果为OFF的话，那么表示二进制日志文件开启失败

5、在master上配置复制所需要的账户，这里创建一个repl的用户，%表示任何远程地址的repl用户都可以连接master主机
```
//这句话相当于创建了repl用户
GRANT replication slave ON *.*TO repl@'%' IDENTIFIED BY '123';

flush privileges;
``` 

6、在my.cnf配置文件里配置master主机的相关信息
```
[mysqld]
server-id=1
binlog-do-db=test
binlog-ignore-db=mysql
```
server-id：表示服务器表示id号，master和slave主机的server-id不能一样

binlog-do-db：表示需要复制的数据库，这里以test库为例

binlog-ignore-db：表示不需要复制的数据库

注意：如果主从库的server-id一样的话，会报下面的错
```
Fatal error: The slave I/O thread stops because master and slave have equal MySQL server ids; 
```

7、重启master主机上的mysql服务，然后输入show master status命令查询master主机的信息
```
> show master status
*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 120
     Binlog_Do_DB: test
 Binlog_Ignore_DB: mysql
Executed_Gtid_Set: 
```
8、将master主机的数据备份出来，然后导入到slave主机中去，具体执行语句如下
```
mysqldump -u root -p -h 127.0.0.1 test >D:\TEST.sql
```
然后在从库，use test,然后执行source命令导入TEXT.txt文件的内容.

9、配置slave机器（192.168.83.250）的my.cnf配置文件

具体配置信息如下：
```
[mysql]
default-character-set=utf8

[mysqld]
log-bin=/data/mysql/mysql-bin
expire_logs_days=10
server-id=2
```
注意前面的[mysql]是必不可少的，否则会报错:
```
MySQL server PID file could not be found!                  [FAILED]
Starting MySQL.The server quit without updating PID file (/[FAILED]ql/localhost.localdomain.pid).
```

提示：配置slave主机my.cnf文件的时候，需要将server-id=2写到[mysqld]后面

另外如果配置文件中还有log_bin的配置，可以将他注释掉，如下所示
```
#Binary Logging
#log-bin
#log_bin="xxx"
```

10、重启slave主机（192.168.83.250）的mysql服务，在slave主机（192.168.83.250）的mysql中执行如下命令

关闭slave服务
```
stop slave;
```
11、设置slave从机实现复制相关的信息，命令如下
```
change master to
master_host='192.168.83.234',
master_user='repl',
master_password='123456',
master_log_file='mysql-bin.000001',
master_log_pos=120;
```
各个参数所代表的具体含义如下：

master_host：表示实现复制的主机ip地址

master_user：表示实现复制的登录远程主机的用户

master_password：表示实现复制的登录远程主机的密码

master_log_file：表示实现复制的binlog日志文件

master_log_pos：表示实现复制的binlog日志文件的偏移量

12、继续在从机执行操作，显示slave从机的状况，如下所示
```
mysql> start slave;
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: 
                  Master_Host: 192.168.83.234
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 120
               Relay_Log_File: localhost-relay-bin.000003
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Connecting
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 120
              Relay_Log_Space: 120
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 1236
                Last_IO_Error:  Got fatal error 1236 from master when reading dat
a from binary log: 'Could not find first log file name in binary log index file'
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 0
                  Master_UUID: 
             Master_Info_File: /data/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 160910 16:17:37
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
1 row in set (0.00 sec)
```
针对Last_IO_Error的错解决办法：
1)重启master（192.168.83.234）主机的mysql服务，执行show master status \G命令,记下File和Position的值，后面slave主机会用到.
2)在slave（192.168.83.250）主机上重新设置信息
```
mysql> stop slave;
mysql> change master to
    master_log_file='mysql-bin.000003',
    master_log_pos=120;
    mysql> start slave;
mysql> SHOW SLAVE STATUS \G;
```
然后重点关注`Slave_IO_Running:`和`Slave_SQL_Running::`，必须都为yes，然后`Slave_IO_Running:=Waiting for master to send event`才是对的。如果不是请看下面其他地方的报错。

**遇上两个问题：**
1）Slave_IO_Running=No,而Slave_SQL_Running=yes。
然后看到下面的错误信息：
Fatal error: The slave I/O thread stops because master and slave have equal MySQL server UUIDs; these UUIDs must be different for replication to work.
解决办法参考：
http://www.fwqtg.net/%E6%95%85%E9%9A%9C%E6%A1%88%E4%BE%8B%EF%BC%9A%E4%B8%BB%E4%BB%8E%E5%90%8C%E6%AD%A5%E6%8A%A5%E9%94%99fatal-error-the-slave-io-thread-stops-because-master-and-slave-have-equal-mysql-server.html

找到data文件夹下的auto.cnf文件，修改里面的uuid值，保证各个db的uuid不一样，重启db即可。

2）lave_IO_Running为connecting,而Slave_SQL_Running=yes。

看到错误信息为：Last_IO_Error: error connecting to master 'repl@192.168.83.234:3306' - retry-time: 60  retries: 1
解决办法参考：https://code.google.com/p/mysql-master-ha/issues/detail?id=85
把repl的密码改了，然后重新stop slave;然后，
change master to
master_host='192.168.83.234',
master_user='repl',
master_password='123456',
master_log_file='mysql-bin.000003',
master_log_pos=120;

最后start slave;就ok了。
 
12、最后朝主库的某个表插入数据，然后在从库看是否跟着变化了即可。

疑问：不知道为什么
log-bin配置好的/data/mysql/mysql-bin，但是产生的二进制文件却在/data/mysql/下面。
开启bin-log，路径要设置在mysql用户所属文件夹下
例如：
log-bin=/home/123/bin-log     123的所属用户就要是Mysql