MySQL 锁机制

官方介绍：https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-transaction-model.html

### 一、MySQl中的锁

~~~shell
锁和事务的实现是存储引擎内的组件管理的，而MariaDB/MySQL是插件式的存储引擎实现方式，所以不同的存储引擎可以支持不同级别的锁和事务。
~~~

#### 1.1 不同存储引擎支持的锁级别

~~~shell
1) MyISAM、Aria(MariaDB中对myisam的改进版本)和memory存储引擎只支持表级别的锁。
2) innodb支持行级别的锁和表级别的锁，默认情况下在允许使用行级别锁的时候都会使用行级别的锁。
3) DBD存储引擎支持页级别和表级别的锁。
~~~

#### 1.2 锁类型

~~~shell
# 表锁级别(支持表锁的存储引擎都会有的锁类型)
1) 共享锁(S)：即读锁，不涉及修改数据，是读取操作才申请创建的锁，其他用户可以并发读取数据，但任何事务都不能对数据进行修改（获取数据上的排他锁），直到已释放所有共享锁。
2) 独占锁(X)：即写锁，增、删、改等涉及修改操作的时候，会申请独占锁，也叫排它锁，若某个事物对某一行加上了排他锁，只能这个事务对其进行读写，在此事务结束之前，其他事务不能对其进行加任何锁，其他进程可以读取,不能进行写操作，需等待其释放。
# 意向锁(支持行锁或页锁才会有的锁类型)
3) 意向共享锁(IS): 获取低级别共享锁的同事，在高级别上也获取特殊的共享锁，这种特殊的共享锁是意向共享锁。
4) 意向独占锁(IX): 获取低级别独占锁的同时，在高级别上也获取特殊的独占锁，这种特殊的独占锁是意向独占锁。

说明：低级别锁表示的是行锁或页锁，意向锁可能是多条记录组成的范围锁，也可能直接就是表意向锁，排它锁会阻塞所有的排它锁和共享锁。
~~~

#### 1.3 锁等级

~~~shell
表级锁：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高 ，并发度最低。
页面锁：开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般。
行级锁：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高
~~~

#### 1.4 锁兼容性

|                | 共享锁(S) | 意向共享锁(IS) | 独占锁(X) | 意向独占锁(IX) |
| -------------- | --------- | -------------- | --------- | -------------- |
| 共享锁(S)      | √         | √              | ×         | ×              |
| 意向共享锁(IS) | √         | √              | ×         | √              |
| 独占锁(X)      | ×         | ×              | ×         | ×              |
| 意向独占锁(IX) | ×         | √              | ×         | √              |

**总结：**

  独占锁和所有的锁都冲突，意向共享锁和共享锁兼容(这是肯定的)，还和意向独占锁兼容。所以加了意向共享锁的时候，可以修改行级非共享锁的记录。同理，加了意向独占锁的时候，可以检索这些加了独占锁的记录。

### 二、MyISAM引擎锁

#### 2.1 表级锁(lock tables和unlock语句)

~~~shell
  MyISAM 是默认采用表锁。
  MySQL中myisam和innodb都支持表级锁。表级锁分为两种：读锁(read lock)和写锁(write lock)。锁表的时候可以一次性锁定多张表，并可以使用不同的锁，而解锁的时候只能一次性解锁当前客户端会话的所有表。
~~~

* 读锁：

  ~~~shell
  # 创建表造数据
  create database tchua charset utf8mb4;
  CREATE TABLE t1(a INT,b CHAR(5))ENGINE=MYISAM;
  CREATE TABLE t2(a INT,b CHAR(5))ENGINE=MYISAM;
  CREATE TABLE t3(a INT,b CHAR(5))ENGINE=MYISAM;
  INSERT INTO t1 VALUES(1,'a');
  INSERT INTO t2 VALUES(1,'a');
  INSERT INTO t3 VALUES(1,'a');
  
  # 会话1 给t1表加读锁
  mysql> lock table t1 read;
  mysql> select * from t1;
  +------+------+
  | a    | b    |
  +------+------+
  |    1 | a    |
  +------+------+
  mysql> update t1 set b = 'b';
  ERROR 1099 (HY000): Table 't1' was locked with a READ lock and can't be updated
  mysql> select * from t2;
  ERROR 1100 (HY000): Table 't2' was not locked with LOCK TABLES
  
  # NOTE： 此时会话1无法操作t1表以外的表，查询也不可以，对t1表只有读的权限。
  # 会话2
  mysql> select * from t1;
  +------+------+
  | a    | b    |
  +------+------+
  |    1 | a    |
  +------+------+
  mysql> select * from t2;
  +------+------+
  | a    | b    |
  +------+------+
  |    1 | a    |
  +------+------+
  mysql> update t2 set b = 'b';  -- 可以更新
  Query OK, 1 row affected (0.00 sec)
  Rows matched: 1  Changed: 1  Warnings: 0
  
  mysql> update t1 set b = 'b';  -- 一直等待
  # NOTE：会话2可以查询所有表，但是无法对t1表进行更新操作，
  # 会话1 给表t2加锁
  mysql> lock table t2 read;
  
  # NOTE：在同一个会话，再次使用lock table命令的时候，会先释放当前会话的所有锁，再对指定的表申请锁。
  
  总结：读锁，当前会话只有对申请读锁的表有读取的权限，对其它表无任何权限;其他会话,读操作不受影响，但是对申请读锁的表也没有更新权限，直到锁释放。另外：lock tables命令会隐式释放当前客户端会话中之前的所有锁。
  
  ~~~

* 写锁

  ~~~shell
  # 会话1 给t1表加写锁
  mysql> lock table t1 write;
  mysql> select * from t1;
  +------+------+
  | a    | b    |
  +------+------+
  |    1 | b    |
  +------+------+
  mysql> select * from t2;
  ERROR 1100 (HY000): Table 't2' was not locked with LOCK TABLES
  mysql> update t1 set b = 'a';
  Query OK, 1 row affected (0.00 sec)
  Rows matched: 1  Changed: 1  Warnings: 0
  mysql> update t2 set b = 'aa';
  ERROR 1100 (HY000): Table 't2' was not locked with LOCK TABLES
  # NOTE: 会话1加写锁后，当前会话只能对申请写锁的表读写操作，对其它表没有任何权限。
  
  # 会话2
  mysql> select * from t1;  -- 无法查询 锁等待
  mysql> select * from t2;
  +------+------+
  | a    | b    |
  +------+------+
  |    1 | a    |
  +------+------+
  mysql> update t1 set b = 'b'; -- 无法更新 所等待
  mysql> update t2 set b = 'b';
  Query OK, 1 row affected (0.00 sec)
  Rows matched: 1  Changed: 1  Warnings: 0
  # NOTE：由于t1表在会话1加了写锁，所以，其它会话对t1表就没有任何操作权限，但是对其他表可以正常操作。
  
  总结：写锁，当前会话对申请写锁的表有更新查询权限，对其它表无任何权限；其它会话，对申请写锁的表没有任何权限,写锁会阻塞所有的写锁和共享锁。
  
  ~~~

### 三、Innodb

~~~shell
innodb支持行级锁，也是在允许的情况下默认申请的锁.
加锁的方式：自动加锁。对于UPDATE、DELETE和INSERT语句，InnoDB会自动给涉及数据集加排他锁；对于普通SELECT语句，InnoDB不会加任何锁；当然我们也可以显示的加锁：
说明：SQL Server中的锁是一种稀有资源，且会在需要的时候锁升级，所以锁越多性能越差。而MariaDB/MySQL中的锁不是稀有资源，不会进行锁升级，因此锁的多少不会影响性能，1个锁和1000000个锁性能是一样的(不考虑锁占用的内存)，锁的多少只会影响并发性。

~~~

**注意：** 行级锁都是基于索引的，如果一条SQL语句用不到索引是不会使用行级锁的，会使用表级锁。

#### 3.1 锁信息查看

* 造数据

  ~~~mysql
  mysql> create database tchua charset utf8mb4;
  mysql> CREATE TABLE t1(a INT,b CHAR(5));
  mysql> use tchua
  mysql> CREATE TABLE t1(a INT,b CHAR(5));
  mysql> CREATE TABLE t2(a INT,b CHAR(5));
  mysql> INSERT INTO t1 VALUES(1,'a1'),(2,'a2');
  mysql> INSERT INTO t2 VALUES(1,'a1'),(2,'a2');
  
  ~~~

* 锁等待

  ~~~shell
  # 会话1 
  mysql> begin;
  mysql> update t1 set b = 'aa1' where a = 1;
  
  #会话2
  mysql> begin;
  mysql> update t1 set b = 'a12' where a = 1;
  ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
  
  # NOTE：因为是通同一个表同一行数据的操作，由于事务的隔离性会话2会被阻塞，如果会话1一直不提交事务，到锁等待超时抛出异常。
  ~~~

* 锁信息查看

  ~~~shell
  # 1) show engine innodb status
  ------------
  TRANSACTIONS
  ------------
  Trx id counter 3874
  Purge done for trx's n:o < 3872 undo n:o < 0 state: running but idle
  History list length 14
  LIST OF TRANSACTIONS FOR EACH SESSION:
  ---TRANSACTION 421224344288880, not started
  0 lock struct(s), heap size 1136, 0 row lock(s)
  ---TRANSACTION 3873, ACTIVE 5 sec starting index read
  mysql tables in use 1, locked 1
  LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
  MySQL thread id 12, OS thread handle 139748912609024, query id 183 localhost root updating
  update t1 set b = 'a12' where a = 1
  ------- TRX HAS BEEN WAITING 5 SEC FOR THIS LOCK TO BE GRANTED:
  RECORD LOCKS space id 29 page no 3 n bits 72 index GEN_CLUST_INDEX of table `tchua`.`t1` trx id 3873 lock_mode X waiting
  Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
   0: len 6; hex 000000000200; asc       ;;
   1: len 6; hex 000000000f20; asc       ;;
   2: len 7; hex 3a0000012e03d1; asc :   .  ;;
   3: len 4; hex 80000001; asc     ;;
   4: len 5; hex 6161312020; asc aa1  ;;
  
  ------------------
  ---TRANSACTION 3872, ACTIVE 16 sec
  2 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 1
  MySQL thread id 11, OS thread handle 139748913149696, query id 181 localhost root
  
  # 信息解释说明：
  1. 三个'---TRANSACTION'说明mysql开启三个会话。
  2. '------- TRX HAS BEEN WAITING 5 SEC FOR THIS LOCK TO BE GRANTED:' >> 表示该事务申请锁已经等待了5。
  3. 'RECORD LOCKS space id 29 page no 3 n bits 72 index GEN_CLUST_INDEX of table `tchua`.`t1` trx id 3873 lock_mode X waiting' >> 表示`tchua`.`t1`表上的记录要申请的行锁(recode lock)是独占锁并且正在waiting，并且标明了该行记录所在表数据文件中的物理位置：表空间id为29，页码为3。
  # 2) show full processlist
  mysql> show full processlist;
  +----+------+-----------+-------+---------+------+----------+-------------------------------------+
  | Id | User | Host      | db    | Command | Time | State    | Info                                |
  +----+------+-----------+-------+---------+------+----------+-------------------------------------+
  | 11 | root | localhost | tchua | Sleep   |   54 |          | NULL                                |
  | 12 | root | localhost | tchua | Query   |   43 | updating | update t1 set b = 'a12' where a = 1 |
  | 13 | root | localhost | NULL  | Query   |    0 | starting | show full processlist               |
  +----+------+-----------+-------+---------+------+----------+-------------------------------------
  
  # NOTE: 从上面的结果可以看出，update语句一直处于updating状态。所以，该方法查出来的并不一定是锁等待，有可能是更新的记录太多或者其他问题，总之这里看出来的是该语句还没有执行完成。
  
  # 3) information_schema 字典数据
    在information_schema架构下，有3个表记录了事务和锁相关的信息。分别是INNODB_TRX,INNODB_LOCKS,INNODB_LOCK_WAITS。
  ## INNODB_TRX表结构介绍(主要列不是全部)
  trx_id: 事务ID
  trx_state: 事务状态：RUNNING、LOCK WAIT、ROLLING BACK、COMMITTING
  trx_started: 事务的开始时间
  trx_requested_lock_id: 等待事务的锁ID，trx_state列为lock wait，这里显示对应的锁ID。
  trx_wait_started: 事务等待的开始时间，即什么时候事务开始进入等待。
  trx_weight: 事务的权重，反应了事务中修改过或者锁住的行数比重，当出现死锁时，会选择权重小的回滚。
  trx_mysql_thread_id: mysql中的线程ID，和show processlist中显示的ID对应。
  trx_query: 事务运行的SQL语句，不过有时候会显示null。
  trx_tables_locked: 当前事务有多少量的innodb表因为当前的语句被锁定。
  
  # INNODB_TRX具体锁查看语句
  mysql> select * from information_schema.INNODB_TRX\G;
  *************************** 1. row ***************************
                      trx_id: 3875
                   trx_state: LOCK WAIT
                 trx_started: 2020-04-22 15:51:13
       trx_requested_lock_id: 3875:29:3:2
            trx_wait_started: 2020-04-22 15:51:13
                  trx_weight: 2
         trx_mysql_thread_id: 12
                   trx_query: update t1 set b = 'a12' where a = 1
         trx_operation_state: starting index read
           trx_tables_in_use: 1
           trx_tables_locked: 1
            trx_lock_structs: 2
       trx_lock_memory_bytes: 1136
             trx_rows_locked: 1
           trx_rows_modified: 0
     trx_concurrency_tickets: 0
         trx_isolation_level: REPEATABLE READ
           trx_unique_checks: 1
      trx_foreign_key_checks: 1
  trx_last_foreign_key_error: NULL
   trx_adaptive_hash_latched: 0
   trx_adaptive_hash_timeout: 0
            trx_is_read_only: 0
  trx_autocommit_non_locking: 0
  *************************** 2. row ***************************
                      trx_id: 3872
                   trx_state: RUNNING
                 trx_started: 2020-04-22 15:07:50
       trx_requested_lock_id: NULL
            trx_wait_started: NULL
                  trx_weight: 3
         trx_mysql_thread_id: 11
                   trx_query: NULL
         trx_operation_state: NULL
           trx_tables_in_use: 0
           trx_tables_locked: 1
            trx_lock_structs: 2
       trx_lock_memory_bytes: 1136
             trx_rows_locked: 3
           trx_rows_modified: 1
     trx_concurrency_tickets: 0
         trx_isolation_level: REPEATABLE READ
           trx_unique_checks: 1
      trx_foreign_key_checks: 1
  trx_last_foreign_key_error: NULL
   trx_adaptive_hash_latched: 0
   trx_adaptive_hash_timeout: 0
            trx_is_read_only: 0
  trx_autocommit_non_locking: 0
  2 rows in set (0.00 sec)
  # 总结： 从结果中可以看出id为3875的事务正处于锁等待状态，该事务中要申请锁的语句是update语句，也就是说是因为该语句而导致的锁等待。innodb_trx表中只能查看到事务的信息，而不能看到锁相关的信息。
  
  ## innodb_locks结构
  lock_id: 锁ID 
  lock_trx_id: 事务ID
  lock_mode: 锁模式(S/X/IS/IX/)
  lock_type: 锁类型(表锁还是行锁)
  lock_table: 要申请锁的表
  lock_index: 锁的索引，若为GEN_CLUST_INDEX表示加锁的表没有索引而自动创建的隐式索引
  lock_space: innodb存储引擎中表空间的ID
  lock_page: 被锁住的页的数量，若为表锁，则值为null
  lock_rec: 被锁住的行数，若为表锁，则值为null
  lock_data: 被锁记录的行的主键值，若为表锁，则值为null
  
  ## innodb_locks锁信息查看
  mysql> select * from information_schema.INNODB_LOCKS\G;
  *************************** 1. row ***************************
      lock_id: 3875:29:3:2
  lock_trx_id: 3875
    lock_mode: X
    lock_type: RECORD
   lock_table: `tchua`.`t1`
   lock_index: GEN_CLUST_INDEX
   lock_space: 29
    lock_page: 3
     lock_rec: 2
    lock_data: 0x000000000200
  *************************** 2. row ***************************
      lock_id: 3872:29:3:2
  lock_trx_id: 3872
    lock_mode: X
    lock_type: RECORD
   lock_table: `tchua`.`t1`
   lock_index: GEN_CLUST_INDEX
   lock_space: 29
    lock_page: 3
     lock_rec: 2
    lock_data: 0x000000000200
  2 rows in set, 1 warning (0.00 sec)
  # 总结：从上面结果看出，锁锁住事务ID为3875，并且锁的模式为独占锁，类型为RECORD即行锁，而且锁定的页数为3页，锁定的行有2行，锁定行的主键值为0x000000000200，这里之所以会有主键值，这是因为MySQL在加锁的时候判断是否有索引，没有索引的时候会自动隐式的添加索引(聚集索引)，从上面锁的索引为"GEN_CLUST_INDEX"也可以看出。
  ####其实，MariaDB/MySQL中的行锁是通过键锁(Key)来实现的####
  ## innodb_lock_waits表结构
  requesting_trx_id: 要申请锁的事务ID
  requested_lock_id: 申请的锁ID
  blocking_trx_id: 阻塞在前方的事务ID
  blocking_lock_id: 阻塞在前方的锁ID
  ## innodb_lock_waits锁信息查看
  mysql> select * from information_schema.INNODB_LOCK_WAITS\G;
  *************************** 1. row ***************************
  requesting_trx_id: 3875
  requested_lock_id: 3875:29:3:2
    blocking_trx_id: 3872
   blocking_lock_id: 3872:29:3:2
  
  # 总结：申请锁的事务ID为14914，阻塞在前方的事务ID为14913。
  
  有了这3张表，还可以将它们联接起来更直观的显示想要的结果。
  # 多表连接查看所信息
  ## 1) 
  mysql> SELECT
           r.trx_id AS waiting_trx_id,
           r.trx_mysql_thread_id AS waiting_thread,
           r.trx_query AS waiting_query,
           b.trx_id AS blocking_trx_id,
           b.trx_mysql_thread_id AS blocking_thread,
           b.trx_query AS blocking_query
       FROM
           information_schema.innodb_lock_waits w
      JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
      JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id\G;
  *************************** 1. row ***************************
   waiting_trx_id: 3876
   waiting_thread: 12
    waiting_query: update t1 set b = 'a12' where a = 1
  blocking_trx_id: 3872
  blocking_thread: 11
   blocking_query: NULL
  1 row in set, 1 warning (0.00 sec)
  
  # ## 总结：可以只管的看到事务3876被事务3872阻塞，被阻塞的语句为update。
  ## 2) 锁和事务关联信息
  mysql> SELECT
           trx_id,
           trx_state,
           lock_id,
           lock_mode,
           lock_type,
           lock_table,
           trx_mysql_thread_id,
           trx_query
       FROM
           information_schema.innodb_trx t
       JOIN information_schema.innodb_locks l ON l.lock_trx_id = t.trx_id;
  +--------+-----------+-------------+-----------+-----------+--------------+---------------------+-------------------------------------+
  | trx_id | trx_state | lock_id     | lock_mode | lock_type | lock_table   | trx_mysql_thread_id | trx_query                           |
  +--------+-----------+-------------+-----------+-----------+--------------+---------------------+-------------------------------------+
  | 3876   | LOCK WAIT | 3876:29:3:2 | X         | RECORD    | `tchua`.`t1` |                  12 | update t1 set b = 'a12' where a = 1 |
  | 3872   | RUNNING   | 3872:29:3:2 | X         | RECORD    | `tchua`.`t1` |                  11 | NULL                                |
  +--------+-----------+-------------+-----------+-----------+--------------+---------------------+-------------------------------------+
  
  ~~~

#### 3.2 innodb表的外键和锁

#### 3.3 innodb锁算法

~~~shell
innodb支持行级锁，但是它还支持范围锁。即对范围内的行记录加行锁。
~~~

三种锁算法：

* record lock：即行锁

* gap lock：范围锁，但是不锁定行记录本身。

    当我们用范围条件而不是相等条件检索数据，并请求共享或排他锁时，InnoDB会给符合条件的已有数据记录的索引项加锁；对于键值在条件范围内但并不存在的记录，叫做“间隙（GAP)”，InnoDB也会对这个"间隙"加锁，这种锁机制就是所谓的间隙锁（Next-Key锁）。

* next-key lock：范围锁加行锁，范围锁并锁定记录本身。

~~~shell
  record lock是行锁，但是它的行锁锁定的是key，即基于唯一性索引键列来锁定(SQL Server还有基于堆表的rid类型行锁)，不是针对记录加的锁。如果没有唯一性索引键列，则会自动在隐式列上创建索引并完成锁定。
  next-key lock是行锁和范围锁的结合，innodb对行的锁申请默认都是这种算法。如果有索引，则只锁定指定范围内的索引键值，如果没有索引，则自动创建索引并对整个表进行范围锁定。之所以锁定了表还称为范围锁定，是因为它实际上锁的不是表，而是把所有可能的区间都锁定了，从主键值的负无穷到正无穷的所有区间都锁定，等价于锁定了表。
~~~

**范围锁事例：**

~~~shell
# 1) 有索引
## 创建一个有索引的t表，插入几行数据
mysql> create table t(id int);
mysql> create unique index idx_t on t(id);
mysql> insert into t values(1),(2),(3),(4),(7),(8),(12),(15);

# 会话1
mysql> begin;
mysql> select * from t where id < 5 lock in share mode;  
+------+
| id   |
+------+
|    1 |
|    2 |
|    3 |
|    4 |
+------+
# NOTE：lock in share mode它的作用是在读取的时候加上共享锁并且不释放
# 会话2
mysql> insert into t values(9); -- 可以插入
mysql> insert into t values(6); -- 锁等待超时
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

# NOTE：第一个会正常插入，第二个会被阻塞。
# 会话3 此时查看锁现象
mysql> show engine innodb status\G;
------------
TRANSACTIONS
------------
Trx id counter 4378
Purge done for trx's n:o < 4375 undo n:o < 0 state: running but idle
History list length 19
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421355163553392, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 4377, ACTIVE 4 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1
MySQL thread id 3, OS thread handle 139879908554496, query id 27 localhost root update
insert into t values(6)
------- TRX HAS BEEN WAITING 4 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 32 page no 4 n bits 80 index idx_t of table `tchua`.`t` trx id 4377 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 6 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000007; asc ;;
 1: len 6; hex 000000000304; asc ;;
 
 ## 总结: 'lock_mode X locks gap' 就表示阻塞insert语句的锁是gap锁，即范围锁。锁定的范围：(-∞,4],(4,7],为什么会有4,7这个区间？是因为锁到操作行(待插入行)的下一个key，此处插入id=6，由于存在id=7的key，所以锁到7为止，这就是next-key的意思。
 
 # 查看锁定范围
 mysql> select * from information_schema.INNODB_LOCKS\G;
*************************** 1. row ***************************
    lock_id: 4378:32:4:6
lock_trx_id: 4378
  lock_mode: X,GAP
  lock_type: RECORD
 lock_table: `tchua`.`t`
 lock_index: idx_t
 lock_space: 32
  lock_page: 4
   lock_rec: 6
  lock_data: 7
*************************** 2. row ***************************
    lock_id: 421355163550656:32:4:6
lock_trx_id: 421355163550656
  lock_mode: S
  lock_type: RECORD
 lock_table: `tchua`.`t`
 lock_index: idx_t
 lock_space: 32
  lock_page: 4
   lock_rec: 6
  lock_data: 7
2 rows in set, 1 warning (0.01 sec)

# 总结：'lock_mode'为"X+GAP" 表示next-key lock算法，lock_rec：表示锁定了6行(1,2,3,4,7待插入锁定的id=6，当查询条件值不存是时，其实就是，5与7之前的行也会被锁定)
lock_data: 表示锁定了值为7的记录(这是最大锁定范围边界值)3.4 锁等待
~~~

**总结：** InnoDB使用间隙锁的目的，一方面是为了防止幻读，以满足相关隔离级别的要求；另外一方面，是为了满足其恢复和复制的需要。

#### 3.4  锁等待

~~~shell
  在innodb存储引擎中，当出现锁等待时，如果等待超时，将会结束事务，超时时长通过动态变量innodb_lock_wait_timeout值来决定，默认是等待50秒。关于锁等待超时，可以直接在语句中设置超时时间。可以设置锁等待超时时间的语句包括：wait n的n单位为秒，nowait表示永不超时。
   超时后结束事务的方式有中断性结束和回滚性结束两种方式，这也是通过变量来控制的，该变量为innodb_rollback_on_timeout，默认为off，即超时后不回滚，也即中断性结束。
mysql> show variables like "innodb%timeout";
+-----------------------------+-------+
| Variable_name               | Value |
+-----------------------------+-------+
| innodb_flush_log_at_timeout | 1     |
| innodb_lock_wait_timeout    | 50    |
| innodb_rollback_on_timeout  | OFF   |
+-----------------------------+-------+
~~~

### 四、锁分析

#### 4.1行锁定分析

~~~mysql
mysql> show status like 'innodb_row_lock%';
+-------------------------------+--------+
| Variable_name                 | Value  |
+-------------------------------+--------+
| Innodb_row_lock_current_waits | 0      |
| Innodb_row_lock_time          | 863772 |
| Innodb_row_lock_time_avg      | 26174  |
| Innodb_row_lock_time_max      | 51052  |
| Innodb_row_lock_waits         | 33     |
+-------------------------------+--------+

参数解释：
innodb_row_lock_current_waits: 当前正在等待锁定的数量
innodb_row_lock_time: 从系统启动到现在锁定总时间长度；非常重要的参数，
innodb_row_lock_time_avg: 每次等待所花平均时间；非常重要的参数，
innodb_row_lock_time_max: 从系统启动到现在等待最长的一次所花的时间；
innodb_row_lock_waits: 系统启动后到现在总共等待的次数；非常重要的参数。直接决定优化的方向和策略。

~~~

**优化思路**

~~~mysql
1、尽可能让所有数据检索都通过索引来完成，避免无索引行或索引失效导致行锁升级为表锁。
2、尽可能避免间隙锁带来的性能下降，减少或使用合理的检索范围。
3、尽可能减少事务的粒度，比如控制事务大小，而从减少锁定资源量和时间长度，从而减少锁的竞争等，提供性能。
4、尽可能低级别事务隔离，隔离级别越高，并发的处理能力越低。
~~~

#### 4.2 表锁定分析

~~~shell
mysql> show status like 'table_locks%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Table_locks_immediate      | 104   |
| Table_locks_waited         | 0     |
+----------------------------+-------+
table_locks_immediate: 表示立即释放表锁数。
table_locks_waited: 表示需要等待的表锁数。此值越高则说明存在着越严重的表级锁争用情况。
~~~

**优化思路**

~~~shell
show open tables where in_use; 1表示加锁，0表示未加锁。
~~~

**表锁使用场景**

~~~shell
1) 全表更新。事务需要更新大部分或全部数据，且表又比较大。若使用行锁，会导致事务执行效率低，从而可能造成其他事务长时间锁等待和更多的锁冲突。
2) 多表查询。事务涉及多个表，比较复杂的关联查询，很可能引起死锁，造成大量事务回滚。这种情况若能一次性锁定事务涉及的表，从而可以避免死锁、减少数据库因事务回滚带来的开销

~~~

### 总结

~~~shell
1) InnoDB 支持表锁和行锁，使用索引作为检索条件修改数据时采用行锁，否则采用表锁。
2) InnoDB 自动给修改操作加锁，给查询操作不自动加锁
3) 行锁可能因为未使用索引而升级为表锁，所以除了检查索引是否创建的同时，也需要通过explain执行计划查询索引是否被实际使用。
4) 行锁相对于表锁来说，优势在于高并发场景下表现更突出，毕竟锁的粒度小。
5) 当表的大部分数据需要被修改，或者是多表复杂关联查询时，建议使用表锁优于行锁。
6) 为了保证数据的一致完整性，任何一个数据库都存在锁定机制。锁定机制的优劣直接影响到一个数据库的并发处理能力和性能。

~~~

### 五、补充

#### 5.1 死锁

~~~shell
死锁（Deadlock）
  所谓死锁：是指两个或两个以上的进程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。由于资源占用是互斥的，当某个进程提出申请资源后，使得有关进程在无外力协助下，永远分配不到必需的资源而无法继续运行，这就产生了一种特殊现象死锁。
~~~

**怎样降低死锁**

~~~shell
1) 尽量使用较低的隔离级别，比如如果发生了间隙锁，你可以把会话或者事务的事务隔离级别更改为 RC(read committed)级别来避免，但此时需要把 binlog_format 设置成 row 或者 mixed 格式
2) 精心设计索引，并尽量使用索引访问数据，使加锁更精确，从而减少锁冲突的机会；
3) 选择合理的事务大小，小事务发生锁冲突的几率也更小；
4) 给记录集显示加锁时，最好一次性请求足够级别的锁。比如要修改数据的话，最好直接申请排他锁，而不是先申请共享锁，修改时再请求排他锁，这样容易产生死锁；
5) 不同的程序访问一组表时，应尽量约定以相同的顺序访问各表，对一个表而言，尽可能以固定的顺序存取表中的行。这样可以大大减少死锁的机会；

~~~

#### 5.2 乐观锁

~~~shell
  用数据版本（Version）记录机制实现，这是乐观锁最常用的一种实现方式。何谓数据版本？即为数据增加一个版本标识，一般是通过为数据库表增加一个数字类型的 “version” 字段来实现。当读取数据时，将version字段的值一同读出，数据每更新一次，对此version值加1。当我们提交更新的时候，判断数据库表对应记录的当前版本信息与第一次取出来的version值进行比对，如果数据库表当前版本号与第一次取出来的version值相等，则予以更新，否则认为是过期数据。

~~~





## 参考文档：

https://juejin.im/post/5b82e0196fb9a019f47d1823

https://www.cnblogs.com/f-ck-need-u/p/8995475.html#auto_id_5