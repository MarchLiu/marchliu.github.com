---
layout: page
title: 人生啊…
tagline: Supporting tagline
---
{% include JB/setup %}

## 就是一个坑又一个坑……

所以我又开了个新坑你们不要打啊……

## 人品保证这个不坑

如果我还有人品这种东西……

{% for post in site.posts %}
  <hr>
  <h1>{{post.title}}</h1>  
  [{{post.date}}]
  {{post.content}}
{% endfor %}
