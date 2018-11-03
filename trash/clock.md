# 1 方案概述

## 1.1  概述

本章主要描述时钟子系统的相关的重要概念、时钟方案、总体流程、代码结构。

## 1.2  重要概念

l **时钟子系统**

这里讲的时钟是给SOC各组件提供时钟的树状框架，并不是内核使用的时间，和其他模块一样，CLK也有框架，用以适配不同的平台。适配层之上是客户代码和接口，也就是各模块（如需要时钟信号的外设，USB等）的驱动。适配层之下是具体的SOC台的时钟操作细节。

l  **时钟树结构**

可运行Linux的主流处理器平台，都有非常复杂的Clock Tree，我们随便拿一个处理器的SPEC，查看Clock相关的章节，一定会有一个非常庞大和复杂的树状图，这个图由Clock相关的器件，以及这些器件输出的Clock组成。

l **相关器件**

Clock相关的器件包括：用于产生Clock的Oscillator（有源振荡器，也称作谐振荡器）或者Crystal（无源振荡器，也称晶振）；用于倍频的PLL（锁相环，PhaseLocked Loop）；用于分频的Divider；用于多路选择的Mux；用于Clock Enable控制的与门；使用Clock的硬件模块（可称作Consumer）；等等。

## 1.3  时钟方案

每一个SOC都有自己的时钟分配方案，主要是包括PLL的设置，各个CLK的父属性、DIV、MUX等。芯片不同，时钟方案是有差异的。

![clk-003-clk-tree-kernel4.4](Rockchip-Developer-Guide-linux4.4-Clock/clk-003-clk-tree-kernel4.4.png)

[图 ]()1‑1 CLK 时钟树的示例图

![clk-005-clk-conf-kernel4.4](Rockchip-Developer-Guide-linux4.4-Clock/clk-005-clk-conf-kernel4.4.png)

[图 ]()1‑2时钟分配示例图

## 1.4  总体流程



![clk-007-config-flow-kernel4.4](Rockchip-Developer-Guide-linux4.4-Clock/clk-007-config-flow-kernel4.4.png)

[图 ]()1‑3时钟配置流程图

** **

主要内容包括（不需要所有clk都支持）：

（1）      Enable/Disable CLK。

（2）     设置CLK的频率。

（3）      选择CLK的Parent。 

## 1.5  代码结构

CLOCK的软件框架由CLK的clk-rk3xxx.c（clk的寄存器描述、clk之间的树状关系等）、Device driver的CLK配置和CLK API三部分构成。这三部分的功能、CLK代码路径如表1-1所示。

[表]() 1‑1 CLK代码构成

| **项目**                     | **功能**                                   | **路径**                                   |
| -------------------------- | ---------------------------------------- | ---------------------------------------- |
| Clk-rk3xxx.c  Rk3xxx-cru.h | .c中主要是clk的寄存器描述、clk之间的树状关系描述等。  .h中是clk的ID定义，通过ID匹配clk name。 | Drivers/clk/rockchip/clk-rk3xxx.c  Include/dt-bindings/clock/rk3xxx-cru.h |
| RK  特别的处理                  | 1、处理RK的PLL时钟  2、处理RK的CPU时钟等              | Drivers/clk/rockchip/clk-xxx.c           |
| CLK  API                   | 提供linux环境下供driver调用的接口                   | Drivers/clk/clk-xxx.x                    |

--------------
# 2    CLOCK开发指南

## 2.1 概述
本章描述如何修改时钟配置、使用API接口及调试CLK程序。

## 2.2  时钟的相关概念

### 2.2.1 PLL

锁相环，是由24M的晶振输入，然后内部锁相环锁出相应的频率。这个是SOC所有CLOCK的时钟的源。SOC的所有总线及设备的时钟都是从PLL分频下来的。RK平台主要PLL有:

[表 ]()2‑1 CLK PLL描述

| **PLL** | **子设备** | **用途**              | **备注**                                   |
| ------- | ------- | ------------------- | ---------------------------------------- |
|         | ARMCLKL | CPU小核的时钟            | 一般只给CPU使用，因为CPU会变频，PLL会根据CPU要求的频率变化      |
| APLLB   | ARMCLKB | CPU大核的时钟            | 一般只给CPU使用，因为CPU会变频，PLL会根据CPU要求的频率变化      |
| DPLL    | Clk_DDR | DDR的时钟              | 一般只给DDR使用，因为DDR会变频，DPLL会根据DDR要求变化        |
| GPLL    |         | 提供总线、外设时钟做备份        | 一般设置在594M或者1200M，保证基本的100、200、300、400M的时钟都有输出 |
| CPLL    |         | 其他设备做备份             | 一般可能是400、500、800、1000M。                  |
| NPLL    |         | GMAC或者给其他设备做备份      | 一般可能是1000M。                              |
| VPLL    |         | HDMI、VOP使用          | 频率根据显示分辨率定，支持任意频率对jitter有要求。             |
| PPLL    |         | PMUCRU时钟，给PMU模块提供时钟 | 一般可能是676M，可以出26M。                        |

### 2.2.2   ACLK、PCLK、HCLK

ACLK是设备的总线的CLK，PCLK跟HCLK一般是用于寄存器读写的。而像CLK_GPU是GPU的控制器的时钟。

我们SOC的总线有ACLK_PERI、HCLK_PERI、PCLK_PERI、ACLK_BUS、HCLK_BUS、PCLK_BUS.各个设备的总线时钟会挂在上面这些时钟下面，如下图结构：

![clk-009-bus-structure-kernel4.4](Rockchip-Developer-Guide-linux4.4-Clock/clk-009-bus-structure-kernel4.4.png)

![clk-011-diff-freqkernel4.4](Rockchip-Developer-Guide-linux4.4-Clock/clk-011-diff-freqkernel4.4.png)

图 2‑1总线时钟结构

（如：EMMC想提高自己设备的总线频率以实现其快速的数据拷贝或者搬移，可以提高ACLK_PERI来实现）

RK3399上设计将高速和低速总线彻底分开，分成高速：ACLK_PERIHP、HCLK_PERIHP、PCLK_PERIHP；低速：ACLK_PERILP0、HCLK_PERILP0、PCLK_PERILP0、HCLK_PERILP1、PCLK_PERILP1。这样做是为了功耗最优，根据不同的需求可以设置不同的总线频率。（具体每个设备在哪条总线下详细见时钟图）

可以参考如下（EMMC、GMAC、USB等有自己的ACLK）：

![clk-013-kernel4.4](Rockchip-Developer-Guide-linux4.4-Clock/clk-013-kernel4.4.png)

### 2.2.3 GATING

Clock的框架中有很多的Gating，这个主要是为了降低功耗使用，在一些设备关闭，Clock不需要维持的时候就可以关闭Gating，来节省功耗。

我们Clock的框架的Gating是按照树的结构，有父子属性。Gating的开关是有一个引用计数机制的，使用这个计数来实现Clock打开时，会遍历打开其父Clock。在子Clock关闭时，父Clock会遍历所有的子Clock，在所有的子都关闭的时候才会关闭父Clock。

（如：I²S2在使用的时候，必须要打开下面这三个Gating，但是软件上只需要开最后一级的Gating，我们的时钟结构会自动的打开其Parent的Gating）

![clk-015-gating-kernel4.4](Rockchip-Developer-Guide-linux4.4-Clock/clk-015-gating-kernel4.4.png)

图 2‑2 GATING示例图

## 2.3 时钟配置

### 2.3.1 时钟初始化配置

与LINUX3.10不同的，4.4内核时钟的初始化使用of_clk_set_defaults然后解析assigned-clocks获取CLK的ID，然后获取clk name，解析assigned-clock-parents获取需要设置的parent，解析assigned-clock-rates获取需要设置的频率。

Rk3xxx.dtsi中：
```c
cru: clock-controller@ff760000 {
		compatible = "rockchip,rk3399-cru";
		reg = <0x0 0xff760000 0x0 0x1000>;
		#clock-cells = <1>;
		#reset-cells = <1>;
		assigned-clocks =
			<&cru ACLK_VOP0>, <&cru HCLK_VOP0>,
			<&cru ACLK_VOP1>, <&cru HCLK_VOP1>,
			<&cru ARMCLKL>, <&cru ARMCLKB>,
			<&cru PLL_GPLL>, <&cru PLL_CPLL>,
			<&cru PLL_NPLL>,
			<&cru ACLK_PERIHP>, <&cru HCLK_PERIHP>,
			<&cru PCLK_PERIHP>,
			<&cru ACLK_PERILP0>, <&cru HCLK_PERILP0>,
			<&cru PCLK_PERILP0>,
			<&cru HCLK_PERILP1>, <&cru PCLK_PERILP1>;
	};}
```
1.频率
CLOCK TREE初始化时设置的频率：
```c
			assigned-clock-rates =
				<400000000>,  <200000000>,
				<400000000>,  <200000000>,
				<816000000>,  <816000000>,
				<594000000>, <800000000>,
				<1000000000>,
				<150000000>, <75000000>,
				<37500000>,
				<100000000>,  <100000000>,
				<50000000>,
				<100000000>,   <50000000>;
				};
```
2.    Parent

CLOCK TREE初始化时设置的parent：
```c
assigned-clock-parents =
			<&cru VPLL>, <&cru VPLL>,
			<&cru CPLL>, <&cru CPLL>,
			<&cru APLLL>, <&cru APLLB>,
			<&cru GPLL>, <&cru GPLL>,
			<&cru GPLL>,
			<&cru GPLL>, <&cru GPLL>;
```
注意：

Assinged的配置Parent和Rate时候，需要跟Assigned-Clocks一一对应，因为设置是按照Assigned-Clocks的CLK ID进行查找并设置的。

3.    Gating

CLOCK TREE初始化时是否默认enable：

（1）      需要在clk-rk3xxx.c中增加critical配置，主要在rk3399_cru_critical_clocks中增加需要默认打开的CLK name，一旦增加CLK的计数被加1，后面这个CLK将不能被关闭。
```c
static const char *constrk3399_cru_critical_clocks[] __initconst = {
		"aclk_usb3_noc",
		"aclk_gmac_noc",
		"pclk_gmac_noc",
		"pclk_center_main_noc",
		"aclk_cci_noc0",
		"aclk_cci_noc1",
		"clk_dbg_noc",
		"hclk_vcodec_noc",
		"aclk_vcodec_noc",
		"hclk_vdu_noc",
		"aclk_vdu_noc",
}；
```
（2）CLK的定义时候增加flag属性CLK_IGNORE_UNUSED，这样即使这个CLK没有使用，在最后CLK关闭没有用的CLK时也不会关闭这个。但是在CLK TREE上看到的enable cnt还是0，但是CLK是开启的。
```c
GATE(PCLK_PMUGRF_PMU,"pclk_pmugrf_pmu", "pclk_pmu_src", CLK_IGNORE_UNUSED,RK3399_PMU_CLKGATE_CON(1), 1, GFLAGS),
```
### 2.3.2    时钟ID

LINUX4.4上对CLK的操作都是引用CLK ID，而ID如何获取？
在clk-rk3xxx.c中找到需要控制的clk(可以用过name查找)：

![clk-017-kernel4.4](Rockchip-Developer-Guide-linux4.4-Clock/clk-017-kernel4.4.png)

有一些是没有ID的，因为暂时不需要引用并控制的时钟就没有增加ID。

### 2.3.3  主要的CLK注册类型函数

常用的有如下几种：

**GATE**：描述GATING，主要包括CLK ID、类型、GATING的寄存器偏移地址、BIT位等。

**MUX**：描述SLECT，主要包括CLK ID、类型、MUX的寄存器偏移地址、BIT位等。

**COMPOSITE**：描述有MUX、DIV、GATING的CLK，主要包括CLK ID、类型、MUX、DIV、GARING的寄存器偏移地址、BIT位等。

![clk-019-kernel4.4](Rockchip-Developer-Guide-linux4.4-Clock/clk-019-kernel4.4.png)

![clk-021-kernel4.4]((Rockchip-Developer-Guide-linux4.4-Clock/clk-021-kernel4.4.png)

Clk-rk3xxx.c中的使用，使用这些CLK的注册函数，描述此CLK的类型，寄存器及父子关系等。

![clk-023-kernel4.4]((Rockchip-Developer-Guide-linux4.4-Clock/clk-023-kernel4.4.png)

### 2.3.4 Driver的时钟配置

1. 获取CLK指针

（1）     DTS设备结点里添加clock引用信息（推荐）

DTS：
```c
clocks = <&cru SCLK_TSADC>, <&cru PCLK_TSADC>;
clock-names = "tsadc", "apb_pclk";
```
（CLK引用的是ID，通过ID找到CLK NAME）
Driver code：
```c
dev->pclk =devm_clk_get(&pdev->dev, "tsadc");
dev->clk =devm_clk_get(&pdev->dev, "apb_pclk");
```
## 2.4 CLOCK API接口

### 2.4.1  主要的CLK API

1. 头文件：
```c
#include <linux/clk.h>
clk_prepare/ clk_unprepare
clk_enable/ clk_disable
clk_prepare_enable / clk_disable_unprepare
clk_get/ clk_put
devm_clk_get/ devm_clk_put
clk_get_rate / clk_set_rate
clk_round_rate
```
2. 获取CLK指针
```c
struct clk*devm_clk_get(struct device *dev, const ch*id)（推荐）
struct clk*clk_get(struct device *dev, const char *id)
```
**3.   **准备使能CLK
```c
int clk_prepare(structclk *clk)
/*开时钟前调用，可能会造成休眠，所以把休眠部分放到这里，可以原子操作的放到enable里*/
voidclk_unprepare(struct clk *clk)
/*prepare的反操作*/
int clk_enable(structclk *clk)
/*原子操作，打开时钟，这个函数必须在产生实际可用的时钟信号后才能返回*/
void clk_disable(structclk *clk)
/*原子操作，关闭时钟*/
```
（1）  clk_enable/clk_disable，启动/停止clock。不会睡眠。
（2）  clk_prepare/clk_unprepare，启动clock前的准备工作/停止clock后的善后工作。可能会睡眠。
（3）  可以使用clk_prepare_enable / clk_disable_unprepare，clk_prepare_enable / clk_disable_unprepare(或者clk_enable / clk_disable) 必须成对，以使引用计数正确。
**注意：**

prepare/unprepare，enable/disable的说明：

这两套API的本质，是把clock的启动/停止分为atomic和non-atomic两个阶段，以方便实现和调用。因此上面所说的“不会睡眠/可能会睡眠”，有两个角度的含义：一是告诉底层的clock driver，请把可能引起睡眠的操作，放到prepare/unprepare中实现，一定不能放到enable/disable中；二是提醒上层使用clock的driver，调用prepare/unprepare接口时可能会睡眠，千万不能在atomic上下文（例如内部包含mutex 锁、中断关闭、spinlock锁保护的区域）调用，而调用enable/disable接口则可放心。

另外，clock的操作为什么需要睡眠呢？这里举个例子，例如enable PLL clk，在启动PLL后，需要等待它稳定。而PLL的稳定时间是很长的，这段时间要把CPU交出（进程睡眠），不然就会浪费CPU。

最后，为什么会有合在一起的clk_prepare_enable/clk_disable_unprepare接口呢？如果调用者能确保是在non-atomic上下文中调用，就可以顺序调用prepare/enable、disable/unprepared，为了简单，framework就帮忙封装了这两个接口。

**4.  **设置CLK频率

```intclk_set_rate(struct clk *clk, unsigned long rate)```（单位Hz）

### 2.4.2  示例

**DTS**
```c
		tsadc: tsadc@ff260000 {
			compatible = "rockchip,rk3399-tsadc";
			reg = <0x0 0xff260000 0x0 0x100>;
			interrupts = <GIC_SPI 97 IRQ_TYPE_LEVEL_HIGH>;
			rockchip,grf = <&grf>;
			clocks = <&cruSCLK_TSADC>, <&cru PCLK_TSADC>;
			clock-names = "tsadc","apb_pclk";
			assigned-clocks = <&cruSCLK_TSADC>;
			assigned-clock-rates =<750000>;
			resets = <&cru SRST_TSADC>;
			reset-names = "tsadc-apb";
			pinctrl-names = "init", "default","sleep";
			pinctrl-0 = <&otp_gpio>;
			pinctrl-1 = <&otp_out>;
			pinctrl-2 = <&otp_gpio>;
			#thermal-sensor-cells = <1>;
			rockchip,hw-tshut-temp = <95000>;
			status = "disabled";
		};
```
**Driver code**
```c
static int rockchip_thermal_probe(structplatform_device *pdev)
{
...
thermal->clk= devm_clk_get(&pdev->dev, "tsadc");
		if (IS_ERR(thermal->clk)) {
			error =PTR_ERR(thermal->clk);
			dev_err(&pdev->dev,"failed to get tsadc clock: %d\n", error);
			return error;
	}
	thermal->pclk =devm_clk_get(&pdev->dev, "apb_pclk");
	if (IS_ERR(thermal->pclk)) {
			error =PTR_ERR(thermal->pclk);
			dev_err(&pdev->dev,"failed to get apb_pclk clock: %d\n",
				error);
			return error;
	}
	error =clk_prepare_enable(thermal->clk);
	if (error) {
			dev_err(&pdev->dev,"failed to enable converter clock: %d\n",
			error);
			return error;
	}
	error =clk_prepare_enable(thermal->pclk);
	if (error) {
			dev_err(&pdev->dev,"failed to enable pclk: %d\n", error);
			goto err_disable_clk;
		}
}
static introckchip_thermal_remove(struct platform_device *pdev)
{
clk_disable(thermal->pclk);
	clk_disable(thermal->clk);
}
```
## 2.5  CLOCK调试

**CLOCK DEBUGS:**
打印当前时钟树结构：
```c
catd/clk/clk_summary 
```
CLOCK设置节点：

**配置选项**：

勾选RK_PM_TESTS

![clk-025-kernel4.4](Rockchip-Developer-Guide-linux4.4-Clock/clk-025-kernel4.4.png)

节点命令：

get rate:

```echo get [clk_name] >/sys/pm_tests/clk_rate```

set rate:

```echo set [clk_name] [rate(Hz)] >/sys/pm_tests/clk_rate```

```echo rawset [clk_name] [rate(Hz)] >/sys/pm_tests/clk_rate```

open rate:

```echo open [clk_name] >/sys/pm_tests/clk_rate```

close rate:

```echo close [clk_name] >/sys/pm_tests/clk_rate```

**TEST_CLK_OUT**测试：

部分时钟是可以输出到test_clk_out，直接测试clk输出频率，用于确认某些时钟波形是否正常。配置方法(以RK3399为例)：

![clk-027-kernel4.4]((Rockchip-Developer-Guide-linux4.4-Clock/clk-027-kernel4.4.png)

（1） 设置CLK的MUX

CRU_MISC_CON 

Address:Operational Base + offset (0x050c)

![clk-029-kernel4.4]((Rockchip-Developer-Guide-linux4.4-Clock/clk-029-kernel4.4.png)

（2）设置CLK的DIV

CRU_CLKSEL58_CON

Address:Operational Base + offset (0x01e8) 

![clk-031-kernel4.4]((Rockchip-Developer-Guide-linux4.4-Clock/clk-031-kernel4.4.png)

（3） 设置CLK的GATING

CRU_CLKGATE13_CON

Address:Operational Base + offset (0x0334)
