# fast & safe upgrade to PostgreSQL 9.4 use pg_upgrade & zfs

### Author : Digoal zhou

### [更新]

已使用pg_upgrade顺利将一个8TB的生产数据库(包含表, 索引, 类型, 函数, 外部对象等对象大概10万个)从9.3升级到9.4, 升级比较快(约2分钟), 因为数据库较大后期analyze的时间比较长, 不过你可以将常用的表优先analyze一下, 就可以放心大胆的提供服务了.


PostgreSQL 9.4于昨天(2014-12-18)正式发布, 为了让大家可以快速的享受9.4带来的强大特性, 写一篇使用zfs和pg_upgrade升级9.4的快速可靠的文章. 希望对大家有帮助.
提醒:
在正式升级9.4前, 请做好功课, 至少release note要阅读一遍, 特别是兼容性. 例如有些应用可能用了某些9.4不兼容的语法或者插件的话, 需要解决了再上. (以前就有出现过版本升级带来的bytea的默认表述变更导致的程序异常)

pg_upgrade支持从8.3.x以及更新的版本的跨大版本升级, 使用LINK模式, 可以减少数据的拷贝工作, 大大提高版本升级的速度.
本文将演示一下使用pg_upgrade将数据库从9.3.5升级到最新的9.4.
使用zfs快照来保存老的数据文件和软件. 如果升级失败, 回滚非常简单, 回退到ZFS快照或者使用ZFS快照克隆都可以.

### 架构
![架构](https://github.com/rockgs/PostgreSQL/blob/master/Upgrade_to_PostgreSQL_9.4/pgupdate9.4.png)

升级步骤简介 : 
假设主机已是基于ZFS
  停库
  创建快照
  使用upgrade升级

假设主机不是基于ZFS
  创建ZFS主机
  创建standby
  主备角色切换
  以下基于新的主
  停主
  创建快照
  使用upgrade升级

如何把老版本的standby升级成为9.4 standby?
  pg start backup
  rsync 数据文件
  pg_stop_backup
  创建recovery.conf 继续.

使用ZFS和pg_upgrade升级9.4的详细步骤 : 
以CentOS 7 3.10.0-123.el7.x86_64内核版本为例
测试环境部署
安装zfs
加入YUM仓库

### 安装zfs文件系统:

```
[root@localhost ~]# yum localinstall --nogpgcheck http://download.zfsonlinux.org/epel/zfs-release.el7.noarch.rpm
[root@localhost ~]#  yum install -y epel-release.noarch 
[root@localhost ~]# uname -r
3.10.0-123.el7.x86_64
[root@localhost ~]# yum install  zfs 
[root@localhost /]# modprobe zfs
```

### 创建数据目录
```
[root@localhost /]# mkdir data01
[root@localhost /]# cd data01
```

### 安装好ZFS后, 创建ZPOOL, 我们使用5个文件来模拟5块磁盘。
```
[root@localhost /]# dd if=/dev/zero of=./disk1 bs=8192k count=1024 oflag=direct
[root@localhost /]# dd if=/dev/zero of=./disk2 bs=8192k count=1024 oflag=direct
[root@localhost /]# dd if=/dev/zero of=./disk3 bs=8192k count=1024 oflag=direct
[root@localhost /]# dd if=/dev/zero of=./disk4 bs=8192k count=1024 oflag=direct
[root@localhost /]# dd if=/dev/zero of=./disk5 bs=8192k count=1024 oflag=direct
```

### 创建zpool
```
[root@localhost /]# zpool create -o ashift=12 zp1 raidz /data01/disk1 /data01/disk2 /data01/disk3 /data01/disk4 /data01/disk5
[root@localhost /]# zpool status
  pool: zp1
 state: ONLINE
  scan: none requested
config:

        NAME               STATE     READ WRITE CKSUM
        zp1                ONLINE       0     0     0
          raidz1-0         ONLINE       0     0     0
            /data01/disk1  ONLINE       0     0     0
            /data01/disk2  ONLINE       0     0     0
            /data01/disk3  ONLINE       0     0     0
            /data01/disk4  ONLINE       0     0     0
            /data01/disk5  ONLINE       0     0     0

errors: No known data errors
```

### 设置zfs默认参数 
```
[root@localhost /]# zfs set atime=off zp1
[root@localhost /]# zfs set compression=lz4 zp1
[root@localhost /]# zfs set canmount=off zp1
```

### 规划一下数据库的目录结构.
假设分开5个文件系统来存放.
```
$PGDATA
pg_xlog
pg_arch
tbs1
tbs2
```

### 创建对应的zfs文件系统
```
[root@localhost /]# zfs create -o mountpoint=/pgdata01 zp1/pg_root
[root@localhost /]# zfs create -o mountpoint=/pgdata02 zp1/pg_xlog
[root@localhost /]# zfs create -o mountpoint=/pgdata03 zp1/pg_arch
[root@localhost /]# zfs create -o mountpoint=/pgdata04 zp1/tbs1
[root@localhost /]# zfs create -o mountpoint=/pgdata05 zp1/tbs2
[root@localhost /]# df -h
Filesystem      Size  Used Avail Use% Mounted on
zp1/pg_root     7.7G  128K  7.7G   1% /pgdata01
zp1/pg_xlog     7.7G  128K  7.7G   1% /pgdata02
zp1/pg_arch     7.7G  128K  7.7G   1% /pgdata03
zp1/tbs1        7.7G  128K  7.7G   1% /pgdata04
zp1/tbs2        7.7G  128K  7.7G   1% /pgdata05
```

### 创建数据目录
```
[root@localhost /]# mkdir /pgdata01/pg_root
[root@localhost /]# mkdir /pgdata02/pg_xlog
[root@localhost /]# mkdir /pgdata03/pg_arch
[root@localhost /]# mkdir /pgdata04/tbs1
[root@localhost /]# mkdir /pgdata05/tbs2
[root@localhost /]# chown -R postgres:postgres /pgdata0*
```

### 安装依赖库，安装PostgreSQL 9.3.5，初始化
```
[root@localhost ~]# tar -xvf postgresql-9.3.5.tar.gz 
[root@localhost ~]# yum -y install glib2 lrzsz sysstat e4fsprogs xfsprogs ntp readline-devel zlib zlib-devel openssl openssl-devel pam-devel libxml2-devel libxslt-devel python-devel tcl-devel gcc make smartmontools flex bison perl perl-devel perl-ExtUtils* OpenIPMI-tools openldap openldap-devel
[root@localhost postgresql-9.3.5]# ./configure --prefix=/opt/pgsql9.3.5 --with-pgport=1921 --with-perl --with-tcl --with-python --with-openssl --with-pam --without-ldap --with-libxml --with-libxslt --enable-thread-safety --with-blocksize=32 --with-wal-blocksize=32 && gmake world && gmake install-world
[root@localhost postgresql-9.3.5]# ln -s /opt/pgsql9.3.5 /opt/pgsql
[root@localhost postgresql-9.3.5]# vi /etc/ld.so.conf
/opt/pgsql/lib
[root@localhost postgresql-9.3.5]# ldconfig
[root@localhost postgresql-9.3.5]# ldconfig -p|grep /opt/pgsql
        libpqwalreceiver.so (libc6,x86-64) => /opt/pgsql/lib/libpqwalreceiver.so
        libpq.so.5 (libc6,x86-64) => /opt/pgsql/lib/libpq.so.5
        libpq.so (libc6,x86-64) => /opt/pgsql/lib/libpq.so
        libpgtypes.so.3 (libc6,x86-64) => /opt/pgsql/lib/libpgtypes.so.3
        libpgtypes.so (libc6,x86-64) => /opt/pgsql/lib/libpgtypes.so
        libecpg_compat.so.3 (libc6,x86-64) => /opt/pgsql/lib/libecpg_compat.so.3
        libecpg_compat.so (libc6,x86-64) => /opt/pgsql/lib/libecpg_compat.so
        libecpg.so.6 (libc6,x86-64) => /opt/pgsql/lib/libecpg.so.6
        libecpg.so (libc6,x86-64) => /opt/pgsql/lib/libecpg.so
[root@localhost postgresql-9.3.5]# vi /etc/profile
export PATH=/opt/pgsql/bin:$PATH
[root@localhost postgresql-9.3.5]# . /etc/profile
[root@localhost postgresql-9.3.5]# which psql
/opt/pgsql/bin/psql
[root@localhost postgresql-9.3.5]# which pg_config
/opt/pgsql/bin/pg_config
```

### 初始化数据库
```
# su - postgres
$ vi .bash_profile
export PS1="$USER@`/bin/hostname -s`-> "
export PGPORT=1921
export PGDATA=/pgdata01/pg_root
export LANG=en_US.utf8
export PGHOME=/opt/pgsql
export LD_LIBRARY_PATH=$PGHOME/lib:/lib64:/usr/lib64:/usr/local/lib64:/lib:/usr/lib:/usr/local/lib:$LD_LIBRARY_PATH
export DATE=`date +"%Y%m%d%H%M"`
export PATH=$PGHOME/bin:$PATH:.
export MANPATH=$PGHOME/share/man:$MANPATH
export PGUSER=postgres
export PGHOST=$PGDATA
export PGDATABASE=postgres
alias rm='rm -i'
alias ll='ls -lh'
postgres@localhost-> . ./.bash_profile
postgres@localhost-> initdb -D $PGDATA -U postgres -E UTF8 --locale=C -W -X /pgdata02/pg_xlog
```

### 修改配置文件, 开启归档
```
vi pg_hba.conf
host all all 0.0.0.0/0 md5

vi postgresql.conf
listen_addresses = '0.0.0.0'            # what IP address(es) to listen on;
port = 1921                             # (change requires restart)
max_connections = 100                   # (change requires restart)
superuser_reserved_connections = 3      # (change requires restart)
unix_socket_directories = '.'   # comma-separated list of directories
unix_socket_permissions = 0700          # begin with 0 to use octal notation
tcp_keepalives_idle = 60                # TCP_KEEPIDLE, in seconds;
tcp_keepalives_interval = 10            # TCP_KEEPINTVL, in seconds;
tcp_keepalives_count = 10               # TCP_KEEPCNT;
shared_buffers = 512MB                  # min 128kB
maintenance_work_mem = 512MB            # min 1MB
vacuum_cost_delay = 10                  # 0-100 milliseconds
vacuum_cost_limit = 10000               # 1-10000 credits
bgwriter_delay = 10ms                   # 10-10000ms between rounds
wal_level = hot_standby                 # minimal, archive, or hot_standby
synchronous_commit = off                # synchronization level;
wal_buffers = 16384kB                   # min 32kB, -1 sets based on shared_buffers
wal_writer_delay = 10ms         # 1-10000 milliseconds
checkpoint_segments = 32                # in logfile segments, min 1, 16MB each
archive_mode = on               # allows archiving to be done
archive_command = 'DIR="/pgdata03/pg_arch/`date +%F`";test -d $DIR || mkdir -p $DIR; cp %p $DIR/%f'               # command to use to archive a logfile segment
archive_timeout = 600           # force a logfile segment switch after this
effective_cache_size = 4096MB
log_destination = 'csvlog'              # Valid values are combinations of
logging_collector = on          # Enable capturing of stderr and csvlog
log_directory = 'pg_log'                # directory where log files are written,
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' # log file name pattern,
log_file_mode = 0600                    # creation mode for log files,
log_truncate_on_rotation = on           # If on, an existing log file with the
log_checkpoints = on
log_connections = on
log_disconnections = on
log_error_verbosity = verbose           # terse, default, or verbose messages
log_lock_waits = on                     # log lock waits >= deadlock_timeout
log_statement = 'ddl'                   # none, ddl, mod, all
log_timezone = 'PRC'
autovacuum = on                 # Enable autovacuum subprocess?  'on'
log_autovacuum_min_duration = 0 # -1 disables, 0 logs all actions and
datestyle = 'iso, mdy'
timezone = 'PRC'
lc_messages = 'C'                       # locale for system error message
lc_monetary = 'C'                       # locale for monetary formatting
lc_numeric = 'C'                        # locale for number formatting
lc_time = 'C'                           # locale for time formatting
default_text_search_config = 'pg_catalog.english'
```

### 启动数据库，创建测试用户
```
postgres@localhost-> pg_ctl start
postgres@localhost-> psql
postgres=# create role zgs login encrypted password 'zgs';
CREATE ROLE
```
### 创建表空间, 数据库
```
postgres=# create tablespace tbs1 location '/pgdata04/tbs1';
CREATE TABLESPACE
postgres=# create tablespace tbs2 location '/pgdata05/tbs2';
CREATE TABLESPACE
postgres=# create database zgs template template0 encoding 'UTF8' tablespace tbs1;
CREATE DATABASE
postgres=# grant all on database zgs to zgs;
GRANT
postgres=# grant all on tablespace tbs1 to zgs;
GRANT
postgres=# grant all on tablespace tbs2 to zgs;
GRANT
postgres=# \c zgs zgs
You are now connected to database "zgs" as user "zgs".
zgs=> create schema zgs;
CREATE SCHEMA
```

### 创建测试数据表, 函数, 创建在tbs1和tbs2.
```
zgs=> create table userinfo (id int primary key, info text, crt_time timestamp);
zgs=> \d userinfo
                Table "zgs.userinfo"
  Column  |            Type             | Modifiers 
----------+-----------------------------+-----------
 id       | integer                     | not null
 info     | text                        | 
 crt_time | timestamp without time zone | 
Indexes:
    "userinfo_pkey" PRIMARY KEY, btree (id)
    
zgs=> alter index userinfo_pkey set tablespace tbs2;
ALTER INDEX

zgs=> create or replace function f_zgs(i_id int) returns void as $$
declare
begin
  update userinfo set info=$_$Hello,I'm zgs.$_$||md5(random()::text), crt_time=now() where id=i_id;
  if not found then
    insert into userinfo(id,info,crt_time) values(i_id, $_$Hello,I'm zgs.$_$||md5(random()::text), now());
  end if; 
  return;
exception when others then
  return;
end;
$$ language plpgsql strict volatile;
CREATE FUNCTION
zgs=> select f_zgs(1);
 f_zgs 
-------
 
(1 row)

zgs=> select * from userinfo;
 id |                      info                      |          crt_time          
----+------------------------------------------------+----------------------------
  1 | Hello,I'm zgs.0ae9cbf18bfd21450dcd7cc3a2df038d | 2017-08-02 17:12:42.216111
(1 row)
```

### 生成测试数据
```
zgs=>  insert into userinfo select generate_series(2,10000000),'test',clock_timestamp();
INSERT 0 9999999
```

## 安装PostgreSQL 9.4:
注意编译参数一致性, 以及内部和外部扩展模块(内部模块gmake world gmake install-world会全部安装).
```
[root@localhost ~]# tar -jxvf postgresql-9.4.0.tar.bz2
[root@localhost postgresql-9.4.0]# cd postgresql-9.4.0
[root@localhost postgresql-9.4.0]#./configure --prefix=/opt/pgsql9.4.0 --with-pgport=1921 --with-perl --with-tcl --with-python --with-openssl --with-pam --without-ldap --with-libxml --with-libxslt --enable-thread-safety --with-blocksize=32 --with-wal-blocksize=32 && gmake world && gmake install-world
```

### 检查安装包含upgrade和upgrade库
```
[root@localhost lib]# ll|grep upgr
-rwxr-xr-x 1 root root   14352 Aug  2 02:23 pg_upgrade_support.so
[root@localhost lib]# ll /opt/pgsql9.4.0/bin/pg_upgrade 
-rwxr-xr-x 1 root root 116416 Aug  2 02:23 /opt/pgsql9.4.0/bin/pg_upgrade
```

### 创建新版本数据目录
如果我们要使用硬链接$PGDATA来加快升级速度的话, 那么新的集群$PGDATA要和老集群的$PGDATA在一个文件系统下,所以我们使用 /pgdata01/pg_root_9.4
初始化XLOG目录和arch目录(如果使用了定制的pg_log, 则还需初始化pg_log目录, 本例使用的是$PGDATA/pg_log, 所以无需创建pg_log)
```
[root@localhost lib]# mkdir /pgdata01/pg_root_9.4
[root@localhost lib]# chown -R postgres:postgres /pgdata01/pg_root_9.4
[root@localhost lib]# chmod 700 /pgdata01/pg_root_9.4
[root@localhost lib]# mkdir /pgdata02/pg_xlog_9.4
[root@localhost lib]# chown -R postgres:postgres /pgdata02/pg_xlog_9.4
[root@localhost lib]# chmod 700 /pgdata02/pg_xlog_9.4
[root@localhost lib]# mkdir /pgdata03/pg_arch_9.4
[root@localhost lib]# chown -R postgres:postgres /pgdata03/pg_arch_9.4
[root@localhost lib]# chmod 700 /pgdata03/pg_arch_9.4
```

### 初始化9.4数据库
```
[root@localhost lib]# su - postgres
Last login: Wed Aug  2 02:23:38 PDT 2017 on pts/2
postgres@/home/postgres ->  /opt/pgsql9.4.0/bin/initdb -D /pgdata01/pg_root_9.4 -X /pgdata02/pg_xlog_9.4 -E UTF8 --locale=C -U postgres -W
```

### 配置9.4集群
将pg_hba.conf改为和老实例的9.3一致,另外, 因为升级需要多次连接新老集群数据库实例, 所以修改为使用本地trust认证.
```
postgres@localhost-> vi /pgdata01/pg_root_9.4/pg_hba.conf
包含以下即可
# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
```

### 修改9.4实例数据库配置文件
注意使用不同的监听端口.（PostgreSQL 9.4新增了很多功能和参数, 本例一并提供了）
```
postgres@localhost->vi /pgdata01/pg_root_9.4/postgresql.conf
listen_addresses = '0.0.0.0'            # what IP address(es) to listen on;
port = 1922                             # (change requires restart)
max_connections = 100                   # (change requires restart)
unix_socket_directories = '.'   # comma-separated list of directories
unix_socket_permissions = 0700          # begin with 0 to use octal notation
tcp_keepalives_idle = 60                # TCP_KEEPIDLE, in seconds;
tcp_keepalives_interval = 10            # TCP_KEEPINTVL, in seconds;
tcp_keepalives_count = 10               # TCP_KEEPCNT;
shared_buffers = 512MB                  # min 128kB
huge_pages = try                        # on, off, or try
maintenance_work_mem = 512MB            # min 1MB
autovacuum_work_mem = -1                # min 1MB, or -1 to use maintenance_work_mem
dynamic_shared_memory_type = posix      # the default is the first option
vacuum_cost_delay = 10                  # 0-100 milliseconds
vacuum_cost_limit = 10000               # 1-10000 credits
bgwriter_delay = 10ms                   # 10-10000ms between rounds
wal_level = logical                     # minimal, archive, hot_standby, or logical
synchronous_commit = off                # synchronization level;
wal_buffers = 16384kB                   # min 32kB, -1 sets based on shared_buffers
wal_writer_delay = 10ms         # 1-10000 milliseconds
checkpoint_segments = 32                # in logfile segments, min 1, 16MB each
archive_mode = on               # allows archiving to be done
archive_command = 'DIR="/pgdata03/pg_arch_9.4/`date +%F`";test -d $DIR || mkdir -p $DIR; cp %p $DIR/%f'         # command to use to archive a logfile segment
archive_timeout = 600           # force a logfile segment switch after this
log_destination = 'csvlog'              # Valid values are combinations of
logging_collector = on          # Enable capturing of stderr and csvlog
log_directory = 'pg_log'                # directory where log files are written,
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' # log file name pattern,
log_file_mode = 0600                    # creation mode for log files,
log_truncate_on_rotation = on           # If on, an existing log file with the
log_checkpoints = on
log_connections = on
log_disconnections = on
log_error_verbosity = verbose           # terse, default, or verbose messages
log_lock_waits = on                     # log lock waits >= deadlock_timeout
log_statement = 'ddl'                   # none, ddl, mod, all
log_timezone = 'PRC'
autovacuum = on                 # Enable autovacuum subprocess?  'on'
log_autovacuum_min_duration = 0 # -1 disables, 0 logs all actions and
datestyle = 'iso, mdy'
timezone = 'PRC'
lc_messages = 'C'                       # locale for system error message
lc_monetary = 'C'                       # locale for monetary formatting
lc_numeric = 'C'                        # locale for number formatting
lc_time = 'C'                           # locale for time formatting
default_text_search_config = 'pg_catalog.english'
```

### 停库
```
postgres@localhost-> /opt/pgsql9.3.5/bin/pg_ctl stop -m fast -D /pgdata01/pg_root
postgres@localhost-> /opt/pgsql9.4.0/bin/pg_ctl stop -m fast -D /pgdata01/pg_root_9.4
```

### 创建9.3数据库的文件系统快照
```
[root@localhost ~]# zfs snapshot zp1/pg_root@pg9.3.5
[root@localhost ~]# zfs snapshot zp1/pg_xlog@pg9.3.5
[root@localhost ~]# zfs snapshot zp1/pg_arch@pg9.3.5
[root@localhost ~]# zfs snapshot zp1/tbs1@pg9.3.5
[root@localhost ~]# zfs snapshot zp1/tbs2@pg9.3.5
[root@localhost ~]# zfs list -t snapshot
```

### 使用9.4的pg_upgrade检测兼容性
```
postgres@/home/postgres -> mkdir upgrade_log
postgres@/home/postgres -> cd upgrade_log/
postgres@/home/postgres -> /opt/pgsql9.4.0/bin/pg_upgrade -b /opt/pgsql9.3.5/bin -B /opt/pgsql9.4.0/bin -d /pgdata01/pg_root -D /pgdata01/pg_root_9.4 -p 1921 -P 1922 -U postgres -j 8 -k -c
Performing Consistency Checks
-----------------------------
Checking cluster versions                                   ok
Checking database user is a superuser                       ok
Checking for prepared transactions                          ok
Checking for reg* system OID user data types                ok
Checking for contrib/isn with bigint-passing mismatch       ok
Checking for invalid "line" user columns                    ok
Checking for presence of required libraries                 ok
Checking database user is a superuser                       ok
Checking for prepared transactions                          ok

*Clusters are compatible*
```

### 验证兼容性正常, 可以正式升级了
```
postgres@localhost-> /opt/pgsql9.4.0/bin/pg_upgrade -b /opt/pgsql9.3.5/bin -B /opt/pgsql9.4.0/bin -d /pgdata01/pg_root -D /pgdata01/pg_root_9.4 -p 1921 -P 1922 -U postgres -j 8 -k -r -v

Creating script to analyze new cluster                      ok
Creating script to delete old cluster                       ok

Upgrade Complete
----------------
Optimizer statistics are not transferred by pg_upgrade so,
once you start the new server, consider running:
    analyze_new_cluster.sh

Running this script will delete the old cluster's data files:
    delete_old_cluster.sh
```

### 给了2个脚本, 用于收集统计信息和删除老集群
```
postgres@/home/postgres -> ll
total 356K
-rwx------ 1 postgres postgres  785 Aug  2 02:46 analyze_new_cluster.sh
-rwx------ 1 postgres postgres  114 Aug  2 02:46 delete_old_cluster.sh
-rw------- 1 postgres postgres  49K Aug  2 02:46 pg_upgrade_internal.log
-rw------- 1 postgres postgres 5.2K Aug  2 02:46 pg_upgrade_server.log
-rw------- 1 postgres postgres 213K Aug  2 02:46 pg_upgrade_utility.log
```

### 接下来要做的是启动新的数据库集群
```
postgres@localhost-> /opt/pgsql9.4.0/bin/pg_ctl start -D /pgdata01/pg_root_9.4
```

### 查看数据
```
postgres@localhost-> /opt/pgsql9.4.0/bin/psql -h 127.0.0.1 -p 1922 -U zgs zgs
postgres@/home/postgres -> /opt/pgsql9.4.0/bin/psql -h 127.0.0.1 -p 1922 -U zgs zgs
psql (9.4.0)
Type "help" for help.

zgs=> \dt
         List of relations
 Schema |   Name   | Type  | Owner 
--------+----------+-------+-------
 zgs    | userinfo | table | zgs
(1 row)

zgs=> \dx
                 List of installed extensions
  Name   | Version |   Schema   |         Description          
---------+---------+------------+------------------------------
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(1 row)

zgs=> select count(*) from userinfo ;
  count   
----------
 10000000
(1 row)
```
