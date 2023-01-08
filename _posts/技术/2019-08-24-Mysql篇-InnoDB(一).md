---
layout: post
title: Mysql篇-InnoDB(一)
description: InnoDB基础组件介绍
category: 技术
---

## 一、Mysql 常用存储引擎

- InnoDB： 支持事务，主要特点是行锁设计、支持外键，最常用的存储引擎(1.2 以上版本支持全文索引)
- MyISAM： 不支持事务、表锁设计，支持全文索引，主要面向数据分析的应用系统
- NDB: 集群引擎，因为连接操作(join)在数据库层完成而非存储引擎，因此查询速度较慢
- Memory:表数据存储在内存，一旦崩溃，表中数据都会消失，因此适合用于存储临时数据，同样只支持表锁，并发性能差
- Archive: 只支持 insert 和 select 操作，且使用 zlib 算法压缩行数据并存储，因此适合存储归档数据(eg: 日志)，支持行锁
- Maria: 最新的存储引擎，用于替代 MyISAM 存储引擎(和 MariaDB 是两回事)
- 还有其他存储引擎，诸如：Federated、Merge、CSV、Sphinx、Infobright 等引擎，

本文主要说明 InnoDB 的底层结构和引擎流程，下图是 Mysql 执行一个 sql 的完整流程

![](https://trace-night.oss-cn-chengdu.aliyuncs.com/prod/20220604/53dd0b35ba144f109bd7b2aa02710b51.png)

## 二、Innodb 存储引擎

InnoDB 的版本历史
老版本 InnoDB ====> 支持 ACID,行锁设计,MVCC
InnoDB 1.0.x ====> Mysql 5.1，继承以上所有功能,增加了 compress 和 dynamic 页格式
InnoDB 1.1.x ====> Mysql 5.5，继承 1.0.x 的所有功能,增加了 Linux AIO ,多回滚段
InnoDB 1.2.X ====> Mysql 5.6，继承了 1.1.x 的所有功能,增加了全文索引的支持,在线索引添加,在线对大表加字段
最新版本的 InnoDB 主要调整在参数上，没有底层架构上的调整，最新的 Mysql 版本是 8.x, mysql6、mysql7 则是因为种种原因就这样没了。

InnoDB 存储引擎主要分为两部分：一是后台线程，二是内存。如下图所示

<div align="center">

![](https://trace-night.oss-cn-chengdu.aliyuncs.com/prod/20220517/b1c8a6c335954ba88e2ba9c6625c9038.png)

</div>

**_(1).后台线程_**
InnoDB 存储引擎是多线程模型，因此其后台有多个不同的后台线程，负责处理不同的任务

1. Master Thread: 主要负责将缓冲池中的数据异步刷新到磁盘，包括合并插入缓冲等功能
2. IO Thread: 在存储引擎中大量使用 AIO(Async IO)来处理写 IO 请求，分别有（write、read、insert buffer、log IO thread）四类线程，可以使用（innodb_read_io_threads 和 innodb_write_io_threads）来调整读、写线程，读、写线程默认为 4，其他默认为 1
3. Purge Thread: 事务提交后，其所使用的 undolog 可能不再需要，因此需要 PurgeThread 来回收已经分配的 undo 页，同样，可以通过参数(innodb_purge_threads[默认为 1])来配置回收线程数
4. Page Cleaner Thread: 将之前 Master Thread 做的脏页刷新操作都放到单独的线程中来完成。

**_(2).内存_**
InnoDB 存储引擎是基于磁盘存储的，由于 CPU 速度和磁盘速度之间的差距过大，因此内存是 InnoDB 中是必不可少的，可以使用 ‘**show engine innodb status**’来查看引擎信息。 InnoDB 存储引擎的内存结构主要分为以下几个部分

![](https://trace-night.oss-cn-chengdu.aliyuncs.com/prod/20220517/92ba26c23545483b954568b8697e9db7.png)

**1.缓冲池**
缓冲池就是一块内存区域，即是把磁盘上的数据缓存到缓冲池，通过内存速度来弥补磁盘速度较慢对数据库的影响。（可以通过 innodb_buffer_pool_size[单位是 byte]）来配置缓冲池的大小。缓冲池主要分为：索引页、数据页、undo 页、插入缓冲(insert buffer)、自适应 hash 索引、锁信息、数据字典信息等。最新版本，允许配置多个缓冲池实例,增加数据库的并发处理(innodb_buffer_pool_instances[缓冲池实例数，默认为 1])

**2.LRU List、Free List、Flush List**
缓冲池中存放了各类页，如何来管理这些页？ 这就是这 3 个列表的作用

- **LRU List：** 使用最少使用算法来管理页(可用的页)，LRU 算法是指最新的页放在列表首部，而 InnoDB 中使用的则是将最新使用的页放在列表长度的 5/8 处(innodb_old_blocks_pct：默认 37，也就是距离列表尾部 37%的位置，反之则是列表 5/8 的地方)，这种调整是为了避免偶然出现的页挤掉热点数据的页(常见操作：索引或数据的扫描操作),同时可以通过 innodb_old_blocks_time 变量来说明，页在被读取到指定位置后多久才会加入到 LRU 列表的热端。如果值是 1000，则代表 1S，如果数据在 1S 中被读取了两次则把当前页放到 LRU 最前面。如果这个值是 0，第一次被插入到 midpoint 位置，如果第二次查询，数据页还在 LRU List 中，则就会把该数据页放到 LRU List 的最前面。同理 innodb_old_blocks_time=1000.则是 1S 中被查询，才被放到 LRU List 的最前面，因为当数据库刚启动时，LRU 列表是空的，因此可使用 innodb_buffer_pool_dump_pct[预热的数据百分百]参数来对数据进行预热[该操作局限性很大]
- **Free List：** 数据库刚启动所有页都存在 Free List, LRU 需要页的时候将改页从 Free 列表删除并加入到 LRU 列表，如果 Free List 为空，则淘汰 LRU 末尾的页
- **Flush List：** LRU 列表中的页被修改后，需要刷新到磁盘，但这个过程是异步的，因此需要该列表记录脏页然后刷新到磁盘，同时 LRU 列表中仍然存在脏页(管理脏页的可用性)

**3.重做日志缓冲**
和 Flush List 类似，InnoDB 在刷新重做日志之前，会先把重做日志写到重做日志缓冲（一般在事务开始时，就开始写重做日志），然后以一定的频率刷新到重做日志文件(一般是每秒刷新一次,此外还有两种操作亦会触发刷新操作：1. 当有事务提交时 2.重做日志缓冲剩余空间不足 1/2)，因此只要控制每秒产生的事务量在缓冲大小之内即可(innodb_log_buffer_size=8M)

**4.额外的内存池**
缓冲池中一些数据结构本身的内存，需要额外从内存池中进行申请(eg: LRU 本身、锁定、等待信息等)

备注：脏页是以异步的方式刷新到磁盘的，那么如何维护这个异步操作？一次性应当刷新多少脏页？从哪里取脏页？什么时间触发脏页？假设数据库宕机，应当如何恢复这些脏页里的数据呢？脏页数据过多，又该如何缩短数据库的恢复时间呢？

这里引入**Checkpoint**技术，该技术主要解决以上问题

Checkpoint 分为 Sharp Checkpoint(默认，数据库关闭时，将所有脏页刷新到磁盘(innodb_fast_shutwon=1)),但在数据库运行期间，数据库的可用性会受到很大影响。故在 InnoDB 中，使用 Fuzzy Checkpoint 模式(只刷新部分脏页)

**Fuzzy Checkpoint**有 4 种触发模式

- Master Thread Checkpoint（主线程以每 1 or 10 秒的速度刷新部分脏页到磁盘，不论事务是否提交，最新版 innodb, 是 Page Cleaner Thread 专门刷新脏页）
- FLUSH_LRU_LIST Checkpoint(当空闲列表少于 100 个空闲页，会移除 LRU 列表尾端的页(如果包含脏页，则刷新到磁盘))
- Async/Sync Flush Checkpoint（后续再讨论）
- Dirty Page too much Checkpoint(脏页太多，innodb_max_dirty_pages_pct=75,当脏页列表超过 75%，则每秒刷新 200 个脏页到磁盘（innodb_io_capacity=200）)

**InnoDB 关键特性**

1. 插入缓冲(Insert Buffer): 当插入数据时，满足**索引是辅助索引(非聚集索引)且索引不是唯一的**[聚集索引一定是唯一的，那 B+tree 一定不存在该索引，然后会去 disk 查询是否有该值，因此聚集索引没有缓存的必要]，会合并插入缓冲，而后衍生 Delete Buffer、 Purge buffer 效果和 Insert Buffer 一致，可通过 innodb_change_buffer_max_size 来控制 change_buffer 的最大内存数量，默认 25，也就是最多使用 25%的缓冲池内存空间。可设置 innodb_change_buffering 来控制可合并缓冲插入的类型(eg:all（insert、delete、update）、none（不缓存任何操纵）、inserts、deletes、purges)

2. 两次写：保障数据页的可靠性，比如说，当数据页写到一半，数据库宕机了，重做日志也是毫无意义的。因此才需要两次写，该操作发生于脏页同步到磁盘，脏页 -> double write -> 写入磁盘

3. 自适应 hash 索引： InnoDB 引擎会根据数据访问频率，自动会某些热点数据创建哈希索引(1.必须单条件查询 2.以该模式至少访问 100 次数 3.数据页须以该模式访问 N 次(N=页中记录/16)),innodb_adaptive_hash_index（默认开启自适应 hash 索引）

4. 异步 IO: 即是合并插入缓冲（innodb_use_native_aio=NO,默认开始异步 IO）

5. 刷新临近页：在刷新脏页的时候是否刷新同区的页（innodb_flush_neighbors=true）,默认开启，机械硬盘建议开启，固态硬盘则建议关闭

关于使用 innodb 关闭的策略有三种策略，同样在启动时也可以指定恢复哪些数据
innodb_fast_shutdown=1(0, 1, 2)

- 0: mysql 关闭时，需完成所有 full purge 和 merge insert buffer
- 1: mysql 关闭时，不需完成所有 full purge 和 merge insert buffer，但仍然会刷新部分脏页到磁盘
- 2：什么都不做，全靠日志文件恢复(mysql 宕机后，会生成错误日志文件，默认后缀为 err,通过错误日志文件，可以得到需要恢复的日志行？)

**_(3).日志文件_**

常见的日志文件有：

- 错误日志：mysql 数据库系统级别错误
- 慢查询日志: long_query_time=10（默认 10s,记录执行时间超过 10 的日志），slow_query_log=OFF,默认关闭慢查询日志记录， log_queries_not_using_indexes=OFF,默认关闭不使用索引日志记录 log_throttle_queries_not_using_indexes=0（默认每分钟记录未使用索引 sql 次数，默认 0，即无限制）
- 查询日志： 使用方式和慢查询日志差不多
- **二进制日志**: 记录执行 insert、update(即便是没有发生数据变化)等修改或删除数据的 sql 时执行

1. 生产环境中，一般必须配置二进制日志。通过配置参数**log-bin=fileName**启动二进制文件
   常用影响二进制文件的参数

- max_binlog_size: 单个二进制文件的最大值，超过该值，重新生成日志文件(文件名称：后缀名+1)，默认 1G
- binlog_cache_size: 所有未提交事务的二进制日志记录到该缓存，事务提交后再写入，默认 32K，同上，超过该值，会生成临时文件，该值的大小可参照临时文件的使用次数
- sync_binlog: 默认为 1，也就是同步刷新到磁盘(包括未提交事务的日志)，也就是会存在一个问题，当执行 sql 后，事务没提交之前，mysql 宕机，这部分日志已经不能回滚。
- binlog-do-db、binlog-ignore-db: 表示需要写入或者忽略哪些日志文件，默认为空，即同步所有到二进制日志
- log_slave_update: 默认开启，即从实例的 mysql 会从 master 读取日志文件并写入到自己的二进制文件
- **binlog_format**: 二进制日志格式：默认 ROW,动态参数
  STATEMENT: 记录日志的 sql 逻辑
  ROW: 记录表的更新情况，在恢复数据上优于 STATEMENT，但在日志存储上则远大于 STATEMENT（现代软件不缺磁盘，要保证准确率，因此默认为 ROW ）
  MIXED: 混合模式，在使用特性函数，会使用 ROW 记录，其余用 STATEMENT 记录

上述提到许多 mysql 的配置参数，mysql 的配置参数主要分为两类，静态参数和[动态参数](https://dev.mysql.com/doc/refman/8.0/en/dynamic-system-variables.html)，修改前者需重启数据库实例，后者则反之.

### 三、表

innodb 存储引擎的逻辑存储结构来看，所有数据均在一个空间内，称之为表空间。表空间由段（数据段、索引段、回滚段等）、区（默认每个区 1M，每页默认 16KB，也就是说一个区有 64 个页）、页（每页是由数据行组成，而每页最多存放 16kb/2 - 200 = 7992 行）组成。如果用户启用(innodb_file_per_table)参数，则每张表的数据都可以单独放到一个表空间内（数据、索引、插入 bitmap 页）。其余数据仍然放在共享表空间内。

innodb 中常见的页类型： 数据页、undo 页、系统页、事务数据页、插入缓冲位图页、插入缓冲空闲列表页、未压缩的二进制大对象页、压缩的二进制大对象页

InnoDB 行记录格式： 主要有四种： (默认 Dynamic)

- Compact： 不管是 CHAR 还是 VARCHAR 类型，NULL 值都不占用任何存储空间
- Redundant：对于 VARCHAR 类型，NULL 值都不占用任何存储空间，而 CHAR 类型则可能暂用更多的空间（n_fileds 指针只有 10 位，故 mysql 最多只能有 1023 列）
- Compressed: 区别于以上格式，行溢出只会在数据页存放 20 个字节的指针而非 Compact 和 Redundant 的 768 个字节。同时会以 zlib 算法进行压缩存储，对大字段（BLOB、TEXT、VARCHAR）很友好
- Dynamic：同上，行溢出只会在数据页存放 20 个字节的指针而非 Compact 和 Redundant 的 768 个字节

分区表：Mysql 数据库支持 RANGE、LIST、HASH、KEY 分区，当表中存在主键或唯一索引时，分区列必须是唯一索引的一个组成部分（Mysql 允许对 NULL 进行分区）。

```sql
-- select * from information_schema.`PARTITIONS`  可在此表查询分区信息
-- alter table t drop partition t1; --删除指定分区

-- 1. RANGE分区： id < 10为t0区间，10 <= id < 20为t1分区，id>=20为t2分区（最为常用的分区）
  create table t (
    id int
  )ENGINE=INNODB
  partition by range(id) (
    partition t0 values less than (10),
    partition t1 values less than (20),
    partition t2 values less than MAXVALUE
  );
  -- id只能为数字，假设有一个时间字段，如果按照年来分组 可使用 range(YEAR(date))

-- 2. LIST分区，和RANGE类似，只是分区值是离散，而非连续的，常用于某些离散程序小的列
   create table t1 (
    id int
   )ENGINE=INNODB
    partition by list(id) (
    partition t10 values in (1, 3, 5, 7),
    partition t11 values in (2, 4, 6, 8)
  );

-- 3. HASH分区
   create table t_hash (
    id int,
    b datetime
   )ENGINE=INNODB
   partition by hash (id) -- id为整数表达式，以取模的方式，载入分区
   partitions 4;  --分区数量, 默认1

-- 4. key分区，和hash分区类似，只是分区函数 hash是定义一个整数表达式，而key分区是基于数据库提供的函数来计算
  create table t_hash (
    id int,
    b datetime
  )ENGINE=INNODB
  partition by key (b) -- b也可为其他字段
  partitions 4;  --分区数量, 默认1

-- 5.columns分区， range、list分区的升级版，支持多列聚合分区
  create table t_column (
    id int,
    name varchar(20),
    time datetime
  )ENGINE=INNODB
  -- range分区
  partition by range columns(id) (
    partition p0 values less than (10)
  );
  -- list分区
  partition by list columns(id) (
    partition p1 values in (1, 3, 5, 7)
  );
  -- 聚合分区
  partition by range columns(id, name) (
    partition p2 values less than (10, 'ggg')
  );
-- 子分区，在分区的基础上再进行分区，允许range、list分区上再次进行hash/key分区
   create table ts (
      id int,
      b date
   )ENGINE=INNODB
-- 基于id分区
   partition by range(id)
-- 在id分区的基础上再次对时间分区，每个分区的子分区数量必须一致，且子分区只支持hash和key分区
   subpartition by hash(TO_DAYS(B)) (
      partition t0 values less than (10)(
	    subpartition ts_0,
	    subpartition ts_1
      ),
      partition t1 values less than (20)(
	    subpartition ts_2,
	    subpartition ts_3
      ),
      partition t2 values less than MAXVALUE(
	    subpartition ts_4,
	    subpartition ts_5
      )
  );
-- 分区之间的数据交换，即分区数据导入、导出
alter table e exchange partition t0 with table e2; -- 将e表中t0分区的数据移动到e2表

```

**总结：** 分区对于某些场景的查询其实并没有显著的性能提升，尤其是对于事务类场景(OLTP)，不合理的分区甚至会导致性能急速下降，因为在 OLTP 场景中，一般只需要查询很少的数据，而相对于分析处理类场景(OLAP),分区确可以很好的提升查询性能。