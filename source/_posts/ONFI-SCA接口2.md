---
title: NAND SCA接口（二）
date: 2024-11-30 11:21:16
tags:
---
接上一篇[NAND SCA接口（一）](https://cqlhupt.github.io/2024/11/23/ONFI-SCA%E6%8E%A5%E5%8F%A3/)的内容，继续讲JESD230G的第十章
<!-- more -->

# LUN选择(LUNSel)包 (NAND可选支持)
![LUN选择包](/img/sca_if/lun_sel_packet.png "LUN选择包")

![LUN选择包的时序](/img/sca_if/tlunsel_require.png "LUN选择包的时序")

* LUN选择包用于获取（设置）后续指令的LUN上下文
* L[3:0]：CA_CE#下的LUN编号
* P[3:0]：LUN中的Plane编号
* 在LUN选择包发送之后要满足tLUNSEL_CA时序才能发送下一个包

![LUN选择包的使用](/img/sca_if/user_lun_sel_packet.png "LUN选择包的使用")

* LUN选择包可用于program命令序列之前（80h和10h），也可用于change read column之前用于选择接下来的LUN上下文
* 厂商可以选择支持部分或全部的LUN选择用法，请参考厂商的数据手册
* 这里可以发现program时发送完地址之后要发送一个地址结束指令（12h），传统的接口中没有这个指令

# 命令指针复位序列

![命令指针复位序列](/img/sca_if/command_ptr_reset.png "命令指针复位序列")

* 在一个包的传输过程中将对应的CA_CE#拉高可以让NAND复位其包接收指针
* 被打断的包将会被抛弃，再次拉低CA_CE#之后主控可以开始发送下一个包

# 命令序列杂项

## Set feature
![Set feature序列](/img/sca_if/set_feature_seq.png "Set feature序列")

* feature的每一个字节都需要重新发送head

## Program
![传统和SCA Program序列对比](/img/sca_if/program_seq.png "传统和SCA Program序列对比")

* SCA协议中发送完program的地址后发送12h命令
* 数据被包含在SCE和SCP/SCT之间
* 厂商可能会要求在80h和10h前发送LUN选择包

## Read status enhanced
![Read status enhanced序列](/img/sca_if/read_status_enhance_seq.png "Read status enhanced序列")

* SCA协议中通过CA[1:0]获取status

## Change Read Column (Enhanced)
![Change Read Column序列开始](/img/sca_if/change_column_start_dataout.png "Change Read Column序列开始")

![Change Read Column序列结束](/img/sca_if/change_column_end_dataout.png "Change Read Column序列结束")

* 使用Change Read Column和SCE序列开始LUN0读数据传输
* LUN0读数据被SCT结束，并用SCE开始LUN1的写数据传输

## Multi-Plane Program
![Multi-Plane Program序列](/img/sca_if/sca_same_lun_multiplane_prgramo_seq.png "Multi-Plane Program序列")

* 地址的数量由NAND厂商指定
* 不同的厂商可能指定不同的program/confirm命令
* tDBSY也是可选的

![LUN interleaving Multi-Plane Program序列](/img/sca_if/sca_diff_lun_multiplane_prgramo_seq.png "LUN interleaving Multi-Plane Program序列")

* 可以用SCP暂停LUNX的数据传输之后进行LUNY的数据传输，LUNY数据传输完毕后再重启LUNX的数据传输

![错误的Multi-Plane Program序列](/img/sca_if/scp_not_support_lun_leave.png "错误的Multi-Plane Program序列")

* 不能在SCP暂停数据传输后直接发送program命令11h/10h

## Multi-Plane Program Sequence with Change Row Address
![带change row地址的Multi-Plane Program序列](/img/sca_if/sca_multi_plane_change_row_program.png "带change row地址的Multi-Plane Program序列")

* 地址的数量由NAND厂商指定
* 不同的厂商可能指定不同的program/confirm命令
* tDBSY也是可选的
* NAND厂商可能会要求在Change Row Address前发送LUN选择包

## Multi-Plane Cache Program
![Multi-Plane Cache Program序列](/img/sca_if/sca_multi_plane_cache_program.png "Multi-Plane Cache Program序列")

* 地址的数量由NAND厂商指定
* 不同的厂商可能指定不同的program/cache confirm命令
* tDBSY也是可选的

# 命令交织
## DQ and Non-DQ Related Commands
![SCA命令的DQ相关性](/img/sca_if/relationship_of_sca_dq_bus.png "SCA命令的DQ相关性")

* SCA命令可以分为与DQ相关和与DQ无关两类
* DQ相关的命令不允许发送给一个在执行前一条DQ相关命令（或者正在进行DQ传输）的die
* DQ无关的命令可以发送给一个在执行前一条DQ相关命令（或者正在进行DQ传输）的die，具体限制取决于厂商

## Command Interleaving例子
![错误的DQ相关Command Interleaving](/img/sca_if/sca_command_interleave.png "错误的DQ相关Command Interleaving")

* LUN0 Plane0的DQ相关命令发送
* 不允许在LUN0有一个DQ相关命令在等待执行时发送LUN0的DQ相关命令
* LUN0 SCE发送
* 不允许在LUN0有一个DQ相关命令在执行时（DQ正在传输数据）发送LUN0的DQ相关命令
* LUN0 SCT发送
* LUN1 Plane0的DQ相关命令发送
* LUN1 SCE发送
* 不允许在LUN1有一个DQ相关命令在执行时（DQ正在传输数据）发送LUN1的DQ相关命令
* LUN1 SCT发送

![正确的单CA_CE# DQ相关Command Interleaving](/img/sca_if/correct_sca_cmd_interleave.png "正确的单CA_CE# DQ相关Command Interleaving")

* LUN0 Plane0的DQ相关命令发送
* LUN0 SCE发送
* LUN1 Plane0的DQ相关命令发送，DQ正在传输数据LUN0的数据
* LUN0 SCT发送
* LUN1 SCE发送
* LUN0 Plane1的DQ相关命令发送，DQ正在传输数据LUN1的数据
* LUN1 SCT发送
* LUN0 SCE发送
* LUN1 Plane1的DQ相关命令发送，DQ正在传输数据LUN0的数据
* LUN0 SCT发送
* LUN1 SCE发送
* LUN0 DQ相关命令发送，DQ正在传输数据LUN1的数据
* LUN1 SCT发送

![正确的多CA_CE# DQ相关Command Interleaving](/img/sca_if/dq_related_cmd_lun_interleave.png "正确的多CA_CE# DQ相关Command Interleaving")

* CA_CE0# LUN0 DQ相关命令发送
* CA_CE0# LUN0 SCE发送
* DQ开始传输CA_CE0# LUN0数据
  * CA_CE0# LUN1 DQ相关命令发送
  * CA_CE1# LUN0 DQ相关命令发送
  * CA_CE1# LUN1 DQ相关命令发送
* CA_CE0# LUN0 SCT发送
* CA_CE0# LUN1 SCE发送
* DQ开始传输CA_CE0# LUN1数据
* CA_CE0# LUN1 SCT发送
* CA_CE1# LUN0 SCE发送
* DQ开始传输CA_CE1# LUN0数据
* CA_CE1# LUN0 SCT发送
* CA_CE1# LUN1 SCE发送
* DQ开始传输CA_CE1# LUN1数据
* CA_CE1# LUN1 SCT发送

![正确的DQ相关和DQ无关Command Interleaving](/img/sca_if/dq_related_and_not_dq_related_cmd_lun_leave.png "正确的DQ相关和DQ无关Command Interleaving")

* LUN0 page read命令发送（DQ相关）
* LUN1 page read命令发送（DQ相关）
* LUN0 read status命令发送（DQ无关）
* LUN0 random data out（change column/change column enhanced）命令发送（DQ相关）
* LUN0 SCE发送（DQ相关）
* DQ开始传输LUN0数据
  * LUN1 read status命令发送（DQ无关）
  * LUN2 set feature命令发送（DQ无关）
  * LUN1 read status命令发送（DQ无关）
  * LUN1 random data out（change column/change column enhanced）命令发送（DQ相关）
* LUN0 SCT发送（DQ相关）
* LUN1 SCE发送（DQ相关）
* DQ开始传输LUN1数据
  * LUN2 get feature命令发送（DQ无关）
  * LUN2 get feature数据接收（DQ无关）
* LUN1 SCT发送（DQ相关）

## Status Reads/LUN Interleaving During Get Feature/Get Feature by LUN
![读状态后的get feature数据取回](/img/sca_if/get_feature_after_read_same_lun.png "读状态后的get feature数据取回")

* get feature/get feature by lun时tFEAT期间的读状态时序由NAND厂商指定
* get feature/get feature by lun时读状态或LUN interleaving不允许没有busy时间
* 有busy时间的get feature/get feature by lun可以使用上图时序在读状态后获取feature

![Lun interleaving后的get feature数据取回](/img/sca_if/get_feature_after_lun_leaving.png "Lun interleaving后的get feature数据取回")

* 有busy时间的get feature/get feature by lun可以使用上图时序在LUN interleaving后获取feature

# CA训练寄存器（25h）（厂商可选支持）
![CA训练寄存器](/img/sca_if/ca_bus_training_seature_reg.png "CA训练寄存器")

* 25h寄存器是可选的，由厂商决定是否实现
* 该寄存器提供一个CA总线的训练机制
* 主控可以先向25h地址set feature，再对25h进行get feature
* 此寄存器不影响任何NAND功能，仅用于CA总线训练

# SCA DQ Mirror Mode Bit（厂商可选支持）

* 在02h地址Byte3[1]为SCA_DQMIRR
* 将SCA_DQMIRR配置为1可以使DQ总线顺序反转：DQ[7:0] -> DQ[0:7]
* 由于DQ总线在SCA协议中仅用于数据传输，实际上该位没有太大的意义，因为我们可以认为DQ总线的每一位都是等价的，可以直接在PCB上任意交换顺序；传统接口中由于DQ总线同时需要传输命令、地址、feature、状态等信息，所以DQ的每一位不是等价的

# SCA Protocol AC and DC Levels
![SCA协议交直流电压表](/img/sca_if/sca_power_ac_dc_level.png "SCA协议交直流电压表")

* CA Input Specs用于CA[1:0]，CA_CLK，CA_CE#，SCA信号（玩不输入到NAND）
* CA Output Specs用于CA[1:0]，NAND输出CA的驱动强度通常为50欧姆，并且不支持ZQ校准
* CA[1:0]，CA_CLK，CA_CE#工作于非端接模式（unterminated）
* 所有交流时序测量均基于信号与二分之一VCCQ的交点

# SCA Protocol AC Timings
![SCA协议交流时序参数](/img/sca_if/sca_timing_paramter.png "SCA协议交流时序参数")

# SCA复位限制
## FFh复位限制
1. 如果CA_CE#的某个LUN正在进行读数据burst（RE_t/RE_c正在翻转），主控需要先停止翻转RE_t/RE_c（RE_t保持为0，RE_c保持为1），然后在CA总线上发送FFh命令，不必在发送FFh命令前发送SCP/SCT包
2. 如果CA_CE#的某个LUN正在进行写数据burst（DQS_t/DQS_c正在翻转），主控需要先停止翻转DQS_t/DQS_c（DQS_t保持为0，DQS_c保持为1），然后在CA总线上发送FFh命令，不必在发送FFh命令前发送SCP/SCT包
3. 为了确保NAND设备接收到FFh命令，需要在发送FFh前发送一次命令指针复位序列

## LUN复位限制（FAh）
1. 如果CA_CE#的某个LUN正在进行读数据burst（RE_t/RE_c正在翻转），主控想要复位该LUN需要先停止翻转RE_t/RE_c（RE_t保持为0，RE_c保持为1），然后在CA总线上发送FAh命令，不必在发送FAh命令前发送SCP/SCT包
2. 如果CA_CE#的某个LUN正在进行写数据burst（DQS_t/DQS_c正在翻转），主控想要复位该LUN需要先停止翻转DQS_t/DQS_c（DQS_t保持为0，DQS_c保持为1），然后在CA总线上发送FAh命令，不必在发送FAh命令前发送SCP/SCT包
3. 如果主控想要复位的LUN没有在进行数据传输（同CA_CE#下的其他LUN可以进行数据传输），则不必停止RE_t/RE_c和DQS_t/DQS_c。前提是：
   1. 被FAh复位的LUN不为正在传输数据的LUN提供non-Target ODT
   2. 被FAh复位的LUN为正在传输数据的LUN提供non-Target ODT，但non-Target ODT不受到FAh命令的影响
4. 为了确保NAND设备接收到FAh命令，需要在发送FAh前发送一次命令指针复位序列

## 复位时序图
![复位中断读数据burst](/img/sca_if/sca_data_out_interupt_reset.png "复位中断读数据burst")

* 上图也可用于FAh命令
* tRPST和tRPSTH的计算以复位命令的最后一个CA_CLK下降沿为参考

![复位中断写数据burst](/img/sca_if/sca_data_in_interupt_reset.png "复位中断写数据burst")

* 上图也可用于FAh命令
* tWPST和tWPSTH的计算以复位命令的最后一个CA_CLK下降沿为参考

# SCA CA Slew Rate Derating
![SCA CA总线摆率降额](/img/sca_if/sca_slew_rate_derating.png "SCA CA总线摆率降额")

* 表中的降额单位是ps
* 当CA_CLK和CA[1:0]信号摆率小于1 V/ns时，tCACS和tCACH参数可以根据表格查询降额数据

# 总结
终于，我们介绍完了关于NAND SCA的内容。SCA协议将CA和DQ总线完全分离开来，有助于固件在上层提高调度的灵活性，但是其实对于底层的硬件实现而言是一个挑战，尤其是需要满足以下两点：

* 兼容传统非SCA协议
* CA总线上有些命令序列必须和DQ总线上的数据传输按照顺序执行

尤其是第二点，让CA总线的控制逻辑和DQ总线的控制逻辑不能完全分开，存在一些硬件层面的交互（不能完全由FW进行协调），不过正因为有这样的挑战，才让芯片设计人员有事可做

感谢您的阅读，相信您对SCA已经有一定的了解了，如有疑问请在评论区留言