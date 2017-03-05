---
layout: post
title: Yocto workflow
category: [Yocto, CN]
---

Yocto 还算一个蛮新的东西，可能很多人都不太熟悉。我最近在看的时候，资料都挺难找的，虽然官方的 documents 很全，但是对入手的人很不友好，
总结起来就是缺少一份入门的 guide。能找到的大部分 guide，在教你输入 bitbake core-image-base 后就没内容了。
然后呢？我能用 yocto 干什么，我该怎么用 yocto 去做这些事情？还是需要很长的时间去整理和提炼。


# 开发内核

当你执行过一次build之后，你可以在 `tmp/work-shared/<MACHINE_NAME>/kernel-source` 下面找到内核的代码，只要不执行 clean，这里的代码是一直存在的，可以放心的编辑。

修改完后可以执行下面的命令去进行编译：  

	bitbake linux-rockchip -c compile -f

上面这步只是编译而已，下面还需要烧写内核的话，分两种场景。
### 烧写完整镜像

这个时候你只要去 build 你需要的 image 就可以了，bitbake 会依次执行 kernel 的 deploy 和 image 的 generate，最终 deploy/images 目录下面生成的完整镜像会包含新编译的kernel。

	bitbake IMAGE_NAME
	# flash image here

### 只烧写内核
在开发一些比较庞大的项目的时候，不可能说每次都烧写几个 Gb 大小的 image，这个时候就要单独烧写内核。  
要注意 compile 步骤只是执行编译而已，编译完内的 binary 还呆在 work 目录下的 build dir 里的，所以你需要执行下面的命令：

	bitbake linux-rockchip -c deploy -f

执行完后 deploy/images 目录下面的 kenrel image 和 dtb 就会被更新，然后你才能通过 fastboot 或者 ums 一类的办法烧写至板子上。


# 开发应用

暂无

# 离线工作

默认用 yocto 有一个问题，就是必须要保证网络在连接状态。如果想离线工作，还需要做一些配置

离线工作，分下面两个场景：

### 场景一：
你有一个 source 是从内网服务器获取的，然后你离开了内网环境，或者 source 是 github 上的而你电脑断开了网络。  

这种场景你执行 bitbake，大概率会发现 yocto 报 network unreachable。  
这个错误的原因是因为我们设置了 SRCREV = "${AUTOREV}"，导致 bitbake 的时候虽然本地已经有了仓库，但他会去尝试连接 git 服务器，看看是不是有更新，如果这个时候无法连接上 git，就会报错。  
解决办法就是把 SRCREV 固定为最新的 commit id，这样只要你不执行 clean，bitbake 就不会去尝试连接 git 服务器，就可以保持离线工作。

	SRCREV = "75dc14fa97df26fbb0638ccd547a01ee1632bad6"

### 场景二：
一个东西到了产品化阶段，我们想固定下所有的代码。

这个通过设置本地的 mirror 来完成，具体参考网络的资料。


[“How do I create my own source download mirror”](https://wiki.yoctoproject.org/wiki/How_do_I  )

注意这个场景也不能使用 SRCREV = "${AUTOREV}”，需要有固定的代码位置。根据网络的资料，可以通过 buildhistory-collect-srcrevs 来帮助获取当前的rev，但我也没用过，有兴趣的可以在自己 google 下。



# 更新代码

不是清楚，暂时用下面的办法更新代码

	bitbake linux-rockchip -c clean && bitbake linux-rockchip

# Debug

默认编出来的 yocto image 不包含任何的 debug symbol 和 debug 工具，不是很方便开发。不过想要安装这些工具，也非常容易。

	EXTRA_IMAGE_FEATURES = "tools-debug tools-profile tools-testapps dbg-pkgs  debug-tweaks"
	IMAGE_INSTALL_append = “ sshfs-fuse”

一般我都会把上面这两句加到 local.conf 里，这样的 image 就非常方便使用，可以用 sshfs 拷贝文件，可以有 gdb 去debug 应用。
具体这些的含义，请点击下面的链接：
[“Image Features”](http://www.yoctoproject.org/docs/1.8/ref-manual/ref-manual.html#ref-features-image)


# 元包开发

Yocto 的开发，他的文档已经很全了，不需要多写什么，下面就提炼一点小tips，节约获取信息的时间。
