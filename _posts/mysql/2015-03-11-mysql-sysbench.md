---
layout: article
title: "sysbench安装、使用"
modified:
categories: mysql
image:
    teaser: /teaser/tool.jpg
date: 2015-02-11 T15:44:52+08:00
---

> sysbench是一个模块化的、跨平台、多线程基准测试工具，主要用于评估测试各种不同系统参数下的数据库负载情况。
目前sysbench代码托管在launchpad上，项目地址：https://launchpad.net/sysbench（原来的官网 http://sysbench.sourceforge.net 已经不可用），源码采用bazaar管理。

### 1.sysbench支持以下几种测试模式
1、CPU运算性能
2、磁盘IO性能
3、调度程序性能
4、内存分配及传输速度
5、POSIX线程性能
6、数据库性能(OLTP基准测试)
目前sysbench主要支持 mysql,drizzle,pgsql,oracle 等几种数据库。

### 2.编译安装sysbench
{% highlight mysql %}
{% raw %}
cd /usr/local/src/zhagyilig/tools
wget http://imysql.com/wp-content/uploads/2014/09/sysbench-0.4.12-1.1.tgz  
cd  sysbench-0.4.12-1.1
./autogen.sh
./configure --with-mysql-includes=/application/mysql/include/ --with-mysql-libs=/aplication/mysql/lib
make
cd sysbench/  
//如果 make 没有报错，就会在 sysbench 目录下生成二进制命令行工具 sysbench
[root@mysql sysbench]# ll sysbench
-rwxr-xr-x 1 root root 3281344 Feb 11 21:54 sysbench
{% endraw %}
{% endhighlight %}

### 3.OLTP测试环境前准备
初始化测试库环境（总共10个测试表，每个表 100000 条记录，填充随机生成的数据）： 
{% highlight mysql %}
{% raw %}
cd /usr/local/src/zhagyilig/tools/sysbench-0.4.12-1.1/sysbench
mysqladmin -p -S /data/3307/mysql.sock  create sbtest

./sysbench --mysql-host=192.168.21.128 --mysql-port=3307 --mysql-user=root \
--mysql-password=zhagyilig.info --mysql-socket=/data/3307/mysql.sock --test=tests/db/oltp.lua \ 
--oltp_tables_count=10 --oltp-table-size=100000 --rand-init=on prepare

参数解释：
--test=tests/db/oltp.lua 表示调用 tests/db/oltp.lua 脚本进行 oltp 模式测试
--oltp_tables_count=10   表示会生成 10 个测试表
--oltp-table-size=100000 表示每个测试表填充数据量为 100000 
--rand-init=on           表示每个测试表都是用随机数据来填充的
加载测试数据时长视数据量而定，若过程比较久需要稍加耐心等待。
{% endraw %}
{% endhighlight %}

真实测试场景中，数据表建议不低于10个，单表数据量不低于500万行，当然了，要视服务器硬件配置而定。
如果是配备了SSD或者PCIE SSD这种高IOPS设备的话，则建议单表数据量最少不低于1亿行。

### 4.进行OLTP测试
在上面初始化数据参数的基础上，再增加一些参数，即可开始进行测试了：
{% highlight mysql %}
{% raw %}
./sysbench --mysql-host=192.168.21.128 --mysql-port=3307 --mysql-user=root \
--mysql-password=zhagyilig.info --mysql-socket=/data/3307/mysql.sock --test=tests/db/oltp.lua \
--oltp_tables_count=10 --oltp-table-size=10000000 --num-threads=8 --oltp-read-only=off \
--report-interval=10 --rand-type=uniform --max-time=3600 --max-requests=0 \
--percentile=99 run >> /tmp/sysbench_oltpX_8_20150211.log

参数解析：
--num-threads=8      表示发起8个并发连接
--oltp-read-only=off 表示不要进行只读测试，也就是会采用读写混合模式测试
--report-interval=10 表示每10秒输出一次测试进度报告
--rand-type=uniform  表示随机类型为固定模式，其他几个可选随机模式：uniform(固定),gaussian(高斯),special(特定的),pareto(帕累托)
--max-time=120       表示最大执行时长为 120秒
--max-requests=0     表示总请求数为 0，因为上面已经定义了总执行时长，所以总请求数可以设定为 0；也可以只设定总请求数，不设定最大执行时长
--percentile=99      表示设定采样比例，默认是 95%，即丢弃1%的长请求，在剩余的99%里取最大值
{% endraw %}
{% endhighlight %}
即：模拟对10个表并发OLTP测试，每个表1000万行记录，持续压测时间为 1小时。
真实测试场景中，建议持续压测时长不小于30分钟，否则测试数据可能不具参考意义。

### 5.测试结果解读：
测试结果解读如下：
{% highlight mysql %}
{% raw %}
sysbench 0.5:  multi-threaded system evaluation benchmark

Running the test with following options:
Number of threads: 8
Report intermediate results every 10 second(s)
Random number generator seed is 0 and will be ignored


Threads started!
-- 每10秒钟报告一次测试结果，tps、每秒读、每秒写、99%以上的响应时长统计
[  10s] threads: 8, tps: 1111.51, reads/s: 15568.42, writes/s: 4446.13, response time: 9.95ms (99%)
[  20s] threads: 8, tps: 1121.90, reads/s: 15709.62, writes/s: 4487.80, response time: 9.78ms (99%)
[  30s] threads: 8, tps: 1120.00, reads/s: 15679.10, writes/s: 4480.20, response time: 9.84ms (99%)
[  40s] threads: 8, tps: 1114.20, reads/s: 15599.39, writes/s: 4456.30, response time: 9.90ms (99%)
[  50s] threads: 8, tps: 1114.00, reads/s: 15593.60, writes/s: 4456.70, response time: 9.84ms (99%)
[  60s] threads: 8, tps: 1119.30, reads/s: 15671.60, writes/s: 4476.50, response time: 9.99ms (99%)
OLTP test statistics:
    queries performed:
        read:                            938224    -- 读总数
        write:                           268064    -- 写总数
        other:                           134032    -- 其他操作总数(SELECT、INSERT、UPDATE、DELETE之外的操作，例如COMMIT等)
        total:                           1340320    -- 全部总数
    transactions:                        67016  (1116.83 per sec.)      -- 总事务数(每秒事务数)
    deadlocks:                           0      (0.00 per sec.)         -- 发生死锁总数
    read/write requests:                 1206288 (20103.01 per sec.)    -- 读写总数(每秒读写次数)
    other operations:                    134032 (2233.67 per sec.)      -- 其他操作总数(每秒其他操作次数)

General statistics:    -- 一些统计结果
    total time:                          60.0053s     -- 总耗时
    total number of events:              67016        -- 共发生多少事务数
    total time taken by event execution: 479.8171s    -- 所有事务耗时相加(不考虑并行因素)
    response time:    -- 响应时长统计
         min:                                  4.27ms    -- 最小耗时
         avg:                                  7.16ms    -- 平均耗时
         max:                                 13.80ms    -- 最长耗时
         approx.  99 percentile:               9.88ms    -- 超过99%平均耗时

Threads fairness:
    events (avg/stddev):           8377.0000/44.33
    execution time (avg/stddev):   59.9771/0.00
{% endraw %}
{% endhighlight %}