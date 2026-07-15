# DRV8701电路设计



本设计主要应用场景为直流有刷电机驱动（所用电机为智能车越野组RS540直流有刷电机）

## 电机参数

|          |               |
| :------: | :-----------: |
| 额定电压 |     9.0V      |
| 空载电流 |   小于1.4A    |
| 堵转电流 |    大于20A    |
| 空载转速 | 20800±3000rpm |

# 芯片电路

## 电流斩波

​		由DRV8701数据手册7.3.3电流调节章节可知，芯片带有电流斩波功能，当电机电流达到斩波电流阈值后，电桥进入制动（低侧慢衰减）模式直到$t_{OFF}$结束。

​		斩波电流阈值计算公式如下：
$$
I_{CHOP}=\frac{V_{REF}-V_{OFF}}{A_V\times R_{SENSE}}
$$
其中，$V_{OFF}=50mV ,A_V=20V/V$

​		通过分压电阻调整$V_{REF}$电压，SN和SP之间的检测电阻可以调成斩波电流阈值

<img src=".\images\斩波参考电压.png" style="zoom:70%;float:center;">

不启动斩波

- 电机启动瞬间，电源电压直接加到电机，产生最大启动转矩
- 没有频繁的限流，MOS损耗小
- 控制简单，只需要PWM调速
- 响应更快，PWM从20%到80%时，电机立即获得更大电压
- 启动电流巨大，可能导致MOS压力大、电源过流、PCB发热
- 堵转时容易烧MOS

启动斩波

- 启动电流可控
- 堵转保护天然存在，保护MOS
- 对电池友好
- 电流控制精确
- 启动转矩下降，重载可能起不来
- 产生额外电流纹波，电流不是平滑的，可能导致电机 啸叫、EMI增加
- 调试更复杂，需要考虑$R_{sense}$、增益AV、$V_{REF}$

​		本设计设置较高的斩波电流阈值12A，既保留了堵转保护，又不会明显损失动力，而且电路简单。

## 运放输出(SO)

​		DRV8701 上的 SO 引脚输出的模拟电压等于 SP 和 SN 引脚上的电压乘以AV。因子 AV 是并联放大器增益，在 DRV8701 中为 20 V/V。 SO仅在向前或向后行驶期间有效。 H 桥电流约等于：
$$
I=\frac{SO-V_{OFF}}{A_V\times R_{SENSE}}
$$
<img src=".\images\Sense Amplifier Output.png" style="zoom:100%;float:center;">

## IDRIVE Pin 

​		IDRIVE用于配置栅极驱动器的源极电流和灌电流能力。相同条件下，驱动电流越大，MOSFET栅极电荷的充放电速度越快，能够更快地跨越米勒平台，从而提高MOSFET的开关速度，降低开关损耗；但过大的驱动电流会增加电压、电流变化率，引起更严重的振铃和电磁干扰。因此需要在开关损耗和EMI性能之间进行权衡。一般预留外部MOSFET栅极串联电阻焊盘，初始可采用0Ω或4.7Ω，后续根据波形测试结果调整。

​		H 桥输出（SHx 引脚）的上升和下降时间可以通过设置 IDRIVE 电阻值或向 IDRIVE 引脚施加电压调整。如果选择较高的 IDRIVE 设置，则 FET 栅极电压斜坡上升得更快。FET 栅极斜坡直接影响 H 桥输出上升和下降时间。

<img src=".\images\IDRIVE.png" style="zoom:100%;float:center;">

​		根据 FET 的栅极电荷选择 IDRIVE。配置该引脚以便 FET 栅极充电完全在 $t_{DRIVE}$ 期间。如果设计选择的 IDRIVE 对于给定 FET 来说太低，则 FET 可能会无法完全开启。对于具有已知栅漏电荷 ($Q_{GD}$) 和期望的上升时间 (RT) 的 FET，请根据以下条件选择 IDRIVE：
$$
IDRIVE>\frac{Q_{GD}}{RT}
$$
​		预留两个电阻焊盘M1、M2便于设置芯片$I_{DRIVE}$，同时在外部MOSFET的栅极串联电阻焊盘R13、R14，方便后期进一步调整驱动电流。图中R17、R18用于保证驱动器未工作时MOSFET可靠关断，同时为栅极电荷提供泄放路径。

<img src=".\images\IDRIVE电路.png" style="zoom:50%;float:center;">

<img src=".\images\栅极限流电阻.png" style="zoom:80%;float:center;">

​		**本设计采用单一电源供电，如若DRV8701的电源与H桥电源不同请参考数据手册P27的7.4.1章节设置IDRIVE**

## Fault

<img src=".\images\Fault Response.png" style="zoom:80%;float:center;">

# 设计流程

## 外部MOS选择

[MOS管选型：参数及其具体意义_mos管参数详解 通俗解释-CSDN博客](https://blog.csdn.net/weixin_45274594/article/details/132907464)

[MOS管选型实战指南-CSDN博客](https://blog.csdn.net/xuzhuangqin/article/details/159736424)

### 耐压选择

MOSFET的漏源耐压应高于系统最大工作电压，并留有一定裕量。

对于3S锂电池供电：
$$
V_{BAT(MAX)}=12.6
$$


考虑电机换向、PCB寄生电感及开关瞬态尖峰，通常选择：
$$
V_{DS}≥2 \times V_{BAT(MAX)}
$$
即：
$$
V_{DS}≥25V
$$


实际设计中优先选择30V或40V MOSFET。

### 电流能力选择

电机堵转电流大于20A：
$$
I_{STALL}>20A
$$
MOSFET连续漏极电流应大于电机堵转电流，并留有足够裕量。

一般建议：
$$
I_D\ge 1.5\times I_{STALL}
$$
即：
$$
I_D\ge 30A
$$
需要注意，数据手册中的漏极电流通常是在理想散热条件下测得，应结合实际PCB散热能力综合考虑。

### 导通电阻选择

MOSFET导通损耗为：导通电阻越小，MOSFET发热越低。

例如：

**10mΩ**                                $P=10^2\times0.01=1W$

**5mΩ**                                 $P=10^2\times0.005=0.5W$

因此应优先选择低导通电阻MOSFET。

### 栅极电荷选择

​		MOSFET栅极电荷越大，DRV8701驱动难度越高。

​		DRV8701能够驱动多大的MOSFET，不是无限制的，而是受到内部电荷泵（Charge Pump）能力和PWM频率的限制。
$$
Q_G<\frac{I_{VCP}}{f_{PWM}}
$$
其中，$f_{PWM}$是输入DRV8701的期望频率和电流斩波频率两者较高的一个；$I_{VCP}$是电荷泵容量，由VM决定（VM>12V时，$I_{VCP}$为12mA）

​		DRV8701内部电流斩波频率最多为：
$$
f_{PWM}<\frac{1}{t_{OFF}+t_{BLANK}}\approx38kHz
$$
​		如果应用需要强制快速衰减（或在驱动和反向驱动之间交替），则最大FET 驱动能力由下式给出：
$$
Q_G<\frac{I_{VCP}}{2\times f_{PWM}}
$$
​		本设计主要工作于 Drive 和 Brake（Slow Decay）模式，仅在需要倒车时切换方向，因此MOS驱动能力按照 $Q_G<\frac{I_{VCP}}{f_{PWM}}$ 进行计算。
$$
Q_{G}<\frac{12mA}{40kHz}=300nC
$$

## PCB布局

<img src=".\images\Layout Recommendation.png" style="zoom:80%;float:center;">

# 问题思考

**1、在IDRIVE配置时，期望上升时间是不是越小越好？**

不行，因为PCB存在寄生电感，当MOS切换太快时$V=L\frac{dI}{dt}$，如果切换太快将产生尖峰，造成振铃或EMI问题。实际工程可以根据$Q_{GD}$选一个初值，上示波器看有没有振铃、MOS是否发烫，如果MOS发热严重，说明开关太慢，可以提高IDRIVE；如果振铃很严重、EMI很大说明开关太快，可以降低IDRIVE。

**2、在一些玩具中的电机中，我们常看到直流有刷电机后面并联了100nF的小电容，这个电容的作用是什么？可不可以不要呢？**

电容作用：[直流有刷电机并联小电容作用分析_直流电机并联电容-CSDN博客](https://blog.csdn.net/s1731987459/article/details/118277544)

如果电机功率小（如微型玩具电机）、转速低，或运行环境干扰少，不加电容也可能正常工作。 
