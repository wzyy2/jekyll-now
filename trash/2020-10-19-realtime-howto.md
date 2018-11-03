---
layout: post
title: 自动驾驶场景的实时性能需求
category: [OS]
tag: [CN]
---

接上文

## 传感器

传感器的采集是一个对实时性需求很高的场景。

### 时钟同步
Systems that interact with the real world must synchronize with it。

计算机在为来自物理世界的传感器的值需要有时间戳，必须要有个计算机参考系
* 传感器数据带物理世界时间戳
* 计算机以获取到数据的计算机时间做时间戳

在一般的传感器采集程序中，我们为某个数值打上的时间戳一般来自于自身的时间。

### IMU
### IMU

真值采集
administrator@NL-R02N001:~$ rostopic info /autogo/sensors/imu
Type: sensor_msgs/Imu

Publishers: 
 * /autogo/sensors/imu_gs1810_capt_node (http://NL-R02N001:38649/)

Subscribers: 
 * /autogo/guardian/guardian (http://NL-R02N001:37103/)
 * /autogo/planning/motion_planner_node (http://NL-R02N001:39751/)
 * /autogo/localization/msckf_ros_cainiao (http://NL-R02N001:39825/)
 * /asp/record_radar/rosbag_record (http://NL-R02N001:34921/)
 * /asp/record_other/rosbag_record (http://NL-R02N001:46497/)
 * /autogo/applications/autogo_monitor (http://NL-R02N001:39353/)
 * /autogo/applications/rostopic_7449_1540467111675 (http://NL-R02N001:39525/)
