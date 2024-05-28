---
title: MySQL
---
<%- toc(page.content) %>
<style type="text/css">* {font-family: YaHei Consolas Hybrid, Consolas;}</style>

# 引擎

## MyISAM

优点：

- 压缩表占用空间小
- 保存了表的总行数，不加条件查询行数时速度快

缺点：

- 不支持行级锁（读取时对表加共享锁，写入时对表加排它锁）
- 因为不支持事务，写操作异常中断时可能导致数据丢失，需要进行修复，非常耗时

特点：

- 不支持外键
- 并发插入（CONCURRENT INSERT）：在表被查询的同时，支持往其中插入新纪录
- 支持空间数据索引

## InnoDB

优点：

- 支持事务
- 支持行级锁
- 支持在线热备份

缺点：

- 不保存表的总行数，查询行数时会尝试遍历一个尽可能小的索引
   > 不保存行数的原因：由于事务特性，一张表的行数在同一时刻对于不同的事务可能是不同的

特点：

- 支持外键
- 表必须有聚集索引

# 索引

## B+Tree 索引（BTREE）

### 聚集索引（clustered index）

InnoDB 的聚集索引的叶子结点存储行记录，因此 InnoDB 必须要有且仅有一个聚集索引。

InnoDB 选择聚集索引的逻辑如下：

1. 如果 InnoDB 表显式地指定了主键，那么会选择主键作为聚集索引
2. 如果 InnoDB 表没有显式指定主键，那么会选择第一个非空唯一索引作为聚集索引
3. 如果以上条件均不满足，那么 InnoDB 会选择一个隐式的 6 字节 rowid 作为聚集索引

### 辅助索引（普通索引/二级索引 secondary index）

InnoDB 的辅助索引的叶子结点存储键值和对应行的聚集索引值。

### 非聚集索引

MyISAM 的 B+Tree 索引结构中，叶子结点存储的额是数据文件的指针。

### 覆盖索引

#### 参考文章

[什么叫做覆盖索引？](https://blog.csdn.net/Aplumage/article/details/117015144)

[什么是覆盖索引？什么是回表查询？怎样实现覆盖索引？](https://blog.csdn.net/lida776537387/article/details/105377731?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-105377731-blog-120846042.pc_relevant_recovery_v2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-105377731-blog-120846042.pc_relevant_recovery_v2&utm_relevant_index=2)

#### 定义

对一张表的查询语句中，如果 select 子句与 where 子句涉及的所有字段，是这张表的某一个索引包含字段的子集，则称该查询使用了覆盖索引。

覆盖索引时，在一颗索引树上即可获取到 SQL 所需的所有列数据，无需回表，速度更快。

MySQL 只能使用 BTREE 索引做覆盖索引。

## 哈希索引（HASH）

哈希索引能以 O(1) 的时间复杂度进行查找，但失去了有序性，因此只支持精确查找，无法用于部分查找和范围查找，无法用于排序与分组。

自适应哈希索引：InnoDB 存储引擎的特殊功能。当某个**辅助索引**被使用得非常频繁时，会对该索引建立哈希索引，这样就让 B+Tree 索引具有哈希索引的一些优点，比如快速的精确查找。

## 全文索引（FULLTEXT）

一般使用倒排索引实现，记录了关键词到其所在文档的映射。

- 只有MyISAM、InnoDB（MySQL 5.6.4 版本开始）支持全文索引
- MySQL 5.7.6 中内置了支持中、日、韩文（CJK）的 ngram 全文解析器

# 锁

## MDL

### 参考文章

[mysql创建索引导致死锁，数据库崩溃，mysql的表级锁之【元数据锁（meta data lock，MDL)】全解](https://blog.csdn.net/A_art_xiang/article/details/127811653)

[MYSQL 持续踩坑之-metadata lock](https://www.jianshu.com/p/00818698b80b)

[关于mysql Meta Lock 机制详细讲解](http://blog.itpub.net/29896444/viewspace-2101567/)

### 总结

metadata lock，元数据锁，表级锁。

所有的 DDL 会对涉及表加 MDL 互斥锁，所有的 DML、DQL 会对涉及表加 MDL 共享锁。

![MySQL MDL](../img/MySQL/MySQL%20MDL.png?raw=true)

如何安全地在生产环境执行DDL：

- 在业务低峰期执行
- 在执行 DDL 的 session 设置一个较短的锁等待超时时间，避免 DDL 的 MDL 互斥锁长时间阻塞后续的 DQL、DML，DDL 若执行超时可以择时重试
   ```sql
   set lock_wait_timeout = 5; -- 单位：秒
   ```

## InnoDO 行锁

### 参考文章

[MySQL记录锁、间隙锁、临键锁（Next-Key Locks）详解](https://www.jianshu.com/p/478bc84a7721)

### 总结

InnoDB 中的行锁的实现依赖于**索引**。某个加锁操作如果没有使用到索引，就会**退化**为对表加锁。

MyISAM 不支持行锁。

记录锁（Record Locks）：存在于**唯一索引（包含主键索引）**中，锁定单条记录。

间隙锁（Gap Locks）：存在于**非唯一索引**中，锁定开区间范围内的一段间隔，RC 隔离级别下不生效。

临键锁（Next-Key Locks）：存在于**非唯一索引**中，锁定一段左开右闭的区间，RC 隔离级别下不生效。

# 事务隔离级别

## 查看

```sql
-- 查看区局隔离级别
select @@global.transaction_isolation;
-- 查看会话隔离级别
select @@transaction_isolation;
show variables like 'transaction_isolation';
```

## 设置

### SQL

```sql
-- 设置全局隔离级别
set global transaction isolation level <隔离级别>;
-- 设置会话隔离级别
set session transaction isolation level <隔离级别>;
```

隔离级别：

- READ UNCOMMITTED（读未提交）
- READ COMMITTED（读已提交）
- REPEATABLE READ（可重复读）
- SERIALIZABLE（串行化）

### 配置文件

```ini
[mysqld]
transaction-isolation = <隔离级别>
```

隔离级别：

- READ-UNCOMMITTED（读未提交）
- READ-COMMITTED（读已提交）
- REPEATABLE-READ（可重复读）
- SERIALIZABLE（串行化）

## 提交读 READ COMMITTED

### 不可重复读测试

|结果|事务1|事务2|结果|
|-:|-|-|-|
||start transaction;|||
|1|select v from t where id = 1;|||
|||update t set v = 2 where id = 1;||
|<font color="red">2</font>|select v from t where id = 1;|||

结论：发生不可重复读。
> 【注】显式加锁（如`select v from t where id = 1 lock in share mode;`）可以正确锁行，与 RR 级别一样

### 幻读测试

|结果|事务1|事务2|结果|
|-:|-|-|-|
||start transaction;|||
|0|select count(1) from t;|||
|||insert into t values();||
|<font color="red">1</font>|select count(1) from t lock in share mode;|||
|||insert into t values();|<font color="red">执行</font>|
|<font color="red">2</font>|select count(1) from t;|||

结论：发生幻读，无间隙锁。

## 可重复读 REPEATABLE READ

### 不可重复读测试

|结果|事务1|事务2|结果|
|-:|-|-|-|
||start transaction;|||
|||select v from t where id = 1;|1|
|||update t set v = 2 where id = 1;||
|2|select v from t where id = 1;|||
|||update t set v = 3 where id = 1;||
|2|select v from t where id = 1;|||
|<font color="red">3</font>|select v from t where id = 1 lock in share mode;|||
|||update t set v = 4 where id = 1;|阻塞|
||rollback;||阻塞|
||||执行|

结论：快照读时，解决不可重复读；当前读时，发生不可重复读，有记录锁。

### 幻读测试

|结果|事务1|事务2|结果|
|-:|-|-|-|
||start transaction;|||
|0|select count(1) from t;|||
|||insert into t values();||
|||select count(1) from t;|1|
|0|select count(1) from t;|||
|<font color="red">1</font>|select count(1) from t lock in share mode;|||
|||insert into t values();|阻塞|
||rollback;||阻塞|
||||执行|

结论：快照读时，解决幻读；当前读时，发生幻读，有间隙锁。

# MVCC

## 说明

InnoDB 中通过 MVCC（多版本并发控制）实现了快照读，使得读-写重读不加锁，做到非阻塞并发读，提供数据库并发性能。

当前读（悲观锁）：

- 共享锁
   - select ... lock in share mode
   - lock table ... read
- 排它锁
   - select ... for update
   - lock table ... write
   - update
   - insert
   - delete

快照读（不加锁）：

- 可以解决读-写导致的并发一致性问题，即脏读、幻读、不可重复读
- 隔离级别为 SERIALIZABLE 时会退化为当前读

## 隐式字段

表中的每行记录除用户自定义字段外，还有数据库隐式定义的字段：

- DB_ROW_ID：6 字节，隐含的自增ID（隐藏主键）。如果数据表没有主键和唯一键，InnoDB 会自动以 DB_ROW_ID 产生一个聚集索引
- DB_TRX_ID：6 字节，最近修改/插入事务ID
- DB_ROLL_PTR：7 字节，回滚指针，指向这条记录的上一个版本（存储于 rollback segment 中）
- DELETED_BIT：1 字节，记录被更新或删除时设置为 true

## undo log

存储于 rollback segment 中的单向链表，头节点为表中的记录，每条记录的 DB_ROLL_PTR 指向比它更旧的最新记录。

## Read View

可以把 Read View 简单理解为有三个全局属性：

- trx_list：未提交事务ID列表，用来维护 Read View 生成时刻系统中正活跃的事务ID
- up_limit_id：记录 trx_list 中最小的事务ID
- low_limit_id：Read View 生成时刻系统待分配的下一个事务ID，即当前已出现过的最大事务ID + 1

## 整体流程

查找满足以下全部条件的最新记录：

- DB_TRX_ID 对应事务在 Read View 生成之前创建（DB_TRX_ID < low_limit_id）
- DB_TRX_ID 对应事务已提交（!trx_list.contains(DB_TRX_ID)）

![MySQL MVCC](../img/MySQL/MySQL%20MVCC.png?raw=true)

## 不同隔离级别实现

RC 隔离级别下，每个快照都会生成并获取最新的 Read View；而在 RR 隔离级别下，一个事务中只有第一个快照读会创建 Read View，之后的快照读获取的都是同一个 Read View。

|结果|事务1|事务2|结果|
|-:|-|-|-|
||*-- 隔离级别：RC*<br />start transaction;|||
|0|*-- 快照读，生成 Read View 1*<br />select count(1) from t;|||
|||insert into t values();||
|1|*-- 快照读，生成 Read View 2，**幻读***<br />select count(1) from t;|||

|结果|事务1|事务2|结果|
|-:|-|-|-|
||*-- 隔离级别：RR*<br />start transaction;|||
|0|*-- 快照读，生成 Read View 1*<br />select count(1) from t;|||
|||insert into t values();||
|0|*-- 快照读，读取 Read View 1，**无幻读***<br />select count(1) from t;|||
|1|*-- 当前读，**幻读***<br />select count(1) from t lock in share mode;|||

# redo log

## 参考文章

[Mysql中的redo log](https://blog.csdn.net/weixin_43213517/article/details/117457184)

## 使用 redo log 的原因

![MySQL redo log](../img/MySQL/MySQL%20redo%20log.jpg?raw=true)

InnoDB 以页为单位对存储空间进行管理，而页大小（innodb_page_size）默认为 16KB。若每次修改几个字节的数据后，就将整个数据页刷回磁盘，太浪费。

事务的持久性要求，事务提交后，其所做的修改将会永远保存到数据库中。即使系统发生崩溃，事务执行的结果也不能丢失。一个事务中不同语句设计的数据通常在不同数据页中，若每次事务提交时都将涉及的数据页刷回磁盘，则涉及较多的随机IO，效率太低，不如使用 redo log 的顺序写来代替。

## 相关参数

- innodb_flush_log_at_trx_commit\
定义了 InnoDB 在每次事务提交时，如何处理未刷盘的 redo log 数据。取值：

   - 0：每秒将 redo log buffer 写入 page cache 并刷盘
   - 1：（默认，最满足持久性要求）每次事务提交时都将 redo log buffer 写入 page cache 并刷盘
   - 2：每次事务提交时都将 redo log buffer 写入 page cache，每秒刷盘
   > page cache 的刷盘可由内核进行调度，即使用户进程在主动刷盘前崩溃，只要系统没有崩溃，数据就不会丢失。

- innodb_flush_log_at_timeout\
刷盘时间间隔上限（秒），取值范围[1, 2700]，默认值 1,。MySQL 进程意外终止时最多丢失 innodb_flush_log_at_timeout 秒的数据。

- sync_binlog\
控制 binlog 刷盘频率。取值：
   - 0：MySQL 不做刷盘，由操作系统控制
   - 1：（默认，最满足持久性要求）每次事务提交前刷盘
   - N（>1）：每 N 个事务提交后刷盘

默认配置最满足持久性要求，但写操作性能最低。若要提高写操作性能，就要放弃一些持久性：

- innodb_flush_log_at_trx_commit=2
- sync_binlog 设置为大于 1 的值，如 500 或 1000

# 主从复制 & 读写分离

## 参考文章

[Mysql主从复制](https://www.cnblogs.com/ccywmbc/p/16614594.html)

[MySQL主从延迟的解决方案](https://blog.csdn.net/KIMTOU/article/details/125033199)

[数据一致性冲突的解决方案](https://blog.csdn.net/java_lifeng/article/details/105803619#t4)

## 主从复制

### 角色

![MySQL 主从复制](../img/MySQL/MySQL%20%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6.jpg?raw=true)

- 主库的增删改操作会全部记录在 binlog 中
- 从库通过 I/O 线程向主库请求 binlog 数据，主库 Dump 线程负责提供数据，而后从库 I/O 线程将获得的数据写入 relay log
- 从库 SQL 线程读取 relay log，解析成 SQL 语句并执行

### 模式

#### 异步模式

默认模式。

主节点执行完客户端提交的请求后立即返回，不会主动推动 binlog 到从节点。从节点自行向主节点请求 binlog。

从节点数据可能存在延迟。如果主节点故障，将从节点提升为主节点，可能导致数据丢失。

#### 全同步模式

主节点执行完事务后，必须等待所有从节点数据同步完成，主节点才能提交事务并向客户端返回结果。

实现了强一致性，但性能必定较差。

#### 半同步模式

主节点执行完事务后，必须等待至少一个从节点完成将 binglog 数据写入 relay log，主节点才能提交事务并向客户端返回结果。

表现介于异步模式与全同步模式之间。与异步模式相比，解决了数据丢失的问题，但主节点的响应延迟至少增加一个 TCP/IP 往返时间，因此最好在低延时网络中使用（至少有一个从节点与主节点之间的网络延时低）。

## 读写分离

将写请求全部发送给主节点，读请求全部发送给从节点。可以通过代理（如 Amoeba）实现，也可以在客户端中增加逻辑进行控制。

好处：
- 缓解锁争用
- 从服务器可以使用 MyISAM，提升查询性能以及节约系统开销
- 增加冗余，提高可用性

# 分区

## 参考文章

[MySQL数据库：分区Partition](https://blog.csdn.net/a745233700/article/details/85250173)

[MySQL分区表详解](https://blog.csdn.net/frostlulu/article/details/122304238)

## 说明

按分区键将数据存储于不同的文件中。

优点：

- 可缩小查询时遍历范围，提高查询性能
- 对于一些处理，可在不同分区并发执行
- 可将数据存储于不同磁盘，解决单磁盘的容量与 IO 性能瓶颈，允许一张表存储更多数据
- 分区数据的删除与恢复与无分区相比更高效

缺点：

- 每个主键与 Unique Key 都必须包含所有分区键字段
- 单表最大分区数为 1024

# InnoDB 页结构

## 参考文章

[InnoDB数据存储结构](https://blog.csdn.net/m0_51295655/article/details/122990223)

[MYSQL 单表可以放多少数据是怎么计算出来的](https://developer.aliyun.com/article/1473169)

## 说明

InnoDB 引擎页大小为 16KB（`show variables like 'innodb_page_size';`），其中固定大小的结构占用 128B：

|名称|占用空间|说明|
|File Header|38B|文件头|
|Page Header|56B|页头|
|Infimum + Supermum|26B|最大、最小记录|
|File Trailer|8B|文件尾|

剩余 16256B 为非固定大小结构：

- User Records：用户记录，存储行记录内容
- Free Space：空闲空间
- Page Directory：页目录，用户记录的偏移量的二分查找稀疏索引

## 单表最大数据量

这个问题一般是指，三层B+树聚集索引可存储的数据量。

![MySQL 单表最大数据量](../img/MySQL/MySQL%20%E5%8D%95%E8%A1%A8%E6%9C%80%E5%A4%A7%E6%95%B0%E6%8D%AE%E9%87%8F.png?raw=true)

计算公式：$x^{z-1} \times y$

- x：非叶子结点内指向其他内存页的指针数量
- y：叶子结点内可容纳记录数，直接受到单记录大小的影响
- z：B+树层数，当前问题中取值为 3

问题在于 x、y 的计算，目前本人尚未找到明确方法。比较常见的计算结果是，单记录大小为 1KB 时，单表最大数据量为 2 千万条。

# SQL 优化

## 尽量用上索引

### 参考文章

[探索MySQL not in到底走索引吗？](https://blog.csdn.net/qq_36588424/article/details/125101516#t4)

### 组合索引

查询条件中使用组合索引字段时:

- 组合索引中第一个未出现在查询条件中的字段之前的字段生效，对于生效的字段应用范围查询
- 若不存在生效字段（即查询条件中没有用到组合索引的第一个字段），则对组合索引进行全索引扫描

创建组合索引时，应将更常用于作为查询条件的字段放在更前面。

假设一张表`t`中存在组合索引`(x, y, z)`（类型均为 int not null），则：

```sql
-- 使用组合索引中 x 字段范围查询
-- type: range, key_len: 4
select count(1) from t where x = ?;

-- 使用组合索引中 x、y 字段范围查询
-- type: range, key_len: 8
select count(1) from t where x = ? and y = ?;

-- 使用组合索引中 x 字段范围查询
-- type: range, key_len: 4
select count(1) from t where x = ? and z = ?;

-- 使用组合索引全索引扫描
-- type: index, key_len: 12
select count(1) from t where y = ? and z = ?;
```

> 【注】Extra 均为 Using where; Using index

### 其他细项

前导模糊查询无法使用索引，如`like '%xxx'`。

对于数据区分度不明显的字段（如枚举字段），使用索引收益不高。

负向查询时，如果 MySQL 认为*全表扫描*比*走索引+回表*效率高，就不会使用索引。可以认为，当查询数据量比较大时，若查询未覆盖索引，就不会使用索引。

例：`user`表有 1000 条数据，`id`为主键，`name`为普通索引，除此之外还有其他字段：

```sql
-- 未覆盖索引，但查询数据量不大，所以使用索引后回表查询
-- type: range, Extra: Using index condition
select * from user where name in ('张三', '李四');

-- 未覆盖索引。若使用索引，须回表查询其余字段，所以不使用索引，而是全表扫描
-- type: ALL, Extra: Using where
select * from user where name not in ('张三', '李四');

-- 覆盖索引，无需回表查询
-- type: index, Extra: Using where; Using index
select name from user where name not in ('张三', '李四');
```

## 减少查询返回的数据量

- 只查询需要查询的字段
- 确定只查询固定量的数据时，使用`limit`，例如查询是否存在满足条件的数据：
   ```sql
   select 1 from table_name where ... limit 1;
   ```

## 查询条件顺序

能够排除更多数据的查询条件应更先执行。

在没有优化器介入的情况下，where 子句中的查询条件执行顺序：

- 从前往后：MySQL
- 从后往前：Oracle、SQL Server

## 减少关联查询

数据量极大或使用分库分表时，可通过冗余字段、变更表结构或改变业务设计的方式，尽量避免关联查询。

## 驱动表

### 参考文章

[MySQL中的驱动表和被驱动表](https://baijiahao.baidu.com/s?id=1683057963945350857&wfr=spider&for=pc)

### 说明

若多表连接查询的执行计划中，被驱动表的`Extra`为`Using join buffer`，则执行时：

1. 读取驱动表数据放入 join buffer
2. 读取被驱动表的数据，与 join buffer 中的数据进行匹配
3. 如果匹配上则保留，否则丢弃

MySQL 会为每一个 join 查询语句都分配一个 join buffer，大小为当前配置的 `join_buffer_size`（默认值 262144B，取值范围[128, 4294967168]）。若 join buffer 不足以容纳驱动表数据，则须将驱动表数据分批次放入 join buffer 与被驱动表进行匹配。所以：

- 应尽量以小表作为驱动表
- 配置合适的`join_buffer_size`有助于提高 join 查询的效率

### 判断驱动表

- <驱动表> left join <被驱动表>
- <被驱动表> right join <驱动表>
- inner join：MySQL 自动选择小表作为驱动表，大表作为被驱动表
   > 【注】关于小表、大表的辨析不明。\
   在 MySQL8 进行测试：表`B`在表`A`的基础上增加一个`varchar(255)`字段，`A`有`1001`跳数据，`B`有`1000`条数据，两张表的`id`都是从 1 连续递增。
   > ```sql
   > -- A 为驱动表（虽然 A 行数更多，但真正参与关联查询的数据量占用空间更小）
   > select * from A inner join B using(id);
   > 
   > -- 执行计划中 B 为驱动表，树形执行计划中 A 为驱动表
   > select * from A inner join B using(id) where A.id <= 1001 and B.id <= 1000;
   > ```
- <驱动表> straight_join <被驱动表>

## 慢查询日志

### 配置

```sql
-- 启用慢查询日志
set global slow_query_log = on;

-- 设置慢查询阈值（秒），全局修改后仅对新会话生效
set global long_query_time = 1;

-- 日志文件名
show global variables like 'slow_query_log_file';

-- 日志文件默认路径
show global variables like 'datadir';
```

### 查看

- Time：慢查询发生事件
- User：发起查询的客户端信息
- Query_time：查询耗时
- Lock_time：等待锁表时间
- Rows_sent：查询结果行数
- Rows_examined：查询执行期间从存储引擎读取的行数

例：
```
# Time: 2022-10-22T04:29:01.502656Z
# User@Host: root[root] @ localhost [127.0.0.1]  Id:    15
# Query_time: 1.629093  Lock_time: 0.000000 Rows_sent: 501  Rows_examined: 2000501
SET timestamp=1666412939;
/* ApplicationName=IntelliJ IDEA 2022.2.3 */ select name from person_info_large order by name desc;
```

## EXPLAIN 查看执行计划

### 查看文章

[&#91;MySQL高级&#93;(一) EXPLAIN用法和结果分析](https://blog.csdn.net/why15732625998/article/details/80388236)

### 字段：id

可以简单理解为嵌套深度。数值越大越先执行，数值相同按从上到下的顺序执行。

### 字段：select_type

查询类型。

- `SIMPLE`：简单查询（不使用union或子查询）。
- `PRIMARY`：如果包含union语句或者子查询，则最外层的查询部分标记为PRIMARY。
- `UNION`：union语句中第二个及后面的查询。若union语句包含在from子句的子查询中，外层select将被标记为DERIVED。
- `DEPENDENT UNION`：union语句中第二个及后面的依赖外部查询的查询。
- `UNION RESULT`：联合查询的结果。
- `SUBQUERY`：子查询中的第一个查询。
- `DEPENDENT SUBQUERY`：子查询中的第一个查询，且依赖外部查询。
- `DERIVED`：用到派生表的查询。在from列表中包含的子查询被标记为DERIVED（衍生），MySQL会递归执行这些子查询，把结果放在临时表（派生表）中。
- `MATERIALIZED（不重要）`：被物化的子查询。
- `UNCACHEABLE SUBQUERY（不重要）`：一个子查询的结果不能被缓存，必须重新评估外层查询的每一行。
- `UNCACHEABLE UNION（不重要）`：关联查询第二个或者后面的语句属于不可缓存的子查询。

### 字段：table

当前行使用的表。

### 字段：type（重要）

连接类型。值为`null`表示不需要访问表或索引，如`select 1`。

一般来说，要保证查询至少达到`range`级别，最好能达到`eq_ref`。

从上到下，性能从优到差：

1. `system`：const的特例：查询对象表只有一条数据，且只能用于MyISAM和Memory引擎的表。
2. `const`：基于主键或唯一索引查询，最多返回一条结果。
3. `eq_ref`：表连接时基于主键或非NULL的唯一索引完成扫描，每个索引键只会匹配一条记录。
4. `ref`：基于普通索引的等值查询，或者表间等值链接，一个索引键可能匹配多条记录。
5. `fulltext`：全文检索。
6. `ref_or_null`：表连接类型是ref，但是进行扫描的索引列中可能包含NULL值。
7. `index_merge`：利用多个索引。
8. `unique_subquery`：子查询中使用唯一索引。
9. `index_subquery`：子查询中使用普通索引。
10. `range`：利用索引进行范围查询。
11. `index`：全索引扫描。
12. `ALL`：全表扫描。

### 字段：possible_keys

可能用到的索引列。

### 字段：key（重要）

实际用到的索引的名称。

### 字段：key_len

实际用到的索引的长度。

### 字段：ref

查询中引用的字段或常数。

### 字段：rows（重要）

根据表统计信息及索引选用情况，大致估算出找到匹配的记录所需要读取的行数。值越小越好。

### 字段：filtered

`通过查询条件获取的最终记录行数 / rows * 100%`，值越接近`100`越好。

### 字段：Extra（重要）

其他详细信息。

- `Using filesort`：将用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘完成排序。
- `Using temporary`：需要创建临时表来保存中间结果，通常发生于对没有索引的列进行GROUP BY时。
- `Using index`：使用覆盖索引。如果同时出现Using where，表明索引被用来执行索引键值的查找。
- `Using where`：使用where语句来处理结果。
- `Impossible WHERE`：where子句判断的结果总是false。
- `Using join buffer (Block Nested Loop)`：关联查询中，被驱动表的关联字段没索引。
- `Using join buffer (hash join)`：关联查询中，被驱动表的关联字段没索引。MySQL8开始支持hash join，性能优于Block Nested Loop。
- `Using index condition`：先条件过滤索引，再回表查数据。
- `Select tables optimized away`：使用某些聚合函数（如max、min）来访问存在索引的某个字段。

## EXPLAIN 查看执行计划树

```sql
-- 以树格式查看查询语句的执行计划
EXPLAIN FORMAT=TREE <查询语句>;

-- 以树格式查看查询语句的实际执行计划
EXPLAIN ANALYZE <查询语句>;
```

执行计划树中，节点执行顺序为后续遍历。
  