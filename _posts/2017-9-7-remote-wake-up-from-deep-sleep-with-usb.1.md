---
layout: post
title: Remote Wake Up from Deep Sleep with USB
category: tips
tag: Embedded and Linux
comments: 1
---

PM Code have support keeping power for USB.

You just need to apply below pacthes to avoid usb phy being disabled.


        From ba76095e25daa35220f681aaccbcd54f2940b222 Mon Sep 17 00:00:00 2001
        From: Jacob Chen <jacob-chen@iotwrt.com>
        Date: Thu, 7 Sep 2017 15:06:13 +0800
        Subject: [PATCH] USB: dwc2: Don't turn off the usbphy in suspend

        Change-Id: I0dc42678f44664dfc2bbbc67bbfdf851bb87369a
        Signed-off-by: Jacob Chen <jacob-chen@iotwrt.com>
        ---
        drivers/usb/dwc2/platform.c | 4 ++++
        1 file changed, 4 insertions(+)

        diff --git a/drivers/usb/dwc2/platform.c b/drivers/usb/dwc2/platform.c
        index 60ffcb6..113bea0 100644
        --- a/drivers/usb/dwc2/platform.c
        +++ b/drivers/usb/dwc2/platform.c
        @@ -641,6 +641,8 @@ static int __maybe_unused dwc2_suspend(struct device *dev)
                if (dwc2_is_device_mode(dwc2))
                        dwc2_hsotg_suspend(dwc2);
        
        +	return 0;
        +
                if (dwc2->ll_hw_enabled)
                        ret = __dwc2_lowlevel_hw_disable(dwc2);
        
        @@ -652,6 +654,8 @@ static int __maybe_unused dwc2_resume(struct device *dev)
                struct dwc2_hsotg *dwc2 = dev_get_drvdata(dev);
                int ret = 0;
        
        +	return 0;
        +
                if (dwc2->ll_hw_enabled) {
                        ret = __dwc2_lowlevel_hw_enable(dwc2);
                        if (ret)
        -- 
        2.7.4

