# otter(MySQL中美机房双写同步)部署手册




## 名词解释

`MySQL复制(Replication)`：MySQL官方提供的在两个（及以上）数据库拷贝数据的方式。主库(Master)在写入数据时同时写入二进制日志(binlog)，从库(Slave)通过读取主库binlog的信息，导入自己的中继日志(Relay log)，然后回放(Replay)日志中的事件写入自己（从库）的数据库中。也叫**MySQL同步**

binlog写入方式有三种：
1. STATEMENT 基于语句的。可以理解为binlog中记录MySQL每次执行的sql语句，从库在复制时重放这些sql语句。如果sql语句中包含有元数据或函数(如时间戳)，则可能无法被正确复制。
2. ROW 基于行的。可以理解为binlog中记录数据的变动，优点是可以准确记录每一行的变动，缺点是一个sql语句可以影响很多行，复制代价很大。
3. MIXED 混合的。MySQL默认方式，一般情况下是STATEMENT复制，如果发现无法被正确复制时则采用ROW方式。

`双主复制`：也叫**主-主复制**，复制双方都配置为对方的主库和备库，两边的数据都可以同步到对方的库中。分为**主动-主动**模式下的主-主复制和**主动-被动**模式下的主-主复制。区别是主动-主动模式下的复制，两个库都可以写入数据；主动-被动模式下的复制，只有一台服务器可以写入数据，另一台作为只读的备库存在。如果不加说明，下面提到的都是主动-主动模式下的复制，也叫双写复制、双主复制、双A复制。

## otter
otter是阿里巴巴用Java开发的基于数据库增量日志解析，准实时同步到本机房或异地机房的mysql/oracle数据库. 一个分布式数据库同步系统。

### otter能解决什么？
1. 异构库同步
mysql -> mysql/oracle. (目前开源版本只支持mysql增量，目标库可以是mysql或者oracle，取决于canal的功能)

2. 单机房同步 (数据库之间RTT < 1ms)
数据库版本升级
数据表迁移
异步二级索引

3. 异地机房同步 (比如阿里巴巴国际站就是杭州和美国机房的数据库同步，RTT > 200ms，亮点)

4. 双向同步
    a. 避免回环算法 (通用的解决方案，支持大部分关系型数据库)
    b. 数据一致性算法 (保证双A机房模式下，数据保证最终一致性，亮点)

5. 文件同步
站点镜像 (进行数据复制的同时，复制关联的图片，比如复制产品数据，同时复制产品图片)

其中，异地机房的双向同步正式我们需要的。而且otter的同步并不依赖于mysql之间的复制关系，只是读取binlog中的事件，在mysql的上层进行同步操作。

###　相关概念

- Pipeline：从源端到目标端的整个过程描述，主要由一些同步映射过程组成
- Channel：同步通道，单向同步中一个Pipeline组成，在双向同步中有两个Pipeline组成
- DataMediaPair：根据业务表定义映射关系，比如源表和目标表，字段映射，字段组等
- DataMedia : 抽象的数据介质概念，可以理解为数据表/mq队列定义
- DataMediaSource : 抽象的数据介质源信息，补充描述DateMedia
- ColumnPair : 定义字段映射关系
- ColumnGroup : 定义字段映射组
- Node : 处理同步过程的工作节点，对应一个jvm
- Manager: 用于管理各个Node节点的管理端
- canal: 阿里巴巴的另一个开源项目，基于数据库增量日志解析，提供增量数据订阅&消费，是otter的直接数据来源。它伪装成一个slave从主库的binlog里获取数据，然后提供给otter

## 搭建过程

### 组件准备

otter是一个复杂的项目，需要依赖的组件很多。

搭建MySQL的过程不在这里赘述了，这里只列举mysql可以正常提供服务，在mysql之上搭建otter的过程。

`java` 1.6.25以上版本
`zookeeper` otter node依赖于zookeeper进行分布式调度，需要安装一个zookeeper节点或者集群。zookeeper集群需要至少三个节点，如果在两台机器上同步，可以在一个节点上起两个zookeeper实例，也可以在另一个无关的机器上建一个zookeeper节点，这个节点上只会保存一些用于分布式的数据。（分布式的东西不太懂）

`node` otter数据库节点，受manager管理，每个需要同步的数据库上都需要安装
`manager` 管理端，管理协调各个node节点，并提供web管理端，只在一个机器上安装即可

`aria2` node节点进行跨机房传输时，会使用到HTTP多线程传输技术，目前主要依赖了aria2c做为其下载客户端


### 安装步骤

1.修改mysql配置参数

otter的数据来源本质上还是binlog，所以MySQL必须开启binlog

```
log-bin=mysql-bin # mysql-bin可以改成其他自定义名字
```
为了确保数据一致性，binlog格式必须是ROW

```
binlog-format=ROW
```

otter集群中每台数据库集群中的MySQL的**server-id必须不同**

```
server-id=1
```

为了保证双主同时写入时可能会发生的数据冲突，必须保证两边的记录绝对不会有完全相同的。MySQL提供了`auto-increment-increment` 和 `auto-increment-offset` 两个字段来保证。其中`auto-increment-increment`意思是每次自增值的增量；`auto-increment-offset`意思是初始偏移量，即从几开始。

```
auto-increment-increment=8
auto-increment-offset=1
```
这样设置之后，MySQL自增键就会是1,9,17...；而集群另一台需要设置成`auto-increment-increment=8``auto-increment-offset=2`，这样这台机器上的自增键就会是2,10,18...这样就保证不会有两条相同的记录了。

以上是必须的配置，下面有些可选的配置

```
sync-binlog=0
```

`sync_binlog` 是MySQL的binlog同步到磁盘的频率。可选值0或n(n>0)。默认值是 0，不主动同步，而依赖操作系统本身不定期把文件内容flush到磁盘。设为**1最安全**，在每个语句或事务后同步一次binlog，即使在崩溃时也最多丢失一个语句或事务的日志，但因此也最慢。
大多数情况下，对数据的一致性并没有很严格的要求，所以并不会把 `sync_binlog` 配置成1。 为了追求高并发，提升性能，可以设置为100或直接用0。 `sync_binlog`设置为0或1，可以产生五倍到二十倍甚至更多的写入性能差距。如果要做双主双向同步，两边的这个参数配置应该一致，推荐默认值或者是10以上的值，否则性能真的很差。

为提高兼容性最好默认就使用UTF8

```
character-set-server=utf8
```


2.安装java

```
rpm -ivh jdk-8u74-linux-x64.rpm

#vim /etc/profile

在profile文件最后追加入如下内容：
export JAVA_HOME=/usr/java/jdk1.8.0_74
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin

#source /etc/profile

```
3.安装aria2

```
tar zxvf aria2-1.17.1.tar.gz
cd aria2-1.17.1
./configure
make && make install
```

4.安装zookeeper集群

`server-id=1`

```
tar zxvf zookeeper-3.4.8.tar.gz -C /usr/local
mv /usr/local/zookeeper-3.4.8/ /usr/local/zookeeper/
cd /usr/local/zookeeper/
mkdir data
echo "1" > data/myid
mv conf/zoo_sample.cfg conf/zoo.cfg
vim conf/zoo.cfg
dataDir=/usr/local/data
server.1 =server1-ip:2888:3888
server.2 =server2-ip:2888:3888
server.3 =server3-ip:2888:3888
:x #保存退出
```
**注意**：`myid`的值需要和`zoo.cfg`中`server.x=ip:port1:port2`中的x一致

`server-id=2` 除了下面几项，其他都一样
```
...
echo "2" > data/myid
...
```

`server-id=3` 除了下面几项，其他都一样
```
...
echo "3" > data/myid
...
```

启动每个几点的zookeeper

```
/usr/local/zookeeper/bin/zkServer.sh start
```
然后查看每个节点的zookeeper的运行状态

```
/usr/local/zookeeper/bin/zkServer.sh status
```

正常情况下有一个节点mode是leader，其他节点是follower。
如果有异常，检查zoo.cfg中的ip，可以改成内网ip或0.0.0.0或主机名并修改hosts文件。或参考错误日志`/usr/local/zookeeper/zookeeper.out`中的内容来排查。

5.向mysql中导入需要的表
需要向mysql中导入两个sql文件
`otter-manager-schema.sql`
会创建数据库otter和若干表，用于维护和管理otter的数据，如数据源、pipeline、和一些统计信息等。
`otter-manager-retl.sql`
会创建retl用户，密码`retl`，并赋予`REPLICATION SLAVE, REPLICATION CLIENT, SELECT, INSERT, UPDATE, DELETE, EXECUTE`的权限，这个用户会读取mysql的真是数据和binlog里面的事件。该文件还会创建retl库和一些表用于管理。
retl用户非常重要，他是同步数据的来源，并且有非常高的权限。
如果重新安装otter删除了retl用户，一定要刷新权限(`flush privileges;`)，否则在导入`otter-manager-retl.sql`的时候会出现create user失败的错误，并且在同步的时候无法获取到全部的数据导致同步失败。（表现到日志里面可能会是`No database selected`）

6.安装manager

manager是otter的管理端，提供web管理界面，多个节点的集群在一个节点上安装就可以

```
mkdir /usr/local/mananger
tar zxvf manager.deployer-4.2.12.tar.gz -C /usr/local/manager
vim /usr/local/manager/conf/otter.properties

## otter manager domain name
otter.domainName = 127.0.0.1 # 在浏览器地址栏输入的访问地址
## otter manager http port
otter.port = 8080 # 在浏览器输入的访问地址端口，正常来说输入127.0.0.1:8080就可以打开otter的管理界面，如果访问不到的话，查一下域名ip或修改hosts
## jetty web config xml
otter.jetty = jetty.xml

## otter manager database config # otter需要连接到mysql中读取刚刚导入的otter库中的内容，如果网页打不开，检查一下这几项是否配置正确，确保otter可以访问到mysql中的数据
otter.database.driver.class.name = com.mysql.jdbc.Driver
otter.database.driver.url = jdbc:mysql://127.0.0.1:3306/otter
otter.database.driver.username = root
otter.database.driver.password = hello

## otter communication port # 其他node节点需要通过这一项配置的端口来访问manager，如果修改了这个端口，在配置其他节点的node配置文件的时候要保持一致
otter.communication.manager.port = 1099

## otter communication pool size
otter.communication.pool.size = 10

## default zookeeper address # 需要和zookeeper端口配置一致
otter.zookeeper.cluster.default = 127.0.0.1:2181
## default zookeeper sesstion timeout = 60s
otter.zookeeper.sessionTimeout = 60000

## otter arbitrate connect manager config
otter.manager.address = ${otter.domainName}:${otter.communication.manager.port}

## should run in product mode , true/false
otter.manager.productionMode = true

## self-monitor enable or disable
otter.manager.monitor.self.enable = true
## self-montir interval , default 120s
otter.manager.monitor.self.interval = 120
## auto-recovery paused enable or disable
otter.manager.monitor.recovery.paused = true
# manager email user config
otter.manager.monitor.email.host = smtp.gmail.com
otter.manager.monitor.email.username = 
otter.manager.monitor.email.password = 
otter.manager.monitor.email.stmp.port = 465

:x # 保存退出

/usr/local/manager/bin/startup.sh 启动manager

```

7.安装node

接下来通过manager提供的web界面来操作




