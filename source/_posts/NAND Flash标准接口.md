---
title: NAND Flash标准接口
top: true
date: 2023-08-12 20:52:25
tags:
---
毕业也已经一年了，2021年11月拿到了现公司的offer，22年4月份应公司邀请提前到公司实习（主要是当时公司人太少了，我来了之后芯片设计一共5人，验证1人，项目开始后迟迟招不到人）。5月回校答辩，6月正式成为社畜打工人。由于工作安排，接受offer之后一直在熟悉NAND flash相关的接口，到目前为止，MPW芯片即将回片，虽然已经预见存在很多问题，不过已经完成了fullmask阶段的修改。NAND flash的接口与DDR SDRAM接口存在一定的相似性，在此记录以便巩固相关知识，也为后续工作和学习打下基础。
<!--more-->

# 概述
* JEDEC协会统一管理，分化出ONFi（开放）和Toggle（封闭）
* 具有和DDR SDRAM类似的接口
* 地址和数据复用同一组数据线（还未发布的JESD230G版本中可能会将二者分开）
* 

# 信号定义
| 名称    | 来源         | 描述                                                                                          |
| ------- | ------------ | --------------------------------------------------------------------------------------------- |
| CE_n     | 主控       | 片选信号，低电平有效，用于使能颗粒                                                                 |
| ALE     | 主控       | 地址锁存使能信号，高电平有效                                                |
| CLE   | 主控        | 命令锁存使能信号，高电平有效                                         |
| WE_n/CLK   | 主控        | SDR模式中写数据、命令和地址的边沿信号<br>NVDDR模式中向颗粒输出持续时钟<br>NVDDR2/NVDDR3/NVLPDDR4/Toggle模式中写命令和地址的边沿信号 |
| RE_n/RE_n_c | 主控        |    SDR模式中读数据边沿信号，用于采样<br>NVDDR模式中指示读写高电平表示写，低电平表示读<br>NVDDR2/NVDDR3/NVLPDDR4/Toggle模式中主控发送给颗粒的读数据的边沿信号<br>**互补信号RE_n_c仅在高速DDR时才开启** |
| DQS/DQS_c  | 主控/颗粒   | 用于NVDDR/NVDDR2/NVDDR3/NVLPDDR4/Toggle模式的数据strobe信号<br>写数据时其边沿位于DQ信号窗口的中间，由主控发往颗粒<br>读数据时其边沿与DQ信号变化边沿对齐，由颗粒接收RE_n信号后返回给主控<br>**互补信号DQS_c仅在高速DDR时才开启**  |
| DQ[7:0]  | 主控/颗粒        | 数据信号，有8位和16位两种，16位的较为少见，且16位数据线在发送地址和命令时只使用低8位，只有在传输数据时才会使用全部16位数据线        |
| WP_n  | 主控         | 写保护，低电平有效<br>可复用为ODT控制线，用于主控控制颗粒的ODT使能时间而不是颗粒自动ODT             |
| Rb_n  | 颗粒         | ready/busy信号，高电平表示ready，低电平表示busy     |

\* 此处的信号均为数字信号，如需了解电源相关模拟信号，请参考具体颗粒手册或ONFi协议

# 时序模式
## 一、SDR/Legacy
SDR即Single Data Rate的缩写，在Toggle组织中也有将该模式称为Legacy的。在2006年的NAND市场混乱不堪，各家的接口标准和时序也不一致，当时的颗粒速率比较低，于是ONFi协会成立并发布的第一版ONFi协议。第一版协议中描述了该种工作模式，此工作模式下最高支持50MB/s。
### 命令锁存
![SDR命令锁存时序](../../../hexo/themes/icarus/source/img/onfi/SDR_cmd_latch.png "SDR命令锁存时序")
```
1. CE_n信号拉低，对要操作的颗粒进行片选
2. CLE信号拉高，ALE信号保持低
3. 主控端将命令放在DQ[7:0]上
4. 主控控制WE_n产生一个边沿
```
从时序图中可以看到，时序参数均以WE_n的上升沿为参考，上升沿前的时间为建立时间（setup），上升沿后的时间为保持时间（hold）。只有WE_n拉低的时间为tWP，此参数取决于当前SDR工作的速率，其值为当前工作速率下的半个时钟周期，如工作于25MHz，则tWP应为20ns，数据速率则为25MB/s。

### 地址锁存
![SDR地址锁存时序](../../../hexo/themes/icarus/source/img/onfi/SDR_addr_latch.png "SDR地址锁存时序")
```
1. CE_n信号拉低，对要操作的颗粒进行片选
2. ALE信号拉高，CLE信号保持低
3. 主控端将地址放在DQ[7:0]上
4. 主控控制WE_n产生一个边沿
```
可以发现地址锁存时序与命令锁存时序非常相似，此处多出来的tWH参数其实和tWP参数一样，而tWC=tWP+TWH。tWH参数的引入主要是因为每次发送地址时总是含有多个Byte（目前主流的是5~6Byte，特殊操作会有10Byte），tWH描述的是两个Byte地址之间的间隔时间。

### 写数据
![SDR写数据时序](../../../hexo/themes/icarus/source/img/onfi/SDR_write_data.png "SDR写数据时序")
```
1. CE_n信号拉低，对要操作的颗粒进行片选
2. ALE信号和CLE信号保持低
3. 主控端将数据连续放在DQ[7:0]上
4. 主控控制WE_n连续产生与每个数据对应的上升沿
```
SDR写数据时，每个上升沿都是与数据相对应的，所以上升沿不能乱给，不然会导致数据顺序出错，在数据读出的时候会出现解码引擎无法完成解码的情况。时序参数增加了tCLH和tCH，规定的是最后一个数据写完之后CE_n和CLE信号需要保持的时间。
### 读数据
* 非EDO模式读
  ![SDR非EDO读数据时序](../../../hexo/themes/icarus/source/img/onfi/SDR_read_data_nonedo.png "SDR非EDO读数据时序")
  ```
  1. CE_n信号拉低，对要操作的颗粒进行片选
  2. ALE信号和CLE信号保持低，主控端释放DQ[7:0]数据线（写数据时输出，读数据时输入）
  3. 主控端拉低RE_n信号
  4. RE_n下降沿后经过tREA时间后，颗粒将数据放在DQ[7:0]上
  5. 主控端拉高RE_n信号，并使用上升沿对DQ[7:0]上的数据进行采样
  6. 重复3~5步，完成所有需要的数据采样
  ```
  读数据时参考的时钟边沿是RE_n的上升沿，RE_n的周期（tRC）是工作时钟的周期，tRC=tRP+tREH。另一个重要的参数是tRR，因为在发完读数据命令之后到开始读数据之前，颗粒会处于Busy的状态（颗粒在此期间将数据从浮栅晶体管阵列中读出到缓存），在颗粒Ready之后到RE_n拉低之前的时间需要大于等于tRR。

* EDO模式读
  ![SDR EDO读数据时序](../../../hexo/themes/icarus/source/img/onfi/SDR_read_data_edo.png "SDR EDO读数据时序")
  ```
  1. CE_n信号拉低，对要操作的颗粒进行片选
  2. ALE信号和CLE信号保持低，主控端释放DQ[7:0]数据线（写数据时输出，读数据时输入）
  3. 主控端拉低RE_n信号
  4. RE_n下降沿后经过tREA时间后，颗粒将数据放在DQ[7:0]上
  5. 主控端拉高RE_n信号，并下一个下降沿对DQ[7:0]上的数据进行采样
  6. 重复3~5步，完成所有需要的数据采样
  ```
  观察时序图可以发现，此时tREA参数似乎长于非EDO的tREA，但是其实tREA参数对应于固定的一颗颗粒是比较一致的，只不过是我们提高了RE_n的速度（tRC变小了），看起来tREA相对变长了。
  这就带来一个问题，颗粒那边的数据输出是由下降沿触发的，采样数据的下降沿比触发数据下降沿晚一个周期，导致最后一个数据没有RE_n的下降沿进行采样。目前我的做法是将RE_n信号整体后延一个周期，再用来采样，也就是说将触发数据的下降沿沿延迟一个周期作为采样数据的下降沿，如果你有更好的方法，烦请与我交流，谢谢。

## 二、~~NVDDDR~~
此模式为源同步模式，其接口与DDR SDRAM最为接近，包含了一个主控输出的同步时钟给颗粒，但是读写数据时仍需使用DQS信号，由于其相对于两外两种模式的复杂性和电气特性要求（主要是时钟信号、DQ信号、DQS信号的PCB等长布线，源同步时钟带来的功耗、高频干扰问题），目前已经没有颗粒厂商再研发相对应的颗粒，其速率也止步于100MHz DDR（200MB/s）。对此模式可以权当了解，无需深入。

### 命令锁存
![NVDDR命令锁存时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR_cmd_latch.png "NVDDR命令锁存时序")
### 地址锁存
![NVDDR地址锁存时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR_addr_latch.png "NVDDR地址锁存时序")
### 写数据
![NVDDR写数据时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR_write_data.png "NVDDR写数据时序")
![NVDDR时钟停止写数据时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR_write_data_clk_stop.png "NVDDR时钟停止写数据时序")
![NVDDR时钟停止和数据暂停写数据时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR_write_data_clk_stop_and_data_pause.png "NVDDR时钟停止和数据暂停写数据时序")

### 读数据
![NVDDR读数据时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR_read_data.png "NVDDR读数据时序")
## 三、NVDDR2/NVDDR3/NVLPDDR4/Toggle

### 命令锁存
![NVDDR234命令锁存时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR234_cmd_latch.png "NVDDR234命令锁存时序")
### 地址锁存
![NVDDR234地址锁存时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR234_addr_latch.png "NVDDR234地址锁存时序")
### 写数据
![NVDDR234写数据时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR234_write_data.png "NVDDR234写数据时序")
### 读数据
![NVDDR234读数据时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR234_read_data.png "NVDDR234读数据时序")
### 读写数据暂停（pause/exit）
![NVDDR234写数据用CE_n暂停时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR234_write_pause_resume.png "NVDDR234写数据用CE_n暂停时序")
![NVDDR234写数据用CLE暂停时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR234_write_pause_resume1.png "NVDDR234写数据用CLE暂停时序")
![NVDDR234读数据用CE_n暂停时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR234_read_pause_resume.png "NVDDR234读数据用CE_n暂停时序")
![NVDDR234读数据用CLE暂停时序](../../../hexo/themes/icarus/source/img/onfi/NVDDR234_read_pause_resume1.png "NVDDR234读数据用CLE暂停时序")
### ODT（On Die Termination）
![NVDDR234 ODT匹配示意](../../../hexo/themes/icarus/source/img/onfi/NVDDR234_ODT.png "NVDDR234 ODT匹配示意")