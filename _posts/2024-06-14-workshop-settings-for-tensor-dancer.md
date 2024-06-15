---
layout: post
title: "Tensor Dancer 项目的开发环境配置"
description: "我准备围绕几个基础的算法工具，编写一系列算法，最终形成一个高阶的算法库，并提供相关的 PostgreSQL 插件，这里记录一下开发环境配置"
category: tech
tags: [c, cpp, postgresql, ai, ggml, blas, lapack, pgvector, matrix]
---

# 环境和依赖
我准备围绕几个基础的算法工具，编写一系列算法，最终形成一个高阶的算法库，并提供相关的 PostgreSQL 插件。
这个系列，我们主要依赖三个第三方库，线性代数计算库BLAS、LAPACK和GGML。
BLAS 全称 Basic Linear Algebra Subprograms ，底层基于 Fortran 代码，是众多线性代数库的基石。
LAPACK 全称Linear Algebra PACKage，基于 BLAS 库开发，提供了一些矩阵计算算法实现，例如SDV分解。
我的日常工作在一台 M3 芯片的 Apple MacBook 笔记本上完成，因此这里也主要围绕 MacOS 展开，兼顾 Linux。不过我手头没有 Navida 显卡，也没有 CUDA 的运行环境，这方面的东西无法实验。
## 开发工具 
BLAS 和 LAPACK 本身有系统内置的版本，但是实验来看，连编时会找不到头文件，好在它们也有 homebrew 版本，可以直接安装。但是ggml 要程序员自己编译。因此我们还是从开发工具的准备开始。
首先，我们需要安装 Apple 的开发工具集，这个不多介绍，Apple Store上现在可以直接下载xcode……嗯我大概是这个世界上少数在apple store上花过45块钱买xcode的苹果用户了。现在xcode是个免费软件。
说个好笑的，xcode和 xcode-commandline 的库搜索路径不一样，我也是在编译 llama.cpp 的一个移植库时，才发现的这一点。我们需要确认系统使用的是 xcode 。
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
因为我不是从一个裸机开始这次工作，严格来说我很难复现到底这个开发环境需要装哪些东西。我们先安装 homebrew 。
Linux 用户可以简单的把 Homebrew 当作类似 YUM 和 APT 的软件安装服务。这个领域曾经有好几个竞争者，我自己就曾经用过很多年 MacPort，它更接近 FreeBSD 的 ports，而我曾经做过 FreeBSD 的 SA，这个风格让我觉得很亲切。但是现在 Homebrew 已经是事实标准了，基本不会再遇到当年那种某个软件只能在 macport 上找到的情况。 
HomeBrew 的安装很简单
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
这个命令来自它的官网 [https://brew.sh/] 。我建议读者还是直接从官方首页查找安装命令，以免某一天官方修改了这个内容。
接下来，我建议安装一个 iTerm2，用它代替 MacOS 内置的终端软件。
brew install iterm2
根据历史，我至少安装过这些开发工具和基础组件库
```shell
brew install meson cmake
brew install ninja
brew install automake libtool boost pkg-config libevent
brew install LAPACK
```
CMake 是目前主流的 C/CPP 项目构建工具，但是 PostgreSQL 官方使用了 meson，我为了熟悉这个工具，以便后续对 PostgreSQL 的内核做一些研究，这次的项目我使用 meson 。而它依赖 ninja 作为自己的下级工具。
LAPACK 中包含了 BLAS，如果为了深入的研究和修改，可以单独安装它，如果仅仅作为依赖使用，其实仅安装 LAPACK 也可以。
编程工具选择比较多，我个人目前主要用 Clion ——多年来我一直是Intellij Ultimate 付费用户。但是并不表示这个项目只能用 clion 。Visual Studio code 完全没问题，我自己还是几十年的 Emacs 用户，我可以保证用 Emacs 也不会有问题，VIM……我虽然不太会用，但是应该也没关系。我自己构建工作主要是围绕命令行进行，所以我重点讨论在命令行和 clion 遇到的问题和工作方法，相信其它工具的用户可以参照这些内容自行处理。
Clion 有 meson 插件，需要单独安装。
## GGML
GGML 是一个高性能张量计算库，使用C语言开发，是LLAMA.cpp的核心算法库。支持多种硬件体系的加速能力。这个库在计算领域有巨大的潜力。
GGML 给我的感觉，似乎就是为 `llama.cpp` 和 `whisper.cpp` 提供算法核心，因此它本身的工程化并不完善，缺少文档，也没有独立的发型版本，甚至 `pkg_config`(`ggml.pc.in`) 配置写的跟玩儿一样：
```pkg
prefix=@CMAKE_INSTALL_PREFIX@
exec_prefix=${prefix}
includedir=${prefix}/include
libdir=${prefix}/lib

Name: ggml
Description: The GGML Tensor Library for Machine Learning
Version: 0.0.0
Cflags: -I${includedir}/ggml
Libs: -L${libdir} -lggml
```

我看到 `Version: 0.0.0` 的时候，也着实呆了一下。
好在 GGML 的编译还是比较顺利的，在项目目录中
```shell
mkdir build
cmd build
cmake ..
make
make install
```
一般来说，就可以顺利的在自己的电脑中安装 ggml 库。
下一节，我们先做一个简单的 meson 项目，验证一下。