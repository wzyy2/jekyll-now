---
layout: post
title: Rockchip Linux Tips
category: [rockchip linux]
tag: [CN]
---


这里主要记录一些个人的测试命令和额外的补丁。


# spi

https://github.com/rockchip-linux/kernel/blob/release-4.4/drivers/spi/spidev.c

如果你想要有spidev出现在sysfs下，你需要在dts里注册这个spidev！

    bus {
      spidev ...
    }

bus是bus，dev是dev，spidev作为虚拟的设备，也是必须要声明这个dev才会被probe。


# i2c

i2c就有一个方便的sys接口了，可以看下面的文章
https://erlerobotics.gitbooks.io/erle-robotics-erle-brain-a-linux-brain-for-drones/en/tutorials/i2c.html


# wifi

见 RK WIFI 文档 

# usb

见 RK USB Compliance Test Note V1.2.pdf

# eth

可以用下面办法直接写寄存器

        cd /sys/bus/mdio_bus/devices/stmmac-0\:01
        echo 20 > phy_reg
        echo 0x400 > phy_regValue

# 单通道ddr 3288

休眠补丁
https://github.com/wzyy2/Scripts/blob/master/tmp/3288_fix_ddr_single_channel.patch

uboot制作命令
dd if=tools/rk_tools/bin/rk32/32_LPDDR2_400MHz_LPDDR3_400MHZ_DDR3_400MHZ_20160803.bin
of=spl.bin bs=4 skip=1 && tools/mkimage -n rk3288 -T rksd -d spl.bin
rk3288_uboot.img && cat u-boot-dtb.bin >> rk3288_uboot.img
