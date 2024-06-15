---
layout: post
title: "在 PG 中支持矩阵乘法"
description: "Tensor Dancer 项目核心目标之一，是在 PostgreSQL 中提供张量计算能力。这里我们从一个最简单的需求开始：支持矩阵乘法，使我们可以对 pgvector 的向量使用 PCA 矩阵降维。"
category: tech
tags: [c, cpp, postgresql, ai, ggml, blas, lapack, pgvector, matrix]
---

# 在 PG 中支持矩阵乘法
Tensor Dancer 项目核心目标之一，是在 PostgreSQL 中提供张量计算能力。这里我们从一个最简单的需求开始：支持矩阵乘法，使我们可以对 pgvector 的向量使用 PCA 矩阵降维。
## 关键代码
将一个n维向量V降到m维，可以简单的理解为用一个 `n*m`的矩阵 T 乘以 n 的转置，
$$
这是一个非常基本的线性代数算法，在 BLAS 库中提供了通用的矩阵乘法。我们可以简单的调用这个函数。
```c
int mul_matrix_vector_f32(struct Matrix *matrix, float *vector, float *result) {
    cblas_sgemm(CblasRowMajor, CblasNoTrans, CblasNoTrans,
                (int) matrix->rows, 1, (int) matrix->columns,
                1.0, (float *) matrix->data, (int) matrix->columns, vector, 1,
                0.0, result, 1);
    return 0;
}
```
这里我们重新封装了上层的数据结构，简化了`cblas_segemm` 的参数列表：
* 我们仅需要确定的功能：一个变换矩阵，将一个向量处理为另一个向量，因此这里三个矩阵的形状可以由这个变换矩阵确定
* pgvector 的 vector 使用 float32，我们可以暂时只考虑这个精度
* 原型阶段，我们可以简单的忽略检查逻辑
这里我们通过一个自定义的结构 Matrix 封装了数据格式，这个结构的定义如下：
```c
struct Matrix {
    unsigned int magic;
    enum ggml_type type;
    size_t rows;
    size_t columns;
    void *data;
};
```
这个类型的字段分别表示：
* `magic` 字段目前还没有使用，我仅仅是预留了这个位置
* `type` 表示矩阵元素的类型，遵循 ggml 的类型枚举
* `rows` 表示矩阵的行数
* `columns` 表示矩阵的列数
* `data` 存储矩阵的元素，它本身是 `void *`，我们通过 ggml 的类型宏，以 type 参数得到对应的类型 size 和转换逻辑
为了简化项目，这里我甚至没有用cpp，直接以c语言简单粗暴的实现了从二进制内存块加载 Matrix 模块的方法：
```c
int load_matrix(struct Matrix *matrix, void *buffer) {
    void *pos = buffer;
    // skip magic code check
    matrix->magic = 0;
    pos += sizeof(unsigned int);
    matrix->type = *(enum ggml_type *) pos;
    pos += sizeof(enum ggml_type);
    matrix->rows = *((size_t *) pos);
    pos += sizeof(size_t);
    matrix->columns = *((size_t *) pos);
    pos += sizeof(size_t);

    size_t items = matrix->rows * matrix->columns;
    size_t item_size = ggml_type_size(matrix->type);
    matrix->data = dalloc(items * item_size);
    memcpy(matrix->data, pos, items * item_size);
    return 0;
}
```
这样，我们就可以跳过在 PostgreSQL 中定义一个新的矩阵类型的工作，直接用 bytea 存储矩阵，我目前的首要诉求是从 4096 维的 ollama embedding 向量降维到 256 维，而一个 `256*4096` 的 float32 矩阵在 4.2 MB 左右，这个尺寸的 BLOB 对于 PostgreSQL，是完全能够处理的。
在跳过错误检查后，最终的矩阵乘法函数可以简化为：
```c
PG_MODULE_MAGIC;

PGDLLEXPORT PG_FUNCTION_INFO_V1(pgv_mulmv);
Datum
pgv_mulmv(PG_FUNCTION_ARGS) {
    bytea *a = PG_GETARG_BYTEA_P(0);
    Vector *b = PG_GETARG_VECTOR_P(1);

    char *data = a->vl_dat;
    struct Matrix *matrix = palloc(sizeof(struct Matrix));
    // todo assert status
    int status = load_matrix(matrix, data);

    Vector *result = InitVector((int) matrix->rows);

    // todo assert status
    status = mul_matrix_vector_f32(matrix, b->x, result->x);

    // pfree(matrix->data);
    pfree(matrix);
    PG_RETURN_VECTOR_P(result);
}
```
## 项目工程
这个子项目在 [tensor-dancer](https://github.com/MarchLiu/tensor-dancer) 的 `pgv_extra` 目录，它是 tensor dancer 项目面向 pgvector 的一个工具子项目，通过 meson 的子项目进行管理。

```meson
vector_extra = shared_module('vector_extra', 'matrix_lite.c', 'pgv_extra.c',
                             dependencies : [blas_dep, lapack_dep, pg_dep, ggml_dep, gettext_dep],
                             # cpp_args : lib_args + ['-DUSE_PG', '-I' + pgv_src_dir, '-I' + project_root_dir],
                             c_args : lib_args + ['-DUSE_PG', '-I' + project_root_dir])

load_matrix_lite = executable('mul-mv-lite', 'matrix_lite.c', 'mul_mv.c',
                              dependencies : [blas_dep, lapack_dep, ggml_dep, gettext_dep],
                              c_args : lib_args + ['-I' + project_root_dir])

install_data('vector_extra.control',
             'sql/vector_extra--0.1.0.sql',
             kwargs: extension_data_args,
)

all_build += [vector_extra, load_matrix_lite]
```
`all_build_`定义在项目根目录的 meson.build 中，目前还没有实际作用。
`pgv_extra` 子项目定义了 `vector_extra` 插件和一个作为测试程序的可执行文件 `mul-mv-lite` 。
同时，我编写了两个 python 脚本，放在项目根目录下的 `scripts` 子目录：
* `pca.py` 用于加载样本数据集，生成 pca 矩阵，将 pca矩阵、样本数据和测试用的验证结果保存在文件中，这个脚本用到了 numpy 和 sklearn ，后面我们单独用一个章节介绍这个工具
* `put_to_db.py` 用于将指定矩阵文件的内容写入到数据库，这个程序非常简单，仅仅用到了 psycopg2 ，它的主要内容是我用 AI 生成的。
目前我还没有做安装脚本，要运行这个插件，需要手工复制文件。在安装`vector_extra`之前，需要先安装pgvector，这部分不用多介绍了，pgvector本身非常的紧凑，直接用了pg官方的pgxs工具编写构建脚本，执行`make install`，然后在数据库中`create extension vector;`即可。 
现在我们回到`tensor-dancer` 项目目录，先构建程序：
# 这个构建配置是我在 macos 中，设定使用 homebrew 的 open blas 和
# lapack，将构建的中间环境保存在 build 目录

```shell
meson setup build \
	--pkg-config-path=/opt/homebrew/opt/openblas/lib/pkgconfig:/opt/homebrew/opt/lapack/lib/pkgconfig/:/usr/lib/pkgconfig:/usr/local/share/pkgconfig \
	--reconfig
cd build
ninja
```
这个编译过程会在`build/pgv_extra`目录下生成 `pgv_extra` 的动态链接库 `libvector_extra.dylib` 将这个文件复制到 `pg_config --libdir` 指向的库目录下的 `postgresql` 子目录，改名为 `vector_extra.dylib`。在我的电脑上，这个命令是
cp pgv_extra/libvector_extra.dylib /opt/homebrew/opt/postgresql@16/lib/postgresql/vector_extra.dylib
。然后将`pgv_extra`目录下的 control 文件`vector_extra.control`和 sql 文件`sql/vector_extra--0.1.0.sql`复制到  `\`pg_config --sharedir\`/extension/` 目录（在我这里是`/opt/homebrew/opt/postgresql@16/share/postgresql@16/extension/`）下。然后进入 psql ，执行
 create extension vector_extra;
即可安装。
## 在数据库中运行矩阵乘法
我在本机的 postgresql 中，建立了一个名为 pgv 的数据库，库里有两个表，一个是保存测试数据的 items：
```sql
create table public.items
(
    id        bigserial
        primary key,
    content   text,
    embedding vector(4096),
    target    vector(256)
);

create unique index idx_uni_items
    on public.items (content);
```
我在这个表中加入了两个vector 字段，可以用于比较降维后的数据质量。
我在 scripts 目录中保存了一个名为 `generate_and_save.py` 的脚本，用于读取指定目录中的所有文件，逐行写入 items表。这个脚本本身并不复杂，它简单的读取文件，访问本地的 ollama 服务，生成嵌入向量，写入数据库。后面我们可以另开一个章节介绍它。
另一个表用于存储矩阵数据：
```sql
create table public.matrix
(
    id      serial
        primary key,
    content bytea,
    meta    jsonb default '{}'::jsonb
);

create index idx_matrix_meta
    on public.matrix using gin (meta);
```
此时，我们可以回顾一下 `put_to_db.py` 中的 insert 操作，它将矩阵数据写入到 matrix 表的 content 字段。假设我们已经将一个转换矩阵保存到 id 为 1 的记录，并且在 items 表中已经有了大量数据的情况下，可以用这个sql进行矩阵计算：
```sql
with m as (select content from matrix where id = 1),
     v as (select embedding from items where id = 324532),
     result as (select pgv_mulmv((select m.content from m),
                                 (select v.embedding from v)) as v)
select (select vector_dims(embedding) from v),
       vector_dims((select v from result)),
       (select v from result) as result;
```
## `pgv_extra` 的未来
我计划将 tensor dancer 项目发展成一个完整的，基于 ggml 和 blas、lapack的张量计算框架，在postgresql中提供代数计算和AI支持。而`pgv_extra`仅提供对 pgvector 有用的部分操作接口，例如将vector结果集聚合为矩阵、将矩阵拆解为vector集合、在数据库内生成pca矩阵等函数。当然，即使限于这部分功能，这些算法依赖的基础工具对于原本的pgvector项目仍然过大了，所以我没有追求将这些代码提交到 pgvector 项目。
目前的 extra 插件仅仅是展示了基于矩阵的乘法计算能力，它还有一些工程工作需要做，例如安装脚本，例如对matrix和vector的边界检查，明确的matrix类型定义等。我会在后续的工作中逐步完善。