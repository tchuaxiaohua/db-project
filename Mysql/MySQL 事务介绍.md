Mysql 事务整理(ACID)

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
  # 1. 窗口A 操作
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
  
  # 2. 窗口B 操作
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
  
  总结：可以看到虽然窗口B事务并未提交，但是窗口A已经读取到事务B更新未提交的的数据，这种级别是所有隔离中最低级的一种，这种情况下当窗口B进行回滚操作(rollback)后，窗口A读取的数据会变回之前未改变之前。
  
  ~~~

* 读已提交(read committed)















参考文档：

https://www.jianshu.com/p/4e3edbedb9a8

https://www.jianshu.com/p/75187e19faf2

https://www.cnblogs.com/wyaokai/p/10921323.html

https://www.cnblogs.com/lmj612/p/10579475.html



| 事务隔离级别                 | 脏读 | 不可重复读 | 幻读 |
| ---------------------------- | ---- | ---------- | ---- |
| 读未提交（read-uncommitted） | 是   | 是         | 是   |
| 读已提交（read-committed）   | 否   | 是         | 是   |
| 可重复读（repeatable-read）  | 否   | 否         | 是   |
| 串行化（serializable）       | 否   | 否         | 否   |

