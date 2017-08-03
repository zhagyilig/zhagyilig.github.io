---
layout: article
title: "多种日志轻松定位硬件故障"
modified:
categories: linux
image:
    teaser: /teaser/hard.jpg
date: 2017-06-29 T22:49:00-07:00
---

>某台机器上message日志数量突然暴增，简单查看了下有内存相关的报错,那么问题来了。  

#### Message日志  
进入服务器查看message日志，先看看同事说的告警到底是什么，如下图：  
![](http://i.imgur.com/2MmxrRa.jpg)  

还真是，通道3，第一个槽位的内存发生故障了。但是，我只知道A1/B1/A2/B2，所以我还是继续。  

### Ipmitool工具
不论怎样
Ipmitool工具查看了下，确实是有内存告警，如下图： 
![](http://i.imgur.com/Yt56p5G.jpg)  

虽然告警，可是无法定位大具体哪根内存坏了呀

### IDRAC-web
不论怎样
我们还有DELL自带的IDRAC的web页面可以查看硬件状态，登陆看看，先看看日志，这里有了吧，B6内存槽故障
![](http://i.imgur.com/VpCXXZD.jpg)

再看看硬件状态，B6内存存在告警
![](http://i.imgur.com/LE2MsnC.jpg)

就此，我找到了我想要的信息，定位到了B6内存故障，需要更换，至于如何更换，需要注意哪些事项，以后再说。

## 总结
硬件安全是服务器最底层的安全，一定要做好各项硬件监控，及时处理硬件故障，否则，你们懂的。介绍接种常见的日志  
1、messages日志  
2、dmesg日志  
3、ipmitool sel list查看硬件日志    
4、远程管理页面上的日志（DELL的IDRAC，HP的ILO，IBM的IMM等等）  
5、smart日志      

[**阅读原文·叶老师**](https://mp.weixin.qq.com/s/R5azA-ZTs1va3Zsjzm_lxQ)