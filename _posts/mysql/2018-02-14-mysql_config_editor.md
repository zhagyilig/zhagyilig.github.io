---
layout: article
title: "mysql_config_editor"
modified:
categories: mysql
toc: true
image:
    teaser: /teaser/mysql01.png
---
> mysql_config_editor — MySQL Configuration Utility

---    
The mysql_config_editor utility enables you to store authentication credentials in an      
encrypted login path file named .mylogin.cnf. The file location is the %APPDATA%\MySQL      
directory on Windows and the current user's home directory on non-Windows systems.        
The file can be read later by MySQL client programs to obtain authentication credentials       
for connecting to MySQL Server.    
   
The unencrypted format of the .mylogin.cnf login path file consists of option groups,    
similar to other option files. Each option group in .mylogin.cnf is called a “login path,”    
which is a group that permits only certain options: host, user, password, port and socket.        

[https://dev.mysql.com/doc/refman/5.7/en/mysql-config-editor.html](https://dev.mysql.com/doc/refman/5.7/en/mysql-config-editor.html)  

### 简单的实现过程  
{% highlight mysql %}
{% raw %}
[root@jdu4e00u53f7 ~]# mysql_config_editor  set -G 0169036 -uroot -p -S /tmp/mysql9036.sock  
Enter password: 
[root@jdu4e00u53f7 ~]#

[root@jdu4e00u53f7 ~]# ll ~/.my
.my.cnf         .mylogin.cnf    .mysql_history  
[root@jdu4e00u53f7 ~]# ll ~/.mylogin.cnf 
-rw------- 1 root root 136 Feb 14 18:26 /root/.mylogin.cnf
[root@jdu4e00u53f7 ~]# file  ~/.mylogin.cnf      
/root/.mylogin.cnf: data
[root@jdu4e00u53f7 ~]# mysql_config_editor print --all 
[0169036]
user = root
password = *****
socket = /tmp/mysql9036.sock

[root@jdu4e00u53f7 ~]# mysql --login-path=0169036 -e "select id from crash.recovery limit 1" 
+----+
| id |
+----+
|  2 |
+----+

[root@jdu4e00u53f7 ~]# mysqladmin --login-path=0169036 pr
+-----+------+-----------+----+---------+------+----------+------------------+
| Id  | User | Host      | db | Command | Time | State    | Info             |
+-----+------+-----------+----+---------+------+----------+------------------+
| 142 | root | localhost |    | Query   | 0    | starting | show processlist |
+-----+------+-----------+----+---------+------+----------+------------------+


[root@jdu4e00u53f7 ~]# mysql --login-path=0169036
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 138
Server version: 5.7.20-log MySQL Community Server (GPL)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

root@localhost:mysql9036.sock [(none)]>select version();
+------------+
| version()  |
+------------+
| 5.7.20-log |
+------------+
1 row in set (0.00 sec)
{% endraw %}
{% endhighlight %}
