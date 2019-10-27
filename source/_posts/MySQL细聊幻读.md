---
title: MySQL细聊幻读
date: 2019-08-03 15:41:30
tags: MySQL
categories: 技术
---

本文将详细介绍幻读现象是什么，以及 InnoDB 存储引擎是如何解决这个问题的。

<!--阅读更多-->

## 引子

```
CREATE TABLE 't'
(
	'id' int(11) NOT NULL,
	'c'  int(11) default null,
	'd'  int(11) default null,
	PRIMARY KEY ('id'),
	KEY 'c' ('c')
) ENGINE=InnoDB


insert into t values(0,0,0),(5,5,5),(10,10,10),(15,15,15),(20,20,20),(25,25,25)
```

通过以上创建SQL语句，建立一张表 `t`，并插入6条数据。

先设想一下下面一句的加锁过程：

```
begin;
select * from t where d=5 for update;
commit;
```

`d` 字段没有加索引，所以会进行全盘扫描，最终会命中 id=5 的一行，然后加上写锁，直到事务 commit。

问题在于：**对于扫描过程中不匹配 d=5 条件的行，是否需要加锁**。

我们在下面会对这两种场景进行验证，分别观察会发生什么？

1. 对不命中的行加锁
2. 只对命中的行加锁

## 场景2

### 幻读

先看下下面一个场景。

|      | sessionA                                                | sessionB                     | sessionC                    |
| ---- | ------------------------------------------------------- | ---------------------------- | --------------------------- |
| T1   | begin;<br />select * from t where d = 5 for update;//Q1 |                              |                             |
| T2   |                                                         | update t set d=5 where id=0; |                             |
| T3   | select * from t where d=5 for update;//Q2               |                              |                             |
| T4   |                                                         |                              | insert into t values(1,1,5) |
| T5   | select * from t where d = 5 for update;//Q3             |                              |                             |
| T6   | commit                                                  |                              |                             |

sessionA 里面执行了三次查询，分别是 `Q1`、`Q2`和`Q3`，而且使用的方式一样，都使用的是**当前读**，并且加上**写锁**。

`Q1` 返回结果：（5,5,5）

`Q2` 返回结果：（0,0,5）和（5,5,5）

`Q3` 返回结果：（0,0,5）、（5,5,5）和（1,1,5）

其中，`Q3` 读到 id=1 这一行记录的现象称为幻读现象。

幻读就是：**一个事务在前后两次查询同一个范围的时候，后一次的查询看到了前一次查询没有看到的行**。

两个关键点：

1. 在 RR 隔离级别下，普通的查询是**快照读**，只有在**当前读**的条件下才会出现幻读；
2. **幻读只针对“新插入的行”而言**。所以在上面的场景中： SessionB 的修改造成 Q2 的查询结果多出一行结果，但是由于 id=0 的那一行记录并非是“新插入”的，所以并不是幻读。

由于 MySQL 默认的存储引擎是 InnoDB，而其默认的事务隔离级别是 RR，所以**本文所讨论的问题均是在该隔离级别的前提下进行的**。

### 幻读带来的问题

#### 语义问题

sessionA 在 `T1` 时刻声明：“我要把所有 d=5 的行锁住，不准别的事务对其进行读写操作”。

但是实际上，有没有全部锁住呢？

我们尝试对上面的场景做一点变动。

|      | sessionA                                              | sessionB                                                     | sessionC                                                     |
| ---- | ----------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| T1   | begin;<br />select * from t where d=5 for update;//Q1 |                                                              |                                                              |
| T2   |                                                       | update t set d=5 where id=0;<br />**update t set c=5 where id=0**; |                                                              |
| T3   | select * from t where d=5 for update;//Q2             |                                                              |                                                              |
| T4   |                                                       |                                                              | insert into t values(1,1,5);<br />**update t set c=5 where id=1;** |
| T5   | select * from t where d=5 for update;//Q3             |                                                              |                                                              |
| T6   | commit                                                |                                                              |                                                              |

`T1` 时刻声明：我要对所有 d=5 的语句加锁，这其中不包含 id=0 这一行。

`T2` 时刻的第一句：将 id=0 这一行的 `d` 字段的值更改为 5，此时 id=0 这一行也符合 `T1` 时刻 SessionA 声明的语义，但是由于在 `T1` 时刻并没有将该行上锁，所以，接下来的更改 `c` 的值语句并不会阻塞。

sessionC 在 `T4` 时刻的更改语句也同理。

#### 数据一致性问题

考虑下面的一个场景。

|      | sessionA                                                     | sessionB                                                     | sessionC                                                     |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| T1   | begin;<br />select * from t where d=5 for update //Q1<br />update t set d=100 where d=5 |                                                              |                                                              |
| T2   |                                                              | update t set d=5 where id=0;<br />update t set c =5 where id=0 |                                                              |
| T3   | select * from t where d = 5 for update //Q2                  |                                                              |                                                              |
| T4   |                                                              |                                                              | insert into values(1,1,5);<br />update t set c=5 where id=1; |
| T5   | select * from t where d=5 for update;//Q3                    |                                                              |                                                              |
| T6   | commit;                                                      |                                                              |                                                              |

根据上表的执行序列可知：

`T1` 时刻：

更改的数据为 id=5 所在行，新的数据为 ： (5,5,100)  ，该更改在 `T6` 时刻提交。

`T2` 时刻：

更改的记录为 id=0 所在行 ，新的数据为： (0,5,5）。

`T4` 时刻：

表中新插入了一行数据：(1,1,5)，接着对其进行更改为：（1,5,5）



从前文我们知道 MySQL 中有 binlog 可以记录更改操作。

binlog 中的内容依照时间顺序变更如下：

1. T2 时刻，SessionB 事务提交，写入两条语句；
2. T4 时刻，SessionC 事务提交，写入两条语句；
3. T6 时刻，SessionA 事务提交。

放到一起如下：

> 事务B：
>
> update t set d=5 where id=0;  /*(0,0,5)*/
> update t set c=5 where id=0; /*(0,5,5)*/
>
> 事务C：
>
> insert into t values(1,1,5); /*(1,1,5)*/
> update t set c=5 where id=1; /*(1,5,5)*/
>
> 事务A：
>
> update t set d=100 where d=5;/* 所有 d=5 的行，d 改成 100*/

如果这个语句序列拿到备库去执行，或者克隆一个库，最终都会发生数据不一致的现象。

对比可知：

在数据表中：

id=0 所在行的数据为：（0,5,5）；

id=1 所在行的数据为：（1,5,5）。

而按照 binlog 执行后的库，这两行记录的结果为：

id=0 所在行的数据为：（0,5,100）；

id=1 所在行的数据为：（1,5,100）

由此产生了**数据不一致**的现象。

以上都是在场景2的设想下进行的，即：只对命中的行加锁。

那么要是在场景1下进行，是否还有问题呢？

## 场景1

在场景1下，我们对所有扫描过的行都加锁。

|      | sessionA                                                     | sessionB                                                     | sessionC                                                     |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| T1   | begin;<br />select * from t where d=5 for update;//Q1<br />update t set d=100 where d=5 |                                                              |                                                              |
| T2   |                                                              | update t set d=5 where id=0;<br />(**<div color=red>blocked</div>**)<br />update t set c=5 where id=0 |                                                              |
| T3   | select * from t where d=5 for update;//Q2                    |                                                              |                                                              |
| T4   |                                                              |                                                              | insert into t values(1,1,5);<br />update t set c=5 where id=1 |
| T5   | select * from t where d=5 for update;//Q3                    |                                                              |                                                              |
| T6   | commit                                                       |                                                              |                                                              |

在此场景下，由于对所有行都加锁，所以 SessionB 在 `T2` 时刻的 update 操作要阻塞，直到在 T6 时刻，SessionA 将事务提交。

表中的数据更新流程如下：

T1 时刻：id=5 所在行命中，更新为 （5,5,100），直到 T6 时刻才提交；

T4 时刻：插入新行（1,1,5），之后又被更新为（1,5,5），id=1（由于是新加的行，所以不受行锁影响）；

T6 时刻，SessionA 的事务提交之后，id=0所在行被更改为 （0,5,5）。



我们再看 binlog 中的更新记录：

```
insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

update t set d=100 where d=5;/* 所有 d=5 的行，d 改成 100*/

update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/
```

对比上面可以看出：id=1 的一行，在表中，最终被更新为：（1,5,5），而根据 binlog 的执行结果却是：（1,5,100）。

**数据不一致的现象依然存在**。

原因在于：`T3` 时刻对所有行上锁时，新插入的记录 （1,1,5）还没有出现在数据表中。

## 如何解决幻读

综合以上两种场景，可以看出：无论是对要更新的行加锁还是对所有行加锁，都无法解决**新插入数据**时引起的幻读问题。

于是针对新插入的数据，InnoDB 引入了间隙锁（Gap Lock）。

### 间隙锁

顾名思义，间隙锁锁的就是两行记录之间的间隙，锁住间隙自然就阻止了数据的插入。

<div align=center>![间隙锁](./gap_lock.png)

上图就是间隙锁的一个示意图。

这样，就可以保证在执行 select * from t where d=5 for update 命令的同时，不止给数据库中已知的 6 行记录加上锁，同时还加了 7 个间隙锁，这样就导致无法再插入新记录。

### 间隙锁关系

行锁的读写锁之间的关系：

|      | 读锁                      | 写锁 |
| ---- | ------------------------- | ---- |
| 读锁 | **<div color=green>兼容** | 冲突 |
| 写锁 | 冲突                      | 冲突 |

行锁中只有读锁和读锁兼容，而**间隙锁和间隙锁之间不存在冲突关系**。

比如下面的场景：

| sessionA                                                  | sessionB                                          |
| --------------------------------------------------------- | ------------------------------------------------- |
| begin;<br />select * from t where c=7 lock in share mode; |                                                   |
|                                                           | begin;<br />select * from t where c=7 for update; |

因为表中没有 c=7 这一条记录，因此 SessionA 和 SessionB 都加的是间隙锁 (5,10）,所以不会造成冲突。

### next-key lock

间隙锁和行锁合称为 next-key lock。

每个 next-key 都是左开右闭的区间。

以引子中的表为例，对该表执行下面的命令：

> select * from t for update

就相当于对该表添加了 7 个 next-key lock：(-∞,0]、 (0,5]、(5,10]、(10,15]、(15,20]、(20,25]、(25,supermum]

supermum 是 InnoDB 给每个索引添加的一个不存在的最大值。



间隙锁和行锁合作，解决了幻读的问题。



（全文完）







## 参考资料

1. 《MySQL 45讲》