---
layout: article
title:  "SLAVE为什么停住了"
categories: mysql
image:
    teaser: /teaser/mysql rep.png
---  

> 遇到SLAVE延迟很大，binlog apply position一直不动的情况如何排查？  


### 问题描述
收到SLAVE延迟时间一直很大的报警，于是检查一下SLAVE状态（无关状态我给隐去了）：  
{% highlight mysql %}
{% raw %}	
   Slave_IO_State: Waiting for master to send event
         Master_Log_File: mysql-bin.000605
     Read_Master_Log_Pos: 1194
          Relay_Log_File: mysql-relay-bin.003224
           Relay_Log_Pos: 295105
   Relay_Master_Log_File: mysql-bin.000604
        Slave_IO_Running: Yes
       Slave_SQL_Running: Yes
              Last_Errno: 0
              Last_Error: 
     Exec_Master_Log_Pos: 294959
         Relay_Log_Space: 4139172581
   Seconds_Behind_Master: 10905
{% endraw %}
{% endhighlight %}    
 
可以看到，延迟确实很大，而且从多次show slave status的结果来看，发现binlog的position一直不动。
{% highlight mysql %}
{% raw %}
     Read_Master_Log_Pos: 1194
          Relay_Log_File: mysql-relay-bin.003224
           Relay_Log_Pos: 295105
   Relay_Master_Log_File: mysql-bin.000604
     Exec_Master_Log_Pos: 294959
         Relay_Log_Space: 4139172581
{% endraw %}
{% endhighlight %}  
从processlist的中也看不出来有什么不对劲的SQL在跑：  
{% highlight mysql %}
{% raw %}
******************** 1. row ******************
     Id: 16273070
   User: system user
   Host:
     db: NULL
Command: Connect
   Time: 4828912
  State: Waiting for master to send event
   Info: NULL
********************* 2. row *****************
     Id: 16273071
   User: system user
   Host:
     db: NULL
Command: Connect
   Time: 9798
  State: Reading event from the relay log
   Info: NULL 
{% endraw %}
{% endhighlight %}   

在master上查看相应binlog，确认都在干神马事：  
{% highlight mysql %}
{% raw %}
[yejr@imysql.com]# mysqlbinlog -vvv --base64-output=decode-rows -j 294959 mysql-bin.000604 | more

/*!40019 SET @@session.max_insert_delayed_threads=0*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
**# at 294959**
#160204  6:16:30 server id 1  end_log_pos 295029     **Query    thread_id=461151**    **exec_time=2144**    error_code=0
SET TIMESTAMP=1454537790/*!*/;
SET @@session.pseudo_thread_id=461151/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=0/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C latin1 *//*!*/;
SET @@session.character_set_client=8,@@session.collation_connection=8,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 295029
# at 295085
# at 296040
# at 297047
# at 298056
# at 299068
# at 300104
{% endraw %}  
{% endhighlight %}   
上面这段内容的几个关键信息：        
{% highlight mysql %}  
{% raw %}
#at 294959        — binlog起点    
thread_id=461151  — master上执行的线程ID    
exec_time=2144    — 该事务执行总耗时    
{% endraw %}
{% endhighlight %}     
再往下看都是一堆的binlog position信息，通过这种方式可读性不强，我们换一种姿势看看：  
{% highlight mysql %}  
{% raw %}
[yejr@imysql.com (test)]> show binlog events in 'mysql-bin.000604' from 294959 limit 10;
+------------------+--------+-------------+-----------+-------------+----------------------------+
| Log_name         | Pos    | Event_type  | Server_id | End_log_pos | Info                       |
+------------------+--------+-------------+-----------+-------------+----------------------------+
| mysql-bin.000604 | 294959 | Query       |         1 |      295029 | BEGIN                      |
| mysql-bin.000604 | 295029 | Table_map   |         1 |      295085 | table_id: 84 (bacula.File) |
| mysql-bin.000604 | 295085 | Delete_rows |         1 |      296040 | table_id: 84               |
| mysql-bin.000604 | 296040 | Delete_rows |         1 |      297047 | table_id: 84               |
| mysql-bin.000604 | 297047 | Delete_rows |         1 |      298056 | table_id: 84               |
| mysql-bin.000604 | 298056 | Delete_rows |         1 |      299068 | table_id: 84               |
| mysql-bin.000604 | 299068 | Delete_rows |         1 |      300104 | table_id: 84               |
| mysql-bin.000604 | 300104 | Delete_rows |         1 |      301116 | table_id: 84               |
| mysql-bin.000604 | 301116 | Delete_rows |         1 |      302147 | table_id: 84               |
| mysql-bin.000604 | 302147 | Delete_rows |         1 |      303138 | table_id: 84               |  
{% endraw %}
{% endhighlight %}  

可以看到，这个事务不干别的，一直在删除数据。    
这是一个Bacula备份系统，会每天自动删除一个月前的过期数据。    
事实上，这个事务确实非常大，从binlog的294959开始，一直到这个binlog结束4139169218，一直都是在干这事，总共大概有3.85G的binlog要等着apply。      
{% highlight mysql %}
{% raw %}
-rw-rw---- 1 mysql mysql 1.1G Feb  3 03:07 mysql-bin.000597
-rw-rw---- 1 mysql mysql 1.1G Feb  3 03:19 mysql-bin.000598
-rw-rw---- 1 mysql mysql 2.1G Feb  3 03:33 mysql-bin.000599
-rw-rw---- 1 mysql mysql 1.4G Feb  3 03:45 mysql-bin.000600
-rw-rw---- 1 mysql mysql 1.8G Feb  3 04:15 mysql-bin.000601
-rw-rw---- 1 mysql mysql 1.3G Feb  3 04:53 mysql-bin.000602
-rw-rw---- 1 mysql mysql 4.5G Feb  4 06:16 mysql-bin.000603
-rw-rw---- 1 mysql mysql 3.9G Feb  4 06:52 mysql-bin.000604
-rw-rw---- 1 mysql mysql 1.2K Feb  4 06:52 mysql-bin.000605  
{% endraw %}
{% endhighlight %}  
可以看到上面的历史binlog，个别情况下，一个事务里一次性要删除数据量太大了，导致binlog文件远超预设的1G，最大的达到4.5G之多。    
 
### 怎么解决  
由于这是Bacula备份系统内置生成的大事务，除非去修改它的源码，否则没有太好的办法。    

对于我们一般的应用而言，最好是攒够一定操作后，就先提交一下事务，比如删除几千条记录后提交一次，而不是像本例这样，一个删除事务消耗了将近3.9G的binlog日质量，这种就非常可怕了。    

除了会导致SLAVE看起来一直不动以外，还可能会导致某些数据行（data rows）被长时间锁定不释放，而导致大量行锁等待发生。      

其他导致SLAVE复制进度看起来停滞了的可能原因：设置了Replicate Ignore/Do DB/Table规则，不符合规则的binlog event都会被忽略，从而看起来像是复制停滞不前。     

[**阅读原文~叶老师**](https://mp.weixin.qq.com/s?__biz=MjM5NzAzMTY4NQ==&mid=506446074&idx=3&sn=5bc1eb59d278e545ab5ea836437ea162&scene=19#wechat_redirect)