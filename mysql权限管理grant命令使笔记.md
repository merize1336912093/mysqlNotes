#Mysql权限管理grant命令使笔记

标签：mysql

http://www.jb51.net/article/50058.htm

代码如下:
`grant 权限 on 数据库对象 to 用户  [identified by '密码']`

最常用的，弄主从同步的时候，给从库的slave用户设置拥有所有权限，权限all
仅允许其从192.168.0.2登录，并限定使用密码 funsion  (密码要用 单/双引号 括起来)
代码如下:
```
grant all on *.* to slave@192.168.0.2 identified by 'funsion';
```
执行完毕后，记得用 `FLUSH PRIVILEGES;`  刷新一下权限.

## 一、grant 普通数据用户，查询、插入、更新、删除 数据库中所有表数据的权利。
代码如下:
```
grant select, insert, update, delete on testdb.* to common_user@'%'
```
## 二、grant 数据库开发人员，创建表、索引、视图、存储过程、函数.....等权限。
代码如下:
```
grant create, alter, drop on testdb.* to developer@'192.168.0.%';
```
grant 操作 MySQL 外键权限。
代码如下:
```
grant references on testdb.* to developer@'192.168.0.%';
```
grant 操作 MySQL 索引权限。    
代码如下:
```
grant index on testdb.* to developer@'192.168.0.%';
```
给所有IP开放权限：
代码如下:
```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
```
grant 操作 MySQL 临时表权限。
代码如下:
```
grant create temporary tables on testdb.* to developer@'192.168.0.%';
```
grant 操作 MySQL 视图、查看视图源代码 权限。
代码如下:
```
grant create view on testdb.* to developer@'192.168.0.%';
grant show   view on testdb.* to developer@'192.168.0.%';
```
grant 操作 MySQL 存储过程、函数 权限。
代码如下:
```
grant create routine on testdb.* to developer@'192.168.0.%'; -- now, can show procedure status
grant alter  routine on testdb.* to developer@'192.168.0.%'; -- now, you can drop a procedure
grant execute  on testdb.* to developer@'192.168.0.%';
```
执行完毕后，记得用 FLUSH PRIVILEGES;  刷新一下权限
## 三、grant 普通 DBA 管理某个 MySQL 数据库的权限。
代码如下:
```
grant all privileges on testdb to dba@'localhost'
```
其中，关键字 privileges 可以省略。

## 四、grant 高级 DBA 管理 MySQL 中所有数据库的权限。
代码如下:
```
grant all on *.* to dba@'localhost'
```
## 五、MySQL grant 权限，分别可以作用在多个层次上。
1. grant 作用在整个 MySQL 服务器上：
代码如下:
```
grant select on *.* to dba@localhost; -- dba 可以查询 MySQL 中所有数据库中的表。
grant all    on *.* to dba@localhost; -- dba 可以管理 MySQL 中的所有数据库
```
2. grant 作用在单个数据库上：
代码如下:
```
grant select on testdb.* to dba@localhost; -- dba 可以查询 testdb 中的表。
```
3. grant 作用在单个数据表上：
代码如下:
```
grant select, insert, update, delete on testdb.orders to dba@localhost;
```

## 六、查看 MySQL 用户权限
查看当前用户（自己）权限：
代码如下:
```
show grants;
```
查看其他 MySQL 用户权限：
代码如下:
```
show grants for dba@localhost;
```

## 七、撤销已经赋予给 MySQL 用户权限的权限。
revoke 跟 grant 的语法差不多，只需要把关键字 to 换成 from 即可：
代码如下:
```
grant  all on *.* to   dba@localhost;
revoke all on *.* from dba@localhost;
```

# ************************************* 常见问题解决方案 ************************************** #
遇到 SELECT command denied to user '用户名'@'主机名' for table '表名' 这种错误，解决方法是需要把吧后面的表名授权，即是要你授权核心数据库也要。
如遇到的是SELECT command denied to user 'my'@'%' for table 'proc'，是调用存储过程的时候出现，原以为只要把指定的数据库授权就行了，什么存储过程、函数等都不用再管了，谁知道也要把数据库
mysql的proc表授权
mysql授权表共有5个表：user、db、host、tables_priv和columns_priv。
授权表的内容有如下用途：
[user 表]
user表列出可以连接服务器的用户及其口令，并且它指定他们有哪种全局（超级用户）权限。在user表启用的任何权限均是全局权限，并适用于所有数据库。例如，如果你启用了DELETE权限，在这里列出的用户可以从任何表中删除记录，所以在你这样做之前要认真考虑。
[db 表]
db表列出数据库，而用户有权限访问它们。在这里指定的权限适用于一个数据库中的所有表。
[host 表]
host表与db表结合使用在一个较好层次上控制特定主机对数据库的访问权限，这可能比单独使用db好些。这个表不受GRANT和REVOKE语句的影响，所以，你可能发觉你根本不是用它。
[tables_priv 表]
tables_priv表指定表级权限，在这里指定的一个权限适用于一个表的所有列。
[columns_priv 表]
columns_priv表指定列级权限。这里指定的权限适用于一个表的特定列。