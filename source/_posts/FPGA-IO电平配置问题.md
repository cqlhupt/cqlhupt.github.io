---
title: FPGA IO电平配置问题
date: 2024-12-22 13:31:57
tags:
---
在芯片开发过程中，验证是很重要的一环，目前我接触到的验证方式主要有三种：功能仿真（Software Simulation），原型验证（Prototype Verification）以及硬件仿真（Hardware Emulation）。根据我的理解，功能仿真是直接通过软件直接根据RTL行为进行仿真的，常用的仿真软件有VCS，irun，iverilog以及Modelsim（Questasim），功能仿真通常由DV团队进行；原型验证则主要通过FPGA平台，构建一个时钟频率被降低的实际系统，同时使用FPGA进行一些资源的替代（如ASIC上的IO，传感器，其他FPGA上没有的模拟器件），让软件团队能够进行早期验证以及提前开发；硬件仿真则需要使用专门的硬件，通常需要专门向EDA厂商采购相应的平台，如Cadence的Palladium和Synopsys的ZeBu Server，并配合相关的仿真套件（软件和硬件），特殊的语法要求进行更为底层的仿真（类似一种加速的后仿）。本文主要是记录我们在原型验证时遇到的一些问题以及解决方案。
<!-- more -->

# DDR接口的DQS和DQ相位问题

如果我们在FPGA平台上直接开发应用，而不是进行原型验证的话，那么我们通常可以使用FPGA厂商提供的免费IP，这些IP通常使用和集成都很简单，也能达到很高的工作频率，能够大大降低开发的时间。如果我们希望在FPGA上做DDR相关的原型验证，这时就不适合直接使用厂商的IP了，因为我们做的设计正是原型验证的关键，如果将它用厂商的IP替换掉，那么我们就不能达到原型验证的目的了。

## 控制器输出DQS
在DDR应用中，通常要求控制器端输出的DQS信号和DQ信号保持90度的相位

![DDR SDRAM写数据](/img/fpga_prototype/ddr_sdram_write_data.png "DDR SDRAM写数据")

![NAND Flash写数据](/img/fpga_prototype/NVDDR234_write_data.png "NAND Flash写数据")

在电路设计中，通常使用同步时钟进行设计，理想情况下数据的变化边沿和时钟的变化边沿往往是对齐的，实际情况下由于组合逻辑的存在，会导致数据的变化滞后于时钟变化

我们通常使用两组逻辑产生DQ和DQS：
* 使用时钟作为数据选择信号，将两倍接口位宽数据转为每半个周期输出一个的DDR数据
* DQS则由clock gate产生

![简易DQ和DQS产生电路结构](/img/fpga_prototype/simple_dq_dqs_generate.png "简易DQ和DQS产生电路结构")

如果直接使用这个电路产生DQS和DQ，显然DQS变换应该是要先于DQ的变化的，所以我们需要想办法让DQS延迟到DQ变化之后去

### 方案一

将clock gate的输入时钟换成clk延迟90度的时钟，clock gate的使能信号仍由clk时钟域的信号控制

![使用90度时钟产生DQS](/img/fpga_prototype/dqs_solution1.png "使用90度时钟产生DQS")

这个方案主要的问题是：
* 需要单独产生一个90度的时钟，占用更多的时钟资源
* DQS从clock gate出来之后到IO处的delay没有考虑，最终的DQ和DQS相位关系仍存在一定的随机性

### 方案二

直接在clock gate出口到IO之间使用组合逻辑绕线

![使用组合逻辑绕线产生DQS](/img/fpga_prototype/dqs_solution2.png "使用组合逻辑绕线产生DQS")

这个方案主要的问题是：
* 可能需要多次实验才能确定反相器的级数
* 反相器数量固定之后延迟就固定了，不能根据不同的时钟进行变化，如果需要多个工作时钟，则只能按照最快的时钟计算级数
* 仿真时需要额外增加延迟，因为仿真时不计任何器件延迟
* 目前仅在Xilinx平台尝试，其他FPGA平台未尝试

实现代码：
```verilog
localparam DELAY = 41; // delay stage
(* dont_touch="true" *) wire dqs_i_delay [DELAY-1:0] /* synthesis syn_keep = 1 */;

assign #4 dqs_i_delay[0] = dqs_i ; // delay for simlation

genvar i;
generate
    for （i = 1; i < DELAY; i=i+1） begin
        assign dqs_i_delay[i] = ~dqs_i_delay[i-1];
    end
endgenerate

wire dqs_i_p_delay = dqs_i_delay[DELAY-1];

(* dont_touch="true" *) BUFG BUFG_DQS(
    .O(dqs_i_delayed),// 1-bit output:Clock output
    .I(dqs_i_p_delay) // 1-bit input:Clock input
);
```

目前我们在Xilinx XCVU13P平台上40级反相器能获得大概10ns的延迟，我们的工作时钟是25MHz，周期40ns，延迟10ns正好是90度

## 控制器输入DQS

在DDR应用中，DDR器件回复给控制器的DQS通常和DQ信号是边沿对齐的

![DDR SDRAM读数据](/img/fpga_prototype/ddr_sdram_read_data.png "DDR SDRAM读数据")

![NAND Flash读数据](/img/fpga_prototype/NVDDR234_read_data.png "NAND Flash读数据")

DQS作为随路时钟，是控制器采样数据的关键，通常我们要将DQS延迟90度之后，让DQS边沿处于DQ稳定的中心来对DQ进行采样

![简单的DDR读采样电路](/img/fpga_prototype/simple_ddr_read_sample.png "简单的DDR读采样电路")

### 方案一

使用FPGA提供的IDELAY器件（Xilinx），对DQS进行延迟

![使用IDELAY延迟DQS](/img/fpga_prototype/read_smp_solution1.png "使用IDELAY延迟DQS")

这个方案的主要问题在于：
* 不清楚非Xilinx器件是否存在类似的IDELAY器件
* 需要为IDELAY器件提供一个参考时钟，XCVU13P器件中为300MHz
* 单级IDELAY的延迟并不高，只有几纳秒，可能需要级联
* IDELAY和ODELAY好像需要同时使用，否则vivado实现会报错
* 可能会存在约束问题，导致最终实现出来的DQS延迟没有被约束

### 方案二

使用两倍时钟采样DQS之后再将采样输出作为采样时钟去采样DQ

![使用2倍时钟采样延迟DQS](/img/fpga_prototype/read_smp_solution2.png "使用2倍时钟采样延迟DQS")

这个方案的问题在于：
* 需要多一个频率2倍于工作时钟的时钟
* 由于DQS和2倍时钟的关系是异步的，所以可能会采样到亚稳态

实际上我们使用2倍时钟的目的仍是将DQS延迟90度，理论上DQS如果和内部的2倍时钟是上升沿对齐的，那么确实可以实现延迟90度，我们也可以使用更高的倍频，比如4倍时钟来采样DQS，将DQS延迟45度

为了解决可能采样到亚稳态的问题，可以将2倍采样时钟的相位做成可调的，每次出FPGA bit前调整好该时钟的相位，当然如果需要实现多个控制器或者一个通道有多个DQS时，可能需要单独进行调整

### 方案三

同控制器输出方案二，在DQS输入到采样寄存器之间使用组合逻辑绕线，具体请参考前面的描述

![使用组合逻辑绕线延迟DQS](/img/fpga_prototype/read_smp_solution3.png "使用组合逻辑绕线延迟DQS")

# IO的电平兼容性

集成电路发展趋势是制程降低，速度提高，这二者一起推动了IO电平的发展，协议的发展往往也伴随着电平标准的改变，目前常用的电平标准有：TTL、CMOS、LVTTL、LVCMOS、ECL、PECL、LVPECL、RS232、RS485、LVDS、GTL、PGTL、CML、HSTL、SSTL等

在使用实际使用过程中，发送端和接收端尽量使用相同的电平标准，也就是说请及时确认你操作的器件使用的电平标准，确保设计时使用相同的电平标准

有时我们也会遇到电平标准不匹配的情况，这个时候需要确认两个电平能否兼容：
* 确认设计输出的VOH和VOL能否满足设备的VIH和VIL
* 确认设备输出的VOH和VOL能否满足设计的VIH和VIL

DDR的电平目前是DQS/DQ等DQ相关的信号使用SSTL，其他信号则使用LVCMOS

## 遇到的问题

我们在约束接口电平标准时一开始就直接按照电平标准要求进行了约束
```tcl
set_property -dict {PACKAGE_PIN AW34 IOSTANDARD LVCMOS12} [get_ports ale]
set_property -dict {PACKAGE_PIN AW36 IOSTANDARD LVCMOS12} [get_ports cle]
set_property -dict {PACKAGE_PIN BB34 IOSTANDARD LVCMOS12} [get_ports we_n]
set_property -dict {PACKAGE_PIN AN34 IOSTANDARD LVCMOS12} [get_ports wp_n]

set_property -dict {PACKAGE_PIN AL34 IOSTANDARD LVCMOS12} [get_ports {ce_n[0]}]
set_property -dict {PACKAGE_PIN AM34 IOSTANDARD LVCMOS12} [get_ports {ce_n[1]}]
set_property -dict {PACKAGE_PIN AT33 IOSTANDARD LVCMOS12} [get_ports {ce_n[2]}]
set_property -dict {PACKAGE_PIN AP34 IOSTANDARD LVCMOS12} [get_ports {ce_n[3]}]

set_property -dict {PACKAGE_PIN AL32 IOSTANDARD SSTL12} [get_ports {dq[0]}]
set_property -dict {PACKAGE_PIN AN33 IOSTANDARD SSTL12} [get_ports {dq[1]}]
set_property -dict {PACKAGE_PIN AL33 IOSTANDARD SSTL12} [get_ports {dq[2]}]
set_property -dict {PACKAGE_PIN AM32 IOSTANDARD SSTL12} [get_ports {dq[3]}]
set_property -dict {PACKAGE_PIN AR33 IOSTANDARD SSTL12} [get_ports {dq[4]}]
set_property -dict {PACKAGE_PIN AN32 IOSTANDARD SSTL12} [get_ports {dq[5]}]
set_property -dict {PACKAGE_PIN AP33 IOSTANDARD SSTL12} [get_ports {dq[6]}]
set_property -dict {PACKAGE_PIN AW33 IOSTANDARD SSTL12} [get_ports {dq[7]}]

# single-end RE_n and DQS
set_property -dict {PACKAGE_PIN AY35 IOSTANDARD SSTL12} [get_ports re_n_p]
set_property -dict {PACKAGE_PIN AY36 IOSTANDARD SSTL12} [get_ports re_n_n]
set_property -dict {PACKAGE_PIN AY33 IOSTANDARD SSTL12} [get_ports dqs_p]
set_property -dict {PACKAGE_PIN BA33 IOSTANDARD SSTL12} [get_ports dqs_n]

# differential RE_n and DQS
set_property -dict {PACKAGE_PIN AY35 IOSTANDARD DIFFSSTL12} [get_ports re_n_p]
set_property -dict {PACKAGE_PIN AY36 IOSTANDARD DIFFSSTL12} [get_ports re_n_n]
set_property -dict {PACKAGE_PIN AY33 IOSTANDARD DIFFSSTL12} [get_ports dqs_p]
set_property -dict {PACKAGE_PIN BA33 IOSTANDARD DIFFSSTL12} [get_ports dqs_n]
```

我们发现在使用差分模式的时候，可以正常操作NAND，但是如果我们的FPGA是单端的约束则会在读数据时遇到一些莫名的数据错误，错误的模式是：DQ的某些位会在某些时候持续采样到同样的值，并且好像和DQS的下降沿有相关性

我们尝试更换数据pattern，能够一定程度上缓解这种情况，这个应该是因为不同的pattern其直流均衡不同

我们尝试使用逻辑分析仪抓取接口上的数据，然后发现逻辑分析仪接上之后读回来的数据就完全正常了，一点问题都没有

这个现象非常神奇，但是我们必须找到问题在哪里，不然FPGA bit的schedule就会被耽误，甚至影响到整个项目交付

经过漫长的排查，发现如果我们将DQ约束为LVCMOS的话，能够解决这个问题
```tcl
set_property -dict {PACKAGE_PIN AL32 IOSTANDARD LVCMOS12} [get_ports {dq[0]}]
set_property -dict {PACKAGE_PIN AN33 IOSTANDARD LVCMOS12} [get_ports {dq[1]}]
set_property -dict {PACKAGE_PIN AL33 IOSTANDARD LVCMOS12} [get_ports {dq[2]}]
set_property -dict {PACKAGE_PIN AM32 IOSTANDARD LVCMOS12} [get_ports {dq[3]}]
set_property -dict {PACKAGE_PIN AR33 IOSTANDARD LVCMOS12} [get_ports {dq[4]}]
set_property -dict {PACKAGE_PIN AN32 IOSTANDARD LVCMOS12} [get_ports {dq[5]}]
set_property -dict {PACKAGE_PIN AP33 IOSTANDARD LVCMOS12} [get_ports {dq[6]}]
set_property -dict {PACKAGE_PIN AW33 IOSTANDARD LVCMOS12} [get_ports {dq[7]}]
```

但是我们仍然不清楚原因是什么，然后我们开始研究SSTL电平，发现SSTL和LVCMOS有一个重要的区别，SSTL是需要一个参考电压的，而LVCMOS不需要，这是由于SSTL的运行速度比较高，使用VREF进行电压的判决比较有利于高速应用，然后发现我们之前其实没有约束这个VREF，Xilinx的FPGA在没有约束VREF时默认使用的是一个外部引脚输入的VREF，然后我们向FPGA开发板的设计同事确认了一下这个VREF的值，发现硬件上没有供这个VREF

一切终于明朗了，由于VREF电压没供，导致SSTL参考的是一个浮空的电平，我们用来采样的DQS可能出现了波动，最终导致数据采样错误，那为什么差分的时候又是好的呢，其实是因为在差分的时候DQS的两根线互为参考，不需要VREF来判决了，所以能够采样到正确的DQ

当我们将逻辑分析仪接上时增加了接口的负载，也缓解了VREF浮空的影响，所以能够采样到正确的值

## 解决方案

增加对应IO bank的VREF约束，让IO使用内部的参考电压
```tcl
set_property INTERNAL_VREF 0.60 [get_iobanks 61]
set_property INTERNAL_VREF 0.60 [get_iobanks 62]
set_property INTERNAL_VREF 0.60 [get_iobanks 63]
set_property INTERNAL_VREF 0.60 [get_iobanks 64]
```

# 结语

问题的积累也是经验的积累，只有在遇到足够多的问题之后才能从容地应对，如果后续还有其他的问题，我也会持续更新，也欢迎通过评论区与我交流讨论