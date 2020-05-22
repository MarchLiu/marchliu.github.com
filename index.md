---
layout: page
title: 挖坑不填兽
tagline: 萝莉控什么的人家才不是哼！
---
{% include JB/setup %}

## 一个坑又一个坑……

所以我又开了个新坑你们不要打啊……

还有就是RSS地址在[这里](http://marchliu.github.com/atom.xml)。

## 坑们

### 原创

 - 专业用户的 Mac OS 工作环境建立和管理

### 翻译

 - Python Tutorial 2.7/3.x（因为硬盘发生了一些悲伤的事情基本要重来了）
 - Parsec 组合子，回头看来，我2010年开始创业的时候，"动手写一个组合子"还属于"有生之年"的事情
 
以及一些旧坑……

### 搬家+填坑

 - 现代化的 Java

{% for post in site.posts %}
  <hr>
  <h1>{{post.title}}</h1>  
  {{post.date|date: "%Y-%m-%d"}}

  {{post.description}}

  {% if post.figure %}
<a href="{{post.url}}"><img src="{{post.figure}}"/></a>
  {% endif %}

  [阅读全文]({{post.url}})
{% endfor %}
