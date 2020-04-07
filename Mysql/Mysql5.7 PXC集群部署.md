## Mysql5.7 PXC集群部署

### 1、规划

| IP           | 主机名 | 角色     |
| ------------ | ------ | -------- |
| 172.16.10.20 | db20   | percona1 |
| 172.16.10.21 | db21   | percona2 |
| 172.16.10.22 | db22   | percona3 |

###  2、初始化机器

~~~shell
# 1) 修改主机名
[root@localhost ~]# hostnamectl set-hostname db20
[root@localhost ~]# hostnamectl set-hostname db21
[root@localhost ~]# hostnamectl set-hostname db22
~~~



### 3、安装PXC

> 所有机器firewall，selinux关闭

~~~shell
官方文档：https://www.percona.com/doc/percona-xtradb-cluster/5.7/howtos/centos_howto.html
# 集群需要使用端口
3306 : 数据库对外服务的端口号
4444 : 请求SST SST: 指数据一个镜象传输 xtrabackup , rsync ,mysqldump
4567 : 组成员之间进行沟通的一个端口号
4568 : 传输IST用的。相对于SST来说的一个增量
## NOTE：4567，4568端口，IST只是在节点下线，重启加入那一个时间有用,4444端口，只会在新节点加入进来时起作用
~~~

~~~shell
# 1) 配置yum源
## 三台机器都配置yum源
[root@db20 ~]# yum install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
[root@db20 ~]# percona-release setup ps57   ## 设置存储库
# 2) 安装
[root@db20 ~]# yum install Percona-XtraDB-Cluster-57 -y


~~~

### 4、配置PXC

~~~shell
# 1) 选择一个节点为逻辑上的主节点，这里以db20为主节点。
## 配置文件位置：
/etc/my.cnf 	# 配置文件
/etc/percona-xtradb-cluster.conf.d/ # 有三个子配置文件
## NOTE：官方使用的是/etc/my.cnf文件，这里我们/etc/percona-xtradb-cluster.conf.d/目录下的配置文件信息都整合到/etc/my.cnf下面。
[root@db20 ~]# vim /etc/my.cnf
[client]
socket=/var/lib/mysql/mysql.sock

[mysqld]
user=mysql 
server-id=20
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
log-bin
log_slave_updates
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
lower_case_table_names=1
log_timestamps=system

# pxc相关
wsrep_provider=/usr/lib64/libgalera_smm.so
wsrep_cluster_address=gcomm://172.16.10.20,172.16.10.21,172.16.10.22
wsrep_node_address=172.16.10.20
wsrep_sst_method=xtrabackup-v2
wsrep_cluster_name=tchua_db_pxc
wsrep_sst_auth="sst:123456"
参数解释：
1、innodb_autoinc_lock_mode	
2、wsrep_provider
3、wsrep_cluster_address    # 集群中所有节点ip
4、wsrep_node_address	# 当前节点地址
5、wsrep_sst_method		# 同步方法(mysqldump、rsync、xtrabackup)
6、wsrep_cluster_name	# 集群名称
7、wsrep_sst_auth		# 同步数据的用户名密码
# 2) 启动db20
[root@db20 ~]# systemctl start mysql@bootstrap.service
# NOTE：The previous command will start the cluster with initial wsrep_cluster_address variable set to gcomm://. If the node or MySQL are restarted later, there will be no need to change the configuration file.
# 3) 查看初始密码
[root@db20 ~]# cat /var/log/mysqld.log |grep pass
2020-03-26T02:20:19.421657Z 1 [Note] A temporary password is generated for root@localhost: iDsPIs-1lr&Z
## 修改密码，初次登陆成功后需要修改密码
[root@db20 ~]# mysql -uroot -p
Enter password: 
mysql> alter user 'root'@'localhost' identified by '123456';
mysql> flush privileges;
# 4） 查看目前集群状态
mysql> show status like 'wsrep%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | 5bd013c0-6f08-11ea-bbb1-6e74542ce507	 |
...
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
...
| wsrep_cluster_size         | 1                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
...
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+
# 5）创建SST用户
## NOTE： 账号、密码，要和配置文件的wsrep_sst_auth=sst:123456对应
mysql> CREATE USER 'sst'@'localhost' IDENTIFIED BY '123456';
mysql> GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO 'sst'@'localhost';
mysql> FLUSH PRIVILEGES;
~~~

配置db21节点

~~~shell
[root@db21 ~]# vim /etc/my.cnf
[client]
socket=/var/lib/mysql/mysql.sock

[mysqld]
user=mysql
server-id=21
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
log-bin
log_slave_updates
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
lower_case_table_names=1
log_timestamps=system

# pxc相关
wsrep_provider=/usr/lib64/libgalera_smm.so
wsrep_cluster_address=gcomm://172.16.10.20,172.16.10.21,172.16.10.22
wsrep_node_address=172.16.10.21
wsrep_sst_method=xtrabackup-v2
wsrep_cluster_name=tchua_db_pxc
wsrep_sst_auth="sst:123456"
# 2) 启动(注意启动进程mysqld)
[root@db21 ~]# systemctl start mysqld

## NOTE：第二个节点启动时会自动接收SST，数据会自动同步db20数据过来，直接使用db20机器的mysql密码即可登录。
# 3) 查看目前集群状态
[root@db21 mysql]# mysql -uroot -p123456
mysql> show status like 'wsrep%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | 5bd013c0-6f08-11ea-bbb1-6e74542ce507 |
...
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
...
| wsrep_cluster_size         | 2                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
...
| wsrep_incoming_addresses   | 172.16.10.20:3306,172.16.10.21:3306  |
...
| wsrep_ready                | ON                                   |
...

~~~

配置db22

~~~shell
[root@db22 ~]# vim /etc/my.cnf
[client]
socket=/var/lib/mysql/mysql.sock

[mysqld]
user=mysql
server-id=22
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
log-bin
log_slave_updates
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
lower_case_table_names=1
log_timestamps=system

# pxc相关
wsrep_provider=/usr/lib64/libgalera_smm.so
wsrep_cluster_address=gcomm://172.16.10.20,172.16.10.21,172.16.10.22
wsrep_node_address=172.16.10.22
wsrep_sst_method=xtrabackup-v2
wsrep_cluster_name=tchua_db_pxc
wsrep_sst_auth="sst:123456"

# 2) 启动
[root@db22 ~]# systemctl start mysqld

# 3) 查看集群
[root@db22 mysql]# mysql -uroot -p123456
mysql> show status like 'wsrep%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | 5bd013c0-6f08-11ea-bbb1-6e74542ce507 |
...
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
...
| wsrep_cluster_size         | 3                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
...
| wsrep_incoming_addresses   | 172.16.10.22:3306,172.16.10.20:3306,172.16.10.21:3306  |
...
| wsrep_ready                | ON                                   |
...
~~~

### 5、集群检查

> 任何一台db节点操作即可

~~~shell
# 查看集群开启状态
mysql> show status like 'wsrep_ready';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| wsrep_ready   | ON    |
+---------------+-------+

# 集群成员的IP地址列表
mysql> SHOW VARIABLES LIKE 'wsrep_cluster_address';
+-----------------------+------------------------------------------------+
| Variable_name         | Value                                          |
+-----------------------+------------------------------------------------+
| wsrep_cluster_address | gcomm://172.16.10.20,172.16.10.21,172.16.10.22 |
+-----------------------+------------------------------------------------+
# 查看集群的成员数
mysql> show status like 'wsrep_cluster_size';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+

~~~

### 6、测试

~~~shell
# 1) db22 创建数据库tchua_db
[root@db22 ~]# mysql -uroot -p123456
mysql> create database tchua_db charset utf8mb4;
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| tchua_db           |
+--------------------+
## db20、21查看同步状态
[root@db20 ~]# mysql -uroot -p123456 -e "show databases;"
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| tchua_db           |
+--------------------+
[root@db21 ~]# mysql -uroot -p123456 -e "show databases;"
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| tchua_db           |
+--------------------+
# 2) db21上在tchua_db库创建表
## NOTE: pxc特性，每个表都需要有主键，否则数据无法插入
mysql> use tchua_db;
mysql> create table t1(id int(5) primary key);
## db20、22查看表同步状态
[root@db20 ~]# mysql -uroot -p123456 -e "use tchua_db;show tables;"
+--------------------+
| Tables_in_tchua_db |
+--------------------+
| t1                 |
+--------------------+
[root@db22 ~]# mysql -uroot -p123456 -e "use tchua_db;show tables;"
+--------------------+
| Tables_in_tchua_db |
+--------------------+
| t1                 |
+--------------------+
# 3) db20插入数据
mysql> use tchua_db
mysql> insert into t1 values(1),(2),(3),(4),(5);
## db21、db22数据验证
[root@db21 ~]# mysql -uroot -p123456 -e "select * from tchua_db.t1;"
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
|  4 |
|  5 |
+----+
[root@db22 ~]# mysql -uroot -p123456 -e "select * from tchua_db.t1;"
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
|  4 |
|  5 |
+----+
~~~

