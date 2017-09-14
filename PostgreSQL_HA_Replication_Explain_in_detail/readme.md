<h1 align = "center">Postgresql主备复制详解</h1>

#### 原作者：PostgreSQL中国社区---阿弟

## 一、服务器架构图
### 1、服务器架构图

![架构](https://github.com/rockgs/PostgreSQL/blob/master/PostgreSQL_HA_Replication_Explain_in_detail/pg_replication_01.png)

### 2、主备复制说明
* 支持一主多备
* 只能是一主一备同步，其它备节点复制为异步
* 支持级联复制，目前级联复制只能异步复制
* 支持archive和复制混合使用
* 支持主备切换
* 支持同步模式时，多个备机互相接替

## 二、部署环境介绍
### 1、主节点
|项目 | 值 | 
| ----------------- |:-----------------------:|
| 操作系统          | RedFlag Asia 4.4 x86-64 | 
| IP地址            | 192.168.245.141        | 
| HostName          | masterdb               |  
| PostgreSQL版本号   | 9.6.3                  |  

### 2、备节点
|项目               | 值                      | 
| ----------------- |:-----------------------:|
| 操作系统          | RedFlag Asia 4.4 x86-64 | 
| IP地址            | 192.168.245.142        | 
| HostName          | slavedb               |  
| PostgreSQL版本号   | 9.6.3                  | 

### 3、级联节点
|项目               | 值                      | 
| ----------------- |:-----------------------:|
| 操作系统          | RedFlag Asia 4.4 x86-64 | 
| IP地址            | 192.168.245.143        | 
| HostName          | casecade               |  
| PostgreSQL版本号   | 9.6.3                  | 

### 4、NFS节点
|项目               | 值                      | 
| ----------------- |:-----------------------:|
| 操作系统          | RedFlag Asia 4.4 x86-64 | 
| IP地址            | 192.168.245.144        | 
| HostName          | nfs                    |  
| PostgreSQL版本号   | 9.6.3                  | 

## 三、主节点部署及配置
### 1、源码编译安装Postgresql
```
[root@masterdb home]# adduser postgres -d /home/postgres
[root@localhost ~]# tar -zxvf postgresql-9.6.3.tar.gz 
[root@masterdb source]# cd postgresql-9.6.3
[root@masterdb postgresql-9.6.3]# ./configure --prefix=/opt/pgsql9.6.3 --with-pgport=1963 --with-perl --with-tcl --with-python --with-openssl --with-pam --without-ldap --with-libxml --with-libxslt --enable-thread-safety --with-blocksize=32 --with-wal-blocksize=32
[root@masterdb postgresql-9.6.3]# gmake world
[root@localhost postgresql-9.6.3]# gmake install-world
```
### 2、 postgres用户.bash_profile基本设置
```
export PGDATABASE="postgres"
export PGUSER=postgres
export PGPORT=1963
export PGHOME=/opt/pgsql9.6.3
export PGDATA=/home/postgres/data63
export PATH=$PATH:$HOME/bin:$PGHOME/bin
alias pg_start="pg_ctl start -l /home/postgres/log/pg_server.log"
alias pg_stop="pg_ctl stop -l /home/postgres/log/pg_server.log"
PATH=$PATH:$HOME/bin
export PATH
```
### 3、initdb一个节点
```
[root@masterdb postgresql-9.6.3]# su postgres
[postgres@masterdb postgresql-9.6.3]$initdb --no-locale -E utf8 -D /home/postgres/data63 -U postgres -W
```
### 4、配置运行参数
修改postgresql.conf
\#预写式日志级别，要归档的话replica或者是logical，如没必需的话使用replica即可，logical会产生更多的日志量
```
[postgres@masterdb ~]$ cd data63
[postgres@masterdb data63]$ vi postgresql.conf
listen_addresses = '*'
port = 9610
wal_level = replica 
synchronous_commit=on 
max_wal_enders = 3 #视自己业务需要调整
wal_keep_segments = 8000 #视自己业务需要调整
hot_standby = on
log_destination = 'csvlog'
logging_collector = on
log_directory = 'pg_log' #下面配置记录所有日志，对io的要求非常的高，线上如果开启记录所有日志，
 ##最好有独立的io设备来保存，否则对业务影响比较大，如果你有其它地方记录了日志，建议这里只收集必要
 ##的日志
log_checkpoints = on
log_connections = on
log_disconnections = on
log_duration = on
log_line_prefix = '%m %a %r %d %u %p %x'
log_min_duration_statement = 1000
log_statement = 'all'
log_timezone = 'PRC'
```
synchronous_commit参数说明
\#参数值为on、remote_write、local和off，如果synchronous_standby_names被设置时，这个参数也控制事务提交是否将等待事务的 WAL 记录被复制到后备服务器上。on表示后备服务器已经收到了事务的提交记录并将其刷入了磁盘，主服务器上的事务才会提交；remote_write后备服务器的一个回复指示该服务器已经收到了该事务的提交记录并且已经把该记录写出到后备服务器的操作系统，但是该数据并不一定到达了后备服务器上的稳定存储；设置local可以用于希望等待本地刷写磁盘但不等待同步复制的事务。当设置为off时，在向客户端报告成功和真正保证事务不会被服务器崩溃威胁之间会有延迟，（最大的延迟是wal_writer_delay的三倍）。不同于fsync，将这个参数设置为off不会产生数据库不一致性的风险：一个操作系统或数据库崩溃可能会造成一些最近数据已提交的事务丢失，但数据库状态是一致的，就像这些事务已经被干净地中止。因此，当性能比完全确保事务的持久性更重要时，关闭synchronous_commit可以作为一个有效的代替手段。

更多的流复制参数配置见
http://www.postgres.cn/docs/9.6/runtime-config-replication.html

更多的预写式日志配置见
http://www.postgres.cn/docs/9.6/runtime-config-wal.html

修改pg_hba.conf
```
[postgres@masterdb data63]$ vi pg_hba.conf 
# TYPE  DATABASE        USER            ADDRESS              METHOD

# "local" is for Unix domain socket connections only
#local   all             all                                  trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
host    all             all             192.168.245.0/24        md5
# IPv6 local connections:
#host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                trust
host    replication     repuser        192.168.245.0/24           md5
#host    replication     postgres        ::1/128                 trust
```
连接防火墙视自己业务需要配置
### 5、启动服务
```
[postgres@masterdb ~]$ pg_start 
server starting
[postgres@masterdb ~]$ 
```
### 6、创建复制角色
```
[postgres@masterdb ~]$ psql -h masterdb -U postgres -d postgres
postgres=# create role repuser with replication login
CREATE ROLE
postgres=#\password repuser
Enter new password: 
Enter it again:
```
### 7、添加PostgreSQL服务port至iptables规则中
在/etc/sysconfig/iptables规则表中添加上下面这条规则即可让1963 port即提供对外访问
```
[root@masterdb postgresql-9.6.3]# vim /etc/sysconfig/iptables
-A INPUT -m state --state NEW -m tcp -p tcp --dport 9610 -j ACCEPT
```
重启iptables服务
```
[root@masterdb postgresql-9.6.3]# service iptables restart
```
配置某个网段可以访问
```
[root@masterdb ~]# vim /etc/sysconfig/iptables
-A INPUT -s 192.168.245.0/24 -j ACCEPT
[root@masterdb ~]# service iptables restart
```

配置iptables开机时不自动启动
```
[root@masterdb log]# chkconfig iptables off
```
### 8、配置PostgreSQL服务开机自动启动
在 /etc/rc.d/rc.local  文件中添加下面启动脚本
```
[root@masterdb postgresql-9.6.3]# vim /etc/rc.d/rc.local
su postgres -c "/usr/local/pgsql9.6.1/bin/pg_ctl start -D /home/postgres/data63"
```

## 四、异步复制配置
### 1、源码编译安装Postgresql
