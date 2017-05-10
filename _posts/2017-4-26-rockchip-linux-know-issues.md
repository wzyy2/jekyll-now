---
layout: post
title: Rockchip Linux Known issues
category: [EN]
---

There are some known issues and i don't have a good way to fix it, so list them in here.

# U-boot

## rk3288 gmac pinctrl

If you have a problem when use gmac in u-boot, try below patch to set pin power strength for gmac.

	diff --git a/drivers/net/gmac_rockchip.c b/drivers/net/gmac_rockchip.c
	index 5e2ca76..0ccc28f 100644
	--- a/drivers/net/gmac_rockchip.c
	+++ b/drivers/net/gmac_rockchip.c
	@ -160,6 +160,11 @ static int gmac_rockchip_probe(struct udevice *dev)
	struct clk clk;
	int ret;

	+ /* Just for test */
	+ writel(0xffffafaf, (void *)(0xff7701ec));
	+ writel(0xffff5b55, (void *)(0xff7701f0));
	+ writel(0xffff555e, (void *)(0xff7701f4));
	+
	ret = clk_get_by_index(dev, 0, &clk);
	if (ret)
	return ret;



# Kernel

## WIFI suspend clk error

Try below patch:  
https://patchwork.kernel.org/patch/9348571/



# Syetem

## video decode in debian hang 

I have no ideas, people say it's libc bugs.  
So the solution is to use yocto.
