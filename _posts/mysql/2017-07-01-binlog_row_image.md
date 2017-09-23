---
layout: article
title:  "binlog_row_image参数测试"
categories: mysql
toc: true
ads: true
---  
> 官方:https://dev.mysql.com/doc/refman/5.6/en/replication-options-binary-log.html     
---  
{% highlight mysql %}
{% raw %}
binlog_format=ROW 格式记录的是每一行的变更，会带来磁盘IO上的开销，同时由于binlog日志变大，网络开销也变大。那么在MySQL 5.6以后binlog的格式默认就是ROW了，同时引入了新的参数binlog_row_image，这个参数默认值是FULL，其还有两个值是minimal,noblob;FULL记录每一行的变更，minimal只记录影响后的行,noblob记录所有列，不需要的BLOB和TEXT列。下面通过简单的一下例子理解下参数。 
{% endraw %}
{% endhighlight %}  
## 测试数据
{% highlight mysql %}
{% raw %}
CREATE TABLE Products (
  Code INTEGER,
  Name VARCHAR(255) NOT NULL ,
  Price DECIMAL NOT NULL ,
  Manufacturer INTEGER NOT NULL,
  PRIMARY KEY (Code)
) ENGINE=INNODB;

INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(1,'Hard drive',240,5);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(2,'Memory',120,6);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(3,'ZIP drive',150,4);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(4,'Floppy disk',5,6);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(5,'Monitor',240,1);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(6,'DVD drive',180,2);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(7,'CD drive',90,2);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(8,'Printer',270,3);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(9,'Toner cartridge',66,3);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(10,'DVD burner',180,2);

root@3306 [xtrdb]>select * from Products;
+------+-----------------+-------+--------------+
| Code | Name            | Price | Manufacturer |
+------+-----------------+-------+--------------+
|    1 | Hard drive      |   240 |            5 |
|    2 | Memory          |   120 |            6 |
|    3 | ZIP drive       |   150 |            4 |
|    4 | Floppy disk     |     5 |            6 |
|    5 | Monitor         |   240 |            1 |
|    6 | DVD drive       |   180 |            2 |
|    7 | CD drive        |    90 |            2 |
|    8 | Printer         |   270 |            3 |
|    9 | Toner cartridge |    66 |            3 |
|   10 | DVD burner      |   180 |            2 |
+------+-----------------+-------+--------------+
10 rows in set (0.01 sec)
{% endraw %}
{% endhighlight %}

## FULL
{% highlight mysql %}
{% raw %}
mysqladmin -p -S /tmp/mysql3306.sock flush-logs

root@3306 [(none)]>show variables like 'binlog_row_image%';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| binlog_row_image | FULL  |
+------------------+-------+
root@3306 [xtrdb]>update Products set Name='Image' where Name = 'Printer';
root@3306 [xtrdb]>select * from Products;
+------+-----------------+-------+--------------+
| Code | Name            | Price | Manufacturer |
+------+-----------------+-------+--------------+
|    1 | Hard drive      |   240 |            5 |
|    2 | Memory          |   120 |            6 |
|    3 | ZIP drive       |   150 |            4 |
|    4 | Floppy disk     |     5 |            6 |
|    5 | Monitor         |   240 |            1 |
|    6 | DVD drive       |   180 |            2 |
|    7 | CD drive        |    90 |            2 |
|    8 | Image           |   270 |            3 |
|    9 | Toner cartridge |    66 |            3 |
|   10 | DVD burner      |   180 |            2 |
+------+-----------------+-------+--------------+
10 rows in set (0.00 sec)

BEGIN
/*!*/;
# at 332
#170915  6:24:09 server id 1373306  end_log_pos 391 CRC32 0x9a606250 	Table_map: `xtrdb`.`products` mapped to number 391
# at 391
#170915  6:24:09 server id 1373306  end_log_pos 471 CRC32 0xcf9bfb57 	Update_rows: table id 391 flags: STMT_END_F
### UPDATE `xtrdb`.`products`
### WHERE
###   @1=8 /* INT meta=0 nullable=0 is_null=0 */
###   @2='Printer' /* VARSTRING(765) meta=765 nullable=0 is_null=0 */
###   @3=270 /* DECIMAL(10,0) meta=2560 nullable=0 is_null=0 */
###   @4=3 /* INT meta=0 nullable=0 is_null=0 */
### SET
###   @1=8 /* INT meta=0 nullable=0 is_null=0 */
###   @2='Image' /* VARSTRING(765) meta=765 nullable=0 is_null=0 */
###   @3=270 /* DECIMAL(10,0) meta=2560 nullable=0 is_null=0 */
###   @4=3 /* INT meta=0 nullable=0 is_null=0 */
# at 471
#170915  6:24:09 server id 1373306  end_log_pos 502 CRC32 0x70adb8e0 	Xid = 6500
COMMIT/*!*/;
{% endraw %}
{% endhighlight %}

## MINIMAL  
{% highlight mysql %}
{% raw %}
root@3306 [xtrdb]>set  binlog_row_image = minimal;
Query OK, 0 rows affected (0.00 sec)

root@3306 [xtrdb]>show variables like 'binlog_row_image';
+------------------+---------+
| Variable_name    | Value   |
+------------------+---------+
| binlog_row_image | MINIMAL |
+------------------+---------+
1 row in set (0.01 sec)

root@3306 [xtrdb]>update Products set Name='Image02' where Name='Memory';
Query OK, 1 row affected (0.40 sec)
Rows matched: 1  Changed: 1  Warnings: 0

root@3306 [xtrdb]>select * from Products;
+------+-----------------+-------+--------------+
| Code | Name            | Price | Manufacturer |
+------+-----------------+-------+--------------+
|    1 | Hard drive      |   240 |            5 |
|    2 | Image02         |   120 |            6 |
|    3 | ZIP drive       |   150 |            4 |
|    4 | Floppy disk     |     5 |            6 |
|    5 | Monitor         |   240 |            1 |
|    6 | DVD drive       |   180 |            2 |
|    7 | CD drive        |    90 |            2 |
|    8 | Image           |   270 |            3 |
|    9 | Toner cartridge |    66 |            3 |
|   10 | DVD burner      |   180 |            2 |
+------+-----------------+-------+--------------+
10 rows in set (0.00 sec)

BEGIN
/*!*/;
# at 640
#170915  6:26:41 server id 1373306  end_log_pos 699 CRC32 0xfe9d84ca 	Table_map: `xtrdb`.`products` mapped to number 391
# at 699
#170915  6:26:41 server id 1373306  end_log_pos 750 CRC32 0x3ba96f37 	Update_rows: table id 391 flags: STMT_END_F
### UPDATE `xtrdb`.`products`
### WHERE
###   @1=2 /* INT meta=0 nullable=0 is_null=0 */
### SET
###   @2='Image02' /* VARSTRING(765) meta=765 nullable=0 is_null=0 */
# at 750
#170915  6:26:41 server id 1373306  end_log_pos 781 CRC32 0x31554560 	Xid = 6506
COMMIT/*!*/;
{% endraw %}
{% endhighlight %}

## NOBLOB
{% highlight mysql %}
{% raw %}
root@3306 [xtrdb]>set  binlog_row_image = noblob;
Query OK, 0 rows affected (0.00 sec)

root@3306 [xtrdb]>show variables like 'binlog_row_image';
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| binlog_row_image | NOBLOB |
+------------------+--------+
1 row in set (0.01 sec)

root@3306 [xtrdb]>update Products set Name='Image03' where Name='DVD drive';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

root@3306 [xtrdb]>select * from Products;
+------+-----------------+-------+--------------+
| Code | Name            | Price | Manufacturer |
+------+-----------------+-------+--------------+
|    1 | Hard drive      |   240 |            5 |
|    2 | Image02         |   120 |            6 |
|    3 | ZIP drive       |   150 |            4 |
|    4 | Floppy disk     |     5 |            6 |
|    5 | Monitor         |   240 |            1 |
|    6 | Image03         |   180 |            2 |
|    7 | CD drive        |    90 |            2 |
|    8 | Image           |   270 |            3 |
|    9 | Toner cartridge |    66 |            3 |
|   10 | DVD burner      |   180 |            2 |
+------+-----------------+-------+--------------+
10 rows in set (0.00 sec)
BEGIN
/*!*/;
# at 919
#170915  6:28:33 server id 1373306  end_log_pos 978 CRC32 0x02b74bc8 	Table_map: `xtrdb`.`products` mapped to number 391
# at 978
#170915  6:28:33 server id 1373306  end_log_pos 1062 CRC32 0x0777b55d 	Update_rows: table id 391 flags: STMT_END_F
### UPDATE `xtrdb`.`products`
### WHERE
###   @1=6 /* INT meta=0 nullable=0 is_null=0 */
###   @2='DVD drive' /* VARSTRING(765) meta=765 nullable=0 is_null=0 */
###   @3=180 /* DECIMAL(10,0) meta=2560 nullable=0 is_null=0 */
###   @4=2 /* INT meta=0 nullable=0 is_null=0 */
### SET
###   @1=6 /* INT meta=0 nullable=0 is_null=0 */
###   @2='Image03' /* VARSTRING(765) meta=765 nullable=0 is_null=0 */
###   @3=180 /* DECIMAL(10,0) meta=2560 nullable=0 is_null=0 */
###   @4=2 /* INT meta=0 nullable=0 is_null=0 */
# at 1062
#170915  6:28:33 server id 1373306  end_log_pos 1093 CRC32 0x0a8d090f 	Xid = 6511
COMMIT/*!*/;
{% endraw %}
{% endhighlight %}

注意:     
NDB Cluster不支持此变量; 设置它对日志记录没有影响 NDB。（错误＃16316828）   
