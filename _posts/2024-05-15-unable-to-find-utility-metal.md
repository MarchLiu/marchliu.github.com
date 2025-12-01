---
layout: post
title: "unable to find utility \"metal\""
description: "解决 macOS 上编译 llama.cpp 时遇到的 metal 工具找不到的问题"
category: tech
tags: [cpp, macos, matel]
---

编译 llama.cpp 的时候，遇到一个错误

```
--- stderr
xcrun: error: unable to find utility "metal", not a developer tool or in PATH
thread 'main' panicked at 'shader compilation failed', ...


```

我没有保存当时的错误信息，这是搜索到的一个类似的讨论。主要的问题是一样的，都是不能加载 macos 的 
matel 资源。从这个[讨论](https://github.com/gfx-rs/gfx/issues/2309)，可以知道，其实是因
为我们这些非 macos 开发者的程序员，默认情况下编译器的绑定路径指向了 commandline-tools ，而
不是 xcode 路径。

执行 
```shell
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
```
后，就可以正常编译了。

这个问题很小，但是我还是决定记一下，因为它多少有点反直觉，xcode的命令行工具和 xcode 居然有两套环境。