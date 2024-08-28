---
layout: post
title: "为 Litchi 添加配置功能"
description: ""
category: 
tags: []
---

# 配置支持

今天终于给 Litchi 加上了配置功能。安装 Litchi 后，可以在 JupyterLab 的插件配置页（Settings Editor）找到 Litchi 的配置，设定其绑定的 ollama ：

![](/images/litchi-settings.jpg)

这个功能其实开发起来简单的可笑，只要写对 schema 定义文件的文件名和信息就可以。然而它却费了我两天时间，因为实在是没有相关的文档。而 AI 又一直给出错误的信息。

最终我在 Google 上找到了相关的 sample [https://github.com/jupyterlab/extension-examples/blob/main/settings/README.md](https://github.com/jupyterlab/extension-examples/blob/main/settings/README.md) 。
这个内容仍然很不完整，但是足够使用了。

有了配置， Litchi 总算是初步有了实用价值。接下来就可以逐步深入的改进它了。包括多种 API 的接入，session 和 system prompt 的管理，更丰富的指令等等。