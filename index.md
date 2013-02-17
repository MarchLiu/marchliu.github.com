---
layout: page
title: 挖坑不填兽
tagline: Supporting tagline
---
{% include JB/setup %}

## 就是一个坑又一个坑……

所以我又开了个新坑你们不要打啊……

还有就是RSS地址在[这里](http://marchliu.github.com/atom.xml)。

## 坑们

### 原创

 - Mac OS App Store 审核中的沙盒问题
 - jekyll 建站的……一些吐槽……
 - 微小型创业团队的技术组合和管理心得

### 翻译

 - Python Tutorial 2.7/3.x（因为硬盘发生了一些悲伤的事情基本要重来了）
 - 一些Mac / iOS 开发文章

以及一些旧坑……

对了，如果融资成功我也许有时间再填一个大的，是什么先不说……

### 搬家+填坑

 - 脱离IDE开发Mac OS/iOS app

{% for post in site.posts %}
  <hr>
  <h1>{{post.title}}</h1>  
  [{{post.date}}]

  {{post.description}}

  [阅读全文]({{post.url}})
{% endfor %}
