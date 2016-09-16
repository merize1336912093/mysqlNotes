# mysql读书笔记之一

标签（空格分隔）：mysql

---


1、http://www.cnblogs.com/gulibao/p/5416245.html

float数值类型用于表示单精度浮点数值，而double数值类型用于表示双精度浮点数值，float和double都是浮点型，而decimal是定点型；

MySQL 浮点型和定点型可以用类型名称后加（M，D）来表示，M表示该值的总共长度，D表示小数点后面的长度，M和D又称为精度和标度，如float(7,4)的 可显示为-999.9999，MySQL保存值时进行四舍五入，如果插入999.00009，则结果为999.0001。

FLOAT和DOUBLE在不指 定精度时，默认会按照实际的精度来显示，而DECIMAL在不指定精度时，会按照默认的（10,0）进行操作。


2、若一张表里面存在varchar、text以及其变形、blob以及其变形的字段的话，那么这个表其实也叫动态表，即该表的row_format是dynamic，就是说每条记录所占用的字节是动态的。其优点节省空间，缺点增加读取的时间开销。
所以，做搜索查询量大的表一般都以空间来换取时间，设计成静态表。
 
row_format还有其他一些值：
DEFAULT
FIXED 固定的
DYNAMIC 动态的
COMPRESSED 压缩的
REDUNDANT 累赘的
COMPACT 简化的

修改行格式
ALTER TABLE table_name ROW_FORMAT = DEFAULT
 
修改过程导致：
fixed--->dynamic: 这会导致CHAR变成VARCHAR
dynamic--->fixed: 这会导致VARCHAR变成CHAR


3、当我们想用SQL_NO_CACHE来禁止结果缓存时发现结果和我们的预期不一样，查询执行的结果仍然是缓存后的结果。其实，SQL_NO_CACHE的真正作用是禁止缓存查询结果，但并不意味着cache不作为结果返回给query。
4、ip转换成无符号整数存储，或者转换回ip用到的函数：inet_aton(),inet_ntoa()
5、存储uuid,应该移除"-"符号，或者更好的做法是使用unhex()函数转换成uuid值为16字节的数字，并存储在一个binary中。
hex('abc')
unhex(616263)
6、mysql设计中的陷阱：
1)太多列
2）太多关联
3）全能枚举
create table test(
  type enum('', '1', '2', '3',...,'31'),
...)
4）变相枚举
create table test(
  is_default set('y', 'n') not null default 'n',
...)
5）非此发明的NULL
create table test(
  dt datetime not null default '0000-00-00 00:00:00',
...)

7、范式的优点和缺点
优点：
1）范式化的更新操作通常要比反范式化的才做要快
2）当数据较好的范式化时，就只有很少或者没有重复数据，所以只需要修改更少的数据
3）范式化的表通常比较小，可以更好的放在内存里，所以执行速度快
4）很少有多余的数据意味着检索列表数据时候更少需要使用distinct或者group by语句

缺点:
1）范式化设计通常需要关联，不仅代价昂贵，也可能使得一些索引策略无效，因为范式化将列存放在不同的表中，而这些列在一个表中本可以属于同一个索引。

8、反范式的优点和缺点
优点：
1）可以避免关联，因为所有的数据几乎都可以在一张表上显示，避免了随机I/O
2）可以设计有效的索引；

缺点：
1）表格内的冗余较多，删除数据时候会造成表有些有用的信息丢失。

实际应用中经常混用范式化和反范式化。


9、ON DUPLICATE KEY UPDATE
如果您指定了ON DUPLICATE KEY UPDATE，并且插入行后会导致在一个UNIQUE索引或PRIMARY KEY中出现重复值，则执行旧行UPDATE。
例如，如果列a被定义为UNIQUE，并且包含值1，则以下两个语句具有相同的效果：
mysql> INSERT INTO table (a,b,c) VALUES (1,2,3)
    -> ON DUPLICATE KEY UPDATE c=c+1;
 
mysql> UPDATE table SET c=c+1 WHERE a=1;
如果行作为新记录被插入，则受影响行的值为1；如果原有的记录被更新，则受影响行的值为2。

注释：如果列b也是唯一列，则INSERT与此UPDATE语句相当！


10、mysql设计需要注意的原则：
1）尽量避免过度设计，例如会导致极其复杂查询的schema设计，或者有很多列的表设计（很多的意思是介于有点多和非常的多之间）
2）使用小而简单的合适数据类型，除非真实数据类型中有确切的需要，否则应该尽可能的避免使用null.
3）尽量使用相同的数据类型来存储相似或相关的值，尤其是要在关联条件中使用的列
4）注意可变长的字符串，其在临时表和排序时可能导致悲观的最大长度分配内存
5）尽量使用整型来定义标识列
6）避免使用mysql已经遗弃的属性，例如指定浮点数的精度，或者整数的显示宽度
7）小心使用enum和set。虽然他们 使用起来很方便，但是不要滥用，否则有时候会变成陷阱，最好避免使用bit。





