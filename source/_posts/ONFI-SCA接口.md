---
title: NAND SCA接口（一）
date: 2024-11-23 22:00:35
tags:
---

随着NAND接口速率的提升，命令和地址的传输时间却没有明显减少，这就导致命令和地址的发送时间占据更大的比例，另外一点是数据速率的提升越来越困难了，由于NAND接口是大量借鉴DDR SDRAM接口的，所以在DDR SDRAM引入SCA之后，NAND接口也开始引入SCA机制。目前JEDEC正式发布了JESD230G，其中介绍了SCA接口的最新信息，目前了解到YMTC已经在最新的产品中加入了SCA接口，Hynix也规划了支持的路线图，但是其实ONFI和Toggle规范中都还未添加SCA相关章节，由于SCA的内容比较多，所以分成多篇博客进行介绍
<!-- more -->

# 接口差异

![传统接口和SCA接口差异汇总](/img/sca_if/sca_vs_conv_summary.png "传统接口和SCA接口差异汇总")

* 第一行描述的是引脚差异，SCA模式下增加了SCA引脚，CE#信号变成了CA_CE#，CLE和ALE合并成CA[1:0]，WE#变成了CA_CLK，并且CA[1:0]变成了双向的
* CA[1:0]的数据传输是DDR的，并且符号的传输速率增加了，不过由于CA总线的位宽比较窄，所以总的传输时间是要比传统接口要长的
* 使用数据头来区分CA总线的数据类型
* 由于CA总线可以读数据，所以增加了读数据的时序要求
* 传统使用CE#进行选择的机制被command packet替代
* SCA接口使能有两种方式：
  * 上电时拉高SCA引脚：上电时将SCA置为高电平表示上电后直接使用SCA模式，如果SCA引脚悬空或者为低电平则使用传统模式
  * 通过set feature打开SCA模式：在传统模式中set feature
* 读写操作需要通过新的数据命令序列进行操作，ODT开关也通过command packet

![单die差异示意](/img/sca_if/sca_if_define_vs_conv_if_define.png "单die差异示意")

![ODP封装差异示意](/img/sca_if/sca_if_define_vs_conv_if_define_package.png "ODP封装差异示意")

可以发现，除了SCA信号之外，封装时的连接与传统接口的封装并无差异，SCA信号单独引出到封装上即可

# SCA模式上电要求

![SCA上电信号时序](/img/sca_if/sca_power_up.png)

使用SCA引脚指定模式时，上电过程中需要注意信号的变化：
* SCA，VCCQ和VCC没有顺序要求
* CA_CLK的默认值为0，可以在上电时一直保持为0，也可以在上电时拉高，在使用前再拉低，这里主要是因为CA_CLK是原来的WE#信号，而WE#的默认值是1
* 上电完成后需要满足tV2CE（VCC到达可用范围之后）才能拉低CA_CE#，并且需要保证在CA_CE#拉低前tWLCEL_CA前把tCACLK拉低

# SCA packet

## SCA packet head定义

![SCA包头定义](/img/sca_if/sca_header_definition.png "SCA包头定义")

包头用于指定当前在CA bus上传输的数据类型和数据传输方向，目前包头的位宽只有4位，只剩下三个编码没有使用，不排除以后会增加。head传输也是DDR的，所以一个周期即可传完4位

## SCA packet输入
![SCA包输入](/img/sca_if/sca_input_packet.png "SCA包输入")

* SCA输入包用于传输命令、地址、set feature等从主控到NAND的数据
* 包由head和body组成，head为4位，body为8位，head内容请参考上一节的表格，body的内容依赖于head
* 每个字节输入前都要带head

## SCA packet输出
### 单字节输出
![单字节SCA包输出](/img/sca_if/sca_single_byte_output.png "单字节SCA包输出")

* 输出包开始于输出head，它可以让NAND进入CA输出模式
* 主控输出head之后，经过tCLKCAZ之后释放CA[1:0]
* NAND在tCLKCAD之后接管CA[1:0]，并拉低为00b
* 在tW2R_CA之后主控开始驱动CA_CLK信号
* NAND接收到CA_CLK之后通过CA[0]返回，数据通过CA[1]返回
* 退出读数据模式时主控需要拉高CA_CE#，NAND经过tCECAZ之后释放CA[1:0]
* 在tCECAZ之后主控可以再次控制CA[1:0]，并发送下一个head

![单字节SCA包get feature](/img/sca_if/sca_get_feature_single_byte.png "单字节SCA包get feature")

* 循环发送单字节输出时序获取feature的所有字节
* 单字节输出模式NAND必须支持

### 多字节输出（NAND选择是否支持）
![多字节SCA包get feature](/img/sca_if/sca_get_feature_multi_byte.png "多字节SCA包get feature")

多字节输出时可以输出超过4个周期的CA_CLK，按照需要的数据位数进行输出CA_CLK即可，其前序时序和结束时序和单字节输出是一致的

### NAND CA输出时序要求

单字节和多字节输出都要满足下图中的时序参数，CA[1:0]之间的时序

![SCA输出时序要求](/img/sca_if/additional_ca_output_timing.png "SCA输出时序要求 feature")

# DQ bus控制包
## Select Chip Enable (SCE) Packet
![SCE包](/img/sca_if/sce_packet.png "SCE包")

* DIR位：1表示NAND接收数据（写），0表示NAND发送数据（读）
* 所有保留位必须为0
* 如果开启了warmup cycle，则在所有的SCE包（首次SCE和重启SCE）之后都要重新发送warmup cycle

## Select Chip Pause (SCP) Packet
![SCP包](/img/sca_if/scp_packet.png "SCP包")

* 用于暂停一个DQ总线上正在对应CA_CE#和LUN上进行的数据burst
* 主控后续需要使用SCE重启数据传输（不能直接发送SCT）
* 所有保留位必须为0

## Select Chip Terminate (SCT) Packet
![SCT包](/img/sca_if/sct_packet.png "SCT包")

* 用于终止一个DQ总线上正在对应CA_CE#和LUN上进行的数据burst
* 主控后续不能再重启该数据传输
* 要终止一个被SCP暂停的传输必须先发送SCE重启该传输，再发送SCT终止传输
* 所有保留位必须为0

## Data Input Burst Sequence
![写数据burst](/img/sca_if/write_data_burst_seq.png "写数据burst")

![开ODT写数据burst](/img/sca_if/write_data_burst_odt_seq.png "开ODT写数据burst")

以上两图展示了使用SCE开始一个写数据burst，使用SCP/SCT暂停或者结束一个写数据burst的序列和时序参数，第二张图中还描述了被选中的die的ODT开启（self-termination）时间

注意图中SCP/SCT的发送时间，其最后1个CA_CLK下降沿作为tWPST_CA和tWPSTH_CA的参考沿，看起来在数据还未发完时就开始发送SCP/SCT了，但是实际过程中我们可能不能正好做到这一点，只能在数据都发完之后再开始发送SCP/SCT

## Data Output Burst Sequence
![读数据burst](/img/sca_if/read_data_seq.png "读数据burst")

![开ODT读数据burst](/img/sca_if/read_data_odt_seq.png "开ODT读数据burst")

这两张图和写数据burst类似的展示了读数据时的时序参数，第二章中展示了self-termination的时间

SCP/SCT的发送时间也有写数据时一样的问题，只能在数据完全接收完之后再开始发送SCP/SCT

## Data Output Sequence Restrictions
![正确的读数据序列](/img/sca_if/proper_read_data_seq.png "正确的读数据序列")

* 在发送完LUN/plane级读命令并且tR结束之后，如果NAND支持change column enhanced/change column命令则必须在SCE前发送change column enhanced/change column命令
* 注意在发送CE#级读命令时，NAND可能会要求必须使用change column命令而不是change column enhanced命令

![缺少change column的错误读数据序列](/img/sca_if/illegal_data_out_without_change_column.png "缺少change column的错误读数据序列")

![使用00h命令代替change column的错误读数据序列](/img/sca_if/illegal_data_out_00h_instead_change_column.png "使用00h命令代替change column的错误读数据序列")

* 缺少change column enhanced/change column命令直接读数据是错误的
* 使用00h命令代替change column enhanced/change column命令也是错误的

## Non-Target ODT (NTO) Packet
![非目标ODT包](/img/sca_if/nto_packet.png "非目标ODT包")

* EN位：1表示开启non-Target ODT，0表示关闭non-Target ODT
* DIR位：1表示NAND接收数据（写），0表示NAND发送数据（读）
* M[1:0]：NAND选择性支持，如果NAND不支持则将这两位置为0；如果NAND支持：
  * M[1:0]==00b时，表示使用当前CA_CE#下的所有LUN进行non-Target ODT并忽略LUN[3:0]
  * M[1:0]==01b时，根据LUN编码确定当前CA_CE#参与non-Target ODT的LUN
* LUN[3:0]：四位直接编码LUN，0h表示LUN0参与接下来的non-Target ODT，1h表示LUN1参与接下来的non-Target ODT，2h表示LUN2参与接下来的non-Target ODT，以此类推

![所有非目标CA_CE# LUN ODT](/img/sca_if/nto_packet_all_other_lun.png "所有非目标CA_CE# LUN ODT")

* 图中对所有非CA_CE0#的其他CA_CE#的所有LUN进行了non-Target ODT选择
* 使用SCE和SCT包包含数据传输burst，最后使用配置相同的NTO包释放non-Target ODT的LUN
* 禁止对将要发送/接收数据的CA_CE#发送选中所有LUN的NTO包

![单个非目标CA_CE# LUN ODT](/img/sca_if/nto_packet_lun_granularity.png "单个非目标CA_CE# LUN ODT")

* 首先选择了CA_CE1#的所有LUN进行non-Target ODT
* 又选择了CA_CE0#的LUN1参与non-Target ODT
* 使用SCE和SCT包包含数据传输burst
* 解除CA_CE0#的LUN1参与non-Target ODT
* CA_CE1#的所有LUN参与non-Target ODT
* 禁止对数据传输的LUN发送单个LUN的non-Target ODT选择NTO包

# 小结
介绍了部分JESD230G第10章的内容，尽请期待后续相关的内容更新，感谢您的阅读，希望对您有所帮助

下一篇直达链接[ONFI-SCA接口（二）](https://cqlhupt.github.io/2024/11/30/ONFI-SCA%E6%8E%A5%E5%8F%A32/)