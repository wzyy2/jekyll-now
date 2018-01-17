---
layout: post
title: Linux Kernel里IOMMU, DMA Mapping, DMA-Buf， DRM, V4l2几者之间函数关系图
category: [linux]
tag: [CN]
---

工作里涉及的项目， 很多都是Graphic（GPU, DRM)和Media(Video decode/encode, Camera)相关的， 所以每天都要接触`IOMMU, DMA Mapping, DMA-Buf`这几个东西， 但说实在， 一直没有一个很清晰的思路， 虽然写了几个驱动，但更多的只是在看别人的驱动怎么写， 然后照猫画虎， 导致debug或者说性能调优的时候工作效率比较低...... ；)

可能这些东西比较Low level， 对于Driver developer，只要照样调用下一些API; 对于User， 只需要知道DMA-Buf`（FAQ： 为什么我自己写的程序效率差？）`的用法和Cache的效率区别`（FAQ： 为什么硬解码上比软解码效率低？）`就可以了， 所以相比其他模块， 网上的资料还是蛮缺的， google出来的就几个演讲ppt， Kernel里的API Docuemnt也很粗略， 没办法很好的理解。

这里简单的花了点时间， 照着代码画了下面这样一个图，来帮助我自己理解整个Kernel上对这块的软件结构。
流程基于4.14的mainline kernel.


# DRM， V4l2

在DRM和V4l2这几个框架里， DMA-Buf的结合也占据了一个很重要的位置， 所以这里我也简单的画了个图来理清其在其中的位置


# See Also

https://github.com/torvalds/linux/blob/master/Documentation/DMA-API-HOWTO.txt

https://github.com/torvalds/linux/blob/master/Documentation/DMA-API.txt

https://github.com/torvalds/linux/blob/master/Documentation/DMA-attributes.txt

https://developer.arm.com/products/architecture/system-architecture/iommu-support

https://gstreamer.freedesktop.org/data/events/gstreamer-conference/2012/omap-dmabuf-gstcon2012.pdf


# DMA Fence

“fence, which provides a mechanism to signal when one device as finished acces”

#### 函数
https://www.linuxtv.org/downloads/v4l-dvb-internals/device-drivers/ch02s03.html

### explicit DRM Fence

之前这个同步都是在内核做的，现在可以把消息发到用户层，这样可以不用Block等GPU Buffer完成，而是实现异步的更新。

分为in-fences和out-fences，分别是用户通知kernel和kernel通知user，Vulkan比较需要这个机制。

in-fence：
“The compositor then passes these fds to DRM in
a atomic commit request, that will not be displayed until the fences signal”函数上把fd送进去drm，即可。

https://www.x.org/wiki/Events/XDC2016/Program/XDC-Explicit-Fencing_Talk.pdf
https://lwn.net/Articles/685049/

