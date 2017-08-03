---
layout: article
title:  "Handler_read_*参数介绍"
categories: mysql
toc: true
ads: true
#    teaser: /teaser/default.jpg
---  

> sort_buffer_size 和 max_length_for_sort_data  

这段时间mysql数据库的性能明显降低,`iowait`达到了30, 响应时间明显变长。通过`show processlist` 查看,发现有很多session在处理sort操作, 增大`sort_buffer_size`，但是效果也不大, 通过查看监控,也没发现有硬盘排序,怀疑是sort导致性能下降,固让开发修改程序, sort由程序来处理。星期五发布后,今天发现压力固然好了很多。    

因此基本上能确定是sort引起的问题. 今天仔细分析问题,查看mysql的参数时,看到一个叫做`max_length_for_sort_data`的参数, 值是1024 仔细查看 mysql 的 filesort 算法时, 发现mysql的filesort有两个方法,MySQL 4.1之前是使用方法A, 之后版本会使用改进的算法B, 但使用方法B的前提是列长度的值小于max_length_for_sort_data, 但我们系统中的列的长度的值会大于1024. 因此也就是说在sort的时候, 是在使用方法A, 而方法A的性能比较差, 也就解释了我们的mysql系统在有sort时,性能差,去掉之后性能马上提高很多的原因.    
  
马上修改`max_length_for_sort_data`这个值，增大到8096, 果然性能就提高了.      

总结:
mysql对于排序,使用了两个变量来控制`sort_buffer_size`和`max_length_for_sort_data`,不像oracle使用SGA控制.     这种方式的缺点是要单独控制,容易出现排序性能问题.  

对于filesort的两个方法介绍,以及优化方式,见:    
[https://dev.mysql.com/doc/refman/5.7/en/order-by-optimization.html](https://dev.mysql.com/doc/refman/5.7/en/order-by-optimization.html)  
{% highlight mysql %}
{% raw %}
The tuples used by the modified filesort algorithm are longer than the pairs used by the original algorithm, and fewer of them fit in the sort buffer. As a result, it is possible for the extra I/O to make the modified approach slower, not faster. To avoid a slowdown, the optimizer uses the modified algorithm only if the total size of the extra columns in the sort tuple does not exceed the value of the max_length_for_sort_data system variable. (A symptom of setting the value of this variable too high is a combination of high disk activity and low CPU activity.)
{% endraw %}
{% endhighlight %}
　　
  

 
 

