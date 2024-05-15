---
layout: post
title: "Java Llama CPP On MacOS"
description: ""
category: 
tags: [java, cpp, llama, llama.cpp]
---


近一段时间，尝试在 Java 中使用 gpu 资源，发现已经有人做过类似的工作，实现了一个[java 的 llama.cpp 封装](https://github.com/kherud/java-llama.cpp)。

我使用过一些 llama 的 runtime，llama.cpp 是我非常喜欢的。它性能足够好，对 MacOS 有很好的支持。基于 llama.cpp 的 ollama 
项目，也是我非常常用的一个工具。我原本就计划在 llama.cpp 和 ggml 项目上下一些功夫。现在有了这个封装，自然要拿来试一试。

项目的结构不算复杂，因为我想要学习它对llama.cpp的封装，所以下载了源代码。按照文档的话，仅需要先执行一组构建工作：

```shell
mvn compile
mkdir build
cd build
cmake .. # add any other arguments for your backend
cmake --build . --config Release
```

即可编译使用。

我以前就构建过 llama.cpp，所以我的电脑上相关的依赖都是完整的，这部分出了必要的网络配置，也没有遇到什么问题。

但是我尝试执行单元测试时，提示错误信息：`fatal error: 'ggml-common.h' file not found` 。

一开始我以为是头文件的搜索问题，做了一些尝试没有成功，去项目站提了 issue：[fatal error: 'ggml-common.h' file not found](https://github.com/kherud/java-llama.cpp/issues/61)。

经作者指导，需要将 llama.cpp 编译为嵌入模式：

```
mvn compile
mkdir build
cd build
cmake .. -DLLAMA_METAL_EMBED_LIBRARY=ON # set project as embed mode
cmake --build . --config Release
```

然后就一切正常了。

我原本以为这个项目直接将llama.cpp的代码复制过来，后来看了一下 cmake 代码，原来是编译时从 github下载的。

```cmake
#################### llama.cpp ####################

FetchContent_Declare(
	llama.cpp
	GIT_REPOSITORY https://github.com/ggerganov/llama.cpp.git
	GIT_TAG        b2797
)
FetchContent_MakeAvailable(llama.cpp)
```

