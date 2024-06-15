---
layout: post
title: "C/C\+\+ 混合编程"
description: "记录一下tensor dancer开发过程中遇到的混合开发问题"
category: 
tags: [c, cpp, 'tensor dancer', llm, postgresql]
---

# C/C\+\+ 混合编程
这几天踩了一些坑，做个笔记。
## 两种语言，两个编译器
十多年前，我曾经是个职业的 CPP 程序员，但是那个时候我用 VC，这是一个高度自动化的 CPP 开发环境，它自动的做了太多的工作，以至于我甚至没有注意到自己的很多问题。
理论上，CPP是C的超集，使用 CPP 编译器可以自动的兼容 C 代码。这也导致了有时候我们其实不太注意自己到底在写C还是CPP。甚至很多构建工具允许我们用 CPP 编译器直接编译C 代码。
这在很多项目中是好的功能，不过我还是决定在 tensor dancer 项目中厘清这个问题，让混合编程的过程放到明面上来。
Meson 在默认情况下，使用 C 编译器编译 .c 文件，用 CPP 编译器编译 .cpp 文件。因为前面的理由，我没有将 c 编译器指定为 `c++`。
这样就暴露出一个问题，如果我在一个 c 程序中 include 一个CPP 头文件，它会递推到相关的 CPP 代码，这就会引发语法错误或符号找不到这样的错误。应对的方式是，定义一个 .h 文件 ，其中仅包含引用符合C标准的导出定义，例如dancer.h的内容是：
//
// Created by 刘鑫 on 2024/6/15.
//

#ifndef TENSOR_DANCER_DANCER_H
#define TENSOR_DANCER_DANCER_H

#include "ggml.h"

#ifdef __cplusplus
extern "C" {
#endif

extern struct Matrix {
    unsigned int magic;
    enum ggml_type type;
    size_t rows;
    size_t columns;
    void *data;
} Matrix;

int load_matrix(struct Matrix *matrix, const char *filename);

int write_matrix(struct Matrix *matrix, void * buffer, size_t size);

int mul_matrix_vector_f32(struct Matrix *matrix, float * vector, float * result);

struct Matrix* InitMatrixF32();

struct Matrix *InitMatrix(enum ggml_type type);

struct Matrix *CreateMatrix(enum ggml_type type, size_t rows, size_t columns);

void FreeMatrix(struct Matrix* matrix);

#ifdef __cplusplus
};
#endif

#endif //TENSOR_DANCER_DANCER_H

然后将整个模组定义成 library：
dancer_core = static_library('dancer_core', 'src/dancer_core.cpp', 'src/dancer.cpp',
                             dependencies : [ggml_dep, blas_dep, lapack_dep],
                             cpp_args : lib_args,
                             link_args : ['-lstdc++'],
                             install : true)

core_dep = declare_dependency(
    link_with : [dancer_core],
    include_directories : include_directories('./src')
)
在c项目中，不直接引用代码，而是link库，引用头文件时，也仅限于这些用于联编的头文件：
test_load_matrix = executable('load-matrix', 'src/insight.cpp', 'test/load_matrix.cpp',
                              dependencies : [ggml_dep, blas_dep, lapack_dep, core_dep])

在meson项目中，我其它的子项目通过依赖 `core_dep` 实现对 tensor dancer 内核的引用。
## C 的超集
在 CPP 这边，extern 块的定义也有一些要注意的。
在 CPP 内部，struct 和 class 仅仅是默认的访问限定范围不同，但是c的struct更简单一些，因此，在 extern 块 中，Matrix 的定义是：
extern struct Matrix {
    unsigned int magic;
    enum ggml_type type;
    size_t rows;
    size_t columns;
    void *data;
} Matrix;

这里除了遵循 c 的 struct 定义语法，还通过 extern 关键字确保 Matrix 不会重复定义。
使用 Matrix 时，也要注意遵循 c 的语法，例如这个 load matrix 方法：
int load_matrix(struct Matrix *matrix, const char *filename) {
    auto fin = std::ifstream(filename, std::ios::binary);
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, filename);
        return -1;
    }
    auto status = fill_matrix(*matrix, fin);
    fin.close();
    return status;
}
它的实现是cpp代码，也可以放心引用 stl，但是接口声明要遵循c语言，若非如此，这里的第一个参数，可以用引用代替 matrix 指针。
需要注意的是，在 `dancer.h` 中，引用的头文件也需要来自 c 兼容的代码，例如我这里引用了 `ggml.h`，这是一个c库，而对cpp资源的引用，都写在实现文件`dancer.cpp`中。也就是说，c能够引用到的头文件，也需要是“符合c语言”的。而那些cpp的部分，都封装在编译后的library中。
引用 tensor dancer 内核的 CPP 代码就没有这种限制，它可以任意引用相关的头文件。
## 运行时
PostgreSQL 的插件是动态链接库，所以在meson中要定义成 `shared_module`：
vector_extra = shared_module('vector_extra', 'matrix_lite.c', 'pgv_extra.c',
                             dependencies : [blas_dep, lapack_dep, pg_dep, ggml_dep, gettext_dep],
                             c_args : lib_args + ['-DUSE_PG', '-I' + project_root_dir])
在PG环境中，需要使用palloc和pfree管理内存，因此我定义了一个封装宏，在PG环境使用 palloc/pfree，在其它环境使用 malloc/free ：
#ifdef USE_PG
#include "postgres.h"
#define dalloc(size) palloc(size);
#define dfree(data) pfree(data);
#else
#define dalloc(size) malloc(size);
#define dfree(data) free(data);
#endif
这个开关在 meson 中配置，在前面介绍的 `vector_extra` 定义中，就包含了这个开关。
