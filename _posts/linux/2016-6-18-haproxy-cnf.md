---
layout: article
title:  "haproxy安装到启动"
categories: linux
toc: true
ads: true
    teaser: /teaser/haproxy.jpg
---  

> haproxy基础,pxc集群高可用

## 安装proxy  
{% highlight mysql %}
{% raw %}
#!/bin/sh
###################################################
#Author: Created by zhagyilig 2016/6
#Blog:www.xtrdb.net
#Function: This scripts function is install haproxy
#Version:4.1.2
###################################################
#Source function library.
. /etc/init.d/functions

TOOLS=/home/zhangyiling/tools/haproxy
BACKAGE01=$TOOLS/haproxy-1.4.24.tar.gz
BACKAGE02=$TOOLS/haproxy_config.tar.gz

#Require root to run this script.
uid=`id|awk -F "[=(]+" '{print $1}'`
if [ $uid -ne 0 ];then
  action "You need to be root to perform the script." /bin/false
  exit 1
fi
#check $TOOLS.
[ -d $TOOLS ] || {
	mkdir -p $TOOLS
}
#start install.
LANG=en
yum install -y gcc gcc-devel
[ -f /etc/init.d/httpd ] && {
	/etc/init.d/httpd stop
} 
if [ -f $BACKAGE01 ]; then
	tar xf $BACKAGE01 -C $TOOLS
else
	echo "pls check install backage."
fi
cd $TOOLS/haproxy-1.4.24
make TARGET=linux26 ARCH=x86_64
make PREFIX=/application/haproxy-1.4.24 install
ln -s  /application/haproxy-1.4.24/ /application/haproxy
#The kernel forwarding.
sed  -i 's#net.ipv4.ip_forward = 0#net.ipv4.ip_forward = 1#g' /etc/sysctl.conf
#create user.
useradd haproxy -s /sbin/nologin -M
#configure LOG
echo -ne "#Haproxy\nlocal0.* /application/haproxy-1.4.24/logs/haproxy.log" >>/etc/rsyslog.conf 
#configure conf.
tar xf $BACKAGE02 -C /
/etc/init.d/rsyslog restart
/application/haproxy/bin/haproxyd restart
{% endraw %}
{% endhighlight %}

## 配置文件详解
{% highlight mysql %}
{% raw %}
global
        log 127.0.0.1 local0 info        #日志格式
        chroot /var/haproxy              #chroot运行的路径  
        group haproxy                    #所属运行的用户
        user haproxy                     #所属运行的用户  
        daemon                           #以后台进程形式运行haproxy
        nbproc 8                         #进程数量(可以设置多个进程提高性能)
        pidfile /var/run/haproxy         #PID文件所在位置
 
 
        #####################默认的全局设置######################
        ##这些参数可以被利用配置到frontend,backend,listen组件##
defaults
        log global                       #应用全局日志格式
        maxconn 20480                    #默认最大连接数  
        option httplog                   #日志类别http日志格式
        option httpclose                 #每次请求完毕后主动关闭http通道
        option dontlognull               #不记录健康检查的日志信息  
        option forwardfor                #如果后端服务器需要获得客户端真实ip需要配置的参数，可以从Http Header中获得客户端ip 
        option redispatch                #serverId对应的服务器挂掉后,强制定向到其他健康的服务器 
        option abortonclose              #当服务器负载很高的时候，自动结束掉当前队列处理比较久的连接  
   
        retries 3                        #3次连接失败就认为服务不可用，也可以通过后面设置  
        balance roundrobin               #默认的负载均衡的方式,轮询方式
       #balance source                   #默认的负载均衡的方式,类似nginx的ip_hash
       #balance leastconn                #默认的负载均衡的方式,最小连接
        contimeout 5000                  #连接超时  
        clitimeout 50000                 #客户端超时  
        srvtimeout 50000                 #服务器超时
        timeout check 2000               #心跳检测超时  
 
        ####################监控页面的设置#######################
listen admin_status                     #Frontend和Backend的组合体,监控组的名称，按需自定义名称
         bind *
         mode http                      #http的7层模式
         stats refresh 5s               #每隔5秒自动刷新监控页面
         stats uri /admin?status        #监控页面的url
         stats realm haproxty\ haproxy  #监控页面的提示信息  
         stats auth admin:admin         #监控页面的用户和密码admin,可以设置多个用户名  
         stats auth admin1:admin1       #监控页面的用户和密码admin1
         stats hide-version             #隐藏统计页面上的HAproxy版本信息 
         stats admin if TRUE            #手工启用/禁用,后端服务器(haproxy-1       
listen  webserver
     bind  172.16.254.30:80             #客户端访问的外网IP，若和heartbeat结合使用的话即VIP
     mode http                          #模式为http
option httpchk HEAD /checkstatus.html HTTP/1.0  #url健康检查
balance roundrobin    #调度算法为轮训
server web1 192.168.254.10:80 check
server web2 192.168.254.20:80 check
{% endraw %}
{% endhighlight %}
 
## 启动脚本  
{% highlight mysql %}
{% raw %}
#!/bin/sh
###################################################
#Author: Created by zhagyilig 2016/6
#Blog:www.xtrdb.net
#Function: This scripts function is haproxy {start|stop}
#Version:4.1.2
###################################################
BASE=/application/haproxy
PROG=$BASE/sbin/haproxy
PIDFILE=$BASE/var/run/haproxy.pid
CONFFILE=$BASE/conf/haproxy.conf
case $1 in
    start)
        $PROG -f $CONFFILE -D
    ;;
    status)
        if [ ! -f $PIDFILE ]; then
            echo "haproxy is no running."
            exit 1
        fi
        for pid in $(cat $PIDFILE)
        do
            kill -0 $pid
            RETVAL="$?"
			            RETVAL="$?"
            if [ ! $RETVAL -eq 0 ]; then
                echo "process $pid died."
                exit 1
            fi
        done
        echo "haproxy is running."
    ;;
    restart)
        $PROG -f $CONFFILE -sf $(cat $PIDFILE)
    ;;
    stop)
        kill $(cat $PIDFILE)
    ;;
    check)
        $PROG -f $CONFFILE -c
    ;;
    *)
        echo "Usage: $0 {start|stop|restart|check|status}"
    ;;
esac
{% endraw %}
{% endhighlight %}  

