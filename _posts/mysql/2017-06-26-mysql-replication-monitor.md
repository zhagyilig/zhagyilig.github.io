---
layout: article
title:  "MySQL主从复制延迟一例"
categories: mysql
image:
    teaser: /teaser/mysql rep.png
---

>发现MySQL的复制（replication）进度总是落后很多肿么办？  
>MySQL复制（replication）是否有延迟，延迟多厉害，只看Seconds_Behind_Master是否可靠？  
  
### MySQL复制延迟很大怎么办
一般而言，slave相对master延迟较大，其根本原因就是slave上的复制线程没办法真正做到并发。
简单说，在master上是并发模式（以InnoDB引擎为主）完成事务提交的，而在slave上，复制线程只有一个sql thread用
于binlog的apply，所以难怪slave在高并发时会远落后master。    

ORACLE MySQL 5.6版本开始支持多线程复制，配置选项` slave_parallel_workers `即可实现在slave上多线程并发复制。
不过，它只能支持一个实例下多个 database 间的并发复制，并不能真正做到多表并发复制。因此在较大并发负载时，slave
还是没有办法及时追上master，需要想办法进行优化。    
  
另一个重要原因是，传统的MySQL复制是**异步**（asynchronous）的，也就是说在master提交完后，才在slave上再应用一遍，并不是真正意义上的同步。  
哪怕是后来的`Semi-sync Repication`（半同步复制），也不是真同步，因为它只保证事务传送到slave，但没要求等到确认事务提交成功。既然是异
步，那肯定多少会有延迟。因此，严格意义上讲，MySQL复制不能叫做MySQL同步。  

另外，不少人的观念里，slave相对没那么重要，因此就不会提供和master相同配置级别的服务器。有的甚至不但使用更差的服务器，而且还在上面跑多实例。

综合这两个主要原因，slave想要尽可能及时跟上master的进度，可以尝试采用以下几种方法：  

- 采用MariaDB发行版，它实现了相对真正意义上的并行复制，其效果远比ORACLE MySQL好的很多。在我的场景中，采用MariaDB作为slave的实例，几乎总是能及时跟上master。如果不想用这个版本的话，那就老实等待官方5.7大版本发布吧； 关于MariaDB的Parallel Replication具体请参考：`Replication and Binary Log` 
`Server System Variables#slave_parallel_threads - MariaDB Knowledge Base`    
- 每个表都要显式指定主键，没有指定主键的话，会导致在row模式下，每次修改都要全表扫描，尤其是大表就非常可怕了，延迟会更严重，甚至导致整个slave库都被挂起,下面的故障就是指定主键导致；  
- 应用程序端多做些事，让MySQL端少做事，尤其是和IO相关的活动，例如：前端通过内存CACHE或者本地写队列等，合并多次读写为一次，甚至消除一些写请求；    
- 进行合适的分库、分表策略，减小单库单表复制压力，避免由于单库单表的的压力导致整个实例的复制延迟；
- 其他提高IOPS性能的几种方法，根据效果优劣，我做了个简单排序：
更换成SSD，或者PCIe SSD等IO设备，其IOPS能力的提升是普通15K SAS盘的数以百倍、万倍，甚至几十万倍计；
- 加大物理内存，相应提高`InnoDB Buffer Pool`大小，让更多热数据放在内存中，降低发生物理IO的频率，当然也不是盲目的调整，一般是服务器内存的50%~70%，具体看另一边博文（[MySQL参数优化配置](http://tj.zhagyilig.info/mysql/mysql%E4%BC%98%E5%8C%96%E5%8F%82%E6%95%B0%E8%AE%BE%E7%BD%AE/)）；    
- 调整文件系统为 XFS 或 ReiserFS，相比ext3可以极大程度提高IOPS能力。在高IOPS压力下，相比ext4有更稳健的IOPS表现（有人认为 XFS 在特别的场景下会有很大的问题，但我们除了剩余磁盘空间少于10%时引发丢数据外，其他的尚未遇到）；    
- 调整RAID级别为raid10，它相比raid1、raid5等更能提高IOPS性能。如果已经全部是SSD设备了，可以2块盘做成RAID 1，或者多快盘做成RAID 5（并且可以设置全局热备盘，提高阵列容错性），甚至有些土豪用户直接将多块SSD盘组成RAID 50；    
- 调整RAID的写cache策略为`WB或FORCE WB`，详情请参考[叶老师在百度文库上的分享](http://www.jb51.net/hardware/464106.html)  

- 调整内核的io scheduler，优先使用d**eadline**，如果是SSD，则可以使用**noop**策略，相比默认的cfq，个别情况下对IOPS的性能提升至少是数倍的。  

### MySQL复制延迟一例
**1.发现问题**
{% highlight mysql %}
{% raw %}	
	root@linux-node3.mysql01.zyl (none) >show slave status\G  
		********** 1. row **********
               Slave_IO_State: Waiting for master to send event
               ...
          Read_Master_Log_Pos: 345318206
        	   ...
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
               ...
          Exec_Master_Log_Pos: 127824051
               ...
        Seconds_Behind_Master: 1177885
          	   ...
      Slave_SQL_Running_State: Reading event from the relay log  

	root@linux-node3.mysql01.zyl (none) >pager cat | egrep -i 'system user|Exec_Master_Log_Pos|Seconds_Behind_Master|Read_Master_Log_Pos';
	PAGER set to 'cat | egrep -i 'system user|Exec_Master_Log_Pos|Seconds_Behind_Master|Read_Master_Log_Pos''  

	root@linux-node3.mysql01.zyl (none) >show slave status\G
          Read_Master_Log_Pos: 345703585
          Exec_Master_Log_Pos: 127824051
        Seconds_Behind_Master: 1178062  
{% endraw %}
{% endhighlight %}
**2.定位问题**  
  省略一万字，无非就是在群里大吼，哈哈...  
  开发同事在半个月前向该MySQL server中新加一个库。巧了，延迟时间就是快半个月，于是快速定位出现问题的表。    
{% highlight mysql %}
{% raw %}	
root@linux-node3.mysql01.zyl (xxx) >show create tabels v_fam_sign_focusgroups_levelone;  
`fam_id` varchar(32) CHARACTER SET utf8 DEFAULT NULL,
`fam_name` varchar(50) CHARACTER SET utf8 DEFAULT NULL,
`teamid` varchar(32) CHARACTER SET utf8 DEFAULT NULL,
`team_name` varchar(32) CHARACTER SET utf8 DEFAULT NULL,
`sign_orgid` varchar(32) CHARACTER SET utf8 DEFAULT NULL,
`sign_date` varchar(10) CHARACTER SET utf8 DEFAULT NULL,
`sign_count` bigint(21) NOT NULL DEFAULT '0',
`type` varchar(10) CHARACTER SET utf8 NOT NULL DEFAULT '',
`hospital_name_levelone` varchar(30) CHARACTER SET utf8 DEFAULT NULL,
`hospital_name_leveltwo` varchar(30) CHARACTER SET utf8 DEFAULT NULL,
`hospital_name_levelthree` varchar(30) CHARACTER SET utf8 DEFAULT NULL,
`country` varchar(32) CHARACTER SET utf8 DEFAULT NULL,
`sign_orgid_lv2` varchar(32) CHARACTER SET utf8 DEFAULT NULL,
`sign_orgid_lv3` varchar(32) CHARACTER SET utf8 DEFAULT NULL,
`ctiy` varchar(32) CHARACTER SET utf8 DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4

root@localhost healthcloud_famdoc > show index from v_fam_sign_focusgroups_levelone;
Empty set (0.00 sec)

显然，百万条数据的大表，竟然没主键。
{% endraw %}
{% endhighlight %}
**3.解决问题**   
3.1 新建表指定主键  
3.2 xtrabackup搭主从    
{% highlight mysql %}
{% raw %} 
 	    ...
 	 PRIMARY KEY (`id`)
	 ) ENGINE=InnoDB AUTO_INCREMENT=1229006 DEFAULT CHARSET=utf8mb4
	 ...

	root@localhost healthcloud_famdoc > show index from v_fam_sign_focusgroups_levelone_two;
	+-------------------------------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
	| Table                               | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
	+-------------------------------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
	| v_fam_sign_focusgroups_levelone_two |          0 | PRIMARY  |            1 | id          | A         |     1222916 |     NULL | NULL   |      | BTREE      |         |               |
	+-------------------------------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
	1 row in set (0.00 sec)   
{% endraw %}
{% endhighlight %}
XtraBackup搭建主从：
知识准备：
1、在备份InnoDB的过程中，记录的变更保存于xtrabackup_logfile文件，所以在prepare(–apply-log)的时候，需要重放该部分数据到表空间。
2、如果库中只使用了innodb或者XtraDB引擎，恢复的时候使用xtrabackup_binlog_pos_innodb文件确定pos信息;
3、如果还有其他引擎(如MyISAM)，恢复的时候使用xtrabackup_binlog_info确定pos信息;
	
1.这主库上进行全备，然后进行prepare
{% highlight mysql %}
{% raw %} 
backup:
[root@linux-node3 log]# innobackupex --defaults-file='/etc/my.cnf' --user=root --password=it.zhagyilig.info --user-memory=4096M  --no-timetamp --backup /opt/   

prepare:
[root@linux-node3 log]#innobackupex  --defaults-file='/etc/my.cnf' --apply-log  2017-06-26_20-53-59/ 
{% endraw %}
{% endhighlight %}  

2.将备份进行压缩，拷贝到slave

	rsync -avzp 2017-06-26_20-53-59/ xxx@xxx:/tmp/  

3.删除之前的日志文件，数据，一般在data目录，并move-back全备。
注意：在删除bin-log的日志：purge 命令
 
	cd /data/mysql/data/
	innobackupex  --move-back /tmp/2017-06-26_20-53-59/

注意：也可用--copy-back，不过--move-back效率更高。

**4.配置slave**

	[root@linux-mysql-slave data]# cat xtrabackup_binlog_pos_innodb
	mysql-bin.000425		366160145
 

	stop slave;

	CHANGE MASTER TO  
	MASTER_HOST='192.168.21.161', 
	MASTER_PORT=3306,
	MASTER_USER='rep', 
	MASTER_PASSWORD='replication', 
	MASTER_LOG_FILE='mysql-bin.000425',
	MASTER_LOG_POS=366160145; 

	start slave;

	show slave status\G
	*********** 1. row ***********
     
         Slave_IO_Running: Yes
        Slave_SQL_Running: Yes
		Seconds_Behind_Master: 0
	Exec_Master_Log_Pos: 352892378
	Read_Master_Log_Pos: 352892378
	 ....
	//利用innobackupex搭建MySQL从库 结束。

**5.故障总结**   
1.MySQL权限管理；  
2.流程控制制度不完善；
3.监控不到位。

### 精确监控slave延迟
> 从前面的故障教训，调整slave监控方案

1、首先看 `Relay_Master_Log_File` 和 `Master_Log_File` 是否有差异；  
2、如果`Relay_Master_Log_File` 和 `Master_Log_File` 是一样的话，再来看`Exec_Master_Log_Pos` 和 `Read_Master_Log_Pos` 的差异，对比SQL线程比IO线程慢了多少个binlog事件；   
3、如果`Relay_Master_Log_File` 和 `Master_Log_File` 不一样，那说明延迟可能较大，需要从`MASTER上取得binlog status`，判断当前的binlog和MASTER上的差距；  
  
因此，相对更加严谨的做法是：
在第三方监控节点上，对`MASTER`和`SLAVE`同时发起`SHOW BINARY LOGS`和`SHOW SLAVE STATUS\G`的请求，最后判断二者`binlog`的差异，以及 `Exec_Master_Log_Pos` 和`Read_Master_Log_Pos` 的差异。

	
