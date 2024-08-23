---
layout: post
title: "A Story About Litchi"
description: ""
category: 
tags: ["jupyter", "jupyterlab", "jupyterlab-extension", "ai", "ollama"]
---

# Litchi 项目
离开 CSDN AI 组以后，我终于有钱买了一台可以运行 Ollama 的工作机。自此以后的近一年间，我一直在尝试做一个适合自己的 AI 工作环境。
## 尝试和探索
这个方向我的第一个尝试是一个名为 [blue-shell](https://github.com/MarchLiu/blue-shell) 的 ollama 客户端。这是一个简单的对话 repl，我仅仅是在 console 环境将 markdown 处理为更容易阅读的形式。
第二个尝试是名为 [In The Shell](https://github.com/MarchLiu/in-the-shell) 的 java 桌面程序，它支持预定义的 prompt 模板，与 Ollama 对话并将对话历史格式化展现。这已经是我每日必用的工具，各种操作习惯都是我为自己量身定制的，我甚至还给它写了一个语音输入。
但是我还是想再写一个，这是因为我需要一个对markdown编辑和技术写作高度友好的环境。我需要它可以将对话过程与编程和写作紧密的结合在一起，并且自然的将AI的对话过程集成进来。并且，In The Shell 没有保存历史记录的功能。我当时只是写 TENSOR DANCER 的间隙，随时增补一些小功能，满足我即时的需要。这使得 In The Shell 始终处在凑合能用的状态。比如，我要拿出 AI生成的内容时，仅需要双击对应的cell，它就会把内容复到剪贴板，这在我用它翻译文档的时候非常方便，但是如果我让AI给我写单元测试，我没有办法直接选中格子中的一部分内容。
因为 In The Shell 也是我拿来练习 JavaFX 的学习项目，我还没有学会怎么让一个 Listview 平滑滚动的同时可以任意选中 cell 中的文字。
我仍在继续尝试改进这个项目，但是如果我仅仅是想在一个复杂文档环境中使用 AI，那么从头写一个文档编辑系统，似乎不是很经济的选择。
技术写作，或者说可执行文档的霸主无疑是 Jupyter。我也就动了在 jupyter 中使用 AI 的心思。
## 在工作环境中集成 AI
在IDE或者编辑器中集成 AI 客户端并不新鲜，Intellij 和 Visual Studio Code 中都有很多 AI 聊天插件和辅助编程工具。
但是对我来说，我更希望可以有一个支持 Markdown 的环境，让我能够编辑复杂的内容，并且在对话过程中可以将一部分内容作为程序代码使用。大多数的 AI 插件，只包含了 markdown view，而输入过程的编辑支持，都做的比较简单。
这方面最接近我期待的，其实是 bing ai 的 notebook 模式。但是我哪怕在香港，也不一定能稳定的访问到这个服务。对我来说，写一个满足自己需要的软件，总是个巨大的诱惑。
因此，我开始了解如何扩展 jupyter 。
首先我尝试了 markdown 的 magic command 。这个功能可以让code cell的内容按照指定的程序去执行，将stdout的输出写到output区。
但是我尝试了一下，发现有两个方面的分歧。
第一是这个功能只能用在 code cell，而我期待能够同时代码和文档的内容。
第二个是 magic command 其实很强大，甚至可以传入参数，但是它对于每个 cell 都是局部的，我不能直接拿到整个文档的内容。
于是，我在 KIMI 的提示下，开始尝试编写一个 JupyterLab 插件。
## Litchi
JupyterLab 是 Jupyter Notebook 的基础上发展出来的集成开发环境，它提供了一套基于前端技术的插件规范——嗯我看到有server插件的选项，还没有去了解。
于是我按照官网的文档，从入门的那个显示 NASA 每日图片的插件开始了修改路程——别问我 KIMI，试了各种我能访问到的 AI ，关于 jupyterlab 插件，大家都没帮上太多忙。  
这不是 AI 能力够不够的问题，JupyterLab 官方的文档，自己也说的很含糊。例如我想要把 ollama 回复的内容写到新建的markdown cell中，如何在插件中创建一个 markdown cell，就没有写在文档里。问 AI 的话，它只能发挥自己的想象力，写出一堆看起来完全正确的胡言乱语。
最终，我在 stackoverflow 找到了这个问题的答案，一个同行指出根本不要管什么 react MVVM 之类的东西，直接调用 jupyter 命令：
```ts
      const { commands } = app;
      commands.execute('notebook:insert-cell-below').then(() => {
        commands.execute('notebook:change-cell-to-markdown');
      });
```
于是，这个问题就用这么一个看起来很离谱却越想越合理的方法解决了。
这个项目，我没有想到特别贴切有好记的名字，这个世界上已经有太多的 AI 工具叫 copilot 或者 associate 了，我决定按照习惯，找一个食物的名字。
Litchi 就是荔枝，看起来这个名字就是从中文音译而来。这很好，它很简单，荔枝又是我非常喜欢的水果。
我在广东生活了十五六年，在那里成家立业，度过了整个青年时代。日啖荔枝三百颗，不辞长作岭南人。
直到我发布这个软件包的时候才发现问题，pypi上的 litchi 项目名已经被占了，这个项目的作者说他仅仅是为了给自己在开发的另一个项目占个位子：
> This is a placeholder package created in July 2015 to ensure we have a place to upload our software, called “lychee.” Since someone else already took that package name, we’ll use this alternative spelling.
问题是快十年过去了，这个项目还是空的。看作者头像还是位白人老兄，图啥啊真是，何必为难我这个精神广东人。
我将项目名改成了 Jupyter-litchi 。
比这个更麻烦的是，我发现按照官方推荐的方式打包上传到 pypi 再用 pip install 安装，会安装失败。
目前我还没有定位到问题出在哪里，目前看起来默认的项目模板本来就少东西，我现在将前端部分发布到 npm ，python部分发布到pypi，然后调用
```bash
jupyter labextension install jupyter-litchi
```
可以安装成功。但是 `jupyter labextension install ` 本身是个 Deprecated 的命令，也许哪一天 jupyterlab 就会去掉这个功能。我还在尝试如何正确的把这个插件编译成可以通过 pip 安装的包。
### 写在最后
从四月底回到北方，我现在仍然没有找到工作。每天我会用掉招聘平台限定的投递次数，大多没有回应。
关于找工作，值得将来单独写出来。很多事情如果不是发生在自己身上，其实也算是有趣的经历。比如经常被猎头回应年龄超出，我也觉得没有太失落，我知道正常情况下，我们在招聘市场上确实很难见到45岁以上的老头子还能用现在的开发技术写程序。
所以写程序这件事对我是一个精神上的治愈过程。我向自己证明了， 我还可以拿起十多年不用的CPP，写出当年根本不可能写出来的数学算法和数据库插件。也可以一个星期从 typescript + react 入门到发布 jupyter 插件。我的工具一直在变，而写程序这件事，我其实是越来越自信的。
最后，希望这个我自己用着很开心的小工具，也可以帮助同行们更愉快的工作。祝我们都有更好的明天。