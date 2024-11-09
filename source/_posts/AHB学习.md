---
title: AHB学习
date: 2024-11-09 18:53:17
tags:
---
AHB是新一代AMBA总线，旨在满足高性能可综合设计的要求。AHB是支持多主机和高带宽操作的高性能总线。学习AHB对我们设计和使用AMBA总线兼容模块具有重要意义。作为AMBA总线的一部分，在芯片中得到了广泛的应用，学习总线，理解总线能让我们在设计模块时游刃有余。
<!--more-->

# AHB学习
AHB为高性能、高时钟频率系统实现了如下特性：
* 突发传输
* 拆分事务
* 单周期总线移交
* 单边沿操作
* 非三态实现
* 数据总线宽度可配置

AHB和ASB/APB的桥接能够非常高效以确保任何现存的设计能够便于集成。

一个基于AHB总线的系统包含一个或多个主机，典型系统中至少包含处理器和测试接口（作为主机）。然而，DMA和DSP也是常见的总线主机。

外部存储器接口、APB桥和任何内部存储器都是常见的AHB从机。任何其他的外设都应当被认为是AHB的从机。但是，低带宽外设通常作为APB的从机。

典型的AHB系统包含如下组件：
**AHB主机**：主机可以通过提供地址和控制信息来发起读写操作。一次只能有一个主机激活并使用总线。
**AHB从机**：一个从机在一个指定的地址空间内响应读写请求。从机返回信号给主机表示成功、失败或者等待数据传输。
**AHB仲裁器**：仲裁器用于确保同一时间只有一个主机使用总线发起数据传输。尽管仲裁协议是固定的，仲裁算法（固定优先级、循环优先级）却可以根据不同的应用场景来实现。一个AHB应只包含一个仲裁器，尽管这在单主机系统中比较琐碎。
**AHB译码器**：用于在数据传输中进行地址译码并向传输中将会用到的从机发送选择信号。在AHB系统中要求只有一个中央译码器。

# AHB总线结构

AHB总线系同一般由几个部分组成：主机，仲裁器，地址译码器，从机

![AHB系统结构图](/img/AHB/ahb_system.png "AHB系统结构图")

![AHB slave接口图](/img/AHB/ahb_slave_interfa.png "AHB slave接口图")

这里的slave接口中只有一个HREADY，实际上我们需要一个HREADY_out和一个HREADY_in，用于避免在总线移交到另一个slave时上一个slave把HREADY_out拉低而延长传输周期

![AHB master接口图](/img/AHB/ahb_master_interfa.png "AHB master接口图")

![AHB arbiter接口图](/img/AHB/ahb_arbiter_interfa.png "AHB arbiter接口图")

![AHB decoder接口图](/img/AHB/ahb_deoder_interfa.png "AHB decoder接口图")

实际上master和slave是比较常用的，另外两个基本上只有做总线互联的时候会用到，一般的公司很少进行总线互联的研发，一般有两种情况会研发总线互连，一是公司被制裁，不能购买成熟的总线互联IP，另一种是公司产品本身就是以总线互联IP

# AHB信号列表
AHB总线信号简述，所有的信号都以H开头以区别系统中其他相似的信号

|name   |source  |description    |
|---|---|---|
|HCLK|时钟源|时钟，上升沿用于AHB的所有传输|
|HRESETn|复位控制器| 复位，唯一低电平有效的信号，用于复位系统和总线|
|HADDR[31:0]|主机|地址，32位系统地址总线|
|HTRANS[1:0]|主机| 指示当前传输类型，可以是NONSEQUENTIAL, SEQUENTIAL, IDLE或BUSY|
|HWIRTE|主机|高表示写传输，低表示读传输|
|HSIZE[2:0]|主机|指示传输数据的宽度，常为字节(8-bit)、半字(16-bit)或字(32-bit)。协议允许最大值为1024bit，但必须小于等于数据总线宽度。|
|HBURST[2:0]|主机| 指示突发传输的形式，自增（increment）和环回（warp）都支持4、8、16拍传输。|
|HPROT[3:0]|主机| 保护控制信号提供关于总线接入的附加信息，并主要用于希望实现某种级别保护的模块。信号指示传输是否是操作码拾取（opcode fetch）或数据访问（data access），以及传输是特权模式（privileged mode）还是用户模式（user mode）。对于带有MMU的总线主机来说还用于指示当前访问的数据是否可以缓存（cacheable）或缓冲（bufferable）|
|HWDATA[31:0]|主机|写数据总线用于写操作时主机向从机传输数据。推荐最小的位宽为32位。但是也很容易为了更高的带宽而进行扩展。|
|HSELx|译码器|每个AHB从机都拥有自己的选择信号，用于指示当前传输的目标是否是本从机。由简单的组合逻辑地址译码得到|
|HRDATA[31:0]|从机|读数据总线用于写操作时从机向主机传输数据。推荐最小的位宽为32位。但是也很容易为了更高的带宽而进行扩展。|
|HREADY|从机|高表示总线上的一次传输结束。为了扩展一次传输，该信号可被拉低。**总线上的从机要求HREADY既有输入端口，又有输出端口。**|
|HRESP[1:0]|从机| 传输响应提供传输状态的额外信息。四种不同的响应为：OKAY，ERROR，RETRY，SPLIT|

AHB中还有一些信号用于支持多主机操作。这些仲裁信号的多数都是点到点的专用信号，表中后缀x表示信号来自模块x，如HBUSREQarm，HBUSREQdma等。

|name   |source  |description    |
|---|--------|---|
|HBUSREQx|主机  |来自于主机x到仲裁器，用于表示主机请求使用总线。每个主机都有一个HBUSREQx，一个系统中最多有16个主机。|
|HLOCKx|主机    |拉高表示主机请求执行总线锁定访问（locked access），而其他主机在该信号拉低之前不能获得总线。|
|HGRANTx|仲裁器 |表示主机x当前拥有最高的总线优先级。地址/控制信号的所有权在一次传输以HREADY为高结束时改变，所以当HREADY（上一次访问结束）和HGRANTx都为高时获得总线。|
|HMASTER[3:0]|仲裁器|这个信号用于表明哪个主机正在进行传输，并被支持SPLIT传输的从机用于判断哪个主机正在尝试访问。HMASTER的时序与地址和控制信号的时序定义在一起。|
|HMASTERLOCK|仲裁器|表示当前主机进行的是锁定序列传输。具有与HMASTER一样的时序|
|HSPLITx[15:0]|从机|用于从机到仲裁器指示哪个主机需要允许事务拆分重传。每个位与一个单一的主机绑定。|

# 传输类型、突发类型和响应类型
## 传输类型（Transfer type）
传输类型是HTRANS[1:0]信号的值编码，具体信息如下

|HTRANS[1:0]    |类型   |描述    |
|----------|------------|------|
|2'b00|IDLE|空闲状态，表示主机获得总线，但不想传输数据，从机在第一个IDLE周期时可以拉低HREADY向主机表示没有准备好接收该IDLE传输，一旦slave接收了一个IDLE，下一个周期无论HTRANS是什么，slave都不能拉低HREADY，必须立即接收，也就是slave当前周期接收了IDLE则下一个周期HREADY必须拉高，并且HRESP必须为OKAY|
|2'b01|BUSY|主机忙，表示主机没有准备好在下一个周期进行数据传输，slave可以在第一个BUSY周期时拉低HREADY，表示没有准备好接收该BUSY传输，一旦slave接收了一个BUSY，下一个周期无论HTRANS是什么，slave都不能拉低HREADY，必须立即接收，也就是slave当前周期接收了BUSY则下一个周期HREADY必须拉高，并且HRESP必须为OKAY|
|2'b10|NONSEQ|非连续传输，表示开始一个新传输，理论上认为该传输与前一个周期的传输无关，表示下一个周期进行一次新的传输，根据HBURST的不同而后续跟着不同数量的SEQ传输，如果HBURST是SINGLE，则没有后续的SEQ|
|2'b11|SEQ|连续传输，表示与前面的传输地址相关的连续的传输，具体的地址计算需要根据HBURST进行计算，当然slave也可以不计算，因为master每个周期都会给出地址|

## 突发类型（Burst Type）
传输类型是HBURST[2:0]信号的值编码，具体信息如下

|HBURST[2:0]    |类型   |描述   |
|----------|-----------|------------|
|3'b000|SINGLE|单次传输，NONSEQ后续没有SEQ，SINGLE的NONSEQ之后下一个周期不能是BUSY|
|3'b001|INCR|不指定长度的地址递增传输，传输数据的数量没有明确限制，但不能超过1KB边界|
|3'b010|WRAP4|4拍地址环回传输|
|3'b011|INCR4|4拍地址递增传输|
|3'b100|WRAP8|8拍地址环回传输|
|3'b101|INCR8|8拍地址递增传输|
|3'b110|WRAP16|16拍地址环回传输|
|3'b111|INCR16|16拍地址递增传输|

## slave响应类型（Response Type）
slave在传输过程中需要向master传输一些响应信息，表示当前数据的结果和slave的状态

|HRESP[1:0]|响应|描述|
|------|-------|------|
|2'b00|OKAY|在HREADY为1时，表示数据传输成功完成，也用于非另外三种响应的HREADY为0时|
|2'b01|ERROR|用于表示当前有错误产生，slave本身和传输之一出现了问题，用于告诉master有错误出现，并且传输失败了；一次ERROR必须通过两个周期传输，第一个周期HREADY为0（为了让master做好准备），第二个周期HREADY为1|
|2'b10|RETRY|slave告诉master当前传输未完成，master应该重试，直到传输成功；一次RETRY必须通过两个周期传输，第一个周期HREADY为0（为了让master做好准备），第二个周期HREADY为1|
|2'b11|SPLIT|slave告诉master当前传输较长一段时间不能完成，master需要结束当前传输，等待下一次拿到总线再重试，slave在能够完成数据传输时会用HSPLITx信号通知仲裁器，让仲裁器将总线交给对应的master，当然如果一个更高优先级的传输正在进行时总线不会立即移交；一次SPLIT必须通过两个周期传输，第一个周期HREADY为0（为了让master做好准备），第二个周期HREADY为1|

RETRY和SPLIT两种模式其实用的很少，因为这会增加slave的复杂程度，也会增加整个访问流程的复杂度。

## HSIZE和地址计算

HSIZE信号决定了一次突发传输过程中的数据位宽，HSIZE可以指定小于等于数据总线位宽的数据宽度

|HSIZE[2:0]|大小|描述|
|---|---|---|
|3'b000|8 bits|1 Byte|
|3'b001|16 bits|1 Halfword|
|3'b010|32 bits|1 Word|
|3'b011|64 bits|-|
|3'b100|128 bits|4 word line|
|3'b101|256 bits|8 word line|
|3'b110|512 bits|-|
|3'b111|1024 bits|-|

如果数据总线位宽小于等于64位，HSIZE最高位可以不使用，HSIZE指定的位宽也是AHB可以支持的数据总线位宽。

如果HSIZE小于数据总线位宽时要区分大小端，我们来看一下，以下是AHB总线spec的例子，数据总线为32 bits

![小端非完整总线位宽传输](/img/AHB/HSIZE_less_than_bus_width.png "小端非完整总线位宽传输")

小端传输时如果HSIZE小于总线位宽，低位字节先传输，地址的增加按照HSIZE大小增加，低位字节的地址小于高位字节

![大端非完整总线位宽传输](/img/AHB/HSIZE_less_than_bus_width_bigendian.png "大端非完整总线位宽传输")

大端传输时如果HSIZE小于总线位宽，高位字节先传输，地址的增加按照HSIZE大小增加，高位字节的地址小于低位字节

WARP传输的地址计算也是一个重要的环节，如果你的设计需要支持环回访问，需要计算出正确的地址：

根据spec的说法，wrap的环回地址边界是HSIZE×HBURST，比如：

1. HBURST = WRAP4, HSIZE = Halfword, WRAP address = 4×2 = 8
2. HBURST = WRAP8, HSIZE = Word, WRAP address = 8 × 3 = 32

## 传输保护

HPROT信号为传输提供了一些保护控制，包括cacheable, bufferable, privileged, data/opcode
|HPROT[3]|HPROT[2]|HPROT[1]|HPROT[0]|描述|
|---|---|---|---|---|
|-|-|-|0|操作码拾取，对CPU而言|
|-|-|-|1|数据访问，对CPU及所有其他传输|
|-|-|0|-|用户访问|
|-|-|1|-|优先访问|
|-|0|-|-|Not bufferable，表示不可以在bus interconnect部分中被buffer|
|-|1|-|-|Bufferable，表示可以在bus interconnect部分中被buffer|
|0|-|-|-|Not cacheable，表示不可以在接收模块的cache中存储|
|1|-|-|-|Cacheable，表示可以在接收模块的cache中存储|

# 基本的传输波形

## INCR传输
![INCR4传输](/img/AHB/inr4_burst.png "INCR4传输")

INCR传输的地址都是按照HSIZE的大小递增的，具体的传输字节顺序可以参考前述的HSIZE章节。传输由1个NONSEQ和0个或多个SEQ传输组成，传输以下一个burst的NONSEQ或者IDLE结束，传输可以提前结束。

## WRAP传输
![WRAP4传输](/img/AHB/wrap4_burst.png "WRAP4传输")

WRAP传输的地址都是按照HSIZE的大小递增的，一直到环回地址边界时回到上一个边界处，具体的传输字节顺序可以参考前述的HSIZE章节。传输由1个NONSEQ和0个或多个SEQ传输组成，传输以下一个burst的NONSEQ或者IDLE结束，传输可以提前结束。