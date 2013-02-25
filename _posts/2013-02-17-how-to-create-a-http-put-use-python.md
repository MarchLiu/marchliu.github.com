---
layout: post
title: "How To Create a HTTP PUT use Python"
description: "旧文搬家，用Python构造HTTP PUT请求的代码"
category: tech
tags: [python, web]
---
{% include JB/setup %}

用python发送put请求

做了一个服务，上传数据时接受put请求，查了一下，客户端代码用Python来写的话非常简单，跟Post基本一致。这里是一个用PUT上传文件数据的例子：

~~~
import urllib2  
opener = urllib2.build_opener(urllib2.HTTPHandler)  
with open("/storage/pic/logo.png") as f:  
    data=f.read()  
request = urllib2.Request("http://localhost:8080/logo.png", data=data)  
request.add_header("Content-Type", "image/png")  
request.get_method = lambda:"PUT"  
url = opener.open(request)  
~~~
{: .language-python}

在这里，因为只需要上传文件，我直接在data里放了全文。如果要put一个form上去，可以参见Python库文档中关于urllib2和urlib中如何发送post请求的部分。