---
layout: post
title: Mysql篇-InnoDB(二)
description: InnoDB基础组件介绍
category: 技术
---

## 一、Mysql 锁概念

锁是数据库系统区别于文件系统的一个关键特性。在 Innodb 存储引擎中，主要是为了保证对不同资源的并发访问的一致性。在不同的数据库中，锁的实现方式也各不相同，而 InnoDB 的锁实现和 Oracle 数据库很相似。提供一致性的非锁定性、行级锁定性。在数据库中 lock(事务级别)和 latch(系统内部程序级)都称之为锁，数据库使用者一般更关注 lock 锁。lock 锁定对象是事务级别，用来锁定数据库中的对象（表、页、行），且一般锁仅在事务 commit 和 rollback 后进行释放。以下以行级锁为例。**特别注意，在 MySQL 中,行级锁并不是直接锁记录,而是锁索引。**也就是说如果锁定的数据列不存在索引，则会升级成表锁定(比如悲观表)。

![](https://trace-night.oss-cn-chengdu.aliyuncs.com/prod/20220604/deb84635ad504919a405f76650d961ba.png)

**(1).行级锁的类型**

- 共享锁(S Lock): 允许事务读取一行数据
- 排他锁(X Lock)：允许事务删除或更新一行数据（均为行锁）

意向共享锁(IS Lock): 事务想要获得表中的某几行的共享锁
意向排他锁(IX Lock): 事务想要获得表中的某几行的排他锁
![](https://trace-night.oss-cn-chengdu.aliyuncs.com/prod/20220604/2d37e8bc71e34b099a43406b92b15169.png)

为什么需要意向锁？
意向锁是表级别的锁，用来标识该表上有数据被锁住或即将被锁，对于表级别的请求(LOCK TABLE…)，就可以直接判断是否有锁冲突，不需要逐行检查锁的状态

**(2). 一致性非锁定读**

一致性非锁定读是指 InnoDB 存储引擎通过行多版本控制的方式来读取当前执行时间数据库中**行（注意是行锁）**的数据。如果正在读取的行上存在 X 锁时，该读取操作并不会等待行的 X 锁释放，而是会读取该行的一个快照数据，从而提升数据库的并发性能。

**MVCC（Multi-Version Concurrency Control）**：多版本并发控制，是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问，**用于支持 RC(读已提交)和 RR(可重复读)隔离级别的实现**。针对前者，MVVC 提供最新数据的快照版本，而对于后者，则是提供事务开始时的数据版本，因为前者不支持不可重复读，而后者支持不可重复读。

MVCC 是通过在每行记录后面保存两个隐藏的列来实现的。这两个列，一个保存了行的创建时间，一个保存行的删除时间。当然存储的并不是实际的时间值，而是系统版本号。每开始一个新的事务，系统版本号都会自动递增。事务开始时刻的系统版本号会作为事务的版本号，用来和查询到的每行记录的版本号进行比较。

**(3).一致性锁定读**
在某些场景下，用户需要显示的对数据库读取操作进行加锁以保证数据逻辑性的一致性。
常用对 select 操作加锁语法： select **_ for update; (添加 X 锁)
select _** lock in share mode）(添加 S 锁)

**(4).锁的算法**
InnoDB 有 3 种行锁算法：

- Record Lock: 单个行记录上的锁
- Gap Lock: 间隙锁，锁定一个范围，但不包含行本身，主要是为了阻止多个事务将记录插入到同一个范围内，从而导致幻读
- Next-Key Lock: Gap Lock + Record Lock，锁定一个范围，并且锁定记录本身。

## 二、锁引发的问题

![](https://trace-night.oss-cn-chengdu.aliyuncs.com/prod/20220606/369b41e09173409392180e9e8a8b5da1.png)

**幻读（Phantom Problem）**： 在同一事务下，连续执行两次同样的 sql 可能导致不同的结果即幻读，通俗讲就是：事务 A 按照一定条件进行数据读取，期间事务 B 插入了相同搜索条件的新数据，事务 A 再次按照原先条件进行读取操作修改时，发现了事务 B 新插入的数据称之为幻读。因此可以解释上述行锁算法为什么对范围加锁。假设在 A 事务中存在多次调用 sql:select \* from table where id > 4， 但在两次 sql 调用之间，事务 B 插入了一条 id 为 10 的数据，那么 A 事务中，两次查询就会得到不相同的结果。

InnoDb 引擎默认的事务隔离级别是 Repeatable read,在该隔离级别中，采用了 Next-Key Lock 算法，而在 read committed 下，仅采用 record lock，因此 read committed 存在幻读情况。

**脏读（Dirty Read）**： 在不同的事务下，当前事务可以读到另外事务未提交的数据，在 read uncommitted 事务隔离级别下，会触发脏读

**不可重复读（Dirty Read）**：在同一个事务范围内，多次查询某个数据，却得到不同的结果。在第一个事务中的两次读取数据之间，由于第二个事务的修改，第一个事务两次读到的数据可能就是不一样的。大多数据库支持不可重复读，而 Mysql 通过将其定义为幻读，处理方式则是通过上述的 Next-Key Lock 算法。

**丢失更新（lost update）**：因为 mysql 采用**一致性非锁定读**来提升数据库的并发性，但同时也衍生出一系列问题，比如说丢失更新，假设两个事务同时查询一行数据，然后又同时对这条数据进行修改，就会导致先修改数据丢失，即为丢失更新。常见的解决方案：悲观锁、乐观锁。

**阻塞**：因为锁兼容关系，通常在面对同一资源时，需要等待其他锁先释放资源。innodb_lock_wait_timeout=50s(默认 50s,最大锁定时间，动态参数)，innodb_rollback_on_timeout=OFF:用于设定等待超时对进行中的事务进行回滚操作，默认关闭

**死锁**：是指两个或两个以上的进程在执行过程中,因争夺资源而造成的一种互相等待的现象,若无外力作用，它们都将无法推进下去.此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。下面是常用的死锁解决方案：

1. 尽量让数据表中的数据检索都通过索引来完成，避免无效索引导致行锁升级为表锁。
2. 合理设计索引，尽量缩小锁的范围。
3. 尽量减少查询条件的范围，尽量避免间隙锁或缩小间隙锁的范围。
4. 尽量控制事务的大小，减少一次事务锁定的资源数量，缩短锁定资源的时间。
5. 如果一条 SQL 语句涉及事务加锁操作，则尽量将其放在整个事务的最后执行。
6. 尽可能使用低级别的事务隔离机制。(隔离级别越低，事务请求的锁越少或保持锁的时间就越短。当然在 InnoDB 中，repeatable read 和 read committed 的用户性能差距并不会很明显，如果使用分布式事务，InnoDB 存储引擎就必须用 serializable 隔离级别)

## 三、Mysql 事务

在数据库层面，数据既是把数据库从一种一致文件转换为另一种一致状态，在数据库提交工作时，可以确保要么所有修改都已经修改，要么所有修改都不保存。
InnoDB 存储引擎事务完全满足 ACID 的特性
A：原子性(atomicity): 在同一事务中，所有 sql 操作要么一起成功，要么一起失败。
C: 一致性(consistency)：事务中某个动作失败后，系统可以自动撤销事务，返回初始化状态。
I: 隔离性(isolation): 即各个事务之间相互独立，互不影响。
D: 持久性(durability): 事务一旦提交，结果便是永久性的(永久存储在磁盘上，不包括物理损坏, 如磁盘损坏)。

事务的原子性和持久性是通过 redo log 来保证的，而一致性则是由 undo log 来保证的。

Redo Log 记录了此次事务**「完成后」** 的数据状态，记录的是更新之 **「后」**的值
Undo Log 记录了此次事务**「开始前」** 的数据状态，记录的是更新之 **「前」**的值
下列是一个 sql 执行期间，如何写入 redo、undo log：
![](https://trace-night.oss-cn-chengdu.aliyuncs.com/prod/20220606/e695ad4894ca451f89ceaef5797ca6f6.png)