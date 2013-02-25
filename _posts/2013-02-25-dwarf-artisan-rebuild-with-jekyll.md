---
layout: post
title: "我用 Jekyll 重构了 Dwarf Artisan"
description: "最初紧急搭建的公司网站已经越来越难以满足要求，我重构了 Dwarf Artisan 的网站。"
category: tech
tags: [tech, web, jekyll]
---
{% include JB/setup %}

## 意外的开始

好吧……我要承认，其实我是在 Dwarf Clipboard 提交审核（呃，这个产品因为技术问题被拒了，我还没决定它的去处，总不能一直在我们的代码仓库里睡觉）的时候才想起来其实 Mac OS App Store 需要一个 App 维护网站。于是，原本因为人力问题搁置的 [Dwarf Artisan](http://dwarf-artisan.com) 网站，就迫在眉睫了。我临时拿起 iweb，用差不多20分钟，拿内置模板架了一个网站。

这个网站很好，iweb是个很赞的个人建站工具，它内置了很多漂亮的模板和组件，甚至允许我直接把iphoto的相册拖到页面上变成一个相册，而且这一切还都是纯静态的。这很省心。

不过接下来就越来越麻烦，首先我不能让同事帮我编辑页面，因为 iweb 根本不能导出自己的文档，而且它只能把生成的网站同步出去，不能拉回来。

第二件麻烦是是它不能直接编辑页面源码，至少我没找到。这个时候才让人明白fontpage有多良心。没有这个功能，不仅仅是不能写一些特效的问题，我不能在页面上放mailto、paypal、google analytics 之类的功能，想都别想，除非它们就出现在 iweb 的组件板上。

## Jekyll 和 Jekyll bootstrap

首先是我们发现了 [github pages](http://pages.github.com) 服务，这很抢眼，每天我们用github服务工作的时候，很难忽略这东西的存在。接下来，从 pages，我们就知道了 [jekyll](https://github.com/mojombo/jekyll) ，简单玩了一下，这是个非常适合我们的技术。

它很简单，用markdown书写内容，content和layout分开，而且可以自动生成静态页面——这足够了，反正我们的公司网站上没有什么动态逻辑。

接下来，我的同事江南发现了它的 [bootstrap 项目](http://jekyllbootstrap.com/) 。这个项目可以大大的减轻工作量，我们直接用它的设置就可以插入 google analytics，选用若干主题。

很快，我搭建了一个原型，接下来遇到了一些麻烦，主要是 markdown 解释器的问题，有的中文支持不好（问题不算太大，但是总要以防万一），有的代码块功能实现的不好（这对一个技术团队也太重要了）。最终我选择了 kramdown 。然后我在我的个人 github pages （就是各位看到的这个站点）上做了一些实验，在选定theme以后，只剩了一个问题：相册。

## 纯 Javascript 相册

除了产品展示，Dwarf Artisan 的网站上还有一些宝贵的东西：我们的朋友阎女士的[画作](http://dwarf-artisan.com/columns/art/)。无论如何，应该弄一个像样的相册工具放这些好东西吧。顺便说一下，Dwarf Artisan 只提供缩略图，对原作感兴趣可以直接联系作者，水印里有她的邮箱。

这件事比预想的麻烦一些，我找了一些纯静态的Javascript相册代码，最后决定用了[这个](http://www.twospy.com/galleriffic/index.html)。不过sample的代码不能直接拿来用。有几个问题，首先js里有一些硬编码，依赖（很少）几个id，还有那么几个id跟我用的 jeykll 主题 Mark Reid 重名（所以它们的css混到了一起）。再就是 Mark Reid 主题的content区只有七百多像素宽，而这个相册的sample用掉了九百多个像素的宽度，直接嵌进去布局会混乱。经过一些细节调整，我做了一些妥协，初步达到了目的。

目前我的做法是，如果我需要在我的某个 jekyll 网站上添加一个相册，就在那个网站的jekyll 根目录建立一个 images 目录，在里面建对应的相册路径，然后在这个根路径下执行脚本，传入相册子目录的名字作为相册名。脚本[^1]会把图片剪裁成小尺寸以后放到对应的路径下，再生成一个模板片段，放到 _include/DA/ 目录下，然后就可以在需要它出现的地方include它了。

当然，这不是正确的做法，这只是我赶工的结果，正确的做法应该是把它变成一个通用的模板，传入一个数组，然后include模板，就像 JB/setup 和 JB/post_collate 这种。我只是不太喜欢像[这位老兄一样](https://groups.google.com/forum/?fromgroups=#!topic/liquid-templates/qwE5hWk-Kik) 用split创建数组——不过，这是个好办法。

我使用的代码放到了[黑科技代码库](https://github.com/Dwarfartisan/BlackCookbook/)。[在这里](https://github.com/Dwarfartisan/BlackCookbook/tree/master/ruby/jekyll-galleriffic)。欢迎随意取用，不过，这个代码没有经过任何改进，用之前最好自己处理一下，例如路径依赖什么的。

[^1]: 好吧，我去掉了“自动”两个字，作为十年的Python用户得瑟自己会这种脚本也太逊了，哪怕这是我写的第三个还是第四个ruby脚本（不算去年写过的两千行puppet）……

