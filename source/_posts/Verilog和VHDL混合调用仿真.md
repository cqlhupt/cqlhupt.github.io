---
title: Verilog和VHDL混合调用仿真
date: 2024-11-23 13:47:43
tags:
---

在芯片开发过程中，目前常用的语言仍是Verilog和VHDL，国内通常使用Verilog，而欧美的厂商常用VHDL，所以当我们外购IP时可能会遇到不同语言的模块，我们需要对不同的语言模块进行编译，综合，仿真等操作，这个时候就需要了解如何进行模块的调用和编译。
<!-- more -->

# Verilog中例化VHDL模块
这个很简单，直接按照普通Verilog模块的例化方式进行例化即可，端口支持位置索引和名称索引

```VHDL
entity test_module is
    port (
        a   : IN STD_LOGIC                      ;
        b   : IN STD_LOGIC                      ;
        s   : OUT STD_LOGIC_VECTOR(1 downto 0)
    );
end entity;
```
如上是一个简单的VHDL模块的头部声明，定义了一个名叫test_module的模块，该模块包含两个输入信号和一个输出信号，输入信号位宽为1，输出信号位宽为2，使用STD_LOGIC关键字定义1位的信号，使用STD_LOGIC_VECTOR关键字和downto关键字定义多位信号。注意VHDL内部不区分大小写，即TEST_MODULE和test_module代表的是同一个模块，关键字也不区分大小写。

Verilog例化，虽然VHDL不区分大小写，但是Verilog是区分的，所以建议在Verilog中例化VHDL模块时使用和VHDL内定义大小写一致的名称

```Verilog
module test_inst();
    reg a0, b0 ;
    wire [1:0] s0 ;

    reg a1, b1 ;
    wire [1:0] s1 ;

    test_module u0(
        a0   ,
        b0   ,
        s0
    );

    test_module u1(
        .a(a0)   ,
        .b(b0)   ,
        .s(s0)
    );
endmodule
```

# VHDL中例化Verilog

在VHDL中例化Verilog模块稍微麻烦一点，因为VHDL不能直接识别Verilog的模块，需要将Verilog先定义成VHDL中的component

```Verilog
module testmodule(
    input       a   ,
    input       b   ,
    output [1:0]s
);
endmodule
```

```VHDL
entity test_inst is
    port (
        ...
    );
end entity;

architecture arch of test_inst is
    -- declare vars, constant
    signal a, b : STD_LOGIC ;
    signal s : STD_LOGIC_VECTOR(1 downto 0) ;

    -- component
    component testmodule is 
        port (
            a0   : IN STD_LOGIC                      ;
            b0   : IN STD_LOGIC                      ;
            s0   : OUT STD_LOGIC_VECTOR(1 downto 0)
        );
    end component;

begin
    -- logic description

    -- instance
    u0 : testmodule
        port map(
            a => a0  ,
            b => b0  ,
            s => s0
        );

end architecture;
```
实际上，我们将Verilog模块声明为component的方法和声明entity头部的方式很像，必须在architecture的begin前声明component，例化时，使用port map关键字进行连线，连线时不论信号的方向如何，只需要使用=>运算符连接即可，运算符左边是component的接口信号，右边是我们声明的外部信号

# VCS混合Verilog和VHDL仿真
当我们使用VCS进行Verilog和VHDL的混合仿真时，需要使用VCS的三步仿真模式，因为VCS不能同时识别两种语言，如果你尝试直接使用vcs命令同时处理Verilog和VHDL，那么你会遇到如下错误：

```shell
Error-[CFCILFBI] Cannot find cell in liblist
```

**分析文件**
```shell
vhdlan -kdb -full64 -f vhdl_file.f -l vhdl_an.log
vlogan -kdb -full64 -timescale=1ns/1ps +libext+.v +v2k -sverilog -f verilog_file.f -l verilog_an.log.
```
我们打开-kdb选项，让vcs编译出Verdi可以打开的数据库，方便后续调试，Verdi不能直接读入vhdl_file.f来解析出设计的hierarchy，我们还可以在这两个命令中添加一些设计中需要的宏，控制编译的代码块

**编译成可执行文件**
```shell
vcs -lca +vcs+flush+all -timescale=1ns/1ps -debug_region=cell+lib +debug_acc+all -full64 -top testbench -fsdb -sverilog -l comp.log -detaceclockdata +fadb+delta +lint=TFIPC-L +lint=PCWM +nospecify +notimingcheck +warn=noLCA_FEATURES_ENABLED +warn=noDRTZ
```
添加的选项非常多，实际上普通的仿真可能不需要这么多选项，一般有问题时VCS会提示你添加哪些选项，这条命令需要在执行vhdlan和vlogan的目录执行，vcs会自动读取前面分析好的数据库，不需要在指令中显式的指定分析好的数据库，由于我们没有指定输出文件的名称，所以其默认是simv，如果你需要指定，你可以通过-o选项指定

**运行仿真**
```shell
./simv  # execute sim
verdi -ssf your_fsdb_name # open database and fssb waveform
verdi -dbdir simv.daidir  # open database only
```
运行仿真文件，等待仿真结束，并打开波形

# Synplify综合VHDL
当我们需要进行FPGA原型验证时，常用到Synplify工具，添加文件时需要注意以下点：

1. 不能添加重复的文件，包括文件名重复和内容重复，比如VHDL的library，相同的library只能读入一份
2. VHDL有严格的先后顺序，越顶层的文件需要越晚读入，VCS也有此问题，请注意
3. Synplify不能处理filelist，需要通过其他方式将filelist展开，逐行添加
   
   * add_file -verilog verilog_file_name.v
   * add_file -vhdl vhdl_file_name.vhd
4. 如果存在加密文件，如后缀为.vhd.e，可以直接使用add_file命令读入，Synplify会自行解密，VCS也可以直接将文件加入filelist中
5. verilog调用VHDL模块时，接口位宽必须完全匹配，否则Synplify会将VHDL处理成blakcbox，最终导致综合失败

# 结语
虽然这些都是很常用的技能，但是网上似乎资料很少，目前权且当作笔记记录在此，后续如有新增也会更新在此，如果有不当的地方还请在评论区留言讨论，感谢您的阅读