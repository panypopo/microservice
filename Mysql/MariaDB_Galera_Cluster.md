#MariaDB Galera Cluster
##MariaDB简介
MariaDB数据库管理系统是MySQL的一个分支，主要由开源社区在维护，采用GPL授权许可。开发这个分支的原因之一是：甲骨文公司收购了MySQL后，有将MySQL闭源的潜在风险，因此社区采用分支的方式来避开这个风险。MariaDB的目的是完全兼容MySQL，包括API和命令行，使之能轻松成为MySQL的代替品。在存储引擎方面，10.0.9版起使用XtraDB（名称代号为Aria）来代替MySQL的InnoDB。MariaDB由MySQL的创始人麦克尔·维德纽斯主导开发，他早前曾以10亿美元的价格，将自己创建的公司MySQL AB卖给了SUN，此后，随着SUN被甲骨文收购，MySQL的所有权也落入Oracle的手中。MariaDB名称来自麦克尔·维德纽斯的女儿玛丽亚（英语：Maria）的名字。


##MariaDB Galera Cluster 介绍
MariaDB集群是MariaDB同步多主机集群。它仅支持XtraDB/ InnoDB存储引擎。

主要功能:

- 同步复制
- 真正的multi-master，即所有节点可以同时读写数据库
- 自动的节点成员控制，失效节点自动被清除
- 新节点加入数据自动复制
- 真正的并行复制，行级
- 用户可以直接连接集群，使用感受上与MySQL完全一致

优势:

- 因为是多主，所以不存在Slavelag(延迟)
- 不存在丢失事务的情况
- 同时具有读和写的扩展能力
- 更小的客户端延迟
- 节点间数据是同步的,而Master/Slave模式是异步的,不同slave上的binlog可能是不同的


##Getting Started
###准备
MariaDB集群至少需要3个节点(避免脑裂问题），所以需要准备3个服务器(ubuntu14.04)。
分别是：

- node1 172.17.0.5
- node2 172.17.0.2
- node3 172.17.0.4


> 可使用docker在其中一台机器上安装相关软件，再commit成镜像，方便增加节点

###安装
MariaDB Galera Cluster目前只支持Linux系统。本示例使用ubuntu14.04进行安装。

执行以下命令进行安装mariaDB。MariaDB10.1版本之后MariaDB Server 和 MariaDB Galera Server安装包已经合并在一起，所以不需要分开安装。

~~~sh
apt-get update
apt-get install python-software-properties
apt-get install software-properties-common
apt-get install vim

apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xcbcb082a1bb943db
add-apt-repository 'deb [arch=amd64,i386] http://mariadb.nethub.com.hk/repo/10.1/ubuntu precise main'

apt-get update
apt-get install mariadb-server mariadb-client
~~~

安装过程中会提示设置数据库root用户的密码。

###配置

修改配置文件（一开始该文件）：

~~~sh
vim /etc/mysql/conf.d/cluster.cnf
~~~

~~~vim
[mysqld]
wsrep_provider=/usr/lib/galera/libgalera_smm.so
wsrep_cluster_name="testcluster"
##集群地址
wsrep_cluster_address="gcomm://172.17.0.2,172.17.0.4,172.17.0.5"
##本机地址
wsrep_node_address="172.17.0.2"
##本机名称
wsrep_node_name="NODE_2"
query_cache_size=0
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
innodb_doublewrite=1
wsrep_on=ON
innodb_flush_log_at_trx_commit=0

[mysql_safe]
log-error=/var/log/mysqld.log  
pid-file=/var/run/mysqld/mysqld.pid
~~~

保存后退出。

> 可在三个节点中的其中一个配置完成后，copy到另外的节点进行修改。


###启动

先新建集群启动，在其中一台机器，比如172.17.0.5，执行：

~~~sh
service mysql start --wsrep-new-cluster
~~~

然后在其他机器中执行：

~~~sh
service mysql start --wsrep_cluster_address=gcomm://172.17.0.5
~~~

自此整个集群就已经启动完毕。

###验证
在其中一台机器上连接数据库：

~~~sh
mysql -uroot -p
~~~

登陆后执行：

~~~sql
> show status like 'wsrep%';

+------------------------------+--------------------------------------+
| Variable_name                | Value                                |
+------------------------------+--------------------------------------+
| wsrep_apply_oooe             | 0.000000                             |
| wsrep_apply_oool             | 0.000000                             |
| wsrep_apply_window           | 1.000000                             |
| wsrep_causal_reads           | 0                                    |
| wsrep_cert_deps_distance     | 1.000000                             |
| wsrep_cert_index_size        | 1                                    |
| wsrep_cert_interval          | 0.000000                             |
| wsrep_cluster_conf_id        | 3                                    |
| wsrep_cluster_size           | 3                                    |
| wsrep_cluster_state_uuid     | b9c8411e-5f8c-11e6-bbf1-bf6fcaff9960 |
| wsrep_cluster_status         | Primary                              |
| wsrep_commit_oooe            | 0.000000                             |
| wsrep_commit_oool            | 0.000000                             |
| wsrep_commit_window          | 1.000000                             |
| wsrep_connected              | ON                                   |
~~~

`wsrep_cluster_size` = 3 表示集群当中现在有3个节点。

执行：

~~~sql
create database test_cluster;
~~~

然后在另一台机器中登陆：

~~~sh
mysql -uroot -p
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test_cluster       |
+--------------------+
4 rows in set (0.00 sec)
~~~

可以看到在另一个节点创建的数据库已经出现了。

