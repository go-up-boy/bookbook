# Docker部署mysql主从架构

* 系统 Windows 11 WSL2
* 镜像 Mysql:8.0

# 基础知识
### 1.Binlog 解析
* binlog即binary log，二进制日志文件。它记录了数据库所有执行的DDL和DML语句(除了数据查询语句select、show等)，以事件形式记录并保存在二进制文件中
* binlog主要有两个应用场景
  * 主从复制，master把它的二进制日志传递给slave来达到master-slave数据一致的目的
  * 数据恢复，还原备份后，可以重新执行备份后新产生的binlog，使得数据库保持最新状态
* binlog日志可以选择三种模式，分别是STATEMENT、ROW、MIXED，下面简单介绍下这三种模式
  * statement：基于SQL语句的复制，每一条会修改数据的SQL语句会记录到binlog中。该模式下产生的binlog日志量会比较少，但可能导致主从数据不一致。
  * row：基于行的复制，不记录每一条具体执行的SQL语句，仅需记录哪条数据被修改了，以及修改前后的样子。该模式下产生的binlog日志量会比较大，但优点是会非常清楚的记录下每一行数据修改的细节，主从复制不会出错
  * mixed：混合模式复制，以上两种模式的混合使用，一般的复制使用statement模式保存binlog，对于statement模式无法复制的操作使用row模式保存binlog，MySQL会根据执行的SQL语句选择日志保存方式
* binlog模式在MySQL 5.7.7之前，默认为statement，在之后的版本中，默认为row模式。因为row模式更安全，可以清楚记录每行数据修改的细节

## Master 节点配置
1. 运行master容器:
- --privileged=true 赋予容器内root权限
- -v 容器卷映射 本地:容器
- MYSQL_ROOT_PASSWORD 必须设置 mysql root 用户密码
- -p 3306:3306 映射端口 本地:容器
- --name 设置容器名称
```shell
docker run -d -p 3306:3306 --privileged=true -v C:/Users/z/mysql/log:/var/log -v C:/Users/z/mysql/data:/var/lib/mysql -v C:/Users/z/mysql/conf:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 --name mysql mysql:8.0
```

2. 启动后在映射的容器卷 C:/Users/z/mysql/conf 创建my.cnf配置文件，并重启容器
```
[client]
default_character_set=utf8mb4
[mysqld]
collation_server = utf8mb4_general_ci
character_set_server = utf8mb4
[mysqld]
## 服务器唯一ID，可以任意设置，局域网但必须唯一
server-id=128
## 启用二进制日志
log-bin=mysql-bin
## 设置二进制日志使用内存大小（事务）
binlog_cache_size=1M
## 设置使用的二进制日志格式(mixed，statement,row)
binlog_format=row
## 二进制日志过期清理时间，默认0 不自动清理
expire_logs_days=0
## 不要复制的数据库
binlog-ignore-db=mysql
## 要复制的数据库名
##binlog-do-db=testdb
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断
## 如：1062 错误是主键重复，1032错误是  主从数据库数据不一致
slave_skip_errors=1062
```
3. 进入容器，创建mysql用户并且赋予读写权限
```
// 创建用户
create user 'slave'@'%' identified by '123456';
// 赋予权限
grant replication slave, replication client on *.* to 'slave'@'%';
```

## Slave 节点配置
1. 运行从节点容器
```shell
docker run -d -p 3307:3306 --privileged=true -v C:/Users/z/mysql_slave/log:/var/log -v C:/Users/z/mysql_slave/data:/var/lib/mysql -v C:/Users/z/mysql_slave/conf:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 --name mysql_slave mysql:8.0
```
2. 从机映射的容器卷 C:/Users/z/mysql_slave/conf 创建my.cnf配置文件
```
[client]
default_character_set=utf8mb4
[mysqld]
collation_server = utf8mb4_general_ci
character_set_server = utf8mb4
[mysqld]
## 服务器唯一ID，可以任意设置，局域网但必须唯一
server-id=129
## 启用二进制日志
log-bin=mall-mysql-slave1-bin
## 设置二进制日志使用内存大小（事务）
binlog_cache_size=1M
## 设置使用的二进制日志格式(mixed，statement,row)
binlog_format=row
## 二进制日志过期清理时间，默认0 不自动清理
expire_logs_days=0
## 不要复制的数据库
binlog-ignore-db=mysql
## 要复制的数据库名
##binlog-do-db=testdb
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断
## 如：1062 错误是主键重复，1032错误是  主从数据库数据不一致
slave_skip_errors=1062
## relay_log 配置中继日志
relay_log=mall-mysql-relay-bin
## log_slave_updates表示slave将复制事件写进自己的二进制日志
log_slave_updates=1
## slave设置为只读 具有super权限用户除外
read_only=1
```
3. 进入到从机容器配置主从复制
```
 change master to master_host='192.168.1.12', master_user='slave', master_password='123456', master_port=3306, master_log_file='mysql-bin.000003', master_log_pos=157, master_connect_retry=30, get_master_public_key=1;
```
4. 查看从库状态
```
show slave status \G;
// 此时 主从复制已创建，但是并未开启同步
       Slave_IO_State:
          Master_Host: 192.168.1.12
          Master_User: slave
          Master_Port: 3306
        Connect_Retry: 30
      Master_Log_File: mysql-bin.000003
  Read_Master_Log_Pos: 157
       Relay_Log_File: b20f034a1453-relay-bin.000001
        Relay_Log_Pos: 4
Relay_Master_Log_File: mysql-bin.000003
     Slave_IO_Running: No
    Slave_SQL_Running: No
```
5. 启动从库进行主从同步
```
5. start slave;
```

6. Mysql8.0 会出现下面的报错
- > error connecting to master 'slave@192.168.1.12:3306' - retry-time: 30 retries: 7 message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.

- >在MySQL8.0之前，身份验证的插件是mysql_native_password，在MySQL 8.0中，caching_sha2_password 是默认的身份验证插件，安全性更高。

- >在MySQL中，系统状态变量Rsa_public_key,此值是sha256_password身份验证插件用于基于RSA密钥对的密码交换的公用密钥 。对于使用该sha256_password 插件的客户端，连接到服务器时，密码永远不会以明文形式公开。密码传输的方式取决于是否使用安全连接或RSA加密：

- > 最优解决方案：在从节点使用复制用户 登录 master 节点，获取公钥
  > ```
  > mysql -h 192.168.1.12 -uslave -p --get-server-public-key
  > ```
  > 然后 start slave 即可
