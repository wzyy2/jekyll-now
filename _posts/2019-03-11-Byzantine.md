---
layout: post
title: 容错设计与无人车
category: OS
tag: Autonomous Drive System
comments: 1
---

![](http://blog.iotwrt.com/images/safety.svg)
<small>(纯属YY, 请勿参考......)  </small>

<small>PS: [最近737-MAX事故了解下](https://baijiahao.baidu.com/s?id=1627695537914121317&wfr=spider&for=pc), 很贴合主题..... </small>

## 背景

前几天看了一个ppt, [Fault-Tolerance in Avionics Systems](https://cs.unc.edu/~anderson/teach/comp790/papers/fault_tolerance_avionics.pdf).    
这个ppt比较易懂的解释了航空系统中的容错机制, 感觉很有意思.   

之前也看过相关新闻, 像SpaceX是在火箭和飞船的控制上粗放的使用了n+个Linux x86电脑, 主要通过额外的容错机制来做到robust.   
所以就遐想一下, 如果将自动驾驶车辆按照航天器标准设计容错, 会是什么样子.  

### 介绍

* Fault Tolerance: 指错误检测, 损坏评估, 故障隔离以及从错误中恢复的能力.   
* Fault:可能导致系统或功能失效的异常条件, 可译为“故障”.  
* Error:计算、观察或测量值或条件, 与真实、规定或理论上正确的值或条件之间的差异, 可译为“错误”.
* Failure:当一个系统不能执行所要求的功能时, 即为Failure, 可译为“失效”.   

现代Fault Tolerance主要关注两种Fault, Byzantine Faults 和 Common Mode Faults.  


### [Byzantine Faults(拜占庭将军问题)](https://baike.baidu.com/item/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%86%E5%86%9B%E9%97%AE%E9%A2%98/265656?fr=aladdin).  

直接看链接里的解释把, 就不翻译ppt了.

#### Fault Containment Region

* 系统可以分成多个Fault Containment Region.
* FCR外的错误不能导致其内部错误, 其内部错误也不能传播到外部.
* 实际中, FCR通常要求独立的处理器，存储器，IO，数据，以及电源.

#### Voting

* Voting is one method of using multiple FCRs, where a voter takes output from the FCRs and decides the correct output.
* 3 varieties:
    * Exact: output values must match bit for bit.
    * Approximate: values must be w/in a certain range of avg.
    * Mechanical: system physically creates correct output. E.g. 
        * Each component provides a fraction of force needed to move output.
        * A bad component will have its output overwhelmed by good

##### Voting Application

* Your messengers must be Byzantine Resilient also.
* Inputs, such as samples of sensor data, can also use a voter.
* Most aircraft provide redundant sensors.
* Transmission media can use voters and multiple connections.
* Voters are components! They must be extremely reliable or the system is pointless.
* Voting is often used to vote on the state of a component.



#### Triple Modular Redundancy(三重模块冗余)
* Originated with Apollo.
* 3 systems operate on input. Majority value or average wins.
* The voter can optionally shutdown and reconfigure bad components.
* Created and used before Byzantine Generals’ paper, though it solves the same problem.
* Still widely in use today.


![](http://blog.iotwrt.com/images/sc.png)

#### Replication

* This is not voting. Uses simpler hardware or software to reconcile outputs.
    * Command / Monitor: command is primary CPU, monitor merely checks output of command.
    * Primary / Backup: same as C/M, but backup computer can take over as primary.
    * Hot Swap: backup always has same state as primary for instantaneous replacement.


#### Dual - Dual
* Two pairs of each processing component, both executing on all input.
* In each pair, both components send their output to simple hardware comparator. If it finds that output differs, switches over to other pair until first pair reconfigured / replaced.
* Used in lots of old interplanetary satellites.

#### Effects on Software

* Programs need to be written without knowledge of hardware resilience.
* Even operating systems are unaware of Byzantine Resilience, instead software just assumes that Byzantine Errors don’t happen.
* The hardware and software used to detect and isolate faults, mask errors, and reconfigure components are kept separate form any operational software.
* Simplifies code.

#### Error Detection
* If bitwise comparison with other components fails, declare failed component faulty. Most voting systems work this way.
* Thresholds / Reasonableness: check that component output is within a certain threshold. 
    * E.g. Asked to calculate time, returns 25 o’clock or STRAWBERRY
* Built-in Test: components check themselves for failures.
    * Power-on BIT: checks itself at startup. Think POST.
    * Continuous BIT: components check themselves periodically during slack time.

#### Component Synchronization

* For components to work together smoothly, their clocks must be synchronized.
* Clocks themselves must be Byzantine Resilient (to a certain degree).
* Surprise, redundancy needed.
* Voting used to determine the specific time.
* After time determined, push out to all Fault Containment Regions.

#### Reconfiguration

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

#### Note on Buses
* The buses (such as communication or power, if you don’t have independent generators) must also be Byzantine Resilientish.
* Similar techniques would apply with small changes. E.g.
    * Communication buses might have voting components located at each interface to an FCR, with multiple buses feeding in
    * Backup buses in case one is damaged
    * Etc..

### Common Mode Failures(共因失效)

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

#### Fault Avoidance

* Don’t reinvent wheel.
    * Problems arise because manufactures like to design own hardware
    * Existing hardware has already been tested / verified in field
    * COTS hardware offers a cheap alternative with lots of unofficial testing **
    * Conform to existing standards for similar reasons
* Formal Methods
    * The Space Shuttle takes this approach
* Design Diversity
    * Run systems using different hardware / software designed by different teams

#### Fault Removal

* Basically means test the system rigorously to find problems.
* Fault Insertion: fake faults to make sure you system can handle them.

#### Fault Tolerance
* Watchdog Timers: task which occasionally runs and verifies system state.
    * Could find an invalid PC, for example
* Hardware Exceptions: when invalid hardware operation found, throw and catch hardware exceptions.
* Runtime Checks: software exceptions, etc.
* Program Checkpointing: periodically save state, reload state when an error is found. Impractical often.
* Restarting: always an option.

### Fault Tolerant Scheduler(容错调度)
* 假设我们的系统有一些重复的处理模块.
* 需求:
    * 任务调度在模块间的调度需要满足timing requirements.
    * 要提供相关容错的机制, 例如管理重复执行和任务成功检查
* Surprise, the solution involves executing tasks at different modules.
* 所有的任务重复和验证必须要满足deadline!

#### 硬件实现

* 每个处理模块都有一个用于同步和投票的冗余管理系统(RMS).
* RMS根据从重复任务中返回的数值以及整个处理模块的状态进行投票选择.
* 应用只会响应于所有RMS都投票的值.
* 如果RMS确定一个模块出现故障, 则该模块会受被降级, 并会得到新数据以进行恢复.

![](http://blog.iotwrt.com/images/hi.png)

#### 软件实现

* 一个任务release后, 会模块间复制多份. 每当一个模块执行完任务, 其结果将移至投票队列.
* 在投票时间内(即任务release到deadline之间), voter会执行然后计算voting queue的结果. 投票消耗时间由所使用的算法确定
* 任何添加到voting queue但未由voter处理的任务必须在下次被处理.

![](http://blog.iotwrt.com/images/si.png)

### 相关历史
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

## 无人车

基于上述背景知识, 来设想一下如何为无人车设计容错架构.

### 不同点

对航空器设计来说, 其所关注的是针对由于软硬件系统"故障失效"而导致的安全风险.  
但对运行着自动驾驶软件的无人车来说, 安全问题并不一定仅仅由于系统"故障失效"引起.  
更多的场景下, 可能是因为自身系统设计上的limitation导致的安全风险.  

因此这里需要再加入针对performance limitation的考虑.  
当然重点不放在如何找到以及改善performance limitation, 这个是扩大到整体系统安全范畴了.  
**我们主要关注的是, 针对环境超出performance limitation后的容错处理.**  


### 相关案例

#### Waymo

![](http://blog.iotwrt.com/images/waymo.png)

上面是Waymo Safety report里的Safety-Critical Systems描述.  
看描述, 基本上是使用 Replication/Backup 处理Byzantine Faults.


#### [Audi A8 zFAS](https://www.audi-technology-portal.de/en/electrics-electronics/driver-assistant-systems/audi-a8-central-driver-assistance-controller-zfas)

    zFAS有4个核心元件, 包括Mobileye的EyeQ3, 负责交通信号识别, 行人检测, 碰撞报警, 光线探测和车道线识别.英伟达的K1负责驾驶员状态检测, 360度全景.英特尔（Altera）的Cyclone V负责目标识别融合, 地图融合, 自动泊车, 预刹车, 激光雷达传感器数据处理.英飞凌的Aurix TC297T负责监测系统运行状态, 使整个系统达到ASIL-D的标准, 同时还负责矩阵大灯

具体这里的Aurix TC297T上面的处理逻辑, 我们不得而知了, 只知道是safety management unit.  
不过基本也逃不开上面所述描述的理论.

![](https://edge.slashgear.com/wp-content/uploads/2017/09/03_vernetzung-zFAS-Aktuatoren-1018x720.jpg)

![](https://edge.slashgear.com/wp-content/uploads/2017/09/Screen-Shot-2017-09-11-at-4.25.50-PM-1280x716.jpg)

![](https://pic2.zhimg.com/v2-8b1f85bba1821eab133f370d6f9d72ed_r.jpg)


**这里值得一提的是, [zFAS上各处理器之间的bus和middleware使用的TTTech的技术](https://www.tttech.com/press/tttech-and-infineon-to-speed-up-the-development-in-autonomous-driving/), 这也是一个很关键的安全设计.**



#### [elektrobit]