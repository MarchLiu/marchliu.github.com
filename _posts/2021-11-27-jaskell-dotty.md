---
layout: post
title: "Jaskell Dotty"
description: "面向下一代 Scala 的 Jaskell 实现"
category: 
tags: [scala, dotty, jaskell, parsec]
---



这是一篇补发的文章。

Dotty 的发展很快，在那段时间，我反复重写了几次 jaskell-dotty ，最终随着 dotty 项目的稳定，jaskell dotty也趋向稳定。

总的来说，Jaskell Dotty尽量使用了Scala 3 的类型特性，实现更为完整、简洁。不过也因此，最终我没能把dotty版本合并到jaskell core。目前它们仍然是两个独立的项目。

例如，一个很细节，但是很要命的事情， 如果要引入 jaskell 空间下的所有的隐式变换规则，scala 2 的 import all 是 `import jaskell._` 而 scala 3 可以 `import jaskell.{given, *}` 精确的引入所有的 given。要说全都归一化到 Scala 2语法，其实也行，但是我目前还没有下定决心。等 common lang 3 bridge 项目有初步成果了再考虑吧。