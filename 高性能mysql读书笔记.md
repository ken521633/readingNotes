#高性能mysql读书笔记

+ 事务的原则 acid （原子性 atomicity、一致性 consistency、隔离性 isolation、持久性 durability）

+ mvcc 多版本并发控制

	

+ 事务隔离界别
	1.read uncommitted   	未提交读	容易脏读 性能也没有很大的提升	 最低
	2.read comitted		提交读	大多数数据库的默认级别（mysql不是）	不可重复读
	3.repeatable read   重复读	mysql 默认级别  不解决幻读
	4.Serializable	    可串行化	最高级别  解决幻读 但性能太差，除非是必须严格确认数据一致性的情况下 考虑采用此级别


+ 设置事务级别
```
	set session transaction isolation level read commited 
```

```
	<!-- 查看当前事务提交  0 off  1 on-->
	show variables like ‘autocommit’
	<!-- 设置 事务提交 -->
	set autocommit=1;
```

+ 显式锁和隐式锁

   显式锁方法
```
	select XXXX lock in share mode
	select XXX for update
```

**建议**:除了事务中禁用autocommit可以使用lock tables 其他任何形式的引擎都不要使用显式锁

+ 查看数据库中表状态

```
	show table status like 'tableName' \G
```

---

+ 存储引擎

> +  **innodb**  mysql默认的存储引擎，也是使用最广泛的存储引擎，可以崩溃后安全恢复
   + **myISAM**  5.1以前的默认引擎，不支持事务和行级锁，崩溃后无法安全恢复。
    + 修复表命令： 
	<!--检查表错误-->
	check table tableName
	<!--修复表 -->
	repair table tableName
     + 延迟更新索引
	如果指定了delay_key_write,每次修改完，不会将修改的索引写入磁盘，会写入内存中的键缓冲区，只有在清理缓冲区或关闭表时才会写入磁盘，极大提升写入速度
     + 压缩表
	如果倒入的数据不需要再进行修改操作，使用<i>myisampack</i>对表进行压缩，减少磁盘IO，压缩后的表是不允许修改的，包括索引，提升查询性能。
  + **archive**
    + 只支持插入和查询操作，不支持事务
    + 利用zlib对插入进行压缩，所以比myISAM磁盘io更少
    + 使用场景：日志和数据采集
    + 支持行级锁和专用的缓冲区，所以可实现高并发插入，从查询开始一直到返回表中存在的所有行数之前，会阻止其他的select执行，保证数据一致性读。
 + blackhole   不推荐，不好用，没有实现任何存储机制
 + cvs
	可以将普通的cvs文件作为mysql的一张表来处理，一般用来作为数据交换的引擎来处理。
+ federated
	通常不用，作为代理来访问远程mysql服务端。
+ memory
	是将数据放入内存中，支持hash索引，并发写入性能低，不支持blob和text且每列长度固定，即使设计为varchar，实际也会变为char，造成资源浪费。
+ merge
	不支持分区，属于myISAM的变种，由多个myISAM表合并来的虚拟表。已经被弃用
+ NDB
	集群存储引擎。

+ 第三方存储引擎

> + oltp类引擎
    + XtraDB ： perconaServer和mariadb中使用，可以做为innodb的替代品，可以兼容innodb的读写，并且支持innodb的所有查询。
    + pbxt：mariadb包含该引擎，对blob和text做了优化，支持ssd

+ 基准测试

>	吞吐量：单位时间内的事务处理数。使用工具通常为TPC-C

> 在线事务处理（OLTP）  测试单位  每秒事务数（TPS）  每分钟事务数(TPM)

> 并发性 ： 任意时间有多少同时同时发生的并发请求,测试时记录msql中的 threads_running

> 可扩展性 ： 理想状态下，给系统增加一倍的工作，在理想情况下能获得两倍的效果，或者给系统增加一倍的资源（如单核变双核），可以获得2倍的吞吐量

> 一般测试方法：系统预热后，读取操作一般在3-4个小时，写操作基本在8小时左右
	
>  收集mysql的测试数据shell

```
  #!/bin/sh

  INTERVAL=5
  PREFIX=$INTERVAL-sec-status
  RUNFILE=/home/sammy/running
  mysql -e 'SHOW GLOBAL VARIABLES' >> mysql-variables
  while test -e $RUNFILE;do
  	file=$(data +%F_%I)
	sleep=$(data+%s.%N | awk "{print $INTERVAL - (\$1 % $INTERVAL)}")
	sleep $sleep
	ts="$(data+"TS %s.%N %F %T")"
	loadavg="$(uptime)"
	echo "$ts $loadavg" >> $PREFIX-${file}-status
	mysql -e 'SHOW ENGINE INNODB STATUS\G' >> $PREFIX-${file}-innodbstatus &
	echo "$ts $loadavg" >> $PREFIX-${file}-processlist
	mysql -e 'SHOW FULL PROCESSLIST\G' >> $PREFILE-${file}-processlist & 
	echo $ts
  done
  echo exiting because $RUNFILE does not exists
```
测试工具  ： ab   http_load  jmeter
压测工具  ：  sysbench


尽量不要用null ，可能会增加索引的成本，比如可能将固定长度索引变为不固定，mysql也会对null进行特殊处理,例外为bit类型，但不适合myisam
timestamp只占用datetime一半的存储空间

 | tinyint | smallint | mediumint | int | bigint |
  ---   |   ---  |  ---		|   ---	 | ---      |
  8    |   16   |    32   |    64  |  128

  > +  整数类型有个unsigned，表示不允许为负值,可以将正数上限提高一倍， tinyint 不加上 unsigned为-127-128  加上后变为0-255
  >  + decimal 最多存储65个数字
  > + mysql 支持enum  和varchar做关联 效率很快  但是enum和enum做关联会慢
  > + mysql 中的timestamp为4个存储空间，但是支持到2038年，datetime占8个存储空间，可以支持无限,timestamp的空间效率比datetime高
  >  + bit最多可以存储64个位
  > + 存储ip地址，可以使用无符号整数来存储，inet_aton()和inet_ntoa()分辨用来做加密和解密的操作
  > + 不建议的做法  1. 太多的列 2.太多的关联，建议极限为12个  3.全能的枚举  4. Null
  > 范式化优点： 1.更新操作更快 2.数据量一般都会少 3.更少的使用group by或者distinct  缺点： 1.关联查询效率会很低

  >汇总表和缓存表
   + 缓存表 每次读取比较慢的
   + 汇总表 需要聚合函数进行统计的表

> 物化视图

> 计数器表

> alter table 加速操作

   + 方法一 ： 现在一台不提供服务的机器上执行alter table，然后和提供服务的主库进行交换
   + 方法二 ： 影子拷贝，原理是先创建一张新表，然后然后通过重命名和删除表来完成，工具选择：online schema change（facebook） openark toolkit(noach)

  +  修改默认值方法
		++ alter table tableName.film modify column columnName tinyint(3) not null default 5 (慢)
		++ alter table tableName.film alter column columnName set default 5 (快) 
	 + <b>只要牵扯到modify column的操作都会导致表重建</b>
	+ 下面技术是不需要重建表(建议备份数据)
	  + 移除一个列的auto_increment属性
	  + 增加 移除 修改 enum或set常量
	+ 基本技术是创建一个新的frm文件，替换掉以前的那张表的frm
	+ 具体操作
	  + 创建一张相同的空表，进行修改
	  + 执行语句 flush tables with read lock ，关闭正在使用的表，并禁止被打开
	  + 交换frm文件
	  + 执行unlock tables释放前面的锁   
	  -----
###关于索引
	   
 > 分类

   + B-Tree 索引
       最常见的索引，Archive引擎除外不支持该索引，5.1以前不支持任何索引，5.1以后开始支持自增列索引。
	   该索引支持一下查询方式：

	   > 1.全值匹配
	   
	   > 2.匹配最左前缀
	   
	   > 3.匹配列前缀
	   
	   > 4.匹配范围值
	   
	   > 5.精确匹配某一列并匹配另外一列
	   
	   > 6.只访问索引的查询
	   
	   >7.支持orderby
	   
	   限制

	   > 如果不是按照索引最左列开始查找，则无法使用索引

	   > 不能跳过索引中的列

	   > 如果查询中有某个列是有查询范围的，则其右边的所有的列都无法使用索引优化查找
	
	+ hash索引

		只有精确匹配索引所有列的查询才有效，对每一行数据都会计算出一个hash码，hash索引将所有的hash码存在索引中，同时每个hash码存储一个对应该数据的指针。
		
		目前只有memory和NDB集群引擎显示支持hash索引。支持非唯一hash索引。如果多个列的hash码相同，则索引会以链表的形式存放多个记录指针到同一个hash条目中。

		限制

		> 1.无法用于排序

		> 2.不支持部分索引列查找，例如创建A B两个列的索引，如果只查A，则索引不成立

		> 3.只支持等值比较查询，如 = > < in ，不支持范围查询

		> 4.如果hash冲突很多，维护操作的代价很大。

		可以在innodb引擎上创建一个自定义hash索引（伪索引），