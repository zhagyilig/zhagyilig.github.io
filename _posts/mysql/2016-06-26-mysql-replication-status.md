---
layout: article
title:  "MySQL复制线程状态"
categories: mysql
image:
    teaser: /teaser/mysql rep.png
---  
> 列出了主服务器的 Binlog Dump 线程的 State 列的最常见的状态  


#### 一.复制主线程状态   
下面列出了主服务器的 Binlog Dump 线程的 State 列的最常见的状态。如果你没有在主
服务器上看见任何 Binlog Dump 线程，这说明复制没有在运行—即，目前没有连接任何
从服务器。   
 
`Sending binlog event to slave`     
二进制日志由各种事件组成，一个事件通常为一个更新加一些其它信息。线程已经从二
进制日志读取了一个事件并且正将它发送到从服务器。 
 
`Finished reading one binlog; switching to next binlog`     
线程已经读完二进制日志文件并且正打开下一个要发送到从服务器的日志文件。 
 
`Has sent all binlog to slave; waiting for binlog to be updated`     
线程已经从二进制日志读取所有主要的更新并已经发送到了从服务器。线程现在正空闲，
等待由主服务器上新的更新导致的出现在二进制日志中的新事件。 
 
`Waiting to finalize termination`     
线程停止时发生的一个很简单的状态。 
 
---

#### 二.复制从 IO 线程状态     
下面列出了从服务器的 I/O 线程的 State 列的最常见的状态。该状态也出现在
Slave_IO_State 列，由 SHOW SLAVE STATUS 显示。   
 
`Connecting to master`     
线程正试图连接主服务器。   
 
`Checking master version`     
建立同主服务器之间的连接后立即临时出现的状态。 
 
`Registering slave on master`     
建立同主服务器之间的连接后立即临时出现的状态。 
 
`Requesting binlog dump`   
建立同主服务器之间的连接后立即临时出现的状态。线程向主服务器发送一条请求，索
取从请求的二进制日志文件名和位置开始的二进制日志的内容。 
 
如果二进制日志转储请求失败(由于没有连接)，线程进入睡眠状态，然后定期尝试重新
连接。可以使用`--master-connect-retry` 选项指定重试之间的间隔。 
 
`Reconnecting after a failed binlog dump request`   
线程正尝试重新连接主服务器。 
 
`Waiting for master to send event`   
线程已经连接上主服务器，正等待二进制日志事件到达。如果主服务器正空闲，会持续
较长的时间。如果等待持续 slave_read_timeout 秒，则发生超时。此时，线程认为连接
被中断并企图重新连接。 
 
`Queueing master event to the relay log`     
线程已经读取一个事件，正将它复制到中继日志供 SQL 线程来处理。 
 
`Waiting to reconnect after a failed master event read`   
读取时(由于没有连接)出现错误。线程企图重新连接前将睡眠 `master-connect-retry` 秒。 
 
`Reconnecting after a failed master event read`     
线程正尝试重新连接主服务器。当连接重新建立后，状态变为 `Waiting for master to send` 
`event`。 
 
`Waiting for the slave SQL thread to free enough relay log space`     
正使用一个非零 relay_log_space_limit 值，中继日志已经增长到其组合大小超过该值。
I/O 线程正等待直到 SQL线程处理中继日志内容并删除部分中继日志文件来释放足够的
空间。 
 
`Waiting for slave mutex on exit`     
线程停止时发生的一个很简单的状态。 
 
 
#### 三.复制从 SQL 线程状态   
下面列出了从服务器的 SQL 线程的 State 列的最常见的状态。   
 
`Reading event from the relay log`    
线程已经从中继日志读取一个事件，可以对事件进行处理了。   
 
`Has read all relay log; waiting for the slave I/O thread to update it`     
线程已经处理了中继日志文件中的所有事件，现在正等待 I/O 线程将新事件写入中继日
志。 
 
`Waiting for slave mutex on exit`    
线程停止时发生的一个很简单的状态。   