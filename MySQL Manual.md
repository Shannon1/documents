# MySQL 运维手册

@(MySQL)

[toc]

----

# 1. MySQL 安装

线上MySQL数据库的线上环境安装，建议采用**编译安装**的方法，性能会有很大提升。源码包的编译参数默认以Debug模式生成二进制代码，Debug模式给MySQL带来的性能损失是比较大的，所以在编译安装时`--without-debug`参数禁用Debug模式。


##　1.1 单实例安装

### 1.1.1 解决perl编译环境问题

```bash
echo 'export LC_ALL=C'>>/etc/profile
source /etc/profile
```

### 1.1.2 安装必须的组件包

```bash
yum  groupinstall -y "Base" ,"Compatibility libraries","Hardware monitoring utilities","Debugging Tools","Dial-up Networking Support","Development tools"
yum install -y  cmake ncurses*
```
### 1.1.3 添加用户和组

```bash
groupadd mysql
useradd mysql -g mysql -M -s /sbin/mysql  //未测试
// 或
useradd -r -g mysql mysql 	// 测试过

// 参数说明
-g The group name or number of the user´s initial login group.
-r Create a system account.
-M Do not create the user´s home directory
-s The name of the user´s login shell. The default is to leave this field blank, which causes the system to select the default login shell specified by the SHELL variable in /etc/default/useradd, or an empty string by default.
```

### 1.1.4 创建MySQL目录

比如`/application/mysql/` 或 `/usr/local/mysql` ,以下用 MYSQL_HOME 代替
```bash
mkdir -p MYSQL_HOME 
chown -R mysql.mysql MYSQL_HOME 
```

### 1.1.5 修改环境变量

```bash
echo 'PATH=$PATH:MYSQL_HOME/bin'>>/etc/profile
或
vim /etc/profile
添加
export PATH=$PATH:MYSQL_HOME/bin

source /etc/profile
```

### 1.1.6 安装

```bash
tar xf mysql-5.5.47.tar.gz // -C dest_dir
cd mysql-5.5.47
cmake . -DCMAKE_INSTALL_PREFIX=MYSQL_HOME \
-DMYSQL_DATADIR=MYSQL_HOME/data \
-DMYSQL_UNIX_ADDR=MYSQL_HOME/mysql.sock \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DEXTRA_CHARSETS=gbk,gb2312,utf8,ascii \
-DENABLED_LOCAL_INFILE=ON \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_FEDERATED_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 \
-DWITHOUT_PARTITION_STORAGE_ENGINE=1 \
-DWITH_FAST_MUTEXES=1 \
-DWITH_ZLIB=bundled \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_READLINE=1 \
-DWITH_EMBEDDED_SERVER=1 \
-DWITH_DEBUG=0

make&&make install
```

### 1.1.7 初始化数据库

```bash
cd /usr/local/mysql/scripts
./mysql_install_db --user=mysql --basedir=MYSQL_HOME --datadir=MYSQL_HOME/data
```

### 1.1.8 my.cnf配置文件

`vim /etc/my.conf`

```ini
[client]
port            = 3306
socket          = MYSQL_HOME/mysql.sock

[mysql]
no-auto-rehash
#disable-auto-rehash #允许通过TAB键提示 same as skip-auto-rehash
[mysqld]
user    = mysql
port    = 3306
socket  = MYSQL_HOME/mysql.sock
basedir = MYSQL_HOME
datadir = MYSQL_HOME/data
open_files_limit = 1024 	# mysql可以打开的最大文件描述符的数量
back_log = 600	 #接受队列，对于没建立tcp连接的请求队列放入缓存中，队列大小为back_log，受限制与OS参数
max_connections = 800	 #最大并发连接数,增大该值需要相应增加允许打开的文件描述符数
max_connect_errors = 3000	#如果某个用户发起的连接error超过该数值，则该用户的下次连接将被阻塞，直到管理员执行flush hosts命令,防止黑客
table_cache = 614	 #所有线程打开的表的数量(内存中缓存的表的数量)。增大该值可以增加mysqld需要的文件描述符的数量
external-locking = FALSE #效果同skip-external-locking,关闭外部锁定,默认关闭即可
max_allowed_packet =8M #mysql server所能接受的单条sql语句的大小,设置过小可能会导致导入数据(导入sql或binlog失败)
sort_buffer_size = 1M	#排序buffer大小；线程级别  
join_buffer_size = 1M   #join buffer 大小;线程级别 
thread_cache_size = 100	#线程缓存
thread_concurrency = 2	#同时运行的线程的数据 此处最好为CPU个数
query_cache_size = 2M	#查询缓存大小 
query_cache_limit = 1M	#不缓存查询大于该值的结果 
query_cache_min_res_unit = 2k  #查询缓存分配的最小块大小  
thread_stack = 192K		#每个线程的堆栈大小  
transaction_isolation = READ-COMMITTED #默认REPEATABLE-READ
tmp_table_size = 2M  #临时表大小，如果超过该值，则结果放到磁盘中  
max_heap_table_size = 2M	#该变量设置MEMORY (HEAP)表可以增长到的最大空间大小
long_query_time = 1 	#慢查询时间 超过1秒则为慢查询
log-error = MYSQL_HOME/error.log # mysql日志文件
log-slow-queries = MYSQL_HOME/slow.log	#慢查询日志文件
pid-file = MYSQL_HOME/mysql.pid	#pid文件,文件记录了mysql的进程id
log-bin = MYSQL_HOME/mysql-bin 	#binlog文件
relay-log = MYSQL_HOME/relay-bin 	#relay-log
relay-log-info-file = MYSQL_HOME/relay-log.info
binlog_cache_size = 1M #为每一个client分配的一次事务的binlog的缓存大小,session级别, 如果经常出现多语句事务, 可以考虑增加该值的大小
max_binlog_cache_size = 1M #和binlog_cache_size对应, binlog所能使用的最大cache内存大小, 当我们执行多语句事务的时候，max_binlog_cache_size 如果不够大的话，系统可能会报出“ Multi-statement transaction required more than 'max_binlog_cache_size' bytes ofstorage”的错误。
max_binlog_size = 2M #单个binlog文件的最大大小,超过这个值会建立新的binlog文件,文件id递增; 一般不要超过1G
expire_logs_days = 7 #binlog过期时间 / 天
key_buffer_size = 16M	#myisam索引buffer,只有key没有data
read_buffer_size = 1M 	#以全表扫描(Sequential Scan)方式扫描数据的buffer大小 ；线程级别  
read_rnd_buffer_size = 1M #MyISAM以索引扫描(Random Scan)方式扫描数据的buffer大小 ；线程级别
bulk_insert_buffer_size = 1M #MyISAM 用在块插入优化中的树缓冲区的大小,这是一个per thread的限制  

lower_case_table_names = 1 #大小写无关
skip-name-resolve	#禁用主机缓存,只允许ip链接,禁用mysql通过ip反向解析
slave-skip-errors = 1032,1062 #跳过复制时的错误
replicate-ignore-db=mysql #主从复制时忽略的库

server-id = 1

innodb_additional_mem_pool_size = 4M #帧缓存的控制对象需要从此处申请缓存，所以该值与innodb_buffer_pool对应  
innodb_buffer_pool_size = 32M #包括数据页、索引页、插入缓存、锁信息、自适应哈希所以、数据字典信息
innodb_data_file_path = ibdata1:128M:autoextend	#表空间
innodb_file_io_threads = 4	#io线程数
innodb_thread_concurrency = 8 #InnoDB试着在InnoDB内保持操作系统线程的数量少于或等于这个参数给出的限制  
innodb_flush_log_at_trx_commit = 1 #每次commit 日志缓存中的数据刷到磁盘中
innodb_log_buffer_size = 2M	#事务日志缓存
innodb_log_file_size = 4M	#事务日志大小
innodb_log_files_in_group = 3 #三组事务日志
innodb_max_dirty_pages_pct = 90 #innodb主线程刷新缓存池中的数据，使脏数据比例小于90%
innodb_lock_wait_timeout = 120 #InnoDB事务在被回滚之前可以等待一个锁定的超时秒数。InnoDB在它自己的 锁定表中自动检测事务死锁并且回滚事务。InnoDB用LOCK TABLES语句注意到锁定设置。默认值是50秒  
innodb_file_per_table = 0
[mysqldump]
quick
max_allowed_packet = 2M

[mysqld_safe]
log-error=/application/mysql/mysql_oldboy3306.err
pid-file=/application/mysql/mysqld.pid
```

部分参数详细说明

`back_log`
要求MySQL能有的连接数量。当主要MySQL线程在一个很短时间内得到非常多的连接请求，这就起作用，然后主线程花些时间(尽管很短)检查连接并且启动一个新线程。 
`back_log`值指出在MySQL暂时停止回答新请求之前的短时间内多少个请求可以被存在堆栈中。只有如果期望在一个短时间内有很多连接，你需要增加它，换句话说，这值 对到来的TCP/IP连接的侦听队列的大小。你的操作系统在这个队列大小上有它自己的限制。 试图设定`back_log`高于你的操作系统的限制将是无效的。在mysql中`back_log`的设置取决于操作系统，在linux下这个参数的值不能大于系统参数`tcp_max_syn_backlog`的值。
通过以下命令可以查看`tcp_max_syn_backlog`的当前值 
`cat /proc/sys/net/ipv4/tcp_max_syn_backlog`
通过以下命令进行修改
`sysctl -w net.ipv4.tcp_max_syn_backlog=n`
深入探讨一点 tcp/ip网络一般会有如下过程，从生成socket到bind端口在listen进而建立连接 具体到listen，就是`listen(int fd, int backlog)`的调用，这里backlog和mysql中`back_log`具有一定的关系，即操作系统backlog的要不小于mysql中`back_log`的值，在linux内核2.6.6中backlog在/include/net/tcp.h中由`TCP_SYNQ_HSIZE`变量定义。当你观察你的主机进程列表，发现大量 `264084 | unauthenticated user | xxx.xxx.xxx.xxx | NULL | Connect | NULL | login | NULL `的待连接进程时，就要加大 `back_log` 的值了。默认数值是50，我把它改为500。

`transaction_isolation` <<高性能mysql>> page 8 

`key_buffer_size`指定索引缓冲区的大小，它决定索引处理的速度，尤其是索引读的速度。通过检查状态值`key_read_requests`和`key_reads`，可以知道`key_buffer_size`设置是否合理。比例`key_reads / key_read_requests`应该尽可能的低，至少是1:100，1:1000更好（上述状态值可以使用`SHOW STATUS LIKE 'key_read%''`获得）。
`key_buffer_size`只对MyISAM表起作用。即使你不使用MyISAM表，但是内部的临时磁盘表是MyISAM表，也要使用该值。可以使用检查状态值`created_tmp_disk_tables`得知详情。对于1G内存的机器，如果不使用MyISAM表，推荐值是16M（8-64M）。

`innodb_flush_log_at_trx_commit` 参数指定了InnoDB在事务提交后的日志写入频率。这么说其实并不严谨，且看其不同取值的意义和表现。
- 当 `innodb_flush_log_at_trx_commit` 取值为0的时候，log buffer会每秒写入到日志文件并刷写（flush）到磁盘。但每次事务提交不会有任何影响，也就是log buffer的刷写操作和事务提交操作没有关系。在这种情况下，**MySQL性能最好**，但如果 mysqld进程崩溃，通常会导致最后1s的日志丢失。
- 当取值为1时，每次事务提交时，log buffer会被写入到日志文件并刷写到磁盘。这也是默认值。这是**最安全**的配置，但由于每次事务都需要进行磁盘I/O，所以也**最慢**。
- 当取值为2时，每次事务提交会写入日志文件，但并不会立即刷写到磁盘，日志文件会每秒刷写一次到磁盘。这时如果mysqld进程崩溃，由于日志已经写入到系统缓存，所以并不会丢失数据；在操作系统崩溃的情况下，通常会导致最后1s的日志丢失。
上面说到的「最后 1s」并不是绝对的，有的时候会丢失更多数据。有时候由于调度的问题，每秒刷写（once-per-second flushing）并不能保证 100% 执行。对于一些数据一致性和完整性要求不高的应用，配置为 2 就足够了；如果为了最高性能，可以设置为 0。有些应用，如支付服务，对一致性和完整性要求很高，所以即使最慢，也最好设置为 1.

`sync_binlog` 是MySQL的二进制日志（binary log）同步到磁盘的频率。MySQL server在binary log每写入 `sync_binlog` 次后，刷写到磁盘。
如果autocommit开启，每个语句都写一次binary log，否则每次事务写一次。可选值0或n(n>0)。默认值是0，不主动同步，而依赖操作系统本身不定期把文件内容flush到磁盘。设为1最安全，在每个语句或事务后同步一次binary log，即使在崩溃时也最多丢失一个语句或事务的日志，但因此也最慢。
大多数情况下，对数据的一致性并没有很严格的要求，所以并不会把 `sync_binlog` 配置成1. 为了追求高并发，提升性能，可以设置为100或直接用0.而和 `innodb_flush_log_at_trx_commit` 一样，对于支付服务这样的应用，还是比较推荐 `sync_binlog = 1`.
对于高并发事务的系统来讲, `sync_binlog`设置为0或1,可以产生五倍甚至更多的写入性能差距.


### 1.1.9 MySQL启动脚本

`vim /etc/init.d/mysqld`

```bash
# chkconfig: 2345 64 36
port=1036
mysql_user="root"
mysql_pwd="abc-123"
lockfile="/var/lock/subsys/mysqld"
CmdPath="/application/mysql/bin"
mysql_sock="/application/mysql/mysql.sock"
if [ -f /etc/rc.d/init.d/functions ]
then
  . /etc/rc.d/init.d/functions
else
  function action {
    echo "$1"
    shift
    $@
  }
  function success {
    echo -n "Success"
  }
  function failure {
    echo -n "Failed"
  }
fi
function Msg(){
	if [ $? -eq 0 ];then
		action "$1" /bin/true
	else
		action "$1" /bin/false
	fi
}
#startup function
function start_mysql()
{
    if [ `netstat -tunlp|grep $port|wc -l` -eq 0 ];then
		if [ -e $mysql_sock ];then
			/bin/rm Rf $mysql_sock
		fi
		touch $lockfile
		/bin/sh ${CmdPath}/mysqld_safe --defaults-file=/etc/my.cnf 2>&1 > /dev/null &
		Msg "Mysql_start"
    else
		printf "MySQL is running...\n"
		exit
    fi
}

#stop function
function stop_mysql()
{
    if [ `netstat -tunlp|grep $port|wc -l` -eq 0 ];then
		printf "MySQL is stopped...\n"
		exit
    else
		${CmdPath}/mysqladmin -u ${mysql_user} -p${mysql_pwd}  shutdown
		Msg "mysql stopped"
   fi
}

#restart function
function restart_mysql()
{
    printf "Restarting MySQL...\n"
    stop_mysql
    sleep 2
    start_mysql
}

#status function
function status_mysql()
{
    if [ `netstat -tunlp|grep $port|wc -l` -eq 0 ];then
		printf "MySQL is NOT running!\n"
		exit
    else
		printf "MySQL is running! \n"
		exit
    fi
} 

case $1 in
start)
    start_mysql
;;
stop)
    stop_mysql
;;
restart)
    restart_mysql
;;
status)
    status_mysql
;;
*)
	 printf "Usage:$0 {start|stop|restart|status}\n"
esac
exit 0
```

```bash
chmod +x /etc/init.d/mysqld
chkconfig --add mysqld
chkconfig mysqld on
service mysqld start
```

### 1.1.10 基本优化

添加密码：

```bash
mysqladmin –uroot password  'abc-123'
```

修改密码：

```bash
mysqladmin –uroot –pabc-123 password 'abcd-1234'
```

删除不必要的用户：

```sql
select user,host from mysql.user;
drop user ''@'localhost';
```

## 1.2 多实例安装

多实例配置方案有2种实现方式：
1. 多配制文件部署
2. 单配置文件部署

推荐使用方案1，虽然方案2是mysql官方推荐方式，但是因为单配制文件耦合度太高容易造成故障。

### 1.2.1 创建多实例配置文件和启动文件

创建多实例mysql, 3306和3307的配置文件和启动文件, 每个配置文件制定好相应的端口号和路径即可

### 1.2.2 初始化mysql多实例数据库

```bash
/application/mysql/scripts/mysql_install_db  --user=mysql --datadir=/data/3306/data    --basedir=/application/mysql 
/application/mysql/scripts/mysql_install_db  --user=mysql --datadir=/data/3307/data    --basedir=/application/mysql
```

### 1.2.3 启动多实例数据库

```
/data/{3306,3307}/mysql start
```

### 1.2.4 多实例数据库管理

登录数据库

```
mysql –uroot –pabc-123 –S /data/3306/mysql.sock 
mysql –uroot –pabc-123 –S /data/3307/mysql.sock
```

远程登录

```
mysql  -uroot –p'abc-123' –h 10.0.0.2 –P 3307
```

重启数据库

```
mysqladmin –uroot –p'abc-123' –S /data/3306/mysql.sock shutdown
mysqladmin –uroot –p'abc-123' –S /data/3307/mysql.sock shutdown
/data/{3306,3307}/mysql start
```

~~强制关闭数据库的方法
killall mysqld
pkill mysqld
killall -9 mysqld~~
强制关闭数据库**非常危险**，可能造成无法启动问题，以下是2个案例：
[http://oldboy.blog.51cto.com/2561410/1431161](http://oldboy.blog.51cto.com/2561410/1431161)
[http://oldboy.blog.51cto.com/2561410/1431172](http://oldboy.blog.51cto.com/2561410/1431172)


# 2. MySQL数据库优化

## 2.1 服务器物理硬件优化

平衡内存和硬盘资源，计算机包含一个金字塔型的缓存体系，更快，更小，更昂贵的存在顶端：
![高性能MySQL p383](./1468293150317.png)

MySQL的*收益递减点*配置大约是256GB RAM, 32核CPU以及一个PCIe flash
当服务器硬件扩展到24个CPU核心时, MySQL性能趋于平缓, 当内存超过128GB时也是这样, 为了充分利用数据库服务器的性能, 不推荐使用虚拟化这样会造成系统资源的大量浪费, 推荐使用多实例. 

### 2.1.1 硬盘

**磁盘I/O是制约MySQL性能的最大因素之一**，推荐使用SAS 15000转以上磁盘作**RAID10**，如果资金允许推荐选用SSD磁盘代替sas磁盘，尽量少使用RAID5,使用RAID要选用带有电池保护单元的RAID控制器。
注：传统HDD硬盘，物理尺寸也有区别：越小的硬盘，移动读取磁头需要的时间就越短。服务器级别的2.5英寸磁盘性能往往比通转速的更大的磁盘要快，而且更省电。

### 2.1.2 CPU

更快的CPU还是更多的CPU？
调优服务器的2个目标：
1. 低延时（快速响应）
做到这一点，需要告诉CPU，因为每个查询只能使用一个CPU。
2. 高吞吐
如果运行很多查询语句，则可以从多个CPU处理查询中收益。这只是理论，以为MySQL还不能在多个CPU中完美的扩展，能用多少个CPU还是有极限的，不是越大越好。MySQL5.1以前版本闲置非常严重，MySQL5.5之后，可以放心扩展到16个以上。
注: 1.MySQL复制主要需求高速CPU，多CPU对复制帮助不大。
    2.CPU使用率是最直接的指标，如果持续一段时间CPU使用率大于80%，表明CPU出现瓶颈。

### 2.1.3	内存

内存价格越来越便宜建议使用大内存，内存最小不能小于2G，建议大于4G，另外服务器大内存**分配给MySQL的innodb的内存池大于数据库的数据文件大小**，这样的MySQL性能**最优**。

### 2.1.4	补充

维护MySQL服务器时要确认自己的服务器是什么类型的机器：
CPU密集型，I/O密集型机器，内存交换机器。
工具：vmstat（vmstat 5）,iostat（iostat -dx 5）
CPU密集型：
vmstat输出通畅在us列有很高的值（花费在非内核代码上的CPU时钟），也可能在sy列有很高的值（表示CPU利用率，超过20%就认为是很高的使用效率）。
如果查看iostat输出，可以发现磁盘的利用率低于50%。
I/O密集型：
如果是I/O密集型，那么CPU花费大量时间是在等待I/O请求。vmstat显示很多处理器在非中断休眠（b列）状态，并且wa值很高。
iostat显示的硬盘一直很忙。
内存交换：
vmstat的swpd列，si和so列都有可能很高。


## 2.2 MySQL配置文件优化

### 2.2.1 mysql性能影响较大参数

```
[mysqld]
包括mysqld服务启动的参数，包括mysql的目录和文件，通信，网络，信息安全，内存管理，优化，查询缓存区，mysql日志等。
port = 3306
mysqld服务运行时端口号。
socket = /tmp/mysql.sock
客户端可以不通过网络直接通过socket连接mysql。
skip-external-locking
MySQL选项以避免外部锁定。该选项默认开启
skip-name-resolve
禁止mysql对外部链接进行DNS解析，使用这一选项可以消除Mysql进行DNS解析的时间。如果开启此项，则所有远程主机连接授权都要使用ip地址进行，否则无法链接。
wait_timeout = 100
指定请求的最长连接时间，对于4GB内存服务器一般设置为5-10.
back_log = 500
mysql暂时停止响应新请求之前，短时间内的多少个请求可以保存在堆栈中。该参数指定到来的TCP/IP链接的监听队列大小。针对linux推荐是小于512的整数。
max_connections=1024
指定mysql的最大链接线程数。如果访问应用层出现Too Many Connections的错误提示，增加该值。
max_connect_errors=10000
每个主机请求链接异常中断的最大次数。当超过该次数，mysql服务器将禁止host的链接请求。知道mysql重启或者是通过flush hosts命令清空host的相关信息。
key_buffer_size=64M
指定用于索引的缓冲区大小，增加它可获得更好的索引处理性能。
max_allowed_packet=128M
设定在网络传输中一次消息传输量的最大值。默认为1MB，最大为1GB，必须为1024的倍数，单位为字节。
table_cache=3096
表示高速缓冲区的大小。当mysql访问一个表时，如果在mysql表缓冲区中还有空间，就打开放入缓冲区，这样的好处是加速访问表中的内容。一般来说需要查看Open_tables和Opened_tables，用以判断是否需要table_cache的值，如果Open_tables接近table_cache，并且Opened_tables这个值在逐步增加，那就要考虑增加。
sort_buffer_size=512K
设定排序时的缓冲区大小。
read_buffer_size=512K
读查询使用的缓存区大小。
read_rnd_buffer_size=512k
进行随机读的时候使用的缓存区。
join_buffer_size=512K
联合查询使用的缓存区大小。
tmp_table_size=64M
设置内存临时表最大值。如果超过该值就将临时表写入硬盘。
max_heap_table_size=64M
定义了用户可以创建的内存表(memory table)的大小。这个值用来计算内存表的最大行数值。这个变量支持动态改变。
thread_cache_size=64
thread_cache可以缓存的连接线程最大数量。当有很多线程的时候可以增加这个值改善系统性能。通过比较connections和threads_created状态的变量，来确认是否增加。一般情况下1G内存设置为8,2GB设置为16,3GB为32,4G以上设置为64及更大。
thread_concurrency=32
该参数一般设置为cpu*2。
thread_stack=256K
mysql每个堆栈的大小，默认值足够大，不需要修改。
innodb_flush_log_at_trx_commit=2
设置为0就是等到innodb_log)buffer_size队列满足再统一存储，默认为1是最安全的。
innodb_thread_concurrency=0
服务器cpu为几就设置为几。
innodb_io_capacity=2000
修改innodb的I/O能力，ssd需要更高的值5000。
transaction_isolation=READ-COMMITTED
MySQL支持4种事务隔离级别，他们分别是：READ-UNCOMMITTED, READ-COMMITTED, REPEATABLE-READ, SERIALIZABLE.如没有指定，MySQL默认采用的是REPEATABLE-READ，ORACLE默认的是READ-COMMITTED
```

### 2.2.2	72G内存my.cnf

```
[client]
port = 3306
socket = /tmp/mysql.sock

[mysqld]
server-id=1
port = 3306
user = mysql
basedir = /usr/local/mysql
datadir = /mysqlData/data
tmpdir = /mysqlData/tmp
socket = /tmp/mysql.sock
skip-external-locking
skip-name-resolve
default-storage-engine = INNODB
character-set-server = utf8
wait_timeout = 100
connect_timeout = 20
interactive_timeout = 100
back_log = 500
myisam_recover
event_scheduler = ON

######binlog#######
log-bin = /mysqlLOG/logs/mysql-bin
binlog_format = row
max_binlog_size = 128M
binlog_cache_size = 2M
expire-logs-days = 5

#######replication#####
slave-net-timeout  = 10
rpl_semi_sync_master_enabled = 1
rpl_semi_sync_master_wait_no_slave = 1
rpl_semi_sync_master_timeout  = 1000
rpl_semi_sync_slave_enabled  =1
skip-slave-start
log_slave_updates  = 1
relay_log_recovery = 1

########slow log#########
slow_query_log = 1
slow_query_log_file = /mysqlLOG/logs/mysql.slow
log_query_time = 2

#########error log#########
log-error = /mysqlLOG/logs/error.log

##########per_thread_buffers######
max_connections=1024
max_user_connections=1000
max_connect_errors=10000
key_buffer_size=64M
max_allowed_packet=128M
table_cache=3096
table_open_cache=6144
table_definition_cache=4096
sort_buffer_size=512K
read_buffer_size=512K
read_rnd_buffer_size=512k
join_buffer_size=512K
tmp_table_size=64M
max_heap_table_size=64M
query_cache_type=0
query_cache_size=0
bulk_insert_buffer_size=32M
thread_cache_size=64
thread_concurrency=32
thread_stack=256K

#########innoDB####################
innodb_data_home_dir=/mysqlData/data
innodb_log_group_home_dir=/mysqlLOG/logs
innodb_data_file_path=ibdata1:2G:autoextend
innodb_buffer_pool_size=50G			#75%
innodb_buffer_pool_instances=8
innodb_additional_mem_pool_size=16M
innodb_log_file_size=1024M
innodb_log_buffer_size=64M
innodb_log_files_in_group=3
innodb_flush_log_at_trx_commit=2
innodb_lock_wait_timeout=10
innodb_sync_spin_loops=40
innodb_max_dirty_pages_pct=90
innodb_support_xa=1
innodb_thread_concurrency=0
innodb_thread_sleep_delay=500
innodb_file_io_threads=4
innodb_concurrency_tickets=1000
log_bin_trust_function_creators=1
innodb_flush_method=O_DIRECT
innodb_file_per_table
innodb_read_io_threads=16
innodb_write_io_threads=16
innodb_io_capacity=2000
innodb_file_format=Barracuda
innodb_purge_threads=1
innodb_purge_batch_size=32
innodb_old_blocks_pct=75
innodb_change_buffering=all
transaction_isolation=READ-COMMITTED

[mysqldump]
quick
max_allowed_packet=128M
myisam_max_sort_file_size=10G

[mysql]
no-auto-rehash

[myisamchk]
key_buffer_size=64M
sort_buffer_size=256k
read_buffer=2M
write_buffer=2M

[mysqlhotcopy]
interactive-timeout
[mysqld_safe]
open-files-limit=28192
```

## 2.3	MySQL上线后根据status优化
MySQL上线后，稳定运行一段时候后，可以根据服务器的status状态进行适当优化，

```
mysql>show global status;
```

具体的用法可以根据自己的喜好：

```
mysql>show global status like '查询值%';
```

### 2.3.1 慢查询


定位sql语句中效率低下的query语句，需要打开慢查询日志（Slow Query Log）

```
mysql> show variables like '%slow%';
mysql> show global status like '%slow%';
```

另外也可以使用mysqldumpslow进行查询，举例

```
mysqldumpslow -s c -t 20 host - slow.log
```

### 2.3.2.连接数

如果经常碰见MySQL:ERROR 1040：Too many connections的情况，一种情况是访问量确实非常大，MySQL确实扛不住，这样就需要进行服务器的升级。另外一种是因为MySQL的`max_connections`值过小。

```
mysql> show variables like 'max_connections';
```

查询一下服务器的最大连接数。

```
mysql> show global status like 'max_used_connections';
```

查询服务器过去使用的最大连接数。
设置最理想的连接数的比例是：

```
max_user_connections/max_connections ≈	0.85
```

### 2.3.3 key_buffer_size

作为MyISAM表索引缓存空间使用，此参数对MyISAM表影响很大。

```
mysql > show variables like 'key_buffer_size';
```

查看分配给`key_buffer_size`的大小。

```
mysql > show global status like 'key_read';
```

查询当前数据库的使用情况:

```
key_read_requests 总共的索引读取请求
key_reads 多少情况内存中没有找到
```

计算索引未命中缓存的概率：

```
key_cache_miss_rate = key_reads / key_read_requests
```

如果`key_cache_miss_rate`的值大于0.01%小于0.1%说明`key_buffer_size`合适。

```
mysql > show global status like 'key_blocks_u%';
```

`key_blocks_unused`表示未使用的缓存簇（blocks）数，`key_blocks_used`表示曾经使用过的最大的簇数。

```
key_blocks_used / (key_blocks_unused+key_blocks_used)≈0.8
```

如果大于这个值就增加`key_buffer_size`。

### 2.3.4 临时表

```
mysql > show global status like 'created_tmp%';
```

每次创建历史表时，`created_tmp_tables` 都会增加，如果在磁盘上创建的临时表那么`created_tmp_disk_tables`。`created_tmp_files`表示mysql创建的临时文件数。
理想值如下

```
created_tmp_disk_tables / created_tmp_tables <= 0.25
mysql > show variables where varable_name in ('tmp_table_size','max_heap_table_size');
```

### 2.3.5 打开表的情况
`open_tables`表示打开表的数量，`opened_tables`表示打开过的表的数量。

```
mysql > show global status like 'open%tables%';
```

如果`opened_tables`打开数量过大，说明`table_open_cache`的值可能过大。

```
mysql > show variables like 'table_open_cache';
```

合适的值：

```
open_tables / Opened_tables >= 0.85
open_tables / table_open_cache <= 0.95
```

### 2.3.6 进程使用情况
`thread_cache_size` 当客户端断开后，服务器会缓存起这个请求线程。

```
mysql > show global status like 'thread%';
```

如果`threads_created`值很大的话，说明Mysql一直在创建进程，这样比较好费资源可以适当的增大`thread_cache_size`的值。

```
mysql > show variables like 'thread_cache_size';
```

### 2.3.7 查询缓存(query cache)
`query_cache_size` 设置MySQL的Query Cache大小，query_cache_type 设置查询缓存的类型。

```
mysql > show global status like 'qcache';
```

`qcache_free_blocks`:缓存中相邻内存块的个数。数目大说明碎片太多，使用FLUSH QUERY CACHE会对缓存中的碎片进行整理。
`qcache_free_memory`:缓存中的空闲内存。
`qcache_hits`:多少次命中，主要作用是查看Query Cache基本效果。
qcache_inserts:插入次数，每次插入一个查询是就增加1.命中次数除以插入次数就是命中比率。
`qcache_lowmen_prunes`结合`qcache_free_memory`相互结合，能够很清楚的知道系统的query cache内存大小是否够用，是否非常频繁地出现因为内存不足而有query 被换出的情况。
`qcache_not_cached`:不适合进行缓存的查询数量，通畅是由于这些查询不是SELECT语句或用了now()之类的函数。
`qcache_queries_in_cache`:当前缓存的查询（响应）数量。
`qcache_total_blocks`:缓存中簇的数量。

```
mysql > show variables like 'query_cache';
```

`query_cache_limit`:超过此大小的查询讲不缓存。
`query_cache_min_res_unit`:缓存块的最小值。
`query_cache_size`:查询缓存大小。
`query_cache_type`:缓存类型，决定缓存什么样的查询。
`query_cache_min_res_unit`:默认为4KB，设置过大对大数据查询有好处，如果查询小数据会造成内存碎片和浪费。

```
查询缓存碎片率=qcache_free_blocks/qcache_total_blocks
```

如果缓存碎片率超过20%就需要使用FLUSH QUERY CACHE整理缓存碎片，或者减少`query_cache_min_res_unit`。

```
查询缓存利用率=(query_cache_size-qcahe_free_memory)/query_cache_size
```

查询缓存利用率在25%以下说明`query_cache_size`设置过大，可适当减小。如果查询缓存利用率大于80%而且`qcache_lowmem_prunes>50`说明`query_cache_size`太小，或者是碎片太多。

```
查询缓存命中率=（qcache_hits	-qcache_inserts）/qcache_hits
```

### 2.3.8 文件打开数
当`open_files`大于`open_files_limit`,数据库会卡住，导致应用层无法操作数据库。

```
mysql > show global status like 'open_files';
mysql > show variables like 'open_files_limit';
```

合适的值：

```
open_files / open_files_limit <=0.75
```

### 2.3.9 innodb_buffer_pool_size
一般建议设置为整个物理内存的50%-80%，这是纯innodb的使用环境，但是在实际的操作环境中因为服务器上运行的服务可能更多已经mysql可能使用myisam和innodb，可以初始给`innodb_buffer_pool_size`配置一个值，之后通过status进行调整。

```
mysql > show status like 'innodb_buffer_pool_%';
```

innodb的read命中率：

```
(innodb_buffer_pool_read_requests - innodb_buffer_pool_reads) / innodb_buffer_pool_read_requests
```

innodb的write命中率：

```
innodb_buffer_pool_pages_data/innodb_buffer_pool_pages_total
```

如果以上2个值过小的话要提高`innodb_buffer_pool_size`的大小。
注：等mysql服务器上线运行稳定后使用`tunning-primer.sh`对服务器的参数进行检查看参数是否合理。


# 3 MySQL 备份和还原

## 3.1	MySQL备份还原

### 3.1.1	冷备份(物理备份)

用于非核心业务，这类服务允许中断，冷备份的特点是速度快，恢复时也最为简单，通畅是直接复制文件来实现冷备份。

备份过程：
1. 关闭MySQL服务进程

```
/etc/init.d/mysql stop
```

2. 把data数据目录（包括ibdata1）和日志目录（包含 `ib_logfile0`, `ib_logfile1`, `ib_logfile2`）复制到磁带机或者本地的另一块磁盘。


恢复过程：
1. 用复制的数据目录和日志目录替换现有的目录。
2. 启动mysql

```
/etc/init.d/mysql start
```

### 3.1.2	逻辑备份

逻辑备份一般用于数据迁移或者数据量很小的情况，逻辑备份采用的是数据导出的方式进行备份。

- 备份过程：
innodb和myisam引擎不同，备份也有稍许不同：
innodb引擎：

```
mysqldump -uroot -p'abc-123' -A -B -F --quick --events \
--flush-privileges --single-transaction --triggers --routines  \
--hex-blob --master-data=1 --default-character-set=utf8  \	
>/opt/full_dump_backup_timestamp.sql
```

myisam引擎：

```
mysqldump -uroot -p'abc-123' -A -B -F --quick --events \
--flush-privileges –x --master-data=1  --triggers \	
--routines --hex-blob --default-character-set=utf8	 \	
>/opt/full_dump_backup_timestamp.sql
```

关键参数解释：
- -A 				备份所有库
- -B 				指定多个库，增加建库语句和use语句
- --compact 		去掉注释，适合调试
- -F 				刷新binlog
- --master-data 	增加binlog日志文件名和位置点。
- -x 				锁所有表
- -d  			只备份表结构
- -t 				只备份数据
- --quick			不缓冲查询，直接导出到标准输出。
- --events		导出事件
- --triggers		导出触发器
- --routines		导出存储过程以及自定义函数
- --hex-blob 	使用十六进制格式导出二进制字符串字段。如果有二进制数据就必须使用该选项。影响到的字段类型有BINARY、VARBINARY、BLOB
- --master-data	该选项将binlog的位置和文件名追加到输出文件中。如果为1，将会输出CHANGE MASTER 命令；如果为2，输出的CHANGE  MASTER命令前添加注释信息。
- --single-transaction 适合innodb事物数据库备份。保证备份的一致性实际上就是设定本次会话的隔离级别为:REPEATABLE  READ,以确保在这次会话中不会再有新数据提交。
查看备份的数据：

```bash
grep -Ev “#|\*|--|^$” /opt/full_dump_backup_timestamp.sql
```

恢复过程：
- mysql标准恢复

```
mysql -uroot -p'abc-123' </opt/full_dump_backup_timestamp.sql
```

- 登录mysql之后恢复

```
mysql>source /opt/full_dump_backup_timestamp.sql
```

### 3.1.3	热备份

热备份是直接复制物理数据文件，和冷备份一样，但是热备份可以不停机直接复制，一般用于7*24小时不间断的重要核心业务。官方的InnoDB Hot Backup付费，使用Percona的xtrabackup可以和官方获得一样的功能。
备份过程：

```
innobackupex --user=root --password=abc-123 --defaults-file=/etc/my.cnf  /back/
```

- --user:指定链接数据库的用户名。
- --password:指定连接数据库的密码。
- --defaults-file:指定数据库的配置文件。

恢复过程：
1. 停止MySQL数据库

```
/etc/init.d/mysql stop
```

2. 删除老数据库中的数据文件和事务日志文件
3. 将备份文件中的日志应用到备份文件中的数据文件上

```
innobackupex --defaults-file=/etc/my.cnf --apply-log /back/2016-02-23_15-40-00
```

4. 将备份文件中的数据恢复到数据库

```
innobackupex --defaults-file=/etc/my.cnf --copy-back /back/2016-02-23_15-40-00/
```

5. 数据恢复完成后，需要修改相关文件的权限

```
chown -R mysql.mysql /usr/local/mysql/data/
```

6. 启动mysql数据库

```
/etc/init.d/mysql start
```

备份到远程服务器
本地服务器IP：192.168.0.10
远程服务器IP：192.168.0.11
备份命令：

```
innobackuped --defaults-file=/etc/my.cnf --stream=tar /usr/local/mysql/data |ssh root@192.168.0.11 cat ">" /back/backup.tar
```

恢复命令：

```
tar -ixvf backup.tar
```

之后按照本地备份的恢复过程进行恢复。


### 3.1.4	增量备份和恢复

逻辑备份的增量备份方法：
逻辑备份的增量备份方法分为2种：
1. 每周一凌晨1点进行全量备份，然后保存1周的binlog日志文件。
2. 周一进行全量备份，周二至周五进行增量备份。

脚本实现：
星期一凌晨1点使用计划任务执行全量备份脚本

```bash
#!/bin/bash
#MySQL全量备份脚本，建议在slave上进行，并且slave开启log_slave_updates=1
mkdir /backup
cd /backup
dateDIR=`date +" %y-%m-%d"`
mkdir -p $dateDIR/data
path=/usr/local/mysql/data
mysqldump -uroot -p'abc-123' -A -B -F --quick --events \
--flush-privileges --single-transaction --triggers --routines --hex-blob \
--master-data=1 --default-character-set=utf8 |gzip	>/backup/$dateDIR/data/full_dump_backup_timestamp.sql

binlog_rm=`tail -n 1 $path/mysql-bin.index | sed 's/.\///'`
mysql -uroot -pabc-123 -e "purge binary logs to `$binlog_rm`"
```

星期二到星期6凌晨1点执行增量备份脚本

```bash
#!/bin/bash
#MySQL增量备份脚本，建议在slave上进行，并且slave开启log_slave_updates=1
cd /backup
dateDIR=`date +" %y-%m-%d"`
mkdir -p $dateDIR/data
path=/usr/local/mysql/data
mysqladmin -uroot -pabc-123 flush-logs
binlog_cp=`head -n -1 $path/mysql-bin.index | sed 's/.\///'`
for i in $binlog_cp
do
mysql -uroot -pabc-123 -e "\!cp -p $path/$i/backup/$dateDIR/data/;"
done
binlog_rm=`tail -n 1 $path/mysql-bin.index | sed 's/.\///'`
mysql -uroot -pabc-123 -e "purge binary logs to `$binlog_rm`"
```

热备份的增量备份方法
1. 先进行全量备份

```
innobackupex --user=root --password=abc-123 --defaults-file=/etc/my.cnf  /back/fuallback
```

2. 再进行增量备份

```
innobackupex --defaults-file=/etc/my.cnf --incremental /back/incrementbak --incremental-basedir=/back/fullback/2016-2-23_17-19-00/
```

进入到备份目录，查看是全量备份还是增量备份方法：

```
cat xtrahackup_checkpoints
backup_type = full-backuped / incremental
```

恢复方法：
1. 先恢复增量事物日志

```
innobackupex --applu-log --redo-only /back/fullback/全量备份文件夹 --incremental-dir=/back/incrementbak/增量备份文件夹
```

2. 再恢复全量事物日志

```
innobackupex --apply-log /back/fullback/全量备份文件夹
```

3. 将备份文件中的数据恢复到数据库中

```
innobackupex --copy-back /back/fullbak/全量备份文件夹
```

4. 修改权限

```
chown -R mysql.mysql /usr/local/mysql/data/
```

5. 启动数据库

```
/etc/init.d/mysql start
```

# 4 MySQL主从复制

## 4.1	主从复制概述
MySQL复制大部分是向后兼容的，新版本可以作为老版本的备库，但是反过来是不可行的。
复制通常不会带来开销，主要是写二进制日志文件带来的开销。

## 4.2	复制解决的问题：

- 数据分布

因为可以随时停止或开始复制，并且可以在不同的地理位置来分布数据备份，我们可以根据自己的需求确认复制的分布，即使在不稳定的网络环境下，远程复制也能工作，但是为了保持很低的复制延迟，最好使用稳定的，低延迟网络连接。

- 负载均衡

可以通过复制把数据分配到多台服务器，如操作负载到多台服务器上，实现对读密集型应用的优化，实现方式比较简单可以开发代码实现也可使用DNS轮询实现。

- 备份

复制对备份进行了有效补充。

- 高可用和故障切换

复制能够帮助应用程序避免MySQL单点失败。

- MySQL升级测试

让要升级的MySQL版本作为备库，保证在升级MySQL之前，之前数据满足在备库上运行。

## 4.3	主从复制工作原理

复制只要是三个步骤：
1. 在主库上把数据更改记录到二进制日志（Binary Log）中。
2. 备库将主库上的日志复制到自己的中继日志（Relay Log）中。
3. 备库读取中继日志中的事件，将其重放到备库数据库中。

![主从复制](https://github.com/Shannon1/documents/blob/master/image/repl.png)

具体实现原理：
1. slave 服务器上执行start slave，开启主从复制开关。
2. slave 服务器的IO线程通过master上授权的复制用户权限链接master服务器，并且请求binlog的文件位置（日志文件名和位置这需要配置主从服务时change master命令指定）之后发送binlog日志。
3. master服务器接收到来自slave服务器的io线程请求后，master服务器上负责复制io线程根据slave的io线程请求读取指定的binlog日志文件和指定位置，然后发给slave端的io线程。
4. 当slave的io线程获取到master服务器的io线程后依次写入relay log文件最末端，最新的binlog位置和名称记录到master-info文件中
5. slave的sql线程检测relay log增加的内容，然后把log文件中的内容解析成master端执行的sql语句，在自身slave服务器上执行sql语句，应用完毕后清理日志。

## 4.4 配置主从复制

### 4.4.1	环境检查

确认master服务器上的binlog日志功能打开。
**master的server-id和slave的server-id不同。**

### 4.4.2	创建复制账户
在主库和备库都创建该账户

```sql
grant replication slave, replication client on *.* to slave@'10.0.0.%' identified by 'abc-123';
```

### 4.4.3 master服务器进行全备
innodb引擎：

```
mysqldump -uroot -p'abc-123' -A -B -F --quick --events \
--flush-privileges --single-transaction --triggers --routines \
--hex-blob --master-data=1 --default-character-set=utf8  \
>/opt/full_dump_backup_timestamp.sql
```

### 4.4.4	slave配置
1. 把master的备份文件复制到slave，然后进行恢复。
2. slave导入语句：

```
mysql> change master to
master_host='10.0.0.2',
master_port=3306,
master_user='slave',
master_password='abc-123';
mysql> start slave;
```

3. 验证是否生效

```
mysql>show slave status\G
```

进行状态查看，如果`Slave_IO_Running`和`Slave_SQL_Running`都是Yes说明主从复制已经生效。


## 4.5半同步复制

默认情况下MySQL的复制功能是异步的，异步复制可以提供最佳的性能，但是这样意味着高风险。为解决这个问题引入了半同步复制，该模式可以确保从服务器接收完主服务器发送的binlog日志文件并且写入自己的relaylog后会给主服务器一个反馈。


### 4.5.1半同步复制安装和配置
半同步复制不需要进行单独安装，此功能集成到mysql中，只需要master和slave启动即可：
- master：

安装插件：

```
mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
```

启动模块：

```
mysql> SET GLOBAL rpl_semi_sync_master_enabled = 1;
```

设置超时时间：

```
mysql> SET GLOBAL rpl_semi_sync_master_timeout = 1000;
```

永久生效：在my.cnf中添加

```
rpl_semi_sync_master_enabled = 1
rpl_semi_sync_master_timeout = 1000;
```

- slave：

安装插件：

```
mysql> INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
```

启动模块：

```
mysql> SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

重启进程使其模块生效：

```
mysql> STOP SLAVE IO_THREAD; START SLAVE IO_THREAD;
```

永久生效：在my.cnf中添加

```
rpl_semi_sync_slave_enabled = 1
```

### 4.5.2查看半同步复制生效

master和slave运行：

```
mysql>SHOW GLOBAL STATUS LIKE "rpl_semi%";
```

master:
![半同步复制Master](https://github.com/Shannon1/documents/blob/master/image/semi_master.png)

slave:
![半同步复制Slave](https://github.com/Shannon1/documents/blob/master/image/semi_slave.png)

### 4.5.3参数说明
- `rpl_semi_sync_master_enabled`，表示master启用半同步复制模式。
- `rpl_semi_sync_master_timeout = 1000`，该参数表示如果主库在某次事物中的等待时间超过10S，就会降级成为异步复制模式，不在等待slave，如果主库在探测到slave恢复了，就会自动恢复成半同步复制。
- `rpl_semi_sync_master_wait_no_slave`，表示是否需要master每个事物提交后都要等待slave的接收确认信号。默认为ON，即每一个事物都要等待。如果为OFF，如果slave追赶上后，也不会启用半同步复制，需要手工开启。
- `rpl_semi_sync_master_trace_level=32`, 用于开启半同步复制模式时的调试界别，默认为32.
- `rpl_semi_sync_slave_enabled`, 表示在slave上开启半同步复制模式。
- `rpl_semi_sync_slave_trace_level=32`, 调试级别。


## 4.6 主从同步注意事项

### 4.6.1	基于语句还是基于行

- 基于语句的复制

实现简单，理论上简单地记录和执行语句，能够确保主备保持同步。
二进制日志里的时间更加紧凑，不会占用更多的带宽。
此模式在复杂语句环境下无法保证同步。
此引擎虽然在mysql5.5还支持但是面临淘汰风险，因为此种方式在数据库领域很少见。

优点：
基于语句的复制方式一般允许更灵活的操作。基于语句复制的过程就是执行SQL语句，如果出现问题定位迅速。

缺点：
很多情况下基于语句的模式无法正确复制，如果使用了触发器或者存储过程就不能使用基于语句的复制。


- 基于行的复制

做大好处是可以正确地复制每一行，一些语句可以被更加有效地复制。
优点：
1. 能适应所有的场景，SQL构造，触发器，存储过程等都能正确执行，只有当你在试图在备库修改schema才可能导致复制失败。
2. 减少锁的使用。
3. 在日志文件中更好的记录数据变更，这样有利于某些数据恢复。
4. 基于行的复制占用的CPU更少。
5. 更好的解决数据不一致的情况。
缺点：
语句没有记录在日志里。

注意：建议在主从复制模式里面推荐使用基于行的复制（row）另外mixed（混合模式）都可以。

### 4.6.2	复制过滤器

存在2种复制过滤方式：

1. 在主库上过滤记录到二进制日志中的事件（此种方式不推荐）

`binlog_do_db`和`binlog_ignore_db`
这种使用方式不仅可能会破坏复制，还可能导致某个时间点的备份进行数据恢复失败，可能会造成数据永久丢失并且无法恢复。

2. 在备库上过滤记录到中继日志的事件

在备库上，可以通过`replicate_*`选项，在中继日志中读取事件时进行过滤，你可以复制或忽略一个或多个数据库，把一个数据库重写到另外一个数据库，或者利用LIKE的模式复制或忽略数据表。

### 4.6.3关注点

1. 主从复制按照通用标准，如果主库接近满负载，不应该建立超过10个以上的备库。

2. 可以通过参数`slave_compressed_protocol`来节约带宽，对跨机房复制很有好处，此选项可以减少80%的数据传输量，但是压缩会耗费系统资源。
如果该选项设置为1，如果从服务器和主服务器均支持，使用压缩从服务器/主服务器协议。

3. 检查主备是否一致，可以使用Percona Toolkit里的`pt-table-checksum`确认主库和备库数据是否一致。
在主库上运行该工具：

```
pt-table-checksum --replicate=test.checksum <master_host>
```

4. `sync_binlog=1`,如果开启此选项，mysql每次在提交事物前都会将二进制日志同步到磁盘上，保证服务器在崩溃是不会丢失事件。

5. 如果无法忍受服务器崩溃造成的表损坏，请勿使用myisam引擎，推荐使用innodb引擎。

6. `innodb_flush_log_at_trx_commit`,如果是innodb引擎，设置为1可以尽可能的保证数据的完整性。但是此选项会造成服务器的压力增加，如果出现服务器资源不够的话可以设置成2。

7. 错误处理：

```
mysql>show slave status;
```
![Alt text](https://github.com/Shannon1/documents/blob/master/image/repl_error.png)

 
解决思路：

```
mysql>stop slave;
mysql>set global sql_slave_skip_counter=1;
mysql>start slave;
或者：
vim /etc/my.cnf
slave-skip-errors=1032,1062,1007
```

8. master宕机

解决思路：
一，登录所有从库slave：

```
mysql>stop slave io_thread;
mysql>show processlist \G
```

知道看到 Has read all relay log;表示所有从库更新完毕。

二，登录所有从库slave，分别查看master.info文件

```
cat  /application/mysql/master.info
```

三，找到master.info中更新最快的然后选择这台为master

```
mysql>stop slave;
mysql>retset master;
```

四，删除master.info和relay-log.info

```
rm -f master.info relay-log.info
```

五，提升从库为主库

```
vim /etc/my.cnf
log-bin=mysql-bin
如果存在log-slave-updates read-only等注释掉。
service mysqld restart
```

六，如果厡master能进入，从binlog中更新原先的数据：
七，其他从库更换新的master

```
mysql>stop slave;
mysql>change master to master_host=’10.0.0.3’;
mysql>start slave;
mysql>show slave status \G
```

9. slave宕机

思路就是重新配置这台从库，方法同上面的配置。

10. 从库只允许只读

slave的my.cnf

```
[mysqld]
read-only=on
```

操作数据的用户不允许带有super和all privileges权限。

# 5 MySQL高可用方案

## 5.1 MySQL自带的Replication架构

![Alt text](https://github.com/Shannon1/documents/blob/master/image/replication.png)

此架构比较简单但是对读多写少的数据库使用有非常好的性能，另外因为MySQL服务对双主模式的处理非常不好，不建议使用双主模式，master服务器如果主要处理写入数据的话，硬件资源相应充足情况下不会成为瓶颈，此集群的主要使用方法参考图进行分析。
实现方法：
1. 参考第四章介绍的主从复制进行配置即可。
2. 多slave服务器使用可以使用程序代码进行读负载均衡，也可使用DNS轮询，也可使用lvs进行写负载均衡。

## 5.2 MMM

MMM即`master-master Replication Manager for MySQL`(MySQL主主复制管理器)，此项目来自Google。
MMM主要功能由三个脚本提供：
1. `mmm_mond`:监控守护进程，决定节点的移除。
2. `mmm_agentd`:运行在MySQL服务器上的代理守护进程，通过简单远程服务提供给监控节点。
3. `mmm_control`:通过命令行管理`mmm_mond`进程。

### 5.2.1 工作逻辑图

![MMM](https://github.com/Shannon1/documents/blob/master/image/mmm.png)

### 5.2.2	配置环境：

```
MMM_monitor:192.168.0.6
MySQL_master1:192.168.0.2
MySQL_master2:192.168.0.3
MySQL_slave1:192.168.0.4
MySQL_slave2:192.168.0.5
VIP_Writer:192.168.0.211
VIP_Reader:192.168.0.212,192.168.0.213
```

### 5.2.3	安装过程：

1. 创建账户

```
mysql>grant replication slave,replication client on *.* to 'repl'@'%' identified by 'abc-123';
mysql>grant process,super,replication client on *.* to 'mmm_agent'@'%' identified by 'abc-123';
mysql>grant repilicaiton client on *.* to 'mmm_monitor'@'%' identified by 'abc-123';
```

说明：
repl账号用来进行复制。
`mmm_agent`代理账号，是MMM代理用来变成只读模式和同步master等操作的。
`mmm_monitor`监听账号，MMM用来对MySQL进行健康检查。

2. 先配置master1和master2的主主复制：

一：master1，master2修改my.cnf
master1：
`vim /etc/my.cnf`

```
log-bin=mysql-bin
binlog_format=row
server-id=1
auto-increment-increment=2
auto-increment-offset=1
```

master2：
`vim my.cnf`

```
log-bin=mysql-bin
binlog_format=row
server-id=2
log-slave-updates=ON
relay_log=relay.log
auto-increment-increment=2
auto-icrement-offset=2
```

二：master1备份数据然后在master2上进行恢复，确认master主主同步的数据同步点。
三：master1和master2互为主从复制即可。

3. 安装MySQL-MMM

一，五台服务器全部安装MySQL-MMM

```
wget http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
rpm -Uvh epel-release-6-8.noarch.rpm
yum install -y mysql-mmm*
```

检查安装情况：

```
rpm -qa | grep mysql-mmm
mysql-mmm-agent-版本
mysql-mmm-版本
mysql-mmm-monitor-版本
mysql-mmm-tools-版本
```

二，配置MMM监控，代理服务
修改mmm_common.conf配置文件（五台服务器的内容相同）
active_master_role      writer
```xml
<host default>
    cluster_interface       eth0
    pid_path            /var/run/mysql-mmm/mmm_agentd.pid
    bin_path               /usr/libexec/mysql-mmm/
    replication_user         repl            
    replication_password     abc-123          
    agent_user             mmm_agent      
    agent_password         abc-123     
</host>

<host db1>
    ip      192.168.0.2
    mode    master
    peer    db2
</host>

<host db2>
    ip      192.168.0.3
    mode    master
    peer    db1
</host>

<host db3>
    ip      192.168.0.4
	mode    slave
</host>

<host db4>
    ip      192.168.0.5
	mode    slave
</host>
<role writer>
    hosts   db1, db2
    ips     192.168.0.211
    mode    exclusive        #只有一个host可以writer
</role>

<role reader>
    hosts   db3,db4
    ips     192.168.0.212,192.168.0.213
    mode    balanced       #多个host可以reader
</role>
```

说明：
exclusive：在这种模式下任何时候只能有一个主机拥有该角色。
balanced：该模式下可以多个主机同时拥有该角色。
绝大多数情况下writer采用exclusive模式，reader采用balanced模式。

三，修改master1，master2，slave1，slave2的mmm_agent.conf

```
master1:this db1
master2:this db2
slave1:this db3
slave2:this db4
```

四，Monitor配置mmm_mon.conf配置文件
include mmm_common.conf

```xml
<monitor>
    ip                  127.0.0.1
    pid_path            /var/run/mysql-mmm/mmm_mond.pid
    bin_path            /usr/libexec/mysql-mmm
    status_path         /var/lib/mysql-mmm/mmm_mond.status
    ping_ips    192.168.0.2,192.168.0.3,192.168.0.4,192.168.0.5
    auto_set_online     10    
    # The kill_host_bin does not exist by default, though the monitor will
    # throw a warning about it missing.  See the section 5.10 "Kill Host 
    # Functionality" in the PDF documentation.
    #
    #kill_host_bin     /usr/libexec/mysql-mmm/monitor/kill_host
    #
</monitor>

<host default>
    monitor_user        mmm_monitor       
    monitor_password    abc-123       
</host>
```
debug 0

五，启动服务
master1,master2,slave1,slave2启动服务：

```
/etc/init.d/mysql-mmm-agent start
```

monitor服务器启动监控服务：

```
/etc/init.d/mysql-mmm-monitor start
```

在monitor服务器查看mmm状态信息：

```
mmm_control show
mmm_control checks all
```

注：此架构还可以添加中间件进行读写分离的升级，此类中间件有Amoeba,Atlas.

![MMM](https://github.com/Shannon1/documents/blob/master/image/mmm2.png)


## 5.3 MHA

### 5.3.1	概述

MHA是一位日本MySQL大牛用Perl写的一套MySQL故障切换方案，来保证数据库系统的高可用.在宕机的时间内（通常10—30秒内），完成故障切换，部署MHA，可避免主从一致性问题，节约购买新服务器的费用，不影响服务器性能，易安装，不改变现有部署。
还支持在线切换，从当前运行master切换到一个新的master上面，只需要很短的时间（0.5-2秒内），此时仅仅阻塞写操作，并不影响读操作，便于主机硬件维护。
在有高可用，数据一致性要求的系统上，MHA 提供了有用的功能，几乎无间断的满足维护需要。
存在隐患：在MHA自动故障切换的过程中，MHA试图从宕掉的主服务器上保存二进制日志，最大程度保证数据的不丢失，但这并不总是可行的。举个例子：根据谷歌运维团队的统计，谷歌的服务器平均每5分钟就有1台服务器宕机。百度运维团队统计数据，百度服务器的宕机90%都是因为存储（硬盘）出现问题造成宕机。按照这些公布出来数据如果服务器主要是硬盘问题造成宕机那么MHA的切换过程将无法获取最新的二进制日志。为了减少此项隐患，建议MySQL的主从复制全部启用半同步复制，保证故障切换之前数据尽量完整。

### 5.3.2	适用场景
目前 MHA 主要支持一主多从的架构，要搭建 MHA，要求一个复制集群必须最少有 3 台数据库服务器，一主二从，即一台充当 Master，一台充当备用 Master，另一台充当从库。出于成本考虑，淘宝在此基础上进行了改造，目前淘宝开发的 TMHA 已经支持一主一从。 
网易、搜狐畅游、1号店、唯品会、音悦台等都是用MHA。

### 5.3.3 MHA 工作原理

![MHA](https://github.com/Shannon1/documents/blob/master/image/mha.png)

1. 从宕机崩溃的 Master 保存二进制日志事件（binlog event）； 
2. 识别含有最新更新的 Slave； 
3. 应用差异的中继日志（relay log）到其他 Slave； 
4. 应用从 Master 保存的二进制日志事件； 
5. 提升一个 Slave 为新的 Master； 
6. 使其他的 Slave 连接新的 Master 进行复制

### 5.3.4	MHA组成

MHA 软件由两部分组成，Manager 工具包和 Node 工具包，具体如下。 

Manager 工具包情况如下： 
`masterha_check_ssh`:检查 MHA 的 SSH 配置情况。 
`masterha_check_repl`:检查 MySQL 复制状况。 
`masterha_manager`:启动 MHA。 
`masterha_check_status`:检测当前 MHA 运行状态。 
`masterha_master_monitor`:检测 Master 是否宕机。 
`masterha_master_switch`:控制故障转移（自动或手动）。 
`masterha_conf_host`:添加或删除配置的 server 信息。 

Node 工具包（通常由 MHA Manager 的脚本触发，无需人工操作）情况如下： 
`save_binary_logs`:保存和复制 Master 的 binlog 日志。 
`apply_diff_relay_logs`:识别差异的中级日志时间并将其应用到其他 Slave。 
`filter_mysqlbinlog`:去除不必要的 ROOLBACK 事件（已经废弃） 
`purge_relay_logs`:清除中继日志（不阻塞 SQL 线程）

### 5.3.5 配置MHA

![Alt text](https://github.com/Shannon1/documents/blob/master/image/mha2.png)



- 主机划分

| 服务器名    | IP地址        | 虚拟IP地址       |
| ------- | ----------- | ------------ |
| master1 | 192.168.0.2 | 192.168.0.11 |
| master2 | 192.168.0.3 | 192.168.0.11 |
| slave1  | 192.168.0.4 |              |
| slave2  | 192.168.0.5 |              |
| Manager | 192.168.0.6 |              |

- 软件版本

| 操作系统         | CentOS release 6.7（2.6.32-573.8.1.el6.x86_64） |
| ------------ | ---------------------------------------- |
| MySQL版本      | mysql-5.5.47.tar.gz                      |
| Keepalived版本 | keepalived-1.2.6.tar.gz                  |
| MHA Manager  | mha4mysql-manager-0.56-0.el6.noarch.rpm  |
| MHA Node     | mha4mysql-node-0.56-0.el6.noarch.rpm     |

- 安装

1. 所有的mysql集群服务器更换yum源：

```bash
rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
```

2. 所有的mysql集群必须安装node的包：

```bash
yum install -y perl-DBD-MySQL perl-DBI
wget http://123.232.37.94/mha4mysql-node-0.56-0.el6.noarch.rpm
rpm -ivh mha4mysql-node-0.56-0.el6.noarch.rpm
```

3. manager服务器安装：

```bash
yum install -y perl-DBI perl-Config-Tiny.noarch \
perl-Log-Dispatch.noarchperl-Parallel-ForkManager.noarch  \
perl-DBD-MySQLperl-DBI perl perl-devel libaio libaio-devel \
perl-Time-HiResperl-DBD-MySQL cpan
wget http://123.232.37.94/mha4mysql-node-0.56-0.el6.noarch.rpm
wget http://123.232.37.94/mha4mysql-manager-0.56-0.el6.noarch.rpm
rpm -ivh mha4mysql-node-0.56-0.el6.noarch.rpm
rpm -ivh mha4mysql-manager-0.56-0.el6.noarch.rpm
```

4. 配置ssh免密码登录

```bash
ssh-keygen-t rsa
ssh-copy-id-i /root/.ssh/id_rsa.pub "root@192.168.0.2"
```

所有的mysql节点免登陆互通。

5. mysql集群配置主从复制+半同步复制(请转到4.4章节配置主从复制)。

6. 创建mha授权监控账户

```sql
grant all on *.* to 'root'@'192.168.0.%'identified by 'abc-123';
```

7. 创建`mha-manager`端的配置文件以及ssh和`master-slave`的检查

```bash
mkdir -p /etc/masterha/
mkdir -p /var/log/masterha/app1/
vim /etc/masterha/app1.cnf
```

```ini
[server default]
remote_workdir=/var/log/masterha/app1
manager_workdir=/var/log/masterha/app1
manager_log=/var/log/masterha/app1/manager.log
user=admin
password=abc-123
ssh_user=root
repl_user=root
repl_password=abc-123
ping_interval=1
shutdown_script=""
master_ip_online_change_script=""
report_script=""

[server1]
hostname=192.168.0.2
port=3306
candidate_master=1
master_binlog_dir="/application/mysql"
check_repl_delay=0

[server2]
hostname=192.168.0.3
port=3306
candidate_master=1
master_binlog_dir="/application/mysql"
check_repl_delay=0 

[server3]
hostname=192.168.0.4
port=3306
no_master=1 

[server4]
hostname=192.168.0.5                                         
port=3306
no_master=1
```

创建软连接，在MHA工作过程中会使用mysql和mysqlbinlog

```
ln -s /application/mysql/bin/mysql   /usr/bin/mysql
ln -s /application/mysql/bin/mysqlbinlog /usr/bin/mysqlbinlog
```

测试ssh：

```
masterha_check_ssh--conf=/etc/masterha/app1.cnf
```

测试复制：

```
masterha_check_repl--conf=/etc/masterha/app1.cnf
```

8. 启动manager服务器

```
nohupmasterha_manager --conf=/etc/masterha/app1.cnf > /tmp/mha_manager.log2>&1 &
```

注：以上是配置MHA的过程，在实际生产过程中需要编写脚本，保证MHA在生产过程中稳定运行。

9. 配置keepalived

环境安装：

```bash
yum install -y  kernel-devel,openssl-devel,popt*
tar xf keepalived-1.2.6.tar.gz
cd keepalived-1.2.6
./configuer
make
make install
```

配置规范启动：

```bash
cp /usr/local/etc/rc.d/init.d/keepalived/etc/init.d/
cp /usr/local/etc/sysconfig/keepalived/etc/sysconfig/
mkdir /etc/keepalived
cp /usr/local/etc/keepalived/keepalived.conf/etc/keepalived/
cp /usr/local/sbin/keepalived /usr/sbin/
service keepalived start
service keepalived stop
ps -ef|grep keep
```

配置文件：

Master1：
`vim /etc/keepalived/keepalived.conf`
Configuration File for keepalived

```js
global_defs {
   notification_email {
   630311268@qq.com
   }
   notification_email_from keepalived@keepalived.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id MySQL-MHA
}
vrrp_script check_run {
script "/etc/keepalived/check_mysql.sh"
interval 1
}
vrrp_instance VI_1 {
    state backup
    interface eth0
    virtual_router_id 55
    priority 150
    advert_int 1
	 nopreempt
    authentication {
        auth_type PASS
        auth_pass 1111
    }
	track_script {
        check_running
       }
    virtual_ipaddress {
        192.168.0.11/24
    }
}
```

Master2:
`vim /etc/keepalived/keepalived.conf`
! Configuration File for keepalived

```js
global_defs {
   notification_email {
   630311268@qq.com
   }
   notification_email_from keepalived@keepalived.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id MySQL-MHA
}
vrrp_script check_run {
script "/etc/keepalived/check_mysql.sh"
interval 1
}
vrrp_instance VI_1 {
    state backup
    interface eth0
    virtual_router_id 55
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
	 track_script {
		check_running
       }
    virtual_ipaddress {
        192.168.0.11/24
    }
}
```

`vim /etc/keepalived/check_mysql.sh`
```bash
#!/bin/bash
MYSQL=/application/mysql/bin/mysql
MYSQL_HOST=localhost
MYSQL_USER=root
MYSQL_PASSWORD=abc-123
CHECK_COUNTS=3
MYSQL_OK=0
function check_Mysql_Runing ()
{
$MYSQL -h $MYSQL_HOST -u $MYSQL_USER -p${MYSQL_PASSWORD}-e "show grants;" >/dev/null 2>&1
if [ $? = 0 ] ;then
        MYSQL_OK=0  
else
        MYSQL_OK=1 
fi
return $MYSQL_OK 
} 
while [ $CHECK_COUNTS -ne 0 ]
do
        let"CHECK_COUNTS -= 1" 
        check_Mysql_Runing  
        if [ $MYSQL_OK= 0 ] ; then
             echo "mysql is runing!" 
             exit 0
        fi 
        if [ $MYSQL_OK -eq 1 ] && [ $CHECK_COUNTS -eq 0 ]
        then
               echo "mysql is not runing!"
               exit 1
        fi
        sleep 1
done
```
`chmod +x keepalived_check_mysql.sh`


## 5.4 Heartbeat+DRBD+MySQL

### 5.4.1所需软件及其介绍：

**Heartbeat介绍**
[官方站点](http://linux-ha.org/wiki/Main_Page)

heartbeat可以资源(VIP地址及程序服务)从一台有故障的服务器快速的转移到另一台正常的服务器提供服务，heartbeat和keepalived相似，heartbeat可以实现failover功能，但不能实现对后端的健康检查。

**heartbeat和keepalived应用场景及区别**
很多网友说为什么不使用keepalived而使用长期不更新的heartbeat，下面说一下它们之间的应用场景及区别
1. 对于web，db，负载均衡(lvs,haproxy,nginx)等，heartbeat和keepalived都可以实现
2. lvs最好和keepalived结合，因为keepalived最初就是为lvs产生的，(heartbeat没有对RS的健康检查功能，heartbeat可以通过ldircetord来进行健康检查的功能)
3. mysql双主多从，NFS/MFS存储，他们的特点是需要数据同步，这样的业务最好使用heartbeat，因为heartbeat有自带的drbd脚本
总结：无数据同步的应用程序高可用可选择keepalived，有数据同步的应用程序高可用可选择heartbeat

**DRBD介绍**
[官方站点](http://www.drbd.org/)

DRBD(Distributed Replicated Block Device)是一个基于块设备级别在远程服务器直接同步和镜像数据的软件，用软件实现的、无共享的、服务器之间镜像块设备内容的存储复制解决方案。它可以实现在网络中两台服务器之间基于块设备级别的实时镜像或同步复制(两台服务器都写入成功)/异步复制(本地服务器写入成功)，相当于网络的RAID1，由于是基于块设备(磁盘，LVM逻辑卷)，在文件系统的底层，所以数据复制要比cp命令更快
DRBD已经被MySQL官方写入文档手册作为推荐的高可用的方案之一

### 5.4.2	架构图

![Alt text](https://github.com/Shannon1/documents/blob/master/image/hk.png)


服务器划分：

| **服务器名** | **IP地址**                 | **虚拟IP地址**   |
| -------- | ------------------------ | ------------ |
| Master1  | 192.168.0.2（WAN数据转发）eth1 | 192.168.0.11 |
|          | 172.16.1.1（心跳检测）eth2     |              |
|          | 172.16.2.1（LAN数据同步）eth3  |              |
| Master2  | 192.168.0.3（WAN数据转发）eth1 | 192.168.0.11 |
|          | 172.16.1.2（心跳检测）eth2     |              |
|          | 172.16.2.2（LAN数据同步）eth3  |              |
| Slave1   | 192.168.0.4              |              |
| Slave2   | 192.168.0.5              |              |
| Slave3   | 192.168.0.6              |              |
| LVS1     | 192.168.0.7              | 192.168.0.12 |
| LVS2     | 192.168.0.8              | 192.168.0.12 |

master服务器分区信息

```
/dev/sdb	1-2G	/dev/sdb1	/data/	存放数据
/dev/sdb2		存放drbd同步的状态信息
```

注意

1.  metadata分区一定不能格式化建立文件系统（sdb2存放drbd同步的状态信息）
2.  分好的分区不要进行挂载
3.  生产环境DRBDmetadata分区一般可设置为1-2G，数据分析看需求
4.  再生产环境中两块硬盘一样大

### 5.4.3   安装

一 heartbeat安装

1. 配置master1和master2之间路由

```bash
master1：
route add -host 172.16.1.2 dev eth2
route add -host 172.168.2.2 dev eth3
master2：
route add -host 172.16.1.1 dev eth2
route add -host 172.168.2.1 dev eth3
```

2. 安装heartbeat

```bash
yum install heartbeat -y
yum install heartbeat -y
```

**此处需要运行2遍安装命令**

3. 配置heartbeat

主备节点两端的配置文件(ha.cfauthkeysharesources)完全相同

配置ha.cf

`vim /etc/ha.d/ha.cf`

```bash
debugfile /var/log/ha-debug
logfile /var/log/ha-log
logfacility local1
#options configure
keepalive 2
deadtime 30
warntime 10
initdead 120
#bcast  eth2
mcast eth2 225.0.0.7 694 1 0
#node configure
auto_failback on
node   master1     <==主节点主机名
node   master2     <==备节点主机名
crm no
```


配置authkeys

`vim /etc/ha.d/authkeys`

```bash
auth 1
1 sha147e9336850f1db6fa58bc470bc9b7810eb397f04
```

配置haresources

`vim /etc/ha.d/haresources`

```bash
master1 IPaddr::192.168.0.11/24/eth1
#master1 IPaddr::192.168.0.11/24/eth1drbddisk::data Filesystem::/dev/drbd1::/data::ext4 mysqld
```

启动heartbeat

```
/etc/init.d/heartbeat start
chkconfig heartbeat off
```

测试heartbeat

正常状态：

```
master1：ip addr|grep eth1
master2：ip addr|grep eth1
```

模拟主节点宕机后的状态：

```
master1：/etc/init.d/heartbeat stop
master2：ip addr|grep eth1
```

模拟主节点故障恢复后的状态：

```
master1：/etc/init.d/heartbeatstart
master1：ipaddr|grep eth1
```

二 DRBD安装部署

1. 新添加硬盘

```bash
fdisk /dev/sdb
partprobe
mkfs.ext4 /dev/sdb1
tune2fs -c -1 /dev/sdb1
```

2. 安装DRBD

```bash
yum install kmod-drbd83 drbd83 -y
modprobe drbd
```

3. 配置DRBD

主备节点两端配置文件完全一致

`vim /etc/drbd.conf`

```bash
global {
# minor-count 64;
# dialog-refresh 5; # 5 seconds
# disable-ip-verification;
usage-count no;
}
common {
protocol C;
disk {
on-io-error   detach;
#size 454G;
no-disk-flushes;
no-md-flushes;
}
net {
sndbuf-size 512k;
# timeout       60;    #  6 seconds  (unit = 0.1 seconds)
# connect-int   10;    # 10 seconds  (unit = 1 second)
# ping-int      10;    # 10 seconds  (unit = 1 second)
# ping-timeout   5;    # 500 ms (unit = 0.1 seconds)
max-buffers     8000;
unplug-watermark   1024;
max-epoch-size  8000;
# ko-count 4;
# allow-two-primaries;
cram-hmac-alg "sha1";
shared-secret "hdhwXes23sYEhart8t";
after-sb-0pri disconnect;
after-sb-1pri disconnect;
after-sb-2pri disconnect;
rr-conflict disconnect;
# data-integrity-alg "md5";
# no-tcp-cork;
}
syncer {
rate 120M;
al-extents 517;
}
}
resource data {
on master1 {
device     /dev/drbd1;
disk       /dev/sdb1;
address    192.168.0.2:7788;
meta-disk  /dev/sdb2 [0];
}
on master2 {
device     /dev/drbd1;
disk       /dev/sdb1;
address    192.168.0.3:7788;
meta-disk  /dev/sdb2 [0];
}
}
```

4. 初始化meta分区

```
drbdadm create-md data
```

5. 初始化设备同步(覆盖备节点，保持数据一致)

```
drbdadm -- --overwrite-data-of-peerprimary data
```

6. 启动drbd

```
drbdadm up all
chkconfig drbd off
```

7. 挂载drbd分区到data数据目录

```
drbdadm primary all
mount /dev/drbd1 /data
```

8. 测试DRBD

正常状态：

```
master1： cat /proc/drbd
master2： cat /proc/drbd
```

模拟master1宕机：

```
master1：umount /dev/drbd1
master1：drbdadm down all
master2：cat /proc/drbd
master2：drbdadmprimary all
master2：mount/dev/drbd1 /data
master2：df-h
```

注意：master1宕机后，master2可以升级为主节点，可挂载drbd分区继续使用


三 MySQL安装部署

注意：五台数据库都安装mysql服务，master2只安装到makeinstall即可，mysqld服务不要设置为开机自启动。

安装过程参考第一章。

故障切换测试

架构正常状态：

```
master1:ip addr|grep eth1
master1:cat /proc/drbd
master1:mysql -uroot -e"create database jovision;"
```

