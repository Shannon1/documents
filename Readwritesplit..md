
# Readwritesplit

This document provides a short overview of the **readwritesplit** router module and its intended use case scenarios. It also displays all router configuration parameters with their descriptions. A list of current limitations of the module is included and examples of the router's use are provided.

## Overview

The **readwritesplit** router is designed to increase the read-only processing capability of a cluster while maintaining consistency. This is achieved by splitting the query load into read and write queries. Read queries, which do not modify data, are spread across multiple nodes while all write queries will be sent to a single node. 

The router is designed to be used with a traditional Master-Slave replication cluster. It automatically detects changes in the master server and will use the current master server of the cluster. With a Galera cluster, one can achieve a resilient setup and easy master failover by using one of the Galera nodes as a Write-Master node, where all write queries are routed, and spreading the read load over all the nodes.

## Configuration

Readwritesplit router-specific settings are specified in the configuration file of MariaDB MaxScale in its specific section. The section can be freely named but the name is used later as a reference from listener section.

For more details about the standard service parameters, refer to the [Configuration Guide](../Getting-Started/Configuration-Guide.md).

## Optional parameters

### `max_slave_connections`

**`max_slave_connections`** sets the maximum number of slaves a router session uses at any moment. The default is to use all available slaves.

	max_slave_connections=<max. number, or % of available slaves>

### `max_slave_replication_lag`
**`max_slave_replication_lag`** specifies how many seconds a slave is allowed to be behind the master. If the lag is bigger than configured value a slave can't be used for routing.

This feature is disabled by default.

	max_slave_replication_lag=<allowed lag in seconds>

This applies to Master/Slave replication with MySQL monitor and `detect_replication_lag=1` options set.
Please note max_slave_replication_lag must be greater than monitor interval.


### `use_sql_variables_in`

**`use_sql_variables_in`** specifies where should queries, which read session variable, be routed. The syntax for `use_sql_variable_in` is:

    use_sql_variables_in=[master|all]

The default is to use SQL variables in all servers.

When value all is used, queries reading session variables can be routed to any available slave (depending on selection criteria). Note, that queries modifying session variables are routed to all backend servers by default, excluding write queries with embedded session variable modifications, such as:

    INSERT INTO test.t1 VALUES (@myid:=@myid+1)

In above-mentioned case the user-defined variable would only be updated in the master where query would be routed due to `INSERT` statement.

## Router options

**`router_options`** may include multiple **readwritesplit**-specific options. All the options are parameter-value pairs. All parameters listed in this section must be configured as a value in `router_options`.

Multiple options can be defined as a comma-separated list of parameter-value pairs.

```
router_options=<option>,<option>
```

### `slave_selection_criteria`

This option controls how the readwritesplit router chooses the slaves it connects to and how the load balancing is done. The default behavior is to route read queries to the slave server with the lowest amount of ongoing queries i.e. `LEAST_CURRENT_OPERATIONS`.

The option syntax:

```
router_options=slave_selection_criteria=<criteria>
```

Where `<criteria>` is one of the following values.

* `LEAST_GLOBAL_CONNECTIONS`, the slave with least connections from MariaDB MaxScale
* `LEAST_ROUTER_CONNECTIONS`, the slave with least connections from this service
* `LEAST_BEHIND_MASTER`, the slave with smallest replication lag
* `LEAST_CURRENT_OPERATIONS` (default), the slave with least active operations

The `LEAST_GLOBAL_CONNECTIONS` and `LEAST_ROUTER_CONNECTIONS` use the connections from MariaDB MaxScale to the server, not the amount of connections reported by the server itself.

`LEAST_BEHIND_MASTER` does not take server weights into account when choosing a server.

### `max_sescmd_history`

**`max_sescmd_history`** sets a limit on how many session commands each session can execute before the session command history is disabled. The default is an unlimited number of session commands.

```
# Set a limit on the session command history
max_sescmd_history=1500
```

When a limitation is set, it effectively creates a cap on the session's memory consumption. This might be useful if connection pooling is used and the sessions use large amounts of session commands.

### `disable_sescmd_history`

**`disable_sescmd_history`** disables the session command history. This way no history is stored and if a slave server fails, the router will not try to replace the failed slave. Disabling session command history will allow connection pooling without causing a constant growth in the memory consumption. The session command history is enabled by default.

```
# Disable the session command history
disable_sescmd_history=true
```

### `master_accept_reads`

**`master_accept_reads`** allows the master server to be used for reads. This is a useful option to enable if you are using a small number of servers and wish to use the master for reads as well.

By default, no reads are sent to the master.

```
# Use the master for reads
master_accept_reads=true
```

### `strict_multi_stmt`

When a client executes a multi-statement query, all queries after that will be routed to
the master to guarantee a consistent session state. This behavior can be controlled with
the **`strict_multi_stmt`** router option. This option is enabled by default.

If set to false, queries are routed normally after a multi-statement query.

**Warning:** this can cause false data to be read from the slaves if the multi-statement query modifies
the session state. Only disable the strict mode if you know that no changes to the session
state will be made inside the multi-statement queries.

```
# Disable strict multi-statement mode
strict_multi_stmt=false
```

### `master_failure_mode`

This option controls how the failure of a master server is handled. By default,
the router will close the client connection as soon as the master is lost.

The following table describes the values for this option and how they treat
the loss of a master server.

| Value          | Description                              |
| -------------- | ---------------------------------------- |
| fail_instantly | When the failure of the master server is detected, the connection will be closed immediately. |
| fail_on_write  | The client connection is closed if a write query is received when no master is available. |
| error_on_write | If no master is available and a write query is received, an error is returned stating that the connection is in read-only mode. |

These also apply to new sessions created after the master has failed. This means
that in _fail_on_write_ or _error_on_write_ mode, connections are accepted as long
as slave servers are available.

**Note:** If _master_failure_mode_ is set to _error_on_write_ and the connection
to the master is lost, clients will not be able to execute write queries without
reconnecting to MariaDB MaxScale once a new master is available.

## Routing hints

The readwritesplit router supports routing hints. For a detailed guide on hint
syntax and functionality, please read [this](../Reference/Hint-Syntax.md) document.

**Note**: Routing hints will always have the highest priority when a routing
decision is made. This means that it is possible to cause inconsistencies in
the session state and the actual data in the database by adding routing hints
to DDL/DML statements which are then directed to slave servers. Only use routing
hints when you are sure that they can cause no harm.

## Limitations

For a list of readwritesplit limitations, please read the [Limitations](../About/Limitations.md) document.

## Examples

Examples of the readwritesplit router in use can be found in the [Tutorials](../Tutorials) folder.

## Readwritesplit routing decisions

这里有一个简单的解释，显示什么类型的查询路由到哪种类型的服务器

### 路由到主库的情况

路由到主库对保证数据的一致性非常关键。由于大多数写操作都需要写入binlog来复制到从库。

以下操作会路由到主库：

* 写语句
* 所有开启事务的语句
* 调用存储过程
* 调用用户自定义函数
* DDL 语句(`DROP`|`CREATE`|`ALTER TABLE` … 等.)
* `EXECUTE` (预处理) 语句
* 所有使用临时表的语句

除此之外，如果 **读写分离(readwritesplit)** 服务配置了 `max_slave_replication_lag` 参数, 并且从库上产生了非常大的复制延迟，语句会被路由到*主库*。 (将来有可能加入其它的配置参数来限制路由到从库的语句数量。)

### 路由到从库的情况

把语句路由到从库的功能非常重要，因为这样可以降低主库的负载。而且与单一主库相比，负载可以与多个从库来共同承担。

可路由到从库的查询必须设置为**自动提交**，并属于以下的操作之一：

* 对数据库的读查询
* 对系统或用户自定义变量的读查询
* `SHOW` 语句
* 系统函数调用

### 路由到所有后端会话

第三类语句包括修改会话数据，比如会话相关的系统变量，用户自定义变量，默认的数据库等。我们称这些操作为**会话命令**，会话命令必须被复制到所有会话，因为它会改变将来的读写操作的结果，并代表当前客户端在所有后端MySQL Server上执行语句。

会话命令举例：

* `SET` 语句
* `USE `*`<dbname>`*
* 包含在只读语句中的系统或用户自定义变量的赋值
* `PREPARE` 语句
* `QUIT`, `PING`, `STMT RESET`, `CHANGE USER`, 等命令

**注：如果变量赋值包含在写语句中，语句将只会被路由到主库。比如`INSERT INTO t1 values(@myvar:=5, 7)`将只会被路由到主库**

读写分离服务会存储所有执行的会话命令，以便在从库故障的情况下，可以有从库替换，并且可以在该新的从库上重复会话命令历史。这意味着读写分离服务在会话期间存储每个执行的会话命令。长时间运行的会话的应用程序可能导致MaxScale消耗的内存不断增长，直到会话关闭。这可以通过在应用程序端设置连接超时来解决。