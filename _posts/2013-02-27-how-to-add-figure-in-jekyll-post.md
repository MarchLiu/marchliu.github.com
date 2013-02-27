---
layout: post
title: "如何给 Jekyll 的 Post 设定附图"
description: "Jeklly 站点的页面可以附加元信息，用这种方法可以很方便的给Post设定附图。"
category: tech
figure: /images/ndahp_small.png
tags: [tech, web, jekyll]
---
{% include JB/setup %}

我的公司 [Dwarf Artisan](http://dwarf-artisan.com) 的网站也是用Jekyll搭建的，但是它不是典型的博客网站，例如，首页需要展示一些固定的内容。我对 Jekyll 的深入也基本上都围绕内容展示进行。

今天，我给文章加上了附图。这样，我可以在首页展示的时候，指定一张图片，而这个图片不一定也出现在正文。

首页上的文章聚合基本上都是类似这样的逻辑：

~~~
{% raw %}
{% for post in site.posts %}
  {% if post.categories contains 'production' %}
  <hr>
  <h2><a href="{{post.url}}">{{post.title}}</a></h2>  
  {{post.description}}
  <br>
  {% endif %}
{% endfor %}
{% endraw %}
~~~

现在我们扩展一下它：

~~~
{% raw %}
{% for post in site.posts %}
  {% if post.categories contains 'production' %}
  <hr>
  <h2><a href="{{post.url}}">{{post.title}}</a></h2>  
  {{post.description}}
  {% if post.figure %}
<a href="{{post.url}}"><img src="{{post.figure}}"/></a>
  {% endif %}
  <br>
  {% endif %}
{% endfor %}
{% endraw %}
~~~

代码中的 post.figure 是图片的url，那么这个url从何而来呢？

我们建立一个 jekyll Page 或 Post 的时候，会看到文件最前面的声明头

~~~~~~

---
layout: post
title: "如何给 Jekyll 的 Post 设定附图"
description: "Jeklly 站点的页面可以附加元信息，用这种方法可以很方便的给Post设定附图。"
category: tech
tags: [tech, web, jekyll]
---

~~~~~~

上面就是本文的头，其实它是一段 YAML，我们只要给它加一个 figure 字段，指向一张图片，例如，我在本页添加了figure，读者可以看到效果

~~~~~~
---
layout: post
title: "Dwarf Desktop Runner"
description: "Turn your desktop into a lively jokes runner!"
categories: [productions, production]
tags: [Screen Dwarves, Desktop Runner]
figure: /images/new_da_home_page.png
---
~~~~~~

当 jekyll 解释 index.md 时，就会把figure字段的内容取出，显示出来。下图是Dwarf Artisan 的新首页的一部分，我们可以看到，在本站首页上显示的 figure 其实是它的缩略图。

![New Dwarf Artisan Home Page](/images/new_da_home_page.png)
