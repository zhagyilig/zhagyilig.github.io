---
layout: article
title:  "Python操作RedisCluster"
categories: python
toc: true
ads: true
image:
    teaser: /python/redis.jpeg
---

> redis.exceptions.ResponseError: MOVED 5581

---  

### 1.redis.exceptions.ResponseError: MOVED 异常   
~~~ python
[root@jdu4e00u53f7 python]# python3 redis_c.py
Traceback (most recent call last):
  File "redis_c.py", line 17, in <module>
    r.set(random_str,random_hit)
  File "/usr/local/lib/python3.5/site-packages/redis/client.py", line 1171, in set
    return self.execute_command('SET', *pieces)
  File "/usr/local/lib/python3.5/site-packages/redis/client.py", line 668, in execute_command
    return self.parse_response(connection, command_name, **options)
  File "/usr/local/lib/python3.5/site-packages/redis/client.py", line 680, in parse_response
    response = connection.read_response()
  File "/usr/local/lib/python3.5/site-packages/redis/connection.py", line 629, in read_response
    raise response
redis.exceptions.ResponseError: MOVED 5581 127.0.0.1:8001
~~~  

### 2.报错原因     
redis由单节点变为集群，而python的redis连接包暂时还不支持redis集群连接方式，需要更换连接包.    


### 3.解决方法      
可以使用rediscluster连接redis集群:      
参考文档:[https://pypi.python.org/pypi/redis-py-cluster](https://pypi.python.org/pypi/redis-py-cluster)      

### 4.安装redis-py-cluster    
~~~ python
(venv)[root@jdu4e00u53f7 ~]# pip3 install redis-py-cluster
Collecting redis-py-cluster
  Downloading redis_py_cluster-1.3.4-py2.py3-none-any.whl
Requirement already satisfied (use --upgrade to upgrade): redis>=2.10.2 in /usr/local/lib/python3.5/site-packages (from redis-py-cluster)
Installing collected packages: redis-py-cluster
Successfully installed redis-py-cluster-1.3.4
You are using pip version 8.1.1, however version 9.0.1 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
~~~

### 5.简单的脚本如下  
~~~ python
#!/usr/local/bin/python3.5
# -*- coding: utf-8 -*-
# Time    : 2018/2/1 下午1:11
# Author  : xtrdb.net


import redis
import time
import random
import string
from rediscluster import StrictRedisCluster


while True:
    random_str = ''.join(random.sample(string.ascii_letters + string.digits, 8))
    random_hit = ''.join(random.sample(string.ascii_letters + string.digits, 10))
    
    # no redis cluster
    # r = redis.StrictRedis(host='127.0.0.1',port=8002)

    # redis cluster
    nodes = [{'host':'127.0.0.1','port':'8002'}]
    r = StrictRedisCluster(startup_nodes=nodes)
    r.set(random_str,random_hit)
    time.sleep(0.5)
~~~
