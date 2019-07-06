---
layout: article
title:  "Django HttpRequest"
categories: python
toc: true
ads: true
image:
    teaser: /python/django.jpeg

---
   
> HttpRequest对象

### HttpRequest  
**由django创建**    
1. 属性：    
{% highlight bash %}
{% raw %}
 HttpRequest.body HttpRequest.path
 HttpRequest.method
 HttpRequest.encoding
 HttpRequest.GET
 HttpRequest.POST HttpRequest.META
{% endraw %}
{% endhighlight %}  

2. 方法   
{% highlight bash %}
{% raw %}
 HttpRequest.get_full_path()
 HttpRequest.is_secure()
 HttpRequest.is_ajax()
{% endraw %}
{% endhighlight %} 

### HttpResponse
{% highlight bash %}
{% raw %}
JsonResponse(data, encoder=DjangoJSONEncoder, safe=True, json_dumps_params=None, **kwargs)

 >>> response = JsonResponse({'foo': 'bar'})
 >>> response.content
 b'{"foo": "bar"}'
 
>>> response = JsonResponse([1, 2, 3], safe=False)
{% endraw %}
{% endhighlight %} 

### 加载模版
1. 
`django.template.loader` 这模块提供两种方法加载模版
{% highlight bash %}
{% raw %}
get_template(template_name, using=None)
{% endraw %}
{% endhighlight %} 

加载指定模版并返回Template对象
{% highlight bash %}
{% raw %}
select_template(template_name_list, using=None)
{% endraw %}
{% endhighlight %}  

它与get_template类似，它尝试每个名称并返回第一个存在的模版
从文件加载内容
{% highlight bash %}
{% raw %}
from django,template import Context, loader
 def index(request):
 t = loader.get_template("test.html")
 context = {"name": "hello reboot !!!"}
 return HttpResponse(t.render(context, request))
{% endraw %}
{% endhighlight %} 

2. 快捷的方式
{% highlight bash %}
{% raw %}
from django.shortcuts import render
 def index(request):
 context = {'name': "reboot"}
 return render(request, 'test.html', context)
{% endraw %}
{% endhighlight %}