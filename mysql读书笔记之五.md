[toc]
# mysql读书笔记之五

标签（空格分隔）： mysql

---

优化特定类型的查询
## 1、优化count()查询

count的作用：
count()有两种非常不同的作用：
1）他可以统计某个列的数量（在统计列值时候，要求列值是非空的，亦即不统计null）
2）也可以统计行数
myisam中，只有在没有任何where条件的count(*)才会非常快，因为可以利用搜索引擎的特性直接去获得这个值，如果mysql知道某个列不可能为null,那么mysql会直接把count(col)表达式转化为count(*)。


简单优化：
1）利用条件反转：
例如：select count(*) from city where id>5;
假设这个查询会扫描4097行数据，那么我们可以优化如下：
select (select count(*) from city) -count(*) from city where id<=5;
2）统计同一列不同值的数量
例如：
select sum(if(money=60.000,1,0)) as blue,sum(if(money=70.000,1,0)) as red from test

也可以使用count()而不是sum()来实现同样的目的，只需要将条件置为真，不满足条件的置为null即可。如下：
select count(money=60.000 or null) as blue,count(money=70.000 or null) as red from test


使用近似值：
有时候某些业务场景并不要求完全精确的count值，此时可以用近似值来代替。explain处理的优化器估算的行数就是一个不错的近似值，执行explain并不需要真正去执行查询，所以成本很低。
更进一步的优化则可以尝试剔除distinct这样的约束来避免文件排序。

更复杂的优化：
通常来说count()需要扫描大量的行才能获得准确的结果，因此是很难优化的，除了前面的方法，mysql层面还能做的就只有覆盖索引扫描。如果这还不够，就需要考虑修改应用的机构，可以增加汇总表或者增加类似memcache的外部缓存系统。


## 2、优化关联查询
1）确保on或者using子句中的列上有索引。在创建索引的时候就要考虑到关联的顺序。当表a和b用col1列来关联的时候，如果优化器的关联顺序是b,a,那么就需要在b表对应的列上建立索引。没有用到的索引只会带来额外的负担。一般来说，除非有其他理由，否则至于在关联顺序中的第二个表的相应列上建立索引。
2）确保任何的group by 和 order by 中的表达式只涉及到一个表中的列，这样mysql才有可能使用索引来优化这个过程，
3）当升级mysql的时候注意，关联语法、运算符优先级等其他可能发生变化的地方。

## 3、优化子查询
关于子查询优化最重要的建议是尽可能使用关联查询代替。并非绝对。

## 4、优化group by 和 distinct
他们都可以使用索引来优化，这也是最有效的。

在mysql中，如果无法使用索引的时候，group by使用两种策略来完成：使用临时表或者文件排序来做分组。对于任何查询语句，这两种策略都有可以提升的地方。可以使用sql_small_result和sql_big_result来让优化器按照你希望的方式运行。

如果需要对关联查询做分组，并且是按照查找表的某个列进行分组，那么通常采用查找表的标识列分组的效率会比其他列更高。

我们建议将mysql的sql_mode设置问only_full_group_by。


如果没有通过group by子句显示的指定排序列，当查询使用group by子句的时候，结果集会自动按照分组的字段进行排序。如果不关心结果集的顺序，而这种默认的排序又导致了需要文件排序，则可以使用order by null，让mysql不再进行文件排序。也可以在group by 子句中直接使用desc或者asc关键字，使得分组的结果按照需要的方向排序。


优化group by with rollup:
分组查询的一个变种就是要求mysql对返回的分组结果再做一次超级聚合。可以使用with rollup 子句来实现这种逻辑，但可能不够优化。通过explain来查看，注意分组是否是通过文件排序或者临时表来实现的。然后去掉with rollup子句看执行计划是否相同。
很多时候如果可以，在应用程序中作差集聚合是更好的。也可以在from子句中嵌套使用子查询，或者通过一个临时表来存放中间数据，然后和临时表执行union来得到最终结果。

## 5、优化limit分页
一个简单的方法是尽可能使用索引覆盖扫描，而不是查询所有的列。然后更需要做一次关联操作再返回所需的列。对于偏移量很大的时候，这样做的效率提升非常大。
例如：
select id,desc from film order by title limit 10000,10;
可以优化为这样：
select id,desc from film 
	inner join(
		select id from film order by title limit 10000,10
	)as lim using(id);
	
这种叫做“延迟关联”。
除此之外，
1）可以使用书签记录上次取数据的位置，那么下次可以直接从改书签记录的位置开始扫描，这样可以避免offset.
2）可以使用预先计算好的汇总表或者关联到一个冗余表，冗余表只包含主键列和需要做排序的数据列。
3）使用sphinx优化一些搜索操作

## 6、优化sql_calc_found_rows
分页的时候，另一个常用的技巧是在limit语句后面加上sql_calc_found_rows提示，这样就可以获得去掉limit以后满足条件的行数，因此可以作为分页的总数。不管是否需要，mysql都会扫描所有满足条件的行，然后再抛弃不需要的行，所以该提示代价特别高。
更好的办法:
1)具体页数换成“下一页按钮”。假设每页显示20条，那么每次查询都查21条，页面展示的时候，如果有21条就展示【下一页】，如果没有就不显示。
2）先获取并缓存较多的数据，
比如缓存1000条。然后每次分页从缓存中取，如果结果集大于1000，则可以在页面设计一个额外的“找到结果多于1000条”之类的按钮。
3）有时候可以考虑使用explain结果中的rows列的值作为结果集总数的近似值。

## 7、优化union查询
mysql总是通过创建临时表的方式来执行union查询，因此很多优化策略在union查询中都没法很好的使用。

除非确实需要服务器消除重复的行，否则就一定要使用union all.如果没有all关键字，mysql会给临时表加上distinct，会导致对整个临时表的唯一性检查。

## 8、静态查询分析
percona toolkit 的pt-query-advisor能够解析查询日志、分析查询模式，然后给出所有可能存在潜在问题的查询，并给出足够详细的建议。

## 9、使用用户自定义变量

### 9.1 简介及示例：
是一个很容易被遗忘的mysql特性。是一个用来存储内容的临时容器。在连接mysql的整个过程中都存在。
eg；
```
set @one :=1;
set @min_actor := (select min(actor_id) from actor);
set @last_week := CURRENT_DATE-INTERVAL 1 WEEK;
```
然后可以在任何可以使用表达式的地方使用这些自定义变量：
```
select ... where col <= @last_week;
```

### 9.2 自定义变量的局限性
1）使用自定义变量的查询，无法使用查询缓存
2）不能在使用常量或者标识符的地方使用自定义变量，例如表名、列名和limit子句中。
3）用户自定义变量的生命周期是在一个连接中有效，所以不能用他们来做连接间的通信。
4）如果使用连接池或者持久化连接，自定义变量可能让看起来毫无关系的代码发生交互（如果是这样，通常是代码bug或者连接池bug）
5）在5.0之前版本是大小写敏感的，所以要注意代码在不同mysql版本间的兼容性问题
6）不能显示地声明自定义变量的类型，确定未定义变量的具体类型的时机可能在不同的mysql版本中也可能不一样。
7）mysql优化器在某些场景下可能会将这些变量优化掉，这可能导致代码不按照预想的方式运行
8）赋值的顺序和赋值的时间点并不总是固定的，这依赖于优化器的决定。实际情况可能让人困惑。
9）赋值符合:=的优先级非常低，所以需要注意，赋值表达式应该使用明确的括号。
10）使用未定义变量不会产生任何语法错误，如果没有注意到这一点，非常容易犯错。

### 9.3 优化排名语句
使用用户自定义变量一个重要特性是你可以在给一个变量赋值的同时使用这个变量。换句话来说，用户自定义变量具有“左值”特性。
例1：实现一个类似“行号”或者“排名”的功能：
```
set @rownum := 0;
select id,@rownum := @rownum +1  as rownum from actor limit 3;
```
例2：每个演员参演电影数量及排名：

```
//@curr_cnt用来记录当前的排名，@prev_cnt用来记录前一个演员的排名，@rank用来记录当前演员参演电影的数量
set @curr_cnt :=0,@prev_cnt :=0,@rank :=0;
select actor_id,
	@curr_cnt := cnt as cnt,
	@rank := if(@prev_cnt <> @curr_cnt,@rank+1, @rank) as rank,
	@prev_cnt := $curr_cnt as dummy
from (
	select actor_id,count(*) as cnt 
	from film_actor group by actor_id order by cnt desc limit 10
) as der
```

### 9.4 避免重复查询刚刚更新的数据
如果在更新行的同时又希望获得该行的信息，怎么做才可以避免重复查询呢？不幸的是mysql并不支持像PostgreSQL那样的update returning语法。mysql可以使用变量来解决这个问题。
如下：
```
update t1 set lastUpdated = now() where id=1 and @now := now();
select @now;
```

### 9.5 统计更新和插入的数量(248)

当使用了`insert on duplicate key update` 的时候，想知道到底插入了多少行数据，多少数据是因为冲突而改写成更新操作的。如下：
```
create table test(
id int unsigned not null primary key,
age int unsigned not null default 0
)engine=innodb DEFAULT CHARSET=utf8;

insert into test values(1,1),(2,2),(3,3);

insert into test(id ,age) values(1,3),(2,6),(4,5)
on duplicate key update 
age = values(age) + (0 * (@x := @x +1));
```
当每次由于冲突导致更新时候对变量@x自增一次，然后通过对这个表达式乘以0来让其不影响要更新的内容。另外mysql的协议会返回被更改的总行数，所以不需要单独统计这个值。

### 9.6 确定取值顺序(248)
 使用用户自定义变量最常见的一个问题就是没有注意到在赋值和读取变量的时候可能是在查询的不同阶段。
 例如下面这个看起来只会返回一个结果，实际上
```
set @rownum := 0;
select id,@rownum := @rownum +1 as cnt from test where @rownum <=1;
```
结果是：
```
id,cnt
1,1
2,2
```
如果改成下面的加上order by last_name，引入文件排序，而where条件是在文件排序之前取值的，所以会返回全部记录。
```
set @rownum := 0;
select id,last_name,@rownum := @rownum +1 as cnt from test where @rownum <=1 order by last_name;
```
解决这个问题的办法在于让取值和查询发生在同一阶段：
```
set @rownum := 0;
select id,@rownum as cnt from test where (@rownum := @rownum +1) <=1;
```

试想如果上边这个加上order by ,会又怎样的结果呢？
```
set @rownum := 0;
select id,@rownum as cnt from test where (@rownum := @rownum +1) <=1 order by last_name;
```

下面这个语句order by子句会改变变量值，通过explain可以看到extra列有 Using where; Using filesort或者Using temporary
```
set @rownum := 0;
select id,last_name,@rownum as cnt from test where @rownum <=1 order by last_name, least(0,@rownum := @rownum +1);
```
结果如下：
```
id,last_name,cnt
1,a,1
2,c,2
```
技巧在于我们将赋值语句放在least()函数中，这样就可以在完全不改变排序顺序的时候完成赋值操作（上边例子中，least总是返回0）。
这个技巧在于不希望对子句的执行结果有影响又要完成变量赋值时候很有用。
这样的函数还有`greatest(),lenght(),isnull(),nullif(),if()和coalesce()`，可以单独使用也可以组合使用。
ifnull(expr1,expr2):如果expr1不为null,则返回expr1;否则返回expr2;
nullif(expr1,expr2):如果expr1=expr2,那么返回null;否则返回expr1;
coalesce(val1,val2,val3)作用是返回传入参数中第一个非null的值，如果全部为null,则返回null.


### 9.7 编写偷懒的union(250)

例如：下面的查询会在两个地方来查找一个用户-一个主用户表，一个长时间不活跃用户表，不活跃用户表是为了实现高效的归档。
```
select greatest(@found := -1, id) as  id ,"users" as which_tb1 from usrs where id=20
union all 
select id,"users_archived" from users_archived where id=20 and @found is null;
union all 
select 1,'reset' from dual where (@found := null) is not null;
```

### 9.8 用户自定义变量的其他用处
有时候优化器会把变量当做一个编译时的常量来对待，而不是对其赋值。将函数放在类似于least()这样的函数中通常可以避免这样的问题、另一个办法是在查询被执行前检查变量是否被赋值。不同的场景下使用不同的办法。
通过一些实践，可以了解用户自定义变量能够做的有趣的事情，例如：
1）查询运行时计算总数和平均值
2）模拟group语句中的函数first()和last()
3）对大量数据做一些数据计算
4）计算一个大表的MD5散列值
5）编写一个样本处理函数，当样本中的数值超过某个边界值的时候就变成0
6）模拟读/游标
7）在show语句的where子句中加入变量值

## 10案例学习：
使用mysql构建一个队列表
一般，我们要尽量避免使用select for update。不光是队列表，任何情况下，都要尽量避免。总是有更好的办法来实现目的。
例如：在队列表的案例中，可以直接使用update来更新记录，然后检查是否还有其他记录需要处理。
```
begin;
select id from unsent_emails
	where owner=0 and status ='unsent' limit 10 for update;
// 123,456,789
update unsent_emails set 
	status='claimed', owner=connection_id() where id in(123,456,789);
commit;
```
如果改成下面这样，会更加高效：
```
set autocommit =1;
commit;

update unsent_emails set 
	status='claimed', owner=connection_id() where owner=0 and status ='unsent' limit 10;
set autocommit=0;
select id from unsent_emails
	where owner=connection_id() and status ='claimed';
// 123,456,789
```

这样根本无需使用select查询去找到哪些记录还没有被处理，客户端的协议会告诉你更新了几条记录，所以可以知道这次需要处理多少条记录。
所有的select for update都可以采用类似的写法。

最后还有一种特殊情况：那些正在被进程处理，而进程本身却由于某种原因退出的情况。这种处理起来很简单。只需要定期运行update将他都更新更原始状态就可以了，然后执行show processlist，获取当前正在工作的进程id,并使用一些where条件避免取到那些刚开始处理的进程。
假设我们获取的进程id有(10,20,30)，下面的更新会讲处理时间超过10分钟的的记录状态丢更新成初始状态：
```
update unsent_emails set 
	status='unsent', owner=0 where owner not in(0,10,20,30) and status ='claimed' and ts < current_timestamp - interval 10 minute;
```


有时候最好的办法就是将任务从数据库中迁移出来。redis就是一个很好的对列容器，也可以使用memcached来实现。