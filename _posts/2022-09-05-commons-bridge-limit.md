---
layout: post
title: "Apache Commons Lang3 Bridge 未（完全）实现的内容"
description: "实现这个封装库时，并非无条件的照搬，而是做了一定的取舍。"
category: 
tags: ["Scala", "Java", "Apache Commons"]
---
# Apache Commons Lang3 Bridge 未（完全）实现的内容

## 未实现的封装

* 跳过所有的 deprecated
* 所有的文本距离算法，现在这些算法都迁移到了 common-text 包，将来对该项目封装时再去处理
* 所有的 join 方法，这些方法在scala中意义不大，scala有干净漂亮的 mkString 方法
* 一些没有主谓宾结构的操作定义，例如 firstNonBlank/fistNonEmpty ，这两个方法相当于 SQL 中的 coalease 操作，所有的参数地位都平等，并不适合对象化
* 一些操作主语是 Char 类型的方法
* 有些操作主语是 CharSequence 的，先统一到了 String。虽然这回损失一些灵活性，但是第一版先集中实现 String 的扩展更务实一些。

## 未完全实现的部分

* char参数主要针对数组和相关的可变参数做了封装，对于大部分 Char 类型的参数，因为直接传值，不需要 Option 保护
* getIfBlank 等 get 和 default 开头的方法，返回值不是 `Option[String]`，而是 String ，因为它们是 get 方法，本身就是从 monad 中取出原始值的操作
* 计算方法，比如 indexOfxxx，都直接返回数值，common 自己的定义已经仔细考虑了空值问题
* 判定方法同理，直接返回 boolean而非 Option

## 虽然实现了但是并不推荐用

一个扩展的 length 方法看起来很奇怪，似乎也就是空值安全这么一个好处了，但如果是 `Option[String]`，getOrElse 方法似乎也还好。总之看起来不像是个会经常用到的东西。未封装的静态方法版本因为支持各种 CharSequence 实现（及其对应的null），反而看起来是个更有用的功能。
