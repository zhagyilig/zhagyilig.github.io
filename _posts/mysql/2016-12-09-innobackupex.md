---
layout: article
title: "Innobackupex远程限流备份mysql数据库"
modified:
categories: mysql
image:
    teaser: /teaser/backup0.jp
---

> 需要将A机器上数据库备份到B机器，需要在B机器上开启一个端口利用nc和pv达到限速的目的。  

**B机器：**
{% highlight mysql %}
{% raw %}
[root@nginx innobackup]#  ifconfig   |sed -nr 's#^.*addr:(.*) Bcast.*$#\1#gp'
192.168.21.139

[root@nginx innobackup]#  mkdir /data/innobackup
[root@nginx innobackup]#  nc -l 888 | pv -q -L 50M | tar xi
{% endraw %}
{% endhighlight %}
---
**A机器：**
{% highlight mysql %}
{% raw %}
[root@mysql back]#  cat /etc/redhat-release 
CentOS release 6.5 (Final)
[root@mysql back]#  hostname 
mysql.ns1.zhagyilig.com
[root@mysql back]#  innobackupex  --version
innobackupex version 2.4.7 Linux (x86_64) (revision id: 6f7a799)
[root@mysql back]# mkdir /tmp/back


[root@mysql scripts]#  innobackupex  --defaults-file=/etc/my.cnf --user=root --password=888888 --host=127.0.0.1 --port=3306 --stream=tar --tmpdir=/tmp/back   --slave-info  /data/innobackup | nc -nvv 192.168.21.139 888

Connection to 192.168.21.139 888 port [tcp/*] succeeded!
161209 23:31:35 innobackupex: Starting the backup operation

......

IMPORTANT: Please check that the backup run completes successfully.
           At the end of a successful backup run innobackupex
           prints "completed OK!".
		   xtrabackup: Transaction log of lsn (9568482) to (9568482) was copied.
161209 23:31:47 completed OK!
{% endraw %}
{% endhighlight %}