# 查看mysql哪些进程在运行和连接数

标签（空格分隔）： mysql

---
SHOW PROCESSLIST;
或者
SHOW FULL PROCESSLIST;
processlist命令的输出结果**显示了有哪些线程在运行**，不仅可以查看当前所有的连接数，还可以查看当前的连接状态，帮助识别出有问题的查询语句等。

如果是root帐号，能看到所有用户的当前连接；如果是其它普通帐号，则只能看到自己占用的连接。

show processlist;只列出前100条。如果使用show full processlist;命令，则可以全部列出。

---

使用**CONNECTION_ID()**函数可以返回**MySQL服务器的连接数**，也就是到现在为止MySQL服务的连接次数。

每个连接都有各自唯一的ID。




