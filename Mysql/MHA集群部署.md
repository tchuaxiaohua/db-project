# MHA集群部署

### 1、机器环境

| IP地址       | 主机名 | 角色                     |
| ------------ | ------ | ------------------------ |
| 172.16.10.20 | db20   | 写入，主库               |
| 172.16.10.21 | db21   | 读，从库                 |
| 172.16.10.22 | db22   | 读，从库，manager-server |

### 2、环境初始化

> 注意：防火墙，selinux已提前关闭

~~~shell
# 1) 主机名修改
[root@localhost ~]# hostnamectl set-hostname db20
[root@localhost ~]# hostnamectl set-hostname db21
[root@localhost ~]# hostnamectl set-hostname db22
# 2) 分别在三台节点配置hosts解析，内容都一样，复制就行
[root@db20 ~]# vim /etc/hosts
172.16.10.20 db20
172.16.10.21 db21
172.16.10.23 db22
# 3) ssh免密登录，三台节点都需操作
[root@db20 ~]# ssh-keygen # 一路回车
[root@db20 ~]# ssh-copy-id 172.16.10.20
[root@db20 ~]# ssh-copy-id 172.16.10.21
[root@db20 ~]# ssh-copy-id 172.16.10.22

[root@db21 ~]# ssh-keygen
[root@db21 ~]# ssh-copy-id 172.16.10.20
[root@db21 ~]# ssh-copy-id 172.16.10.21
[root@db21 ~]# ssh-copy-id 172.16.10.22

[root@db22 ~]# ssh-keygen
[root@db22 ~]# ssh-copy-id 172.16.10.20
[root@db22 ~]# ssh-copy-id 172.16.10.21
[root@db22 ~]# ssh-copy-id 172.16.10.22
~~~

###  3、mysql环境搭建

> mysql搭建三台都需要操作，由于步骤都一样，这里以一台部署为例

~~~shell
# mysql下载地址：https://downloads.mysql.com/archives/community/
# 1) 下载
[root@db20 ~]# wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.28-linux-glibc2.12-x86_64.tar.gz
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

### 4、搭建主从环境

~~~shell
# 根据架构，搭建一主二从主从环境，主从基于gtid复制
# 1) 主库配置
[mysqld]
user=mysql
basedir=/opt/mysql5.7
datadir=/opt/mysql/data
server_id=20
log-bin=mysql-bin
log-bin=/opt/mysql/mysqlbinlog/mysql-bin
binlog_format=row
sync_binlog=1
gtid-mode=on
enforce-gtid-consistency=true
port=3306
socket=/tmp/mysql.sock
[mysql]
socket=/tmp/mysql.sock
# 从库21配置
[mysqld]
user=mysql
basedir=/opt/mysql5.7
datadir=/opt/mysql/data
server_id=21
log-bin=/opt/mysql/mysqlbinlog/mysql-bin
binlog_format=row
sync_binlog=1
gtid-mode=on
enforce-gtid-consistency=true
port=3306
socket=/tmp/mysql.sock
[mysql]
socket=/tmp/mysql.sock
# 从库22配置
[mysqld]
user=mysql
basedir=/opt/mysql5.7
datadir=/opt/mysql/data
server_id=22
log-bin=/opt/mysql/mysqlbinlog/mysql-bin
binlog_format=row
sync_binlog=1
gtid-mode=on
enforce-gtid-consistency=true
port=3306
socket=/tmp/mysql.sock
[mysql]
socket=/tmp/mysql.sock
# 2) 创建bin-log目录，三台节点都创建
[root@db20 ~]# mkdir /opt/mysql/mysqlbinlog
[root@db20 ~]# chown -R mysql.mysql /opt/mysql/mysqlbinlog
[root@db20 ~]# systemctl restart mysqld
# 3) 数据同步
[root@db20 ~]# mysqldump -uroot -p123456 -A --master-data=2 --single-transaction -R --triggers > /tmp/all.sql
[root@db20 ~]# scp /tmp/all.sql 172.16.10.21:/tmp/
[root@db20 ~]# scp /tmp/all.sql 172.16.10.22:/tmp/
# 还原
[root@db21 ~]# mysql -uroot -p123456 < /tmp/all.sql
[root@db22 ~]# mysql -uroot -p123456 < /tmp/all.sql
# 4) 创建授权用户(主库操作)
[root@db20 ~]# mysql -uroot -p123456
mysql> grant replication  slave on *.* to rep@'172.16.10.%' identified by "123456";
mysql> flush privileges;
# 5) 构建主从(从库操作)
## 21从库
[root@db21 ~]# mysql -uroot -p123456
mysql> change master to 
master_host='172.16.10.20',
master_user='rep',
master_password='123456',
MASTER_AUTO_POSITION=1;
mysql> start slave;
## 22从库
[root@db22 ~]# mysql -uroot -p123456
mysql> change master to 
master_host='172.16.10.20',
master_user='rep',
master_password='123456',
MASTER_AUTO_POSITION=1;
mysql> start slave;
# 6) 验证主从环境
## 从库执行，IO与SQL线程为Yes即可
mysql> show slave status \G;
           Slave_IO_Running: Yes
           Slave_SQL_Running: Yes
~~~

### 5、部署MHA

~~~shell
# 1) 创建mha管理账号，主从执行即可，账号会自动同步至从库
mysql> grant all privileges on *.* to mha@'172.16.10.%' identified by '123456';
# 2) 下载软件
## mha官网：https://code.google.com/archive/p/mysql-master-ha/
## github下载地址
## https://github.com/yoshinorim/mha4mysql-node/releases
## https://github.com/yoshinorim/mha4mysql-manager/releases

# 3) node组件部署(三个节点都需要部署)
[root@db20 ~]# wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
[root@db21 ~]# wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
[root@db22 ~]# wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
# 4) 安装依赖
[root@db20 ~]# yum -y install perl-DBD-MySQL
# 5) 安装node
[root@db20 ~]# rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
[root@db21 ~]# rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
[root@db22 ~]# rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
## node工具包解释
root@db22 ~]# rpm -ql mha4mysql-node
/usr/bin/apply_diff_relay_logs  # 识别差异的中继日志事件并将其差异的事件应用于其他的
/usr/bin/filter_mysqlbinlog	    
/usr/bin/purge_relay_logs
/usr/bin/save_binary_logs	    # 清除中继日志（不会阻塞SQL线程）
# 6) manager组件部署(manager管理节点)
[root@db22 ~]# wget https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
# 7) 安装依赖
[root@db22 ~]# yum install -y perl-Config-Tiny epel-release perl-Log-Dispatch perl-Parallel-ForkManager perl-Time-HiRes
# 8) 安装manager
[root@db22 ~]# rpm -ivh mha4mysql-manager-0.58-0.el7.centos.noarch.rpm 
## manager工具包
[root@db22 ~]# rpm -ql mha4mysql-manager
/usr/bin/masterha_check_repl   # 检查mysql复制状况
/usr/bin/masterha_check_ssh	   # 检查mha的ssh配置状况
/usr/bin/masterha_check_status # 检查当前mha运行状态
/usr/bin/masterha_conf_host    # 添加或删除配置的server信息
/usr/bin/masterha_manager	   # 启动MHA
/usr/bin/masterha_master_monitor# 检测master是否宕机
/usr/bin/masterha_master_switch # 控制故障转移（自动或者手动）
/usr/bin/masterha_secondary_check # 多种线路检测master是否存活
/usr/bin/masterha_stop			 # 停止MHA
# 9) 准备目录配置文件(manager节点)
[root@db22 ~]# mkdir /etc/mha
[root@db22 ~]# mkdir -p /var/log/masterha/app1
[root@db22 ~]# mkdir -p /var/log/mha/app1
[root@db22 ~]# vim /etc/mha/app1.cnf
[server default]
manager_log=/var/log/mha/app1/manager        // manager的工作目录
manager_workdir=/var/log/mha/app1			// manager的日志
master_binlog_dir=/data/binlog    
user=mha								// 授权mha远程访问数据库
password=123456							//  MHA账户密码
ping_interval=2
repl_user=rep							// mysql复制帐号，用来在主从机之间同步二进制日志等
repl_password=123456					//  mysql复制帐号密码
ssh_user=root							// ssh免密钥登录的帐号名
[server1]
hostname=172.16.10.20
port=3306
master_binlog_dir=/opt/mysql/mysqlbinlog/

[server2]
hostname=172.16.10.21
port=3306
master_binlog_dir=/opt/mysql/mysqlbinlog/

[server3]
hostname=172.16.10.22
port=3306
master_binlog_dir=/opt/mysql/mysqlbinlog/
# 10) 状态检查
## 机器互信检查
[root@db22 ~]# masterha_check_ssh --conf=/etc/mha/app1.cnf 
Tue Mar 31 17:32:35 2020 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Tue Mar 31 17:32:35 2020 - [info] Reading application default configuration from /etc/mha/app1.cnf..
Tue Mar 31 17:32:35 2020 - [info] Reading server configuration from /etc/mha/app1.cnf..
Tue Mar 31 17:32:35 2020 - [info] Starting SSH connection tests..
Tue Mar 31 17:32:38 2020 - [debug] 
Tue Mar 31 17:32:35 2020 - [debug]  Connecting via SSH from root@172.16.10.20(172.16.10.20:22) to root@172.16.10.21(172.16.10.21:22)..
Tue Mar 31 17:32:36 2020 - [debug]   ok.
Tue Mar 31 17:32:36 2020 - [debug]  Connecting via SSH from root@172.16.10.20(172.16.10.20:22) to root@172.16.10.22(172.16.10.22:22)..
Tue Mar 31 17:32:37 2020 - [debug]   ok.
Tue Mar 31 17:32:38 2020 - [debug] 
Tue Mar 31 17:32:36 2020 - [debug]  Connecting via SSH from root@172.16.10.22(172.16.10.22:22) to root@172.16.10.20(172.16.10.20:22)..
Tue Mar 31 17:32:37 2020 - [debug]   ok.
Tue Mar 31 17:32:37 2020 - [debug]  Connecting via SSH from root@172.16.10.22(172.16.10.22:22) to root@172.16.10.21(172.16.10.21:22)..
Tue Mar 31 17:32:38 2020 - [debug]   ok.
Tue Mar 31 17:32:38 2020 - [debug] 
Tue Mar 31 17:32:36 2020 - [debug]  Connecting via SSH from root@172.16.10.21(172.16.10.21:22) to root@172.16.10.20(172.16.10.20:22)..
Tue Mar 31 17:32:37 2020 - [debug]   ok.
Tue Mar 31 17:32:37 2020 - [debug]  Connecting via SSH from root@172.16.10.21(172.16.10.21:22) to root@172.16.10.22(172.16.10.22:22)..
Tue Mar 31 17:32:37 2020 - [debug]   ok.
Tue Mar 31 17:32:38 2020 - [info] All SSH connection tests passed successfully.

都是ok即是正常的
## 主从状态检查
[root@db22 ~]# masterha_check_repl --conf=/etc/mha/app1.cnf
Tue Mar 31 17:40:20 2020 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Tue Mar 31 17:40:20 2020 - [info] Reading application default configuration from /etc/mha/app1.cnf..
Tue Mar 31 17:40:20 2020 - [info] Reading server configuration from /etc/mha/app1.cnf..
Tue Mar 31 17:40:20 2020 - [info] MHA::MasterMonitor version 0.58.
Tue Mar 31 17:40:21 2020 - [info] GTID failover mode = 1
Tue Mar 31 17:40:21 2020 - [info] Dead Servers:
Tue Mar 31 17:40:21 2020 - [info] Alive Servers:
Tue Mar 31 17:40:21 2020 - [info]   172.16.10.20(172.16.10.20:3306)
Tue Mar 31 17:40:21 2020 - [info]   172.16.10.21(172.16.10.21:3306)
Tue Mar 31 17:40:21 2020 - [info]   172.16.10.22(172.16.10.22:3306)
Tue Mar 31 17:40:21 2020 - [info] Alive Slaves:
Tue Mar 31 17:40:21 2020 - [info]   172.16.10.21(172.16.10.21:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Tue Mar 31 17:40:21 2020 - [info]     GTID ON
Tue Mar 31 17:40:21 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Tue Mar 31 17:40:21 2020 - [info]   172.16.10.22(172.16.10.22:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Tue Mar 31 17:40:21 2020 - [info]     GTID ON
Tue Mar 31 17:40:21 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Tue Mar 31 17:40:21 2020 - [info] Current Alive Master: 172.16.10.20(172.16.10.20:3306)
Tue Mar 31 17:40:21 2020 - [info] Checking slave configurations..
Tue Mar 31 17:40:21 2020 - [info]  read_only=1 is not set on slave 172.16.10.21(172.16.10.21:3306).
Tue Mar 31 17:40:21 2020 - [info]  read_only=1 is not set on slave 172.16.10.22(172.16.10.22:3306).
Tue Mar 31 17:40:21 2020 - [info] Checking replication filtering settings..
Tue Mar 31 17:40:21 2020 - [info]  binlog_do_db= , binlog_ignore_db= 
Tue Mar 31 17:40:21 2020 - [info]  Replication filtering check ok.
Tue Mar 31 17:40:21 2020 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Tue Mar 31 17:40:21 2020 - [info] Checking SSH publickey authentication settings on the current master..
Tue Mar 31 17:40:22 2020 - [info] HealthCheck: SSH to 172.16.10.20 is reachable.
Tue Mar 31 17:40:22 2020 - [info] 
172.16.10.20(172.16.10.20:3306) (current master)
 +--172.16.10.21(172.16.10.21:3306)
 +--172.16.10.22(172.16.10.22:3306)

Tue Mar 31 17:40:22 2020 - [info] Checking replication health on 172.16.10.21..
Tue Mar 31 17:40:22 2020 - [info]  ok.
Tue Mar 31 17:40:22 2020 - [info] Checking replication health on 172.16.10.22..
Tue Mar 31 17:40:22 2020 - [info]  ok.
Tue Mar 31 17:40:22 2020 - [warning] master_ip_failover_script is not defined.
Tue Mar 31 17:40:22 2020 - [warning] shutdown_script is not defined.
Tue Mar 31 17:40:22 2020 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.
....整个复制环境状况是ok的

~~~

### 6、管理MHA

~~~shell
# 1) 开启MHA Manager监控
[root@db22 ~]# nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover  < /dev/null> /var/log/mha/app1/manager.log 2>&1 &
[1] 3924
## 参数解释：
--remove_dead_master_conf     // 该参数代表当发生主从切换后，老的主库的ip将会从配置文件中移除
--ignore_last_failover    	  // 在缺省情况下，如果MHA检测到连续发生宕机，且两次宕机间隔不足8小时的话，则不会进行Failover，之所以这样限制是为了避免ping-pong效应。该参数代表忽略上次MHA触发切换产生的文件，默认情况下，MHA发生切换后会在日志录，也就是上面我设置的/data产生app1.failover.complete文件，下次再次切换的时候如果发现该目录下存在该文件将允许触发切换，除非在第一次切换后收到删除该文件，为了方便，这里设置为--ignore_last_failover。
# 2) 检查MHA Manager的状态
[root@db22 ~]# masterha_check_status --conf=/etc/mha/app1.cnf
app1 (pid:3924) is running(0:PING_OK), master:172.16.10.20

可以看见已经在监控了，而且master的主机为172.16.10.20,如果启动失败，可以查看启动日志：
/var/log/mha/app1/manager.log
~~~

以上，MHA架构的mysql集群就已经部署完毕。

###  7、引入vip

> 主从切换时，应用程序无需修改mysql地址

~~~shell
# vip配置可以采用两种方式，一种通过keepalived的方式管理虚拟ip浮动；另外一种通过脚本方式启动虚拟ip的方式，这里采用第二种脚本的方式
# 1) master手动绑定vip
[root@db20 mysqlbinlog]# ifconfig ens33:1 172.16.10.50/24
[root@db20 mysqlbinlog]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:12:12:01 brd ff:ff:ff:ff:ff:ff
    inet 10.66.19.46/24 brd 10.66.19.255 scope global noprefixroute dynamic ens32
       valid_lft 78945sec preferred_lft 78945sec
    inet6 fe80::38dc:df2a:7b66:7f71/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:12:12:0b brd ff:ff:ff:ff:ff:ff
    inet 172.16.10.20/24 brd 172.16.10.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 172.16.10.50/24 brd 172.16.255.255 scope global ens33:0
# 2)  manager节点创建/usr/local/bin/master_ip_failover
[root@db22 ~]# vim /usr/local/bin/master_ip_failover
[root@db22 ~]# chmod +x /usr/local/bin/master_ip_failover 
[root@db22 ~]# cat /!$
cat //usr/local/bin/master_ip_failover
#!/usr/bin/env perl
 
use strict;
use warnings FATAL => 'all';
 
use Getopt::Long;
 
my (
    $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);
 
my $vip = '172.16.10.50/24';
my $key = '1';
my $ssh_start_vip = "/sbin/ifconfig ens33:$key $vip";
my $ssh_stop_vip = "/sbin/ifconfig ens33:$key down";
 
GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);
 
exit &main();
 
sub main {
 
    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";
 
    if ( $command eq "stop" || $command eq "stopssh" ) {
 
        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {
 
        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}
 
sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
     return 0  unless  ($ssh_user);
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}
 
sub usage {
    print
    "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
# 3) 更改manager配置文件
[root@db22 ~]# vim /etc/mha/app1.cnf 
[server default] 新增以下配置
master_ip_failover_script= /usr/local/bin/master_ip_failover
~~~

### 8、测试

~~~~shell
# 1) 在主库使用sysbench工具生成测试数据
[root@db20 ~]# curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
[root@db20 ~]# yum install sysbench -y
# 2) 创建测试库
mysql> create database tchua_db charset utf8mb4;
# 3) 在主库使用sysbech工具生成数据100W数据
[root@db20 ~]#  sysbench /usr/share/sysbench/oltp_insert.lua \
  --mysql-port=3306 \
  --mysql-user=root \
  --mysql-password='123456' \
  --mysql-db=tchua_db \
  --db-driver=mysql \
  --tables=1 \
  --mysql-socket=/tmp/mysql.sock \
  --table-size=1000000 \
  --report-interval=10 \
  --threads=1 \
  --time=120 \
  prepare
# 4) 停一台从节点(db21)的slave io线程，模拟主从延迟
[root@db21 ~]# mysql -uroot -p123456
mysql> stop slave io_thread;
# 5) 主库继续压测2分钟，产生大量的数据及binlog，2分钟后停止压测
[root@db20 ~]# sysbench /usr/share/sysbench/oltp_insert.lua \
  --mysql-port=3306 \
  --mysql-user=root \
  --mysql-password='123456' \
  --mysql-db=tchua_db \
  --db-driver=mysql \
  --tables=1 \
  --mysql-socket=/tmp/mysql.sock \
  --table-size=1000000 \
  --report-interval=10 \
  --threads=1 \
  --time=120 \
  run
~~~~

![image-20200403110630319](C:\Users\86155\AppData\Roaming\Typora\typora-user-images\image-20200403110630319.png)

~~~shell
# 6) 120s后开启从库(db21) io线程，开始追赶落后于master的binlog
[root@db21 ~]# mysql -uroot -p123456
mysql> start slave io_thread;
# 7) kill 主库进程模拟注会宕机
[root@db20 ~]# pkill mysql
# 8) 查看mha日志观察切换过程
[root@db22 ~]# cat /var/log/mha/app1/manager
..............................
..............................
From:
172.16.10.20(172.16.10.20:3306) (current master)
 +--172.16.10.21(172.16.10.21:3306)
 +--172.16.10.22(172.16.10.22:3306)

To:
172.16.10.21(172.16.1：0.21:3306) (new master)
 +--172.16.10.22(172.16.10.22:3306)
..................................
..................................
N SCRIPT TEST====/usr/sbin/ifconfig ens33:1 down==/usr/sbin/ifconfig ens33:1 172.16.10.50/24===

Enabling the VIP - 172.16.10.50/24 on the new master - 172.16.10.21 
Fri Apr  3 14:22:56 2020 - [info]  OK.
Fri Apr  3 14:22:56 2020 - [info] ** Finished master recovery successfully
....................................
....................................

从配置文件信息大概流程：
1、配置文件检查，检查mha主配置文件
2、master节点转移操作、vip转移
3、被选为新master的节点同步master日志(relay log)
这里，进行实验有一个问题就是，虽然开始db21数据远远落后于master，但是还选择db21为新的master节点。但是数据会同步过来，推测应该是从db22同步过来的。

~~~

mha完整Failover切换日志

~~~shell
Fri Apr  3 14:19:46 2020 - [info] MHA::MasterMonitor version 0.58.
Fri Apr  3 14:19:47 2020 - [info] GTID failover mode = 1
Fri Apr  3 14:19:47 2020 - [info] Dead Servers:
Fri Apr  3 14:19:47 2020 - [info] Alive Servers:
Fri Apr  3 14:19:47 2020 - [info]   172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:19:47 2020 - [info]   172.16.10.21(172.16.10.21:3306)
Fri Apr  3 14:19:47 2020 - [info]   172.16.10.22(172.16.10.22:3306)
Fri Apr  3 14:19:47 2020 - [info] Alive Slaves:
Fri Apr  3 14:19:47 2020 - [info]   172.16.10.21(172.16.10.21:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Fri Apr  3 14:19:47 2020 - [info]     GTID ON
Fri Apr  3 14:19:47 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:19:47 2020 - [info]   172.16.10.22(172.16.10.22:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Fri Apr  3 14:19:47 2020 - [info]     GTID ON
Fri Apr  3 14:19:47 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:19:47 2020 - [info] Current Alive Master: 172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:19:47 2020 - [info] Checking slave configurations..
Fri Apr  3 14:19:47 2020 - [info]  read_only=1 is not set on slave 172.16.10.21(172.16.10.21:3306).
Fri Apr  3 14:19:47 2020 - [info]  read_only=1 is not set on slave 172.16.10.22(172.16.10.22:3306).
Fri Apr  3 14:19:47 2020 - [info] Checking replication filtering settings..
Fri Apr  3 14:19:47 2020 - [info]  binlog_do_db= , binlog_ignore_db= 
Fri Apr  3 14:19:47 2020 - [info]  Replication filtering check ok.
Fri Apr  3 14:19:47 2020 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Fri Apr  3 14:19:47 2020 - [info] Checking SSH publickey authentication settings on the current master..
Fri Apr  3 14:19:47 2020 - [info] HealthCheck: SSH to 172.16.10.20 is reachable.
Fri Apr  3 14:19:47 2020 - [info] 
172.16.10.20(172.16.10.20:3306) (current master)
 +--172.16.10.21(172.16.10.21:3306)
 +--172.16.10.22(172.16.10.22:3306)

Fri Apr  3 14:19:47 2020 - [info] Checking master_ip_failover_script status:
Fri Apr  3 14:19:47 2020 - [info]   /usr/local/bin/master_ip_failover --command=status --ssh_user=root --orig_master_host=172.16.10.20 --orig_master_ip=172.16.10.20 --orig_master_port=3306 


IN SCRIPT TEST====/usr/sbin/ifconfig ens33:1 down==/usr/sbin/ifconfig ens33:1 172.16.10.50/24===

Checking the Status of the script.. OK 
Fri Apr  3 14:19:47 2020 - [info]  OK.
Fri Apr  3 14:19:47 2020 - [warning] shutdown_script is not defined.
Fri Apr  3 14:19:47 2020 - [info] Set master ping interval 2 seconds.
Fri Apr  3 14:19:47 2020 - [warning] secondary_check_script is not defined. It is highly recommended setting it to check master reachability from two or more routes.
Fri Apr  3 14:19:47 2020 - [info] Starting ping health check on 172.16.10.20(172.16.10.20:3306)..
Fri Apr  3 14:19:47 2020 - [info] Ping(SELECT) succeeded, waiting until MySQL doesn't respond..
Fri Apr  3 14:22:17 2020 - [warning] Got error on MySQL select ping: 2006 (MySQL server has gone away)
Fri Apr  3 14:22:17 2020 - [info] Executing SSH check script: exit 0
Fri Apr  3 14:22:18 2020 - [info] HealthCheck: SSH to 172.16.10.20 is reachable.
Fri Apr  3 14:22:19 2020 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '172.16.10.20' (111))
Fri Apr  3 14:22:19 2020 - [warning] Connection failed 2 time(s)..
Fri Apr  3 14:22:21 2020 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '172.16.10.20' (111))
Fri Apr  3 14:22:21 2020 - [warning] Connection failed 3 time(s)..
Fri Apr  3 14:22:23 2020 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '172.16.10.20' (111))
Fri Apr  3 14:22:23 2020 - [warning] Connection failed 4 time(s)..
Fri Apr  3 14:22:23 2020 - [warning] Master is not reachable from health checker!
Fri Apr  3 14:22:23 2020 - [warning] Master 172.16.10.20(172.16.10.20:3306) is not reachable!
Fri Apr  3 14:22:23 2020 - [warning] SSH is reachable.
Fri Apr  3 14:22:23 2020 - [info] Connecting to a master server failed. Reading configuration file /etc/masterha_default.cnf and /etc/mha/app1.cnf again, and trying to connect to all servers to check server status..
Fri Apr  3 14:22:23 2020 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Fri Apr  3 14:22:23 2020 - [info] Reading application default configuration from /etc/mha/app1.cnf..
Fri Apr  3 14:22:23 2020 - [info] Reading server configuration from /etc/mha/app1.cnf..
Fri Apr  3 14:22:24 2020 - [info] GTID failover mode = 1
Fri Apr  3 14:22:24 2020 - [info] Dead Servers:
Fri Apr  3 14:22:24 2020 - [info]   172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:22:24 2020 - [info] Alive Servers:
Fri Apr  3 14:22:24 2020 - [info]   172.16.10.21(172.16.10.21:3306)
Fri Apr  3 14:22:24 2020 - [info]   172.16.10.22(172.16.10.22:3306)
Fri Apr  3 14:22:24 2020 - [info] Alive Slaves:
Fri Apr  3 14:22:24 2020 - [info]   172.16.10.21(172.16.10.21:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Fri Apr  3 14:22:24 2020 - [info]     GTID ON
Fri Apr  3 14:22:24 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:22:24 2020 - [info]   172.16.10.22(172.16.10.22:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Fri Apr  3 14:22:24 2020 - [info]     GTID ON
Fri Apr  3 14:22:24 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:22:24 2020 - [info] Checking slave configurations..
Fri Apr  3 14:22:24 2020 - [info]  read_only=1 is not set on slave 172.16.10.21(172.16.10.21:3306).
Fri Apr  3 14:22:24 2020 - [info]  read_only=1 is not set on slave 172.16.10.22(172.16.10.22:3306).
Fri Apr  3 14:22:24 2020 - [info] Checking replication filtering settings..
Fri Apr  3 14:22:24 2020 - [info]  Replication filtering check ok.
Fri Apr  3 14:22:24 2020 - [info] Master is down!
Fri Apr  3 14:22:24 2020 - [info] Terminating monitoring script.
Fri Apr  3 14:22:24 2020 - [info] Got exit code 20 (Master dead).
Fri Apr  3 14:22:24 2020 - [info] MHA::MasterFailover version 0.58.
Fri Apr  3 14:22:24 2020 - [info] Starting master failover.
Fri Apr  3 14:22:24 2020 - [info] 
Fri Apr  3 14:22:24 2020 - [info] * Phase 1: Configuration Check Phase..
Fri Apr  3 14:22:24 2020 - [info] 
Fri Apr  3 14:22:25 2020 - [info] GTID failover mode = 1
Fri Apr  3 14:22:25 2020 - [info] Dead Servers:
Fri Apr  3 14:22:25 2020 - [info]   172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:22:25 2020 - [info] Checking master reachability via MySQL(double check)...
Fri Apr  3 14:22:25 2020 - [info]  ok.
Fri Apr  3 14:22:25 2020 - [info] Alive Servers:
Fri Apr  3 14:22:25 2020 - [info]   172.16.10.21(172.16.10.21:3306)
Fri Apr  3 14:22:25 2020 - [info]   172.16.10.22(172.16.10.22:3306)
Fri Apr  3 14:22:25 2020 - [info] Alive Slaves:
Fri Apr  3 14:22:25 2020 - [info]   172.16.10.21(172.16.10.21:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Fri Apr  3 14:22:25 2020 - [info]     GTID ON
Fri Apr  3 14:22:25 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:22:25 2020 - [info]   172.16.10.22(172.16.10.22:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Fri Apr  3 14:22:25 2020 - [info]     GTID ON
Fri Apr  3 14:22:25 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:22:25 2020 - [info] Starting GTID based failover.
Fri Apr  3 14:22:25 2020 - [info] 
Fri Apr  3 14:22:25 2020 - [info] ** Phase 1: Configuration Check Phase completed.
Fri Apr  3 14:22:25 2020 - [info] 
Fri Apr  3 14:22:25 2020 - [info] * Phase 2: Dead Master Shutdown Phase..
Fri Apr  3 14:22:25 2020 - [info] 
Fri Apr  3 14:22:25 2020 - [info] Forcing shutdown so that applications never connect to the current master..
Fri Apr  3 14:22:25 2020 - [info] Executing master IP deactivation script:
Fri Apr  3 14:22:25 2020 - [info]   /usr/local/bin/master_ip_failover --orig_master_host=172.16.10.20 --orig_master_ip=172.16.10.20 --orig_master_port=3306 --command=stopssh --ssh_user=root  


IN SCRIPT TEST====/usr/sbin/ifconfig ens33:1 down==/usr/sbin/ifconfig ens33:1 172.16.10.50/24===

Disabling the VIP on old master: 172.16.10.20 
SIOCSIFFLAGS: Cannot assign requested address
Fri Apr  3 14:22:26 2020 - [info]  done.
Fri Apr  3 14:22:26 2020 - [warning] shutdown_script is not set. Skipping explicit shutting down of the dead master.
Fri Apr  3 14:22:26 2020 - [info] * Phase 2: Dead Master Shutdown Phase completed.
Fri Apr  3 14:22:26 2020 - [info] 
Fri Apr  3 14:22:26 2020 - [info] * Phase 3: Master Recovery Phase..
Fri Apr  3 14:22:26 2020 - [info] 
Fri Apr  3 14:22:26 2020 - [info] * Phase 3.1: Getting Latest Slaves Phase..
Fri Apr  3 14:22:26 2020 - [info] 
Fri Apr  3 14:22:26 2020 - [info] The latest binary log file/position on all slaves is mysql-bin.000005:109998970
Fri Apr  3 14:22:26 2020 - [info] Retrieved Gtid Set: fa1167f9-7265-11ea-ac36-000c29121201:583849-824020
Fri Apr  3 14:22:26 2020 - [info] Latest slaves (Slaves that received relay log files to the latest):
Fri Apr  3 14:22:26 2020 - [info]   172.16.10.21(172.16.10.21:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Fri Apr  3 14:22:26 2020 - [info]     GTID ON
Fri Apr  3 14:22:26 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:22:26 2020 - [info]   172.16.10.22(172.16.10.22:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Fri Apr  3 14:22:26 2020 - [info]     GTID ON
Fri Apr  3 14:22:26 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:22:26 2020 - [info] The oldest binary log file/position on all slaves is mysql-bin.000005:109998970
Fri Apr  3 14:22:26 2020 - [info] Retrieved Gtid Set: fa1167f9-7265-11ea-ac36-000c29121201:583849-824020
Fri Apr  3 14:22:26 2020 - [info] Oldest slaves:
Fri Apr  3 14:22:26 2020 - [info]   172.16.10.21(172.16.10.21:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Fri Apr  3 14:22:26 2020 - [info]     GTID ON
Fri Apr  3 14:22:26 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:22:26 2020 - [info]   172.16.10.22(172.16.10.22:3306)  Version=5.7.29-log (oldest major version between slaves) log-bin:enabled
Fri Apr  3 14:22:26 2020 - [info]     GTID ON
Fri Apr  3 14:22:26 2020 - [info]     Replicating from 172.16.10.20(172.16.10.20:3306)
Fri Apr  3 14:22:26 2020 - [info] 
Fri Apr  3 14:22:26 2020 - [info] * Phase 3.3: Determining New Master Phase..
Fri Apr  3 14:22:26 2020 - [info] 
Fri Apr  3 14:22:26 2020 - [info] Searching new master from slaves..
Fri Apr  3 14:22:26 2020 - [info]  Candidate masters from the configuration file:
Fri Apr  3 14:22:26 2020 - [info]  Non-candidate masters:
Fri Apr  3 14:22:26 2020 - [info] New master is 172.16.10.21(172.16.10.21:3306)
Fri Apr  3 14:22:26 2020 - [info] Starting master failover..
Fri Apr  3 14:22:26 2020 - [info] 
From:
172.16.10.20(172.16.10.20:3306) (current master)
 +--172.16.10.21(172.16.10.21:3306)
 +--172.16.10.22(172.16.10.22:3306)

To:
172.16.10.21(172.16.10.21:3306) (new master)
 +--172.16.10.22(172.16.10.22:3306)
Fri Apr  3 14:22:26 2020 - [info] 
Fri Apr  3 14:22:26 2020 - [info] * Phase 3.3: New Master Recovery Phase..
Fri Apr  3 14:22:26 2020 - [info] 
Fri Apr  3 14:22:26 2020 - [info]  Waiting all logs to be applied.. 
Fri Apr  3 14:22:56 2020 - [info]   done.
Fri Apr  3 14:22:56 2020 - [info]  Replicating from the latest slave 172.16.10.22(172.16.10.22:3306) and waiting to apply..
Fri Apr  3 14:22:56 2020 - [info]  Waiting all logs to be applied on the latest slave.. 
Fri Apr  3 14:22:56 2020 - [info]  Resetting slave 172.16.10.21(172.16.10.21:3306) and starting replication from the new master 172.16.10.22(172.16.10.22:3306)..
Fri Apr  3 14:22:56 2020 - [info]  Executed CHANGE MASTER.
Fri Apr  3 14:22:56 2020 - [info]  Slave started.
Fri Apr  3 14:22:56 2020 - [info]  Waiting to execute all relay logs on 172.16.10.21(172.16.10.21:3306)..
Fri Apr  3 14:22:56 2020 - [info]  master_pos_wait(mysql-bin.000002:154) completed on 172.16.10.21(172.16.10.21:3306). Executed 1 events.
Fri Apr  3 14:22:56 2020 - [info]   done.
Fri Apr  3 14:22:56 2020 - [info]   done.
Fri Apr  3 14:22:56 2020 - [info] Getting new master's binlog name and position..
Fri Apr  3 14:22:56 2020 - [info]  mysql-bin.000002:154
Fri Apr  3 14:22:56 2020 - [info]  All other slaves should start replication from here. Statement should be: CHANGE MASTER TO MASTER_HOST='172.16.10.21', MASTER_PORT=3306, MASTER_AUTO_POSITION=1, MASTER_USER='rep', MASTER_PASSWORD='xxx';
Fri Apr  3 14:22:56 2020 - [info] Master Recovery succeeded. File:Pos:Exec_Gtid_Set: mysql-bin.000002, 154, fa1167f9-7265-11ea-ac36-000c29121201:1-824020
Fri Apr  3 14:22:56 2020 - [info] Executing master IP activate script:
Fri Apr  3 14:22:56 2020 - [info]   /usr/local/bin/master_ip_failover --command=start --ssh_user=root --orig_master_host=172.16.10.20 --orig_master_ip=172.16.10.20 --orig_master_port=3306 --new_master_host=172.16.10.21 --new_master_ip=172.16.10.21 --new_master_port=3306 --new_master_user='mha'   --new_master_password=xxx
Unknown option: new_master_user
Unknown option: new_master_password


IN SCRIPT TEST====/usr/sbin/ifconfig ens33:1 down==/usr/sbin/ifconfig ens33:1 172.16.10.50/24===

Enabling the VIP - 172.16.10.50/24 on the new master - 172.16.10.21 
Fri Apr  3 14:22:56 2020 - [info]  OK.
Fri Apr  3 14:22:56 2020 - [info] ** Finished master recovery successfully.
Fri Apr  3 14:22:56 2020 - [info] * Phase 3: Master Recovery Phase completed.
Fri Apr  3 14:22:56 2020 - [info] 
Fri Apr  3 14:22:56 2020 - [info] * Phase 4: Slaves Recovery Phase..
Fri Apr  3 14:22:56 2020 - [info] 
Fri Apr  3 14:22:56 2020 - [info] 
Fri Apr  3 14:22:56 2020 - [info] * Phase 4.1: Starting Slaves in parallel..
Fri Apr  3 14:22:56 2020 - [info] 
Fri Apr  3 14:22:56 2020 - [info] -- Slave recovery on host 172.16.10.22(172.16.10.22:3306) started, pid: 61234. Check tmp log /var/log/mha/app1/172.16.10.22_3306_20200403142224.log if it takes time..
Fri Apr  3 14:22:57 2020 - [info] 
Fri Apr  3 14:22:57 2020 - [info] Log messages from 172.16.10.22 ...
Fri Apr  3 14:22:57 2020 - [info] 
Fri Apr  3 14:22:56 2020 - [info]  Resetting slave 172.16.10.22(172.16.10.22:3306) and starting replication from the new master 172.16.10.21(172.16.10.21:3306)..
Fri Apr  3 14:22:57 2020 - [info]  Executed CHANGE MASTER.
Fri Apr  3 14:22:57 2020 - [info]  Slave started.
Fri Apr  3 14:22:57 2020 - [info]  gtid_wait(fa1167f9-7265-11ea-ac36-000c29121201:1-824020) completed on 172.16.10.22(172.16.10.22:3306). Executed 0 events.
Fri Apr  3 14:22:57 2020 - [info] End of log messages from 172.16.10.22.
Fri Apr  3 14:22:57 2020 - [info] -- Slave on host 172.16.10.22(172.16.10.22:3306) started.
Fri Apr  3 14:22:57 2020 - [info] All new slave servers recovered successfully.
Fri Apr  3 14:22:57 2020 - [info] 
Fri Apr  3 14:22:57 2020 - [info] * Phase 5: New master cleanup phase..
Fri Apr  3 14:22:57 2020 - [info] 
Fri Apr  3 14:22:57 2020 - [info] Resetting slave info on the new master..
Fri Apr  3 14:22:57 2020 - [info]  172.16.10.21: Resetting slave info succeeded.
Fri Apr  3 14:22:57 2020 - [info] Master failover to 172.16.10.21(172.16.10.21:3306) completed successfully.
Fri Apr  3 14:22:57 2020 - [info] Deleted server1 entry from /etc/mha/app1.cnf .
Fri Apr  3 14:22:57 2020 - [info] 

----- Failover Report -----

app1: MySQL Master failover 172.16.10.20(172.16.10.20:3306) to 172.16.10.21(172.16.10.21:3306) succeeded

Master 172.16.10.20(172.16.10.20:3306) is down!

Check MHA Manager logs at db22:/var/log/mha/app1/manager for details.

Started automated(non-interactive) failover.
Invalidated master IP address on 172.16.10.20(172.16.10.20:3306)
Selected 172.16.10.21(172.16.10.21:3306) as a new master.
172.16.10.21(172.16.10.21:3306): OK: Applying all logs succeeded.
172.16.10.21(172.16.10.21:3306): OK: Activated master IP address.
172.16.10.22(172.16.10.22:3306): OK: Slave started, replicating from 172.16.10.21(172.16.10.21:3306)
172.16.10.21(172.16.10.21:3306): Resetting slave info succeeded.
Master failover to 172.16.10.21(172.16.10.21:3306) completed successfully.

~~~

