---
layout: post
title: "搭建 SAE 本地开发环境"
description: "现在在本地，特别是*nix系统下搭建SAE开发环境，已经是一个相当方便的事情。"
category: tech
tags: [tech, web, python]
---
{% include JB/setup %}

SAE 平台上，大概 PHP 才是 First Language，相对来说 Python 支持起步比较晚，不过至今也已经相当完善。

SAE Python 版的本地开发环境，搭建已经相当简单。这里介绍一种比较简单和容易操作的方式。

我的开发环境是一个debian testing虚拟机，所以后面的内容都基于这个前提。

首先，安装一个virtualenv环境（关于virtualenv的详细资料，可以参见它自己的官网 http://www.virtualenv.org/en/latest/ ）：

    $ ~/: virtualenv sae


激活它


    $ ~/: source ~/sae/bin/activate

然后就进入了一个独立的虚拟环境。这样做的好处是，SAE支持的各种组件有限，有些是需要特点的版本，在独立的虚拟环境下更容易管理。

在virtualenv中，用pip可以安装Python包。

一般来说，我们需要在这个环境下安装：

  - sae-python-dev ，SAE 的开发环境，这个包安装好后，virtualenv的path中会有一个dev_server.py 脚本
  - django或webpy这样的web框架，这个一般来说比较容易。
  - mysqldb，debian的话需要确认libmysqlclient-dev 已经安装
  - pylibmc， 我没太注意这个库的依赖，如果有问题的话需要安装对应的memcache c库
  - PIL 
  - ...

需要注意的是，有时候我们本地会遇到PIL库使用有问题，提示缺少 jpeg或zip等decoder。查了一下网络，应该是一些环境下（例如我使用的debian是在虚拟机里），debian没有正确的link对应的库到搜索路径。


解决方法很简单，首先确认 libjpeg, libz, libpng 等我们需要用到的库（例如处理 png 和 jpeg 需要这三个库）已经通过 apt 安装，然后在  /usr/lib/x86_64-linux-gnu/ （或者你的系统版本对应的目录）这样的路径下找到这些so，link 到/usr/lib 即可。

   $ ~/: sudo ln -s /usr/lib/x86_64-linux-gnu/libjpeg.so /usr/lib
   $ ~/: sudo ln -s /usr/lib/x86_64-linux-gnu/libfreetype. /usr/lib
   $ ~/:  sudo ln -s /usr/lib/x86_64-linux-gnu/libfreetype.so /usr/lib
   $ ~/:  sudo ln -s /usr/lib/x86_64-linux-gnu/libz.so /usr/lib 

sae的启动脚本不复杂，一般来说配置和启动可以参照官方文档 http://sae.sina.com.cn/?m=devcenter&catId=304 。我使用的启动脚本可以做个参考：

    dev_server.py --mysql=xxxx:xxxx@localhost:3306 --host=10.37.129.11 --storage-path=/tmp

这里解释一下： 

 指定mysql服务器配置后，就不用判断是否在服务器环境下来设置数据库连接了，可以直接使用文档中的连接方式。提供 storage path 参数后，也就可以在本地环境中模拟sae服务器的storage服务。