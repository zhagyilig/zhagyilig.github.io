---
layout: article
title: "expalin 使用"
modified:
categories: mysql
image:
    teaser: /teaser/tool.jpg
date: 2016-03-11 12:10:08
---

> 为什么要使用 explain？   

explain 可以帮助我们分析 select 语句，让我们知道查询效率低下的原因，从而改
进我们查询，让查询优化器能够更好的工作，但是哪些关键信息值得注意呢？
  
MySQL 查询优化器是如何工作的?
MySQL 查询优化器有几个目标，但是其中最主要的目标是尽可能地使用索引，并且使
用最严格的索引来消除尽可能多的数据行。最终目标是提交 SELECT 语句查找数
行，而不是排除数据行。优化器试图排除数据行的原因在于它排除数据行的速度越快，
那么找到与条件匹配的数据行也就越快。如果能够首先进行最严格的测试，查询就可以执行地更快。
### explain使用详解
{% highlight mysql %}
{% raw %}
root@zyl 06:43:08 [sakila]->explain select staff_id from payment where staff_id limit 10\G
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    210
Current database: sakila

*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: payment
         type: index
possible_keys: NULL
          key: idx_fk_staff_id
      key_len: 1
          ref: NULL
         rows: 16451
        Extra: Using where; Using index
1 row in set (0.00 sec)
{% endraw %}
{% endhighlight %}
---
{% highlight mysql %}
{% raw %}
1.id  MySQL Query Optimizer 选定的执行计划中查询的序列号。表示查询中执行 select 子句或操作表的顺序，id 值越大优先级越高，越先被执行。id 相同，执行顺序由上至下。 

2.select_type  			查询类型 
   SIMPLE   			简单的 select 查询，不使用表连接及子查询 
   PRIMARY  			主查询，即最外层的 select 查询 
   UNION    			UNION 中的第二个或随后的 select 查询，不依赖于外部查询的结果集 
   DEPENDENT UNION  	UNION 中的第二个或随后的 select 查询，依赖于外部查询的结果集
   UNION RESULT  		UNION 查询的结果集 
   SUBQUERY  			子查询中的第一个 select 查询，不依赖于外部查询的结果集 
   DEPENDENT SUBQUERY  	子查询中的第一个 select 查询，依赖于外部查询的结果集 
   DERIVED  			用于 from 子句里有子查询的情况。MySQL 会递归执行这些子查询，把结果放在临时表里。 
   UNCACHEABLE SUBQUERY 结果集不能被缓存的子查询，必须重新为外层查询的每一行进行评估 
   UNCACHEABLE UNION  	UNION 中的第二个或随后的 select 查询，属于不可缓存的子查询 

3.table 				输出行所引用的表 

4.type 					重要的项，显示连接使用的类型或者叫访问类型，按最差到优的类型排序:
   ALL	 				执行full table scan，这是最差的一种方式    
   index				执行full index scan，遍历整个索引来查询匹配的行，并且可以通过索引完成结果扫描并且直接从索引中取的想要的结果数据，也就是可以避免回表，比ALL略好，因为索引文件通常比全部数据要来的小	  	

   range				利用索引进行范围查询，比index略好，常用于<、>、<=、>=、between等
   
   index_subquery		子查询中可以用到索引  
   
   unique_subquery		子查询中可以用到唯一索引，效率比 index_subquery 更高些
   
   index_merge	    	可以利用index merge特性用到多个索引，提高查询效率
   
   ref_or_null  		表连接类型是ref，但进行扫描的索引列中可能包含NULL值
   
   fulltext	   			全文检索
   
   ref  				使用非唯一索引或唯一索引的前缀扫描，返回匹配某个单独值得记录，基于索引的等值查询，或者表间等值连接
   
   eq_ref				表连接时基于主键或非NULL的唯一索引完成扫描，比ref略好，简单的说,是多表连接中使用primary key 或者 unique  index作为关联条件
   
   const				基于主键或唯一索引唯一值查询，最多返回一条结果，比eq_ref略好
   
   system				查询对象表只有一行数据，这是最好的情况    

5.possible_keys  		指出 MySQL 能在该表中使用哪些索引有助于查询。如果为空，说明没有可用的索引 

6.key 					 MySQL实际从 possible_key 选择使用的索引。如果为 NULL，则没有使用索引。
很少的情况下，MYSQL 会选择优化不足的索引。这种情况下，可以在 SELECT 语句中使用 
USE  INDEX（indexname）来强制使用一个索引或者用IGNORE  INDEX（indexname）来强制 
MYSQL忽略索引 

7.key_len  				使用的索引的长度。在不损失精确性的情况下，长度越短越好。  
    key_len 用于表示本次查询中，所选择的索引长度有多少字节，通常我们可借此判断联合索引有多少列被选择了。
   在这里 key_len 大小的计算规则是：
   
   一般地，key_len 等于索引列类型字节长度，例如int类型为4-bytes，bigint为8-bytes；
   如果是字符串类型，还需要同时考虑字符集因素，例如：CHAR(30) UTF8则key_len至少是90-bytes；
   若该列类型定义时允许NULL，其key_len还需要再加 1-bytes；
   若该列类型为变长类型，例如 VARCHAR（TEXT\BLOB不允许整列创建索引，如果创建部分索引，也被视为动态列类型），其key_len还需要再加 2-bytes;
   综上，看下面几个例子：
   
   id int	key_len = 4+1 = 5							允许NULL，加1-byte
   id int not null	key_len = 4							不允许NULL
   user char(30) utf8	key_len = 30*3+1				允许NULL
   user varchar(30) not null utf8	key_len = 30*3+2	动态列类型，加2-bytes
   user varchar(30) utf8	key_len = 30*3+2+1			动态列类型，加2-bytes；允许NULL，再加1-byte
   detail text(10) utf8	key_len = 30*3+2+1				TEXT列截取部分，被视为动态列类型，加2-bytes；且允许NULL
   
   备注，key_len 只指示了WHERE中用于条件过滤时被选中的索引列，是不包含 ORDER BY/GROUP BY 这部分被选中的索引列。
   例如，有个联合索引 idx1(c1, c2, c3)，3个列均是INT NOT NULL，那么下面的这个SQL执行计划中，key_len的值是8而不是12：
   SELECT…WHERE c1=? AND c2=? ORDER BY c1;  
   
8.ref  显示索引的哪一列被使用了

9.rows 预计需要扫描的记录数，预计需要扫描的记录数越小越好

10.Extra 				额外附加信息，主要确认是否出现 Using filesort、Using temporary 这两种情况
   Using filesort		将用外部排序而不是按照索引顺序排列结果，数据较少时从内存排序，否则需要在磁盘完成排序，代价非常高，需要添加合适的索引  
   
   Using temporary		需要创建一个临时表来存储结果，这通常发生在对没有索引的列进行GROUP BY时，或者ORDER BY里的列不都在索引里，需要添加合适的索引  
   
   Using index			表示MySQL使用覆盖索引避免全表扫描，不需要再到表中进行二次查找数据，这是比较好的结果之一。注意不要和type中的index类型混淆  
   
   Using where			通常是进行了全表引扫描后再用WHERE子句完成结果过滤，需要添加合适的索引  
   
   Impossible WHERE		对Where子句判断的结果总是false而不能选择任何数据，例如where 1=0，无需过多关注  
   
   Select tables optimized away	使用某些聚合函数来访问存在索引的某个字段时，优
   化器会通过索引直接一次定位到所需要的数据行完成整个查询，例如MIN()\MAX()，这种也是
   比较好的结果之一  
   
   再说下，5.6开始支持optimizer  trace功能，看样子在执行计划方面是要逐渐和ORACLE看齐 ：）
{% endraw %}
{% endhighlight %}

**总的来说，我们只需要关注结果中的几列：**
type  
key   
key_len   
rows   
Extra  
