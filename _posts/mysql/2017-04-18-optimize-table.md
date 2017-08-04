---
layout: article
title:  "optimize优化mysql"
categories: linux
toc: true
ads: true
	teaser: ![teaser](http://ou529e3sj.bkt.clouddn.com/default.jpg)
---  

> optimize table在优化mysql    

## 原始数据
`1.获取信息`
{% highlight mysql %}
{% raw %}
root@zyl.db.node1 [trans]>  select count(*) as total from ts_error;
+--------+
| total  |
+--------+
| 323778 |
+--------+

root@zyl.db.node1 [trans]> show table status like 'ts_error'\G
*************************** 1. row ***************************
           Name: ts_error
         Engine: InnoDB
        Version: 10
     Row_format: Compact
           Rows: 161882
 Avg_row_length: 16439
    Data_length: 2661285888
Max_data_length: 0
   Index_length: 0
      Data_free: 4194304
 Auto_increment: NULL
    Create_time: 2017-07-20 11:50:25
    Update_time: NULL
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options: 
        Comment: 
1 row in set (0.00 sec)
{% endraw %}
{% endhighlight %}

`2.存放在硬盘中的表文件大小`
{% highlight mysql %}
{% raw %}
12K		ts_error.frm
2.6G	ts_error.ibd
{% endraw %}
{% endhighlight %}

`3.查看一下索引信息`
{% highlight mysql %}
{% raw %}
root@zyl.db.node1 [trans]> show index from ts_error\G
*************************** 1. row ***************************
        Table: ts_error
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 161882
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE
      Comment: 
Index_comment: 
1 row in set (0.00 sec)

索引信息中的列的信息说明:     
Table :表的名称。    
Non_unique:如果索引不能包括重复词，则为0。如果可以，则为1。    
Key_name:索引的名称。    
Seq_in_index:索引中的列序列号，从1开始。    
Column_name:列名称。     
Collation:列以什么方式存储在索引中。在MySQLSHOW INDEX语法中，有值’A’（升序）或NULL（无分类）。    
Cardinality:索引中唯一值的数目的估计值。通过运行ANALYZE TABLE或myisamchk      
-a可以更新。基数根据被存储为整数的统计数据来计数，所以即使对于小型表，该值也没有必要是精确的。    
基数越大，当进行联合时，MySQL使用该索引的机会就越大。    
Sub_part:如果列只是被部分地编入索引，则为被编入索引的字符的数目。如果整列被编入索引，则为NULL。    
Packed:指示关键字如何被压缩。如果没有被压缩，则为NULL。  
Null:如果列含有NULL，则含有YES。如果没有，则为空。  
Index_type：存储索引数据结构方法（BTREE, FULLTEXT, HASH, RTREE）  
{% endraw %}
{% endhighlight %}

## 删除一半数据
{% highlight mysql %}
{% raw %}
root@zyl.db.node1 [trans]> delete from ts_error where id < 161889;  
Query OK, 290255 rows affected, 65535 warnings (1 min 4.84 sec)  

root@zyl.db.node1 [trans]>  select count(*) as total from ts_error;                                                        
+-------+
| total |
+-------+
| 33523 |
+-------+
1 row in set (37.48 sec)  

[root@braindeath2 healthcloud_trans]#   ls |grep ts_error |xargs -i du -sh {}  
12K		ts_error.frm  
2.6G	ts_error.ibd  
{% endraw %}  
{% endhighlight %}  
按常规思想来说，如果在数据库中删除了一半数据后，相对应的.frm,.ibd文件也应当变为之前的一半。
但是删除一半数据后，尽然连1KB都没有减少，这是多么的可怕啊。

我们再来看一看，索引信息: 
{% highlight mysql %}
{% raw %}
root@zyl.db.node1 [trans]> show index from ts_error\G
*************************** 1. row ***************************
        Table: ts_error
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 0
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE
      Comment: 
Index_comment: 
1 row in set (0.01 sec)
{% endraw %}
{% endhighlight %}
对比一下，这次索引查询和上次索引查询，里面的数据信息基本上是上次一次的一本，这点还是合乎常理。  

### 用optimize table来优化一下
{% highlight mysql %}
{% raw %}
root@zyl.db.node1 [trans]> optimize table ts_error;  //删除数据后的优化
+----------------------------+----------+----------+-------------------------------------------------------------------+
| Table                      | Op       | Msg_type | Msg_text                                                          |
+----------------------------+----------+----------+-------------------------------------------------------------------+
| healthcloud_trans.ts_error | optimize | note     | Table does not support optimize, doing recreate + analyze instead |
| healthcloud_trans.ts_error | optimize | status   | OK                                                                |
+----------------------------+----------+----------+-------------------------------------------------------------------+
2 rows in set (8.89 sec)

1.查看一下.frm,.ibd文件的大小
[root@braindeath2 healthcloud_trans]#   ls |grep ts_error |xargs -i du -sh {}
12K	    ts_error.frm
268M	ts_error.ibd

2.删除后的数据
root@zyl.db.node1 [trans]>  show table status like 'ts_error'\G
*************************** 1. row ***************************
           Name: ts_error
         Engine: InnoDB
        Version: 10
     Row_format: Compact
           Rows: 15923
 Avg_row_length: 17385
    Data_length: 276824064
Max_data_length: 0
   Index_length: 0
      Data_free: 0
 Auto_increment: NULL
    Create_time: 2017-07-24 16:30:04
    Update_time: NULL
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options: 
        Comment: 
1 row in set (0.00 sec

3.查看一下索引信息 
root@zyl.db.node1 [trans]> show index from ts_error\G
*************************** 1. row ***************************
        Table: ts_error
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 15923
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE
      Comment: 
Index_comment: 
1 row in set (0.00 sec)
{% endraw %}
{% endhighlight %}

从以上数据我们可以得出，ad_code，ad_code_ind，from_page_url_ind等索引机会差不多都提高了85％，这样效率提高了好多。
 
## 小结   
结合mysql官方网站的信息，个人是这样理解的。当你删除数据时，mysql并不会回收，被已删除数据的占据的存储空间，以及索引位。而是空在那里，而是等待新的数据来弥补这个空缺，这样就有一个缺少，如果一时半会，没有数据来填补这个空缺，那这样就太浪费资源了。所以对于写比较频烦的表，要定期进行optimize，一个月一次，看实际情况而定了。    

举个例子来说吧。有100个php程序员辞职了，但是呢只是人走了，php的职位还在那里，这些职位不会撤销，要等新的php程序来填补这些空位。招一个好的程序员，比较难。我想大部分时间会空在那里。哈哈。      

## 手册中关于OPTIMIZE的一些用法和描述     
OPTIMIZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ...      
如果您已经删除了表的一大部分，或者如果您已经对含有可变长度行的表（含有VARCHAR, BLOB或TEXT列的表）进行了很多更改，则应使用
OPTIMIZE TABLE。被删除的记录被保持在链接清单中，后续的INSERT操作会重新使用旧的记录位置。您可以使用OPTIMIZE TABLE来重新  
利用未使用的空间，并整理数据文件的碎片。     
在多数的设置中，您根本不需要运行OPTIMIZE TABLE。即使您对可变长度的行进行了大量的更新，您也不需要经常运行，每周一次或每月一次
即可，只对特定的表运行。    
OPTIMIZE TABLE只对MyISAM, BDB和InnoDB表起作用。      
注意，在OPTIMIZE TABLE运行过程中，MySQL会锁定表。     

[阅读原文](http://blog.51yip.com/mysql/1222.html)






