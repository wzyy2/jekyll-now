---
layout: post
title: Gstreamer学习
category: [rockchip linux]
tag: [EN]
---

接触Gstreamer有一段时间了，但一直没有系统的了解过Gstreamer的结构，只是停留在”打补丁“这样的一种开发思维上。
虽然不是很喜欢gobject的风格，但毕竟开发不能总是没有头绪，还是需要系统的了解一下。

# gobject

gstreamer是基于gobject开发的一个库，所以学习gstreamer前，需要先了解下gobject相关的基础。

### 中文 ： https://www.ibm.com/developerworks/cn/linux/l-gobject/

一开始，可以先简单的看看这篇中文文章，对gobject有一个基础的认识。

### 英文 ： https://developer.gnome.org/gobject/stable/

在后面的学习过程中，如果有遇到不理解，可以再回来check官方的manual

# What is Gstreamer

![](https://gstreamer.freedesktop.org/documentation/application-development/introduction/images/gstreamer-overview.png)

https://gstreamer.freedesktop.org/documentation/application-development/introduction/index.html

这里的链接介绍了什么是gstreamer，他的结构是什么样。一般大家印象里，一个提供视频编解码的库，应该是ffmpeg那样，而gstreamer有点反这种印象。其实gstreamer不能说是一个库，而应该算是一种嵌入小程序。

关于gstreamer对使用者最主要的用处，见这段话:  
 > The application programmer who wants to build a media pipeline. The programmer can use an extensive set of powerful tools to create media pipelines without writing a single line of code. Performing complex media manipulations becomes very easy.


# Others

如果你的App不满足于media pipeline，你不会也不想写gstreamer plugin，那可以看看博客里的另一篇gstreamer+opencv，可以了解下如果使用appsink和appsrc，来达到类似ffmpeg输入输出的效果。