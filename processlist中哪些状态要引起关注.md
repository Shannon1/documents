# processlist中哪些状态要引起关注


一般而言，我们在processlist结果中如果经常能看到某些SQL的话，至少可以说明这些SQL的频率很高，通常需要对这些SQL进行进一步优化。


<table border="1" cellspacing="1" cellpadding="1">
<tbody>
<tr>
<td><strong>状态</strong></td>
<td><strong>建议</strong></td>
</tr>
<tr>
<td><strong>copy to tmp table</strong></td>
<td>执行ALTER TABLE修改表结构时<strong>建议：</strong>放在凌晨执行或者采用类似pt-osc工具</td>
</tr>
<tr>
<td><strong>Copying to tmp table</strong></td>
<td>拷贝数据到内存中的临时表，常见于GROUP BY操作时<strong>建议：</strong>创建适当的索引</td>
</tr>
<tr>
<td><strong>Copying to tmp table on disk</strong></td>
<td>临时结果集太大，内存中放不下，需要将内存中的临时表拷贝到磁盘上，形成 #sql***.MYD、#sql***.MYI（在5.6及更高的版本，临时表可以改成InnoDB引擎了，可以参考选项<strong>default_tmp_storage_engine</strong>）<strong>建议：</strong>创建适当的索引，并且适当加大<strong>sort_buffer_size/tmp_table_size/max_heap_table_size</strong></td>
</tr>
<tr>
<td><strong>Creating sort index</strong></td>
<td>当前的SELECT中需要用到临时表在进行ORDER BY排序<strong>建议：</strong>创建适当的索引</td>
</tr>
<tr>
<td><strong>Creating tmp table</strong></td>
<td>创建基于内存或磁盘的临时表，当从内存转成磁盘的临时表时，状态会变成：Copying to tmp table on disk<strong>建议：</strong>创建适当的索引，或者少用UNION、视图(VIEW)、子查询(SUBQUERY)之类的，确实需要用到临时表的时候，可以在session级临时适当调大 <strong>tmp_table_size/max_heap_table_size</strong> 的值</td>
</tr>
<tr>
<td><strong>Reading from net</strong></td>
<td>表示server端正通过网络读取客户端发送过来的请求<strong>建议：</strong>减小客户端发送数据包大小，提高网络带宽/质量</td>
</tr>
<tr>
<td><strong>Sending data</strong></td>
<td>从server端发送数据到客户端，也有可能是接收存储引擎层返回的数据，再发送给客户端，数据量很大时尤其经常能看见<strong>备注：</strong>Sending Data不是网络发送，是从硬盘读取，发送到网络是Writing to net</p>
<p><strong>建议：</strong>通过索引或加上LIMIT，减少需要扫描并且发送给客户端的数据量</td>
</tr>
<tr>
<td><strong>Sorting result</strong></td>
<td>正在对结果进行排序，类似Creating sort index，不过是正常表，而不是在内存表中进行排序<strong>建议：</strong>创建适当的索引</td>
</tr>
<tr>
<td><strong>statistics</strong></td>
<td>进行数据统计以便解析执行计划，如果状态比较经常出现，有可能是磁盘IO性能很差<strong>建议：</strong>查看当前io性能状态，例如iowait</td>
</tr>
<tr>
<td><strong>Waiting for global read lock</strong></td>
<td>FLUSH TABLES WITH READ LOCK整等待全局读锁<strong>建议：</strong>不要对线上业务数据库加上全局读锁，通常是备份引起，可以放在业务低谷期间执行或者放在slave服务器上执行备份</td>
</tr>
<tr>
<td><strong>Waiting for tables,</strong><strong>Waiting for table flush</strong></td>
<td>FLUSH TABLES, ALTER TABLE, RENAME TABLE, REPAIR TABLE, ANALYZE TABLE, OPTIMIZE TABLE等需要刷新表结构并重新打开<strong>建议：</strong>不要对线上业务数据库执行这些操作，可以放在业务低谷期间执行</td>
</tr>
<tr>
<td><strong>Waiting for lock_type lock</strong></td>
<td>等待各种类型的锁：• Waiting for event metadata lock• Waiting for global read lock• Waiting for schema metadata lock• Waiting for stored function metadata lock• Waiting for stored procedure metadata lock• Waiting for table level lock• Waiting for table metadata lock• Waiting for trigger metadata lock<strong>建议：</strong>比较常见的是上面提到的global read lock以及table metadata lock，建议不要对线上业务数据库执行这些操作，可以放在业务低谷期间执行。如果是table level lock，通常是因为还在使用MyISAM引擎表，赶紧转投InnoDB引擎吧，别再老顽固了</td>
</tr>
</tbody>
</table>


[原文](http://imysql.com/2015/06/10/mysql-faq-processlist-thread-states.shtml)
