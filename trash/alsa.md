
# DEBUG

输入 `aplay -l` 可以看设备是不是有注册上

	root@MiWiFi-R1CL-srv:~# aplay -l                                                                
	aplay: device_list:268: no soundcards found..


测试的话可以用下面这个命令，其中hw:0,0代表声卡0,通道0,以上面的dts为例，就是fennec-codec sound card 的 hdmi codec

	aplay -D hw:0,0 -d 1 -f cd /dev/zero
	
	wget http://www.hochmuth.com/mp3/Haydn_Cello_Concerto_D-1.mp3
	mv Haydn_Cello_Concerto_D-1.mp3 test.mp3
	aplay test.mp3
  
设置control也可以使用aplay命令，具体可以help一下



一个声卡包含 cpu_dai, codec_dai, 以及 dai_link 组成，分别对应 cpu dai 的 dirver，比如I²S driver, spdif driver, codec driver， 比如 rt5640 codec driver, dai_link driver，也就是 machine driver， 比如 sound/soc/rockchip/rockchip_rt5640.c。

## sound-card

有两种方式创建声卡，一种是通用的 simple-card framework，一种是传统的编写自定义的 machine
driver 来创建。

直接看upstream文档比较好，  
http://elixir.free-electrons.com/linux/latest/source/Documentation/devicetree/bindings/sound/simple-card.txt


样例：

  	sound {
		compatible = "simple-audio-card";
		simple-audio-card,format = "i2s";
		simple-audio-card,name = "rockchip,fennec-codec";
		simple-audio-card,mclk-fs = <512>;

		simple-audio-card,dai-link@0 {
			format = "i2s";
			cpu {
				sound-dai = <&i2s>;
			};
			codec {
				sound-dai = <&hdmi>;
			};

		};

		simple-audio-card,dai-link@1 {
			format = "i2s";
			cpu {
				sound-dai = <&i2s>;
			};

			codec {
				sound-dai = <&es8323>;
			};
		};
	};





# Others(拷贝的)

本章主要描述Audio的相关的概念、代码结构。

## 1.2    概念

- CPU DAI:主控端的Audio Data Interface，比如I²S,Spdif,Pdm,Tdm
- CODEC DAI：即Codec
- DAI_LINK:绑定Cpu_Dai和Codec_Dai为一个声卡，等同于Machine Driver。
- DMAENGINE：用于Cpu和I²S/Spdif等Dai之间的Dma传输引擎，实际是通过Dma来进行数据的搬运。
- DAPM：动态音频电源管理，用于动态管理Codec等的电源管理，根据通路的开启配置开关，以达到保证功能的前提下功耗尽量小。
- JACK:耳机的接口检测，大部分使用Codec自身的检测机制，小部分使用IO来进行模拟。

## 1.3    代码结构

表 1‑1 SOUND代码构成

| **项目**            | **功能**                                   | **路径**             |
| ----------------- | ---------------------------------------- | ------------------ |
| Sound soc         | 主要包含公共部分代码，包括dapm控制，jack，dmaengine，core等等 | sound/soc/         |
| rockchip platform | Rockchip平台的cpu dai的驱动，比如I²S, spdif等以及自定义声卡machine driver | sound/soc/rockchip |
| generic platform  | simple  card framework                   | sound/soc/generic  |
| codec driver      | 所有的codec driver存放位置                      | sound/soc/codecs   |

------------------
# 2    Audio开发指南

## 2.1   概述

本章描述如何添加声卡，调试声卡以及通路等。

## 2.2   音频开发指南

一个声卡包含cpu_dai, codec_dai, 以及dai_link组成，分别对应cpu dai的dirver，比如I²S driver, spdifdriver；codec driver， 比如rt5640 codecdriver；dai_link driver，也就是machine driver， 比如sound/soc/rockchip/rockchip_rt5640.c。 4.4的内核中支持两种方式创建声卡，一种是通用的simple-card framework，一种是传统的编写自定义的machine driver来创建。本文档均以rt5640为例。

### 2.2.1    Simple-card

Simple card即简单通用的machine driver， 如果simple-card框架足够满足需求，建议优先使用simple card框架，简单，方便，且易用。

1.    添加codec driver，比如添加：sound/soc/codec/rt5640.c

2.    修改sound/soc/codec/Kconfig以及Makefile加入驱动编译。

```c
sound/soc/codec/Kconfig:
config SND_SOC_RT5640
		tristate "RealtekALC5640 CODEC"
		depends on I2C
sound/soc/codec/Makefile:
snd-soc-rt5640-objs := rt5640.o
obj-$(CONFIG_SND_SOC_RT5640)   += snd-soc-rt5640.o
```
3.	menuconfig中enable simple card以及codec
```c
make menuconfig
Device Drivers  --->
[*] Sound card support --->
[*]   Advanced Linux SoundArchitecture  --->
[*]   ALSA for SoC audiosupport  --->
	[*]	ASoC support for Rockchip
	[*]  Rockchip I2S Device Driver
				CODEC drivers  --->
					[*]Realtek ALC5640 CODEC
[*]   ASoC Simple sound cardsupport
```

4.产品的DTS中添加Simple Card Node
```c
		rt5640-sound {
				compatible ="simple-audio-card";
				simple-audio-card,format = "i2s";
				simple-audio-card,name = "rockchip,rt5640-codec";
				simple-audio-card,mclk-fs = <256>;
				simple-audio-card,widgets =
						"Microphone", "Mic Jack",
						"Headphone", "Headphone Jack";
						simple-audio-card,routing =
						"Mic Jack", "MICBIAS1",
						"IN1P", "Mic Jack",
						"Headphone Jack", "HPOL",
						"Headphone Jack","HPOR";
				simple-audio-card,cpu {
						sound-dai = <&i2s_8ch>;
				};
				simple-audio-card,codec {
						sound-dai = <&rt5640>;
				};
		};
&i2c1 {
		status = "okay";
		rt5640: rt5640@1c {
				#sound-dai-cells = <0>;
				compatible ="realtek,rt5640";
				reg = <0x1c>;
				clocks = <&cruSCLK_I2S_8CH_OUT>;
				clock-names = "mclk";
				realtek,in1-differential;
			};
};
```
需要注意的是，如上rt5640的clocks即mclk， upstream代码遵循谁使用clk谁申请的原则，所以后续自己添加的codec driver，如果有使用外部clk作为mclk，需要做同样的适配。

### 2.2.2    DRM HDMI Audio

Rk的4.4内核中，目前HDMI存在两套框架，一套是RK自定义HDMI框架，一套是DRM标准的HDMI框架，相应的，音频也有两套。所以，HDMI的音频配置需要和使用的HDMI框架相匹配。如下以RK3399为例，其他芯片类似。

1.   RK自定义HDMI框架音频配置，对应的HDMI驱动选择如下时：

```c
   hdmi_rk_fb: hdmi-rk-fb@ff940000 {
                compatible = "rockchip,rk3399-hdmi";
				……
		}；
```

需要选用如下的音频配置：

```c
hdmi_sound: hdmi-sound {
				compatible = "simple-audio-card";
				simple-audio-card,format = "i2s";
				simple-audio-card,mclk-fs = <256>;
				simple-audio-card,name = "rockchip,hdmi";
				simple-audio-card,cpu {
						sound-dai = <&i2s2>;
				};
				simple-audio-card,codec {
					sound-dai = <&dw_hdmi_audio>;
				};
			};

			dw_hdmi_audio: dw-hdmi-audio {
				status = "okay";
				compatible = "rockchip,dw-hdmi-audio";
				#sound-dai-cells = <0>;
			};
```

2.   DRM HDMI框架音频配置，当hdmi选用如下驱动时：
```c
hdmi: hdmi@ff940000 {
				compatible = "rockchip,rk3399-dw-hdmi";
				……
				#address-cells = <1>;
				#size-cells = <0>;
				#sound-dai-cells = <0>;
		};
```

​    需要选用如下的音频配置：
```c
make menuconfig
Device Drivers  --->
Graphics support  --->
Display Interface Bridges  --->
<*> Synopsis Designware I2S Audio interface

		hdmi_sound: hdmi-sound {
				status = "okay";
				compatible = "simple-audio-card";
				simple-audio-card,format = "i2s";
				simple-audio-card,mclk-fs = <256>;
				simple-audio-card,name = "rockchip,hdmi";
				simple-audio-card,cpu {
						sound-dai = <&i2s2>;
				};
				simple-audio-card,codec {
						sound-dai = <&hdmi>;
				};
			};
```
### 2.2.3    自定义的Machine Driver
当Simple card不足以满足需求时，这个时候就需要编写相对应的Machine driver，比如：sound/soc/rockchip/rockchip_rt5640.c，然后在这个machine driver添加特殊的控制，路由等等。这里不做举例，延续原有的格式，以及目录下均有参考代码可作为参照。
### 2.2.4    声卡调试

1.     通过如下命令确认声卡是否注册成功
```c
root@rk3366:/ # cat /proc/asound/cards
 0[rockchiprt5640c]:rockchip_rt5640 - rockchip,rt5640-codec
						rockchip,rt5640-codec
root@rk3366:/ # ls -l /dev/snd/
crw-rw---- system   audio    116,  2 2013-01-18 08:51 controlC0
crw-rw---- system   audio    116,  4 2013-01-18 08:51 pcmC0D0c
crw-rw---- system   audio    116,  3 2013-01-18 08:51 pcmC0D0p
```
2.    通过命令行播放录制调试声卡：

      播放：一般播放1khz 0db正弦波，然后在codec输出端示波器简单测量是否失真，杂音，然后再使用音频分析仪测试指标。

```c
root@rk3366:/ # tinyplay
Usage: tinyplay file.wav [-D card] [-d device] [-p period_size] [-nn_periods]
|root@rk3366:/ # tinyplay /sdcard/test44.wav -D 0 -d 0 -p 1024 -n 3
Playing sample: 2 ch, 44100 hz, 32 bit
```
录制：
```c
root@rk3366:/ # tinycap
Usage: tinycapfile.wav [-D card] [-d device] [-c channels] [-r rate] [-b bits] [-pperiod_size] [-n n_periods]
|root@rk3366:/ # tinycap/sdcard/rec.wav-D 0 -d 0 –c 2 –r 44100 –b 16 –p 1024 –n 3
```
3.通过命令行调试声卡的通路：
一般复杂的codec可提供各种通路的配置，如下图：分别是数字部分通路和模拟部分通路，通路旁边都有标注控制的寄存器bit，codec driver负责将这些控制实例化为kcontrol，提供给上层设置切换通路使用，实际的调试方法为从数字部分的通路开始，比如DACDAT然后顺着找出一条最优的路径到达模拟输出端，比如HPOUT。然后通过tinymix控制路径上的相应节点开关，打通通路。 
![Digital-mixer-path](Rockchip-Developer-Guide-linux4.4-Audio/Digital-mixer-path.png)

![Audio-mixer-path](Rockchip-Developer-Guide-linux4.4-Audio/Audio-mixer-path.png)
4.tiny mix 调试通路：

```c
root@rk3366:/ # tinymix
Mixer name: 'rockchip,rt5640-codec'
Number of controls: 123
ctl     type    num    name                                    value
0       BOOL    1      Mono Playback Switch                    Off
1       INT     2      Mono DAC Playback Volume                175 175
2       BOOL    2      Speaker Channel Switch                  Off Off
3       INT     2      Speaker Playback Volume                 31 31
4       BOOL    2      HP Channel Switch                       Off Off
```
可通路ctr id或者name来控制，例子如下，不带val设置时，为查询该mix的当前状态
``` c
root@rk3366:/ # tinymix 0 1
root@rk3366:/ # tinymix 0
Mono Playback Switch: On

root@rk3366:/ # tinymix"Mono Playback Switch" 1
root@rk3366:/ # tinymix"Mono Playback Switch"
Mono Playback Switch: On
```
5.     声卡功能以及通路调试ok后，需要把通路配置配置到hal层，然后可以配置不同场景下的通路路由，通路的配置即为tinymix配置成功后的通路列表的值，把这些值做成相应codec_config.h加入到hal中，举例如下：
```c
hardware/rockchip/audio/tinyalsa_hal/codec_config/rt5640_config.h
\#ifndef _RT5640_CONFIG_H_
\#define _RT5640_CONFIG_H_
\#include "config.h"
const struct config_control rt5640_speaker_normal_controls[] = {
	{
		.ctl_name = "DAIselect",
		.str_val ="1:2|2:1",
		},
		{
		.ctl_name = "MonoDAC Playback Volume",
		.int_val = {175, 175},
		},
	{
	.ctl_name = "DAC2Playback Switch",
		.int_val = {on, on},
},
……
hardware/rockchip/audio/tinyalsa_hal/codec_config/config_list.h
struct alsa_sound_card_config sound_card_config_list[] = {
		……
		{
			.sound_card_name ="rockchiprt5640c",
			.route_table =&rt5640_config_table,
			},
			……
```
通过以上步骤即完成基本的声卡创建，简单调试， 以上使用的tinyplay, tinycap, tinymix代码位于android/external/tinyalsa中，如果系统中没有该命令，可进到该目录执行mm生成相应的命令。

------------------
# 3  常用调试方法
## 3.1   常用调试方法：
1.查看codec寄存器，I²S寄存器，spdif寄存器等等，出现问题时，往往需要常看寄存器的状态是否正常，来定位分析问题。

a, 凡是使用regmap的驱动， 在/sys/kernel/debug/regmap都有相应的查询入口，如下：
```c
root@rk3366:/sys/kernel/debug/regmap# ls
0-001c
0-0040
1-001c
ff880000.spdif
ff898000.i2s-8ch
```
例如：1-001c为rt5640的i2c地址，挂载在i2c1, codec地址为0x1c，那么此目录中的registers即为codec的register，其他类似。
2.Xrun debug， 一般用于debugunderrun或者overrun，出现此两者情况时内核会打印log协助问题的定位分析。Menuconfig中需要开启如下选项：
```c
[*]Advanced Linux Sound Architecture  --->
[*]Debug
[*]	More verbose debug
[*]	Enable PCM ring buffer overrun/underrundebugging
```
然后在对应声卡/proc/asound/card0/xrun中写入相应的值，值如下：
```c
\#define XRUN_DEBUG_BASIC       (1<<0)
\#define XRUN_DEBUG_STACK       (1<<1)  /* dump also stack*/
\#define XRUN_DEBUG_JIFFIESCHECK (1<<2)  /* do jiffies check */
```
比如 echo 1 > xrun或者echo 3 > xrun 或者 echo 7 > xrun开启所有debug信息检测。
3.通过查看clk tree确认相应的audio  clk是否正常，比如mclk：如下为采样率为44100hz的mclk： 11.2896M。
```c
c at/sys/kernel/debug/clk/clk_summary | grep i2s
i2s_2ch_src                     0            0  576000000          0 0
i2s_2ch_frac                 0            0   28800000          0 0
i2s_8ch_src                     0            0  576000000          0 0
i2s_8ch_frac                 0            0   11289600          0 0
i2s_8ch_pre               0            0   11289600          0 0
sclk_i2s_8ch           0            0   11289600          0 0
i2s_8ch_clkout           0            0   11289600          0 0
```

4.要学会使用示波器测量音频的信号， 软件方式的确认有时会有误差，最精确最根本的方式就是确认音频clk是否正常，满足规范。音频的信号包含mclk, bclk, lrck, data。需要确认信号幅度是否正常，如果io电压为3.3v，测试出来的信号幅值应当在3.3v左右。如果幅值太低，则会照成采集不到数据而无声。Clk的频偏也不宜过大，有可能会照成杂音。Bclk, lrck要符合设置的采样率，如果不相符，则会照成音频快进或者播放缓慢。

5.播放测试：一般播放1khz 0db正弦波，然后使用示波器确认输出是否有削顶失真，相位失真，杂音等。

6.录音测试：可使用信号发生器产生1khz的波形从codec模拟端导入，然后录制波形，可以通过回放来确认波形是否正常，无失真，或者使用电脑上的软件工具adobe audition来分析底噪等等基本指标。

7.基本功能过完后，需要使用音频分析仪进行codec后续的指标测试以及调优。
