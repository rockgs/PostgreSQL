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
[root@masterdb home]# adduser postgres -U -u 500
[root@masterdb home]# cd /home/postgres/
[root@masterdb postgres]# mkdir source
[root@masterdb postgres]# cd source/
[root@masterdb source]# wget https://ftp.postgresql.org/pub/source/v9.6.1/postgresql-9.6.1.tar.gz
[root@masterdb source]# tar zxf postgresql-9.6.1.tar.gz 
[root@masterdb source]# cd postgresql-9.6.1
[root@masterdb postgresql-9.6.1]# ./configure --prefix=/usr/local/pgsql9.6.1
[root@masterdb postgresql-9.6.1]# gmake -j 8
[root@masterdb postgresql-9.6.1]# gmake install
```
