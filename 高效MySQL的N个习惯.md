

[toc]

# 用更好的硬件

## 用更好的CPU

- 主频高，让每个SQL处理时间更快，减少等待
- （L1/L2/L3）cache 大， 每次CPU计算速率更快
- 线程多，同时支持更多并发SQL， 提高TPS
- 同时记得关闭NUMA斌该设置为最大性能模式
- MySQL 5.6.27后，增加innodb_numa_control选项

 
## 用更好的内存

- 主频高，内存读写速率跟高，更高吞吐，更低延迟
- 内存大，更多数据在内存中个，减少直接磁盘读写，提高TPS

## 用更好的磁盘

- 通常来说，磁盘I/O是最大瓶颈
- 如果是机械磁盘，一定要配阵列卡，以及阵列卡的CACHE&BBU，并且使用（FORCE）WB策略
- 最好是选用SSD或者PCIe SSD，IOPS可以提升成千上万倍

## 用更好的网络/网卡

- 文件传输速率高，异地文件备份更快
- 主从数据复制数据传输延时更小
- 适合大数据的分布式存储环境
- 老版本内核中，网络请求太高时会引发中断瓶颈，建议升级内核
- 多个网卡可以进行绑定，提高传输速率并能提高可用性


# 让OS跑的更快

## 关闭无用的服务

- 减少系统开销
- 避免潜在的安全隐患

## 尽可能使用本地高速存储

- 坚决不适用NFS
- 除非是基于SSD的高速网络分布式存储
- 用于备份场景除外

## 让数据库跑在专用的服务器上，不混搭

- 性能上不相互影响
- 提高安全性
- 必须混搭时要做好权限管理及安全隔离


## IO Scheduler

选择deadline, noop，坚决不用cfq

## 文件系统选择

- 优先选用xfs或ext4（RHEL 7及以上， xfs已是默认文件系统）
- zfs/btrfs比较小众
- 坚决不用ext3

## 其他内核选项

- `vm.swappiness` <= 10
- 降低使用 swap 的概率
- 内核 2.6.32-303 及其以上版本，慎重设置为0，可能引发OOM
- `vm.dirty_ratio` <= 5
- `vm.dirty_backgroud_ratio` <= 10
- 避免因为IO压力瞬间飙升导致内核进程卡死，OS hung住

# DDL, SQL写得好

## 一定要有主键PRIMARY KEY

- 如果没有主键，数据多次读写后可能更离散，有更多的随机IO
- 如果没有主键，MySQL复制环境中，如果选择RBR（Row - Based Replication）模式，没有主键的update需要读全表，导致复制延迟
- 好的主键，没有业务用途
- 好的主键，数值呈连续增长，最好自增
- 好的主键，坚决不能选用CHAR/UUID等类型

## 关于数据长度
- 够用前提下，越短越好
- 消耗更少的存储空间
- 需要进行排序时，消耗更少的内存空间

例如 用 INT UNSIGNED 存储 IPv4地址，不用 CHAR(15) 类型

## 适当使用 TEXT/BLOB 类型

- data page 默认16KB
- 每行长度超过8KB时，就需要分裂 data page 
- 产生更多离散I/O

一个 100G 的表拆分成 4 个表后，总大小仅 25G 

## 每个表增加 `create_time` `update_time` 两个字段

- 分别表示写入时间及最后更新时间
- 业务上可能用不到，但是对日常运维管理则非常有用
- 可以用来判断哪些是可以归档的老数据，定期进行归档
- 用来做自定义的差异备份也很方便

## 索引

- 索引很重要， Innodb 的行锁是基于索引实现的，如果没有索引将会是灾难性的
    - 读取时，全表扫描
    - 修改时，全表记录锁
- 基数（cardinality）低的字段一般没必要建立**单列**索引
- 字符型字段上建立索引时优先采用部分索引（prefix index）
- 5.6.9之后，optimizer 能识别到普通索引同事存储主键值，无需显示定义加上主键列（index extensions）
- 优先多列联合索引，（multi column index），少用单列索引

## 什么是好的SQL

- 所有 `WHERE` 条件都加上引号
    - 避免潜在的类型转换风险
    - 避免个别条件失效时SQL语法错误
- 不 `SELECT *` 
    - 减少不必要的I/O
    - 提高可以利用覆盖索引的几率
- 避免SQL注入风险
    - 所有用户输入值都要做过滤
    - 利用 `PREPARE` 做预处理
    - 利用 SQL_MODE 做限制
- `LIKE` 查询时，不要用%通配符最左前导（无法使用索引）
- 能 `UNION ALL` 就不要 `UNION` （`UNION` 需要去重，会产生临时表）
- SQL中最好不要有运算
- `WHERE` 子句中，不要有函数
- 关于 `JION`
    - 满足业务需求前提下，优先使用 `INNER JOIN`，让优化器自动选择驱动表
    - 有时候优化器选择的驱动表未必是最优的，可以尝试手动调整
    - 最后的排序字段如果不在驱动表中，则会有 filesort

糟糕的 SQL :

```sql
UPDATE t SET c2 = ? AND c3 = ? WHERE c1 = '?'
=>
UPDATE t SET c2 = ?, c3 = ? WHERE c1 = '?'

UPDATE t SET c2 = ?
   WHERE c1 = '?'
=>
UPDATE t SET c2 = ? WHERE c1 = '?'

SELECT * FROM t WHERE c1 = '?'
=>
SELECT c2,c3 FROM t WHERE c1 = '?'

SELECT c2,c3 FROM t WHERE c4 like '%???%'
=>
SELECT c2,c3 FROM t WHERE c1 >= ? AND c1 <= ? AND c4 like '%???%'

SELECT c2,c3 FROM t WHERE date(c1) = '2016-10-15'
=>
SELECT c2,c3 FROM t WHERE c1 >= '2016-10-15' AND c1 < '2016-10-16'

SELECT c2,c3 FROM t WHERE char_col = int_value
=>
SELECT c2,c3 FROM t WHERE char_col = 'int_value'
```

- 关于 `EXPLAIN`
    - 关键业务SQL上线前，都要 `EXPLAIN` 确认其执行计划
    - 或提前分析 `slow query log` ，防患未然
    - `EXPLAIN` 中如果有 `using temporary` `using filesort` 或 `type=ALL` 时，尽量想办法进行优化

# 运维习惯好

## 存储引擎

- 抛弃 MyISAM， Innodb 为王
- 适当场景下可以采用 TokuDB
- 误区：MEMORY不见得就快

## 关闭 QUREY CACHE

- 绝大多数情况下是鸡肋，最好欧关闭
- QC 锁是全局锁，每次更新 QC 的内存块锁代价高，出现 `waiting for query cache lock` 状态的频率很高
- 实例启动前设置 `query_cache_type = 0` & `qurey_cache_size = 0`

## 使用独立 UNDO 表空间

- 避免 ibdata1 文件存储空间暴涨
- MySQL5.6 开始支持独立表空间
- MySQL5.7 还可以回收已经 purge 的表空间
- 提高 file I/O 能力，并适当增加 purge 线程数 innodb_purge_threads
- 事务及时提交，不要积压
- 默认打开 `autocommit = 1`

## 启用  thread pool

- 应对突发短连接
- extra port
- 没有 thread pool 想办法启用连接池或其他替代方案
- 适当调低超时阈值， 减少空闲连接

## 几个关键选项

- `innodb_buffer_pool_size` 约为物理内存的 50%~70%
- `innodb_log_file_size` 5.5 及其以上版本 2G+， 5.5 以下版本建议不超过 512M
- `innodb_flush_log_at_trx_commit` 0:最快数据最不安全，1: 最慢最安全， 2:折中
- `innodb_max_dirty_pages_pct` 25% ~ 50% 为宜
- `max_connections` 突发最大连接数的 80% 为宜，过大容易导致全部卡死

# 其他好习惯

## 启用辅助监控机制

- 干掉超过 N 秒的 SQL
- 干掉疑似注入 SQL
- 干掉长时间不活跃的 sleep 连接

## online DDL

- 优先 pt-osc，但不见得一定要用 pt-osc
- 5.6 以后对 online DDL 有了很大提升改善

## 删除大表

- 不要真的删除，而是先 rename
- 确认对业务真的没有影响
- 再用硬连接的方法物理删除，效率更高

## autocommit

- 避免某些行锁被长时间持有， 影响tps
- 更严重时可以连接数暴涨，导致整个实例挂掉
- 采用 GUI 客户端连接时，记得及时关闭连接，或设置超时阈值以及自动提交，否则容易发生行锁等待问题





[原文](https://pan.baidu.com/s/1dFdtC0T)
