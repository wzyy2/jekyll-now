---
layout: post
title: 如何提交补丁给开源项目
category: linux
tag: CN
comments: 1
---

开源项目，社区，你国的程序员对这些东西可能都觉得很神秘，很酷，“哇，你提交过Kernel Patch，大牛啊”。
但其实一个开源项目里活跃的开发者，都是和我们一样的coder而已，just feel free to work with community。

一般情况下，大部分的linux相关开源项目模式都是差不多，git管理代码，然后会有很多的子项目maintainer，他们负责平时收集整理一般开发者的patch，然后按项目的开发周期定期发pull request到master branch或者他们上一层的maintainer。

所以做一个普通的开发者，我觉得大可不必关心这些项目乱七八糟的管理模式，甚至document里的如何提交patch也可以直接忽略掉。  
普通的开发者要做的很简单：   
找到这个项目里你要修改部分的dev分支作为code base，整理patches，发邮件给maintainer，修改补丁直到被接受。

下面就举几个例子。

# Linux Kernel

假设你已经有修改了5个commit，你想把这些修改发到上游。

首先，先把你的commit都生成patches：

	git format-patch -5 --cover-letter -v 2 -o outgoing/

5表示把从HEAD往后的5个commit变成patch，cover-letter表示生成一个封面，你可以在封面里添加你这个系列补丁的说明，-v 2表示这是第二个版本，因为补丁会重写很多次，-o outgoing表示patches存放的目录文件夹。

生成patches之后，最好先检查一下是不是有格式问题（其实这里可以直接用kernel里带的scripts/Lindent来帮你格式化代码）：

	./scripts/checkpatch.pl -f outging/*


确定没有问题之后，你要获取你修改文件的matainer以及相关的开发者，他们会负责review你的patch

	./scripts/get_maintainer.pl arch/arm/mach-rockchip -f

最后用下面的命令发出的你的邮件：

	git send-email outgoing/* --to ***@gmail.com --cc ***@rock-chips.com

其中cc改成你要发的matainer名字， to改成你要发往的mail list。

# U-Boot

U-Boot的补丁提交和内核其实差不多，只不过U-Boot有一个很方便的工具可以封装上面的命令，[“Patman”](https://github.com/siemens/u-boot/blob/master/tools/patman/README )。

同样假设有5个commit要发：

	./tools/patman/patman -c5

执行完上面的命令，补丁就发出去了。更多的用法，还是需要点击链接，根据readme里的描述来操作。

# Others

大部分的开源项目，用好git format-patch和git send-email就可以了。如果是github的，直接web端发pull-request也可以。  
对于mail list管理开发的项目，还有一个很重要的工具，就是patchwork，具体可以点击下面的链接。  
https://patchwork.ozlabs.org/，https://patchwork.kernel.org/

#### format

我的内核代码格式化命令, 只Lindent的话, 有些地方还是有缺点的

	~/workbench/debian-sdk/kernel/scripts/Lindent ./*
	astyle *.c  --style=knf --indent=force-tab=8
