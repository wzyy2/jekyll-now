---
layout: post
title: Use OpenCV with Gstreamer
category: [rockchip linux]
tag: [CN]
---

最近在Rockchip Linux的平台尝试了一下OpenCV + Gstreamer的组合, 发现效果还蛮不错的. :)  
过程中有些心得, 在这里记录一下.... 我想这些也不只适用RockChip平台,因为涉及的都是标准的概念,
比如DMABUF, DRM, OpenCL,G2D..., 放到像Intel, Nvdia这些平台也是成立的.

下面的内容会涉及一些Linux概念, 如果你不懂的话建议先查阅下相关文章, 当然最好是接触下对应的开发: 
(有需要解释可以留言, 如果你看的到disqus的话 :-P)
* [DRM](https://en.wikipedia.org/wiki/Direct_Rendering_Manager)
* [DMABuf](http://elinux.org/images/a/a8/DMA_Buffer_Sharing-_An_Introduction.pdf6)
* [V4l2](https://linuxtv.org/downloads/legacy/video4linux/v4l2dwgNew.html)
* [OpenCL](https://en.wikipedia.org/wiki/OpenCL)

# Background

#### Gstreamer

首先,要先讨论下为什么需要在OpenCV上用上Gstreamer. 比如我直接一个摄像头 v4l2 图像传给 OpenCV 不行吗?

Gstreamer是嵌入式平台处理Media的首选组件, 像Nvdia/TI/NXP/Rockchip平台, 都是使用Gstreamer来整合Media应用.  
在Rockchip平台上, 我们已经有为Gstreamer开发了像Decode/Encode/ISP-Camera/2D加速器/DRM-Display-sink这些的Plugin.  

所以OpenCV如果链接上Gstreamer, 输入源就不仅仅是摄像头, 还可以是RTSP/本地视频；输出显示的代码可以不用写, 让Gstreamer来显示；
转换格式让Gstreamer来转, 利用硬件加速; 处理的图像送回Gstreamer编码.

#### ARM

在ARM系统上做Media的开发, 有一个原则要很重要, 就是 : **避免拷贝**.  
如果你手边正好有一块ARM板子和Linux PC, 可以尝试在上面跑一些memcpy的Test. 一般来说, 测试的性能会相差5,6倍.  
即时是DDR同频的两个系统, 性能也会差到3-4倍(不过也可能是DDR其他参数有影响?).  
内存操作速度的劣势是RISC天生的, ARM也不列外. (虽然也没有研究过对应微处理器结构,道听途说  :-P)

在开发OpenCV + Gstreamer的过程中, 一定要尽量避免拷贝的发生, 如果一定要有, 也不能是由CPU来做. (替代可以是2D加速器, GPU)

#### OpenCV

我之前只在X86上使用过OpenCV, 其实不太了解OpenCV在ARM Device需要怎么开发.
(怀疑其他ARM平台上到底能不能用OpenCV, 因为像TI/NXP这种, CPU/GPU太弱, 估计只能内部的DSP跑算法; 像全志, 基本没有Linux平台的组件支持; 唯一能搞的估计也就是Nvdia的terga了, cuda还是厉害.  ；) )

根据上面ARM的原则, 开发的时候要避免调用到OpenCv的cvtcolor和clone这些函数.  
OpenCV也支持[OepnCL加速](https://chromium-review.googlesource.com/c/455596), 使用GPU来干脏活.


# Desgin

#### Pipeline

Pipeline Prototype 1:

        video/rtsp/camera -> decoder -> opencv

这是我最先想到的, 通过gstreamer拿到decoder的buffer, 然后全部由opencv来处理. 但是前面说过, 要避免拷贝, 而opencv的显示 imshow , 是存在大量拷贝的, 所以不能这么做.

Pipeline Prototype 2:

        video/rtsp/camera -> decoder -> opencv -> display sink

为了优化显示, 需要把buffer送回给gstreamer, 这样就得到了Prototype 2. 但是是要注意, OpenCV的默认格式是bgr的, 所有的画图函数都是基于bgr； CV的大部分算法都是基于灰度图做处理, 而某些图像格式排列不适合转换灰度图.   
在Rockchip平台上Decoder出来的颜色格式是NV12的, 必须要想办法转换成BGR格式.  
所以decoder到opencv之间还需要有处理颜色格式的单元, 这个工作不可能由CPU来做, 一般可以使用[专有硬件](http://rockchip.fr/RK3288%20TRM/rk3288-chapter-28-rga2.pdf), 如果相应的平台没有这样的硬件, 
也可以使用GPU用特定的Shader来转(OpenGL的设计目的里, 2D加速就很重要的一块, 我们有时候看到QT/Wayland这些说GPU加速, 就是用GPU做这样的事).

Pipeline Prototype 3:

        video/rtsp/camera -> decoder -> 2d convert -> opencv -> display sink

#### Implement

Gstreamer Pipeline:

        video/rtsp/camera ! decoder ! v4l2videoconvert ! appsink
        appsrc ! display

Rockchip Gstreamer Pipeline:

        "filesrc location=/usr/local/test.mp4 ! qtdemux ! h264parse ! mppvideodec \
        ! v4l2video0convert output-io-mode=dmabuf capture-io-mode=dmabuf ! \
        video/x-raw,format=BGR,width=(int)1920,height=(int)1080 ! \
        appsink caps=video/x-raw,format=BGR sync=false name=sink"

        "appsrc caps=video/x-raw,format=(string)BGR,width=(int)1920,height=(int)1080,framerate=(fraction)30/1 \
        block=true name=src ! rkximagesink sync=false";

Code:

https://github.com/wzyy2/gstreamer-opencv

# Others

还是挺好玩的...
改天试试Tensorflow(可惜只有CUDA加速, 不知道能不能搞进去OpenCL)