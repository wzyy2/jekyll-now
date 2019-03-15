---
layout: post
title: 容错设计与无人车
category: OS
tag: Autonomous Drive System
comments: 1
---

<small>(YY, 请勿参考)  </small>
![](http://blog.iotwrt.com/images/safety.svg)



## 1. 背景

最近看了一个ppt, [Fault-Tolerance in Avionics Systems](https://cs.unc.edu/~anderson/teach/comp790/papers/fault_tolerance_avionics.pdf).    
ppt比较易懂的解释了航空系统中的容错机制, 感觉有些意思.   
之前也看过相关片段, 像SpaceX是在火箭和飞船的控制上粗放的使用了n+个Linux x86电脑, 主要通过额外的容错机制来做到robust.   
这里也跟着遐想, 如何将自动驾驶车辆按照航天器标准设计.

下面是对ppt的截取.

### 1.1. 介绍

* Fault Tolerance: 指错误检测, 损坏评估, 故障隔离以及从错误中恢复的能力.   
* Fault:可能导致系统或功能失效的异常条件, 可译为“故障”.  
* Error:计算、观察或测量值或条件, 与真实、规定或理论上正确的值或条件之间的差异, 可译为“错误”.
* Failure:当一个系统不能执行所要求的功能时, 即为Failure, 可译为“失效”.   

现代Fault Tolerance主要关注两种Fault, Byzantine Faults 和 Common Mode Faults.  


### 1.2. [Byzantine Faults(拜占庭将军问题)](https://baike.baidu.com/item/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%86%E5%86%9B%E9%97%AE%E9%A2%98/265656?fr=aladdin).

#### 1.2.1. Fault Containment Region

* 系统可以分成多个Fault Containment Region.
* FCR外的错误不能导致其内部错误, 其内部错误也不能传播到外部.
* 实际中, FCR通常要求独立的处理器，存储器，IO，数据，以及电源.

#### 1.2.2. Voting

* Voting is one method of using multiple FCRs, where a voter takes output from the FCRs and decides the correct output.
* 3 varieties:
    * Exact: output values must match bit for bit.
    * Approximate: values must be w/in a certain range of avg.
    * Mechanical: system physically creates correct output. E.g. 
        * Each component provides a fraction of force needed to move output.
        * A bad component will have its output overwhelmed by good

##### 1.2.2.1. Voting Application

* Your messengers must be Byzantine Resilient also.
* Inputs, such as samples of sensor data, can also use a voter.
* Most aircraft provide redundant sensors.
* Transmission media can use voters and multiple connections.
* Voters are components! They must be extremely reliable or the system is pointless.
* Voting is often used to vote on the state of a component.



#### 1.2.3. Triple Modular Redundancy
* Originated with Apollo.
* 3 systems operate on input. Majority value or average wins.
* The voter can optionally shutdown and reconfigure bad components.
* Created and used before Byzantine Generals’ paper, though it solves the same problem.
* Still widely in use today.


![](http://blog.iotwrt.com/images/sc.png)

#### 1.2.4. Dual Modular Redundancy/Replication

* This is not voting. Uses simpler hardware or software to reconcile outputs.
    * Command / Monitor: command is primary CPU, monitor merely checks output of command.
    * Primary / Backup: same as C/M, but backup computer can take over as primary.
    * Hot Swap: backup always has same state as primary for instantaneous replacement.


#### 1.2.5. Dual - Dual
* Two pairs of each processing component, both executing on all input.
* In each pair, both components send their output to simple hardware comparator. If it finds that output differs, switches over to other pair until first pair reconfigured / replaced.
* Used in lots of old interplanetary satellites.

#### 1.2.6. Effects on Software

* Programs need to be written without knowledge of hardware resilience.
* Even operating systems are unaware of Byzantine Resilience, instead software just assumes that Byzantine Errors don’t happen.
* The hardware and software used to detect and isolate faults, mask errors, and reconfigure components are kept separate form any operational software.
* Simplifies code.

#### 1.2.7. Error Detection
* If bitwise comparison with other components fails, declare failed component faulty. Most voting systems work this way.
* Thresholds / Reasonableness: check that component output is within a certain threshold. 
    * E.g. Asked to calculate time, returns 25 o’clock or STRAWBERRY
* Built-in Test: components check themselves for failures.
    * Power-on BIT: checks itself at startup. Think POST.
    * Continuous BIT: components check themselves periodically during slack time.

#### 1.2.8. Component Synchronization

* For components to work together smoothly, their clocks must be synchronized.
* Clocks themselves must be Byzantine Resilient (to a certain degree).
* Surprise, redundancy needed.
* Voting used to determine the specific time.
* After time determined, push out to all Fault Containment Regions.

#### 1.2.9. Reconfiguration

* Remove faulty component at the cost of redundancy.
* Replace faulty component with hot swap or spare, costs more in power or money.
* Fix state. E.g.
    * Use voting to remake internal state of offending component
    * Error Correcting Codes: use extra data in bits to find correct values
    * Rollback: load state of system before crash. Usually impractical
* Transient Errors: vast majority of errors encountered do not repeat.
* Two ways of dealing with this:
    * Keep rechecking value. If it persists, fault is permanent. Reconfigure something.
    * Immediately reconfigure on failure, run BIT, and reintegrate if passes

#### 1.2.10. Note on Buses
* The buses (such as communication or power, if you don’t have independent generators) must also be Byzantine Resilientish.
* Similar techniques would apply with small changes. E.g.
    * Communication buses might have voting components located at each interface to an FCR, with multiple buses feeding in
    * Backup buses in case one is damaged
    * Etc..

### 1.3. Common Mode Failures(共因失效)

在上面的措施中, 拜占庭将军问题可以相对的得到解决, 剩下不能处理的问题还有共因失效.  
共因失效会同时影响到多个Fault Containment Region.  

Types:
* Transient (external): temporary result of environment (lightning)
* Permanent (external): constant interference from environment (flying in a tsunami)
* Intermittent (design): introduced during design of the system (can’t turn left in Illinois sometimes)
* Permanent (design): introduced during design of the system, doesn’t leave (can’t ever turn left in Illinois)

针对共因失效的问题, 可以这么解决:
* Fault Avoidance during design
* Fault Removal through testing, evaluation, fault insertion
* Fault Tolerance through exception handlers and program checkpointing / restarting

#### 1.3.1. Fault Avoidance

* Don’t reinvent wheel.
    * Problems arise because manufactures like to design own hardware
    * Existing hardware has already been tested / verified in field
    * COTS hardware offers a cheap alternative with lots of unofficial testing **
    * Conform to existing standards for similar reasons
* Formal Methods
    * The Space Shuttle takes this approach
* Design Diversity
    * Run systems using different hardware / software designed by different teams

#### 1.3.2. Fault Removal

* Basically means test the system rigorously to find problems.
* Fault Insertion: fake faults to make sure you system can handle them.

#### 1.3.3. Fault Tolerance
* Watchdog Timers: task which occasionally runs and verifies system state.
    * Could find an invalid PC, for example
* Hardware Exceptions: when invalid hardware operation found, throw and catch hardware exceptions.
* Runtime Checks: software exceptions, etc.
* Program Checkpointing: periodically save state, reload state when an error is found. Impractical often.
* Restarting: always an option.

### 1.4. Fault Tolerant Scheduler(容错调度)
* 假设我们的系统有一些重复的处理模块.
* 需求:
    * 任务调度在模块间的调度需要满足timing requirements.
    * 要提供相关容错的机制, 例如管理重复执行和任务成功检查
* Surprise, the solution involves executing tasks at different modules.
* 所有的任务重复和验证必须要满足deadline!

#### 1.4.1. 硬件实现

* 每个处理模块都有一个用于同步和投票的冗余管理系统(RMS).
* RMS根据从重复任务中返回的数值以及整个处理模块的状态进行投票选择.
* 应用只会响应于所有RMS都投票的值.
* 如果RMS确定一个模块出现故障, 则该模块会受被降级, 并会得到新数据以进行恢复.

![](http://blog.iotwrt.com/images/hi.png)

#### 1.4.2. 软件实现

* 一个任务release后, 会模块间复制多份. 每当一个模块执行完任务, 其结果将移至投票队列.
* 在投票时间内(即任务release到deadline之间), voter会执行然后计算voting queue的结果. 投票消耗时间由所使用的算法确定
* 任何添加到voting queue但未由voter处理的任务必须在下次被处理.

![](http://blog.iotwrt.com/images/si.png)

### 1.5. 相关历史
<!-- (Uh-huh? 平行驾驶板?) -->

* 最早是NASA和一些航空电子设备制造商开始关心故障避免.
    * Apollo had a computer with rigorously verified code. No permanent failure was ever recorded.
* 使用Triple Modular Redundant和Dual-Dual的系统在70年代早期开始出现
    * Exact consensus was used for fault detection / isolation
* 第一个相关商业案例出现在大型喷气式飞机的自动着陆功能中.
    * 747 used TMR, Lockheed L-1011 Dual-Dual
* Dual-Dual和TMR被设计来防止拜占庭故障, 虽然这个时候其理论尚未被提出.
* 后来对拜占庭容错的研究为这些设计提供了理论依据.
* 这之后, 最主要导致出错的原因变成了Common-Mode Failures.
* 航天飞机设计上必须能够在一次出错后还能完成任务, 在两次出错之后还能安全降落.
    * 因此使用了4个独立的计算系统, 每个都是一个FCR.
    * 同时还装备了一个简单的备份系统, 如果需要可以完成降落的任务.
        * 这算是早期的Common Mode Failures保护措施, 虽然设计上没有明确出来

## 2. 相关案例

### 2.1. Waymo

[Waymo Safety Report 2018](https://storage.googleapis.com/sdc-prod/v1/safety-report/Safety%20Report%202018.pdf)  

![](http://blog.iotwrt.com/images/waymo.png)

上面是Waymo Safety report里的Safety-Critical Systems描述.  

备用运算, 备用制动, 备用转向, 备用电源, 备用碰撞, 冗余惯性测量.  

### 2.2. Elektrobit

[Challenges and current solutions for
safe and secure connected vehicles](https://www.safeware-engineering.org/site/assets/files/1016/02_boettger.pdf)

![](http://blog.iotwrt.com/images/Elektrobit2.png)

这个文档里不是讲自动驾驶的, 主要是汽车ECU相关的safety architecture.  
大致也可以看到相关思路也是符合上述理论的.  


[Designing a software framework for
automated driving](http://on-demand.gputechconf.com/gtc-eu/2017/presentation/23257-sebastian-ohl-designing-a-software-framework-for-automated-driving.pdf)

### 2.3. [Audi A8 zFAS](https://www.audi-technology-portal.de/en/electrics-electronics/driver-assistant-systems/audi-a8-central-driver-assistance-controller-zfas)

    zFAS有4个核心元件, 包括Mobileye的EyeQ3, 负责交通信号识别, 行人检测, 碰撞报警, 光线探测和车道线识别.英伟达的K1负责驾驶员状态检测, 360度全景.英特尔（Altera）的Cyclone V负责目标识别融合, 地图融合, 自动泊车, 预刹车, 激光雷达传感器数据处理.英飞凌的Aurix TC297T负责监测系统运行状态, 使整个系统达到ASIL-D的标准, 同时还负责矩阵大灯

<!-- ![](https://edge.slashgear.com/wp-content/uploads/2017/09/03_vernetzung-zFAS-Aktuatoren-1018x720.jpg) -->

![](https://edge.slashgear.com/wp-content/uploads/2017/09/Screen-Shot-2017-09-11-at-4.25.50-PM-1280x716.jpg)

![](https://pic2.zhimg.com/v2-8b1f85bba1821eab133f370d6f9d72ed_r.jpg)


**这里值得一提的是, [zFAS上各处理器之间的bus和middleware使用的TTTech的技术](https://www.tttech.com/press/tttech-and-infineon-to-speed-up-the-development-in-autonomous-driving/), 这也是一个很关键的安全设计.**


## 3. 无人车

基于上述背景知识, 来设想一下如何为无人车设计容错架构.

### 3.1. 思路

按照功能安全的分析方法, 思路应该是:
* 项目定义
* 危险分析和风险评估
* 功能安全概念设计
* 系统开发
* 安全确认

这里就随意点, 按照如下的思路来:  
* 列出可能的 Byzantine Faults 和 Common Mode Faults
* 找合适解决方案
* 输出系统结构


#### 3.1.1. 不同点

对航空器设计来说, 其所关注的是针对由于软硬件系统"故障失效"而导致的安全风险.但对运行着自动驾驶软件的无人车来说, 安全问题并不一定仅仅由于系统"故障失效"引起.更多的场景下, 可能是因为自身系统设计上的limitation导致的安全风险.  

因此这里需要再加入针对performance limitation的考虑.当然重点不放在如何找到以及改善performance limitation, 这个是扩大到整体系统安全范畴了. 主要关注的是, 针对环境超出performance limitation后的容错处理. 

### 3.2. Faults

底盘域不考虑, 假设已符合标准.  

<!-- Byzantine Faults
Common Mode Faults -->

硬件:
* 电源故障
* 总线(CAN/ETH)故障
* 传感器故障
* 计算平台故障
    * 元器件故障
    * 粒子翻转
* 移动网络故障
* ......


按模块分的软件故障(以apollo为例):
* Sensor software Faults
* Planning Faults
* Perception Faults
* Localization Faults
* Map Faults
* Control Faults
* System software Faults

模块是有上下游的, 因此对于下游的模块, 更需要注意.

![](http://blog.iotwrt.com/images/apollo.png)

软件故障会引起的现象:
* 进入循环
* 挂起
* 崩溃
* 性能下降
* 结果不正确

### 3.3. Performance Limitation

下面是我现在可以想像到的Performance Limitation:

<!-- * 传感器 -> 感知/预测 -> 决策 -> 规划 -> 控制 -> 硬件
* 传感器 -> 感知/预测 -> 防撞决策 -> 控制 -> 硬件
* 传感器 -> 防撞决策 -> 控制 -> 硬件
* 监控 -> 控制 -> 硬件 -->


* 整体能力的限制, 程序无法处理当前环境(狭窄路况, 恶劣天气)
* 感知的局限性, 范围内未识别出障碍物
* 行为决策的不确定性, 做出错误的驾驶决策
* 响应延迟, 障碍物出现在安全距离内(突然出现的行人)


### 3.4. 解决方法
<!-- 
对于硬件问题, 基本可以增加成本, 通过冗余的机制解决.   
对于软件问题, 冗余也多半成立.    
不过有相当部分软件问题是Common Mode Faults, 因此还需要增加Diversity或者使用其他runtime措施.

总结来说, 就是几个方法: -->
解决手段:  
* 多样
* 冗余
* 监控

严谨的流程中, 使用何种解决手段, 使用到什么程度, 是根据失效概率和安全等级来决定的. 不过作为单纯的YY, 不可能做到这种程度, 因此这里只简单使用下列规则.

选择双重冗余与三重冗余与多样化冗余的规则.
* 如果一个模块, 在其出错的时候, 基本都能检测出来(error detection), 那就使用双重冗余.  
* 如果模块的错误有很大的概率无法被检测, 则使用三重冗余.  
* 如果不同类型的模块可以互补替代, 则优先使用多样化冗余.

再增加一个不使用冗余的规则.
* 出错可以被检测, 同时短期内不会对驾驶造成影响.


### 3.5. 结构


#### 3.5.1. 系统

![](http://blog.iotwrt.com/images/system.svg)


#### 3.5.2. 硬件容器

![](http://blog.iotwrt.com/images/system2.svg)

##### 3.5.2.1. 电源

电源上的冗余是底盘平台设计要综合考虑的事情, 所以图中就不再画出.  
假设底盘已经有12v主电源, 12v备用电池的概念, 支撑传感器域和计算域的电源.

对计算平台的部分来说, 其电压容限比较小, 最常见的共因失效就是电源.  
因此对计算平台ABC来说, 必须板载独立的[PMIC或者电源隔离](https://www.maximintegrated.com/content/dam/files/design/technical-documents/white-papers/Balancing-Power-Supply-Requirements-in-ADAS-Applications-cn.pdf), 同时可以在板级采用不同的设计来增加多样性.

##### 3.5.2.2. 总线

图上没有解释总线的设计.  
这是因为总线有很多成熟的技术方案, 可靠性也比较好, 都会有信息冗余/硬件冗余的支持.  

对无人车来说, 车载以太网是比较好的选择.  
现有车载以太网方案里很多都是针对娱乐/ADAS系统设计的, 可能不太合适无人车.  
适合自动驾驶设计的, 可以看[Time-Triggered Ethernet](https://www.tttech.com/technologies/time-triggered-ethernet/).

##### 3.5.2.3. 传感器

imu:  
imu失效会快速导致无人车定位的误差, 引起重大的安全风险, 所以必须采用硬件冗余.  
imu可以用confidence决定数据采用, 因此采用双冗余的方式即可.  
<small>参考: [无人机IMU三冗余](https://www.itread01.com/content/1546191209.html)</small>

雷达/摄像头:  
障碍物识别相关的传感器以多样化互补为主, 这里的冗余能力主要取决于算法融合的效果.  
不过除了障碍物识别, 一些边角的感知需求, 比如说红绿灯识别/信号灯识别/车道线识别, 是无法基于雷达的数据获取的, 
则有必要考虑在摄像头部分做硬件冗余.

其他:  
GPS和时间同步等并不影响短期内的驾驶行为, 出错后车辆可以完成自主停车操作, 因此不做备份.  
如果GPS失效几率会比较大, 影响到无人车运行效率, 则再考虑添加冗余.


##### 3.5.2.4. 计算系统

备份安全平台:  
物理隔离的备份安全平台, 在百度apollo和waymo的相关透露信息中都有提到.备份安全平台在硬件设计上主要是能采用更可靠的硬件/驱动程序, 使用已经被广泛验证的平台和技术, 不像计算单元那样受限制于计算能力. 如此, 为整体带来Design Diversity. 

计算平台:  
按经验来说, 计算平台硬件出现失效的概率应该是不大的, 做双冗余即可. 或者不做冗余关系也不大, 如果有备份安全平台的存在.  
此处做三冗余, 主要出于软件上的考虑.  

Control ECUs:
以百度apollo为例, 其Trajectory Control, Longitudinal/Lateral Control是直接在计算平台内做的.此处分到ECU上, 主要是看所有车厂放出的架构里, Control都是在ECU上做的, 比如说[BMW](https://roscon.ros.org/2015/presentations/ROSCon-Automated-Driving.pdf).
当然他们这么做, 多半是为了直接复用ADAS上自适应巡航, 自动泊车的模块.  
我们这里这样选择, 主要是出于Diversity, 成熟设计稳定性, share ability的原故. 


##### 3.5.2.5. 车联网

网络的风险主要在于空口问题引起的延迟/断网, 因此采用不同网络并存.  
网络虽然不是自动驾驶所必须的部分, 但是出于监控的需要, 硬件上也有必要使用双冗余.


#### 3.5.3. 软件组件
<!-- 
, 大概猜测, 其主要功能可能有:
* 监控/巡视
* 靠边停车的能力
* 熔断安全(防撞)

除了这些, 备份安全平台还有非常多的想象能力, 只要符合"可靠"这样特点的功能, 都可以引入. -->

<!-- 突然想到, 如果有足够强的虚拟化机制 -->

![](http://blog.iotwrt.com/images/system3.svg)

Control ECUs:  
都是成熟设计, 省略......

计算平台:  
采用三冗余的设计, 从容错的角度上讲.
* 解决因为程序异常, 包括崩溃/挂起/循环/性能下降, 引起的Byzantine Faults
* 可以布置多样性的算法策略, 从design上减少共因失效. 

从软件角度上讲, 更多的算力和融合能力可以改善算法的Performance Limitation.

基于决策和路径规划来做Vote的实现, 可以看看这篇文章.

备份安全平台:


关于熔断安全,

### 3.6. 分析设计

对上述设计基于失效概率/安全等级进行分析    
省略......


### 3.7. 其他

加入的这些结构会增加成本, 增加复杂度, 影响驾驶效率等等.  
最终还是要靠合理的论证和测试, 确定如何平衡安全与成本与效率.  

