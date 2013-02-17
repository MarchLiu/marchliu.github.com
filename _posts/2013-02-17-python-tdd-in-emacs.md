---
layout: post
title: "Python TDD IN EMACS"
description: "Emacs有非常强大的扩展能力，可以用它和iPython组合成为Python的TDD式开发环境。"
category: tech
tags: [Emacs, Python, IPython]
---
{% include JB/setup %}

Emacs 是著名的万能编辑器，它同样可以用来建立一个非常强大的Python开发环境。

其实很多 Python 程序员都用 Emacs，有很多不同的组合方式。这里，我们用 iPython 建立一个可以 TDD 的组合。

[iPython](http://ipython.org) 是一个著名的 Python 扩展环境。通常我们将它视作一个增强的Python Shell。但其实它的能力远不止如此。在其官网上，这样介绍 iPython：

- Powerful Python shells (terminal and Qt-based).
- A web-based notebook with the same core features but support for code, text, mathematical expressions, inline plots and other rich media.
- Support for interactive data visualization and use of GUI toolkits.
- Flexible, embeddable interpreters to load into your own projects.
- Easy to use, high performance tools for parallel computing.

这里，我们也仅仅用到它的一小部分功能：用它代替默认的Python环境，嵌入到 Emacs 中，作为Emacs的内置Python Shell；以及实现交互式的开发-测试过程。

## 在 Emacs 中嵌入 iPython

要在Emacs中嵌入 iPython，需要一个名为 python-mode 的插件。我曾经被这个名字误导，误以为是 Emacs 内置的 python-mode 支持。其实它是[这个](https://launchpad.net/python-mode) 。不过我曾经试过在我的工作机上用直接下载的版本配置失败，最终我找了一个可用的版本，冻结在了[usemacs](https://bitbucket.org/yinwm/usemacs/src/11b8fb4abd69c9c394c9b537b137cae844b6eb10/raw-elisp/march/macos/site-lisp/python-mode.el?at=default) 项目中。

最终，我用这样一组代码调用它。这个路径是因为我的 MAC OS 系统中用 mac ports 安装了一个 Python 2.7 。如果你的设置不同，请自行修改。

~~~
(load-library "python-mode")
(setq ipython-command "/opt/local/bin/ipython")
(setq py-python-command "/opt/local/bin/python2.7")
(setq py-python-command-args '("--colors" "LightBG"))
(require 'ipython)
~~~
{: .language-lisp}

这样，当你输入 M-x py-shell，得到的是一个 ipython shell 。

![py-shell](/images/py-shell.png)

它当然，仅仅是这样，它已经很有用，我在编写 Python 代码的时候，经常想要在 Python Shell 里运行一些东西，看看结果，验证一些想法。更不用说 iPython 还可以用来执行系统 Shell。不过它结合一些编码风格和工具操作，还可以更有用。

## Python 的TDD

Python 有两个内置的 testing 模块，doctest和unittest。unittest 是个比较正规的东西，有很多种不同的编程语言的实现。这里不做赘述了。我们说说 doctest。关于它，官方有一个[详细的文档](http://docs.python.org/2/library/doctest.html)。它鼓励程序员一边写代码，一边把验证代码的测试逻辑写下来，而这些代码还可以作为 docstring ，打包在模块中发布，供以后阅读。虽然写起来麻烦点（需要一些啰里巴嗦的修饰字符，是每行），还是很值得做的。

如果你的 Emacs 环境已经配好了，现在你的 Python 代码保存以后，可以随时按 C-c C-c 执行它，并且，执行结果会返回到你的 py-shell 中。所以，只要在业务代码的最后加这么一行

~~~


if __name__ == "__main__":
    import doctest
        doctest.testmod()

~~~
{: .language-python}

你就可以在编码过程中随时按下 C-c C-c ，验证你的代码了：

![doctesting](/images/doctesting.png)

## 其他配置

还有一些配置，我曾经用过，或者是使用过，它们提供了重构、折叠等功能。

### 重构

Emacs 的 Rope 插件支持python 代码重构，它需要安装 pymacs 。我没有真正用过这个插件，不过有兴趣的朋友可以试下，从我的试用体验来说，比 cedet 要靠谱一些。简单的配置如下：

~~~
(add-to-list 'load-path "~/.emacs.d/site-lisp/pymacs")
(require 'pymacs)
(pymacs-load "ropemacs" "rope-")
~~~
{: .language-lisp}

### 代码折叠

这个代码是我参考了同行的做法己尝试出来的，后来在mac上python代码写的少，目前就没有激活。在hook 中还有缩进设置。从经验来说，python的缩进最好用4空格。

~~~
;; 代码折叠设置来自 gb@cs.unc.edu， 感谢他。
(add-hook 'python-mode-hook 'python-mode-hook t)

(defun py-outline-level ()
  (let (buffer-invisibility-spec)
    (save-excursion
      (skip-chars-forward "\t ")
      (current-column))))

(defun python-mode-hook ()
  ; this gets called by outline to deteremine the level. Just use the length of the whitespace
  (custom-set-variables
   '(indent-tabs-mode nil)
   '(tab-width 4)
   '(tab-stop-list nil)
   )
   ; outline uses this regexp to find headers. I match lines with no indent and indented "class"
  ; and "def" lines.
  ; 这里我利用了 Martin Sand Christensen 提供的正则表达式，感谢他。
  (setq outline-regexp "[^ \t]\\|[ \t]*\\(def\\|class\\|if\\|elif\\|else\\|while\\|for\\|try\\|except\\|finally|with\\) ")
  ; enable our level computation
  (setq outline-level 'py-outline-level)
  ; turn on outline mode
  (outline-minor-mode t)
  ; make paren matches visible
  (show-paren-mode 1))
~~~
{: .language-lisp}

### cedet

[cedet](http://cedet.sourceforge.net/) 现在已经是 emacs 的内置插件。它有一整套完整的功能，特别是安装了它的组件包ECB以后，很有点eclipse的味道。支持包括C/C++，Python，Java在内的很多语言。不过我是好几年没有用这么重的包啦。何况把它调教顺了也是个看运气的事情。

### yasnippet

[yasnippet](http://code.google.com/p/yasnippet/) 是个相当好用的模板插件，扩展起来比 cedet的同类功能要简单的多。

