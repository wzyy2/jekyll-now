---
layout: post
title: Debug Kernel(kgdb, gcover, blktrace, ftrace)(Rockchip Linux)
category: [rockchip linux]
tag: [CN]
---

# kgdb

### Introduction
以前上学的时候, 我参加过一开源活动, 给一个RTOS移植GDB Stub, 那个时候看过内核的kgdb实现.  
在我的印象里, 内核的arm kgdb比我实现的还简陋..只有软件断点, 没有数据断点, 没有单步支持...
现在看`git log`, 似乎还是这样子......

不过相比log debug, kgdb可以用作vanilla kernel上的ttyfiq替代项, 专门来debug hang住的系统, 比如死锁问题.
因此kgdb在实际环境上还是蛮有用的, 更多的用处可以看这里linaro的文章:<br>
https://www.linaro.org/blog/core-dump/debugging-arm-kernels-using-nmifiq/

### Usage

Kconfig:

    CONFIG_KGDB
    CONFIG_SERIAL_KGDB_NMI

1.编译烧写完内核后, 在Kernel Cmdline里加入

    kgdbwait kgdbcon kgdboc=ttyS2,115200

kgdbwait:<br>
The Kernel command line option kgdbwait makes kgdb wait for a debugger connection during booting of a kernel.

kgdbcon:<br>
The kgdbcon feature allows you to see printk() messages inside gdb while gdb is connected to the kernel. Kdb does not make use of the kgdbcon feature.

kgdboc:<br>
The kgdboc driver was originally an abbreviation meant to stand for "kgdb over console". Today it is the primary mechanism to configure how to communicate from gdb to kgdb as well as the devices you want to use to interact with the kdb shell.

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

如果打开了`CONFIG_SERIAL_KGDB_NMI`, 可以直接通过nmi终端唤醒, 在串口上做如下操作就可进到gdb:

    To trigger kdb you must manually type the gdbserver protocols wake up command $3#33

如果是调式kernel module的话, 需要另外加载符号

    add-symbol-file ./test/test.ko 0xbf000000

地址可在client用下面命令取得:

    MODULE_NAME=your_module_name
    MODULE_FILE=$(modinfo $MODULE_NAME| awk '/filename/{print $2}')
    DIR="/sys/module/${MODULE_NAME}/sections/"
    echo add-symbol-file $MODULE_FILE $(cat "$DIR/.text") -s .bss $(cat "$DIR/.bss") -s .data $(cat "$DIR/.data")

# ftrace

Ftrace is an internal tracer designed to help out developers and
designers of systems to find what is going on inside the kernel.

Ftrace是一个非常好用的profiling/trace工具, 如果你需要在项目中分析函数的调用时间(perf那是用来测i/o效率的), 或者函数的调用过程.

Ftrace的理解应该结合Kernel tracepoint来看, 所以最好先看下下面这篇文章:


### Usage

Kconfig:

    CONFIG_HAVE_FTRACE_NMI_ENTER=y 
    CONFIG_HAVE_DYNAMIC_FTRACE=y 
    CONFIG_HAVE_FTRACE_MCOUNT_RECORD=y 
    CONFIG_FTRACE_NMI_ENTER=y 
    CONFIG_FTRACE=y
    CONFIG_FTRACE_SYSCALLS=y 
    CONFIG_DYNAMIC_FTRACE=y 
    CONFIG_FTRACE_MCOUNT_RECORD=y 


进入trace目录:

    cd /sys/kernel/debug/tracing/

查看可用的tracer:

    cat available_tracers 


开始跟踪:

    echo > set_event
    echo 0 > tracing_on
    echo > trace
    echo function_graph > current_tracer 
    echo 1 > tracing_on
    cat trace

下面的命令可以过滤函数:

    echo vmalloc_* > set_ftrace_filter

或者特定事件:

    cat available_events 

ftrace的参数还有很多, 这里也不好列出来.<br>
有需要, 你可以转到下面的链接来看看解释:<br>
https://github.com/wzyy2/linux/blob/master/Documentation/trace/ftrace.txt  
https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace/


# blktrace

# gcover

