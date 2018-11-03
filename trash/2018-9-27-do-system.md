---
layout: post
title: [OS]脑动：无人车
title: [OS]脑动：无人车
category: [OS]
tag: [CN]
---

Linux诞生于上个世纪90年代，到快接近30年，被广泛应用在个人桌面/服务器/嵌入式设备等场景，为这些设备制作/管理Linux System Image是一个比较成熟的事情。


# 什么是Linux系统镜像

什么是系统镜像： [wikipedia](https://en.wikipedia.org/wiki/System_image)

    a system image is a serialized copy of the entire state of a computer system stored in some non-volatile form such as a file.

## Typical Linux System Image Layout

Android
![](https://resinos.io/images/misc/resinOSarch.png)

ChromeOS
![](https://www.chromium.org/chromium-os/chromiumos-design-docs/disk-format/layout.png?attredirects=0)

Ubuntu in x86 platform
![](https://resinos.io/images/misc/resinOSarch.png)

典型的Linux系统镜像，会包括三个大部分
* BootLoader
* Kernel
* RootFS

在RootFS往后，一些系统会在中间加入一些组件，或者使用多个备份, 多个层级等，用于OTA/Recovery等功能。

<small>Docker Image并不是一个System Image。因为Docker Image只有RootFS（文件系统</small>

## Boot Flow

光是看Disk Layout，会不太容易理解每个部分的作用，可以结合整个系统的启动流程来看。






# 如何制作Linux系统镜像

## 现有的镜像制作手段

| Elements       | Type  |  Comments  | 
| :----:  | :----:  | :----:  | 
| mppvideodec    | Video Decoder | h264, h265, jpeg, vp8, vp9 |
| mpph264enc        |   Video Encoder    | h264   | 

# 怎么为无人车制作系统镜像

Full feature的Ubuntu
传统的Ubuntu是如何制作镜像的un？
为什么不用Ubuntu Core?
有用Ubuntu Core的精力的话，早就可以用Yocto/Portage等工具，走完全自己定制的系统了

### Idea：ContainerOS


* 1980: https://wiki.debian.org/DebianLive


![](https://resinos.io/images/misc/resinOSarch.png)