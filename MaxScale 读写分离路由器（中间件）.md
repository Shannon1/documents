# MaxScale读写分离路由器（中间件）

[TOC]

本文解释了MaxScale读写分离路由器对于查询语句的路由策略，以及可以影响路由决策的参数配置。

## MaxScale 读写分离路由决策

MaxScale 构建在MySQL复制的基础上，后端MySQL服务器对于MaxScale来说并不是等价的，路由器可以区分后端服务器哪个是主库那个是从库，并可以监控主从复制的运行状况，比如从库的健康状况、复制延迟等，并可以根据主从复制的运行状况，决定路由策略。

即使在主从复制正常的情况下，也不是任意读查询都可以均衡的路由到后端的MySQL服务器上，这里有一些简单的解释，来说明不同类型的查询的路由策略。

### 路由到主库的情况

路由到主库对保证数据的一致性非常关键，因为需要保证写查询可以准确写入binlog，进而复制到从库。

以下操作会路由到主库：

-   写查询
-   所有开启事务的查询（包括开启事务的读查询）
-   调用存储过程
-   调用用户自定义函数
-   DDL （数据操作）语句(`DROP`|`CREATE`|`ALTER TABLE` … 等.)
-   `EXECUTE` （预处理） 语句
-   所有使用临时表的语句

除此之外，如果 **读写分离(readwritesplit)** 服务配置了 `max_slave_replication_lag` 参数, 并且从库上产生了非常大的复制延迟，查询会被路由到*主库*上执行。 (将来有可能加入其它的配置参数来限制路由到从库的语句数量。)

>   注：`max_slave_replication_lag` 参数指定的是从库复制落后于主库的时间，如果实际从库上的复制延迟超过这个时间，那么查询会被路由到主库上来执行。



### 路由到从库的情况

把语句路由到从库可以有效降低主库的负载。而且大量读查询负载可以与多个从库来共同承担。

可路由到从库的查询必须设置为**自动提交(autocommit=1)**，并属于以下的操作之一：

-   对数据库的读查询
-   对系统或用户自定义变量的读查询
-   `SHOW` 语句
-   系统函数调用



**综上，可以被路由到从库上的读查询必须满足 1.不能开启事务 2.自动提交**



### 路由到所有后端服务器

第三类语句包括修改会话数据，比如会话相关的系统变量，用户自定义变量，默认的数据库等。我们称这些操作为**会话命令(session commands)**，会话命令必须被复制到所有会话，因为它会改变将来的读写操作的结果，并代表当前客户端在所有后端MySQL Server上执行语句。

会话命令举例：

-   `SET` 语句
-   `USE `*`<dbname>`*
-   包含在只读语句中的系统或用户自定义变量的赋值
-   `PREPARE` 语句
-   `QUIT`, `PING`, `STMT RESET`, `CHANGE USER`, 等命令

**注：如果变量赋值包含在写语句中，语句将只会被路由到主库。比如`INSERT INTO t1 values(@myvar:=5, 7)`将只会被路由到主库**

读写分离服务会存储所有执行的会话命令，以便在从库故障的情况下，可以有从库替换，并且可以在该新的从库上重复会话命令历史。这意味着读写分离服务在会话期间存储每个执行的会话命令。长时间运行的会话的应用程序可能导致MaxScale消耗的内存不断增长，直到会话关闭。这可以通过在应用程序端设置连接超时来解决。



### 关于预处理

一般来说，预处理需要三个步骤：

1.  准备查询语句(prepare statement)
2.  绑定参数(bind value)
3.  执行语句(excute query)

比如(以MySQL Connector C++为例)：

```cpp
PreparedStatement *prep_stmt;
ResultSet *res;            
prep_stmt = con->prepareStatement("select * from tb where id=?;");  //准备查询语句
prep_stmt->setInt(1, 1234);  										//绑定参数
res = prep_stmt->executeQuery();  									//执行语句
```

根据上面的路由策略可以知道：1.准备查询语句 和2.绑定参数 会被路由到所有后端服务器，3.执行语句会在主库上进行。



## 配置参数

### `max_slave_connections`

**`max_slave_connections`** 参数设置一个路由会话所使用的最大从库的数量。默认使用所有的可用从库。

```ini
max_slave_connections=<max. number, or % of available slaves>
```

### `max_slave_replication_lag`
**`max_slave_replication_lag`** 指定允许从库落后于主库的最大时间。如果MySQL复制的延迟大于该值，从库将不会用来被路由。该配置默认不生效，也就是无论从库落后主库多少，也会在路由候选服务器列表里。

```ini
max_slave_replication_lag=<allowed lag in seconds>
```

该选项适用于MySQL主从复制监控，且需要配置 `detect_replication_lag=1` 参数。请注意 `max_slave_replication_lag` 必须大于监控时间间隔。


### `use_sql_variables_in`

**`use_sql_variables_in`** 指定会话参数的读查询会被路由到哪里。默认路由到所有后端服务器。

```ini
use_sql_variables_in=[master|all]
```
如果这只为`all`，会话参数的读查询会被路由到任一后端服务器(由_路由选择标准_决定)。注意对会话参数的修改默认会被路由到所有后端服务器，除非参数的修改包含在一个写查询中(写查询会被路由到主库)，比如 

```mysql
INSERT INTO test.t1 VALUES (@myid:=@myid+1)
```
上面给出的例子中，用户自定义变量将只会在主库上执行，因为该操作包含在一个写查询`INSERT`中。



## 路由选项router_options


**`router_options`** 包括了多个读写分离选项配置。所有的选项都是_参数-值_对。本部分列出的所有参数都需要配置为`router_option`参数的值。多个选项的值可以用逗号分隔开来。

```ini
router_options=<option>,<option>
```

### `slave_selection_criteria`

这个选项决定读写分离路由器对从库的选择标准以及如何实现负载均衡。默认情况下，读查询会被路由到当前正在查询的数量最少的从库上进行。对应的配置参数是 `LEAST_CURRENT_OPERATIONS`。

```ini
router_options=slave_selection_criteria=<criteria>
```

`<criteria>`需要在一下枚举之中选择其中一个:

* `LEAST_GLOBAL_CONNECTIONS`, 与MaxScale连接最少的从库
* `LEAST_ROUTER_CONNECTIONS`, 与提供的服务(service)连接最少的从库
* `LEAST_BEHIND_MASTER`, 复制延迟最少的从库
* `LEAST_CURRENT_OPERATIONS` (默认), 活跃操作最少的从库

`LEAST_GLOBAL_CONNECTIONS` 和`LEAST_ROUTER_CONNECTIONS` 中的连接，是指MySQL服务器和MaxScale之间的连接，而不是MySQL服务器自身报告的连接数。

`LEAST_BEHIND_MASTER` 在选择服务器时，不考虑服务器权重。

### `max_sescmd_history`

**`max_sescmd_history`** 设置会话命令(session commands, sescmd)所保存的历史命令数量，超过这个数量，最早设置的参数将会被丢弃。 默认不限制，即保存所有的历史会话命令，这显然会导致内存消耗。

```ini
# Set a limit on the session command history
max_sescmd_history=1500
```

这个限制被设定后，就会产生会话消耗的内存上限。这对连接池或者包含有大量参数设置命令的会话非常有用。

### `disable_sescmd_history`

**`disable_sescmd_history`**取消会话命令历史记录。设置此项之后不会记录任何命令历史，如果从库发生故障，路由器也不会用其他从库去取代它。不记录会话命令历史不会使连接池产生持续不断的内存消耗。默认记录命令历史，且记录数量无限制。

```ini
# Disable the session command history
disable_sescmd_history=true
```

### `master_accept_reads`

**`master_accept_reads`** 允许主库用来做读操作。这对于服务器数量较少的情形非常有用。默认情况下，没有读操作在主库上执行。

```ini
# Use the master for reads
master_accept_reads=true
```

### `strict_multi_stmt`

当客户端执行多语句查询(multi-statement query)时，所有这之后的查询都会被**路由到主库**以保证一致的会话状态。可以通过 **`strict_multi_stmt`** 路由选项来控制这个行为。该选项默认开启。

如果关闭该选项，多语句查询之后的查询会恢复正常的路由状态。

**警告:** 如果多语句查询修改了会话状态，关闭该选项可能会造成从库数据读取失败。只有当你确定所执行的多语句查询不会修改会话状态时，才可以关闭该选项。

```ini
# Disable strict multi-statement mode
strict_multi_stmt=false
```

### `master_failure_mode`

该选项设置主库发生故障时的行为。默认情况下，一旦主库故障，路由器会关闭客户端连接。下表描述了该选项的可选值以及对应的主库故障时的行为。

| 可选值            | 描述                                   |
| :------------- | :----------------------------------- |
| fail_instantly | 主库故障时，立刻关闭连接。                        |
| fail_on_write  | 如果收到写查询，且没有主库可用时，关闭客户端连接。            |
| error_on_write | 如果收到写查询，且没有主库可用时，返回错误，连接以只读方式继续提供服务。 |

当设置为 _fail_on_write_ 或 _error_on_write_ 模式时，服务可以在还有从库可用时继续创建新的会话连接。


**注:** 如果设置为 _error_on_write_ ，客户端无法在主库恢复可用状态的同时恢复可写的连接状态（需要手动建立新的连接）。
