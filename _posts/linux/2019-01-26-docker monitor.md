---
layout: article
title:  "cAdvisor"
categories: linux
toc: true
ads: true
image:
    teaser: /linux/cadvisor.jpg
---

> 由 Google 公司开发的用来监控容器的工具

### 一、问题  
在生产环境中运行着很多容器，我们需要像多个进程运行在宿主机上一样能够监控他们的资源利用和性能。 

### 二、解决方案
- 使用`cAdvisor`， 不仅可以搜集一台机器上所有运行的容器信息，还提供基础查询界面和 http 接口，方便其他组件如 Prometheus 进行数据抓取;  
- `cAdvisor` 可以对节点机器上的资源及容器进行实时监控和性能数据采集，包括 CPU 使用情况、内存使用情况、网络吞吐量及文件系统使用情况;
- 使用 cAdvisor + Influxdb + Grafana 三大开源框架组成的 docker 服务群进行监控。

### 三、架构图  
![cadvisor1]({{ site.url }}/images/linux/cadvisor1.png)

### 四、部署  

1. 为了网络隔离，建虚拟网卡    
{% highlight bash %}
{% raw %} 
[root@p-asg-docker-6 ~]# docker network create docker-monitor    
427480e9c7b883f37cecb0a2d34f382682a2ba1e67fd174a33a18947bdc87586     
{% endraw %}
{% endhighlight %} 

2. Influxdb  
{% highlight bash %}
{% raw %}   
docker run -d --name influxdb --net docker-monitor -p 8083:8083 -p 8086:8086 tutum/influxdb  

浏览器: http://ip:8086  

创建管理员角色 root 密码 root 供使用:  
CREATE USER "root" WITH PASSWORD 'root' WITH ALL PRIVILEGES  
创建管理员角色 root 密码 root 供使用:   
CREATE DATABASE "cadvisor"   
{% endraw %}
{% endhighlight %}  

3. 部署 cAdvisor
{% highlight bash %}
{% raw %}  
docker run --privileged=true --net docker-monitor --volume=/:/rootfs:ro --volume=/var/run:/var/run:rw --volume=/sys:/sys:ro --volume=/data/docker:/data/docker:ro --volume=/sys/fs/cgroup:/sys/fs/cgroup:ro -p 8087:8080 --detach=true --name=cadvisor google/cadvisor -storage_driver=influxdb -storage_driver_db=cadvisor -storage_driver_host=influxdb:8086  

浏览器: http://ip:8087
{% endraw %}
{% endhighlight %}    


3. 部署 Grafana  
{% highlight bash %}
{% raw %}   
docker run -d --name grafana --net docker-monitor -p 3000:3000  grafana/grafana

浏览器: http://ip:3000
之后就是设置数据源，添加面板。
{% endraw %}
{% endhighlight %}   

### 五、效果    
![cadvisor2]({{ site.url }}/images/linux/cadvisor2.jpeg)   

### 项目地址   
项目主页：(http://github.com/google/cadvisor](http://github.com/google/cadvisor)



 





