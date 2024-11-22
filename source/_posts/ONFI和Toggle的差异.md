---
title: ONFI和Toggle的差异
date: 2024-11-20 19:12:51
tags:
---
在[NAND Flash标准接口](https://cqlhupt.github.io/2023/08/12/NAND%20Flash%E6%A0%87%E5%87%86%E6%8E%A5%E5%8F%A3/)一文中我详细的介绍了ONFI协议，也给出了一些在设计NAND Flash控制器时遇到的一些问题和解决方案。这篇文章准备介绍一下ONFI和Toggle的差异，二者虽然都是基于[JESD230](https://www.jedec.org/document_search?search_api_views_fulltext=JESD230)开发，但是会有细微的差异。虽然说是讲ONFI和Toggle的差异，但是我也无权访问Toggle的协议文档，因为Toggle协议的授权费十分昂贵，并且需要签署相关的保密协议，所以只能基于能够访问到的一些NAND Flash datasheet进行对比，实际上这样更贴近实际情况。
<!-- more -->
# Read status, Read ID & Get feature
在read ID，get feature时，Toggle输出数据会提前（DQS第一个下降沿），ONFI输出数据则在DQS第一个上升沿，此时数据是repeat twice的，所以我们数据以第一个上升沿为准的话可以兼容两种。Toggle这样设计的初衷应该是让用户在repeat twice读时只使用DQS的上升沿（不用DLL延迟DQS）去采样数据，因为此时DQS的上升沿正好在数据window的中央，是最好的数据采样时机，而ONFI的设计则相反，repeat twice的数据window中央正好是DQS的下降沿（不用DLL延迟DQS），当我们考虑兼容两种协议的同时兼容读数据的时序时，使用DQS经过DLL延迟之后的上升沿采样数据是最好的选择

![Toggle read status](/img/onfi_diff_toggle/toggle_read_status.png "Toggle read status")

注意这里的DQS和DQSn信号在read status时undefined，根据某Toggle协议NAND datasheet的描述，此时虽然是DDR mode，但是建议直接使用REn进行status的采样，如果需要使用DQS进行采样则需要保证REn信号质量。在实际使用中，在tDQSRE时间结束之后，DQS会拉低，并且status会被驱动到DQ上

这里其实还有一个坑，由于一开始厂商提供的model存在问题，导致我们一开始没有注意到这个问题，但是我现在还不打算讲它，我会在本文末尾再callback这个问题

![ONFI read status](/img/onfi_diff_toggle/onfi_read_status.png "ONFI read status")

可以看到在ONFI上，当我们把RE_t拉低之后，经过tDQSD之后DQS会拉低，但是status没有被驱动到DQ上，当我们发送了RE_t脉冲之后，DQS回复时status才会被驱动到DQ上

![Toggle read id](/img/onfi_diff_toggle/toggle_read_id.png "Toggle read id")

Toggle的read id操作也是在tDQSRE结束，DQS被拉低时，第0字节的id就被驱动到了DQ上，并且DQS和DQ在被NAND驱动前都是High-z态，此状态表示主控和颗粒都没有驱动该信号，信号处于float状态

![ONFI read id](/img/onfi_diff_toggle/onfi_read_id.png "ONFI read id")

ONFI的read id操作则是在RE_t发出第一个上升沿之后，DQS_t回复第一个上升沿时id的第0字节才被驱动到DQ上，DQS在移交时没有出现High-z，只标注了短暂的x态（被颗粒驱动，但没有规定具体的值），在x态之前DQS一直被主控驱动为1，DQ则有部分High-z和x态

Get feature的时序和read id的时序是一致的，所以这里就不再赘述

# 读写数据时序

## 写数据时序
![Toggle write data](/img/onfi_diff_toggle/toggle_tcdqss.png "Toggle write data")

Toggle协议中描述的tCDQSS最小为100ns，虽然这里没有画出来，但是这里还有一个点要注意，就是program指令需要在发完最后一个数据DQS最后一个下降沿之后保持为0的时候发送，这个与ONFI是一致的，如下图ONFI中最后一部分的0x10命令

![ONFI write data](/img/onfi_diff_toggle/onfi_tcdqss.png "ONFI write data")

ONFI协议中描述的tCDQSS最小为30ns，另外还需要遵守tCD时序，program指令需要在发完最后一个数据DQS最后一个下降沿之后保持为0的时候发送

## 读数据时序

![Toggle read data hand over](/img/onfi_diff_toggle/toggle_read_bus_hand_over.png "Toggle read data hand over")

Toggle协议中在开始tRPRE前需要满足tCRES，并且在tRPRE开始后经过tDQSRE后DQS会被NAND拉低，在NAND拉低DQS前DQS可以是High-z，也就是说主控可以在RE_n拉低前就释放DQS，tDQSRE最大值是25ns

![ONFI read data hand over](/img/onfi_diff_toggle/onfi_read_bus_hand_over.png "ONFI read data hand over")

ONFI协议中在开始tRPRE前需要满足tDBS和tCR，当tRPRE开始后，主控仍需控制DQS为1满足tDQSRH时间，tDQSRH是5ns，在RE_n拉低之后经过tDQSD时间DQS会被NAND拉低，这里的tDQSD最大值是18ns，在NAND拉低DQS前，DQS需要保持为1，所以主控不能过早释放DQS，必须等待tDQSRH满足之后才能释放

**NAND Flash read data的结束**

读数据结束时tRPST+tRPSTH之后拉高RE_n，并强行接管DQS和DQ信号的做法是存在问题的，正确的操作方式其实是在tRPST之后拉高CE/CLE/ALE，tRPSTH之后再拉高RE_n，在tCHZ（CE拉高之后到NAND释放DQS和DQ的时间）之后接管DQS和DQ，如果你仔细观察上面两张图会发现Toggle对这种要求是明确的，但是ONFI这边没有明确的要求，但在实践过程中显然使用CE拉高进行结束是一种好的方案，这样的操作方式意味着读数据的总线释放与CE相关，如果在读数据结束后仍需要向同一个CE发送命令或者地址则需要再次将CE拉低，并且必须满足tCEH时间

# Hynix的set parameter
set parameter是Hynix独有的一个特殊功能，其作用是调整read时的判决电压，这个时序其实稍显奇怪，因为它完全是由Hynix独有的，其他的厂商基本上都使用set feature或者使用额外地址字节等能够与NAND普通操作兼容的操作进行电压调整

![SDR set parameter](/img/onfi_diff_toggle/h_sdr_set_parameter.png "SDR set parameter")

SDR模式下进行set parameter没有什么特别的，只需要发送命令0x36之后发送调整电压的地址，在tADL时间满足后将要设置的值发送给NAND即可，连续发送地址和数据，结束时发送0x16命令，在此过程中所有信号的要求值都是与普通SDR操作兼容的

![DDR multiple set parameter](/img/onfi_diff_toggle/h_ddr_set_parameter_multi.png "DDR multiple set parameter")

![DDR single set parameter](/img/onfi_diff_toggle/h_ddr_set_parameter_single.png "DDR single set parameter")

在DDR模式下set parameter分为multiple和single两种，观察时序发现在single时最后发送0x16命令时的DQS_t/c有明确的电平（DQS_t为0，DQS_c为1），而multiple则没有，据hynix说后续将会统一改成在发送16h时有明确DQS_t/c电平要求（与single的电平要求一致）的方式。发送地址时的DQS_t/c同样有明确的电平要求，是和发送0x16时相反的（DQS_t为1，DQS_c为0）。也就是说在发送完一个数据之后u，如果要继续发送地址，则应该在tWPST之后拉高DQS_t，如果要发送结束命令0x16，则应该保持DQS_t为低电平。如果你对NAND有一定的了解，那么你会发现，当我们在set parameter过程中要继续发送地址则DQS_t拉高是和set feature一致的，而要发送0x16结束命令时则与发送program命令的DQS_t要求是一致的，所以需要注意这个时序。set parameter是在Application Note中描述的，在普通的datasheet中并没有描述这个操作

# ODT_PIN的开启和关闭时间
ODT_PIN也是一个ONFI5.0时代才被各个NAND厂商引入的一个功能，以前的ODT控制是不需要专门用一个引脚进行控制的，引入ODT_PIN的目的是为了更精准的控制NAND端开启ODT的时间

ODT_PIN其实也是由DDR SDRAM先引入的，NAND接口一贯的借鉴DDR SDRAM协议，所以也才引入这个信号，不过NAND不是通过新增引脚来实现的，而是复用了原本的WP_n信号，通过set feature来选择WP_n引脚是WP_n还是ODT_PIN，当使用WP_n作为ODT_PIN时，默认不会对NAND进行写保护

![Toggle ODT_PIN on/off](/img/onfi_diff_toggle/h_toggle_odt_pin_pull_down.png "Toggle ODT_PIN on/off")

Toggle协议下，写数据时，ODT_PIN在DQS拉低之后经过tODTL之后拉低，在经过tODTPREH之后才能发送第一个DQS上升沿，这期间还要注意tWPRE时间的满足；在最后一个DQS下降沿之后要经过tODTPSTH才能拉高ODT_PIN，拉高之后要保持tODT2RST时间才能拉高DQS，注意在这期间还要满足tWPST时间

Toggle协议下，读数据时，ODT_PIN在RE_n拉低之后经过tODTL之后拉低，在经过tODTPREH之后才能发送第一个RE_n上升沿，这期间还要注意tRPRE时间的满足；在最后一个RE_n下降沿之后要经过tODTPSTH才能拉高ODT_PIN，拉高之后要保持tODT2RST时间才能拉高RE_n，注意在这期间还要满足tRPST时间

![ONFI ODT_PIN on/off](/img/onfi_diff_toggle/onfi_ym_odt_pin_pull_down.png "ONFI ODT_PIN on/off")

ONFI协议下，写数据时，在DQS_t仍为高电平时就要拉低ODT_PIN，并且ODT_PIN下降沿到第一个DQS上升沿要满足tODTSD时间（大于等于tCS2），在最后一个DQS_t下降沿之后要等待tODTHD时间（大于等于tWPST）才能拉高ODT_PIN

![ONFI ODT_PIN on/off](/img/onfi_diff_toggle/onfi_ym_read_odt_pin_pull_down.png "ONFI ODT_PIN on/off")

ONFI协议下，读数据时，在RE_t仍为高电平时就要拉低ODT_PIN，并且ODT_PIN下降沿到第一个RE_t上升沿要满足tODTSR时间（大于等于tCS2），在最后一个RE_t下降沿之后要等待tODTHR时间（大于等于tRPST）才能拉高ODT_PIN

# Data burst exit
NAND在读写过程中支持两种停止数据传输的方法：

* 通过暂停DQS和RE_n的边沿发送，让NAND暂停数据接收和输出
* 通过exit时序退出数据的发送和接受过程

ONFI中描述的能够通过暂停DQS和RE_n来暂停数据传输过程的最大速率为800MT/s(400MHz)；Toggle中描述的能够通过暂停DQS和RE_n来暂停数据传输过程的最大速率为1200MT/s(600MHz)

ONFI中描述的强制需要warmup（dummy toggle，latency cycle）的速率为大于800MT/s(400MHz)；Toggle中描述的强制需要warmup（dummy toggle，latency cycle）的速率为大于1200MT/s(600MHz)

Data burst exit功能能让主控控制数据流的启停，但是它没有直接使用DQS/RE_n进行暂停方便，但是由于NAND接口速率越来越高，为了保证信号质量，也是无奈之举。Data burst exit支持使用CE_n/ALE/CLE之一来进行，并且要注意退出时间，如果退出时CE_n拉高超过1us，则需要按照厂商指定（ONFI和Toggle未明确指定）的操作时序进行数据burst的恢复，这种恢复实际上用纯硬件逻辑方式是比较难实现的，通常需要使用微码架构配合一个小的CPU进行实现

![Toggle write data burst exit](/img/onfi_diff_toggle/h_toggle_write_exit.png "Toggle write data burst exit")

Toggle通过CE_n拉高进行write burst exit时，要在DQS为0时拉高CE_n，如果需要拉高CLE，则需要满足tCSD时间，DQS需要在满足tWPSTH之后才能拉高，在恢复write burst时，需要满足tCDQSS（最小100ns）和tCEH时间，如果warmup cycle不为0，则恢复burst时需要重新发送warmup cycle

Toggle通过CLE拉高进行write burst exit时，要在DQS为0时拉高CLE，DQS需要在满足tWPSTH之后才能拉高，在恢复write burst时，需要满足tCDQSS（最小100ns）和tCEH时间，如果warmup cycle不为0，则恢复burst时需要重新发送warmup cycle，通过ALE进行write burst exit的时序与使用CLE时一致

![Toggle read data burst exit](/img/onfi_diff_toggle/h_toggle_read_exit.png "Toggle read data burst exit")

Toggle通过CE_n拉高进行read burst exit时，要在RE_n为0时拉高CE_n，如果需要拉高CLE，则需要满足tCSD时间，RE_n需要在满足tRPSTH之后才能拉高，在恢复read burst时，需要满足tCRES（最小10ns），tCLR和tCEH时间，如果warmup cycle不为0，则恢复burst时需要重新发送warmup cycle

Toggle通过CLE拉高进行read burst exit时，要在RE_n为0时拉高CLE，RE_n需要在满足tRPSTH之后才能拉高，在恢复read burst时，需要满足tCRES（最小10ns），tCLR和tCEH时间，如果warmup cycle不为0，则恢复burst时需要重新发送warmup cycle，通过ALE进行read burst exit的时序与使用CLE时一致

![ONFI write data burst exit](/img/onfi_diff_toggle/ym_onfi_write_exit.png "ONFI write data burst exit")

ONFI通过CLE拉高进行write burst exit时，要在DQS为0时拉高CLE，DQS需要在满足tRPSTH之后才能拉高，在恢复read burst时，需要满足tDBS（最小5ns），如果warmup cycle不为0，则恢复burst时需要重新发送warmup cycle，通过ALE进行read burst exit的时序与使用CLE时一致

![ONFI read data burst exit](/img/onfi_diff_toggle/ym_onfi_read_exit.png "ONFI read data burst exit")

ONFI通过CLE拉高进行read burst exit时，要在RE_n为0时拉高CLE，RE_n需要在满足tRPSTH之后才能拉高，在恢复read burst时，需要满足tDBS（最小5ns），如果warmup cycle不为0，则恢复burst时需要重新发送warmup cycle，通过ALE进行read burst exit的时序与使用CLE时一致

# Warmup support

![WD not support warmup at all operation](/img/onfi_diff_toggle/wd_toggle_warmup.png "WD not support warmup at all operatio")

Warmup cycle是接口速率高到一定程度之后为了防止在tWPRE和tRPRE时间之后最早的几个cycle时DQS和RE_n不能完全拉到VOH电平，进而导致最开始的几个数据出现采样错误。Warmup cycle的别称有dummy toggle，latency cycle

通常情况下，当我们打开了warmup cycle配置之后的所有与数据相关的操作：读数据，写数据，set feature，get feature，read id，read status等都需要增加对应数量的warmup cycle，但是总有例外，如上图所示，该厂商的datasheet明确表示在read status，read id，get feature，set feature时不支持warmup cycle，也就是说就算配置了warmup cycle也不能在这些操作中使用

# Read status call back

![Toggle read status](/img/onfi_diff_toggle/toggle_read_status.png "Toggle read status")

前面有提到，Toggle在使用过程中希望我们直接使用RE_n进行采样（SDR和DDR），而不是通过DQS采样，并且还有一个tRPP要求，就是这个tRPP要求由于厂商的model问题我们一开始验证的时候没有发现，在厂商的新model中才检查出了这个问题。在ONFI中则没有tRPP的要求，只需要按照普通的DDR read操作进行read status即可（使用DQS采样status）

# 颗粒与Model的行为不一致

* 在电气特性方面，Modle是不能进行模拟的，所以关于电气特性方面的功能必须要做好万全的准备，避免出现一些数字验证中不能验证的情况导致问题
* 另一个方面是数字功能方面，由于做颗粒的人和写Model的人可能不是同一个，可能会出现颗粒和Model不一致的情况，比如我们之前遇到的问题就是在tR还未结束时直接开始发送RE_n进行read data，Model表现为会回复DQS，但数据不正确，而真正的颗粒则表现为直接不回复DQS

# 关于Hynix

在我的印象中，Hynix（非Solidigm）一直是遵从Toggle协议的，但是前段时间和他们对接时，他们说他们遵从的是JESD230和ONFI，并对外宣称支持ONFI。实际上根据他们提供的datasheet和仿真模型来看，他们明显参考了大量Toggle的细节。对此我的猜测是他们可能在准备从Toggle转向ONFI，可能与Toggle组织内部的成员产生了利益冲突（对，我说的是三星），但目前在他们的产品中遗留了相当一部分的Toggle的技术债。我认为他们如果要转型的话至少需要两到三代产品的时间，不过由于Hynix收购了Intel的NAND业务，这个转型过程可能没有那么艰难，因为原本Intel的路线是和Micron相同的，都是支持ONFI标准的

# 结语

本篇我主要介绍了ONFI和Toggle的差异，可能存在不全面的地方，如果后续发现其他的差异也会更新在这里，如果你希望跟我交流，请在评论区留下你的评论