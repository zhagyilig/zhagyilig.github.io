---
layout: article
title: "mysql中pager命令使用"
modified:
categories: mysql
image:
    teaser: /teaser/tool.jpg
date: 2015-03-11 T15:44:52+08:00
---

> 在mysql中,如果linux下,使用pager命令将大大提高工作效率。

### 1.基本用法
{% highlight mysql %}
{% raw %}
nopager   (\n) Disable pager, print to stdout.
pager     (\P) Set PAGER [to_pager]. Print the query results via PAGER.
{% endraw %}
{% endhighlight %}

	mysql> pager less 
	PAGER set to 'less' 
	mysql> show engine innodb status\G 
 
这个时候就可以开始使用了**less模式**了,可以使用空格到下一页,quit退出; 
甚至可以直接执行linux下的脚本,比如有个脚本在 /tmp/下的lock_waits.sh 
则可以:     

	mysql> pager /tmp/lock_waits   
	PAGER set to '/tmp/lock_waits'  

   
### 2.处理大量的数据集  
如果只想关心结果,可以这样:     

	mysql> pager cat > /dev/null   
	PAGER set to 'cat > /dev/null'   
 
比如执行一系列的冗长的执行计划语句 ,忽略中间过程输出,直接只显示耗时   

	mysql> SELECT ... 
	1000 rows in set (0.91 sec) 
 
	mysql> SELECT ... 
	1000 rows in set (1.63 sec) 
 
又比如,如果你在进行SQL调优,有大量的结果产生   
	mysql> SELECT ...   
	[..]   
	989 rows in set (0.42 sec)   

可以通过checksum去比较每次调整后的SQL语句所产生的结果是否是相同的     
 
	mysql> pager md5sum 
	PAGER set to 'md5sum' 
 
	# Original query 
	mysql> SELECT ... 
	32a1894d773c9b85172969c659175d2d  - 
	1 row in set (0.40 sec) 
 
	# Rewritten query - wrong 
	mysql> SELECT ... 
	fdb94521558684afedc8148ca724f578  - 
	1 row in set (0.16 sec) 
 	 这里checksum不同,所以重写的SQL语句有问题   
   
如果有大量的连接,用show processlist看会比较不大方便,比如要知道哪些当前的连接是睡眠或者死掉的,就不大方便,可以这样:   

	mysql> pager grep Sleep | wc -l 
	PAGER set to 'grep Sleep | wc -l' 
	mysql> show processlist; 
	337 
	346 rows in set (0.00 sec) 
  	马上看到当前有多少连接sleep了;   
  	进一步,要知道每一种状态的连接情况,可以这样:  
{% highlight mysql %}
{% raw %} 
mysql> pager awk -F '|' '{print $6}' | sort | uniq -c | sort -r 
PAGER set to 'awk -F '|' '{print $6}' | sort | uniq -c | sort -r' 
mysql> show processlist; 
 309  Sleep       
   3 
   2  Query       
   2  Binlog Dump 
   1  Command 

当然,也可以用SQL查询的方式实现了: 
mysql> SELECT COMMAND,COUNT(*) TOTAL FROM INFORMATION_SCHEMA.PROCESSLIST GROUP BY COMMAND ORDER BY TOTAL DESC;
+---------+-------+
| COMMAND | TOTAL |
+---------+-------+
| Connect |     2 |
| Query   |     1 |
+---------+-------+
2 rows in set (0.02 sec)

mysql> SELECT COUNT(*) FROM INFORMATION_SCHEMA.PROCESSLIST WHERE COMMAND='Sleep';
+----------+
| COUNT(*) |
+----------+
|        0 |
+----------+
1 row in set (0.00 sec  
{% endraw %}
{% endhighlight %}

#### 3.设置Innodb_log_file_size,控制检查点
{% highlight mysql %}
{% raw %}
mysql> pager grep -i "log sequence number"
PAGER set to 'grep -i "log sequence number"'
mysql> show engine innodb status\G select sleep(60); show engine innodb status\G
Log sequence number 860789102920
1 row in set (0.11 sec)

1 row in set (59.99 sec)

Log sequence number 860790229906
1 row in set (0.00 sec)
mysql> nopager;
PAGER set to stdout
mysql> select round((860790229906 - 860789102920) / 1024 / 1024) as MB;
+------+
| MB   |
+------+
|    1 |
+------+
//每分钟产生的日志量是1M，半小时就是30*1=30M
//经验：平均没半个小时写满一个文件比较合适
//那么 innodb_log_file_size = 32M 比较合适
{% endraw %}
{% endhighlight %}
