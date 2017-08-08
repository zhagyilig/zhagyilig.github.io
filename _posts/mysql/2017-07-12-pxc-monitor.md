---
layout: article
title:  "pxc集群主要status监控"
categories: mysql
toc: true
ads: true
<<<<<<< HEAD
	teaser: /teaser/mysql01.png
=======
image:
    teaser: ![teaser](http://ou529e3sj.bkt.clouddn.com/pxc.jpg)
>>>>>>> d4302f56baf7c4dfdc7897377cf99059e131dbda
---  

> Percona XtraDB Cluster集群status监控指标介绍

## 一、监控status
{% highlight mysql %}
{% raw %}
1、wsrep_local_state_comment

root@(none) 03:57:28>show  global status like  'wsrep_local_state_comment';
+---------------------------+--------+
| Variable_name             | Value  |
+---------------------------+--------+
| wsrep_local_state_comment | Synced |
+---------------------------+--------+
以人能读懂的方式显示节点的状态，正常的返回值是Joining, Waiting on SST, Joined, Synced or  
Donor，返回Initialized说明已不在正常工作状态

2、wsrep_evs_state

root@(none) 03:45:24>show  global status like '%wsrep_evs_state%';
+-----------------+-------------+
| Variable_name   | Value       |
+-----------------+-------------+
| wsrep_evs_state | OPERATIONAL |  // operational:操作
+-----------------+-------------+

3、wsrep_cluster_size

root@(none) 03:45:29>show  global status like '%wsrep_cluster_size%';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+

4、wsrep_cluster_status

root@(none) 03:45:39>show  global status like '%wsrep_cluster_status%';  
+----------------------+---------+
| Variable_name        | Value   |
+----------------------+---------+
| wsrep_cluster_status | Primary |
+----------------------+---------+

5、wsrep_ready  

root@(none) 03:45:48>show  global status like '%wsrep_ready%';  
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| wsrep_ready   | ON    |
+---------------+-------+
wsrep_ready显示了节点是否可以接受queries。ON表示正常，如果是OFF几乎所有的query都会报错，报错信息提示“ERROR 1047 (08501) Unknown Command”   

6、wsrep_incoming_addresses  

root@(none) 03:45:58>show  global status like '%wsrep_incoming_addresses%';  
+--------------------------+-------------------------------------------------------+
| Variable_name            | Value                                                 |
+--------------------------+-------------------------------------------------------+
| wsrep_incoming_addresses | 172.19.29.11:5501,172.19.29.12:5501,172.19.29.13:5501 |
+--------------------------+-------------------------------------------------------+
{% endraw %}
{% endhighlight %}


1.下面内容原文:
[http://www.open-open.com/lib/view/open1451554653683.html](http://www.open-open.com/lib/view/open1451554653683.html)

2.wsrep_notify_cmd.sh——监控状态的变化。使用方法参见:
[http://galeracluster.com/documentation-webpages/notificationcmd.html](http://galeracluster.com/documentation-webpages/notificationcmd.html)

## 二、检查节点状态
节点状态显示了集群中的节点接受和更新write-set状态，以及可能阻止复制的一些问题
{% highlight mysql %}
{% raw %}
1、wsrep_ready显示了节点是否可以接受queries。ON表示正常，如果是OFF几乎所有的query都会报错，报错信息提
示“ERROR 1047 (08501) Unknown Command”

SHOW GLOBAL STATUS LIKE 'wsrep_ready';
+---------------+-------+

| Variable_name | Value |
+---------------+-------+
| wsrep_ready   | ON    |    // ready:准备
+---------------+-------+

2、SHOW GLOBAL STATUS LIKE 'wsrep_connected';
显示该节点是否与其他节点有网络连接。(实验得知，当把某节点的网卡down掉之后，该值仍为on。说明网络还在)丢
失连接的问题可能在于配置wsrep_cluster_address或wsrep_cluster_name的错误
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| wsrep_connected | ON    |
+-----------------+-------+

3、wsrep_local_state_comment 以人能读懂的方式显示节点的状态，正常的返回值是Joining, Waiting on SST, 
Joined, Synced or Donor，返回Initialized说明已不在正常工作状态
+---------------------------+--------+
| Variable_name             | Value  |
+---------------------------+--------+
| wsrep_local_state_comment | Synced |
+---------------------------+--------+
{% endraw %}
{% endhighlight %}

## 三、集群复制状态检查
{% highlight mysql %}
{% raw %}
1、SHOW GLOBAL STATUS LIKE 'wsrep_%';

2、wsrep_cluster_state_uuid显示了cluster的state UUID,由此可看出节点是否还是集群的一员

SHOW GLOBAL STATUS LIKE 'wsrep_cluster_state_uuid'

集群内每个节点的value都应该是一样的，否则说明该节点不在集群中了

+--------------------------+--------------------------------------+
| Variable_name                    | Value                        |
+--------------------------+--------------------------------------+
| wsrep_cluster_state_uuid | 9f6a992a-7dd9-11e5-9f85-f760745ffb39 |
+--------------------------+--------------------------------------+

3、wsrep_cluster_conf_id显示了整个集群的变化次数。所有节点都应相同，否则说明某个节点与集群断开了

4、wsrep_cluster_size显示了集群中节点的个数

5、wsrep_cluster_status显示集群里节点的主状态。标准返回primary。如返回non-Primary或其他值说明是多个节点改
变导致的节点丢失或者脑裂。如果所有节点都返回不是Primary，则要重设quorum。具体参见http://galeracluster.com/d
ocumentation-webpages/quorumreset.html如果返回都正常，说明复制机制在每个节点都能正常工作，下一步该检查每个
节点的状态确保他们都能收到write-set

show global status like 'wsrep_cluster_status';
+----------------------+---------+
| Variable_name        | Value   |
+----------------------+---------+
| wsrep_cluster_status | Primary |
+----------------------+---------+
{% endraw %}
{% endhighlight %}

## 四、检查节点状态

节点状态显示了集群中的节点接受和更新write-set状态，以及可能阻止复制的一些问题
{% highlight mysql %}
{% raw %}
1、wsrep_ready显示了节点是否可以接受queries。ON表示正常，如果是OFF几乎所有的query都会报错，
报错信息提示“ERROR 1047 (08501) Unknown Command”

SHOW GLOBAL STATUS LIKE 'wsrep_ready';

+---------------+-------+

| Variable_name | Value |
+---------------+-------+
| wsrep_ready   | ON    |
+---------------+-------+

2、SHOW GLOBAL STATUS LIKE 'wsrep_connected’显示该节点是否与其他节点有网络连接。
（实验得知，当把某节点的网卡down掉之后，该值仍为on。说明网络还在）丢失连接的问题可能在于配置
wsrep_cluster_address或wsrep_cluster_name的错误

+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| wsrep_connected | ON    |
+-----------------+-------+

3、wsrep_local_state_comment 以人能读懂的方式显示节点的状态，正常的返回值是Joining, Waiting on SST,
 Joined, Synced or Donor，返回Initialized说明已不在正常工作状态

+---------------------------+--------+
| Variable_name             | Value  |
+---------------------------+--------+
| wsrep_local_state_comment | Synced |
+---------------------------+--------+
{% endraw %}
{% endhighlight %}  

## 五、查看复制的健康状态

通过Flow Control的反馈机制来管理复制进程。当本地收到的write-set(wsrep)超过某一阀值时，该节点会启动flow 
control来暂停复制直到它赶上进度。监控本地收到的请求和flow control，有如下几个参数：
{% highlight mysql %}
{% raw %}
1、wsrep_local_recv_queue_avg:平均请求队列长度。当返回值大于0时，说明apply write-sets比收write-set慢，有等待。堆积太多可能导致启动flow control

root@(none) 04:04:40>SHOW GLOBAL STATUS LIKE 'wsrep_local_recv_queue_avg';
+----------------------------+----------+
| Variable_name              | Value    |
+----------------------------+----------+
| wsrep_local_recv_queue_avg | 0.000115 |
+----------------------------+----------+

wsrep_local_recv_queue_max 和 wsrep_local_recv_queue_min可以看队列设置的最大最小值

2、wsrep_flow_control_paused 显示了自从上次查询之后，节点由于flow 
control而暂停的时间占整个查询间隔时间比。总体反映节点落后集群的状况。如果返回值为1，说明自
上次查询之后，节点一直在暂停状态。如果发现某节点频繁落后集群，则应该调整wsrep_slave_threads或者把节点剔除

root@(none) 04:05:26>SHOW GLOBAL STATUS LIKE 'wsrep_flow_control_paused';
+---------------------------+----------+
| Variable_name             | Value    |
+---------------------------+----------+
| wsrep_flow_control_paused | 0.000000 |
+---------------------------+----------+


root@(none) 04:06:27>show variables like 'wsrep_slave_threads';
+---------------------+-------+
| Variable_name       | Value |
+---------------------+-------+
| wsrep_slave_threads | 1     |
+---------------------+-------+

3、wsrep_cert_deps_distance显示了平行apply的最低和最高排序编号或者sql编号之间的平均距离值。这代表了节点潜在
的并行程度，和线程相关

root@(none) 04:06:40>SHOW GLOBAL STATUS LIKE 'wsrep_cert_deps_distance';
+--------------------------+-----------+
| Variable_name            | Value     |
+--------------------------+-----------+
| wsrep_cert_deps_distance | 56.984045 |
+--------------------------+-----------+
{% endraw %}
{% endhighlight %}

## 六、检测网络慢的问题
通过检查发送队列来看传出的连接状况
{% highlight mysql %}
{% raw %}
1、wsrep_local_send_queue_avg显示自上次查询之后的平均发送队列长度。比如网络瓶颈和flow control都可能是原因

root@(none) 04:07:05>SHOW GLOBAL STATUS LIKE 'wsrep_local_send_queue_avg';
+----------------------------+----------+
| Variable_name              | Value    |
+----------------------------+----------+
| wsrep_local_send_queue_avg | 0.001036 |
+----------------------------+----------+

wsrep_local_send_queue_max 和 wsrep_local_send_queue_min可以看队列设置的最大值和最小值
{% endraw %}
{% endhighlight %}

## 七、日志监控
在my.cnf中做如下配置
{% highlight mysql %}
{% raw %}
//wsrep Log Options

wsrep_log_conflicts=ON   	// 会将冲突信息写入错误日志中,例如两个节点同时写同一行数据

wsrep_provider_options="cert.log_conflicts=ON"    // 复制过程中的错误信息写在日志中

wsrep_debug=ON    // 显示debug 信息在日志中，其中也包括鉴权信息，例如账号密码。因此在生产环境中不开启
{% endraw %}
{% endhighlight %}

## 八、附加的日志
当某节点在从节点上应用一个事件失败时,数据库服务器会创建一个特殊的binary log文件。文件名默认是GRA_*.log.





