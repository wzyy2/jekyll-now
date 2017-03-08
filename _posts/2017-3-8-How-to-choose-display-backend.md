---
layout: post
title: Embedded Linux 图形（长期更新）
category: [Linux, CN]
---

“How to choose display backend”， 如果有错误，请帮忙评论修正。

# Overview

对于Android开发者来说，基本不用关心图形方案这些细节，你只要调用java的class，最后的性能都是有原厂和谷歌验证过的。
但对Linux开发者来说，情况要复杂的多，因为没有一个完美方案。这主要是下面两个原因。
* 可选项太多  
Linux的图形显示选择太多，Xserver, Wayland，raw的drm，fbdev，细化到Xserver，wayland的每个版本，GUI库的编译选项，这些不同的选择会导致很大的效果差异
* 原厂没有太大的支持投入  
一个芯片可能有更好的图形解决方案，但是原厂没有去支持，你也很有可能在理论更好的方案上得到一个更差的性能。之所以有这个问题，其实和第一个原因也有很大关系。

所以当你决定要在Linux要开发应用的时候，一定要明确你的需求,对比方案间的优劣。

小框图：
![](https://github.com/wzyy2/wzyy2.github.io/raw/master/images/graphics.png)

# X11

X11的基础构架，建议先谷歌一下，太庞大，历史遗留比较多，到现在我也没弄清楚一些调用流程。  

下面主要讲讲dri2。dri2是xserver用来连接gpu的结构，下面这个链接里蛮详细的， <https://en.wikipedia.org/wiki/Direct_Rendering_Infrastructure>。  
大概理解，dri2自己管理一个window下面的buffers， xserver都不会过问，只有swap front buffer的时候，才会调一些函数来wait page flip来进行画面的同步。不过这个front buffer是false的，要注意，最后显示还要进行compoiste。理论上（理论上哦，实际我测出来还是一样慢。。），dri2全屏和不全屏的性能差距会比较大，因为全屏的情况下，dri2出来的flase front buffer，也就是这个window的drawbuffer， 是直接被作为全局的font buffer，送到ddx显示的，省去了compoiste。  
所以在x11下开发3d应用的时候，一定要全屏，保证没有多余的compoiste。     


另外一提，rk平台上的xserver，还支持了glamor，意味一些compoiste可以被gpu加速到，如果是做多窗口的应用或者desktop类型的产品，这个featrue还是非常有用的，能运行x11上的所有软件，又有gpu加速合成。   

Xserver在移动平台除了一些软件兼容性的问题可以用用，长久来看，是一定要被淘汰的。


#### links
<https://en.wikipedia.org/wiki/X.Org_Server>  
<http://www.comptechdoc.org/os/linux/howlinuxworks/linux_hlxwindows.html>  
<https://dri.freedesktop.org/wiki/DDX/>   
<https://www.freedesktop.org/wiki/Software/Glamor/>


# QT EGLFS
QT EGLFS是qt自己实现的一个gui系统，不支持多窗口，但也因此少了window compoiste。   
QT EGLFS和dri2的方式也差不多，区别就在于，qt eglfs的font buffer在自己用gpu compoiste后，是直接送给drm去显示，而X里是送Window manager去做compoiste，所以EGLFS在效率上是有优势的。


#### links
<http://doc.qt.io/qt-5/embedded-linux.html>

# Wayland

wayland是Linux上下一代的display server，从结构上来讲，也最相近android上的[“hwc”](http://dragon.leanote.com/post/Android%E5%9B%BE%E5%BD%A2%E7%B3%BB%E7%BB%9F-II-%E6%9E%B6%E6%9E%84)，全部的compoiste都是gpu来做的，不会有xserver那样cpu合成的场景。    
wayland除了gpu合成以外，另一个优势，就是overlay接口的存在，能允许移动平台上的一些2d加速模块，display模块在这个接口上被调用（这些模块才是移动平台能跑大分辨率ui的关键）。

我觉得，wayland会在移动平台上更快成为主流。


#### links
 <https://en.wikipedia.org/wiki/Wayland_>

# 应用场景

## 3d应用

3d应用的瓶颈最主要在计算单元上，拷贝，compoiste一类的开销，根据具体场景再考虑。  
建议直接raw的drm api或者qt eglfs。

## 视频播放

对视频播放来说，拷贝，compoiste的开销是决定性的。

Spec上的视频播放极限，比如rk3399,rk3288播放4k，rk3036播放1080p，基本上是不可能在通用框架，也就是走gpu实现的。
因为达到了芯片bandwidth的上限场景，如果让gpu去拷贝和转格式的话速度会很慢，必须要display的部分自己去处理显示视频数据。  
但想让display部分去处理的话，软件上必须有对应的支持-------然而desktop based的gui framework大多缺失了这样一个东西。

之前在rk的系统上，我base X11做了一个[“gstreamer sink”](https://github.com/rockchip-linux/gstreamer-rockchip/tree/master/gst/rksink/rkximage)。通过x的api获取窗口的位置，然后直接drm的api，绕过X系统，overlay画在窗口的位置。这样做确实可以发挥视频播放的极限，主要的问题就是没办法和gui系统融合，没办法叠加控件，如果使用的场景都是fullscreen，可以试试这做。  
上文提了下wayland框架支持overlay，所以最理想的，还是wayland通过overlay的机制直接call的display单元显示，像android那样。但wayland毕竟还是在一个初步开发和推广的阶段，不管上游的community，还是下游的原厂，library，都没看到有完善的overlay视频方案。  


在linux的图形系统上加入display处理的逻辑还有很多工作要做，所以如果视频性能不是那么高，又需要复杂UI，就可以用gpu的框架。
x11走gles，在rk平台的软件上，测试下来，性能比较差，也许有很多compoiste发生，也许有多余的vsync调用。  
qt eglfs是不错的选择。放视频的话，按rk3288的性能，可以达到1080p 60fps。  


# tips

## libmali
libmali是mali gpu的userspace library，我也看不到代码，完全是黑盒，只能说根据一些类似的代码和文档猜测他的实现方式。

libmali有很多编译选项，我猜的话，除了软件硬件版本，还有下面两种。

一个是fbdev和gbm，分别对应了fbdev和drm两种内核驱动的场景。fbdev对比gbm有几个差异。  
1.vblank   
用fbdev去跑on-screen的glmark，分数一般是要比gbm的高，原因就是这套流程没有去等待vblank。gbm的实现都会在最后swap front buffer的时候等待vblank，所以on-screen只能跑几十fps，fbdev的不会去等，因此fbdev的libmali on-screen的fps和off-screen的差不多。  
2.zero-copy  
所谓zero-copy就是不拷贝texture。一段在内存里的texture，要让gpu去使用，必须先用cpu把数据从这段内存拷到gpu能用的buf（dma-buf)里。如果这个texture数据本来就在一个dma-buf里的话，可以通过特定的api直接让gpu load，从而避免拷贝。dma-buf在gbm上的实现，搜索EGL_LINUX_DMA_BUF_EXT就可以。在fbdev上的实现，比较麻烦，[“fbdev zero-copy”](https://community.arm.com/graphics/b/blog/posts/improving-arm-mali-drivers-on-fbdev)。

还有就是display server的选项，比如xserver，比如wayland。
这个就是支持在display server下运行，没什么好说的。
