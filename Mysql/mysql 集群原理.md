## mysql 集群原理

### 一、MHA

#### 1.1 介绍

~~~shell
  mha 是日本的一位MySQL专家采用Perl语言编写的一个脚本管理工具，是一套优秀的作为MySQL高可用性环境下故障切换和主从提升的高可用软件。在MySQL故障切换过程中，MHA能做到在0~30秒之内自动完成数据库的故障切换操作，并且在进行故障切换的过程中，MHA能在最大程度上保证数据的一致性，以达到真正意义上的高可用。
~~~

#### 1.2 软件组成

~~~shell
  MHA是自动的master故障转移和Slave提升的软件包.它是基于标准的MySQL复制(异步/半同步).该软件由两部分组成：MHA Manager（管理节点）和 MHA Node（数据节点）。
1. MHA Manager可以单独部署在一台独立的机器上管理多个master-slave集群，也可以部署在一台slave节点上。MHA Manager会定时探测集群中的node节点,当发现master出现故障的时候,它可以自动将具有最新数据的slave提升为新的master,然后将所有其它的slave导向新的master上.整个故障转移过程对应用程序是透明的。
2. MHA Node运行在每台MySQL服务器上，它通过监控具备解析和清理logs功能的脚本来加快故障转移的。
~~~

#### 1.3 架构原理说明

~~~shell
  MHA的目的在于维护MYSQl Replication中的Master库的高可用性，最大特点是修复多个slave之间的差异日志，最终使所有Slave保持数据一致，然后从中选择一个充当新的Master，并将其它Slave指向它。 
~~~

 工作流程主要如下：

1. 从宕机崩溃的master节点保存二进制日志事件(binlog events)

2. 识别含有最新更新的slave

3. 应用差异的中继日志(relay log)到其它slave

4. 应用master保存的二进制日志事件(binlog events)

5. 提升一个slave为新的master

6. 使其它slave连接新的master进行复制

   

   

思考： 

关于主库宕机情况说明：

* 主库挂了，但是主库所有日志都被全部从库接收，此时会应用binlog最全的一台从库作为新的主库，其它从库指定到新的主库地址即可。
* 主库挂了，主库所有binlog日志被从库接收，但是主库上还有几条记录还没有sync到binlog中，所以从库也没有接收到这个event，如果此时做切换，会丢失这个event。此时，如果主库还可以通过ssh访问到，binlog文件可以查看，那么先copy该event到所有的从库上，最后再切换主库。
* 主库挂了，硬件宕机或者 SSH 不能连接，不能获取到最新的 binlog 日志，如果复制出现延迟，会丢失数据，需要结合半同步复制。

~~~shell
# 工作原理：
  当master出现故障时，通过对比slave之间I/O线程读取master binlog的位置，选取最接近的slave做为latest slave。其它slave通过与latest slave对比生成差异中继日志。在latest slave上应用从master保存的binlog，同时将latest slave提升为master。最后在其它slave上应用相应的差异中继日志并开始从新的master开始复制。
  在MHA实现Master故障切换过程中，MHA Node会试图访问故障的master（通过SSH），如果可以访问（不是硬件故障，比如InnoDB数据文件损坏等），会保存二进制文件，以最大程度保证数据不丢失。MHA和半同步复制一起使用会大大降低数据丢失的危险。
~~~

#### 1.4 软件架构

> MHA由两部分组成: Manager工具包 和 Node工具包

~~~shell
# 1) Manager工具包
masterha_check_ssh             # 检查MHA的SSH配置状况
masterha_check_repl            # 检查MySQL复制状况
masterha_manger                # 启动MHA
masterha_check_status          # 检测当前MHA运行状态
masterha_master_monitor        # 检测master是否宕机
masterha_master_switch         # 控制故障转移（自动或者手动）
masterha_conf_host             # 添加或删除配置的server信息
# 2) Node工具包
save_binary_logs               #（保存二进制日志） 保存和复制master的二进制日志
apply_diff_relay_logs          # （应用差异中继日志） 识别差异的中继日志事件并将其差异的事件应用于其他slave
filter_mysqlbinlog             #  去除不必要的ROLLBACK事件（MHA已不再使用这个工具）
purge_relay_logs               # （清理中继日志） 清除中继日志（不会阻塞SQL线程）
~~~

思考：

1. MHA如何保持数据的一致性？

   > 没有绝对的数据一致性

  save_binary_logs  如果master的二进制日志可以存取的话，保存复制master的二进制日志，最大程度保证数据不丢

 apply_diff_relay_logs 以最新的slave为参照，生成差异的中继日志并将所有差异事件应用到其他所有的slave

NOTE:  对比的是relay log，relay log越新就越接近于master，才能保证数据是最新的。

#### 1.5 MHA优势

~~~shell
# 1) 故障切换快
  在主从复制集群中，只要从库在复制上没有延迟，MHA通常可以在数秒内实现故障切换。9-10秒内检查到master故障，可以选择在7-10秒关闭master以避免出现裂脑，几秒钟内，将差异中继日志(relay log)应用到新的master上，因此总的宕机时间通常为10-30秒。恢复新的master后，MHA并行的恢复其余的slave。即使在有数万台slave，也不会影响master的恢复时间。
# 2) master故障不会导致数据不一致
  当目前的master出现故障时，MHA自动识别slave之间中继日志(relay log)的不同，并应用到所有的slave中。这样所有的salve能够保持同步，只要所有的slave处于存活状态。和Semi-Synchronous Replication一起使用，（几乎）可以保证没有数据丢失。
# 3) 无需修改当前的MySQL设置
  MHA的设计的重要原则之一就是尽可能地简单易用。MHA工作在传统的MySQL版本5.0和之后版本的主从复制环境中。和其它高可用解决方法比，MHA并不需要改变MySQL的部署环境。MHA适用于异步和半同步的主从复制。
  启动/停止/升级/降级/安装/卸载MHA不需要改变（包扩启动/停止）MySQL复制。当需要升级MHA到新的版本，不需要停止MySQL，仅仅替换到新版本的MHA，然后重启MHA　Manager就好了。
  MHA运行在MySQL 5.0开始的原生版本上。一些其它的MySQL高可用解决方案需要特定的版本（比如MySQL集群、带全局事务ID的MySQL等等），但并不仅仅为了master的高可用才迁移应用的。在大多数情况下，已经部署了比较旧MySQL应用，并且不想仅仅为了实现Master的高可用，花太多的时间迁移到不同的存储引擎或更新的前沿发行版。MHA工作的包括5.0/5.1/5.5的原生版本的MySQL上，所以并不需要迁移。
# 4) 无需增加大量的服务器
  MHA由MHA Manager和MHA Node组成。MHA Node运行在需要故障切换/恢复的MySQL服务器上，因此并不需要额外增加服务器。MHA Manager运行在特定的服务器上，因此需要增加一台（实现高可用需要2台），但是MHA Manager可以监控大量（甚至上百台）单独的master，因此，并不需要增加大量的服务器。即使在一台slave上运行MHA Manager也是可以的。综上，实现MHA并没用额外增加大量的服务。
# 5) 无性能下降
  MHA适用于异步或半同步的MySQL复制。监控master时，MHA仅仅是每隔几秒（默认是3秒）发送一个ping包，并不发送重查询。可以得到像原生MySQL复制一样快的性能。
# 6) 适用于任何存储引擎
  MHA可以运行在只要MySQL复制运行的存储引擎上，并不仅限制于InnoDB，即使在不易迁移的传统的MyISAM引擎环境，一样可以使用MHA。
~~~

#### 1.6 参数

~~~shell
# MHA
1) candidate_master  
  设置为1，表示该节点为候选master，发生主从切换以后将会将此从库提升为主库，即使这个主库不是集群中事件最新的。
2) check_repl_delay  
  默认情况下如果一个slave落后master 100M的relay logs的话，MHA将不会选择该slave作为一个新的master，因为对于这个slave的恢复需要花费很长时间，通过设置check_repl_delay=0,MHA在触发切换选择一个新的master的时候将会忽略复制延时，这个参数对于设置了candidate_master=1的主机非常有用，因为这个候选主在切换的过程中一定是新的master。
 # MYSQL
 1) relay log(slave节点)
   MHA在发生切换的过程中，从库的恢复过程中依赖于relay log的相关信息，所以这里要将relay log的自动清除设置为OFF，采用手动清除relay log的方式。
  在默认情况下，从服务器上的中继日志会在SQL线程执行完毕后被自动删除。但是在MHA环境中，这些中继日志在恢复其他从服务器时可能会被用到，因此需要禁用中继日志的自动删除功能(relay_log_purge = 0)。定期清除中继日志需要考虑到复制延时的问题。
 2) sync_binlog(master节点)
当sync_binlog 的默认值是0，像操作系统刷其他文件的机制一样，MySQL不会同步到磁盘中去而是依赖操作系统来刷新binary log。
当sync_binlog =N (N>0) ，MySQL 在每写 N次 二进制日志binary log时，会使用fdatasync()函数将它的写二进制日志binary log同步到磁盘中去。这种方式对磁盘I/O会造成 10~20%的影响
~~~

#### 1.7 补充思考

1. 为什么说MHA结合Semi-Synchronous Replication会减少数据丢失的？

   ~~~shell
   # 半同步原理
     半同步复制时，为了保证主库上的每一个Binlog事务都能够被可靠的复制到从库上，主库在每次事务成功提交时，并不及时反馈给前端应用用户，而是等待其中的一个从库也接收到Binlog事务并成功写入中继日志后，主库才返回commit操作成功给客户端。半同步复制保证了事务成功提交后，至少有两份日志记录，一份在主库的Binlog日志上，另一份在至少一个从库的中继日志Relay log上，从而更近一步保证了数据的完整性。
   ~~~

2. purge_relay_logs 删除relay log脚本

   ~~~shell
   #!/bin/bash
   user=root
   passwd=123456
   port=3306
   host=localhost
   log_dir='/data/masterha/log'  
   work_dir='/data'
   purge='/usr/local/bin/purge_relay_logs'
    
   if [ ! -d $log_dir ]
   then
      mkdir $log_dir -p
   fi
    
   $purge --user=$user --host=$host --password=$passwd --disable_relay_log_purge --port=$port --workdir=$work_dir >> $log_dir/purge_relay_logs.log 2>&1
   ~~~


### 二、PXC

#### 2.1 介绍

  ~~~shell
Galera是Codership提供的多主数据同步复制机制，可以实现多个节点间的数据同步复制以及读写，并且可保障数据库的服务高可用及数据一致性。基于Galera的高可用方案主要有MariaDB Galera Cluster和Percona XtraDB Cluster（简称PXC），目前PXC用的会比较多一些。mariadb的集群原理跟PXC一样,maridb-cluster其实就是PXC，两者原理是一样的。
  ~~~

#### 2.2 软件架构

> Percona XtraDB Cluster（简称PXC集群）提供了MySQL高可用的一种实现方法。

~~~shell
1. 集群是有节点组成的，推荐配置至少3个节点，但是也可以运行在2个节点上。
2. 每个节点都是普通的mysql/percona服务器，可以将现有的数据库服务器组成集群，反之，也可以将集群拆分成单独的服务器。
3. 每个节点都包含完整的数据副本。
# NOTE： PXC集群主要由两部分组成：Percona Server with XtraDB和Write Set Replication patches（使用了Galera library，一个通用的用于事务型应用的同步、多主复制插件）
~~~

#### 2.3 PXC特性

> 强一致性、无同步延迟

* 多主架构：真正的多点读写的集群，在任何时候读写数据，都是最新的。
* 同步复制：集群不同节点之间数据同步，没有延迟，在数据库挂掉之后，数据不会丢失。
* 并发复制：从节点在APPLY数据时，支持并行执行，有更好的性能表现。
* 故障切换：在出现数据库故障时，因为支持多点写入，切的非常容易。
* 热插拔：在服务期间，如果数据库挂了，只要监控程序发现的够快，不可服务时间就会非常少。在节点故障期间，节点本身对集群的影响非常小。
* 自动节点克隆：在新增节点，或者停机维护时，增量数据或者基础数据不需要人工手动备份提供，Galera Cluster会自动拉取在线节点数据，最终集群会变为一致。
* 应用透明：集群的维护，对应用程序是透明的，几乎感觉不到。 以上几点，足以说明Galera Cluster是一个既稳健，又在数据一致性、完整性及高性能方面有出色表现的高可用解决方案。

#### 2.4 PXC优缺点

* 优势

  ~~~shell
  1. 实现mysql数据库集群架构的高可用性和数据的强一致性。
  2. 数据同步复制(并发复制)，几乎无延迟。
  3. 多个可同时读写的节点，可实现写扩展，不过最好事先进行分库分表，让各个节点分别写不同的表或者库，避免让galera解决数据冲突。
  5. 新增节点可以自动部署，无需手动备份恢复，部署操作简单。
  6. 完美兼容Mysql。
  ~~~

* 劣势

  ~~~shell
  1. 只支持Innode引擎
  2. PXC一致性控制机制，并不是绝对的，比如：集群允许在两个节点上同时执行操作同一行的两个事务，但是只有一个能执行成功，另一个会被终止，集群会给被终止的客户端返回死锁错误(Error: 1213 SQLSTATE: 40001 (ER_LOCK_DEADLOCK))
  3. 写入效率取决于节点中配置最弱的一台，因为PXC集群采用的是强一致性原则，一个更改操作在所有节点都成功才算执行成功。
  4. 所有表都需要有主键。
  5. 不支持LOCK TABLE等显示锁操作。
  6. 锁冲突、死锁问题相对更多: 因为需要保证数据的一致性，所以在多节点并发写时锁冲突问题比较严重。
  7. 不支持XA。
  8. 新加入节点开销大，数据量大时，采用SST传输代价高，需要复制完整的数据。
  9. 存在写扩大问题。
  10. 如果并发事务量很大的话，建议采用InfiniBand网络，降低网络延迟。
  11. PXC牺牲性能保证数据一致性。
  # NOTE：采用PXC的主要目的是解决数据的一致性问题，高可用是顺带实现的。因为PXC存在写扩大以及短板效应，并发效率会有较大损失，类似semi sync replication机制
  # NOTE: MySQL XA 是基于Open Group 的<<Distributed Transaction Processing:The XA Specification>> 标准实现的，支持分布式事务，允许多个数据库实例参与一个全局的事务。MySQl XA 从MySQL 5.0 开始引入，仅innodb存储引擎支持MySQL XA事务
  
  ~~~

#### 2.5 PXC原理

* 端口

  ~~~shell
  3306：数据库对外服务的端口号.
  4444：请求SST的端口(SST是指数据库一个备份全量文件的传输)。
  4567: 组成员之间进行沟通的一个端口号.
  4568: 传输IST用的。相对于SST来说的一个增量。
  ~~~

* 概念

  ~~~shell
  SST：全量传输,xtrabackup、mysqldump和rsync三种方法。
  IST：增量传输。
  ~~~

* 原理

  ~~~shell
    当client端执行dml操作时，将操作发给server，server的native进程处理请求，client端执行commit，server将复制写数据集发给group(cluster)，cluster中每个动作对应一个GTID，其它server接收到并通过验证(合并数据)后，执行appyl_cb动作和commit_cb动作，若验证没通过，则会退出处理;当前server节点验证通过后，执行commit_cb，并返回，若没通过，执行rollback_cb。
  ~~~

  注意：PXC结构里面，如果主节点写入过大，apply_cb 时间会不会跟不上，那么wsrep_slave_threads参数 解决apply_cb跟不上问题 配置成和CPU的个数相等或是1.5倍。当前节点commit_cb 后就可以返回了，推过去之后，验证通过就行了可以返回客户端了，cb也就是commit block 提交数据块.

#### 2.6 注意问题

* 节点状态说明

  ~~~shell
  1. OPEN: 节点启动成功，尝试连接到集群，如果失败则根据配置退出或创建新的集群
  2. PRIMARY: 节点处于集群PC中，尝试从集群中选取donor进行数据同步
  3. JOINER: 节点处于等待接收/接收数据文件状态，数据传输完成后在本地加载数据
  4. JOINED: 节点完成数据同步工作，尝试保持和集群进度一致
  5. SYNCED：节点正常提供服务：数据的读写，集群数据的同步，新加入节点的sst请求
  6. DONOR(贡献数据者)：节点处于为新节点准备或传输集群全量数据状态，对客户端不可用。
  
  # NOTE: doner节点就是数据的贡献者，如果一个新节点加入集群，此时又需要大量数据的SST传输，就有可能因此而拖垮整个集群的性能。所以在生产环境中，如果数据量小，还可以使用SST全量传输，但如果数据量很大就不建议使用这种方式了。可以考虑先建立主从关系，在加入集群。
  ~~~

* 操作问题

  ~~~shell
  1. 大表更新操作尽量使用：pt-online-schema-change。
  ~~~

#### 2.7 补充扩展

*** Galera 知识扩展 ***

### 三、MRG

