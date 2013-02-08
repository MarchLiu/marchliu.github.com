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

不过，我不保证：

 * 用中文还是英文写文章
 * 某个连载会不会完
 * 过去其它地方发的文章会不会全迁过来
 * 过去的观点现在是否还坚持


我会保证：
 * 尽量不转载，转载或翻译会给出出处
 * 翻译会给出英文原文
 * 尽量有趣
 * 尽量不说傻话
 * 尽量不无聊

{% for post in site.posts %}
  <hr>
  <h1>{{post.title}}</h1>  
  [{{post.date}}]
  {{post.content}}
{% endfor %}
