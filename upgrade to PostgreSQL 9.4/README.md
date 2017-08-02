
# fast & safe upgrade to PostgreSQL 9.4 use pg_upgrade & zfs

### [更新]

已使用pg_upgrade顺利将一个8TB的生产数据库(包含表, 索引, 类型, 函数, 外部对象等对象大概10万个)从9.3升级到9.4, 升级比较快(约2分钟), 因为数据库较大后期analyze的时间比较长, 不过你可以将常用的表优先analyze一下, 就可以放心大胆的提供服务了.


PostgreSQL 9.4于昨天(2014-12-18)正式发布, 为了让大家可以快速的享受9.4带来的强大特性, 写一篇使用zfs和pg_upgrade升级9.4的快速可靠的文章. 希望对大家有帮助.
提醒:
在正式升级9.4前, 请做好功课, 至少release note要阅读一遍, 特别是兼容性. 例如有些应用可能用了某些9.4不兼容的语法或者插件的话, 需要解决了再上. (以前就有出现过版本升级带来的bytea的默认表述变更导致的程序异常)

pg_upgrade支持从8.3.x以及更新的版本的跨大版本升级, 使用LINK模式, 可以减少数据的拷贝工作, 大大提高版本升级的速度.
本文将演示一下使用pg_upgrade将数据库从9.3.5升级到最新的9.4.
使用zfs快照来保存老的数据文件和软件. 如果升级失败, 回滚非常简单, 回退到ZFS快照或者使用ZFS快照克隆都可以.

### 架构
![架构](https://github.com/rockgs/PostgreSQL/blob/master/upgrade%20to%20PostgreSQL%209.4/pgupdate9.4.png)

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
[root@localhost disks]# dd if=/dev/zero of=./disk1 bs=8192k count=1024 oflag=direct
[root@localhost disks]# dd if=/dev/zero of=./disk2 bs=8192k count=1024 oflag=direct
[root@localhost disks]# dd if=/dev/zero of=./disk3 bs=8192k count=1024 oflag=direct
[root@localhost disks]# dd if=/dev/zero of=./disk4 bs=8192k count=1024 oflag=direct
[root@localhost disks]# dd if=/dev/zero of=./disk5 bs=8192k count=1024 oflag=direct
```

### 创建zpool
```
[root@localhost disks]# zpool create -o ashift=12 zp1 raidz /data01/disk1 /data01/disk2 /data01/disk3 /data01/disk4 /data01/disk5
[root@localhost disks]# zpool status
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
[root@localhost disks]# zfs set atime=off zp1
[root@localhost disks]# zfs set compression=lz4 zp1
[root@localhost disks]# zfs set canmount=off zp1
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

### 安装PostgreSQL 9.3.5，初始化
```













