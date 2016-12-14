# MySQL 预处理
[TOC]


## 预处理简介
预处理大致分为4个步骤
1. 预备语句

2. 设置变量，绑定参数

3. 执行语句

4. 删除预备语句




体现在MySQL语法上：

```mysql
PREPARE stmt_name FROM preparable_stmt;
SET @var_name = var_value;
EXECUTE stmt_name [USING @var_name [, @var_name] ...];
{DEALLOCATE | DROP} PREPARE stmt_name;
```



PREPARE语句用于预备一个语句，并给这个语句起名字为`stmt_name`，在后面的步骤中将会引用这个预备语句；预备语句`preparable_stmt`中需要在接下来的步骤中绑定的变量，由`?`问号作为占位符；

第二步设置预备语句中所需要用到的变量，和一般的MySQL设置变量的方法一样；实际上，第一步中预备语句也可以先设置为一个变量；

```mysql
SET @var_stmt = preparable_stmt;
PREPARE stmt_name FROM @var_stmt;
```



第三步是用刚刚设置的变量按顺序替换第一步的预备语句中的占位符，并执行该完整的SQL语句。

举个例子：
```mysql
mysql> PREPARE prod FROM 'INSERT INTO tb(name,age) VALUES(?,?)';
Query OK, 0 rows affected (0.00 sec)
Statement prepared

mysql> set @p='hello';
Query OK, 0 rows affected (0.06 sec)

mysql> set @q=20;
Query OK, 0 rows affected (0.00 sec)

mysql> execute prod using @p, @q;
Query OK, 1 row affected (0.05 sec)

mysql> deallocate prepare prod;
```



## MaxScale的预处理

MaxScale完全兼容MySQL的协议，对于客户端来说，连接MaxScale与连接MySQL没有区别。

从MaxScale的角度来看，作为一个高可用中间件，它必须与MySQL的行为完全一致才，可以保证对客户端的无差别性。所以对于客户端的预处理请求，与MySQL的处理方式类似，MaxScale也是将这个过程分步骤进行的。

上面解释了，预处理的第一步和第二步实际上都是设置变量，这就要求与MaxScale连接的所有后端数据库服务器都要执行相同的操作，以保证一致性，这种一致性可以保证即便在一个服务器故障之后，其他服务器上依然有同样的变量设置。所以，构造一个含有占位符的预备语句时，MaxScale会把这个过程发送到所有后端服务器上执行一遍。

而第三步执行预处理，这个SQL语句的类型是`EXEC_STMT`，MaxScale无法根据这个类型来区分执行的语句是读是写。换句话说，MySQL本身在处理这个地方的时候也是没有区分是执行读语句还是写语句。所以MaxScale只能把这个过程放在主库上执行。



C++代码（Mysql connector c++）

```cpp
PreparedStatement *prep_stmt;
ResultSet *res;            
prep_stmt = con -> prepareStatement ("select * from tb where id=?;");   
prep_stmt -> setInt (1, i+1);  
res = prep_stmt -> executeQuery();  
prep_stmt -> close();
delete res;
delete prep_stmt;
```



MaxScale 在收到上面的代码的时候的执行过程是：

```mysql
> Autocommit: [enabled], trx is [not open], cmd: MYSQL_COM_STMT_PREPARE, type: QUERY_TYPE_READ|QUERY_TYPE_PREPARE_STMT, stmt: select * from tb where id=?; 
Route query to master 	172.18.3.104:3306 <
> Autocommit: [enabled], trx is [not open], cmd: MYSQL_COM_STMT_EXECUTE, type: QUERY_TYPE_EXEC_STMT, stmt: 
Route query to master 	172.18.3.104:3306 <
> Autocommit: [enabled], trx is [not open], cmd: UNKNOWN MYSQL PACKET TYPE, type: QUERY_TYPE_WRITE, stmt: 
Route query to master 	172.18.3.104:3306 <
```



可以看到，MaxScale在处理预处理语句时，可以区分`MYSQL_COM_STMT_PREPARE`和`MYSQL_COM_STMT_EXECUTE`两种消息类型。但是MaxScale把`MYSQL_COM_STMT_PREPARE`当做变量设置发到所有后端服务器上执行， `MYSQL_COM_STMT_EXECUTE`由于无法区分读写只能在主库上执行。


**结论**：MaxScale的读写分离和预处理并不能完美的配合。在准备语句时，MaxScale会发给所有后端数据库执行；在执行语句时，只会在主库上执行。如果MaxScale可以把`QUERY_TYPE_READ|QUERY_TYPE_PREPARE_STMT`和`QUERY_TYPE_EXEC_STMT`两步骤结合起来，就可以判断执行预处理时的读写了。




## Mycat的预处理


Mycat与MaxScale的逻辑不同，它在处理预处理语句的时候，并不是分步骤提交给后端MySQL的，而是自己先缓存预备语句，然后等待下一步的变量绑定，然后组成一个完整的SQL语句发给后端来执行。也就是说后端服务器不会执行一个真正的预处理命令。



Mycat通过两个Map来管理预处理语句，为每一个预处理语句分配一个id值，来防止参数绑定可能带来的混乱


```java
private volatile long pstmtId;
private Map<String, PreparedStatement> pstmtForSql;
private Map<Long, PreparedStatement> pstmtForId;
```


通过下面的函数来组组装成完整的SQL语句

```java
/**
 * 组装sql语句,替换动态参数为实际参数值
 * @param pstmt
 * @param bindValues
 * @return
 */
private String prepareStmtBindValue(PreparedStatement pstmt, BindValue[] bindValues) {
    ...
    return sb.toString();
}
```



## OneProxy中关于预处理的说明

[OneProxy](http://www.onexsoft.com/?page_id=3383)是一款国产数据库中间件产品，可以实现透明连接池、故障切换、分库分表、读写分离、负载均衡等功能。OneProxy在它的技术白皮书里说，透明的连接池带来方便的同时也带来的一些限制。[出处](http://onexsoft.cn/wp-content/uploads/2015/07/%E7%A8%8B%E5%BA%8F%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97.pdf)



>   ...
>
>   任何set命令都会直接返回成功，而不做任何处理，必须保持连接池里的每一个连接处于同样会话配置的状态。后端个别数据库参数的调整，需要直接访问后端数据库进行操作。
>
>   不支持Server Side Cursor接口，比如MySQL C API里的Prepare、Bind、Execute调用接口，因为MySQL的Prepare是与某个会话相关的。在MySQL上使用Prepared方式无性能优势，象JDBC、PHP PDO等客户端接口的默认方式，只是在API上有Prepared字样，其实还是进行字符串拼接方式处理的。
>
>   ...



文中说在MySQL上使用Prepared方式无性能优势，是不太严谨的。因为预处理可以只进行一次SQL语法分析，而多次绑定参数可以获得多次的结果。如果是一次预处理只有一次绑定参数获取一次结果之后，预处理语句就丢弃了，可能在性能上确实是没有优势。但是预处理把SQL逻辑与数据的分离，能在一定程度上防止SQL注入攻击，增加系统安全性，对系统来说还是非常有用的功能。



## 结论

目前考察了很多读写分离中间件，都存在一些大大小小的问题

| 中间件              | 问题                                       |
| ---------------- | ---------------------------------------- |
| Mycat1.6-Release | Mycat内部拼接预处理字符串，拼接时会有数据类型检查；不支持绑定参数里有`?'`等特殊符号 |
| MaxScale2.02     | 以预处理方式执行的读查询无法路由到从库上                     |
| OneProxy         | 不支持预处理                                   |
| MySQL Router     | 基于连接的读写分离路由器，不能解析语句                      |
| MySQL Proxy      | 基于lua脚本，alpha版，已停止更新                     |
| Amoeba           | 已停止更新                                    |
| Atlas            | 读写分离基于MySQL Proxy，已停止更新                  |



要么等Mycat下个版本更新，再测试对预处理的支持情况。要么自己用组字符串，进行类型检查（比如使用`boost::format`），组好SQL语句后发给后端执行。