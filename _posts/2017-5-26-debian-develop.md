---
layout: post
title: Linux系统开发随笔
category: [CN]
---


在Rockchip Linux平台开发的一些个人意见。


# 网络资料

我们的wiki：
* http://opensource.rock-chips.com/wiki_Main_Page

比较有用的guide，
* http://opensource.rock-chips.com/wiki_Linux_user_guide
* http://opensource.rock-chips.com/wiki_Yocto
* http://opensource.rock-chips.com/wiki_Cross_Compile

如果要搭建编译环境， 一定要看*Linux user guide*。  
如果不是具体的项目应用开发，rootfs选debian就可以。

# 系统环境
系统上，推荐是[“ubuntu 16.04”](https://www.ubuntu.com/download/desktop)或者debian9 lxde。
如果发行版版本过低，会造成使用的软件版本低，比如gcc版本，binutils版本等等，容易发生编译问题。

最好是有一台local的linux host，至少也应该保证有一个linux虚拟机！


# 固件

for rk3288-evb-lpddr3-act8846:  

https://pan.baidu.com/s/1eSkpqQe


## 烧写方法

*Linux user guide*有详细文档，不解的话再看下面。

### Linux

如果是虚拟机，会比较麻烦，一定要确认usb设备挂载在*虚拟机*！.
根据[“Linux user guide”](http://opensource.rock-chips.com/wiki_Linux_user_guide)，
在板子进入*maskrom*以后，使用以下命令烧写完整镜像，这条命令本质是先通过rkdeveloptool下载我们的loader到板子内存里运行，
然后再通过loader把firware.img写到偏移为0的地址，system字符串代表的偏移地址在[“这里”](https://github.com/rockchip-linux/build/blob/debian/partitions.sh#L10)定义：

        build/flash_tool.sh -p system -i firmware.img

### Windows

[“AndroidTool”](https://github.com/rockchip-linux/rkbin/tree/master/tools/AndroidTool_Release_v2.39)
这个工具就是根据偏移写固件，和linux上的是一样的。

Window下还有有一个[“量产工具”](https://github.com/rockchip-linux/rkbin/tree/master/tools/ProductionTool)
，专门用来烧写完整固件，就是写镜像到偏移地址0，但是看起来会对gpt表做一些操作，所以烧完后，要手动gpt write下。

### Tips

* linux上的烧写全是要进入maskrom完成的，因为uboot不支持rkusb协议，要使用这个，需要back to maskrom
* 本质上我们linux没有严格限制的分区，不存在什么地方一定要烧到那里，但为了方便统一我们也定了一个[“分区表”](http://opensource.rock-chips.com/wiki_Partitions)。
* 所有烧写，实际都不是根据这个*GPT分区*来的，而是通过指定存储的*偏移地址*烧的, Window工具和Linux工具最终用的都是偏移。之所以烧写的时候看到什么boot，loader参数，是因为提前根据分区表约定了好这个地方是干嘛。所以烧的时候要灵活，比如window工具烧完整固件，把偏移地址改成0就好了。
* 除了指定偏移地址烧写，我自己开发的时候也比较多的依赖*uboot*下的*ums*模式（guide里有介绍），本质就是通过uboot把设备模拟成u盘，这样可以在电脑上像u盘一样操作板子，拷贝资料，替换内核等等。
* 内核和dtb实际是没有任何打包的，我们烧写的时候出现一个boot.img，实际上那是一个fat镜像，然后把zimage和dtb拷了进去，然后方便我们通过偏移地址直接写。如果是ums模式，直接 `cp out/kernel/* /media/user/boot` 就可以了。
* 注意GPT分区表！ gpt分区表分布在0x00的地方，如果一开始我们不是烧的完整镜像，而是分开烧的，会缺失GPT表，需要在uboot下`gpt write mmc 0 $partitions`，或者rkdeveloptool也可以写这个gpt。


# 开发

## 内核编译

见[”Linux user guide“](http://opensource.rock-chips.com/wiki_Linux_user_guide)

## 本地编译

如果要编mpp和gstreamer，在板子上编比较容易，缺什么apt什么，和电脑一样。

## 交叉编译

本地编译很慢，方便的话还是交叉编译。

交叉编译的原理是利用debian系的multiarch机制，不过这种办法需要有一个与目标机一致的系统。

[”Docker“](https://github.com/rockchip-linux/docker-rockchip) 可以解决这个问题，而且[”dockerfile“](https://github.com/rockchip-linux/docker-rockchip/blob/master/dockerfile)里可以提前配置相关的环境。
一般使用docker的workflow是:

进docker bash，还有挂上源码目录：

        docker run -it -v ~/workbench/debian-sdk/packages:/home/rk/packages rockchip /bin/bash

在docker里进入应用目录

        cd /home/rk/packages/mpp

编译（实际上这条命令就是根据写好的编译规则调用相应的cmake，make这些） （加 -nc 可以不clean）

        DEB_BUILD_OPTIONS=nocheck debuild  -i-us -uc -b -d  -aarmhf

编译完成后，~/workbench/debian-sdk/packages里就会出现对应的deb,可以dpkg -i安装。

编译选项可以在rules文件里配置：  
https://github.com/rockchip-linux/mpp/blob/for_linux/debian/rules

## 传文件

我个人比较喜欢sshfs。

配一个路由器，让板子和电脑在一个网段上,然后写一个这样的脚本在板子上

    sshfs chen@192.168.31.163:/ ~/host                                                                                                     
    cd ~/host/home/chen/workbench/debian-sdk/packages/

每次开机只要`source 脚本`，就可以把自己的pc机挂在板子上，测试起来很方便。


# 服务器
服务器都是挂在vpn上的，为了防止网络问题。
不过我们使用都比较少。

jenkins（基本没用，废弃了）：  
http://172.16.22.122:8080/ 

yocto toaster（用来检测YOCTO的构建，如果想下image的可以点第二个网址）:  
http://172.16.22.122:8000/toastergui/builds/?limit=25&page=1&orderby=-completed_on&default_orderby=-completed_on&  
http://172.16.22.122/data/poky 
