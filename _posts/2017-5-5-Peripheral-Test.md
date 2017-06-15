---
layout: post
title: 常用外设测试
category: [CN]
---


这里主要记录一些个人的测试命令。


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
