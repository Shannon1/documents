## mysql 冷备

mysqlbackup.sh

```bash
!/bin/bash
mysql_user=root
mysql_password='123456'
MYDATE=`date +%Y%m%d`

mysqldump -u$mysql_user -p$mysql_password --all-databases --flush-logs --quick --events --flush-privileges --single-transaction --triggers --routines --hex-blob --master-data=2 --default-character-set=utf8 >/data/backup/dump_backup_$MYDATE.sql
```

**注，如果在crontab执行这个脚本导出的文件是空的话，有可能是找不到mysqldump命令的位置，需要给出mysqldump的全路径，比如`/usr/local/mysql/bin/mysqldump` ** 

## 参数说明

```
--all-databases -A

--databases -B导出几个数据库。参数后面所有名字参量都被看作数据库名。

--events 导出事件

--quick 不缓冲查询，直接导出到标准输出。默认为打开状态，使用--skip-quick取消该选项。

--flush-logs -F 开始导出之前刷新日志。

--flush-privileges 在导出mysql数据库之后，发出一条FLUSH  PRIVILEGES 语句。为了正确恢复，该选项应该用于导出mysql数据库和依赖mysql数据库数据的任何时候。

--single-transaction 该选项在导出数据之前提交一个BEGIN SQL语句，BEGIN 不会阻塞任何应用程序且能保证导出时数据库的一致性状态。
     它只适用于多版本存储引擎，仅InnoDB。本选项和--lock-tables 选项是互斥的，因为LOCK  TABLES 会使任何挂起的事务隐含提交。
     要想导出大表的话，应结合使用--quick 选项。

--triggers 导出触发器。该选项默认启用，用--skip-triggers禁用它。

--routines, -R导出存储过程以及自定义函数。

--hex-blob 使用十六进制格式导出二进制字符串字段。如果有二进制数据就必须使用该选项。影响到的字段类型有BINARY、VARBINARY、BLOB。

--master-data 该选项将binlog的位置和文件名追加到输出文件中。如果为1，将会输出CHANGE MASTER 命令；如果为2，输出的CHANGE  MASTER命令前添加注释信息。
    该选项将打开--lock-all-tables 选项，除非--single-transaction也被指定（在这种情况下，全局读锁在开始导出时获得很短的时间；
    其他内容参考下面的--single-transaction选项）。该选项自动关闭--lock-tables选项。如果在slave上导出数据，不能开启这个选项，需要用--dump-slave
--dump-slave 该选项将导致主的binlog位置和文件名追加到导出数据的文件中。
    设置为1时，将会以CHANGE MASTER命令输出到数据文件；设置为2时，在命令前增加说明信息。
    该选项将会打开--lock-all-tables，除非--single-transaction被指定。该选项会自动关闭--lock-tables选项。默认值为0。
    该选项和--master-data作用一样，会覆盖--master-data的设定

--include-master-host-port 在--dump-slave产生的'CHANGE  MASTER TO..'语句中增加'MASTER_HOST=<host>，MASTER_PORT=<port>'
```

## 加入系统定期任务

```
crontab -e
30 03 * * * sh /home/sql_backup/mysqlbackup.sh
分 时 日 月 周
```


## 删除过期备份

```bash
#!/bin/bash

DBBACKUP_PATH=/data/backup

find ${DBBACKUP_PATH}/mysql/ -name "*" -ctime +5 | xargs rm -f

```

