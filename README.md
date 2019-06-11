pg集群搭建-20190607
一，方案分析
最近研究了PG的两种集群方案，分别是Pgpool-II和Postgres-XL，在这里总结一下二者的机制、结构、优劣、测试结果等。
1、 Pgpool-II和Postgres-XL简介
据我目前的了解，Pgpool-II和Postgres-XL是PG集群开源实现中比较成功的两个项目，其中Pgpool-II的前身的Pgpool-I，Postgres-XL的前身是Postgres-XC。
1.1、Pgpool-II
Pgpool-II相当于中间件，位于应用程序和PG服务端之间，对应用程序来说，Pgpool-II就相当于PG服务端；对PG服务端来说，Pgpool-II相当于PG客户端。由此可见，Pgpool-II与PG是解耦合的，基于这样的机制，Pgpool-II可以搭建在已经存在的任意版本的PG主从结构上，主从结构的实现与Pgpool-II无关，可以通过slony等工具或者PG自身的流复制机制实现。除了主从结构的集群，Pgpool-II也支持多主结构，称为复制模式，该模式下PG节点之间是对等的，没有主从关系，写操作同时在所有节点上执行，这种模式下写操作的代价很大，性能上不及主从模式。PG 9.3之后支持的流复制机制可以方便的搭建主从结构的集群（包括同步复制与异步复制），因此Pgpool-II中比较常用的模式是流复制主从模式（也可以一主多从）。


既然PG可以通过自身的流复制机制方便的搭建主从结构集群，为什么还要在它上面搭建Pgpool-II呢？因为简单的主从结构集群并不能提供连接池、负载均衡、自动故障切换等功能，Pgpool-II正好可以做到这些，当然负载均衡只针对读操作，写操作只发生在主节点上。为了避免单点故障，Pgpool-II自身也可以配置为主从结构，对外提供虚拟IP地址，当主节点故障后，从节点提升为新的主节点并接管虚拟IP。
1.2、Postgres-XL
Postgres-XL的机制和Pgpool-II大不相同，它不是独立于PG的，是在PG源代码的基础上增加新功能实现的。简单来说，Postgres-XL将PG的SQL解析层的工作和数据存取层的工作分离到不同的两种节点上，分别称为Coordinator节点和Datanode节点，而且每种节点可以配置多个，共同协调完成原本单个PG实例完成的工作。此外，为了保证分布模式下事务能够正确执行，增加了一个GTM节点。为了避免单点故障，可以为所有节点配置对应的slave节点。Postgres-XL结构图见下图，来自官网。


Postgres-XL的Coordinator节点是整个集群的数据访问入口，可以配置多个，然后在它们之上通过Nginx等工具实现负载均衡。Coordinator节点维护着数据的存储信息，但不存储数据本身。接收到一条SQL语句后，Coordinator解析SQL，制定执行计划，然后分发任务到相关的Datanode上，Datanode返回执行结果到Coordinator，Coordinator整合各个Datanode返回的结果，最后返回给客户端。
Postgres-XL的Datanode节点负责实际存取数据，数据在多个Datanode上的分布有两种方式：复制模式和分片模式，复制模式下，一个表的数据在指定的节点上存在多个副本；分片模式下，一个表的数据按照指定的规则分布在多个数据节点上，这些节点共同保存一份完整的数据。这两种模式的选择是在创建表的时候执行CREATE TABLE语句指定的，也可以通过ALTER TABLE语句改变数据的分布方式。
2、 Pgpool-II和Postgres-XL对比


3、 Pgpool-II和Postgres-XL的性能测试
我分别使用pgbench和benchmarksql测试了Pgpool-II集群和Postgres-XL集群的性能，为了对比，还测试单机PG的性能。
测试条件：Pgpool-II集群是搭建在两台虚机上的主从复制（异步）集群；Postgres-XL集群也是搭建在相同条件上的两台虚机的集群，其中包含两个Coordinator节点和两个Datanode节点。单机PG也是运行在相同条件的虚机上。操作系统是CentOS 6.6，单机PG和Pgpool-II集群种的PG版本号是9.5，Postgres-XL的版本号是Postgres-XL 9.5 R1.3，也只基于PG 9.5的。
3.1、pgbench测试
pgbench是PG自带的一款简单的PG性能测试工具，测试指标是TPS，表示每秒钟完成的事务数。测试过程如下：
1) 建库
psql -h 10.192.33.244 -p7777 -c "create database pgbench"
2) 生成数据
pgbench -i -s 1000 -h 10.192.33.244 -p 7777 pgbench 
#参数-s指定数据量，这里使用1000，最终生成的数据量大小约16G。
3) 测试
pgbench -h 10.192.33.244 -p7777  -c30 -T300 -n 
#测试时间5分钟，连续测试3次。 
pgbench测试结果：


pgbench的测试结果显示，Pgpool-II集群的性能比单机PG的性能差一些，约为84%；Postgres-XL集群的性能比单机PG的性能好一些，约为137%。
3.2、benchmarksql测试
benchmarksql的是一款常用的TPC-C测试工具，TPC-C测试衡量的是数据库的OLTP性能。测试过程如下：
1) 建库
psql -h 10.192.33.244 -p7777 -c "create database tpcc"
2) 生成数据
./runDatabaseBuild.sh props.pg
#props.pg为配置文件，配置数据库链接信息以及测试数据量、测试时间等，
#这里配置的数据量是100 warehouse，最终生成的数据约10G，测试时间1小时。
3) 测试
./runBenchmark.sh props.pg
benchmarksql测试结果：


benchmarksql测试结果显示，两种集群与单机PG的性能指标几乎一致，无法分辨高下。出现这种结果的可能原因之一是：测试数据量较小，无法发挥集群的性能优势，尤其像Postgres-XL这个集群在设计上针对大数据处理做了一些优化，应该更加适合大数据处理的场景。鉴于benchmarksql测试生成数据十分耗时，这里就不再进行较大数据量的测试了。
最后，综合来看，我更倾向于Postgres-XL，如果公司今后打算用的话，我会推介。

二，准备环境搭建
1搭建linux环境
选取centos7做系统
root：123456
1.1永久修改主机名：hostnamectl set-hostname postgres1/2/3
  		重启系统reboot
查看hostname： cat /etc/hostname 或者 hostname
1.2设置网卡：ifconfig 查看虚拟机网关名称ens160 发现没有ip地址
 			cat /etc/sysconfig/network-scripts/ifcfg-ens160修改网络配置
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens32
UUID=29c76cc1-6688-404f-8537-81a3561c3bf9
DEVICE=ens32
ONBOOT=yes
IPADDR=192.168.2.130
NETMASK=255.255.255.0
GATEWAY=192.168.2.1
DNS1=210.22.70.3
DNS2=8.8.8.8

/etc/init.d/network restart
ping 内网外网地址 正常 用xshell连接查看正常
os: centos 7
pgxl:pg.version '10.3 (Postgres-XL 10alpha2)
pgxl 是一款非常实用的横向扩展的开源软件，继承了很多pgxc的功能，在replication 和sharding 方面有着非常棒的用处。
pgxl 不严格的说是 pgxc的升级加强版。是对官方 postgresql 的版本的修改提升，为大牛点赞。
Global Transaction Monitor (GTM)
全局事务管理器，确保群集范围内的事务一致性。 GTM负责发放事务ID和快照作为其多版本并发控制的一部分。
集群可选地配置一个备用GTM，以改进可用性。此外，可以在协调器间配置代理GTM， 可用于改善可扩展性，减少GTM的通信量。
GTM Standby
GTM的备节点，在pgxc,pgxl中，GTM控制所有的全局事务分配，如果出现问题，就会导致整个集群不可用，为了增加可用性，增加该备用节点。当GTM出现问题时，GTM Standby可以升级为GTM，保证集群正常工作。
GTM-Proxy
GTM需要与所有的Coordinators通信，为了降低压力，可以在每个Coordinator机器上部署一个GTM-Proxy。
Coordinator
协调员管理用户会话，并与GTM和数据节点进行交互。协调员解析，并计划查询，并给语句中的每一个组件发送下一个序列化的全局性计划。
为节省机器，通常此服务和数据节点部署在一起。
Data Node
数据节点是数据实际存储的地方。数据的分布可以由DBA来配置。为了提高可用性，可以配置数据节点的热备以便进行故障转移准备。
总结：
gtm是负责ACID的，保证分布式数据库全局事务一致性。得益于此，就算数据节点是分布的，但是你在主节点操作增删改查事务时，就如同只操作一个数据库一样简单。
Coordinator是调度的，将操作指令发送到各个数据节点。
datanodes是数据节点，分布式存储数据。

规划如下：
node1 192.168.2.164 gtm
node2 192.168.2.165 gtm-proxy,coordinator,datanode
node3 192.168.2.166 gtm-proxy,coordinator,datanode

下载
Postgres-XL 9.5 R1.6发布 - 2017年8月24日
Postgres-XL 10  R1.1 发布2019年2月18日
官方瞎咋i地址https://www.postgres-xl.org/

https://git.postgresql.org/gitweb/?p=postgres-xl.git;a=summary
git://git.postgresql.org/git/postgres-xl.git

三.安装集群环境
以下操作除了 mkdir {gtm,gtm_slave,pgxc_ctl} 命令外都需要在node2、node3 上执行
node1 需要安装依赖包
# yum install -y bison flex perl-ExtUtils-Embed readline-devel zlib-devel pam-devel libxml2-devel libxslt-devel openldap-devel python-devel gcc gcc-c++ openssl-devel cmake openjade docbook-style-dsssl uuid uuid-devel

node1 节点上关闭防火墙，selinux
# systemctl stop firewalld.service 
# systemctl disable firewalld.service 
# vim /etc/selinux/config
disabled

node1 节点上创建用户
# groupadd postgres
# useradd postgres -g postgres 
# passwd postgres

# mkdir -p /usr/pgxl-10
# chown -R postgres:postgres /usr/pgxl-10

# mkdir -p /var/lib/pgxl
# cd /var/lib/pgxl
# mkdir {gtm,gtm_slave,pgxc_ctl}
# chown -R postgres:postgres /var/lib/pgxl

修改host文件
[root@postgres1 ~]# vi /etc/hosts
添加各个节点地址
192.168.2.164 node1
192.168.2.165 node2
192.168.2.166 node3


node1 节点 postgres 用户的环境变量
# su - postgres
$ vi ~/.bash_profile
export PGUSER=postgres
export PGHOME=/usr/pgxl-10
export PGXC_CTL_HOME=/var/lib/pgxl/pgxc_ctl

export LD_LIBRARY_PATH=$PGHOME/lib
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/lib:/usr/lib:/usr/local/lib
export PATH=$PGHOME/bin:$PATH

export TEMP=/tmp
export TMPDIR=/tmp

export PS1="\[\e[32;1m\][\u@\h \W]$>\[\e[0m\]"

node1 上编译安装
$ cd /tmp
$ git clone git://git.postgresql.org/git/postgres-xl.git
$ cd postgres-xl
$ git branch -r
  origin/HEAD -> origin/master
  origin/XL9_5_STABLE
  origin/XL_10_STABLE
  origin/master
  origin/xl_dbt3_expt
  origin/xl_doc_update
  origin/xl_test
$ git checkout XL_10_STABLE
Branch XL_10_STABLE set up to track remote branch XL_10_STABLE from origin.
Switched to a new branch 'XL_10_STABLE'
$ git status
# On branch XL_10_STABLE
nothing to commit, working directory clean  
$ ./configure --prefix=/usr/pgxl-10 --with-perl --with-python --with-openssl --with-pam --with-ldap --with-libxml --with-libxslt
$ make 
$ make install
$ cd contrib
$ make 
$ make install


2.另外在 node2、node3节点上还需要运行如下命令
# su - postgres
$ cd /var/lib/pgxl
$ mkdir {gtm_proxy}
$ mkdir {coord,coord_slave,coord_archlog}
$ mkdir {dn_master,dn_slave,dn_archlog}

node1、node2、node3配置ssh相互免密登录
su - pgxl

ssh-keygen -t rsa

cat ~/.ssh/id_rsa.pub>> ~/.ssh/authorized_keys

chmod 600 ~/.ssh/authorized_keys

将刚生成的认证文件拷贝到另外2台服务器：

scp ~/.ssh/authorized_keys postgres@192.168.2.165:~/.ssh/

scp ~/.ssh/authorized_keys postgres@192.168.2.166:~/.ssh/

node1、node2、node3同步下时间
# ntpdate asia.pool.ntp.org


node1,node2,node3 节点修改环境变量（~/.bashrc和~/.bash_profile都要修改
/etc/hosts
ps -aux | grep pgxl
echo $PGHOME

[postgres@postgres2 ~]$>vi ~/.bashrc
# .bashrc
export PGUSER=postgres
export PGHOME=/usr/pgxl-10
export PGXC_CTL_HOME=/var/lib/pgxl/pgxc_ctl

export LD_LIBRARY_PATH=$PGHOME/lib
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/lib:/usr/lib:/usr/local/lib
export PATH=$PGHOME/bin:$PATH

export TEMP=/tmp
export TMPDIR=/tmp

export PS1="\[\e[32;1m\][\u@\h \W]$>\[\e[0m\]"

# Source global definitions
if [ -f /etc/bashrc ]; then
        . /etc/bashrc
fi

# Uncomment the following line if you don't like systemctl's auto-paging feature:
# export SYSTEMD_PAGER=

# User specific aliases and functions

##############################################################################
[postgres@postgres2 ~]$>vim ~/.bash_profile
# .bash_profile
export PGUSER=postgres
export PGHOME=/usr/pgxl-10
export PGXC_CTL_HOME=/var/lib/pgxl/pgxc_ctl

export LD_LIBRARY_PATH=$PGHOME/lib
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/lib:/usr/lib:/usr/local/lib
export PATH=$PGHOME/bin:$PATH

export TEMP=/tmp
export TMPDIR=/tmp

export PS1="\[\e[32;1m\][\u@\h \W]$>\[\e[0m\]"

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs

#PATH=$PATH:$HOME/.local/bin:$HOME/bin

#export PATH



不修改环境会出现 command not found 的错误提示
这里是依赖 ~/.bashrc
Current directory: /var/lib/pgxl/pgxc_ctl
Initialize GTM master
ERROR: target directory (/var/lib/pgxl/gtm) exists and not empty. Skip GTM initilialization
bash: gtm: command not found
bash: gtm_ctl: command not found
Done.
Start GTM master
bash: gtm_ctl: command not found
Initialize GTM slave
bash: initgtm: command not found


pgxc_ctl 生成配置文件
$ which pgxc_ctl
/usr/pgxl-10/bin/pgxc_ctl

$ pgxc_ctl prepare
/bin/bash
Installing pgxc_ctl_bash script as /var/lib/pgxl/pgxc_ctl/pgxc_ctl_bash.
ERROR: File "/var/lib/pgxl/pgxc_ctl/pgxc_ctl.conf" not found or not a regular file. No such file or directory
Installing pgxc_ctl_bash script as /var/lib/pgxl/pgxc_ctl/pgxc_ctl_bash.
Reading configuration using /var/lib/pgxl/pgxc_ctl/pgxc_ctl_bash --home /var/lib/pgxl/pgxc_ctl --configuration /var/lib/pgxl/pgxc_ctl/pgxc_ctl.conf
Finished reading configuration.
   ******** PGXC_CTL START ***************

Current directory: /var/lib/pgxl/pgxc_ctl

$ ls -l /var/lib/pgxl/pgxc_ctl
total 24
-rw-r--r-- 1 postgres postgres   246 Jul 18 16:42 coordExtraConfig
-rw-r--r-- 1 postgres postgres 17815 Jul 18 16:42 pgxc_ctl.conf



pgxc_ctl 修改配置文件: $ vi /var/lib/pgxl/pgxc_ctl/pgxc_ctl.conf
#
