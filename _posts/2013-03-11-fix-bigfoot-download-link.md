---
layout: post
title: "大脚文件下载链接错误"
description: ""
category: 
tags: [tech]
figure: /images/bigfoot403.png
---


前几天，我发了一篇Blog [WOW 大脚的 Mac 版打包](/2013/03/06/wow-bigfoot-mac-os-package)，
放了一个WOW大脚的Mac包，因为原本就是给家人随便做的一个小东西，放上去以后也没注意看。偶然我发
现点击链接居然不能下载。（印象里也有读者反馈过，只是一下班我就忘了这个事情了）。

![Download 403](/images/bigfoot403.png)

错误提示很奇怪，是权限问题，上图是我特意重现出来的。我回想一下，应该不是文件访问不到，因为我之前有同样在jekyll中放下载
链接并且下载成功的。这看起来像是……

等等……如果仔细看URL，会发现有个很诡异的现象，就是最后有个/。这……其实是进入了一个目录吧……

仔细回想了一下，Mac OS的应用程序有个非常不同于Windows/Linux的地方，它不是一个可执行文件，而是一个目录。是的，每个 
xxx.app 都是一个有特别结构的目录，如果你在任何一个app上右键，然后在弹出菜单上选“查看包内容”，会看到类似这样的结构：

![Big Foot Structure](/images/bigfoot_structure.png)

在 Apple 的开发者文档，或者类似 Advanded Mac OS X Programming 这样的书里，有对 Mac OS 包的详细介绍，这里不
多解释了。这其实是个非常小的事情，但是有时候细节也会绊倒人。只能说庆幸这次不是错在工作项目吧：）。

最后，[原文](/2013/03/06/wow-bigfoot-mac-os-package)的下载链接已经修正，也可以直接点击[这里](/static/BigFoot.zip)。