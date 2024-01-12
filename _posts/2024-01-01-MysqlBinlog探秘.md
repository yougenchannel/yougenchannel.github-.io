---
title: Mysql Binlog 探秘
date: 2024-01-01 11:22:33
categories: [Mysql]
tags: [mysql] # TAG names should always be lowercase
---

# Mysql Binlog 探秘

## 前言

 因使用 cannel 增量同步 mysql 数据修改， 使用 java 模拟 Mysql Binlog 日志同步，旨在深入理解订阅 mysql binlog 后，同步数据过程

## 搭建 Mysql 环境

环境概述： 这里采用 docker + mysql:5.7.36 进行搭建，参考[12306 项目环境搭建](https://nageoffer.com/pages/008ee6/#%E6%95%B0%E6%8D%AE%E5%BA%93%E5%88%9D%E5%A7%8B%E5%8C%96)

- **先启动 Mysql， 复制一份配置文件**

```dockerfile
docker run --name mysql \
-p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7.36

# ~/docker/software/mysql/conf 是本地目录，没有的话需要创建
docker cp mysql:/etc/mysql/mysql.conf.d/mysqld.cnf ~/docker/software/mysql/conf

```

- `-d`：以后台的方式运行。
- `--name mysql`：指定容器的名称为 mysql。
- `-p3306:3306`：将容器的 3306 端口挂载到宿主机的 3306 端口上。
- `-e MYSQL_ROOT_PASSWORD=root`：指定 root 的密码为 root。
  打开 ~/docker/software/mysql/conf mysqld.cnf 文件，增加以下内容。

* 配置文件中添加如下内容，注意是在在[mysqld]目录中

```
log-bin=mysql-bin  # 开启 binlog
binlog-format=ROW  # 选择 ROW 模式
server-id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
```

- 删除原 MySQL 容器，通过新配置创建新的容器。

```
# 删除运行中的 MySQL 容器
docker rm -f mysql

# 运行 Docker 容器命令
# /etc/localtime 时间同步
# /docker/software/mysql/conf 同步配置文件，上面配置的内容就会覆盖容器中的配置文件
# /docker/software/mysql/log 同步日志目录
# /docker/software/mysql/data 同步 MySQL 的一些文件内容（对数据进行备份）
# MYSQL_ROOT_PASSWORD=root 默认 root 的密码是 root
docker run --name mysql \
-p 3306:3306 \
-v /etc/localtime:/etc/localtime \
-v ~/docker/software/mysql/conf:/etc/mysql/mysql.conf.d \
-v ~/docker/software/mysql/log:/var/log/mysql \
-v ~/docker/software/mysql/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7.36
```

- 进入到 MySQL 的命令行模式来给 root 账号授权所有 ip 能够访问。

```
# 使用 MySQL 容器中的命令行
docker exec -it mysql /bin/bash

# 使用 MySQL 命令打开客户端：
mysql -uroot -proot --default-character-set=utf8

# 接着创建一个账户，该账号所有 IP 都能够访问
grant all privileges on *.* to 'root' @'%' identified by 'root';

# 刷新生效
FLUSH PRIVILEGES;

# 查看 binlog 日志是否开启
show variables like 'log_%';

# 查看主结点当前状态
show master status;
```



* 至此， 我们搭建好了一个开启binlog日志的mysql



## 源码地址

[subscribe_binlog](https://github.com/yougenchannel/subscribe_binlog)
