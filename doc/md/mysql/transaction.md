## 基础概念
事务（Transaction）是访问和更新数据库的程序执行单元；事务中可能包含一个或多个sql语句，这些语句要么都执行，要么都不执行。
作为一个关系型数据库，MySQL支持事务，本文介绍基于MySQL5.6
## 基础语法
- 事务基本内容
```
start transaction;
……    #sql语句
commit;
```
- MySQL中默认采用的是自动提交（autocommit）模式。查看：show variables like 'autocommit';
- 在自动提交模式下，如果没有start transaction显式地开始一个事务，那么每个sql语句都会被当做一个事务执行提交操作
- 关闭autocommit,可以针对连接，在一个连接中修改参数  set autocommit = 0;
- DDL语句(create table/drop table/alter/table)、lock tables语句 会强制执行commit操作

## ACID 
### 原子性（Atomicity，或称不可分割性）
原子性是指一个事务是一个不可分割的工作单位，其中的操作要么都做，要么都不做；如果事务中一个sql语句执行失败，则已执行的语句也必须回滚，数据库退回到事务前的状态。
#### undo log(回滚日志)
undo log则是事务原子性和隔离性实现的基础<br>
**当事务对数据库进行修改时，InnoDB会生成对应的undo log；如果事务执行失败或调用了rollback，导致事务需要回滚，便可以利用undo log中的信息将数据回滚到修改之前的状态**<br>
当发生回滚时，InnoDB会根据undo log的内容做与之前相反的工作：
- 对于每个insert，回滚时会执行delete；
- 对于每个delete，回滚时会执行insert；
- 对于每个update，回滚时会执行一个相反的update，把数据改回去。

### 持久性（Durability）
#### redo log
redo log和undo log都属于**InnoDB**的事务日志<br>

**Buffer Pool**
```
Buffer Pool中包含了磁盘中部分数据页的映射，作为访问数据库的缓冲：
当从数据库读取数据时，会首先从Buffer Pool中读取，如果Buffer Pool中没有，则从磁盘读取后放入Buffer Pool；
当向数据库写入数据时，会首先写入Buffer Pool，Buffer Pool中修改的数据会定期刷新到磁盘中（这一过程称为刷脏）
带来的问题：
如果MySQL宕机，而此时Buffer Pool中修改的数据还没有刷新到磁盘，就会导致数据的丢失，事务的持久性无法保证。
```

**redo log工作原理**<br>
数据修改时，除了修改Buffer Pool中的数据，还会在redo log记录这次操作；
当事务提交时，会调用fsync接口对redo log进行**刷盘**。
如果MySQL宕机，重启时可以读取redo log中的数据，**对数据库进行恢复**。
redo log采用的是**WAL（Write-ahead logging，预写式日志）**，所有修改先写入日志，
再更新到Buffer Pool，保证了数据不会因MySQL宕机而丢失，从而满足了持久性要求。<br>

**为什么它比直接将Buffer Pool中修改的数据写入磁盘(即刷脏)要快呢？**<br>
- 刷脏是随机IO，因为每次修改的数据位置随机，但写redo log是追加操作，属于顺序IO。
- 刷脏是以数据页（Page）为单位的，MySQL默认页大小是16KB，一个Page上一个小修改都要整页写入；而redo log中只包含真正需要写入的部分，无效IO大大减少。

### 隔离性（Isolation）
隔离性是指，事务内部的操作与其他事务是隔离的，并发执行的各个事务之间不能互相干扰。严格的隔离性，对应了事务隔离级别中的Serializable (可串行化)，但实际应用中出于性能方面的考虑很少会使用可串行化<br>
- (一个事务)写操作对(另一个事务)写操作的影响：锁机制保证隔离性
- (一个事务)写操作对(另一个事务)读操作的影响：MVCC保证隔离性

**锁机制**<br>
1. 事务在修改数据之前，需要先获得相应的锁；
2. 获得锁之后，事务便可以修改数据；
3. 该事务操作期间，这部分数据是锁定的，其他事务如果需要修改数据，需要等待当前事务提交或回滚后释放锁。

**行锁与表锁**<br>
表锁在操作数据时会锁定整张表，并发性能较差；行锁则只锁定需要操作的数据，并发性能好。<br>
MyIsam只支持表锁，而InnoDB同时支持表锁和行锁，且出于性能考虑，绝大多数情况下使用的都是行锁。
```
select * from information_schema.innodb_locks; #锁的概况
show engine innodb status; #InnoDB整体状态，其中包括锁的情况
```

**脏读、不可重复读和幻读**<br>
- 脏读：当前事务(A)中可以读到其他事务(B)未提交的数据（脏数据）
- 不可重复读：在事务A中先后两次读取同一个数据，两次读取的结果不一样，这种现象称为不可重复读。脏读与不可重复读的区别在于：前者读到的是其他事务未提交的数据，后者读到的是其他事务已提交的数据。
- 幻读：在事务A中按照某个条件先后两次查询数据库，两次查询结果的条数不同，这种现象称为幻读。不可重复读与幻读的区别可以通俗的理解为：前者是数据变了，后者是数据的行数变了

**事务隔离级别**<br>

|  隔离级别   | 脏读  |  不可重复读   | 幻读  |
|  ----  | ----  |  ----  | ----  |
| Read Uncommitted(读未提交) | 可能 | 可能  | 可能 |
| Read Committed(读已提交)  | 不可能 | 可能  | 可能 |
| Repeatable Read(可重复读)  | 不可能 | 不可能  | 可能 |
| Serializable(可串行化)  | 不可能 | 不可能  | 不可能 |

**InnoDB实现的RR，通过锁机制、数据的隐藏列、undo log和类next-key lock，实现了一定程度的隔离性，可以满足大多数场景的需要。**

```
## 查看全局隔离级别和本次会话隔离级别  InnoDB默认的隔离级别是RR
select @@global.tx_isolation;
select @@tx_isolation;
```
##### MVCC
MVCC的特点：
- 在同一时刻，不同的事务读取到的数据可能是不同的(即多版本)
- 读不加锁，因此读写不冲突，并发性能好。

**MVCC实现原理**：InnoDB实现MVCC，多个版本的数据可以共存，主要是依靠数据的隐藏列(也可以称之为标记位)和undo log。其中数据的隐藏列包括了该行数据的版本号、删除时间、指向undo log的指针等等；
当读取数据时，MySQL可以通过隐藏列判断是否需要回滚并找到回滚需要的undo log<br>

**避免脏读**

|  时间   | 事务A  |  事务B   | 
|  ----  | ----  |  ----  | 
| T1   | 开始事务 | 开始事务  | 
| T2   |  | 修改数据Record由100改为200  | 
| T3   | 查询Record数据为100(避免了脏读) |   | 
| T4   |  | 提交事务  | 

当事务A在T3时间节点读取Record时，会发现数据已被其他事务修改，且状态为未提交。
此时事务A读取最新数据后，根据数据的undo log执行**回滚操作**，得到事务B修改前的数据，从而避免了脏读。

<br>**避免不可重复读**

|  时间   | 事务A  |  事务B   | 
|  ----  | ----  |  ----  | 
| T1   | 开始事务 | 开始事务  | 
| T2   | 查询Record数据为100 |  | 
| T3   | | 修改数据Record由100改为200   | 
| T4   |  | 提交事务  | 
| T4   | 查询Record数据为100(避免了不可重复读)  |   | 

1. 当事务A在T2节点第一次读取数据时，会记录该数据的版本号（数据的版本号是以row为单位记录的），假设版本号为1；
2. 当事务B提交时，该行记录的版本号增加，假设版本号为2；
3. 当事务A在T5再一次读取数据时，发现数据的版本号（2）大于第一次读取时记录的版本号（1），因此会根据undo log执行回滚操作，得到版本号为1时的数据，从而实现了可重复读。

<br>**避免幻读**<br>
InnoDB实现的RR通过next-key lock机制避免了幻读现象。<br>

next-key lock是行锁的一种，实现相当于record lock(记录锁) + gap lock(间隙锁)；其特点是不仅会锁住记录本身(record lock的功能)，还会锁定一个范围(gap lock的功能)。

|  时间   | 事务A  |  事务B   | 
|  ----  | ----  |  ----  | 
| T1   | 开始事务 | 开始事务  | 
| T2   | 查询RecordId数据 0~5的数据，所有数据总和200 |  | 
| T3   | | 插入新Record数据   | 
| T4   |  | 提交事务  | 
| T4   | 查询RecordId数据 0~5的数据，所有数据总和还是200(避免了不可重复读)  |   | 

### 一致性（Consistency）
一致性是指事务执行结束后，数据库的完整性约束没有被破坏，事务执行的前后都是合法的数据状态。<br>
数据库的完整性约束包括但不限于：实体完整性（如行的主键存在且唯一）、列完整性（如字段的类型、大小、长度要符合要求）、外键约束、用户自定义完整性（如转账前后，两个账户余额的和应该不变）。

## 参考文章
[深入学习MySQL事务：ACID特性的实现原理](https://www.cnblogs.com/kismetv/p/10331633.html)
[详细分析MySQL事务日志(redo log和undo log)](https://juejin.im/entry/5ba0a254e51d450e735e4a1f)