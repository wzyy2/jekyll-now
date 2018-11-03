---
layout: post
title: Bluetooth(Rockchip Linux)
category: linux
tag: CN
comments: 1
---

Rockchip 蓝牙模块统一使用的HCI驱动，因此调试主要是电源相关的配置。

net/rkkill/rfkill-rk.c下面有一些rk写的rfkill代码，主要用来控制电源睡眠唤醒这些。
在移植到自己板子上的时候，主要是配下面这样的dts。

    wireless-bluetooth {
      compatible = "bluetooth-platdata";
      uart_rts_gpios = <&gpio4 19 GPIO_ACTIVE_LOW>;
      pinctrl-names = "default", "rts_gpio";
      pinctrl-0 = <&uart0_rts>;
      pinctrl-1 = <&uart0_gpios>;
      BT,reset_gpio    = <&gpio4 29 GPIO_ACTIVE_HIGH>;
      BT,wake_gpio     = <&gpio4 26 GPIO_ACTIVE_HIGH>;
      BT,wake_host_irq = <&gpio4 31 GPIO_ACTIVE_HIGH>;
      status = "okay";
    };

我也不知道linux下的程序，比如blueman，能不能和这个rkkill配合，一般都是先自己手动确认下

    root@evb-rk3288:~# rfkill list                                                     
      0: bt_default: bluetooth                                                           
              Soft blocked: yes                                                          
              Hard blocked: no                                                           
      1: phy0: wlan                                                                      
              Soft blocked: no                                                           
              Hard blocked: no                                                           
      2: brcmfmac-wifi: wlan                                                             
              Soft blocked: no                                                           
              Hard blocked: no 
              
      root@evb-rk3288:~# rfkill unblock 0                                                
      [  599.915815] [BT_RFKILL]: ENABLE UART_RTS                                        
      [  600.025465] [BT_RFKILL]: DISABLE UART_RTS                                       
      [  600.029514] [BT_RFKILL]: bt turn on power

确认电源打开了,然后进行hci attach


rtk：

    rtk_hciattach -n -s 115200 ttyS0 rtk_h5

brcm：

    brcm_patchram_plus -d --enable_hci --no2bytes --use_baudrate_for_download --tosleep 20000 --baudrate 1500000 --patchram /etc/firmware/bcm4339a0.hcd /dev/ttyS0 &

brcm_patchram_plus版本：  
https://github.com/rockchip-linux/meta-rockchip/tree/morty/recipes-bsp/brcm-patchram-plus

然后再 hciconfig, 可以看到bt起来

    root@mq:~# hciconfig -a
    hci0: Type: Primary Bus: UART
    
# DEBUG

参考RK BT文档里的常见问题，一般是没上电阿，uart通信有问题，晶体问题

# Function

## BT 4.0

### A2DP

choppy的话看下cpufreq, 是不是太低, sbc解码比较吃cpu 

audio conf, 可以设置force sbc和mpeg
https://android.googlesource.com/platform/external/bluetooth/bluez/+/android-4.1.1_r4/audio/audio.conf

## BLE 

### Central

### Peripheral


# Known issue

要注意不要在CONFIG里打开HCI_SDIO, 发现有时候WIFI SDIO会被认成HCI SDIO，导致一系列出错。
