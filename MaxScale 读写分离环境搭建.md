# MaxScale 读写分离环境搭建



[TOC]



## 搭建MySQL主从复制环境，至少一主一从



### 创建复制用户并授权



主库从库上都执行这个命令

```mysql
create user repl identified by 'repl';
grant replication slave,replication client on *.* to repl@'%';
```



检查一下有没有创建成功

```mysql
show grants for repl;

+--------------------------------------------------------------------------------------------+
| Grants for repl@%                                                                          |
+--------------------------------------------------------------------------------------------+
| GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'%' IDENTIFIED BY PASSWORD '*A424E797037BF97C19A2E88CF7891C5C2038C039'                                                  |
+--------------------------------------------------------------------------------------------+
```



查看一下当前主库binlog位置

```mysql
show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000003 |      339 |              |                  |
+------------------+----------+--------------+------------------+
```



备库开启复制

```mysql
change master to  master_host='172.16.28.218', master_user='repl', master_password='repl', master_log_file='mysql-bin.000003', master_log_pos=339;
start slave;
show slave status\G

*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.28.218
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 339
               Relay_Log_File: localhost-relay-bin.000002
                Relay_Log_Pos: 253
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: performance_schema
           Replicate_Do_Table: 
....


```



至此，一主一从的最简单的MySQL集群环境搭建完毕。



## 安装MaxScale

从[**官网**](https://mariadb.com/products/mariadb-maxscale)下载最新的安装包。

我下载的是[2.0.2版本](https://downloads.mariadb.com/MaxScale/2.0.2/rhel/6/x86_64/maxscale-2.0.2-1.rhel.6.x86_64.rpm)，安装包很小，只有2.42M。



建议和MySQL数据库安装在不同的服务器上，但是如果没有多余的服务器（比如我），可以安装在其中一台MySQL所在的服务器上。

我安装在主库`172.16.28.218`所在的服务器上。



```shell
rpm -ivh maxscale-2.0.2-1.rhel.6.x86_64.rpm
```



主目录在`/usr/share/maxscale/`

```shell
ll -h /usr/share/maxscale/
total 76K
-rw-r--r--. 1 root root 4.1K Nov 22 09:22 cdc_schema.go
-rw-r--r--. 1 root root 3.7K Nov 22 09:34 Changelog.txt
-rw-r--r--. 1 root root  386 Nov 22 09:21 COPYRIGHT
-rw-r--r--. 1 root root 3.1K Nov 22 09:21 LICENSE.TXT
-rw-r--r--. 1 root root  839 Nov 22 09:21 lsyncd_example.conf
-rwxr-xr-x. 1 root root 4.2K Nov 22 09:34 maxscale
-rw-r--r--. 1 root root 3.2K Nov 22 09:22 maxscale_binlogserver_template.cnf
-rw-r--r--. 1 root root   20 Nov 22 09:34 maxscale.conf
-rw-r--r--. 1 root root  347 Nov 22 09:34 maxscale.service
-rw-r--r--. 1 root root 2.1K Nov 22 09:22 maxscale_template.cnf
drwxr-xr-x. 3 root root 4.0K Dec  1 00:34 plugins
-rwxr-xr-x. 1 root root 2.2K Nov 22 09:34 postinst
-rwxr-xr-x. 1 root root  611 Nov 22 09:34 postrm
-rw-r--r--. 1 root root 2.2K Nov 22 09:22 README
-rw-r--r--. 1 root root 6.0K Nov 22 09:34 ReleaseNotes.txt
-rw-r--r--. 1 root root 1.8K Nov 22 09:34 UpgradingToMaxScale12.txt
```



配置文件在`/etc/`目录下

```shell
ll -h /etc/max*
-rw-r--r--. 1 root root 2.1K Dec  1 00:34 /etc/maxscale.cnf
-rw-r--r--. 1 root root 2.1K Nov 22 09:22 /etc/maxscale.cnf.template
```

`maxscale.cnf.template`是配置文件模板，初始状态下`maxscale.cnf.template`和`maxscale.cnf`文件内容是完全一样的，我们直接修改`maxscale.cnf`文件就可以。



日志文件在`/var/log/maxscale`



以上目录都可以在MaxScale配置文件中改变位置



## 配置MaxScale

### 创建用户



在开始配置之前，需要在**主库**中为`MaxScale`创建两个用户，用于监控模块和路由模块。



用于路由的用户我们称为maxrouter

```mysql
create user maxrouter identified by 'maxscale';

grant SELECT on mysql.user to 'maxrouter'@'%' identified by 'maxscale';
GRANT SELECT ON mysql.db TO 'maxrouter'@'%' identified by 'maxscale';
GRANT SELECT ON mysql.tables_priv TO 'maxrouter'@'%' identified by 'maxscale';
GRANT SHOW DATABASES ON *.* TO 'maxrouter'@'%' identified by 'maxscale';
```



查一下用户创建的情况

```mysql
show grants for maxrouter;
+----------------------------------------------------------+
| Grants for maxrouter@%                                   |
+----------------------------------------------------------+
| GRANT SHOW DATABASES ON *.* TO 'maxrouter'@'%'           |
| GRANT SELECT ON `mysql`.`db` TO 'maxrouter'@'%'          |
| GRANT SELECT ON `mysql`.`tables_priv` TO 'maxrouter'@'%' |
| GRANT SELECT ON `mysql`.`user` TO 'maxrouter'@'%'        |
+----------------------------------------------------------+

```



注：如果grant时限制了ip域， 如`'172.16.28.%'`，直接`show grants for router`会出现错误

>   ERROR 1141 (42000): There is no such grant defined for user 'maxrouter' on host '%'

需要加上授权时候使用的ip域



如果不能登录，试着再给用于localhost的授权。（我还没找到更优雅的处理方式）



用于监控的用户我们称为maxmonitor

```mysql
create user maxmonitor identified by 'maxscale';

grant REPLICATION CLIENT on *.* to 'maxmonitor'@'localhost';
```



检查一下创建情况

```mysql
show grants for maxmonitor\G
*************************** 1. row ***************************
Grants for maxmonitor@%: GRANT REPLICATION CLIENT ON *.* TO 'maxmonitor'@'%' IDENTIFIED BY PASSWORD '*53E5D0C7885BD540911663B04133F20C94AD4306'
```



其实完全可以只创建一个用户，我们这里为了整明白些，根据功能创建不同的用户名。



### 修改配置文件

修改`/etc/maxscale.cnf`文件，文件是ini标准的，我们逐个section来讨论。



#### server section

每一个section都对应后端的一个MySQL服务器实例（也可以是MariaDB），section name可以随便起

```ini
[master-server]
type=server
address=172.16.28.218
port=3306
protocol=MySQLBackend

[slave-server]
type=server
address=172.16.28.217
port=3306
protocol=MySQLBackend
```



#### monitor section

server填上我们上面的两个服务器，user信息填我们刚刚创建的maxmonitor

```ini
[MySQL Monitor]
type=monitor
module=mysqlmon
servers=master-server,slave-server
user=maxmonitor
passwd=maxscale
monitor_interval=10000
```



#### service section

service表示提供的服务类型，比如读写分离，用户是maxrouter

```ini
[Read-Write Split Service]
type=service
router=readwritesplit
servers=master-server,slave-server
user=maxrouter
passwd=maxscale
max_slave_connections=100%
```



#### listener section

listener配置端口和服务的绑定关系

```ini
[Read-Write Split Listener]
type=listener
service=Read-Write Split Service
protocol=MySQLClient
port=4006
```



其他内容默认就好



### 启动MaxScale

```shell
service maxscale start
```



用netstat查看端口情况，会发现我们`listener`中配置的端口，但是不代表MaxScale可以正常提供服务了。

检查日志情况，如果全是notice，恭喜你，MaxScale已经配置成功，可以进行下一步测试了。

如果有error，就要根据错误信息的情况进行修改，我在配置过程中遇到的大部分是创建的用户无法登陆mysql，这时候需要重新检查用户授权情况，一般是调整允许用户登录的ip域。





## 测试



```shell
mysql -umaxrouter -pmaxscale -h172.16.28.218 -P4006
```



可以通过下面的命令查看MaxScale的运行状态

```shell
maxadmin list services

Services.
--------------------------+----------------------+--------+---------------
Service Name              | Router Module        | #Users | Total Sessions
--------------------------+----------------------+--------+---------------
Read-Write Split Service  | readwritesplit       |      1 |    12
MaxAdmin Service          | cli                  |      2 |     2
--------------------------+----------------------+--------+---------------

```



```shell
maxadmin list servers

Servers.
-------------------+-----------------+-------+-------------+--------------------
Server             | Address         | Port  | Connections | Status              
-------------------+-----------------+-------+-------------+--------------------
master-server      | 172.16.28.218   |  3306 |           0 | Master, Running
slave-server       | 172.16.28.217   |  3306 |           0 | Slave, Running
-------------------+-----------------+-------+-------------+--------------------

```



```shell
maxadmin list listeners

Listeners.
---------------------+--------------------+-----------------+-------+--------
Service Name         | Protocol Module    | Address         | Port  | State
---------------------+--------------------+-----------------+-------+--------
Read-Write Split Service | MySQLClient        | *               |  4006 | Running
MaxAdmin Service     | maxscaled          | default         |     0 | Running
---------------------+--------------------+-----------------+-------+--------

```



如果按照上面的配置，读写分离的路由情况是读操作全部在从库上进行，主库不承担写操作。如果所有从库都挂了，那么读操作会转移到主库上执行。



PS.该中间件支持MySQL Connector C++  中的PrepareStatement预处理，且对所有符号（键盘直接输入）都可以正确绑定参数。



## 更新 

经研究发现，通过配置也可以让主库承担读操作，配置好`router_options`的参数就可以。

具体为，修改`service section`的配置(增加了第4行)

```ini
[Read-Write Split Service]
type=service
router=readwritesplit
router_options=master_accept_reads=true
servers=master-server,slave-server
user=maxrouter
passwd=maxscale
max_slave_connections=100%
```

