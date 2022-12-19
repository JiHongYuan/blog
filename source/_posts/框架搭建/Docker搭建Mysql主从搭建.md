---
title: Docker搭建Mysql主从搭建
date: 2022-12-18 21:28
categories: 框架搭建
---

# 前言  
环境选择: `centos7`、`docker:20.10.21`、`mysql:8.0.31`。  
一般来说数据库的瓶颈基本都是IO, 基本不会出现一台服务器部署多个Mysql服务，本次只作为Demo测试及简单使用，切勿在生产环境部署。  

# 一、 镜像拉取
在<a href="https://hub.docker.com/_/mysql/tags">Docker Hub</a>中选择合适的Mysql版本（8.0.31）。
```shell
docker pull mysql:8.0.31
```

# 二、 准备配置
### 1. 手动创建相关配置
**准备以下文件夹**：
1. `/docker/mysql/master`
2. `/docker/mysql/slave1`
3. `/docker/mysql/slave1`

```shell
cd /docker/mysql/master
mkdir config
mkdir data
mkdir logs

cd config
mkdir conf.d
vim conf.d/my.cnf

# 将以上结构复制到slave1、slave2文件夹中
```
**my.cnf文件内容如下：**
```shell
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/8.0/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M

# Remove leading # to revert to previous value for default_authentication_plugin,
# this will increase compatibility with older clients. For background, see:
# https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_default_authentication_plugin
# default-authentication-plugin=mysql_native_password
skip-host-cache
skip-name-resolve
datadir=/var/lib/mysql
socket=/var/run/mysqld/mysqld.sock
secure-file-priv=/var/lib/mysql-files
user=mysql

pid-file=/var/run/mysqld/mysqld.pid

## 设置server_id，一般设置为IP，注意要唯一
server_id=1
## 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql
## 开启二进制日志功能，可以随便取，最好有含义（关键就是这里了）
log-bin=replicas-mysql-bin
## 为每个session分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed
## 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062

[client]
socket=/var/run/mysqld/mysqld.sock

!includedir /etc/mysql/conf.d/
```
**修改Slave配置：**
```shell
# slave1
[mysqld]
## 设置server_id,注意要唯一
server-id=2  
## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
log-bin=mysql-slave-bin   
## relay_log配置中继日志
relay_log=edu-mysql-relay-bin  

# slave2
[mysqld]
## 设置server_id,注意要唯一
server-id=3  
## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
log-bin=mysql-slave-bin   
## relay_log配置中继日志
relay_log=edu-mysql-relay-bin  
```

### 2. 创建Mysql容器cp
略, 大部分和**1**相同，只是从容器中复制出`conf.d`文件夹，根据`mysql`版本可能存在目录位置不一样。

## 三、创建容器
**1. master**  
```shell
docker run -d --name mysql-master \
    --privileged=true \
    -p 17717:3306 \
    -v /docker/mysql/master/data:/var/lib/mysql \
    -v /docker/mysql/master/logs:/var/log/mysql \
    -v /docker/mysql/master/conf:/etc/mysql \
    -e MYSQL_ROOT_PASSWORD=123456 \
    -d mysql:8.0.31
```
**2. slave1**  
```shell
docker run -d --name mysql-slave1 \
   --privileged=true \
   -p 17718:3306 \
   -v /docker/mysql/slave1/data:/var/lib/mysql \
   -v /docker/mysql/slave1/logs:/var/log/mysql \
   -v /docker/mysql/slave1/conf:/etc/mysql \
   -e MYSQL_ROOT_PASSWORD=123456 \
   -d mysql:8.0.31
```

**3. slave2**  
```shell
docker run -d --name mysql-slave2 \
    --privileged=true \
    -p 17719:3306 \
    -v /docker/mysql/slave2/data:/var/lib/mysql \
    -v /docker/mysql/slave2/logs:/var/log/mysql \
    -v /docker/mysql/slave2/conf:/etc/mysql \
    -e MYSQL_ROOT_PASSWORD=123456 \
    -d mysql:8.0.31
```

# 四、绑定主从关系
```shell
change master to master_host='IP',master_user='root',master_password='',master_port=17717,master_log_file='replicas-mysql-bin.000001',master_log_pos=0;
flush privileges;
start slave;
# 设置只读
set global read_only=1;
```

# 五、总结
总体下来搭建并不复杂，只需要几行配置命令即可，实际上还是花了大半天的时间。  
总结下来：文档没有仔细阅读、docker并不熟悉导致的结果。