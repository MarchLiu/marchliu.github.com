---
layout: post
title: "Common Lang3 Bridge 发布第一个版本"
description: "就在今天，我发布了 Apache Commons Lang Bridge 的第一个版本"
category: 
tags: ["Scala", "Apache Commons Lang3"]
---
今天，我发布了 Commons-Lang3-Bridge 项目的 0.0.1 版。

这个项目是 Apache Commons Lang 3 的 Scala 封装，旨在为 Scala 程序员提供 Scala 
风格的 Apache Commons Lang 3 资源。这个包使得 Scala 的 `String` 和 `Options[String]`
类型通过 ops 成员，可以以面向对象风格调用久负盛名的 Apache Commons Lang3 工具库
中的大量 StringUtils 方法。

项目地址：[https://github.com/scala-workers/commons-lang3-bridge](https://github.com/scala-workers/commons-lang3-bridge)

关于这个项目的设计思路，我在以前的文章中介绍过。这里不重复了。而具体的实现方面，
我的搭档 djx314（邓杰星）实现了一个极其强大的 TypeMapping 系统，兼容 Scala 2.11 直至
3.2.0 的各个版本。如果没有这个类型管理机制，整个 bridge 仓库至少要付出四倍的代码量，
并且功能上很可能也会打折扣。

这个项目是我自 Jaskell 族系之后，最重要的工具库，它会伴随我未来的开发工作，帮助我方便的
使用 Apache Commons 项目几十年来积累的实用技术资源。希望它也会帮助更多的同行，为大家
的开发工作提供便利。

Apache Commons 提供了极其丰富的单元测试和文档注释，目前我已经完整迁移了 StringUtils 的各个
测试（除了我没有封装的那些功能），这也是我能够发布 0.0.1 版的依靠。这个版本虽然只是起步，但是
它有丰富的测试保障，工程质量值得信任。未来一段时间，我在这个项目的工作，主要是继续迁移 StringUtils 的文档注释。待所有工具方法的文档注释完成，我会发布 0.1.0 版本。

按照惯例，此时应该已经可以在 search.maven.org  搜索到这个库，但是很遗憾，我还没有见到，或许是
sonatype 新启用的子站同步较慢。不过我已经在后台看到这个仓库发布成功，使用 maven 配置

```xml
<dependency>
  <groupId>net.scalax</groupId>
  <artifactId>commons-lang3-bridge_2.13</artifactId>
  <version>0.0.1</version>
</dependency>
```

就可以在 Scala 2.13 项目中引入这个包。对于 sbt 管理的项目，则是

```scala
libraryDependencies += "net.scalax" %% "commons-lang3-bridge" % "0.0.1"
```

对于其它版本的 Scala ，注意修改其版本后缀即可。

待 StringUtils 迁移完毕，我接下来的关注点会是 statistic 、 text等其它工具库。在这之前，我期待 djx314 的通用版本的 TypeMapping 项目发布。这个强大的类型工具会是未来我们的一系列 Scala
工具库的核心和基础。我们将统一将这些项目放置在 [https://github.com/scala-workers](https://github.com/scala-workers) 组织。以
`scalax.net` group 发布。
