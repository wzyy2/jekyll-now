---
layout: post
title: Swap video layer
category: [EN]
---

By default, Video layer is on top of GUI layer, which means you can't put GUI items on top of video.

To allow video work with GUI, there are two ways.

1. Use Wayland dma-buf api  
If there is something cover the video layer, wayland compositor will use GPU to composite.
The advantage is that it's a standard and the api is clean.
The disadvantage is that wayland have many compatibility problem(e.g QT) and the using of GPU will reduce performance.

2. Change video layer to the bottom  
In this way, we could make the place we want to show video transparent, so the bottom layer would show up.
The advantage is that it has the best performance.
The disadvantage is that it's kind of hacky.


In below, i will introduce how to enable the second way on different display server.

First, you need apply the below kernel patches.

The Kernel Patch for swap layer:  
[kernel patch 1](https://github.com/LongChair/LibreELEC.tv/blob/rockchip_wip/projects/Rockchip/patches/linux/rockchip-4.4/linux-0007-drm-rockchip-make-video-overlay-bottom-layer.patch)


The Kernel Patch for dsiable double vsync with drm api:  
[kernel patch 2](https://github.com/LongChair/LibreELEC.tv/blob/rockchip_wip/projects/Rockchip/patches/linux/rockchip-4.4/linux-0017-drm-skip-wait-on-vblank-for-set-plane.patch)



## QT eglfs

"we could make the place we want to show video transparent"  
To make qt background transparent, we need this patches to change the default color format to ARGB.  
[qt patch 1](https://github.com/LongChair/LibreELEC.tv/blob/4622c5fdb30a9639e6f76973af5398308bf5ec74/packages/plex/multimedia/qt5/patches/qt5-017-eglfs_kms-make-framebuffer-transparent.patch)

Then set the alpha value of the window you want to show video to zero, VOP will skip them when do alpha blending.

#### display

Before starting application, please set below env value to force qt use kmssink to show video on bottom.

  export QT_GSTREAMER_WINDOW_VIDEOSINK=kmssink

#### qt class
Please use qmlvideo

## X11/Wayland

On X11 and Wayland backend, we can't make GUI layer transparent by alpha becuase of the exists of desktop background.
If you set window backgournd color to transparent, it would show background rather than video lyaer, so we can only use color key.

To set color key, it need some changes in kernel driver.  

[kernel patch 5](https://github.com/wzyy2/Scripts/blob/master/tmp/0001-drm-rockchip-vop-add-color-key-reg-for-rk3288.patch)

[kernel patch 6](https://github.com/wzyy2/Scripts/blob/master/tmp/0002-drm-rockchip-make-black-a-color-key.patch)

it will make black color a transparency color key.

#### display

Before starting application, please set below env value to force qt use rkximagesink to show video on bottom.

  export QT_GSTREAMER_WINDOW_VIDEOSINK=rkximagesink
  
#### qt class
qmlvideo, qtvideowidget

## Others

Somtimes we need a 1080P UI with the video layer in 4K resolution for smooth, then we might need this patch.  

The Kernel Patch for 1080P UI + 4K video layer :  
[kernel patch 3](https://github.com/LongChair/LibreELEC.tv/blob/rockchip_wip/projects/Rockchip/patches/linux/rockchip-4.4/linux-0016-drm-allow-framebuffer-and-videomodes-not-to-have-sam.patch)


The Qt patch for 1080 UI:  
[kernel patch 4](https://github.com/LongChair/LibreELEC.tv/blob/4622c5fdb30a9639e6f76973af5398308bf5ec74/packages/plex/multimedia/qt5/patches/qt5-016-eglfs_kms-allow-framebuffer-upscale.patch)
