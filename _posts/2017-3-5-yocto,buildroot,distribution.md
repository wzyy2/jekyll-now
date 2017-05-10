---
layout: post
title: 嵌入式Linux的选择
category: [CN]
---


在嵌入式或者移动系统上， Android 一直是保持一个持续增长的份额，特别是在一些比较功能需求比较复杂的场合，比如说各种消费类电子，传统的一些其他 OS 基本上只能做 low cost 的事情，比如说起个busybox，跑一些小型服务。  
但是 Google 借着 Android 获取了这种垄断地位， 必然引起其他商业公司的应对，不管是系统商，设备商，还是半导体，都推出了自己的嵌入式系统方案。这些初生方案的单凭自己没办法是没办法对抗Android的，投入的资源不够覆盖整个系统，所以他们都要基于一些已有方案来做 ———— GNU Linux就出场了。  
这些年消费电子新跑出来的 OS， 比如说 Tizen, Ubuntu touch, MeeGo 这些，基本上都是基于 Gnu Linux 来做的。在这个过程中，GNU Linux的下一代组件， 比如说wayland， 比如说 systemd， 不再仅仅是面向 Desktop 或者 Server 的了， Mobile 变成了一个很重要的目标， 移动端的系统厂，半导体厂也都越来越积极参与。
另外，像 QT 这样的开发库也受益很多，现在你可以直接在 GNU Linux 的系统上画出 material design 的 UI， 不用再忍受 Desktop 风格。  
Linux 除这些消费电子上体验的改进，他现在越来越垄断的服务器生态优势，也非常方便你在嵌入式端集成 Web 相关的服务。  


说了这些废话，就是想说，Linux 端的应用开发已经变得和Android一样重度，对于开发者来说，已经不太可能自己控制整个系统，你需要依靠一些已经制作好的系统。

* Desktop distribution （ubuntu， debian）
* Yocto
* Buildroot

看到这三个选项，其实就可以看出来我是推销 Yocto 的了。这三个不是对等的概念，但可以是三个不同的 choise。从概念上讲， Yocto 包括了 distribution 和 Buildroot。 Buildroot 是构建工具，distribution 是系统， Yocto 则是包括了构建工具和系统的一个 project。


# Distribution

Distribution 就是我们日常在 Linux Host 上使用的哪些系统， 都应该比较熟悉。  
想一下，在一个 ubuntu 上，你想要安装 python 的环境，需要怎么做？`apt-get install python`，换成 Yocto 和 Buildroot呢？

如果你基于 Distribution 来开发你的应用，事情会很简单，因为你整个环境的配置，只要从网络源下已经编译和配置好的二进制包就可以了。（不要说 genntoo 之类的异类。。。。）我想要 qtmultimedia，我想要 python，你不用去管 qtmultimedia 有哪些依赖，python 编译要做什么配置，这些发行版下面上千个开发者已经帮你做好了配置，已经打好了包，上万的用户也已经替你测试过。

这样的好处很明显，你节省了不必要的资源，比如你在开发一些重度的 Web 应用，基于 python， nodejs 的后端，配置nginx， iptables 一类的，你要想从 Buildroot 和 Yocto 来自己做，恐怕你得准备一个几十人的 team。

缺点也很明显，没办法定制，就带来了冗余和缺损。冗余的体现就是更大的体积，比如 Xserver，默认你找到的发行版，他们的 Xserver 都会开启 selinux， glx这些嵌入式系统上根本用不到的功能，体积会更大。一个包大一点，到最后整个系统就会比如 Yocto 做出来的大很多了，同样的功能，可能Yocto做出来可以到 500 Mb， 而 Debian 就要 1 个 G。缺损的体现就是性能损失，没有做到最优的配置，比如QT， 默认发行版的都没有开启 gles 的支持，默认都是使用 gl， 而嵌入式系统上使用的都是 gles，这就导致你在发行版上运行 Qt 的时候，UI都是通过软件的 gl 来完成，而没走硬件的 gles，性能出现了损失。



# Buildroot

Buildroot is a simple, efficient and easy-to-use tool to generate embedded Linux systems through cross-compilation

使用 Buildroot 的 project 最出名的就是 Openwrt了。可以看到，他制作出来的镜像可以跑在搭载16 Mb spi nor的路由器上，系统基本没包含多余的东西。
这就得益于 Buildroot 的原则， simple。

整个 Buildroot project在一个git维护。
https://git.busybox.net/buildroot/

使用kconfig和make，一个配置代表一种bsp支持
https://git.busybox.net/buildroot/tree/configs/beaglebone_defconfig

Buildroot本身不具备扩展能力，用户需要自己通过脚本来完成工作。

这些列出来的特点，都是和 Yocto 不同的地方，下面我们会对比。

# Yocto

Yocto 和 Buildroot 一样， 是一套构建嵌入式系统的工具，但是他们两个的风格完全不同。

Yocto project 是通过一个个单独的包（meta）来维护，比如有的包负责核心，有的负责外围。有的包是为了你跑 Rockchip 的芯片，有的包是让你安装上 QT, 有的包是则是让你跑起一个 debian，
这个特点是我最喜欢的地方，同样采用类似机制的 nodejs，社区膨胀非常厉害，活跃度很高，每个人都分享自己低质量垃圾包到 github 上。。。。这样的机制保证了我们可以从互联网复用别人的工作成果，还是很好用的。

Yocto 有一个非常灵活的构建系统，可以允许你使用 shell 和 python，可以处理各种特殊情况。
