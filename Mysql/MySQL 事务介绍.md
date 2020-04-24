## MySQL 事务整理(ACID)

### 1 、事务特性

* 原子性(Atomicity)

  ~~~python
    原子性(Atomicity) 原子性是指事务包含的一系列操作要么全部成功，要么全部回滚，不存在部分成功或者部分回滚，是一个不可分割的操作整体
  ~~~

* 一致性(Consistency)

  ~~~python
    一致性是可以理解为事务对数据完整性约束的遵循，这些约束可能包括主键约束、唯一索引约束、外键约束等等。事务执行前后，数据都是合法的状态，不会违背任何的数据完整性。
    就拿转账来说，A和B加起来有5000块钱，不管A和B如何转账，转几次账，A和B加起来的钱永远都是5000块。
  总之，可以理解为：一致性是为了保证数据的完整性。
  ~~~

* 隔离性(Isolation)

  ~~~python
    隔离性是指当多个用户并发操作数据库，比如操作同一张表，数据库为每一个用户开启的事务，不能被其他的事务所干扰或者影响，事务之间是彼此独立的
  ~~~

* 永久性(Durability)

  ~~~python
    永久性是指一个事务一旦提交了，那么对数据库中数据的改变就是永久的，即使是在数据库发生故障时，也不会丢失事务提交的数据
  ~~~

### 2、 事务的隔离性

* 隔离级别设置

  ~~~python
  # 查看隔离级别参数
  show VARIABLES like 'tx_isolation' 
  select @@global.tx_isolation
  事务的隔离性。当多个线程开启事务操作数据库中的数据时，数据库要能进行隔离操作，以保证各个线程获取数据的准确性; 如果不考虑事务的隔离性，会发生以下几个问题;
  1. 脏读
  2. 不可重复读
  3. 幻读
  ~~~

* 读未提交(read-uncommitted) 

  ~~~python
  事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据，也称之为脏读。
  ~~~

  事例：

  ~~~mysql
  # 1. 造数据
  create database bank charset utf8mb4;
  use bank
  mysql> create table account(account_num varchar(10) primary key,balance int);
  mysql> inset into account values('A',600),('B',400);
  mysql> insert into account values('A',600),('B',400);
  # 2. 数据查看
  mysql> SELECT * FROM account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |     600 |
  | B           |     400 |
  +-------------+---------+
  
  ~~~

  ~~~mysql
  # 1. 窗口A 操作 设置事务隔离界别 开启事务
  mysql> set session transaction isolation level read uncommitted;
  mysql> begin;
  Query OK, 0 rows affected (0.00 sec)
  
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |     600 |
  | B           |     400 |
  +-------------+---------+
  
  # 2. 窗口B 操作 开启事务 更新数据 不提交
  mysql> set session transaction isolation level read uncommitted;
  mysql> use bank
  mysql> begin;
  mysql> update account set balance=balance + 200 where account_num = 'B';
  
  # 3. 窗口A 查询
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |     600 |
  | B           |     600 |
  +-------------+---------+
  
  # NOTE：虽然窗口B事务还未提交此时，窗口已经读取到窗口B更新的数据，这也属于脏读。
  总结：可以看到虽然窗口B事务并未提交，但是窗口A已经读取到事务B更新未提交的的数据，这种级别是所有隔离中最低级的一种，这种情况下当窗口B进行回滚操作(rollback)后，窗口A读取的数据会变回之前未改变之前。
  
  ~~~

* 读已提交(read committed)

  ~~~python
  可以读取其他事务提交的数据
  ~~~

  事例：

  ~~~mysql
  # 窗口A 设置事务隔离级别 开启事务，更新数据 不提交
  mysql> set session transaction isolation level read uncommitted;
  mysql> use bank
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |     600 |
  | B           |     400 |
  +-------------+---------+
  mysql> begin;
  mysql> update account set balance=balance-200 where account_num='A';
  
  # 窗口B 
  mysql> use bank
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |     600 |
  | B           |     400 |
  +-------------+---------+
  # NOTE： 由于窗口A事务未提交，所以窗口此时查看数据未改变
  
  # 窗口A
  mysql> commit;  
  
  # 窗口B
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |     400 |
  | B           |     400 |
  +-------------+---------+
  # NOTE：窗口B再次查询时，发现数据较上次已经发生了变化，这是因为窗口A事务已提交。
  
  总结：当把事务的隔离级别设置为read committed的时候，会话只能读取到其他事务提交的数据，未提交的数据读不到，已提交读隔离级别解决了脏读的问题，但是出现了不可重复读的问题，即事务B在两次查询的数据不一致，因为在两次查询之间事务B更新了一条数据。已提交读只允B许读取已提交的记录，但不要求可重复读。
  
  ~~~

* repeatable read(可重复读)

  ~~~python
  MySQL默认的隔离级别
  ~~~

  事例：

  ~~~python
  # 窗口A 设置事务隔离级别，启动事务
  mysql> set session transaction isolation level repeatable read;
  mysql> begin;
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |     400 |
  | B           |     400 |
  +-------------+---------+
  # 窗口B 启动事务 更新数据，不提交
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |     400 |
  | B           |     400 |
  +-------------+---------+
  mysql> begin;
  mysql> update account set balance = 1000 where account_num = 'A';
  
  # 窗口A 再次查看数据
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |     400 |
  | B           |     400 |
  +-------------+---------+
  
  # 窗口B 提交事务
  mysql> commit;
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |    1000 |
  | B           |     400 |
  +-------------+---------+
  
  # 窗口A 窗口B提交事务后窗口A再次查看
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |     400 |
  | B           |     400 |
  +-------------+---------+
  # NOTE：数据还是未变化，说明可以重复读了
  
  # 窗口B 插入新的数据，提交
  mysql> begin;
  mysql> insert into account values('C','500');
  mysql> commit;
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |    1000 |
  | B           |     400 |
  | C           |     500 |
  +-------------+---------+
  
  # 窗口A 窗口B提交后窗口A继续查看
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |     400 |
  | B           |     400 |
  +-------------+---------+
  
  # NOTE：由于窗口A的事务一直没提交，所以查询到的数据一直未变化，实际上数据已经发生，窗口A中的事务查询的数据与实际不一致。这就是幻读。
  # 窗口A 提交事务，再次查看
  mysql> commit;
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |    1000 |
  | B           |     400 |
  | C           |     500 |
  +-------------+---------+
  # NOTE： 事务提交后，数据可以正常查看
  
  总结：当前会话事务可以重复读，就是每次读取的结果集都相同，而不管其他事务有没有提交。但该事务不要求与其他事务可串行化。例如，当一个事务可以找到由一个已提交事务更新的记录，但是可能产生幻读问题(注意是可能，因为数据库对隔离级别的实现有所差别)。在实际的应用场景中，比如事务A新添加一条account_num = 'D'的记录，由于事务隔离界别为可重复读，此时事务B无法查看事务A更新的该条记录，这时事务B也插入一条account_num = 'D'记录，此时就会报错，因为实际上数据已经发生改变，数据库还是要保持数据的一致性。这个就是幻读。
  ~~~

* 串行化(serializable)

  ~~~python
  事务最严格的隔离级别
  ~~~

  事例：

  ~~~mysql
  # 窗口A 设置事务隔离级别 开启事务
  mysql> set session transaction isolation level serializable;
  mysql> begin;
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |    1000 |
  | B           |     400 |
  | C           |     500 |
  +-------------+---------+
  
  # 窗口B 对该表进行数据修改操作
  mysql> update account set balance = 1200 where account_num = 'A';
  ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
  # NOTE：发现B此时进入了等待状态，原因是因为A的事务尚未提交，只能等待，直到事务A提交（如果超时会提示：Lock wait time out），如果在等待期间我们窗口A所在的会话事务提交，那么窗口B所在的事务的更新操作将提示操作成功。
  
  总结：
  serializable隔离级别完全锁定字段，若另一个事务来对同一份数据进行更新就必须等待，直到前一个事务完成并解除锁定为止。是完整的隔离级别，会锁定对应的数据表格，因而会有效率的问题。
  ~~~

### 3、事务隔离对应表

| 事务隔离级别                 | 脏读 | 不可重复读 | 幻读 |
| ---------------------------- | ---- | ---------- | ---- |
| 读未提交（read-uncommitted） | 是   | 是         | 是   |
| 读已提交（read-committed）   | 否   | 是         | 是   |
| 可重复读（repeatable-read）  | 否   | 否         | 是   |
| 串行化（serializable）       | 否   | 否         | 否   |

### 4、事务生命周期

#### 4.1 控制语句

~~~mysql
begin：开启事务(start transaction)
commit：提交事务
rollback ：回滚事务

# 注意：在存储过程中开启事务时必须使用start transaction，因为begin会被存储过程解析为begin...end结构块
~~~

#### 4.2 自动提交策略

~~~mysql
select @@autocommit; 查询提交策略
autocommit参数说明：
1：事务开启和事务提交 都是自动
0：手动提交

set autocommit=0; 当前会话，重新登录失效
set global autocommit=0; 全局会话，重启后失效
~~~

#### 4.3 隐式提交

* 参数

  ~~~mysql
  autocommit=1
  ~~~

* 上一个事务未提交，直接开启下一个事务

  ~~~mysql
  # 窗口A
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |    1000 |
  | B           |     400 |
  | C           |     500 |
  +-------------+---------+
  mysql> begin;
  mysql> update account set balance = 1200 where account_num = 'A';
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |    1200 |
  | B           |     400 |
  | C           |     500 |
  +-------------+---------+
  mysql> begin;
  # 窗口B
  mysql> select * from account;
  +-------------+---------+
  | account_num | balance |
  +-------------+---------+
  | A           |    1200 |
  | B           |     400 |
  | C           |     500 |
  +-------------+---------+
  ~~~

* 导致提交的非事务语句

  ~~~shell
  导致提交的非事务语句：
  DDL语句： (ALTER、CREATE 和 DROP)
  DCL语句： (GRANT、REVOKE 和 SET PASSWORD)
  锁定语句：(LOCK TABLES 和 UNLOCK TABLES)
  管理语句： (analyze table、cache index、check table、load index into cache、optimize table、repair table)
  ~~~

* 导致隐式提交的语句

  ~~~shell
  TRUNCATE TABLE
  LOAD DATA INFILE
  SELECT FOR UPDATE
  ~~~

### 5、InnoDB 事务的ACID保证

#### 5.1 redo log

* redo log  重做日志，提供前滚操作，来保证事务的原子性和持久性，恢复提交事务修改的数据页操作

  ~~~shell
  属于物理日志，记录的是数据页的物理修改，而不是某一行或某几行修改成怎样怎样，它用来恢复提交后的物理数据页(恢复数据页，且只能恢复到最后一次提交的位置)。
  redo log由两部分组成：
  	1) 内存中的日志缓冲(redo log buffer)，该部分日志是易失性的
  	2) 磁盘上的重做日志文件(redo log file)，该部分日志是持久的
  redo log磁盘日志文件：ib_logfile0、ib_logfile1 
  	磁盘上redo log file个数由innodb_log_files_group决定，默认为2
  mysql> show global variables like "innodb_log%";
  +-----------------------------+----------+
  | Variable_name               | Value    |
  +-----------------------------+----------+
  | innodb_log_buffer_size      | 16777216 |
  | innodb_log_checksums        | ON       |
  | innodb_log_compressed_pages | ON       |
  | innodb_log_file_size        | 50331648 |
  | innodb_log_files_in_group   | 2        |
  | innodb_log_group_home_dir   | ./       |
  | innodb_log_write_ahead_size | 8192     |
  +-----------------------------+----------+
  
  ~~~
  
* redo log block
  
  ~~~mysql
    innodb存储引擎中，redo log以块为单位进行存储的，每个块占512字节，这称为redo log block。
    每个redo log block由3部分组成：日志块头、日志块尾和日志主体。其中日志块头占用12字节，日志块尾占用8字节，所以每个redo log block的日志主体部分只有512-12-8=492字节。
  ~~~
  
  * LSN (Log seq num) 日志逻辑序列号
  
    ~~~mysql
    mysql> show engine innodb status\G;
    ---
    LOG
    ---
    Log sequence number 2642560
    Log flushed up to   2642560
    Pages flushed up to 2642560
    Last checkpoint at  2642551
    参数说明：
    1) Log sequence number: 当前的redo log(内存)中的LSN
    2) Log flushed up to: 刷到redo log file(磁盘redo log文件)中的LSN
    3) pages flushed up to: 已经刷到磁盘数据页上的LSN
    4) last checkpoint at: 上一次检查点所在位置的LSN
    
    ~~~
  
    
  
  * 日志刷盘的策略
  
    >InnoDB通过Force Log at Commit机制来实现持久性，当commit时，必须先将事务的所有日志写到重做日志文件进行持久化，待commit操作完成才算完成。
  
    
  
  ~~~shell
  参数：innodb_flush_log_at_trx_commit ，事务日志落盘策略
  0：事务提交时不会将log buffer中日志写入到os buffer，而是每秒写入os buffer并调用fsync()写入到log file on disk中。也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，当系统崩溃，会丢失1秒钟的数据。
  1：事务每次提交都会将log buffer中的日志写入os buffer并调用fsync()刷到log file on disk中。这种方式即使系统崩溃也不会丢失任何数据，但是因为每次提交都写入磁盘，IO的性能较差。
  2：每次提交都仅写入到os buffer，然后是每秒调用fsync()将os buffer中的日志写入到log file on disk。
  ~~~

#### 5.2 undo log 

> undo log 提供回滚和多个行版本控制(MVCC) 来保证事务的一致性，undo回滚行记录到某个特性版本及MVCC功能

* 概念

  ~~~shell
  # 概念
    在数据修改的时候，不仅记录了redo，还记录了相对应的undo，如果因为某些原因导致事务失败或回滚了，可以借助该undo进行回滚。
    undo log和redo log记录物理日志不一样，它是逻辑日志。可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的update记录。
    当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚。有时候应用到行版本控制的时候，也是通过undo log来实现的：当读取的某一行被其他事务锁定时，它可以从undo log中分析出该行记录以前的数据是什么，从而提供该行版本信息，让用户实现非锁定一致性读取。
  undo log是采用段(segment)的方式来记录的，每个undo操作在记录的时候占用一个undo log segment。
  另外，undo log也会产生redo log，因为undo log也要实现持久性保护。
  ~~~

* 存储方式

  ~~~shell
  默认存放在共享表空间中：ibdata1(5.7版本)，如果开启了innodb_file_per_table参数，将放在每个表的.ibd文件中
  ~~~

* MVCC

  ~~~shell
  
  ~~~

  参考文档：
  
  https://juejin.im/post/5e97d6b7e51d4546f5790f7e

#### 5.3 事务隔离级别

#### 5.4 锁机制

*** 单独章节讲解 ***

### 6、故障

通过阿里云sql洞察可以查看到有insert插入，实际数据库并未看到有数据插入

分析：

1、业务层有显式事务操作，但是并未进行事务提交，后面直接logout导致事务回滚

### 7、参考文档

https://www.cnblogs.com/f-ck-need-u/archive/2018/05/08/9010872.html#auto_id_9