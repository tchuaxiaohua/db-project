## Mysql 5.7 基于组复制(MGR)多主模式

###  1、 服务器规划

| IP           | 主机名 | 角色 |
| ------------ | ------ | ---- |
| 172.16.10.20 | db20   |      |
| 172.16.10.21 | db21   |      |
| 172.16.10.22 | db22   |      |

###  2、环境初始化

~~~shell
# 1) 主机名修改
[root@localhost ~]# hostnamectl set-hostname db20
[root@localhost ~]# hostnamectl set-hostname db21
[root@localhost ~]# hostnamectl set-hostname db22
# 2) 关闭防火墙
[root@db22 ~]# systemctl disable firewalld
[root@db22 ~]# systemctl stop firewalld
[root@db21 ~]# systemctl disable firewalld
[root@db21 ~]# systemctl stop firewalld
[root@db20 ~]# systemctl disable firewalld
[root@db20 ~]# systemctl stop firewalld
# 3) selinux
[root@db20 ~]# sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config 
[root@db21 ~]# sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config 
[root@db22 ~]# sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config 
# 4) 配置host解析
[root@db20 ~]# vim /etc/hosts
172.16.10.20 db20
172.16.10.21 db21
172.16.10.22 db22
[root@db21 ~]# vim /etc/hosts
172.16.10.20 db20
172.16.10.21 db21
172.16.10.22 db22
[root@db22 ~]# vim /etc/hosts
172.16.10.20 db20
172.16.10.21 db21
172.16.10.22 db22

~~~

###  3、mysql环境搭建

> mysql搭建三台都需要操作，由于步骤都一样，这里以一台部署为例

~~~shell
# mysql下载地址：https://downloads.mysql.com/archives/community/
# 1) 下载
[root@db20 ~]# wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.29-linux-glibc2.12-x86_64.tar.gz
# 2) 创建目录
[root@db20 ~]# mkdir /opt/mysql/{data,conf,logs} -p
# 3) 解压重命名
[root@db20 ~]# tar -xf mysql-5.7.29-linux-glibc2.12-x86_64.tar.gz
[root@db20 ~]# mv mysql-5.7.29-linux-glibc2.12-x86_64 /opt/mysql5.7
# 4) 创建mysql用户并授权
[root@db20 ~]# useradd mysql
[root@db20 ~]# chown -R mysql.mysql /opt/mysql
# 5) 初始化数据库,初始密码会打印出来
[root@db20 ~]# cd /opt/mysql5.7/bin/
[root@db20 bin]# ./mysqld --initialize --user=mysql --basedir=/opt/mysql5.7 --datadir=/opt/mysql/data
2020-03-30T09:08:09.744308Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2020-03-30T09:08:09.985530Z 0 [Warning] InnoDB: New log files created, LSN=45790
2020-03-30T09:08:10.020365Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
2020-03-30T09:08:10.092963Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: fa1167f9-7265-11ea-ac36-000c29121201.
2020-03-30T09:08:10.093790Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
2020-03-30T09:08:10.534966Z 0 [Warning] CA certificate ca.pem is self signed.
2020-03-30T09:08:10.743145Z 1 [Note] A temporary password is generated for root@localhost: O<m,TUW%*95(
# 6) 编辑默认配置文件
[root@db20 bin]# vim /opt/mysql/conf/my.cnf
[mysqld]
user=mysql
basedir=/opt/mysql5.7
datadir=/opt/mysql/data
server_id=20
port=3306
socket=/tmp/mysql.sock
[mysql]
socket=/tmp/mysql.sock
[root@db20 bin]# chown -R mysql.mysql /opt/mysql/conf/my.cnf 
# 7) 编辑启动脚本
[root@db20 ~]# vim /etc/systemd/system/mysqld.service
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target
[Install]
WantedBy=multi-user.target
[Service]
User=mysql
Group=mysql
ExecStart=/opt/mysql5.7/bin/mysqld --defaults-file=/opt/mysql/conf/my.cnf
LimitNOFILE = 5000
[root@db20 ~]# systemctl enable mysqld
Created symlink from /etc/systemd/system/multi-user.target.wants/mysqld.service to /etc/systemd/system/mysqld.service.
[root@db20 ~]# systemctl start mysqld
[root@db20 ~]# netstat -nlpt|grep 3306
tcp6       0      0 :::3306                 :::*                    LISTEN      1711/mysqld  
# 8) 初始化密码
[root@db20 ~]# /opt/mysql5.7/bin/mysqladmin -uroot -p password 123456
Enter password: 
mysqladmin: [Warning] Using a password on the command line interface can be insecure.
Warning: Since password will be sent to server in plain text, use ssl connection to ensure password safety.
# 9) 添加环境变量
[root@db20 ~]# vim /etc/profile
# set mysql
export PATH=/opt/mysql5.7/bin:$PATH
[root@db20 ~]# source /etc/profile
[root@db20 ~]# mysql -uroot -p123456
~~~

### 4、MGR组复制配置

* 第一个节点配置 （db20）

  ~~~mysql
  # 1) 配置文件
  [root@db20 ~]# vim /opt/mysql/conf/my.cnf 
  [mysqld]
  port=3306
  user=mysql
  basedir=/opt/mysql5.7
  datadir=/opt/mysql/data
  socket=/tmp/mysql.sock
  server_id=20
  log_bin=binlog
  log_slave_updates=ON
  binlog_format=ROW
  master_info_repository=TABLE
  relay_log_info_repository=TABLE
  gtid_mode=on
  enforce_gtid_consistency = on
  binlog_checksum=NONE
  # 组复制参数
  skip_slave_start = 1
  transaction_write_set_extraction=XXHASH64
  loose-group_replication_group_name="d2777eab-7a12-11ea-9450-000c29121201"
  loose-group_replication_start_on_boot=OFF
  loose-group_replication_local_address= "172.16.10.20:33061"
  loose-group_replication_group_seeds= "172.16.10.20:33061,172.16.10.21:33061,172.16.10.22:33061"
  loose-group_replication_bootstrap_group=OF
  loose-group_replication_single_primary_mode = off
  loose-group_replication_enforce_update_everywhere_checks=on
  loose-group_replication_ip_whitelist="172.16.10.0/24,127.0.0.1/8"
  [mysql]
  socket=/tmp/mysql.sock
  # 2) 启动mysql
  [root@db20 ~]# systemctl restart mysqld
  
  ~~~

  ############################

  *** 参数解释 ***

  ~~~shell
  1、transaction_write_set_extraction ： 在server收集写集合的同时将其记录到二进制日志。写集合基于每行的主键，并且是行更改后的唯一标识此标识将用于检测冲突
  2、loose-group_replication_group_name ：组的名字可以随便起,但不能用主机的GTID! 所有节点的这个组名必须保持一致。
  3、loose-group_replication_start_on_boot ：为了避免每次启动自动引导具有相同名称的第二个组,所以设置为OFF
  4、loose-group_replication_enforce_update_everywhere_checks ：开启多主模式的参数
  5、loose-group_replication_ip_whitelist ：允许加入组复制的客户机来源的ip白名单
  ~~~

  ############################

* 第二个节点配置（db21）

  ~~~shell
  # 1) 配置文件
  [root@db21 ~]# vim /opt/mysql/conf/my.cnf 
  [mysqld]
  port=3306
  user=mysql
  basedir=/opt/mysql5.7
  datadir=/opt/mysql/data
  socket=/tmp/mysql.sock
  server_id=21
  log_bin=binlog
  log_slave_updates=ON
  binlog_format=ROW
  master_info_repository=TABLE
  relay_log_info_repository=TABLE
  gtid_mode=on
  enforce_gtid_consistency = on
  binlog_checksum=NONE
  # 组复制参数
  skip_slave_start = 1
  transaction_write_set_extraction=XXHASH64
  loose-group_replication_group_name="d2777eab-7a12-11ea-9450-000c29121201"
  loose-group_replication_start_on_boot=OFF
  loose-group_replication_local_address= "172.16.10.21:33061"
  loose-group_replication_group_seeds= "172.16.10.20:33061,172.16.10.21:33061,172.16.10.22:33061"
  loose-group_replication_bootstrap_group=OF
  loose-group_replication_single_primary_mode = off
  loose-group_replication_enforce_update_everywhere_checks=on
  loose-group_replication_ip_whitelist="172.16.10.0/24,127.0.0.1/8"
  [mysql]
  socket=/tmp/mysql.sock
  # 2) 启动数据库
  [root@db21 ~]# systemctl restart mysqld
  
  ~~~

* 第三个节点配置（db22）

~~~shell
# 1) 配置文件
[root@db22 ~]# vim /opt/mysql/conf/my.cnf 
[mysqld]
port=3306
user=mysql
basedir=/opt/mysql5.7
datadir=/opt/mysql/data
socket=/tmp/mysql.sock
server_id=22
log_bin=binlog
log_slave_updates=ON
binlog_format=ROW
master_info_repository=TABLE
relay_log_info_repository=TABLE
gtid_mode=on
enforce_gtid_consistency = on
binlog_checksum=NONE
# 组复制参数
skip_slave_start = 1
transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="d2777eab-7a12-11ea-9450-000c29121201"
loose-group_replication_start_on_boot=OFF
loose-group_replication_local_address= "172.16.10.22:33061"
loose-group_replication_group_seeds= "172.16.10.20:33061,172.16.10.21:33061,172.16.10.22:33061"
loose-group_replication_bootstrap_group=OF
loose-group_replication_single_primary_mode = off
loose-group_replication_enforce_update_everywhere_checks=on
loose-group_replication_ip_whitelist="172.16.10.0/24,127.0.0.1/8"
[mysql]
socket=/tmp/mysql.sock
# 2) 启动数据库
[root@db22 ~]# systemctl restart mysqld
~~~

### 5、组复制初始化

> 以下操作需要在所有节点操作

~~~shell
# 1) db20
[root@db20 ~]# mysql -uroot -p123456
# 安装MGR插件
mysql> INSTALL PLUGIN group_replication SONAME 'group_replication.so';
# 设置复制账号
mysql> SET SQL_LOG_BIN=0;  # 临时关闭二进制记录
mysql> GRANT REPLICATION SLAVE ON *.* TO rpl@'%' IDENTIFIED BY '123456';
mysql> FLUSH PRIVILEGES;
mysql> reset master;
mysql> SET SQL_LOG_BIN=1;
mysql>  CHANGE MASTER TO MASTER_USER='rpl', MASTER_PASSWORD='123456' FOR CHANNEL 'group_replication_recovery';
# 2) db21
[root@db21 ~]# mysql -uroot -p123456
# 安装MGR插件
mysql> INSTALL PLUGIN group_replication SONAME 'group_replication.so';
# 设置复制账号
mysql> SET SQL_LOG_BIN=0;  # 临时关闭二进制记录
mysql> GRANT REPLICATION SLAVE ON *.* TO rpl@'%' IDENTIFIED BY '123456';
mysql> FLUSH PRIVILEGES;
mysql> reset master;
mysql> SET SQL_LOG_BIN=1;
mysql>  CHANGE MASTER TO MASTER_USER='rpl', MASTER_PASSWORD='123456' FOR CHANNEL 'group_replication_recovery';
# 3) db22
[root@db22 ~]# mysql -uroot -p123456
# 安装MGR插件
mysql> INSTALL PLUGIN group_replication SONAME 'group_replication.so';
# 设置复制账号
mysql> SET SQL_LOG_BIN=0;  # 临时关闭二进制记录
mysql> GRANT REPLICATION SLAVE ON *.* TO rpl@'%' IDENTIFIED BY '123456';
mysql> FLUSH PRIVILEGES;
mysql> reset master;
mysql> SET SQL_LOG_BIN=1;
mysql>  CHANGE MASTER TO MASTER_USER='rpl', MASTER_PASSWORD='123456' FOR CHANNEL 'group_replication_recovery';
~~~

### 6、启动MRG

~~~shell
# 1) 只需要在第一个节点执行(db20)
mysql> SET GLOBAL group_replication_bootstrap_group=ON;
mysql> START GROUP_REPLICATION;
mysql> SET GLOBAL group_replication_bootstrap_group=OFF;

# 2) 查看组状态
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | b3f13879-7a01-11ea-8714-000c292cbcf6 | db20        |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
# 3) db21 db22开启复制
[root@db21 ~]# mysql -uroot -p123456
mysql> START GROUP_REPLICATION;
Query OK, 0 rows affected (6.88 sec)
[root@db22 data]# mysql -uroot -p123456
mysql> START GROUP_REPLICATION;
Query OK, 0 rows affected (3.12 sec)
# 4) 再次查看组复制状态
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | b3f13879-7a01-11ea-8714-000c292cbcf6 | db20        |        3306 | ONLINE       |
| group_replication_applier | bd62d4c1-7a0e-11ea-b1eb-000c29d9208b | db21        |        3306 | ONLINE       |
| group_replication_applier | c9ea3ec2-7a0e-11ea-b411-000c29a5ec58 | db22        |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+

~~~

![image-20200409132548299](C:\Users\86155\AppData\Roaming\Typora\typora-user-images\image-20200409132548299.png)

可以看到三个节点都已经加入到组中，注意节点状态必须是ONLINE才算是加入完成。

### 7 、组复制测试

~~~shell
# 1) db20创建数据库
[root@db20 ~]# mysql -uroot -p123456
mysql> create database mgr_db charset utf8mb4;
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mgr_db             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
# 2) 查看db21 db22库的同步状态
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mgr_db             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
# 3) db21创建表插入数据
mysql> use mgr_db
mysql> create table t1(id int(10) PRIMARY KEY AUTO_INCREMENT,name varchar(50) NOT NULL);
mysql> insert into t1 values(1,'zs'),(2,'ls'),(3,'ww');
mysql> select * from t1;
+----+------+
| id | name |
+----+------+
|  1 | zs   |
|  2 | ls   |
|  3 | ww   |
+----+------+
# 4) db20 21查看同步状态
mysql> use mgr_db
mysql> select * from t1;
+----+------+
| id | name |
+----+------+
|  1 | zs   |
|  2 | ls   |
|  3 | ww   |
+----+------+
~~~

### 8、组节点故障测试

> 当组内的某个节点发生故障时，会自动从将该节点从组内踢出，与其他节点隔离。剩余的节点之间保持主从复制的正常同步关系。当该节点的故障恢复后，只需手动激活组复制即可(即执行"START GROUP_REPLICATION;");

~~~shell
# 1) 关闭db20节点mysql模拟故障
[root@db20 ~]# systemctl stop mysqld
# 2) db21查看组成员状态
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | bd62d4c1-7a0e-11ea-b1eb-000c29d9208b | db21        |        3306 | ONLINE       |
| group_replication_applier | c9ea3ec2-7a0e-11ea-b411-000c29a5ec58 | db22        |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
# NOTE:如上，节点故障后，会自动从组内剔除
# 3) db21上更新数据
mysql> update t1 set id=11 where name='ls';
mysql> select * from t1;
+----+------+
| id | name |
+----+------+
|  1 | zs   |
|  3 | ww   |
| 11 | ls   |
+----+------+
# 4) db22查看同步
mysql> use mgr_db
mysql> select * from t1;
+----+------+
| id | name |
+----+------+
|  1 | zs   |
|  3 | ww   |
| 11 | ls   |
+----+------+
# NOTE: 数据可以正常同步
# 5) 故障恢复
[root@db20 ~]# systemctl start mysqld
[root@db20 ~]# mysql -urpoot -p123456
mysql> START GROUP_REPLICATION;
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | b3f13879-7a01-11ea-8714-000c292cbcf6 | db20        |        3306 | ONLINE       |
| group_replication_applier | bd62d4c1-7a0e-11ea-b1eb-000c29d9208b | db21        |        3306 | ONLINE       |
| group_replication_applier | c9ea3ec2-7a0e-11ea-b411-000c29a5ec58 | db22        |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
mysql> use mgr_db
mysql> select * from t1;
+----+------+
| id | name |
+----+------+
|  1 | zs   |
|  3 | ww   |
| 11 | ls   |
+----+------+
# NOTE: db20节点恢复后，并重新添加到组内后，其他节点更新的数据也会及时同步过来！

~~~

==============================

NOTE: 若是三个节点都故障，节点恢复后，需要手动重新做组复制

db20：

~~~mysql
mysql> reset master;
mysql> SET SQL_LOG_BIN=1;
mysql> CHANGE MASTER TO MASTER_USER='rpl', MASTER_PASSWORD='123456' FOR CHANNEL 'group_replication_recovery';
mysql> STOP GROUP_REPLICATION;
mysql> SET GLOBAL group_replication_bootstrap_group=ON;
mysql> START GROUP_REPLICATION;
mysql> SET GLOBAL group_replication_bootstrap_group=OFF;
mysql> SELECT * FROM performance_schema.replication_group_members;
~~~

db21:

~~~mysql
mysql> reset master;
mysql> SET SQL_LOG_BIN=1;
mysql> CHANGE MASTER TO MASTER_USER='rpl', MASTER_PASSWORD='123456' FOR CHANNEL 'group_replication_recovery';
mysql> START GROUP_REPLICATION;
mysql> SELECT * FROM performance_schema.replication_group_members;
~~~

db22:

~~~mysql
mysql> reset master;
mysql> SET SQL_LOG_BIN=1;
mysql> CHANGE MASTER TO MASTER_USER='rpl', MASTER_PASSWORD='123456' FOR CHANNEL 'group_replication_recovery';
mysql> START GROUP_REPLICATION;
mysql> SELECT * FROM performance_schema.replication_group_members;
~~~

==============================