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