---
layout: post
title: Yocto workflow 
category: [CN]
---

Yocto 还算一个蛮新的东西，可能很多人都不太熟悉。我最近在看的时候，资料都挺难找的，虽然官方的 documents 很全，但是对入手的人很不友好，
总结起来就是缺少一份入门的 guide。能找到的大部分 guide，在教你输入 bitbake core-image-base 后就没内容了。
然后呢？我能用 yocto 干什么，我该怎么用 yocto 去做这些事情？还是需要很长的时间去整理和提炼。


# 开发内核

当你执行过一次build之后，你可以在 `tmp/work-shared/<MACHINE_NAME>/kernel-source` 下面找到内核的代码，只要不执行 clean，这里的代码是一直存在的，可以放心的编辑。

修改完后可以执行下面的命令去进行编译：  

	bitbake linux-rockchip -c compile -f

如果修改的是configure，改为下面的命令  

	bitbake linux-rockchip -c menuconfig

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

目前修改应用我和修改kernel差不多用法，但是 yocto 的 document 里还介绍了一些工具，如 devtool，Quilt，等待研究。

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

这个可以通过设置本地的 mirror 来完成，具体参考下面的网络资料。


[“How do I create my own source download mirror”](https://wiki.yoctoproject.org/wiki/How_do_I  )

注意这个场景也不能使用 SRCREV = "${AUTOREV}”，需要有固定的代码位置。根据网络的资料，可以通过 buildhistory-collect-srcrevs 来帮助获取当前的rev，但我也没用过，有兴趣的可以在自己 google 下。


# Debug

默认编出来的 yocto image 不包含任何的 debug symbol 和 debug 工具，不是很方便开发。不过想要安装这些工具，也非常容易。

	EXTRA_IMAGE_FEATURES = "tools-debug tools-profile tools-testapps dbg-pkgs  debug-tweaks"
	IMAGE_INSTALL_append = “ sshfs-fuse”

一般我都会把上面这两句加到 local.conf 里，这样的 image 就非常方便使用，可以用 sshfs 拷贝文件，可以有 gdb 去debug 应用。
具体这些的含义，请点击下面的链接：
[“Image Features”](http://www.yoctoproject.org/docs/2.3/ref-manual/ref-manual.html#ref-features-image)


# Others

Yocto 的开发，他的文档已经很全了，不需要多写什么，下面就提炼一点小tips，节约获取信息的时间。

### 指定任务
bitbake 可以指定只运行一个任务，比如

	bitbake linux -c compile

他就只会执行compile命令。这里要注意如果不加 -f 的话，命令有检查完 stamp 之后就直接 skip了。  
常见的一些操作有，fetch， unpack， configure， compile， install， deploy等等，对应的是更新download，更新workdir下的source，配置参数，编译，安装，打包这些步骤。

### 覆盖文件
覆盖文件有两种，一种是通过 bappend 来修改提供这个文件的包，这种修改比较适合 bsp 包，有普遍性的修改。第二种就是通过  update-alternatives 来设定不同包的优先级，优先级最高的文件会覆盖进去。

### 打印变量

Yocto 里有很多什么 IMAGE_FEATURES 啊 DISTRO_FEATURES 啊， 因为很多层文件都有修改这些变量，所以有时候会不确定这些变量具体的值。
之前找的资料显示可以用 bitbake -e， 但是这个不是很好用。  
其实可以在 do_install 加一句 echo， 输出这个变量，bitbake 加 -v 就可以看到echo的内容了，比 -e 要容易看懂一点。

### clean

执行 clean 会把 workdir 里的东西都删掉，执行 cleanstate 会多一个 shared cache，执行 cleanall 还会把 download 删掉。

### 更新代码

不是清楚，暂时用下面的办法更新代码

	bitbake linux-rockchip -c clean && bitbake linux-rockchip

### 查看镜像里安装的包

Yocto 有一套乱七八糟的包依赖管理，所以到后面我们会不确定自己的镜像上到底装了些什么东西，这个时候就需要一些命令去把信息打印出来。

	 bitbake -g cvr-image

执行完这个命令就可以得到3个文件，分别记录了各个级别的依赖和版本关系。


### 集成服务器

Yotco 有一个构建服务器叫 toaster， 等待研究。

toaster看起来并不是我想象中的CI服务器，应该来说只是一个build track web interface而已，在上面可以用web的方式比较方便的获取构建信息，比如build了几个包，花了多少时间，用的什么配置。

只能简单的用toaster来承担CI服务器的效果了，可以对接上测试服务器测试当前的代码，另外还可以用apache把build完后的image暴露出来，对内提供固件。


# 参考资料

[“Bitbake”](http://www.yoctoproject.org/docs/2.3/bitbake-user-manual/bitbake-user-manual.html )  
[“Dev”](http://www.yoctoproject.org/docs/2.3/dev-manual/dev-manual.html )  
[“Ref”](http://www.yoctoproject.org/docs/2.3/ref-manual/ref-manual.html  )  
