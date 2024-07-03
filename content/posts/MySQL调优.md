+++
title=  'MySQL调优'
date= 2024-05-31T11:10:21+08:00

tags= ["MySQL", "数据库"]

categories= ["数据库"]

+++

### 什么是MySQL调优

MySQL调优是指对MySQL数据库进行性能优化的一系列操作，旨在提高数据库的运行效率、响应速度和稳定性。简单来说MySQL调优就是将原本执行较慢的SQL语句，通过一些列的优化转换成执行相对较快的SQL语句。

在MySQL的各种语句中（SELECT UPDATE INSERT DROP），SELECT语句往往是最需要进行调优的

### 调优金字塔

MySQL调优可以从多个方面进行，包括架构调优、MySQL本身调优、硬件调优

![image.png](https://raw.githubusercontent.com/EscapeBearSecond/BlogPic/main/img/202405311752555.png)

越往上成本、难度越来越高，但是带来的收益却是越来越小，所以在优化时，往往优先考虑下方的优化方式

> 架构调优

1. 在进行架构设计时，首先要考虑业务的实际情况，是否可以把不适合数据库做的事情放在其他服务中，如数据仓库、搜索引擎、数据缓存等等
2. 考虑数据库的并发量是否较大，是否采用分布式架构
3. 考虑读的压力是否较大，是否需要读写分离

> MySQL调优

1. 设计合理的表结构
2. 优化SQL语句
3. 添加索引

> 硬件调优

这个一般不需要太多的关注，如果是DBA的话，需要自己去学一些操作系统和硬件的知识

### 慢查询

#### 什么是慢查询

慢查询就是一条SELECT语句执行需要花费大量时间，这个时间往往不被系统或用户能接受（比如10s钟）

在MySQL中慢查询就是指执行时间超过MySQL服务器所设定的`long_quer_time`时间的SELECT语句，所有超过该时间的语句都会被记录在慢查询日志中。

在MySQL中可以通过`show VARIABLES like '%slow_query_time%'` `set global long_query_time=0`来查看和设置慢查询的时间阈值。（设置为0，就表示任何查询都是慢查询，都会被记录在慢查询日志中）

在MySQL中可以通过`show VARIABLES like 'slow_query_log'`和`set GLOBAL slow_query_log=1`来查询和开启慢查询日志，如果慢查询日志没有开启，则不会被记录。

如果希望将没有使用索引的SELECT语句也记录在慢查询日志中，可以通过`set VARIABLES 'log_queries_not_using_indexes'`来开启

在MySQL中通过`show VARIABLES like '%slow_query_log_file%'`来查看慢日志所在的磁盘位置

#### 为何会产生慢查询

其实产生慢查询的最终原因就是因为MySQL服务器要扫描的数据过多，这里可以是因为要扫描的数据行过多，也可能是因为要返回的数据列过多。所以MySQL调优主要是尽可能的让MySQL服务器只扫描自己需要的数据，不去扫描额外的数据，这样就能将MySQL性能发挥到最大。

> 几个概念

1. 响应时间：响应时间是指语句执行所花费的时间，它由服务时间和排队时间两个部分组成
   - 排队时间是指服务器因为等待某些资源而没有真正执行查询的时间，可能是等待IO操作完成，或等待行锁
   - 服务时间是指这条SELECT语句真正执行的时间
   - 这两个时间可以通过慢查询日志查看
2. 扫描行数：MySQL为了找到目标的数据，在服务器中所扫描的记录数
3. 返回行数：最终需要的记录数

显然对于一条SELECT语句，响应时间越短越好，扫描行数与返回行数的比，越小越好，但是最小是1，即扫描多少条数据，返回多少条数据

如果发现查询需要扫描大量的数据，但是只返回少量的数据，那么可以通过下面几种方法尝试优化：

1. 使用覆盖索引，把所有需要使用的列都放到索引中，减少回表次数
2. 改变库表结构，例如使用汇总表
   - 这个查询频繁使用，访问大量数据并进行复杂计算
   - 对实时性要求不高，可以接受一定程度的延迟更新汇总数据
3. 如果SELECT较为复杂，可以尝试重写优化，让MySQL优化器能够以更优化的方式执行这个查询

**注意**：在一条SELECT语句中，除非特殊情况，否则一定要避免使用`SELECT * FROM table`，最好是按需查询。我自己在刚工作时，就遇到过这种情况，在一张有两万多条记录的表中，有四五条记录的一个字段数据非常大（大约几MB），所以我当时使用`SELECT *`时，一直报慢查询相关错误，恰巧这个字段我并不需要。但是后面还是有大佬通过一些压缩算法把数据压缩了。

### 执行计划

#### 什么是执行计划

前面说了慢查询会被记录在慢查询日志中，那么如何排查这条慢查询是因何而导致呢？是因为没有使用索引？还是因为扫描的记录数较多？这时候就需要用到执行计划去排查究竟是什么原因了。

执行计划就是一条语句在经过MySQL查询优化器的各种基于成本和规则的优化后生成的一个执行计划，该计划展示了接下来具体执行的查询的方式，具体可以看到下面这些信息：

1. 表的读取顺序
2. 数据读取操作的操作类型（下面会有详细介绍）
3. 哪些索引可以使用、哪些索引被实际使用
4. 表之间的引用
5. 每张表有多少行被优化查询

在MySQL中可以通过`EXPLAIN`关键字查看执行计划，进行分析

#### 概览

通过`EXPLAIN SELECT ****`语句得到的执行计划一般如下表所示（先有个大致概念，后面会详细介绍）

| 字段名称 | id                                     | select_type | table | partitions     | type               | possible_keys  | key            | key_len              | ref                                                    | rows                     | filtered                                     | extra        |      |
| -------- | -------------------------------------- | ----------- | ----- | -------------- | ------------------ | -------------- | -------------- | -------------------- | ------------------------------------------------------ | ------------------------ | -------------------------------------------- | ------------ | :--: |
| 注释     | 一般一个SELECT对应一个id（有例外情况） | 查询的类型  | 表名  | 匹配的分区信息 | 针对单表的访问方法 | 可能用到的索引 | 实际用到的索引 | 实际使用到的索引长度 | 当使用索引列等值查询时，与索引列进行等值匹配的对象信息 | 预估的需要读取的记录条数 | 某个表经过搜索条件过滤后剩余记录条数的百分比 | 一些额外信息 |      |

#### id

在MySQL中，一般情况下一个SELECT关键字对应一个id，比如下面这个虽然它查询了两张表，但是只有一个SELECT关键字，所以只有一个id

![image-20240601112304584](https://raw.githubusercontent.com/EscapeBearSecond/BlogPic/main/img/202406011123624.png)

下面条SQL语句有两个SELECT关键字，故有两个id

![image-20240601113044604](https://raw.githubusercontent.com/EscapeBearSecond/BlogPic/main/img/202406011130651.png)

> 特殊情况

上面说一个SELECT关键字对应一个id，但是有一些情况比较特殊，即使有多个SELECT关键字，但是执行计划里只有一个id，下面我们来看看这些情况。

**包含子查询**

很多时候MySQL的查询优化器会将涉及子查询的查询语句进行重写，转换成连接查询，这时候即使你自己写的查询语句有两个SELECT关键字，但是通过MySQL优化器优化过后只有一个SELECT关键字，所以执行计划中只有一个id。

```sql
--优化前
SELECT * FROM user_basic WHERE user_basic.id IN (SELECT contact.id FROM contact WHERE contact.id < 10);
--优化后
SELECT user_basic.*  FROM user_basic JOIN contact ON user_basic.id = contact.id  WHERE contact.id < 10;
```

![image-20240601113738240](https://raw.githubusercontent.com/EscapeBearSecond/BlogPic/main/img/202406011137318.png)

为什么要优化？这里只能简述子查询被优化成`JOIN`的原因，其他查询语句的优化，容我后面专门写一篇文章讲解

1. 临时表的使用：执行子查询时，MySQL需要为内层查询语句（子查询）的查询结果建立一个临时表，然后外层查询语句从这个临时表中查询记录。这个过程会消耗大量的CPU和IO资源，产生大量的慢查询
2. 索引问题：子查询结果集存储的临时表，不论是内存临时表还是磁盘临时表，通常都不会存在索引，这会导致查询性能低下
3. 结果集过大的问题：如果子查询的结果集数量过多，会导致内存不足够建立临时表，从而会在磁盘中建立临时表。且如果涉及写操作，数据集大可能会导致持有锁的时间更长，影响其他并发查询的性能。

**包含UNION子句**

先观察下面这张图，看看有何不同

```sql
EXPLAIN SELECT * FROM user_basic UNION SELECT * FROM user_basic;
```

![image-20240601133408745](https://raw.githubusercontent.com/EscapeBearSecond/BlogPic/main/img/202406011334804.png)

虽然有两个`SELECT`语句，但是在执行计划中却有三个id，这是为什么？`UNION`子句会对并集的结果进行去重，怎么去重呢？MySQL使用的是内部的临时表，上图中`UNION`子句为了把id为1和2的结果集并集去重，在内部建立了一个名称为`<union1,2>`的临时表。

和`UNION`比起来，`UNION ALL`不需要去重，所有只有两个id

#### table

不论查询语句有多复杂，里面包含了多少张表，到最后也是对单表进行访问，MySQL规定EXPLAIN语句输出的每条记录对应着某个单表的访问方法/访问类型，该条记录的table列代表着该表的表名

#### partitions

和分区有关，一般情况下都是null

#### type

前面说EXPLAIN语句输出的每条记录对应着某个单表的访问方法/访问类型，其中type列就是具体的访问类型/访问方法，是一个非常重要的指标。出现较多的有七个值，结果值从好到坏依次是 `system > const > eq_ref > ref > range > index > all` 

一般来说，要保证查询至少达到`range`类型，最好能达到`ref`，下面分别介绍这几个的概念

> system

当表中只有一条记录，且该表使用的存储引擎的统计数据是精确的，比如`MyISAM、Memory`那么对该表的访问类型就是`system`

解释：什么是存储引擎的统计数据是精确的，存储引擎可以精确的维护表的大小和记录的统计信息，这使得查询优化器能够基于这些准确的数据做出更好的决策。具体原因可能要单独一篇文章解释，不知道我是否有时间能更新，望谅解。

> const

1. 当我们根据**主键**或**唯一二级索引**列与**常数**进行**等值**匹配时，对单表的访问就是const，因为只匹配一行，所以非常快。注意加粗字体！！

```SQL
EXPLAIN SELECT * FROM user_basic UNION ALL SELECT * FROM user_basic;
```

![image-20240601124232509](https://raw.githubusercontent.com/EscapeBearSecond/BlogPic/main/img/202406011242554.png)

2. 如果二级索引列有多列的话，那么每一列都需要与常数进行等值匹配，最后的访问类型type才是const

```SQL
EXPLAIN SELECT phone,email FROM user_basic WHERE phone = '110' AND email = '2923780891@qq.com';
```

![image-20240601124645487](https://raw.githubusercontent.com/EscapeBearSecond/BlogPic/main/img/202406011246530.png)

3. 对于唯一二级索引来说，查询该列为NULL值的情况比较特殊，因为唯一二级索引列并不限制 NULL 值的数量，所以上述语句可能访问到多条记录，也就是说is null不可以使用const访问方法来执行。

```sql
EXPLAIN SELECT phone,email FROM user_basic WHERE phone IS NUL
```

![image-20240603151140966](https://gitee.com/escapebear/blogpic/raw/master/img/202406031511007.png)

> eq_ref

在连接查询中，如果**被驱动表**是通过主键或唯一二级索引列等值匹配的方式进行访问的，则对该驱动表的访问类型是eq_ref。

注意：A表和B表进行连接查询，如果通过A表的结果集作为循环基础数据，然后一条一条的通过结果集中的数据作为过滤条件到B表中查询，然后合并结果，那么A表就是驱动表，B表就是被驱动表

```sql
EXPLAIN SELECT * FROM user_basic u1 LEFT JOIN user_basic u2 ON u1.id = u2.id
```

![image-20240603151850203](https://gitee.com/escapebear/blogpic/raw/master/img/202406031518271.png)

从执行结果中看，被驱动表是u2，驱动表是u1，u2的访问方式是eq_ref，表明在访问u2表时可以通过主键进行等值匹配来访问。

> ref

当通过普通二级索引列与等值进行匹配时，那么对该表的访问方法可能是ref。本质上也是一种索引访问，但是不是唯一索引，索引可能会有多条匹配的结果。

```sql
EXPLAIN SELECT *FROM user_basic WHERE name='DYG'
```

![image-20240603153609373](https://gitee.com/escapebear/blogpic/raw/master/img/202406031536438.png)

> range

如果使用**索引**获取某些范围间的记录，那么访问方法可能就是range，一般就是在WHERE语句中出现了between、<、>、!=、in等。这种范围扫描比全表扫描略好。

```sql
EXPLAIN SELECT *FROM user_basic WHERE name!='DYG'
```

![image-20240603154013822](https://gitee.com/escapebear/blogpic/raw/master/img/202406031540890.png)

> index

当使用覆盖索引，但是需要扫描全部索引时，对表的访问类型就是index

```sql
EXPLAIN SELECT name,salt FROM user_basic WHERE salt != '111'
```

![image-20240603154237083](https://gitee.com/escapebear/blogpic/raw/master/img/202406031542129.png)

> all

全表扫描，为找到需要的数据，需要遍历全部的数据行

#### possible_keys与key

possible_keys表示在对某个表进行单表查询时可能会用到的索引，key表示实际用到的索引，为NULL则表示没有使用索引

```sql
EXPLAIN SELECT phone,email FROM user_basic WHERE phone = '110' AND email = '2923780891@qq.com';
```

![image-20240603155211473](https://gitee.com/escapebear/blogpic/raw/master/img/202406031552519.png)

#### key_len

key_len表示实际使用的索引所能记录的最大长度，比如上面使用的索引是phone char(15)，该表的编码是utf8mb4，那么该列的最大长度就是15*4=60字节，还有1字节用来记录值的实际长度。

#### rows

对某个表执行查询时，rows表示预计扫描的行数

```SQL
EXPLAIN SELECT *FROM user_basic WHERE name!='DYG'
```

![image-20240603155916494](https://gitee.com/escapebear/blogpic/raw/master/img/202406031559547.png)

> filtered

查询优化器预测有多少条记录满⾜其余的搜索条件，什么意思呢？看具体的语句：

```sql
EXPLAIN SELECT * FROM user_basic AS u INNER JOIN user_basic AS c ON u.id=c.id WHERE u.email > 'choyeeku@gmail.com' and u.phone > '12312'
```

![image-20240603170826481](https://gitee.com/escapebear/blogpic/raw/master/img/202406031708522.png)

从执行计划中可以看到，查询优化器将u看作驱动表，c看作被驱动表，u表扫描的行数预计是2787，filtered与等于50，说明过滤出2787*0.5=1394条数据，所以被驱动表只需要再进行大约1394次查询即可。

#### extra

额外信息：是否使用索引、是否使用where等等（关注度不高）

### 查询优化器执行过程

![b0dc955876354f09938142e60bc1f4ce](https://gitee.com/escapebear/blogpic/raw/master/img/202406031712507.png)

1.如果是查询语句（select语句），首先会查询缓存是否已有相应结果，有则返回结果，无则进行下一步（如果不是查询语句，同样调到下一步）

2.解析查询，创建一个内部数据结构（解析树），这个解析树主要用来SQL语句的语义与语法解析；

3.优化：优化SQL语句，例如重写查询，决定表的读取顺序，以及选择需要的索引等。这一阶段用户是可以查询的，查询服务器优化器是如何进行优化的，便于用户重构查询和修改相关配置，达到最优化。这一阶段还涉及到存储引擎，优化器会询问存储引擎，比如某个操作的开销信息、是否对特定索引有查询优化等。

