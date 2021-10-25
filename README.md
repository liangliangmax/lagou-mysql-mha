1. ## 环境软件版本

   | 环境&软件          | 版本   |
   | ------------------ | ------ |
   | VMware Workstation | 15 Pro |
   | CentOS             | 7.8    |
   | Mysql              | 5.7.28 |
   |                    |        |

2. ## 环境架构介绍

   架构如图所示，4台机器的IP和角色如下表：

   | 机器名称     | IP            | 角色         | 权限         |
   | ------------ | ------------- | ------------ | ------------ |
   | Mysql_Master | 172.16.62.216 | 数据库Master | 可读写、主库 |
   | Mysql_Slave1 | 172.16.62.217 | 数据库Slave  | 只读、从库   |
   | Mysql_Slave2 | 172.16.62.218 | 数据库Slave  | 只读、从库   |
   | Mysql_MHA    | 172.16.62.220 | MHA Manager  | 高可用监控   |

   

3. ## MySQL主从搭建

   #### 3.1 MySQL安装（3台）

   ##### 下载

   ```
   wget https://cdn.mysql.com/archives/mysql-5.7/mysql-5.7.28-1.el7.x86_64.rpmbundle.tar
   ```

   ##### 解压

   ```
   tar xvf mysql-5.7.28-1.el7.x86_64.rpm-bundle.tar
   ```

   ##### 安装

   要移除CentOS自带的mariadb-libs，不然会提示冲突

   ```
   rpm -qa|grep mariadb
   rpm -e mariadb-libs-5.5.41-2.el7_0.x86_64 --nodeps
   ```

   由于MySQL的server服务依赖了common、libs、client，所以需要按照以下顺序依次安装。 RPM是Red Hat公司随Redhat Linux推出的一个软件包管理器，通过它能够更加方便地实现软件的安 装。rpm常用的命令有以下几个:

   ```
   -i, --install 安装软件包
   -v, --verbose 可视化，提供更多的详细信息的输出
   -h, --hash 显示安装进度
   -U, --upgrade=<packagefile>+ 升级软件包
   -e, --erase=<package>+ 卸载软件包
   --nodeps 不验证软件包的依赖
   ```

   组合可得到几个常用命令：

   ```
   安装软件：rpm -ivh rpm包名
   升级软件：rpm -Uvh rpm包名
   卸载软件：rpm -e rpm包名
   查看某个包是否被安装 rpm -qa | grep 软件名称
   ```

   下面就利用安装命令来安装mysql：

   ```
   rpm -ivh mysql-community-common-5.7.28-1.el7.x86_64.rpm
   rpm -ivh mysql-community-libs-5.7.28-1.el7.x86_64.rpm
   rpm -ivh mysql-community-libs-compat-5.7.28-1.el7.x86_64.rpm
   rpm -ivh mysql-community-client-5.7.28-1.el7.x86_64.rpm
   rpm -ivh mysql-community-server-5.7.28-1.el7.x86_64.rpm
   rpm -ivh mysql-community-devel-5.7.28-1.el7.x86_64.rpm
   ```

   ##### 启动

   初始化用户

   ```
   mysqld --initialize --user=root
   ```

   查看初始密码

   ```
   cat /var/log/mysqld.log | grep password
   ```

   启动mysql服务

   ```
   systemctl start mysqld
   ```

   配置为开机启动

   ```
   systemctl enable mysqld
   ```

   接下来登录mysql，修改默认密码。

   ```
   mysql -uroot -p
   xxxxxx输入初始密码
   mysql> SET PASSWORD = PASSWORD('root');
   Query OK, 0 rows affected, 1 warning (0.00 sec)
   ```

   #### 3.2 关闭防火墙

   不同的MySQL直接要互相访问，需要关闭Linux的防火墙，否则就要在配置/etc/sysconfig/iptables中增 加规则。配置防火墙不是本次作业的重点，所以四台服务器均关闭防火墙。

   ```
   systemctl stop firewalld
   ```

   #### 3.3 MySQL主从配置

   ##### Master节点

   使用vi /etc/my.cnf命令修改Master配置文件

   ```
   #bin_log配置
   log_bin=mysql-bin
   server-id=1
   sync-binlog=1
   binlog-ignore-db=information_schema
   binlog-ignore-db=mysql
   binlog-ignore-db=performance_schema
   binlog-ignore-db=sys
   #relay_log配置
   relay_log=mysql-relay-bin
   log_slave_updates=1
   relay_log_purge=0
   ```

   ##### 重启服务

   ```
   systemctl restart mysqld
   ```

   ##### 主库给从库授权

   登录MySQL，在MySQL命令行执行如下命令：

   ```
   mysql> grant replication slave on *.* to root@'%' identified by 'root';
   mysql> grant all privileges on *.* to root@'%' identified by 'root';
   mysql> flush privileges;
   //查看主库状态信息，例如master_log_file='mysql-bin.000007',master_log_pos=154
   mysql> show master status;
   ```

   ##### Slave节点

   修改Slave的MySQL配置文件my.cnf，两台Slave的server-id分别设置为2和3

   ```
   #bin_log配置
   log_bin=mysql-bin
   #服务器ID,从库1是2,从库2是3
   server-id=2
   sync-binlog=1
   binlog-ignore-db=information_schema
   binlog-ignore-db=mysql
   binlog-ignore-db=performance_schema
   binlog-ignore-db=sys
   #relay_log配置
   relay_log=mysql-relay-bin
   log_slave_updates=1
   relay_log_purge=0
   read_only=1
   ```

   ##### 重启服务

   ```
   systemctl restart mysqld
   ```

   ##### 开启同步

   登录MySQL，在Slave节点的MySQL命令行执行同步操作，例如下面命令（注意参数与上面show master status操作显示的参数一致）：

   ```
   change master to
   master_host='172.16.62.216',master_port=3306,master_user='root',master_password
   ='root',master_log_file='mysql-bin.000007',master_log_pos=154;
   
   start slave; // 开启同步
   ```

   #### 3.4 配置半同步复制

   ##### Master节点

   登录MySQL，在MySQL命令行执行下面命令安装插件

   ```
   install plugin rpl_semi_sync_master soname 'semisync_master.so';
   show variables like '%semi%';
   ```

   使用vi /etc/my.cnf，修改MySQL配置文件

   ```
   # 自动开启半同步复制
   rpl_semi_sync_master_enabled=ON
   rpl_semi_sync_master_timeout=1000
   ```

   重启MySQL服务

   ```
   systemctl restart mysql
   ```

   ##### Slave节点

   两台Slave节点都执行以下步骤。 

   登录MySQL，在MySQL命令行执行下面命令安装插件

   ```
   install plugin rpl_semi_sync_slave soname 'semisync_slave.so
   ```

   使用vi /etc/my.cnf，修改MySQL配置文件

   ```
   # 自动开启半同步复制
   rpl_semi_sync_slave_enabled=ON
   ```

   重启服务

   ```
   systemctl restart mysqld
   ```

   ##### 测试半同步状态 

   首先通过MySQL命令行检查参数的方式，查看半同步是否开启。

   ```
   show variables like '%semi%';
   ```

   然后通过MySQL日志再次确认。

   ```
   cat /var/log/mysqld.log
   ```

   可以看到日志中已经启动半同步信息，例如：

   ```
   Start semi-sync binlog_dump to slave (server_id: 2), pos(mysql-bin.000005, 154)
   ```

   

4. ### MHA高可用搭建

   #### 四台机器ssh互通

   在四台服务器上分别执行下面命令，生成公钥和私钥（注意：连续按换行回车采用默认值）

   ```
   ssh-keygen -t rsa
   ```

   在三台MySQL服务器分别执行下面命令，密码输入系统密码，将公钥拷到MHA Manager服务器上

   ```
   ssh-copy-id 172.16.62.220
   ```

   之后可以在MHA Manager服务器上检查下，看看.ssh/authorized_keys文件是否包含3个公钥

   ```
   cat /root/.ssh/authorized_keys
   ```

   执行下面命令，将MHA Manager的公钥添加到authorized_keys文件中（此时应该包含4个公钥）

   ```
   cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
   ```

   从MHA Manager服务器执行下面命令，向其他三台MySQL服务器分发公钥信息。

   ```
   scp /root/.ssh/authorized_keys root@172.16.62.216:/root/.ssh/authorized_keys
   scp /root/.ssh/authorized_keys root@172.16.62.217:/root/.ssh/authorized_keys
   scp /root/.ssh/authorized_keys root@117.16.62.218:/root/.ssh/authorized_keys
   ```

   可以MHA Manager执行下面命令，检测下与三台MySQL是否实现ssh互通。

   ```
   ssh 172.16.62.216
   exit
   ssh 172.16.62.217
   exit
   ssh 172.16.62.218
   exit
   ```

   #### MHA下载安装

   ##### MHA下载

   MySQL5.7对应的MHA版本是0.5.8，所以在GitHub上找到对应的rpm包进行下载，MHA manager和 node的安装包需要分别下载：

   ```
   https://github.com/yoshinorim/mha4mysql-manager/releases/tag/v0.58
   https://github.com/yoshinorim/mha4mysql-node/releases/tag/v0.58
   ```

   下载后，将Manager和Node的安装包分别上传到对应的服务器。（可使用WinSCP等工具）

   - 三台MySQL服务器需要安装node 

     - MHA Manager服务器需要安装manager和node

   ```
   提示：也可以使用wget命令在linux系统直接下载获取，例如
   wget https://github.com/yoshinorim/mha4mysqlmanager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
   ```

   ##### MHA node安装

   在四台服务器上安装mha4mysql-node。 

   MHA的Node依赖于perl-DBD-MySQL，所以要先安装perl-DBD-MySQL。

   ```
   yum install perl-DBD-MySQL -y
   wget https://github.com/yoshinorim/mha4mysqlnode/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
   rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
   ```

   ##### MHA manager安装

   在MHA Manager服务器安装mha4mysql-node和mha4mysql-manager。 

   MHA的manager又依赖了perl-Config-Tiny、perl-Log-Dispatch、perl-Parallel-ForkManager，也分别 进行安装。

   ```
   wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
   rpm -ivh epel-release-latest-7.noarch.rpm
   yum install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-ParallelForkManager -y
   wget https://github.com/yoshinorim/mha4mysqlnode/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
   rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
   wget https://github.com/yoshinorim/mha4mysqlmanager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
   rpm -ivh mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
   ```

   提示：由于perl-Log-Dispatch和perl-Parallel-ForkManager这两个被依赖包在yum仓库找不到， 因此安装epel-release-latest-7.noarch.rpm。在使用时，可能会出现下面异常：Cannot retrieve metalink for repository: epel/x86_64。可以尝试使 用/etc/yum.repos.d/epel.repo，然后注释掉metalink，取消注释baseurl。

   

   ##### MHA 配置文件

   MHA Manager服务器需要为每个监控的 Master/Slave 集群提供一个专用的配置文件，而所有的 Master/Slave 集群也可共享全局配置。

   

   初始化配置目录

   ```
   #目录说明
   #/var/log (CentOS目录)
   # /mha (MHA监控根目录)
   # /app1 (MHA监控实例根目录)
   # /manager.log (MHA监控实例日志文件)
   mkdir -p /var/log/mha/app1
   touch /var/log/mha/app1/manager.log
   ```

   配置监控全局配置文件

   vim /etc/masterha_default.cnf

   ```
   [server default]
   #主库用户名，在master mysql的主库执行下列命令建一个新用户
   #create user 'mha'@'%' identified by '123456';
   #grant all on *.* to mha@'%' identified by '123456';
   #flush privileges;
   user=mha
   password=123456
   port=3306
   #ssh登录账号
   ssh_user=root
   #从库复制账号和密码
   repl_user=root
   repl_password=root
   port=3306
   #ping次数
   ping_interval=1
   #二次检查的主机
   secondary_check_script=masterha_secondary_check -s 172.16.62.216 -s
   172.16.62.217 -s 172.16.62.218
   
   ```

   配置监控实例配置文件

   先使用 mkdir -p /etc/mha 命令创建目录，然后使用 vim /etc/mha/app1.cnf 命令编辑文件

   ```
   [server default]
   #MHA监控实例根目录
   manager_workdir=/var/log/mha/app1
   #MHA监控实例日志文件
   manager_log=/var/log/mha/app1/manager.log
   #[serverx] 服务器编号
   #hostname 主机名
   #candidate_master 可以做主库
   #master_binlog_dir binlog日志文件目录
   [server1]
   hostname=172.16.62.216
   candidate_master=1
   master_binlog_dir="/var/lib/mysql"
   [server2]
   hostname=172.16.62.217
   candidate_master=1
   master_binlog_dir="/var/lib/mysql"
   [server3]
   hostname=172.16.62.218
   candidate_master=1
   master_binlog_dir="/var/lib/mysql"
   ```

   ##### MHA 配置检测

   执行ssh通信检测

   在MHA Manager服务器上执行：

   ```
   masterha_check_ssh --conf=/etc/mha/app1.cnf
   ```

   检测MySQL主从复制

   在MHA Manager服务器上执行：

   ```
   masterha_check_repl --conf=/etc/mha/app1.cn
   ```

   出现“MySQL Replication Health is OK.”证明MySQL复制集群没有问题。

   

   ##### MHA Manager启动

   在MHA Manager服务器上执行：

   ```
   nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --
   ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &
   ```

   查看监控状态命令如下：

   ```
   masterha_check_status --conf=/etc/mha/app1.cnf
   ```

   查看监控日志命令如下：

   ```
   tail -f /var/log/mha/app1/manager.log
   ```

   

   #### 测试MHA故障转移

   模拟主节点崩溃 

   在MHA Manager服务器执行打开日志命令：

   ```
   tail -200f /var/log/mha/app1/manager.log
   ```

   关闭Master MySQL服务器服务，模拟主节点崩溃

   ```
   systemctl stop mysqld
   ```

   查看MHA日志，可以看到哪台slave切换成了master

   ```
   show master status;
   ```

   

   #### 测试SQL脚本

   ```
   create TABLE position (
   id int(20),
   name varchar(50),
   salary varchar(20),
   city varchar(50)
   ) ENGINE=innodb charset=utf8;
   ```

   ```
   insert into position values(1, 'Java', 13000, 'shanghai');
   insert into position values(2, 'DBA', 20000, 'beijing');
   ```

   ```
   create TABLE position_detail (
   id int(20),
   pid int(20),
   description text
   ) ENGINE=innodb charset=utf8;
   ```

   ```
   insert into position_detail values(1, 1, 'Java Developer');
   insert into position_detail values(2, 2, 'Database Administrator');
   ```

   

   

   

   

   

   

   










































































