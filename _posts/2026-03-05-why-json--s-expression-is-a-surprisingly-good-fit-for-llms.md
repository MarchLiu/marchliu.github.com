---
layout: post
title: "Why JSON + S Expression is a Surprisingly Good Fit for LLMs"
description: ""
category: 
tags: ["AI", "LISP", "JSON"]
---

# 讨论总结（JSE设计核心）

从你给出的 spec 和 HN 贴可以总结出几个关键设计思想：

### 1. JSE 的核心目标

JSE（JSON Structural Expression）的目标是：

> 在 **完全保持 JSON 合法性** 的前提下，让 JSON 可以表达 **S-expression 风格的结构逻辑**。

换句话说：

```
JSON = data
S-expression = structure / computation
JSE = JSON + symbolic structure
```

JSE 并不是新的语言，而是：

**JSON 上的一层结构协议。**

---

### 2. 为什么需要它

LLM 已经非常擅长生成 JSON，但现实问题是：

JSON 只表达 **数据**，而 AI 经常需要表达：

* 推理步骤
* tool pipeline
* DSL
* query language
* agent plan
* transformation tree

现在常见解决方案：

| 方法                 | 问题     |
| ------------------ | ------ |
| ad-hoc JSON schema | 每个系统不同 |
| tool calling       | API绑定  |
| code string        | 不安全    |
| YAML DSL           | 不稳定    |

JSE试图提供：

**统一的结构表达方式。**

---

### 3. 最关键的设计：`$Symbol`

JSE 的核心语法规则只有一个：

```
"$something"
```

表示 **Symbol**。

例子：

```
"$add"
"$if"
"$call"
```

而：

```
"$$add"
```

表示普通字符串 `$add`。

这样解决了：

**数据 vs 操作符 的歧义问题。**

---

### 4. 两种表达形式

JSE只有两种表达式：

### Array form（最像 Lisp）

```
["$add", 1, 2]
```

等价于

```
(add 1 2)
```

---

### Object form（支持 metadata）

```
{
  "$if": [...],
  "confidence": 0.98
}
```

好处：

* 表达结构
* 同时附带 metadata

---

### 5. `$quote`

这是一个非常重要的设计。

因为 JSON 数据里可能出现 `$something`：

```
{
  "$raw": "data"
}
```

为了避免被解释为 expression：

```
["$quote", {...}]
```

---

### 6. JSE 的哲学

JSE **不定义执行语义**。

它只定义：

```
structure
```

解释逻辑由实现决定。

---

# 博客文章（完整版）

下面是我为你整理的博客文章草稿。

---

```markdown
# Why JSON + S-Expression is a Surprisingly Good Fit for LLMs

Large language models have become very good at generating structured data.

In practice, this usually means **JSON**.

Almost every modern AI API now supports some form of:

- JSON output
- JSON schema
- structured responses
- tool calling

However, something interesting happens once you start building more complex AI systems.

JSON works extremely well for **data**.

But AI systems often need to express something slightly different:

- reasoning steps
- tool pipelines
- transformation trees
- DSL-like instructions
- agent plans

These are not just data.

They are **structure**.

And historically, one representation has been extremely good at expressing structure:

**S-expressions.**

---

# A Quick Reminder: Why S-Expressions Work

S-expressions (popularized by Lisp) represent computation using a simple rule:

```

(operator arg1 arg2 ...)

```

Example:

```

(add 1 2)

```

or

```

(if (> 5 3)
(add 1 2)
(sub 4 1))

```

This representation has several remarkable properties:

- completely uniform
- trivially parseable
- naturally recursive
- very easy to generate programmatically

In fact, many AST representations internally look almost exactly like S-expressions.

---

# The Problem: LLMs Don’t Generate Lisp Well

While S-expressions are elegant, they are not ideal for LLM outputs.

Models today are trained heavily on:

- JSON
- YAML
- markdown
- natural language

But not much on raw Lisp syntax.

In practice, asking models to output Lisp-like syntax often leads to:

- parenthesis mismatches
- formatting instability
- token ambiguity

JSON, on the other hand, is **extremely stable** for LLMs.

Models generate JSON surprisingly reliably.

So an obvious question emerges:

> What if we combine the structural power of S-expressions with the reliability of JSON?

---

# The Idea: JSON Structural Expressions (JSE)

JSE is a small convention that allows **S-expression style structures to live inside valid JSON**.

Instead of writing:

```

(add 1 2)

````

we write:

```json
["$add", 1, 2]
````

The rule is simple:

* strings starting with `$` are **symbols**
* arrays whose first element is a symbol represent **expressions**

So:

```json
["$add", 1, 2]
```

means

```
(add 1 2)
```

while

```json
[1,2,3]
```

is still just normal JSON data.

---

# Two Expression Forms

JSE supports two structural forms.

### 1. Array Form

The closest equivalent to Lisp.

```json
["$add", 1, 2]
```

Example:

```json
["$if",
  ["$gt", 5, 3],
  ["$add", 1, 2],
  ["$sub", 4, 1]
]
```

---

### 2. Object Form

Objects allow metadata to coexist with expressions.

```json
{
  "$if": [
    ["$gt", 5, 3],
    ["$add", 1, 2],
    ["$sub", 4, 1]
  ],
  "confidence": 0.98
}
```

Here:

* `$if` defines the operator
* `confidence` is metadata

This pattern turns out to be extremely useful for AI systems.

---

# Escaping and Data Safety

A challenge appears immediately.

What if we want a literal string that starts with `$`?

JSE solves this with a simple escape rule:

```
$$
```

Example:

```
"$$add"
```

means the literal string `$add`.

JSE also defines a `$quote` operator:

```json
["$quote", {...}]
```

which prevents interpretation of the contents.

This ensures that **raw JSON data can always be preserved safely**.

---

# Why This Works Surprisingly Well for LLMs

After experimenting with this structure, several interesting properties emerge.

### 1. JSON Stability

LLMs already generate JSON reliably.

JSE never breaks JSON validity.

---

### 2. Deterministic Parsing

The interpretation rule is extremely simple:

1. Array + first element is Symbol → expression
2. Object + exactly one Symbol key → expression
3. Otherwise → data

No ambiguity.

---

### 3. Natural AST Representation

Most compilers internally use something like:

```
Node(op, args)
```

JSE maps directly onto this.

---

### 4. Compatible With Existing Systems

Systems that don't understand JSE can still treat the data as **normal JSON**.

This property is very powerful.

It allows gradual adoption.

---

### 5. LLM-Friendly Syntax

LLMs tend to do better when:

* syntax is repetitive
* structure is explicit
* tokens are predictable

JSE has all three.

---

# Example: AI Tool Pipeline

A reasoning pipeline might look like this:

```json
["$pipeline",
  ["$search", "JSON S-expression"],
  ["$summarize"],
  ["$translate", "zh"]
]
```

Or with metadata:

```json
{
  "$pipeline": [
    ["$search", "JSON S-expression"],
    ["$summarize"]
  ],
  "confidence": 0.91
}
```

---

# What JSE Does *Not* Try to Do

JSE intentionally avoids defining:

* execution semantics
* standard operator sets
* evaluation rules

Those are left to the implementation.

JSE only defines **structure**.

---

# Possible Use Cases

Some areas where this structure may be useful:

AI orchestration systems

Agent communication protocols

Structured reasoning traces

Prompt-embedded DSLs

Cross-model communication formats

---

# Final Thought

S-expressions solved the problem of representing computation in the 1960s.

JSON solved the problem of representing data on the web.

Large language models now sit somewhere in between:

they need to express **structured intent**.

JSE is a small experiment in combining the two.

It may turn out that the simplest bridge between **data and computation** for AI systems is just:

> **JSON + S-expressions.**

```
