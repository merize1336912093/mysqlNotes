[toc]
# mysql读书笔记之四

**查询优化器的局限性和提示(hint)**

标签（空格分隔）： mysql

---

## 1、mysql查询优化器的局限性
mysql万能的“嵌套循环”并不是对每种查询都是最优的。不过还好，mysql查询优化器只对少部分查询不适用，不过我们往往可以通过改写查询让mysql高效完成工作。
### 1.1 关联子查询
mysql的子查询实现的非常糟糕，尤其糟糕的一类是是where条件中包含in()的子查询。
比如：
```
select * from film 
	where film_id in(
		select film_id from actor where actor_id=1
	);
```
根据mysql对in()列表的选项有专门优化策略，一般会认为mysql会先执行子查询返回所有包含actor_id为1的film_id。如下面这样：
```
select group_concat(film_id) from actor where actor_id=1;
// 12,23,54,65,85
select * from film where film_id in(12,23,54,65,85);
```

很不幸mysql并不是这么做的，mysql会将相关的外层表压缩到子查询中，他认为这样可以高效率查找到数据行，亦即会改写成下面这样：
```
select * from film
	where exists(
		select * from actor where actor_id=1
		and actor.film_id=film.film_id
	)
```

这时候子查询需要根据film_id来关联外部表film。因为需要film_id字段，索引mysql认为无法先执行子查询，通过explain可以看到相关子查询是dependent subquery。可以用explain extended查看这个查询被改写成什么。
注意:
DEPENDENT SUBQUERY：子查询中的第一个SELECT，取决于外面的查询。

优化的办法：
1）改写成：
```
select film.* from film 
	inner join actor using(film_id)
where actor_id=1
```
2）另一个办法是使用函数group_concat()在exists()中构造一个由逗号分隔的列表,如下：
```
select * from film
	where exists(
		select * from actor where actor_id=1 and actor.film_id=film_id
	)
```

如何用好关联子查询：
并不是所有的关联子查询的性能都很差。
一般情况下会建议使用左外连接来代替关联子查询，但是各个情况不一定，需要实际的测试之后做出取舍。（检查QPS）.


### 1.2 union的限制
有时候，mysql无法将限制条件从外层“下推”到内层，这使得原版能够限制部分返回结果的条件无法应用到内层查询的优化上。
eg:
```
(select first_name,last_name from actor order by last_name)
union all 
(select first_name,last_name from customer order by last_name)
limit 20;
```

可以通过在union的两个子查询中分别加上也limit 20来减少表中的数据。
优化成：
```
(select first_name,last_name from actor order by last_name limt 20)
union all 
(select first_name,last_name from customer order by last_name limit 20)
limit 20;
```
注意：从临时表中取出数据的顺序并不是一定的，所以如果想获得正确的顺序，还需要加上一个全局的order by 和limit操作。

### 1.3 索引合并优化
当where子句中包含多个复杂条件的时候，mysql能够访问单个表的多个索引以合并和交叉过滤的方式来定位需要查找的行。

### 1.4 等值传递
某些时候等值传递会带来一些意想不到的额外消耗。例如有一个非常大的in()列表，而mysql优化器发现存在where,on,using的子句，将这个列表的值和另外一个列表的的某个列相关联。
那么优化器会将in()列表都复制应用到关联的各个表中。通常，因为各个表新增了过滤条件，优化器可以更高效的聪存储引擎过滤记录。但是如果这个表非常大，则会导致优化和执行都会变慢。

### 1.5 并行执行
mysql无法利用多核特性来并行执行查询。不用再花时间找并行执行的方法了。

### 1.6 哈希关联
mysql不支持哈希关联，mysql的所有关联都是嵌套循环关联。不过可以通过建立一个哈希索引啦曲线实现哈希关联。如果使用的是memory引擎，则索引都是哈希索引，所以关联的时候也类似与哈希关联。另外MariaDB已经实现了真正的哈希关联。


### 1.7 松散索引扫描
由于历史原因，mysql不支持松散索引扫描，也就无法按照不连续的方式扫描一个索引。通常mysql的索引扫描需要先定义一个起点和终点，即使需要的数据只是这段索引中很少数的几个，mysql仍需要扫描这段索引中的每一个条目。

### 1.8 最大值和最小值优化
对于max()和min()查询，mysql优化做的并不好。
比如为了获得更高性能没我们不得不放弃一些原则（通过sql并不能看出我们其实是想要最小值）,因为actor_id是主键而且是严格按照大小排列的。
```
select actor_id from actor use index(primary) where first_name ='PENELOPE' limit 1
```

### 1.9 在同一个表上查询和更新
mysql不允许对同一张表进行查询和更新。这其实并不是优化器的限制。
```
mysql> update test as otb1 set money =(select count(*) from test as itb1 where itb1.id=otb1.id);
ERROR 1093 (HY000): You can't specify target table 'otb1' for update in FROM clause
```

可以通过使用生成表的形式来绕过上面的限制，因为mysql只会把这个表当做一个临时表来处理。
可以改写成：
```
update test inner join(
		select id,count(*) as money from test group by id
	) as der using(id)
set test.money=der.money;
```

## 2、查询优化器的提示（hint）
### 2.1 high_priority和low_priority
这两个提示告诉mysql，当多个语句同时访问某一个表的时候，哪些语句的优先级相对高些，哪些语句的优先级稍微低些。
high_priority用于select语句的时候，mysql会将此select语句重新调度到所有正在等待表锁以便修改数据的语句之前。实际上mysql是将其放在表的队列的最前面，而不是安装常规顺序等待。high_priority还可以用于insert语句，其效果只是简单的抵消了全局low_priority设置对该语句的影响。
low_prioriry则正好相反，他会让语句一直处于等待状态，只要队列中还有需要访问同一个表的语句-即使是那些比该语句晚提交到服务器的。low_priority提示在select,insert,update,delete语句都可以用。
这两个提示只对使用表锁的存储引擎有效，千万不要在innodb或者其他有细粒度锁机制和并发机制的引擎中使用。即使是在myisam中使用也要注意。因为这两个提示会导致并大插入被禁用，可能会严重降低性能。
high_priority和low_priority经常让人感到困惑。这两个提示并不会获取跟多资源让查询“积极”工作，也不会少获取资源让查询“消极”工作，他们只是简单地控制了mysql访问某个数据表的队列顺序。
### 2.2 delayed
这个提示对insert和replace有效。mysql会将使用该提示的语句立即返回给客户端，并将插入的行数数据插入到缓冲区，然后在表空闲时候批量将数据写入。日志系统使用这样的提示日常有效，或者是其他需要写入大量数据但是客户端却不需要等待单条语句完成i/o的应用。
这个用法有一些限制：并不是所有的存储引擎都支持这样的做法；并且该提示会导致函数last_insert_id()无法正常工作。
### 2.3 straight_join
这个提示放置在select语句的select关键字之后，也可以放置在任何两个关联表的名字之间。第一个用法是让查询中所有的表按照在语句中出现的顺序进行关联。第二个用法则是固定其前后两个表的关联顺序。
当mysql没能选择正确关联顺序的时候，或者由于可能的顺序太多导致mysql无法评估所有的关联顺序的时候，straight_join都会很有用。在后面这种情况，mysql可能会花费大量的时间在"statistics"状态，加上这个提示则会大大减少优化器的搜索空间。
可以优先使用explain来查看优化器选择的关联顺序，然后使用该提示来重写查询，再看看他的关联顺序。
注意：在升级mysql版本后，需要重新审视下这类查询，某些新的优化特性可能会因为该提示而失效。
### 2.4 sql_small_result 和 sql_big_result
这两个提示只对select有效。他们告诉优化器对group by 或者distinct 查询如何使用临时表及排序。sql_small_result告诉优化器可以将结果集放在内存中的索引临时表,sql_big_result告诉优化器建议用磁盘临时表来做排序操作。
### 2.5 sql_buffer_result
这个提示告诉优化器将查询结果放入到一个临时表，然后尽可能快的释放表锁。这和前面提到的客户端缓存结果不同。当你没法使用客户端缓存时候，使用服务器缓存通常很有效。带来的好处是无须在客户端消耗太多的内存，还可以尽可能快的释放对应的表锁。代价是，服务器将需要更多的内存。
### 2.6 sql_cache和sql_no_cache
这个提示告诉mysql这个结果集是否应该缓存在查询缓存中，
### 2.7 sql_calx_found_rows
严格来说，这并不是一个优化器提示。他不会告诉优化器任何关于执行计划的东西。他会让mysql返回的结果集包含更多的信息。查询中加上该提示mysql会计算除去limit子句后这个查询要返回的结果集总数，而实际上只返回limit要求的结果集。可以通过函数found_row()获得这个值。（后面会解释到为啥不应该使用该提示）
### 2.8 for update 和lock in share mode
这也不是真正的优化器提示。这两个提示主要控制select 语句的锁机制，但只对实现了行级锁的存储引擎有效。使用该提示会对符合查询条件的数据行枷锁，对于insert ... select 语句是不需要加这两个提示的。因为对于mysql5.0及新版本默认给这些锁加上读锁（可以禁用默认行为，但不是个好主意）
唯一内置的支持这两个提示的引擎就是innodb。另外需要记住的是，这两个提示会让某些优化无法正常使用，例如索引覆盖扫描。innodb不能在不访问主键的情况西安排他的锁定行，因为行的版本信息保存在主键中。
糟糕的是，这两个提示经常被滥用，很容易造成服务器的锁争用问题。应该尽可能得避免这两个提示，通常都有更好的方式可以实现同样的目的。
### 2.9 use index,ignore index和force index
这几个提示会告诉优化器使用或者不使用哪些索引来查询记录。在mysql5.1和之后的版本可以通过新增选项for order by 和for group by 来指定是否对排序和分组有效。
force index和use index基本相同，除了一点：force indx会告诉优化器全表扫描的成本会远远高于索引扫描，哪怕实际上该索引用处不大。当发现优化器选择了错误的索引，或者由于某些原因（比如不使用order by的时候希望结果有序）要使用另外一个索引时候，可以使用该提示。

**在5.0和更新版本中新增了一些参数来控制优化器的行为**
### 2.10 optimizer_search_depth
这个参数控制优化器在穷举执行计划时的限度。如果查询长时间处理"statistics"状态，那么可以考虑调低此参数。
### 2.11 optimizer_prune_level
该参数默认是打开的，这让优化器会更需要扫描的行数来决定是否跳过某些执行计划
### 2.12 optimizer_switch
这个变量包含了一些开启/关闭优化器特性的标志位。例如在mysql5.1中可以通过这个参数来控制禁用索引合并的特性。
应该根据实际需要来修改这些参数。





