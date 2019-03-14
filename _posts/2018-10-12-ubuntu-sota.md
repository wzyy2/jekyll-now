---
layout: post
title: 在自动驾驶车上使用Ubuntu
category: OS
tag: Autonomous Drive System
comments: 1
---

在自动驾驶的机动车上使用classic Ubuntu，是个很乱七八糟的事情。 ;-)
![](https://github.com/wzyy2/wzyy2.github.io/raw/master/images/ubuntu-crash.png)

但其实这也是很合理。  

在过去几年的工作里，我主要为一些行业客户提供GNU/Linux的支持。在系统的选型上，有几种选择提供给他们：Ubuntu/Debian发行版，Yocto， BuildRoot/OpenWRT，Busybox。因为平台的规格较高，应用场景相对复杂的缘故，所以主要集中前面两种选择上（后面两种主要专注于制作tiny os，类似家用路由器/行车记录仪等产品。  
我一直偏执的向他们推广Yocto，因为：
* “powerful customization architecture”，小到几mb的IoT OS，大到完整的Debian Desktop，都可以在这个框架下生成。
* "compile everything from source"，整个项目，从最底层的libc/gcc到最终运行的应用，都有比较清晰的管控。 

但是，除开一些比较专业的客户，会去选择Yocto以外，基本不吃我这套忽悠，Ubuntu/Debian发行版选择的比例更大。

# 1. Disadvantages

看下面这个Ubuntu为了推销Ubuntu Core所画的图会更加直观
![](https://github.com/wzyy2/wzyy2.github.io/raw/master/images/ubuntu.png)

可以看到classic的发行版是一堆package的集合，不像现在的移动端系统，没有层级的概念 —— 这直接导致了
* 应用软件无法快速和方便部署
    * 依赖基于系统，升级应用的同时需要对系统进行升级
    * 应用间共享依赖，升级单个应用会影响到其他应用
* 不具备SOTA/备份等功能
    * 没有用户分区/系统分区概念, 没办法区分擦除
    * 系统只有一份实例，不能在运行时升级自己
* 应用软件默认没有权限控制/隔离

# 2. Advantages


这么多区县级独角兽客户喜欢选择Ubuntu，肯定还是有原因的。

Ubuntu最主要的优势在于生态。  
比如说现在我们想要运行tensorflow，如果你使用的Yocto，那你要做的事情有
* 移植bazel到yocto
* 整理tensorflow的依赖到yocto
* 基于yocto编译tensorflow

对于Classic Ubuntu而言，你只需要输入`sudo apt-get install xxx`就可以安装了。

编译部署过后，代码还会存在版本适配的问题：lib1的版本A跑的正常，用版本B替换后出现问题。  
在Ubuntu上，有非常多的开发者和你用着同样的环境，这种问题基本已经暴露。

另外，很多缺陷也都是可以被容忍的。
* 硬件平台的规格越来越高，Gigabyte的存储需求对很多应用场景来说并不是问题
* 没有定制性无所谓，在没有几百人的OS Team前，没人想去动他们
* OTA是可以通过粗暴的方法实现的，比如说A/B分区，比如说snapshot
  
关于OTA，对有历史包袱的桌面系统而言，很多通过snapshot的方式来实现SOTA/备份等功能。虽然对端侧设备而言，这样的实现开销是比较大的：snapshot的copy-on-write特性导致系统体积增大。

![](https://github.com/wzyy2/wzyy2.github.io/raw/master/images/overlay.png)

# 3. Summary

|  缺点 |  |
| ------ | ------ |
| 臃肿 | Ubuntu/Debian系统标准安装下附带的软件包及其庞大，至少是Gigabyte计量 |
| 没有定制性 | [concept of a "distribution" of Linux](https://en.wikipedia.org/wiki/Linux_distribution)，发行版诞生之初的目的，是提供预编译/配置的应用程序库，让用户可以通过CD/网络等方式跑起需要的Linux程序 —— 但这种便利也意味着定制能力的缺失 |
| OTA | 结构上的历史包袱，大部分发行版并不能做到像Android这种移动端/IoT系统所支持的atomic system OTA |
| 维护成本高 | 系统以应用包为单位，应用之间耦合大 |
| 其他 | 专利/License风险 |


|  优点 |  |
| ------ | ------ |
| apt源 | 源码编译是个坑，不解释 |
| 生态 | ![](https://github.com/wzyy2/wzyy2.github.io/raw/master/images/ubuntu-support.png)   |
| 环境一致 | 开发环境和生产环境的一致性可以带来很大便利， 不然像以前所对接的客户都有专门的所谓rootfs工程师（类似运维？ |


# 4. 其他

## 4.1. Classic Linux Distribution

现在SBC(Single Board System)越来越流行，比如说Raspberry PI。
基于SBC，有大量开源的小伙伴们都在尝试classic发行版在这些平台的应用

* [rasbian](https://www.raspberrypi.org/documentation/raspbian/), 树莓派的操作系统
* [armbian](https://www.armbian.com/), 专门针对sbc的ubuntu/debian定制版本
* ......

虽然说端侧设备，大多是low-cost的方案，并不适用发行版。
但在storage越来越便宜的未来，还是有很大的空间的。


## 4.2. Container-like OS

对于现在的系统而言，实现层级的方式也有很多，除开像Android/ChromeOS这种从libc开始重写的，目前linux上比较流行的是更换package manager，采用containerize application/system的方式实现层级, 比如说rpm-ostree,flatpak,snap。

![](https://github.com/wzyy2/wzyy2.github.io/raw/master/images/resinOSarch.png)

这块用于我们的无人车，主要面对的问题是投入:
* containerize增加额外开发成本，不像classic Ubuntu, 直接apt/pip
* container到kernel之间过了一层containerd+runc， app的运行环境不能确认和host userspace一致

理想的情况：
![](https://github.com/wzyy2/wzyy2.github.io/raw/master/images/Architecture_OS-02.jpg)

但是，如下图，container内部要和GPU交互的话，可能还需要driver+containerd端的支持（也可能不需要?如果运行在privilege的模式下：
![](https://github.com/wzyy2/wzyy2.github.io/raw/master/images/dgx-docker-1024x970.png)

