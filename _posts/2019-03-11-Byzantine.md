---
layout: post
title: 容错设计与无人车
category: OS
tag: Autonomous Drive System
comments: 1
---

<small>(纯属YY, 请勿参考......)  
  PS: [最近737-MAX事故了解下](https://baijiahao.baidu.com/s?id=1627695537914121317&wfr=spider&for=pc), 很贴合主题..... </small>

![](http://blog.iotwrt.com/images/safety.svg)

前几天无聊看文章, 搜到了这面这个ppt, [Fault-Tolerance in Avionics Systems](https://cs.unc.edu/~anderson/teach/comp790/papers/fault_tolerance_avionics.pdf).    
这个ppt比较易懂的解释了航空系统中的容错机制, 看了感觉非常有意思.  

之前也看过相关新闻, 像SpaceX, 是在火箭和飞船的控制上粗放的使用了n+个Linux x86电脑, 主要通过额外的容错机制来做到robust.   
所以就遐想一下, 如果将自动驾驶车辆按照航天器标准设计容错, 会是什么样子.  


## 背景

### [Byzantine Faults(拜占庭将军问题)](https://baike.baidu.com/item/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%86%E5%86%9B%E9%97%AE%E9%A2%98/265656?fr=aladdin).  

为了解决拜占庭将军问题, 可以将系统分成多个Fault Containment Region(故障包容区), 然后通过Triple Modular Redundancy/Voting System/Dual-Dual等方法同步输出结果.


### Common Mode Failures(共因失效)

在上面的措施中, 拜占庭将军问题可以相对的得到解决, 剩下不能处理的问题还有共因失效.  
共因失效会同时影响到多个Fault Containment Region.

针对共因失效的问题, 可以这么解决:
* Fault Avoidance during design
* Fault Removal through testing, evaluation, fault insertion
* Fault Tolerance through exception handlers and program checkpointing / restarting

在候选处理器之间安排可以处理时序要求的任务

### Fault Tolerant Scheduler(容错调度)
* 假设我们的系统有一些重复的处理模块.
* 需求:
    * 任务调度在模块间的调度需要满足timing requirements.
    * 要提供相关容错的机制, 例如管理重复执行和任务成功检查
* Surprise, the solution involves executing tasks at different modules.
* 所有的任务重复和验证必须要满足deadline!

#### 硬件实现



#### 软件实现


### 相关历史
<!-- (Uh-huh? 平行驾驶板?) -->

* 最早是NASA和一些航空电子设备制造商开始关心故障避免.
    * Apollo had a computer with rigorously verified code. No permanent failure was ever recorded.
* 使用Triple Modular Redundant和Dual-Dual的系统在70年代早期开始出现
    * Exact consensus was used for fault detection / isolation
* 第一个相关商业案例出现在大型喷气式飞机的自动着陆功能中。
    * 747 used TMR, Lockheed L-1011 Dual-Dual
* Dual-Dual和TMR被设计来防止拜占庭故障，虽然这个时候其理论尚未被提出。
* 后来对拜占庭容错的研究为这些设计提供了理论依据。
* 这之后, 最主要导致出错的原因变成了Common-Mode Failures.
* 航天飞机设计上必须能够在一次出错后还能完成任务，在两次出错之后还能安全降落.
    * 因此使用了4个独立的计算系统，每个都是一个FCR。
    * 同时还装备了一个简单的备份系统，如果需要可以完成降落的任务。
        * 这算是早期的Common Mode Failures保护措施, 虽然设计上没有明确出来

## 无人车

基于上述背景知识, 来设想一下如何为无人车设计容错架构.

### 不同点

对航空器设计来说, 其所关注的是针对由于软硬件系统"故障失效"而导致的安全风险.  
但对运行着自动驾驶软件的无人车来说, 安全问题并不一定仅仅由于系统"故障失效"引起.  
更多的场景下, 可能是因为自身系统设计上的limitation导致的安全风险.  

因此这里需要再加入针对performance limitation的考虑.  
当然重点不放在如何找到以及改善performance limitation, 这个是扩大到整体功能安全范畴了.  
**我们主要关注的是, 针对环境超出performance limitation后的容错处理.**  


### 相关案例

#### [Audi A8 zFAS](https://www.audi-technology-portal.de/en/electrics-electronics/driver-assistant-systems/audi-a8-central-driver-assistance-controller-zfas)

    zFAS有4个核心元件，包括Mobileye的EyeQ3，负责交通信号识别，行人检测，碰撞报警，光线探测和车道线识别。英伟达的K1负责驾驶员状态检测，360度全景。英特尔（Altera）的Cyclone V负责目标识别融合，地图融合，自动泊车，预刹车，激光雷达传感器数据处理。英飞凌的Aurix TC297T负责监测系统运行状态，使整个系统达到ASIL-D的标准，同时还负责矩阵大灯

具体这里的Aurix TC297T上面的处理逻辑, 我们不得而知了, 只知道是safety management unit.  
不过基本也逃不开上面所述描述的理论.

![](https://edge.slashgear.com/wp-content/uploads/2017/09/03_vernetzung-zFAS-Aktuatoren-1018x720.jpg)

![](https://edge.slashgear.com/wp-content/uploads/2017/09/Screen-Shot-2017-09-11-at-4.25.50-PM-1280x716.jpg)

![](https://pic2.zhimg.com/v2-8b1f85bba1821eab133f370d6f9d72ed_r.jpg)


**这里值得一提的是, [zFAS上各处理器之间的bus和middleware使用的TTTech的技术](https://www.tttech.com/press/tttech-and-infineon-to-speed-up-the-development-in-autonomous-driving/), 这也是一个很关键的安全设计.**


#### [elektrobit]
