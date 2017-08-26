---
layout: article
title:  "Handler_read_*参数介绍"
categories: mysql
toc: true
ads: true
image:
    teaser: /teaser/mysql01.png
---  

> Handler_read_*，它们显示了数据库处理SELECT查询语句的状态，对于调试SQL语句有很大意义，本文简单介绍下。

## 参数介绍

{% highlight mysql %}
{% raw %}
root@izyl-mysql5.6 [none]-> show global status  like '%handler_read%';
+----------------------------+---------------+
| Variable_name              | Value         |
+----------------------------+---------------+
| Handler_read_first         | 172238768     |
| Handler_read_key           | 44927404888   |
| Handler_read_last          | 38666         |
| Handler_read_next          | 121083383847  |
| Handler_read_prev          | 1628279944    |
| Handler_read_rnd           | 8573509625    |
| Handler_read_rnd_next      | 2168262576114 |
+----------------------------+---------------+
18 rows in set (0.08 sec)
{% endraw %}
{% endhighlight %}
　　
分析这几个值，我们可以查看当前索引的使用情况：      
`Handler_read_first`    
1.索引中第一条被读的次数。如果较高，它表示服务器正执行大量全索引扫描；例如，SELECT col1 FROM foo，假定col1有索引（这个值越低越好）。      
2.表明SQL是在做一个全索引扫描，注意是全部，而不是部分，所以说如果存在WHERE语句，这个选项是不会变的。如果这个选项的数值很大，既是好事   也是坏事。说它好是因为毕竟查询是在索引里完成的，而不是数据文件里，说它坏是因为大数据量时，简便是索引文件，做一次完整的扫描也是很费时的。  

`Handler_read_key`    
如果索引正在工作，这个值代表一个行被索引值读的次数，如果值越低，表示索引得到的性能改善不高，因为索引不经常使用（这个值越高越好）。    


`Handler_read_next`    
1.按照键顺序读下一行的请求数。如果你用范围约束或如果执行索引扫描来查询索引列，该值增加。     
2.表明在进行索引扫描时，按照索引从数据文件里取数据的次数。   

`Handler_read_prev`    
此选项表明在进行索引扫描时，按照索引倒序从数据文件里取数据的次数，。该读方法主要用于优化ORDER BY ... DESC。    

`Handler_read_rnd`    
1.根据固定位置读一行的请求数。如果你正执行大量查询并需要对结果进行排序该值较高。你可能使用了大量需要MySQL扫描整个表的查询或你的连接没有正确使用键。这个值较高，意味着运行效率低，应该建立索引来补救。    
2.简单的说，就是查询直接操作了数据文件，很多时候表现为没有使用索引或者文件排序。    
  
`Handler_read_rnd_next`    
1.在数据文件中读下一行的请求数。如果你正进行大量的表扫描，该值较高。通常说明你的表索引不正确或写入的查询没有利用索引。    
2.此选项表明在进行数据文件扫描时，从数据文件里取数据的次数。    
  
计算表扫描率：       
表扫描率 ＝ Handler_read_rnd_next / Com_select       
如果表扫描率超过4000，说明进行了太多表扫描，很有可能索引没有建好，增加read_buffer_size值会有一些好处，但最好不要超过8MB     

 
 

