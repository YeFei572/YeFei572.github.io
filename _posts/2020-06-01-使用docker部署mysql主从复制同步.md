---
layout: post
tags: MySQL
categories: 技术分享
title:  "使用docker部署mysql主从复制同步；"
---

> 使用`docker`部署`mysql`主从同步，因为没钱买服务器， 所以用docker比较靠谱且省钱，主从同步可以做数据备份，可以读写分离等等好处，此文借助网上很多大神的博客，如有冒犯请联系。

### 一、准备工作：

要做主从复制这里肯定要准备`docker`， 还有镜像文件，本文以`mysql5.7`为例。以`Ubuntu18.04`服务器为准，centOS应该也没毛病，因为`docker`是通用的：

```bash
# 拉取镜像mysql， 如果已经有了镜像，此处命令跳过。
docker pull mysql:5.7
# 创建mysql的配置挂载地址
mkdir -p /data/mysql/config
# 创建mysql的存储文件挂载地址
mkdir -p /data/mysql/data_master
mkdir -p /data/mysql/data_slave
# 创建主从数据库的配置文件
touch /data/mysql/config/master.conf
touch /data/mysql/config/slave.conf
```

### 二、准备mysql的配置文件

- 准备主节点的配置文件：

```bash
vi /data/mysql/config/master.conf

# 以下是主节点的配置文件
[client]  
default-character-set=utf8 
[mysql]  
default-character-set=utf8 
[mysqld]  
log_bin = log #开启二进制日志，用于从节点的历史复制回放 
collation-server = utf8_unicode_ci 
init-connect='SET NAMES utf8'  
character-set-server = utf8 
server_id = 1 #需保证主库和从库的server_id不同， 假设主库设为1 
replicate-do-db=test #需要复制的数据库名，需复制多个数据库的话则重复设置这个选项
```

- 准备从节点的配置文件

```
vi /data/mysql/config/slave.conf

# 以下是从节点的配置文件
[client]  
default-character-set=utf8 
[mysql]  
default-character-set=utf8 
[mysqld]  
log_bin = log #开启二进制日志，用于从节点的历史复制回放 
collation-server = utf8_unicode_ci 
init-connect='SET NAMES utf8'  
character-set-server = utf8 
server_id = 2 #需保证主库和从库的server_id不同， 假设主库设为1 
replicate-do-db=test #需要复制的数据库名，需复制多个数据库的话则重复设置这个选项
```

### 三、启动主从节点容器
- 主节点启动：

```bash
docker run 
-d 
--name mysql-master 
-p 13306:3306 
-v /data/mysql/config/master.conf:/etc/mysql/mysql.conf.d/mysqld.cnf 
-v /data/mysql/data_master:/var/lib/mysql  
-e MYSQL_ROOT_PASSWORD=123456 mysql:5.7
```

参数说明：
> -d: 后台运行
--name mysql-master: 容器的名称设为mysql-master  
-p 13306:3306: 将host的13306端口映射到容器的3306端口  
-v /data/mysql/config/master.conf:/etc/mysql/mysql.conf.d/mysqld.cnf ： master.conf配置文件挂载  
-v /data/mysql/data_master:/var/lib/mysql ： mysql容器内数据挂载到host的/data/mysql/datam， 用于持久化  
-e MYSQL_ROOT_PASSWORD=123456 : mysql的root登录密码为123456

- 从节点启动：

```bash  
docker run 
-d 
--name mysql-slave 
-p 13307:3306 
-v /data/mysql/config/slave.conf:/etc/mysql/mysql.conf.d/mysqld.cnf 
-v /data/mysql/data_slave:/var/lib/mysql  
-e MYSQL_ROOT_PASSWORD=123456 mysql:5.7  
```

参数说明参考主节点的参数说明

### 四、完善主从节点的复制配置
先登录主节点配置

```bash
 # 容器id以实际为准
docker exec -it 【容器id】 bash  
mysql -uroot -p 123456
GRANT replication slave ON *.* TO 'slave'@'%' IDENTIFIED BY 'slave';
`flush privileges;`

# 该处的查询稍后要在从节点配置上使用到
show master status \G
*************************** 1. row ***************************
             File: log.000003
         Position: 1183
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)
```

说明：
![1234.png](https://pic.v2ss.cn/1234.png)


开始完善从节点的配置：

```bash
 # 容器id以实际为准
docker exec -it 【容器id】 bash  
mysql -uroot -p 123456
# 关闭从节点的slave
stop slave;
# 配置复制相关账号信息, 这里要注意， ip地址， log和pos都要按照上面的master的修改，确保一致
CHANGE MASTER TO MASTER_HOST='服务器的内网IP(172.18.142.xxx)',master_port=13306,MASTER_USER='slave',MASTER_PASSWORD='slave',MASTER_LOG_FILE='log.000003',MASTER_LOG_POS=1183;
# 启动slave
start slave;
# 查看从节点的slave状态
show slave status /G
# 如果查询结果中包含以下两项就表示成功了
Slave_IO_Running:  Yes  
Slave_SQL_Running:  Yes
```

### 五、校验是否成功：

在主节点中创建数据库
```
create database test
```

到从节点检查是否已经有test库，如果没有可以检查一下整个流程

### 六、总结容易踩坑的地方

- 1. 主从节点的配置中有限制哪个数据库可以同步，如果不是指定的数据库是无法同步的。
- 2. docker进入到指定的容器退出不要用`ctrl+c`，否则退出容器就停止了，使用`ctrl+p`和`ctrl+q`来退出容器。
- 3. 主节点创建子节点复制的账号记得密码和账号不要搞错了

### 七、鸣谢：

大量借鉴了该博文，谢谢作者
https://www.cnblogs.com/Draymonder/p/10993367.html
