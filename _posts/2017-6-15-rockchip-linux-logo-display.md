---
layout: post
title: Show Boot Logo(Rockchip Linux)
category: tips
tag: Embedded and Linux
comments: 1
---


Unlik rockchip android, we don't have a complete method for boot logo on linux.

It means you can only show logo by generic way, and there are three parts.


## U-boot

First， you need enable video driver in u-boot and make it show console in screen.
and then, apply below patches.(Don't forget to remove console in screen)

https://github.com/wzyy2/Scripts/blob/master/tmp/U-Boot-v1-1-2-board-rockchip-common-Add-bmp-logo-support.patch  
https://github.com/wzyy2/Scripts/blob/master/tmp/U-Boot-v1-2-2-rockchip-include-Enable-logo-display-for-evb-rk3399.patch  

Since rockchip rom code have litmit the uboot size to 1Mb, so you can't display with big size bmp.
If you want to display big size bmp, you might need use gzipped bmp.

https://github.com/rockchip-linux/u-boot/blob/release/cmd/bmp.c#L40


## Kernel

http://blog.csdn.net/wang_zheng_kai/article/details/41847687

set logos_freed to false  
http://elixir.free-electrons.com/linux/latest/source/drivers/video/logo/logo.c#L32

## System

You can use 'plymouth' to show logo before desktop launched.
    
If you are using debian, there are some bug need fix.  

1.  
Change ExecStartPost in /lib/systemd/system/plymouth-start.service  to ExecStartPost=-/usr/bin/udevadm settle --timeout=30
--exit-if-exists=/sys/class/drm/card0/dev ; /usr/bin/udevadm settle
--timeout=30 --exit-if-exists=/sys/class/graphics/fb0/dev ;

2.  
Add plymouth-quit.service to After in /lib/systemd/system/lightdm.service 
