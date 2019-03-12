---
layout: post
title: 个人自动驾驶学习整理
category: OS
tag: Autonomous Drive System
comments: 1
---

下面是个人收集的了解整个无人驾驶部分的相关学习书籍.  
是假设你已经对无人驾驶系统有了大概的认识, 然后希望对各个模块有稍微细致的了解.  
标准是"基础,简单,入门", 不求了解琐碎的工程实现和行业前沿, 只是用尽量短的时间通过建立相对基础的认知, 不至于看ppt的时候只能不明觉厉.

大概阅读时间: 200 - 300h

### 入门

如果对无人驾驶系统还没有认知, 那推荐:

##### << 第一本无人驾驶技术书 >>

https://book.douban.com/subject/27038410/

挺好的科普书, 你可以当成网上那一堆无人驾驶博客的替代品.  
抽空当厕所读物一样看下去, 建立起整个无人驾驶系统技术架构的认知.

##### Apollo公开资料

https://github.com/zhxt/apollo/wiki/Posts-From-Apollo_Developers


### SLAM

<!-- ![](https://img-blog.csdn.net/20181013104219844?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NDI3OTcy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70) -->
<!-- 视觉SLAM, GPS/惯导, 点云定位/高精地图, 传感器融合 -->

#### 书

##### << 视觉SLAM十四讲 >>

https://book.douban.com/subject/27028215/

#### 文章

就基础知识而言, 看完视觉SLAM那本书就差不多了.  
其他的东西都是融会贯通的工程问题, 那就随便找点科普的Paper看看.

##### Baidu

Apollo 2.0多传感器融合定位模块:  
[https://mp.weixin.qq.com/s/PFCCRBGWmaTQkeyOlG1qng](https://mp.weixin.qq.com/s/PFCCRBGWmaTQkeyOlG1qng)

Apollo公开课--定位:  
[https://mp.weixin.qq.com/s/KJbm4wfsTLxT5LGR7GV_SA](https://mp.weixin.qq.com/s/KJbm4wfsTLxT5LGR7GV_SA)


##### Google
[http://www.roboticsproceedings.org/rss03/p16.pdf](http://www.roboticsproceedings.org/rss03/p16.pdf)

#### Others

<!-- ![](https://img-blog.csdn.net/20180613083543557) -->

定位/建图这块涉及知识比较杂，深入的过程其实是一个追求广度和工程实现的过程, 都是engineering trick.  
所以做了解即可.  

### 规划/决策/控制:

<!-- 行为决策/动态规划/控制  
强化学习   -->

#### 书

这里实在是找不到中文入门书, 这些书就有兴趣翻翻吧.....

##### << Principles of Robot Motion: Theory, Algorithms, and Implementation >>

运动规划方向的经典著作

##### << Reinforcement Learning:An Introduction >>

[https://book.douban.com/subject/2866455/](https://book.douban.com/subject/2866455/)

强化学习在规划/决策/控制几个方向上都有应用, 如果要理解相关的文章, 有必要了解一下强化学习.   
其他机器学习有空直接看文章理解.  


#### 文章

<!-- 运动规划有比较多的知识点.   -->
控制的话, 可以直接Google一下LQR/MPC来理解, 系统的控制理论不必做了解.  

##### Baidu

Apollo公开课--规划:  
[https://mp.weixin.qq.com/s/ZBHKDtnaEI9sVCfEfeeUlQ](https://mp.weixin.qq.com/s/ZBHKDtnaEI9sVCfEfeeUlQ)  
[https://mp.weixin.qq.com/s/1yA-kgS_rL4o9I4OWeI4-A](https://mp.weixin.qq.com/s/1yA-kgS_rL4o9I4OWeI4-A)  

Apollo公开课--控制:  
[https://mp.weixin.qq.com/s/8hLVHuv635fyHRNftXJQ4w](https://mp.weixin.qq.com/s/8hLVHuv635fyHRNftXJQ4w )  
[https://mp.weixin.qq.com/s/AnWWnD3ScSW6GhrwRGpRDw](https://mp.weixin.qq.com/s/AnWWnD3ScSW6GhrwRGpRDw)  

行为决策:  
[https://blog.csdn.net/adamshan/article/details/80633099](https://blog.csdn.net/adamshan/article/details/80633099)  
[https://www.roboticschina.com/news/201705160936.html](https://www.roboticschina.com/news/201705160936.html)  

#### Others

个人而言, 这块的知识太专一了.  
除开`Robotics/Automation`的方向, 其他领域也不会有所应用.  
所以没太多兴趣的话做了解即可.

### 感知

#### 书

##### << 深度学习(deep learning) >>

了解完deep learning就够了......


#### 文章

##### Baidu

Apollo公开课--感知:  
[https://mp.weixin.qq.com/s/BCNrsuoThJt031jCnbAjAA](https://mp.weixin.qq.com/s/BCNrsuoThJt031jCnbAjAA)  
[https://mp.weixin.qq.com/s/FqW2X9LJZvNe9efllAYhTQ](https://mp.weixin.qq.com/s/FqW2X9LJZvNe9efllAYhTQ)  

Apollo公开课--预测:  
[https://mp.weixin.qq.com/s/6ro9y69h-6yHZ7MfodxUHQ](https://mp.weixin.qq.com/s/6ro9y69h-6yHZ7MfodxUHQ)  

### 其他

其他相关的方向还有:

* 硬件传感器采集与处理
* 实时系统/中间件与安全设计

等等.  

这些方向有本科EE/CS的知识就足够了, 主要重点在engineering, 不做系统学习.



