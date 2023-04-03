---
title: MySQL 扫盲
date: 2023-03-31 22:59:20
cover: /img/cover/img1003.jpg
categories: 
    - IT
    - IT.数据库
toc: true
tags: 
    - mysql
excerpt: MySQL是一个GPL开源的关系数据库管理系统(DBMS)，根据DB-Engines，截止2023年它的流行程度已仅次于Oracle。其实有很多其他比MySQL好用的DBMS，但为什么它能如此受欢迎呢，个人认为主要有三点：开源免费、简单易用、知名度高。
---

# 什么是MySQL

***MySQL***是一个*GPL开源*的关系数据库管理系统(**DBMS**)，根据[DB-Engines](https://db-engines.com/en/ranking)，截止2023年它的流行程度已仅次于***Oracle***。其实有很多其他比MySQL好用的DBMS，但为什么它能如此受欢迎呢，个人认为主要有三点：**开源免费、简单易用、知名度高**。

- **[如何选择开源许可证？](https://zion4h.github.io/2023/03/31/osi-intro/)**

# 索引

首先我们应当明确一点，索引是用来[优化SQL查询](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)的，或者说提高`SELECT`操作速度的，当然前提是要在查询条件中使用索引列。

## 创建索引

由[CREATE INDEX Statement](https://dev.mysql.com/doc/refman/8.0/en/create-index.html)，

```bash
CREATE [UNIQUE | FULLTEXT | SPATIAL] INDEX index_name
    [index_type]
    ON tbl_name (key_part,...)
    [index_option]
    [algorithm_option | lock_option] ...

key_part: {col_name [(length)] | (expr)} [ASC | DESC]

index_option: {
    KEY_BLOCK_SIZE [=] value
  | index_type
  | WITH PARSER parser_name
  | COMMENT 'string'
  | {VISIBLE | INVISIBLE}
  | ENGINE_ATTRIBUTE [=] 'string'
  | SECONDARY_ENGINE_ATTRIBUTE [=] 'string'
}

index_type:
    USING {BTREE | HASH}

algorithm_option:
    ALGORITHM [=] {DEFAULT | INPLACE | COPY}

lock_option:
    LOCK [=] {DEFAULT | NONE | SHARED | EXCLUSIVE}
```

## 前缀索引

有时候索引列很长比如**CHAR(200)**，但在前10个或者前20个字符，大多数值又是唯一的，我们可以对索引列的前N个字符创建索引，从而节省索引空间并加速查询。

## 全文本索引

***MyISAM***和***InnoDB***存储引擎都支持全文本索引`FULLTEXT`的，并且只限于**CHAR**、**VARCHAR**和**TEXT**列，但是索引是对整个列的，不支持前缀索引。

## 索引设计原则

这里我直接引用[《深入浅出MySQL》](https://www.linuxidc.com/Linux/2016-05/130922.htm)，

> 最适合索引的列是出现在 WHERE子句中的列，或连接子句中指定的列，而不是出现在 SELECT 关键字后的选择列表中的列。
> 
> 
> …
> 
> 如果有一个 CHAR(200)列，如果在前 10 个或 20 个字符内，多数值是惟一
> 的，那么就不要对整个列进行索引。对前10个或20个字符进行索引能够节省大量索引空间，
> 也可能会使查询更快。
> 
> …
> 
> 利用最左前缀。
> 
> …
> 
> 如果有明确定义的主
> 键，则按照主键顺序保存。如果没有主键，但是有唯一索引，那么就是按照唯一索引的顺序
> 保存。如果既没有主键又没有唯一索引，那么表中会自动生成一个内部列，按照这个列的顺
> 序保存。
> 

## 索引结构

MySQL有两种索引结构，**B-TREE**和**HASH**，***MyISAM***和***InnoDB***都只支持B-TREE，而***MEMORY/HEAP***支持HASH和B-TREE。HASH结构的数据库引擎有几个最显著的限制，只能在`WHERE`中使用`=`或`<=>`符，无法使用`ORDER BY`，本质是为了换取更小内存做出的牺牲。

- 博客：[B树家族](https://zion4h.github.io/2023/04/01/data-structure-storage/#more)

## 组合索引的最左前缀特性

如果我们创建了一个组合索引，我们也不强求在条件句中用上全部的索引列，而是只要用到了最左边的索引列就能用上这个组合索引。

```bash
mysql> create index ind_sales2_companyid_moneys on sales2(company_id,moneys);
Query OK, 1000 rows affected (0.03 sec)
Records: 1000 Duplicates: 0 Warnings: 0

mysql> explain select * from sales2 where company_id = 2006\G;
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: sales2
 type: ref
possible_keys: ind_sales2_companyid_moneys
key: ind_sales2_companyid_moneys
 key_len: 5
 ref: const
 rows: 1
 Extra: Using where
1 row in set (0.00 sec)
```

## 索引的like查询

如果查询条件是`like`，那么`“%”`不能放在开头，否则无法使用索引；对于大文本查询，我们要用全文搜索，而不是`like “%…%”`。

## 无法使用索引的场景

1. MySQL估计索引查询比全表扫描还要慢时
2. 使用MEMORY/HEAP，并且`where`条件中不带`“=”`时，因为HASH索引自身的结构原因不支持
3. 采用复合索引时，`WHERE`条件没有索引列的第一部分
4. 用`or`分割开的`WHERE`条件句，如果有一个列没有索引，则都不会用到
5. 如果`like`是用`“%”`开头的则不支持

# 事务和锁定语句

[MySQL事务](https://dev.mysql.com/doc/refman/8.0/en/sql-transactional-statements.html)一般指的**本地事务**，但它也能通过XA事务支持**分布式事务**。

## 事务控制命令

```sql
START TRANSACTION
    [transaction_characteristic [, transaction_characteristic] ...]

transaction_characteristic: {
    WITH CONSISTENT SNAPSHOT
  | READ WRITE
  | READ ONLY
}

BEGIN [WORK]
COMMIT [WORK] [AND [NO] CHAIN] [[NO] RELEASE]
ROLLBACK [WORK] [AND [NO] CHAIN] [[NO] RELEASE]
SET autocommit = {0 | 1}
```

- **START TRANSACTION** 或 **BEGIN** 开启一个新的事务，后者就是个别名，不用纠结
- **COMMIT** 提交当前事务
- **ROLLBACK** 回滚当前事务，回滚即取消回滚点后它发生的变化
- **SET autocommit** 禁止或允许（默认允许）自动提交事务
- **READ WRITE** 或 **READ ONLY** 事务访问模式，默认是读写模式

当我们开始一个事务时，如果还有未提交的事务存在那么它会被立刻提交。

如果有`LOCK TABLES`，那么也会强制释放表锁。另外一点是，任何**DDL语句**都无法回滚。

## 事务断点

```sql
SAVEPOINT identifier
ROLLBACK [WORK] TO [SAVEPOINT] identifier
RELEASE SAVEPOINT identifier
```

事务断点的使用方法如上，很直观。如果创建同名断点时会覆盖旧的断点。

在执行回滚操作后，晚于回滚位置之后的保存点都将删除，并且回滚位置之后存储的行锁都不会释放，除非是新增的行。

## 锁定语句

```sql
LOCK TABLES
    tbl_name [[AS] alias] lock_type
    [, tbl_name [[AS] alias] lock_type] ...

lock_type: {
    READ [LOCAL]
  | [LOW_PRIORITY] WRITE
}

UNLOCK TABLES
```

1. 使用LOCK TABLES的只能访问被锁定表格
    
    ```bash
    mysql> LOCK TABLES t1 READ;
    mysql> SELECT COUNT(*) FROM t1;
    +----------+
    | COUNT(*) |
    +----------+
    |        3 |
    +----------+
    mysql> SELECT COUNT(*) FROM t2;
    ERROR 1100 (HY000): Table 't2' was not locked with LOCK TABLES
    ```
    
2. 锁定时使用表名或者别名都可以，但是我们在访问这些锁定表一定要和锁定语句的声明保持一致
    
    ```bash
    mysql> LOCK TABLE t WRITE, t AS t1 READ;
    mysql> INSERT INTO t SELECT * FROM t;
    ERROR 1100: Table 't' was not locked with LOCK TABLES
    mysql> INSERT INTO t SELECT * FROM t AS t1;
    ```
    
3. [LOCK TABLES](https://dev.mysql.com/doc/refman/8.0/en/lock-tables.html) 和 [UNLOCK TABLES](https://dev.mysql.com/doc/refman/8.0/en/lock-tables.html) 配合事务使用：
    
    ```sql
    SET autocommit=0;
    LOCK TABLES t1 WRITE, t2 READ, ...;
    ... do something with tables t1 and t2 here ...
    COMMIT;
    UNLOCK TABLES;
    ```
    

## 设置事务

```sql
SET [GLOBAL | SESSION] TRANSACTION
    transaction_characteristic [, transaction_characteristic] ...

transaction_characteristic: {
    ISOLATION LEVEL level
  | access_mode
}

level: {
     REPEATABLE READ
   | READ COMMITTED
   | READ UNCOMMITTED
   | SERIALIZABLE
}

access_mode: {
     READ WRITE
   | READ ONLY
}
```

1. 设置事务隔离级别
    InnoDB提供了4种隔离级别[READ UNCOMMITTED](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-uncommitted), [READ COMMITTED](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-committed), [REPEATABLE READ](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) and [SERIALIZABLE](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_serializable)，默认是**REPEATABLE READ**。
    
2. 设置事务访问模式
    默认是**READ WRITE**，如果设置成**READ ONLY**，就无法在事务中修改**table**了。
    
3. 设置事务特征范围
    就是这次设置事务会影响到全局（尽管叫全局但显然不可能包括已经提交的事务）或者当前会话。
    

## 隔离级别

- **READ UNCOMMITTED**
    最低隔离级别，在读未提交这个级别下，MySQL会以无锁模式读取行，*导致有可能读到未提交事务更新后的数据*，这种问题也被叫做**脏读**。
    
- **READ COMMITTED**
    读已提交首先解决了脏读问题，但是T1在两次读取期间，可能T2对数据修改，导致前后T1前后读取结果不一致，这种问题叫做**不可重复读**。
    
- **REPEATABLE READ**
    InnoDB默认隔离级别，T1在读取时，其他事务不能操作其访问行，但是其他事务可以插入新行，因此T1在读取期间表的行数一直在增加，这就是**幻读问题**。InnoDB引入了next-key lock解决了该问题。
    
- **SERIALIZABLE**
    最严格的隔离级别。
    

## ACID模型

***ACID***是一套数据库设计原则，主要强调的是可靠性方面。

> 《*深入浅出MySQL*》
> 
> 
> > 原子性（Atomicity）：事务是一个原子操作单元，其对数据的修改，要么全都执行，
> 要么全都不执行。
> > 
> 
> > 一致性（Consistent）：在事务开始和完成时，数据都必须保持一致状态。这意味着
> 所有相关的数据规则都必须应用于事务的修改，以保持数据的完整性；事务结束时，
> 所有的内部数据结构（如B 树索引或双向链表）也都必须是正确的。
> > 
> 
> > 隔离性（Isolation）：数据库系统提供一定的隔离机制，保证事务在不受外部并发操
> 作影响的“独立”环境执行。这意味着事务处理过程中的中间状态对外部是不可见
> 的，反之亦然。
> > 
> 
> > 持久性（Durable）：事务完成之后，它对于数据的修改是永久性的，即使出现系统
> 故障也能够保持。
> > 

# 锁

## 共享锁和独占锁

InnoDB通过**共享锁**和**独占锁**实现了行级锁，共享锁允许事务读该行数据，而独占锁允许对该行修改和删除。共享锁和独占锁我们一般用**S锁**和**X锁**简称，如果事务T1持有一行数据的**S锁**，那么当另一个事务请求**S锁**时能够立即获取，但请求**X锁**则需要等待**T1**先释放**S锁**。

## 意向锁

InnoDB通过意向锁来实现表级锁，**意向共享锁（IS）**会在整张表的每个行设共享锁，而**意向独占锁（IX）**则是设独占锁。

比如，`“SELECT…FOR SHARE”`就是**IS**，`”SELECT…FOR UPDATE”`就是**IX**。一个事务要想获取一行数据的**X锁**，那么必须先获取所在表的**IX锁**；而如果想要获取**S锁**，那么获取**IS锁**。意向锁本质是相当于告诉MySQL，我将要对某些数据加**X锁**或者**S锁**，在通过锁冲突间接管理锁颗粒细度。

## 记录锁

**记录锁（record lock）**是加在索引上的，比如`SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;`语句会阻止其他事务通过`t.c1=10`的索引去间接修改、删除其数据所在行。

像我们之前提到的**X锁**和**S锁**都是加在行上的，而**记录锁**是加在索引上的，那么如果这一列没有索引怎么办？InnoDB会创建一个隐含的**聚集索引**。

## 间隙锁

**间隙锁（gap lock）**也是加在索引上的，和**记录锁**区别在于，它是针对一个范围内的索引记录，比如`SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;`，如果这时候有其他事务想要插入一个**15**的值就会被阻止。

不同事务可以在同一段索引记录中取得冲突的间隙锁，这是因为间隙锁本身是为“**排他**”而生，如果想从索引中清除记录，则需要合并不同事务持有的该记录上的间隙锁。

## 下一键锁

InnoDB通过**下一键锁next-key lock**来实现**REPEATABLE READ**隔离级别下对**幻读问题**的解决，而下一键锁遵循**左开右闭原则**。

TODO:

4.**redo log、undo log、binlog**

5.SQL实际执行

# 参考

[[1] Oracle: What is MySQL？](https://www.oracle.com/mysql/what-is-mysql/)

[[1] Oracle: What is MySQL？](https://www.oracle.com/mysql/what-is-mysql/)

[[2] 美团技术：MySQL索引原理及慢查询优化](https://tech.meituan.com/2014/06/30/mysql-index.html)

[[3] 深入浅出MySQL](https://www.linuxidc.com/Linux/2016-05/130922.htm) 

[[4] MySQL Doc](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)

[[5] learning-note: MySQL](https://github.com/rbmonster/learning-note/blob/master/src/main/java/com/toc/MYSQL.md)