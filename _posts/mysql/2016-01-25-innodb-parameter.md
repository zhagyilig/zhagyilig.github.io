---
layout: article
title: "InnoDB global status"
modified:
categories: mysql
image:
    teaser: /teaser/innodb.jpg
---
> 常用的一些参数
---
#### Innodb_buffer_pool_pages_free
发现`Innodb_buffer_pool_pages_free` 为 0，则说明 `buffer pool` 已经被用光，需要增大`innodb_buffer_pool_size`

#### Innodb_buffer_pool_wait_free 
写入 InnoDB 缓冲池通常在后台进行，但有必要在没有干净页的时候读取或创建页，有必要先等待页被刷新。Innodb的IO线程从数据文件
中读取了数据要写入`buffer pool`的时候，需要等待空闲页的次数。单位是次。  

#### innodb_io_capacity
在5.1.X版本中，最多只会刷新100个脏页到磁盘、合并20个插入缓冲，即使磁盘有能力处理更多的请求，只能会处理这么多，这样在更新量
较大的时候，脏页刷新就可能跟不上，导致性能下降。  
但在5.5.X版本里，`innodb_io_capacity`参数可以动态调整刷新脏页的数量，这在一定程度上解决了这一问题。  
`innodb_io_capacity默认是200`，单位是页，该参数的设置大小取决于硬盘的IOPS，即每秒每秒的输入输出量(或读写次数)。  
可以动态调整参数:`set global innodb_io_capacity=2000;`  
磁盘配置与innodb_io_capacity参数值:  
innodb_io_capacity|磁盘配置    
-|-
200|单盘SAS/SATA
2000|SAS*12  RAID  10
5000|SSD
50000|FUSION-IO

#### innodb_io_capacity_max
`innodb_io_capacity`和`innodb_io_capacity_max` 这些设置会影响InnoDB每秒在后台执行多少操作. 在 以前的一篇文章 里我描
述了大多数写IO(除了写InnoDB日志)是后台操作的. 如果你深度了解硬件性能(如每秒可以执行多少次IO操作),则使用这些功能是很可取
的,而不是让它闲着。  
有一个很好的类比示例:  假如某次航班一张票也没有卖出去 —— 那么让稍后航班的一些人乘坐该次航班,有可能是很好的策略,以防后面遇到恶劣的天气. 即有机会就将后台操作顺便处理了,以减少同稍后可能的实时操作产生竞争。  

有一个很简单的计算:  如果每个磁盘每秒读写(IOPS)可以达到 200次, 则拥有10个磁盘的 RAID10 磁盘阵列IOPS理论上 =(10/2)* 
200 = 1000. 我说它“很简单”,是因为RAID控制器通常能够提供额外的合并,并有效提高IOPS能力. 对于SSD磁盘,IOPS可以轻松达到好几千。  
将这两个值设置得太大可能会存在某些风险,你肯定不希望后台操作妨碍了前台任务IO操作的性能. 过去的经验表明,将这两个值设置的太
高,InnoDB持有的内部锁会导致性能降低(按我了解到的信息,在MySQL5.6中这得到了很大的改进)。    

innodb_lru_scan_depth - 默认值为 1024(mysql 5.7 默认是4000). 这是mysql 5.6中引入的一个新选项. Mark Callaghan   
提供了 一些配置建议. 简单来说,如果增大了 innodb_io_capacity 值, 应该同时增加 innodb_lru_scan_depth.  

#### sync_binlog
表示每次刷新binlog到磁盘的数目。  


#### Innodb_log_waits
`Innodb_log_waits`状态变量，如果它不是0，增加`innodb_log_buffer_size`。


#### innodb_purge_threads = 1
InnoDB中的清除操作是一类定期回收无用数据的操作。在之前的几个版本中，清除操作是主线程的一部分，这意味着运行时它可能会堵塞其它的数据库操作。


#### innodb_log_file_size
这是redo日志的大小。redo日志被用于确保写操作快速而可靠并且在崩溃时恢复。一直到MySQL 5.1，它都难于调整，因为一方面你想让它
更大来提高性能，另一方面你想让它更小来使得崩溃后更快恢复。幸运的是从MySQL 5.5之后，崩溃恢复的性能的到了很大提升，这样你就
可以同时拥有较高的写入性能和崩溃恢复性能了。一直到MySQL 5.5，redo日志的总尺寸被限定在4GB(默认可以有2个log文件)。这在
MySQL 5.6里被提高。  

#### innodb_log_buffer_size
这项配置决定了为尚未执行的事务分配的缓存。其默认值（1MB）一般来说已经够用了，但是如果你的事务中包含有二进制大对象或者大文本
字段的话，这点缓存很快就会被填满并触发额外的I/O操作。看看Innodb_log_waits状态变量，如果它不是0，增加`innodb_log_buffer_size`。
---

变量|状态|解释
-|-|-
Innodb_buffer_pool_pages_data 	 | 496 |包含数据的页数(脏或干净)。
Innodb_buffer_pool_pages_dirty	 | 5   |当前的脏页数
Innodb_buffer_pool_pages_flushed 	 | 1182773 |已经flush的页面数
Innodb_buffer_pool_pages_free | 0 |空页数
Innodb_buffer_pool_pages_misc | 16 |优先用作管理的页数
Innodb_buffer_pool_pages_total | 512 |总页数
Innodb_buffer_pool_read_ahead_rnd | 515979 |随机预读的次数(读大部分数据时)
Innodb_buffer_pool_read_ahead_seq | 3408867 |顺序预读的次数(全表扫描时)
Innodb_buffer_pool_read_requests | 10760142502 |InnoDB已经完成的逻辑读请求数
Innodb_buffer_pool_reads | 6912521 |从磁盘上一页一页的读取的页数，从缓冲池中读取页面， 但缓冲池里面没有， 就会从磁盘读取
Innodb_buffer_pool_wait_free | 0 |缓冲池等待空闲页的次数， 当需要空闲块而系统中没有时， 就会等待空闲页面
Innodb_buffer_pool_write_requests | 8136890 |缓冲池总共发出的写请求次数
Innodb_data_fsyncs | 1457304 |fsync()操作数
Innodb_data_pending_fsyncs | 0 |innodb当前等待的fsync次数
Innodb_data_pending_reads | 0 |innodb当前等待的读的次数
Innodb_data_pending_writes | 0 |innodb当前等待的写的次数
Innodb_data_read | 1009745694720 |总共读入的字节数
Innodb_data_reads | 11756800 |innodb完成的读的次数
Innodb_data_writes | 2308122 |innodb完成的写的次数
Innodb_data_written | 40737600000 | 总共写出的字节数
Innodb_dblwr_pages_written | 1182773 |双写已经写好的页数
Innodb_dblwr_writes | 76828 |已经执行的双写操作数量
Innodb_log_waits | 10 |因为日志缓冲区太小，我们在继续前必须先等待对它清空
Innodb_log_write_requests | 2979109 |日志写请求数
Innodb_log_writes | 1252587 |向日志文件的物理写数量
Innodb_os_log_fsyncs | 1304054 |向日志文件完成的fsync()写数量
Innodb_os_log_pending_fsyncs | 0 |挂起的日志文件fsync()操作数量。
Innodb_os_log_pending_writes | 0 |挂起的日志文件写操作。
Innodb_os_log_written | 1954254848 |写入日志文件的字节数
Innodb_page_size | 16384 |编译的InnoDB页大小(默认16KB)
Innodb_pages_created | 58540 |创建的页数
Innodb_pages_read | 61630057 |从buffer_pool中读取的页数
Innodb_pages_written | 1182773 |写入的页数
Innodb_row_lock_current_waits | 0 |当前等待的待锁定的行数
Innodb_row_lock_time | 595899 |行锁定花费的总时间，单位毫秒
Innodb_row_lock_time_avg | 54  |行锁定的平均时间
Innodb_row_lock_time_max | 7241 |行锁定的最长时间
Innodb_row_lock_waits | 11032 |一行锁定必须等待的时间数
Innodb_rows_deleted | 7199 |删除
Innodb_rows_inserted | 736893 |插入
Innodb_rows_read | 10400853035 |从InnoDB表读取的行数
Innodb_rows_updated | 932768 |更新
