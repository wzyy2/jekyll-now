---
layout: post
title: Debug Kernel(kgb, crash, gcover, blktrace, ftrace)(Rockchip Linux)
category: [rockchip linux]
tag: [CN]
---

# kgdb

### Introduction
以前上学的时候, 我参加过一开源活动, 给一个RTOS移植GDB Stub, 那个时候看过内核的kgdb实现.  
在我的印象里, 内核的arm kgdb其实是蛮简陋的..只有软件断点, 没有数据断点, 没有单步支持...
现在看`git log`, 似乎还是这样子......
所以我是不太倾向使用kgdb在内核上debug arm device, 本来要是能支持外部中断也不错, 可以用来debug死锁问题.
有看到linaro提交过一组kgdb fiq的补丁,用作vanilla kernel上的ttyFIQ替代.
但是看起来好像也没有接受.</br>
**在arm上就忘了这个东西把**


### Usage

Kconfig:</br>
`CONFIG_KGDB`</br>

1.编译烧写完内核后, 在Kernel Cmdline里加入

    kgdbwait kgdbcon kgdboc=ttyS2,115200

kgdbwait:</br>
The Kernel command line option kgdbwait makes kgdb wait for a debugger connection during booting of a kernel.

kgdbcon:</br>
The kgdbcon feature allows you to see printk() messages inside gdb while gdb is connected to the kernel. Kdb does not make use of the kgdbcon feature.

kgdboc:</br>
The kgdboc driver was originally an abbreviation meant to stand for "kgdb over console". Today it is the primary mechanism to configure how to communicate from gdb to kgdb as well as the devices you want to use to interact with the kdb shell.</br>
kgdb实际上就是一种gdb stub, 通过一个通信方式和host联系, host的gdb发送设置断点, 修改/读取内存数据的命令, kgdb进行响应.</br>

也可以使用下面命令通过以太网debug.

    kgdbwait kgdbcon kgdboe=@192.168.248.76/,@192.168.242.24/00:11:22:33:44:55

内核起来到所需的driver初始化完后, 就会停住等待Host链接.

2.In Host side,

    $arm-none-eabi-gdb vmlinux

        gdb > set serial baud 115200
        gdb > target remote /dev/ttyUSB0

or

    arm-none-eabi-gdb vmlinux -ex "set print pretty on" -ex "set pagination off" -ex "set serial baud 115200" -ex "target remote /dev/ttyUSB0"

执行continue后, 可以在client上输入下面命令回到中断上:

    echo g > /proc/sysrq-trigger

如果是调式kernel module的话, 需要另外加载符号

    add-symbol-file ./test/test.ko 0xbf000000

地址可在client用下面命令取得:

    MODULE_NAME=your_module_name
    MODULE_FILE=$(modinfo $MODULE_NAME| awk '/filename/{print $2}')
    DIR="/sys/module/${MODULE_NAME}/sections/"
    echo add-symbol-file $MODULE_FILE $(cat "$DIR/.text") -s .bss $(cat "$DIR/.bss") -s .data $(cat "$DIR/.data")

# crash

# gcover

# blktrace

# ftrace