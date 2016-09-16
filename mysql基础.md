[toc]
# mysql基础

标签（空格分隔）： mysql

---

## 1、什么是数据库
按照特定结构存储数据的仓库，形态电子文件。
## 2、数据库分类
### 2.1 关系型数据库
二维表存储（mysql,oracle,sqlserver,db2等）
### 2.2 非关系型数据库（NoSql）
mongodb，redis等
## 3、安装
```
groupadd mysql #添加mysql组
useradd -g mysql mysql -s /bin/false #创建用户mysql并加入到mysql组，不允许mysql用户直接登录系统
mkdir -p /data/mysql #创建MySQL数据库存放目录 
chown -R mysql:mysql /data/mysql #设置权限
mkdir -p /usr/local/mysql #创建安装目录
tar zxvf mysql-5.6.28.tar.gz 
cd mysql-5.6.28  
cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DMYSQL_DATADIR=/data/mysql -DSYSCONFDIR=/etc #配置　　
make #编译　　
make install #安装　　
cd /usr/local/mysql　　
cp ./support-files/my-default.cnf /etc/my.cnf #拷贝配置文件(注意：/etc目录下面默认有一个my.cnf，直接覆盖即可)　　
vi /etc/my.cnf #编辑配置文件,在 [mysqld] 部分增加　　
datadir = /data/mysql #添加MySQL数据库路径　　
./scripts/mysql_install_db --user=mysql #生成mysql系统数据库　　
cp ./support-files/mysql.server /etc/rc.d/init.d/mysqld #把Mysql加入系统启动　　
chmod 755 /etc/init.d/mysqld #增加执行权限　　
chkconfig mysqld on #加入开机启动　　
vi /etc/rc.d/init.d/mysqld #编辑　　
basedir = /usr/local/mysql #MySQL程序安装路径　　
datadir = /data/mysql #MySQl数据库存放目录　　
service mysqld start #启动　　
vi /etc/profile #把mysql服务加入系统环境变量：在最后添加下面这一行　　
export PATH=$PATH:/usr/local/mysql/bin　　
下面这两行把myslq的库文件链接到系统默认的位置，这样你在编译类似PHP等软件时可以不用指定mysql的库文件地址。　　
ln -s /usr/local/mysql/lib/mysql /usr/lib/mysql　　
ln -s /usr/local/mysql/include/mysql /usr/include/mysql　　
shutdown -r now #需要重启系统，等待系统重新启动之后继续在终端命令行下面操作　　
mysql_secure_installation #设置Mysql密码　　
根据提示按Y 回车输入2次密码　　
或者直接修改密码/usr/local/mysql/bin/mysqladmin -u root -p password "123456" #修改密码　　
service mysqld restart #重启
```
## 4、配置文件
win下是my.ini，linux下是my.cnf
```
basedir=路径：mysql的安装路径
datadir=路径：mysql数据存放路径
character-set_server=utf8:mysql的编码
default-storage-engine=innodb:mysql存储引擎
```

## 5、mysql服务启动命令
win:
```
net start mysql
net stop mysql
net restart mysql

linux:
```
service mysql[d] start
service mysql[d] stop
service mysql[d] restart
```

## 6、mysql登录和退出
登录：
```
shell>mysql -u用户名 -p密码 -h服务器名称 -P端口 -D选择数据库
```

退出：
```mysql>exit
mysql>quit
mysql>\q
```

## 7、mysql注释
```
#注释内容
--注释内容
```

## 8、SQL语句(Structruced Query Language结构化查询语句)
1) DDL:数据定义语言，数据库创建，表创建，视图创建等
	创建(CREATE)  删除(DROP)  修改(ALTER)      
2) DML:数据操作语言,对表中的数据增删改
    增(INSERT )  删(DELETE)  改(UPDATE)
3)DQL:数据查询语言
    查询(SELECT)
(4)DCL：数据控制语言，对用户设置权限(GRANT)和撤销权限(REVOKE)

## 9、SQL 命令规范
1) 系统命令大写,名称小写(数据库名，表名，字段名等)
2) 命令结束符用分号或\g，修改命令结束符 delimiter 结束符号
3) 支持折行，但不能在函数，名称，引号中折行。
4) 名称不能用 关键字或保留字，如果用关键字或保留字必须用 反引号引起来 例如:  `database`

## 10、数据库操作
1) 查询数据库
```
show databases;
```
2) 创建数据库
```
create database [if not exists] 数据库名称 [[default] character set [=] 编码];
```
3) 删除数据库
```
drop database [if exists] 数据库名称;
```
4) 查看数据库创建命令
```
show create database 数据库名称;
```
5) 查看数据库的结构
```
desc 数据库名称;
```
6) 修改数据库的编码
```
alter database 数据库名称 [default] character set [=] 编码;
```
7) 打开数据库
user 数据库名称;
8) 查看当前打开的数据库
```
select database();
```

## 11、表结构操作
1) 查看表
```
show tables;
```
2) 创建表结构
```
create table [if not exists] 表名(
	字段(field)名称 数据类型 [字段类型|约束条件],
	字段(field)名称 数据类型 [字段类型|约束条件],
	...
)engine=innodb default charset=utf8;
```
3) 查看表结构
```
desc 表名;
describe 表名;
show columns from 表名;
```
4) 查看创建表结构
```
show create table 表名;
```
说明:
a. 数据表(Table) ,二维表格有列(Column)又叫字段(Field)和行(Row)又叫记录(Record),数据表和数据库的关系好比Excel中工作表和工作簿的关系.
b. 数据表中最少1个字段，记录至少0 行  
c. MySQL引擎eg:InnoDB (外键和事务必须是InnoDB引擎),MyISAM 等

5) 修改自增值
```
alter table 表名 auto_increment=自增值;
```

## 11、对表的数据操作
1) 添加数据
```
insert [into] 表名(字段,字段...)
	value[s] 
		(值,值...),
		(值,值...),
		...;
```

2) 查看表数据
```
select * from 表名;
select 字段名,字段名... from 表名;
```

## 12、表中的数据查询
```
SELECT  字段名|expr,字段名|expr...
FROM 表名
[WHERE 条件]
[GROUP BY 字段]
[HAVING 条件]
[ORDER BY  字段]
[LIMIT [$offset,] $length]
```
1) WHERE 条件:条件过滤
  a. 比较运算符: > >= < <= <> !=  =  <=>判断null值
  b.  IS [NOT] NULL  (判断null值)
  c. [NOT] BETWEEN ... AND ...  （范围值）
  d. [NOT] IN(值,值...) (不连续的某几个值)
  e. 逻辑运算符
	   ! 非   AND && 与,并且   OR || 或者
  f.  LIKE '字符串'  (模糊查询)
	  关键字
			_: 代表任意一个字符
			%:代表任意字符(0个，1个，多个)
2) GROUP BY 字段 : 分组
   说明:
	 a. 分组:将字段中相同的值分为一组
			每组显示一个结果(小编号)
			所以一般显示分组的那个字段
			或是显示聚合函数
			
	 b. 聚合函数:结合GROUP BY使用
		 COUNT(*) :获得每组中的个数,包含NULL值
		 COUNT(字段):获得每组中的个数,不包含NULL值
		 AVG(字段):获得每组中平均值
		 MAX(字段):获得每组中最大值
		 MIN(字段):获得每组中最小值
		 SUM(字段):获得每组中和
                 
3) HAVING 条件 : 二次过滤
	说明：
		a. WHERE对字段的条件过滤
		b. HAVING 一般对一个结果的过滤
		 一般结合GROUP BY使用
4) ORDER BY  字段 : 排序
	说明：
		a.  ASC 升序(默认值)，DESC 降序
                      
5) LIMIT [$offset,] $length : 获得前n条记录
	说明:
		 a. $offset 偏移量,起始编号,编号从0开始
		 b. $length ：显示记录数(长度)
		 c.  WEB程序的分页效果
		计算 $offset =(当前页 -1)*每页显示记录数
					   $offset =($curpage -1)*$pagesize
                                   

## 13、多表操作
1）多表联合查询
```
SELECT 字段名,字段名
      FROM  表1
     连接类型  表2
      ON 两个表的逻辑关系
     连接类型  表3
      ON 两个表的逻辑关系
      ...
```
  连接类型 ：
	内连接: INNER JOIN 两个表都符合的信息
	外连接:
			 左外连接: LEFT (OUTER) JOIN:显示左
				 表中的所有信息和右表符合条件的信息
				 如果左边信息右表没有用NULL值填补
			 右外连接: RIGHT(OUTER) JOIN:显示右
				 表中的所有信息和左表符合条件的信息
				 如果右边信息左表没有用NULL值填补
2）子查询：嵌套查询
    a.子查询:SQL语句中嵌套SELECT 语句
    b. 特点:
		 (1) SELECT 语句
		 (2) 用小括号括起来
		 (3) 一般结合 WHERE 和 GROUP BY使用
             
    c.子查询的使用
	   (1) WHERE:
				(a)  IN 
				(b) 比较运算符
				说明: 比较运算符后子查询是一个结果
						如果产生多个结果会报错,可以通
						过ALL,SOME/ANY来解决
					 >  >=  ALL  大于最大值
					 >  >=  SOME/ANY  大于最小值
					 <  <=  ALL  小于最小值
					 <  <=  SOME/ANY 小于最大值
					  =         SOME/ANY    IN 
						
	   (2) FROM后: FROM一般跟表，也可用子查询
			 SELECT 产生一个新表，并且用括号括起来
		 起别名

## 14、约束(Contriant)
### 14.1. 约束解释:
对字段实现的不为空，唯一等的约束
### 14.2. 约束种类
   (1) DEFAULT 默认值
   (2) NOT NULL  不为空
   (3) PRIMARY KEY 主键
   (4) UNIQUE KEY 唯一性
   (5) FOREIGN KEY 外键
### 14.3.约束格式
    1) 列约束：约束条件写字段后面的约束
        说明:  DEFAULT ,NOT NULL 必须是列约束
    2) 表约束:  对一个字段以上的约束
        说明:  PRIMARY KEY,UNIQUE KEY,FOREIGN KEY 可以实现表约束
      
        
### 14.4. FOREIGN KEY 外键:
对字段的完整性，一致性的约束
一对一的关系  一对多的关系
格式:
```
CREATE TABLE test(
	...,
	FOREIGN KEY (外键列)
	REFERENCES 参考表(字段)
);
```
说明:
```
	 a.  FOREIGN KEY 一定是表约束
	 b. 一定先有参考表(父)，后有外键表(子)
	  先有参考表的记录,后有外键表的对应记录 
	  先删除外键表 ,后删除参考表
	  先删除外键表记录，后删除参考表记录
	 
	 c.  外键表数据类型要参考表中对应字段数据类
	  一致(如果是整型同时UNSIGNED一致
	  如果字符串类型,编码一致,长度可以不一致)
	 d.创建外键，表结构一定是InnoBD引擎 
	 e. 创建外键，MySQL引擎会自动添加索引约束
	 名称
```                      
### 14.5. 逻辑外键:  参考表和外键表是一个表
             例如 :  无限极菜单分类(级联菜单)




