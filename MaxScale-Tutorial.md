# 安装MariaDB MaxScale

[TOC]



本文档是快速安装部署MariaDB MaxScale的介绍。MaxScale可以部署在MySQL一主多从的复制集群环境，或是一个多节点的Galera集群环境。本文介绍了MariaDB MaxScale安装和配置的步骤。

本文不会介绍MySQL复制和Galera集群的安装和配置，也不会讨论复制集群中用于故障转移的自动化或半自动化安装管理工具。 [Setting Up Replication](https://mariadb.com/kb/en/mariadb/setting-up-replication/) 这篇文章可以帮助你基于MariaDB构建复制集群， [Getting Started With Mariadb Galera Cluster](https://mariadb.com/kb/en/mariadb/getting-started-with-mariadb-galera-cluster/) 这篇文章会帮助你安装Galera集群。

本教程假定用户使用的是二进制发行版MaxScale并且安装到了默认位置。从GitHub上的原码编译安装在这篇文章中有介绍  [Building from Source](../Getting-Started/Building-MaxScale-from-Source-Code.md) 。



## 主要步骤

安装 MariaDB MaxScale 所需的步骤有:

* 安装与你系统环境对应的发行包
* 在MariaDB或MySQL复制集群中创建所需的用户
* 创建MariaDB MaxScale配置文件



## 安装

精确的安装步骤会随着不同的发行包的不同而在细节方面有所区别，您可以在下载页面选择不同的RPM和DEB发行包。先要安装包含MariaDB仓库的包管理器，然后在您的发行版上运行包管理器（通常来说是yum或者apt-get）。

如果成功的完成安装命令，MariaDB MaxScale需要进行配置才可以运行。在第一次运行MaxScale之前你需要创建配置文件，配置过程接下来的部分会有说明。



## 在数据库中创建用户

MariaDB MaxScale 需要连接后端数据库服务器来执行查询。这样做有两个原因：一是查明数据库目前的运行状态，用来做模块监控；二是获取数据库集群的用户信息，共MaxScale自身使用。可以创建两个不同的用户来满足需求，也可以只创建一个。

第一个用户需要有`mysql.user`的`SELECT`权限，可以通过以下步骤创建用户：

1. 用root用户连接主库
2. 创建用户，将 username, password 和 host 根据maxscale运行的环境来替换
```
MariaDB [(none)]> create user '*username*'@'*maxscalehost*' identified by '*password*';

**Query OK, 0 rows affected (0.00 sec)**
```
3. 授予`mysql.user`表的`SELECT`权限
```
MariaDB [(none)]> grant SELECT on mysql.user to '*username*'@'*maxscalehost*';

**Query OK, 0 rows affected (0.03 sec)**
```
另外，需要授予`mysql.db`、`mysql.tables_priv` 的 `SELECT` 权限，以及`SHOW DATABASES` 权限，用来加载数据库和适用于数据库的权限。
```
MariaDB [(none)]> GRANT SELECT ON mysql.db TO 'username'@'maxscalehost';

**Query OK, 0 rows affected (0.00 sec)**

MariaDB [(none)]> GRANT SELECT ON mysql.tables_priv TO 'username'@'maxscalehost';

**Query OK, 0 rows affected (0.00 sec)**

MariaDB [(none)]> GRANT SHOW DATABASES ON *.* TO 'username'@'maxscalehost';

**Query OK, 0 rows affected (0.00 sec)**
```
第二个用户用来监控集群的运行状态。第二个用户（也可以与第一个用户同名）需要获取各种来源的监控信息的权限。为了能监控集群的复制状态，该用户需要授予`REPLICATION CLIENT`的角色。这是MySQL监控和多主监控模块所必须的。

```
MariaDB [(none)]> grant REPLICATION CLIENT on *.* to '*username*'@'*maxscalehost*';

**Query OK, 0 rows affected (0.00 sec)**
```

如果您希望对监视和收集用户信息的两种不同角色使用两个不同的用户名，请使用上面的前两个步骤创建不同的用户名。



## 为用户授予额外权限

由于MariaDB MaxScale 位于客户端和后端数据库之间，后端数据库会看到所有连接都来自于 MariaDB MaxScale的地址。 这通常需要用户为MariaDB MaxScale的主机名创建额外的授权。下面通过一个例子来解释这个情况。

用户 `'jdoe'@'192.168.0.200` 在集群上拥有下列权限: `GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'jdoe'@'192.168.0.200'`。当该用户直接连接服务器时，服务器会发现来自 `'jdoe'@'192.168.0.200` 的连接，会匹配到 `'jdoe'@'192.168.0.200`的权限。

如果 MariaDB MaxScale 部署在 `192.168.0.101` 的地址上，用户 `jdoe` 连接这台 MariaDB MaxScale，后端服务器会连接来自 `'jdoe'@'192.168.0.101'`。由于后端服务器并没有`'jdoe'@'192.168.0.101'`的授权，因此该连接会被后端服务器拒绝。

我们可以通过两种方法修复这个问题，一是为这个用户在MaxScale所在的地址创建匹配的权限，二是使用通配符覆盖MaxScale所在的地址段。

最快的方式是通过 `SHOW GRANTS` 查询:
```
MariaDB [(none)]> SHOW GRANTS FOR 'jdoe'@'192.168.0.200';
+-----------------------------------------------------------------------+
| Grants for jdoe@192.168.0.200                                         |
+-----------------------------------------------------------------------+
| GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'jdoe'@'192.168.0.200' |
+-----------------------------------------------------------------------+
1 row in set (0.01 sec)
```
然后创建用户 `'jdoe'@'192.168.0.101'` 并授予同样权限:
```
MariaDB [(none)]> CREATE USER 'jdoe'@'192.168.0.101';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'jdoe'@'192.168.0.101';
Query OK, 0 rows affected (0.00 sec)
```

另一个选择是用通配符授权  `GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'jdoe'@'%'`。这样会更方便，但是与精确指定MaxScale和客户端的地址相比，安全性有所降低。


## 创建配置文件

配置文件的创建在下面几篇教程中涉及。

### 主从复制集群

* [MySQL Replication Connection Routing Tutorial](https://github.com/mariadb-corporation/MaxScale/blob/2.0/Documentation/Tutorials/MySQL-Replication-Connection-Routing-Tutorial.md)
* [MySQL Replication Read-Write Splitting Tutorial](https://github.com/mariadb-corporation/MaxScale/blob/2.0/Documentation/Tutorials/MySQL-Replication-Read-Write-Splitting-Tutorial.md)

### Galera集群

* [Galera Cluster Connection Routing Tutorial](https://github.com/mariadb-corporation/MaxScale/blob/2.0/Documentation/Tutorials/Galera-Cluster-Connection-Routing-Tutorial.md)
* [Galera Cluster Read Write Splitting Tutorial](https://github.com/mariadb-corporation/MaxScale/blob/2.0/Documentation/Tutorials/Galera-Cluster-Read-Write-Splitting-Tutorial.md)

