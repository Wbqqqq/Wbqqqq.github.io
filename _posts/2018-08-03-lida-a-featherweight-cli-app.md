---
layout: post
title: "lida: 一个羽量级 cli 应用"
date: 2018-08-03 00:00:00.000000000 +08:00
tags: Develop
---
程序员总免不了要跟终端打交道，其中 `cd` 大概是我们敲得最多的终端命令之一，可大多数时候我们都需要键入三次甚至五次以上才能到达目的地，即使有 zsh 的帮助，有时可能也需要你费些功夫。

lida 是我用 ruby 写的一个简单的命令行应用，它可以让你在一些特定情况下摆脱 `cd` 的困扰，它的功能有：

```shell
# 打开 finder，并定位到现在终端的工作路径
lida 

# cd 到当前活动的 xcode 选中的文件夹目录、或正在编辑的文件的父目录中。pod install 的时候尤其有用。😄
lida xcode # 或者 "lida x"

# cd 到当前活动的 xcode-beta 选中的文件夹目录、或正在编辑的文件的父目录中。
lida xcodebeta # 或者 "lida xb"

# cd 到当前活动的 finder 中最前的窗口的文件夹目录。
lida finder # 或者 lida f
```

lida 的原理是使用 AppleScript 与 [Apple events](https://en.wikipedia.org/wiki/Apple_event) 进行交互，它现在支持 iterm 和 terminal 两种终端模拟器。

你可以使用 gem 来很方便地安装它！

```shell
gem install lida
```

lida 的源码在 https://github.com/jianstm/lida ，它是我的第一个 Ruby 开源项目，如果有什么想法，可以通过 issue 与我交流。🍻

---

人家都说懒才是生产力的进步动力，果然没错。👻

> ❤️ Ruby