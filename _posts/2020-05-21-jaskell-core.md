---
layout: post
title: "Jaskell Core 0.3"
description: "我和代数组合子的十年"
category: Tech
tags: [Scala, Java, Parsec]
---
{% include JB/setup %}

# Jaskell 与 Scala

大概在十几年前，我因为 Perl 6  项目，听说了 Haskell ，并由此开始接触 Parsec 算子。

通常来说，此类组合子用在解释器构造。但是在这个过程中，我发现它有更广泛的实用价值。在现代
软件开发中，对于有顺序依赖的信息结构越来越重视，比如 promise 模式或者（action-state）
反应式模式，对信息有序关系的分析都越来越重视。Parsec 完全可以用于任意信息序列的分析。

在十多年的时间里，我在多种编程语言上实现了超过十个版本的 Parsec ，有些很快用在了生产环境，
例如我在云游道工作的时期，公司的业务后台系统其实深度依赖 goparsec2 和基于它开发的 s 表
达式解析器 gisp2 。有一些属于探索和教学项目，得到阶段性成果后就搁置了，例如 jsparsec 是
我指导当时的实习生刘立完成的，因为当时公司的前端并没有解析任务，达到验证目的就可以了。例如
rust 和 swift 的实现，因为经过这个项目，确定了当时这两种语言并没有达到足够稳定可靠的程度，
也就没有继续推动。再例如 PyParsec ，这本是我在一次技术会议的休息区，闲暇时随手写出的，虽
然后续也做了一些改进，但是限于没有足够的应用场景，后面并没有一直跟进。

相对来说，在 Go Parsec 2 后，Java Parsec 库是又一个有动机驱动的项目。为了一些信息校验
工作所需，我开发了 Java 平台上的 Parsec 库，这个库后来经过几次修改和补充，与一组 SQL 工
具合并为 Jaskell 库。

Jaskell 开发过程中，始终以我个人的开发工作所需为驱动，这算是我二十年职业生涯中对开源项目的
一个总结：首先要对自己有用，才能谈得上"有用"。

由于近期开发工作的需要，我从 Jaskell 项目中分离出了两个子项目，一个是专用于 Java8 的
jaskell-java8 ，一个是由 Scala 编写的 Jaskell Core。前一个项目去掉了 clojure 部分，
后一个项目则专注于实现一个在设计上尽可能"完美"的版本。而多年来的探索已经证明，组合子中的一部
分功能需要一些现代语言的"高级"特性，这些特性在 Java 中有所欠缺。

这其中包括：
 - Monad 支持，或者至少应该是对各种环境有高度一致的封装。Java 的 Steam API 仅可以说是一个雏形
 - 对泛型的深度支持，这一点 Java 是明显不足的，Go 更不必说。反过来说，如果是动态语言如 python 
 和 Javascript，虽不能利用类型系统提供编码阶段的辅助，但是因为没有类型约束，也就不会有扩展不足的
 困扰。
 - 类型别名，这个问题我本想算在泛型里，但是 Jaskell Core ，嗯，我想更早的时候，几年前我开发 
 ruskell（rust 版本）的时候，其实就注意到了这个问题，类型别名对于这个组合子系统中一个重要的抽象
 非常关键。

最终，我完成 Scala 版本的核心功能时，我意识到，自己终于写出了一个足够理想的版本。这个版本基于 Java 
平台，在必要时可以提供足够的生产力，它的设计也足够理想，基于过往 Rust、Swift 和 Java 版本的经验，
我终于能够充分利用 Scala 的语言特性，构造一个足够严谨和灵活的版本。它可以写出足够详细的类型规约，又
不至于过度繁琐。

```scala
class TextSpec extends AnyFlatSpec with Matchers {
  "Simple" should "Run some simple tests" in {
    import Txt._
    val state = State("Hello World");
    for {
      head <- "Hello" either state
      _ <- skipSpaces either state
      tail <- text("world", caseSensitive = false) either state
    } yield {s"$head $tail"} match {
      case content: String => content should be ("Hello World")
    }
  }
}
```

十几年了，为之付出过那么多无眠之夜，终于我得到了一个能让自己满意的版本，它不仅仅是文本解析工具，还是
真正的代数意义上的组合子。我终于把这个技术从文本解析领域推广到更广泛的抽象范畴，从Haskell移植到应用
编程语言。这段旅途终于到达终点的时候，我却不知道该如何表达这种心情。我并没有很激动，但这也绝不是沮丧。
那天晚上我对太太说"我收工了"的时候，她说，你看起来就像一下子被抽空了。

Jaskell Core: [https://github.com/MarchLiu/jaskell-core](https://github.com/MarchLiu/jaskell-core)
Jaskell Java8: [https://github.com/MarchLiu/jaskell-java8](https://github.com/MarchLiu/jaskell-java8)

后面我准备花一些时间，详细的说一下 Jaskell 各个版本的设计和使用。