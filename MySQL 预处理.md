# MySQL 预处理
[TOC]


## 语法
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

MaxScale完全兼容MySQL的协议，对于客户端来说，连接MaxScale与连接MySQL没有区别。但是从MaxScale的角度来看，作为一个高可用中间件，它必须与MySQL的行为完全一致才可以对客户端的无差别性。所以对于客户端的预处理请求，MaxScale也是分步骤提交给MySQL的。
上面解释了，预处理的第一步和第二步实际上都是设置变量，这就要求与MaxScale连接的所有后端数据库服务器都要执行相同的操作，以保证一致性，即便在一个服务器故障之后，其他服务器上依然有同样的变量设置。所以，构造一个含有占位符的预备语句时，MaxScale会把这个过程发送到所有后端服务器上执行一遍。而第三步执行预处理，这个SQL语句的类型是`EXEC_STMT`，MaxScale无法根据这个类型来区分执行的语句是读是写。换句话说，MySQL本身在处理这个地方的时候也是没有区分是执行读语句还是写语句。所以MaxScale只能把这个过程放在主库上执行。



```mysql
> Autocommit: [enabled], trx is [not open], cmd: MYSQL_COM_STMT_PREPARE, type: QUERY_TYPE_READ|QUERY_TYPE_PREPARE_STMT, stmt: select * from tb where id=?; 
Route query to master 	172.18.3.104:3306 <
> Autocommit: [enabled], trx is [not open], cmd: MYSQL_COM_STMT_EXECUTE, type: QUERY_TYPE_EXEC_STMT, stmt: 
Route query to master 	172.18.3.104:3306 <
> Autocommit: [enabled], trx is [not open], cmd: UNKNOWN MYSQL PACKET TYPE, type: QUERY_TYPE_WRITE, stmt: 
Route query to master 	172.18.3.104:3306 <
```




**结论**：MaxScale的读写分离和预处理并不能完美的配合。在准备语句时，MaxScale会发给所有后端数据库执行；在执行语句时，只会在主库上执行。




## Mycat的预处理


Mycat与MaxScale的逻辑不同，它在处理预处理语句的时候，并不是分步骤提交给后端MySQL的，而是自己先缓存预备语句，然后等待下一步的变量绑定，然后组成一个完整的SQL语句发给后端来执行。也就是说后端服务器永远不会执行一个真正的预处理命令。






```

```