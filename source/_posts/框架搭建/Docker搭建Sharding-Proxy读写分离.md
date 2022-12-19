---
title: Docker搭建Sharding-Proxy读写分离
date: 2022-12-18 21:28
categories: 框架搭建
---

# 前言  
环境选择: `centos7`、`docker:20.10.21`、`mysql:8.0.31`、`sharding-proxy:latest`。  
相对于`mysql`主从搭建，`sharding-proxy`搭建时确实花了写功夫，再次总结并反思。
# 一、 镜像拉取
在<a href="https://hub.docker.com/r/apache/sharding-proxy/tags">Docker Hub</a>中选择合适的版本。
```shell
docker pull apache/sharding-proxy:latest
```
# 二、准备配置
搭建前，一定要仔细的看完<a href="https://shardingsphere.apache.org/document/5.3.0/cn/user-manual/shardingsphere-proxy/startup/docker/">官方文档</a>，下载<a href="https://github.com/apache/shardingsphere/tree/master/examples/shardingsphere-proxy-example/shardingsphere-proxy-boot-mybatis-example">官方Example</a>。  
**准备以下文件**：
1. `/docker/sharding-proxy/conf`
```yaml
# server.yaml
authority:
  users:
    - user: root
      password: root
    - user: sharding
      password: sharding
  privilege:
    type: ALL_PERMITTED

props:
  max-connections-size-per-query: 1
  kernel-executor-size: 16  # Infinite by default.
  proxy-frontend-flush-threshold: 128  # The default value is 128.
  proxy-hint-enabled: false
  sql-show: false
  check-table-metadata-enabled: false
```
```yaml
# config-readwrite-nacos-config.yaml
databaseName: nacos_config

dataSources:
  primary_ds:
    url: jdbc:mysql://127.0.0.1:17717/nacos_config?allowPublicKeyRetrieval=true&serverTimezone=UTC&useSSL=false
    username: root
    password: 123456
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
  replica_ds_0:
    url: jdbc:mysql://127.0.0.1:17718/nacos_config?allowPublicKeyRetrieval=true&serverTimezone=UTC&useSSL=false
    username: root
    password: 123456
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
  replica_ds_1:
    url: jdbc:mysql://127.0.0.1:17719/nacos_config?allowPublicKeyRetrieval=true&serverTimezone=UTC&useSSL=false
    username: root
    password: 123456
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1


rules:
  - !READWRITE_SPLITTING
    dataSources:
      readwrite_ds:
        staticStrategy:
          writeDataSourceName: primary_ds
          readDataSourceNames:
            - replica_ds_0
            - replica_ds_1
```
2. `/docker/sharding-proxy/ext-lib`  
将`mysql jbdc`驱动放进去。  
# 三、运行容器
```shell
docker run -d \
    -v /docker/sharding-proxy/conf:/opt/shardingsphere-proxy/conf \
    -v /docker/sharding-proxy/ext-lib:/opt/shardingsphere-proxy/ext-lib \
    -p 3306:3307 \
    apache/shardingsphere-proxy:latest
```
# 四、总结
看文档，看文档，看文档真的很重要。