---
layout: post
title: "Arg Parser"
description: "A simple arguments parser base special rules and Jaskell style"
category: 
tags: ["java", "jaskell"]
---

前几天我写了一个简单的词法分析器项目：[https://github.com/MarchLiu/oliva/tree/main/lora-data-generator](https://github.com/MarchLiu/oliva/tree/main/lora-data-generator) 。
通过词法分析快速生成 lora 训练集。在这个过程中，我需要通过命令行参数给这个 java 程序传递一些参数。

这个工作让我想起了一些不好的回忆。我这些年来做过太多类似的东西，随着程序开发的进展，命令行参数的规则越来越复杂，于是简单的几个赋值操作迅速变成
了一大堆逻辑分支。

对于 Python 程序，至少内置的命令行解释工具 [argparse](https://docs.python.org/zh-cn/3/library/argparse.html) 足够好用，对于通常
的开发工作已经足够。但是 Java 标准库中并没有这样的组件。

目前我所知道的，[apache commons clit](https://commons.apache.org/proper/commons-cli/)  或许是个好选择。但是我也有一些自己特定的
期待：

 - 我希望有一个能够很方便的和 jaskell try 机制良好配合的工具
 - 希望它的构造足够方便
 - 对我常用的命令行设计风格有足够的支持，具体的内容后面我会介绍

于是，我顺手在 [jaskell-rocks](https://github.com/MarchLiu/jaskell-rocks) 库中加入了一个 ArgParser 工具，用于处理以下的命令行设计：

 - option: 可以指定 --xxx 类型的参数，这类参数需要带有参数值
   - option 可以有默认值
   - option 可以是 required 或者可选的
   - option 可以设置为只能在某几个值中选择
   - 允许多次传入同一个 option 名的参数，所有同名 option 的参数聚合为一个集合
 - with option：with option 不需要带有值，
   - 可以通过 --with-xxx 或 --without-xxx 表示某个 with option 是否设定
   - with option 有默认值，但是没有 required 限制
 - switch 开关
   - 开关有默认值
   - 可以通过 --enable-xxx 或 --disable-xxx 表示一个 switch 的状态
   - switch 有默认值
   - switch 有 required 或可选的状态
 - args
   - 前面介绍的三类都是有显式参数名的参数项，在其后可以有零到多个无名参数
   - 这些参数可以隐含有 require 约束，例如复制操作必须要提供 source 和 target，args 的 size 就至少需要为 2
   - args 参数也有可能有默认值，例如一个连接http服务的调试脚本可能默认连接 `localhost:8080` ，没必要显式给出。
   - 显然，required args 应该在 所有 args 的最前面，而有默认值的应该在最后
 - help 所有显式设定的参数都允许提供 help 文本，argParser 内置对 `--help`，`-h` 的识别，输出参数的文档
 - 允许为参数名设置缩写，例如 `--source` 可以设定为 `-s` 。

目前的 ArgParser 已经完全满足我的需要，例如 oliva 的 lora 数据生成工具，就使用了这个命令行解释器：

```Java
        var lexer = new LexerRouter();

        var source = Option.create("source")
                .help("source project directory")
                .required(true);
        var target = Option.create("target")
                .help("where save lora train dataset")
                .required(true);

        var argParser = ArgParser.create()
                .header("Oliva is a assistant program. It just cut source code to lora training data.")
                .formatter("%1$-20s %2$-20s %3$-60s\n")
                .option(source)
                .option(target)
                .footer("Power by Jaskell");

        argParser.parse(args)
                .onFailure(err -> {
                    System.err.println(err.getMessage());
                }).onSuccess(result -> {
                    result.autoHelp();
                    //...
```

这里就是 `lora-data-generator` 项目的参数解析部分。如果传入了 help 参数，autoHelp 会向控制台打印帮助然后 `System.exit(0)` 退出。如果
需要深度的控制help行为，这个解释器还暴露了几个与帮助文档有关的中间方法，包括帮助格式的模板字符串。这个工具已经初步满足了我的需要，在未来，也
许我会加入一些便利的工具方法，类似 `intValue` 这种。但是总的来说，这个设计不需要再有大的改动，如果真的遇到在结构上不能满足我的需求，也许我会
考虑 apache commons cli。