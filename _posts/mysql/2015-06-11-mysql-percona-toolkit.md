
---
layout: article
title: "percona-toolkit 简介"
modified:
categories: mysql
image:
    teaser: /teaser/tool.jpg
---

> 使用percona-toolkit 吧！这些工具主要包括开发、性能、配置、监控、复制、系统、实用六大类，作为一个优秀的DBA，里面有的工具非常有用，如果能掌握并加以灵活应用，将能极大的提高工作效率。
---

## 一.percona-toolkit 功能
- 检查master 和slave 数据的一致性  
- 有效地对记录进行归档  
- 查找重复的索引  
- 对服务器信息进行汇总  
- 分析来自日志和tcpdump 的查询  
- 当系统出问题的时候收集重要的系统信息

## 二.安装出现问题
注意：如果出现`Can’t locate Time/HiRes.pm in @INC`错误，需要安装`perl-Time-`
`HiRes`，执行命令`yum install perl-Time-HiRes`安装即可。
{% highlight mysql %}
{% raw %} 
编译安装percona-toolkit-2.2.2
perl Makefile.PL
Checking if your kit is complete...
Looks good
Warning: prerequisite DBD::mysql 3 not found.
Writing Makefile for percona-toolkit

解决方案：
安装缺少的包
yum install perl-DBD-MySQL
然后，重新编译
perl Makefile.PL 
Writing Makefile for percona-toolkit
继续
make && make test && make install
{% endraw %}
{% endhighlight %}


## 三.常用工具
#### 1.pt-query-digest
(1)直接分析慢查询文件:
`pt-query-digest  slow.log > slow_report.log`

(2)分析最近12小时内的查询：
`pt-query-digest  --since=12h  slow.log > slow_report2.log`

(3)分析指定时间范围内的查询：
{% highlight mysql %}
{% raw %}
pt-query-digest slow.log --since '2014-04-17 09:30:00' --until '2014-04-17 10:00:00'> > slow_report3.log
{% endraw %}
{% endhighlight %}
(4)分析指含有select语句的慢查询
{% highlight mysql %}
{% raw %}
pt-query-digest -p'cgddnai!@#.hp' --filter '$event->{fingerprint} =~ m/^select/i' mysql-slow.log-20170703> slow_report4.log
{% endraw %}
{% endhighlight %}
(5) 针对某个用户的慢查询
{% highlight mysql %}
{% raw %}
pt-query-digest--filter '($event->{user} || "") =~ m/^root/i' slow.log> slow_report5.log
{% endraw %}
{% endhighlight %}
(6) 查询所有所有的全表扫描或full join的慢查询
{% highlight mysql %}
{% raw %}
pt-query-digest--filter '(($event->{Full_scan} || "") eq "yes") ||(($event->{Full_join} || "") eq "yes")' slow.log> slow_report6.log
{% endraw %}
{% endhighlight %}
(7)把查询保存到query_review表
{% highlight mysql %}
{% raw %}
pt-query-digest  --user=root –password=abc123 --review  h=localhost,D=test,t=query_review--create-review-table  slow.log
{% endraw %}
{% endhighlight %}
(8)把查询保存到query_history表
{% highlight mysql %}
{% raw %}
pt-query-digest  --user=root –password=abc123 --review  h=localhost,D=test,t=query_ history--create-review-table  slow.log_20140401
pt-query-digest  --user=root –password=abc123--review  h=localhost,D=test,t=query_history--create-review-table  slow.log_20140402
{% endraw %}
{% endhighlight %}
(9)通过tcpdump抓取mysql的tcp协议数据，然后再分析
{% highlight mysql %}
{% raw %}
tcpdump -s 65535 -x -nn -q -tttt -i any -c 1000 port 3306 > mysql.tcp.txt
pt-query-digest --type tcpdump mysql.tcp.txt> slow_report9.log
{% endraw %}
{% endhighlight %}
(10)分析binlog
{% highlight mysql %}
{% raw %}
mysqlbinlog mysql-bin.000093 > mysql-bin000093.sql
pt-query-digest  --type=binlog  mysql-bin000093.sql > slow_report10.log
{% endraw %}
{% endhighlight %}
(11)分析general log
`pt-query-digest  --type=genlog  localhost.log > slow_report11.log`

#### 2.pt-heartbeat
(1)原理：pt-heartbeat 通过真实的复制数据来确认mysql 和postgresql复制延迟。这个避免了对
复制机制的依赖，从而能得出准确的落后复制时间，包含两部分：第一部分在主上pt-heartbeat 
的–update 线程会在指定的时间间隔更新一个时间戳，第二部分是 pt-heartbeat 的–monitor 
线程或者–check 线程连接到从上检查复制的心跳记录（前面更新的时间戳），并和当前系统时间进
行比较，得出时间的差异。你可以手工创建heartbeat 表或者添加--create-table 参数。表结构
为：

(2)使用方法:
`pt-heartbeat [OPTIONS] [DSN] --update|--monitor|--check|--stop`

(3)在主库上开启守护进程来更新game.heartbeat表：
`pt-heartbeat -D game --update -h master-server --daemonize`

(4)监控从的延迟情况：
`pt-heartbeat -D game --monitor -h slave-server  #一直执行，不退出`
`pt-heartbeat -D game --check h=slave-server     #执行一次就退出`

注意：需要指定的参数至少有 `--stop，--update，--monitor，--check。`其中`--update，--monitor和--check`是互斥的，`--daemonize`和`--check`也是互斥。

(5)参数解析
{% highlight mysql %}
{% raw %}
--ask-pass 隐式输入MySQL密码

--charset 字符集设置

--check 检查从的延迟，检查一次就退出，除非指定了--recurse会递归的检查所有的从服务器

--check-read-only 如果从服务器开启了只读模式，该工具会跳过任何插入

--create-table 在主上创建心跳监控的表，如果该表不存在。可以自己建立，建议存储引擎改成memory。通过更新该表知道主从延迟的差距。

CREATE TABLE heartbeat (
  ts                    varchar(26) NOT NULL,
  server_id             int unsigned NOT NULL PRIMARY KEY,
  file                  varchar(255) DEFAULT NULL,    -- SHOW MASTER STATUS
  position              bigint unsigned DEFAULT NULL, -- SHOW MASTER STATUS
  relay_master_log_file varchar(255) DEFAULT NULL,    -- SHOW SLAVE STATUS
  exec_master_log_pos   bigint unsigned DEFAULT NULL  -- SHOW SLAVE STATUS
);

heratbeat表一直在更改ts和position,而ts是我们检查复制延迟的关键。

--daemonize
执行时，放入到后台执行

--user
-u，连接数据库的帐号

--database
-D，连接数据库的名称

--host
-h，连接的数据库地址

--password
-p，连接数据库的密码

--port
-P，连接数据库的端口

--socket
-S，连接数据库的套接字文件

--file 【--file=output.txt】
打印--monitor最新的记录到指定的文件，很好的防止满屏幕都是数据的烦恼。

--frames 【--frames=1m,2m,3m】
在--monitor里输出的[]里的记录段，默认是1m,5m,15m。可以指定1个，如：--frames=1s，多个用逗号隔开。可用单位有秒（s）、分钟（m）、小时（h）、天（d）。

--interval
检查、更新的间隔时间。默认是见是1s。最小的单位是0.01s，最大精度为小数点后两位，因此0.015将调整至0.02。

--log
开启daemonized模式的所有日志将会被打印到制定的文件中。

--monitor
持续监控从的延迟情况。通过--interval指定的间隔时间，打印出从的延迟信息，通过--file则可以把这些信息打印到指定的文件。

--master-server-id
指定主的server_id，若没有指定则该工具会连到主上查找其server_id。

--print-master-server-id
在--monitor和--check 模式下，指定该参数则打印出主的server_id。

--recurse
多级复制的检查深度。模式M-S-S...不是最后的一个从都需要开启log_slave_updates，这样才能检查到。

--recursion-method
指定复制检查的方式,默认为processlist,hosts。

--update
更新主上的心跳表。

--replace
使用--replace代替--update模式更新心跳表里的时间字段，这样的好处是不用管表里是否有行。

--stop
停止运行该工具（--daemonize），在/tmp/目录下创建一个“pt-heartbeat-sentinel” 文件。后面想重新开启则需要把该临时文件删除，才能开启（--daemonize）。

--table
指定心跳表名，默认heartbeat。


主：192.168.0.11
从：192.168.0.9
主数据库上面运行:

pt-heartbeat --user=root --ask-pss --host=127.0.0.1 --create-table -D game --interval=1 --update --replace --daemonize

主数据库上执行监控：
pt-heartbeat -D game --table=heartbeat --monitor -h192.168.0.9 -uroot --ask-pass

只监控一次
pt-heartbeat -D game --table=heartbeat --check -h192.168.0.9 -uroot --ask-pass 


监控replication延迟:
/usr/bin/pt-heartbeat -D slave_monitor --ask-pass --update --user=root --host=localhost --daemonize

pt-heartbeat -D slave_monitor --ask-pass --table=heartbeat --monitor --host=172.16.101.51 -urep_monitor  --port=9033  --interval=120 --master-server-id=2015121003  --daemonize  --file=/tmp/pt-heartbeat_9033.txt
{% endraw %}
{% endhighlight %}


#### 3.pt-kill 
因为空闲连接较多导致超过最大连接数、某个有问题的sql导致mysql负载很高时，都需要将一些连接kill掉
> 注意：kill:select  别kill:update-->会导致数据不一致问题

(1)参数:
{% highlight mysql %}
{% raw %}
–busy-time运行时间

–idle-time空闲时间

–victims所有匹配的连接，对应有最久的连接

–interval间隔时间，默认30s，有点长，可以根据实际情况来调节

–print 打印出来kill掉的连接

–match-command匹配当前连接的命令
Query Sleep Binlog DumpConnectDelayed insertExecuteFetchInit DBKillPrepareProcesslistQuitReset stmtTable Dump

–match-state匹配当前连接的状态
Lockedlogincopy to tmp tableCopying to tmp tableCopying to tmp table on diskCreating tmp tableexecutingReading from netSending dataSorting for orderSorting resultTable lockUpdating

–match-info使用正则表达式匹配符合的sql

–match-db –match-user –match-host见名知意
{% endraw %}
{% endhighlight %}
(2)常用用法:
{% highlight mysql %}
{% raw %}
杀掉空闲链接
pt-kill –match-command Sleep –idle-time 5 –host –port –interval –print –kill –victims all

杀掉运行时间超过5s的链接
pt-kill –match-command Query –busy-time 5 –host –port –interval –print –kill –victims all

杀掉匹配某个规则的正在运行的sql
pt-kill –match-command Query –busy-time 5 –host –port –interval –print –kill –victims all –match-info

杀掉正在进行filesort的sql
pt-kill –match-command Query –match-state “Sorting result” busy-time 5 –host –port –interval –print –kill –victims all

杀掉正在Copying to tmp table的sql
pt-kill –match-command Query –match-state “Copying to tmp table” busy-time 5 –host –port –interval –print –kill –victims all
{% endraw %}
{% endhighlight %}

#### 4.pt-table-checksum 

生产环境使用 pt-table-checksum 检查MySQL数据一致性，公司数据中心从托管机房迁移到阿里云，需要对mysql迁移（Replication）后的数据一致性进行校验，但又不能对生产环境使用造成影响，pt-table-checksum 成为了绝佳也是唯一的检查工具。