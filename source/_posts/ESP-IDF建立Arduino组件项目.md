---
title: ESP-IDF建立Arduino组件项目
date: 2024-11-03 12:27:01
tags:
---
Arduino是一个非常著名的开源开发板，是一款便捷灵活、方便上手的开源电子原型平台。包含硬件（各种型号的Arduino板）和软件（ArduinoIDE）。由一个欧洲开发团队于2005年冬季开发。其成员包括Massimo Banzi、David Cuartielles、Tom Igoe、Gianluca Martino、David Mellis和Nicholas Zambetti等。经过多年的发展，Arduino已经形成了一个庞大的生态，越来越多的硬件平台开始支持Arduino-IDE，乐鑫公司也为他们的ESP系列芯片提供了Arduino的支持，有两种方式：1. 在Arduino-IDE中安装esp开发板工具；2. 在ESP-IDF中将官方维护的Arduino包作为项目组件。
<!--more-->
# Arduino-IDE简介
本篇的重点不是Arduino-IDE，所以这里简单介绍一下它的安装和使用。

## Arduino-IDE下载
Arduino提供Windows，Linux和MacOS的支持，我们可以直接前往[官网](https://www.arduino.cc/en/software)下载安装包进行安装。

![Arduino IDE官网](/img/esp_arduino/arduino_ide_official.png "Arduino IDE官网")

安装完成后，进入Arduino IDE

![Arduino IDE界面](/img/esp_arduino/arduino_ide_gui.png "Arduino IDE界面")

选择工具->开发板->开发板管理器，打开开发板管理器后搜索esp32，选择安装Espressif提供的esp32包

![Arduino IDE开发板管理器](/img/esp_arduino/arduino_ide_board_manager.png "Arduino IDE开发板管理器")

如果你的网络能够正常访问Github，那么应该一段时间后开发板包就会安装完成，但是由于国内的网络管控，更可能出现的情况是你需要等待很长时间，并且可能由于网络波动而失败。另外Arduino-IDE会将这些开发板包安装到你的C盘，导致你的C盘容量告急。以上也是我不太推荐这种方式的主要原因。

当你安装完毕之后可以在右下角的没有选择开发板处选择ESP开发板，并在编辑器窗口编写你的代码。
# ESP-IDF组件
在上一篇博客[ESP-IDF环境搭建](https://cqlhupt.github.io/2024/11/01/ESP-IDF-environment-setup/)中介绍了在VS code中搭建ESP-IDF的方法，最终完成了使用官方例程hello_world进行打印的工作。

ESP-IDF中的project可以引入依赖组件，由于Arduino的开发便利性，语法也较为简答，所以乐鑫公司也提供了一个Arduino组件，让ESP的芯片可以使用Arduino的开发方式，大大降低开发和学习门槛，并且Arduino组件中包含了芯片中的硬件驱动供我们进行高抽象层级的调用。

首先打开VS code，按Ctrl + Shift + P，打开功能搜索框，搜索ESP-IDF:从扩展模板创建项目

![从扩展模板创建项目](/img/esp_arduino/arduino_component_project.png "从扩展模板创建项目")

选择项目存放目录

![选择项目目录](/img/esp_arduino/arduino_component_project_sel_dirctory.png "选择项目目录")

选择arduino_as_component

![选择扩展模板](/img/esp_arduino/arduino_component_project2.png "选择扩展模板")

随后ESP-IDF开始创建项目

![创建项目](/img/esp_arduino/idf_start_create_project.png "创建项目")

紧接着项目创建失败

![创建项目失败](/img/esp_arduino/idf_create_project_fail.png "创建项目失败")

这是由于ESP-IDF尝试从GitHub克隆[arduino-esp32仓库](https://github.com/espressif/arduino-esp32)，但是由于网络问题不能完成clone导致的失败，虽然组件仓库clone失败，但是其实IDF已经将项目的目录结构创建完成了，我们可以手动打开你刚才选择的项目文件夹，然后我尝试通过其他方式将仓库clone下来放到component文件夹中

![component目录](/img/esp_arduino/arduino_component_dirctory.png "component目录")

整个arduino-esp32仓库有1.6GB，当我尝试clone时发现很多镜像都有1GB流量的限制，导致clone到1GB之后就会失败，最后我找到的一个方法是通过CSDN提供的GitHub仓库加速服务进行clone。希望CSDN看到后能给我打钱。最终将仓库clone下来放进component文件夹，然后尝试进行项目构建，遇到了一些小问题，我们后面还会提到，先说最大的一个问题。

目前的git clone下来的Arduino组件并不能支持最新的ESP-IDF版本，这也是前一篇搭建ESP-IDF环境时选择v5.1.4版本的原因。当我们使用最新的ESP-IDF尝试使用Arduino组件时，构建项目时会遇到如下问题（部分信息）
``` shell
E:/vstudio/esp32_example_project/arduino-as-component_bk/components/arduino/cores/esp32/esp32-hal-i2c-slave.c:319:3: error: implicit declaration of function 'i2c_ll_set_fifo_mode'; did you mean 'i2c_ll_set_data_mode'? [-Werror=implicit-function-declaration]
  319 |   i2c_ll_set_fifo_mode(i2c->dev, true);
      |   ^~~~~~~~~~~~~~~~~~~~
      |   i2c_ll_set_data_mode
E:/vstudio/esp32_example_project/arduino-as-component_bk/components/arduino/cores/esp32/esp32-hal-i2c-slave.c: In function 'i2c_slave_set_frequency':
E:/vstudio/esp32_example_project/arduino-as-component_bk/components/arduino/cores/esp32/esp32-hal-i2c-slave.c:521:3: error: implicit declaration of function 'i2c_ll_cal_bus_clk'; did you mean 'i2c_ll_enable_bus_clock'? [-Werror=implicit-function-declaration]
  521 |   i2c_ll_cal_bus_clk(XTAL_CLK_FREQ, clk_speed, &clk_cal);
      |   ^~~~~~~~~~~~~~~~~~
      |   i2c_ll_enable_bus_clock
E:/vstudio/esp32_example_project/arduino-as-component_bk/components/arduino/cores/esp32/esp32-hal-i2c-slave.c:526:3: error: implicit declaration of function 'i2c_ll_set_bus_timing'; did you mean 'i2c_ll_set_scl_timing'? [-Werror=implicit-function-declaration]
  526 |   i2c_ll_set_bus_timing(i2c->dev, &clk_cal);
      |   ^~~~~~~~~~~~~~~~~~~~~
      |   i2c_ll_set_scl_timing
E:/vstudio/esp32_example_project/arduino-as-component_bk/components/arduino/cores/esp32/esp32-hal-i2c-slave.c:527:3: error: implicit declaration of function 'i2c_ll_set_filter'; did you mean 'i2c_ll_set_tout'? [-Werror=implicit-function-declaration]
  527 |   i2c_ll_set_filter(i2c->dev, 3);
      |   ^~~~~~~~~~~~~~~~~
      |   i2c_ll_set_tout
cc1.exe: some warnings being treated as errors
```
然后我去查看了错误信息中提到的文件以及函数，发现文件中确实没有定义这些函数，然后我就到Arduino仓库提了一个[issue](https://github.com/espressif/arduino-esp32/issues/10486)，后来官方回复说我使用的ESP-IDF版本不对，目前可用的应该是Arduino Release v3.0.6搭配ESP-IDF5.1.3版本。在我收到官方回复的当天他们又发布了Arduino Release v3.0.7

经过以上的踩坑我就发现一些问题：

1. 乐鑫公司既然在ESP-IDF安装时可以选择国内的服务器，为什么不在国内提供一个可以下载arduino-esp32 release的地方
2. 乐鑫公司在ESP-IDF创建arduino组件项目时为什么要尝试clone整个arduino-esp32仓库，而不是直接获取相应的release版本

希望以后能够有人能够解答我的疑问

随后我就发现另一条路：我们可以直接下载Arduino release压缩包，他只有几十MB，完全不能与整个仓库1.6GB相比，下载之后解压放到component文件夹中。然后再尝试构建整个项目。

![下载/解压/重命名](/img/esp_arduino/release_arduino_download_unpack.png "下载/解压/重命名")

尝试构建arduino_as_component

![Arduino组件项目构建](/img/esp_arduino/arduino_component_build.png "Arduino组件项目构建")

项目构建失败，原因如红框中所示

![Arduino组件项目构建失败](/img/esp_arduino/arduino_component_build_fail.png "Arduino组件项目构建失败")

打开项目根目录下的sdkconfig文件，找到CONFIG_FREERTOS_HZ=100，将100改为1000

![修复FREERTOS频率的问题](/img/esp_arduino/arduino_component_build_fail.png "修复FREERTOS频率的问题")

改完之后直接开始构建会遇到一个之前遇到过的问题
``` shell
ninja: error: loading 'build.ninja': 系统找不到指定的文件。
```
只需要删除构建之后再重新构建即可

![重新构建成功](/img/esp_arduino/arduino_component_build_success.png "重新构建成功")

# 总结
到目前位置，我们已经完成了arduino_as_component的环境构建，可以在main.cpp文件中编写我们的业务代码，通过包含官方的驱动头文件调用我们需要的功能。

![代码文件](/img/esp_arduino/code_view.png "代码文件")

我们可以展开component文件夹中的文件，这里面包含了官方的库源码，当然也可以参考[官方的文档](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/get-started/index.html)。

最后附上一段使用ntp服务从阿里云获取时间并配置到芯片上打印到串口的程序。
``` c++
#include "Arduino.h"
#include <WiFi.h>
#include "time.h"    // 为了使用tm结构体
#include "esp_sleep.h"

const char *ssid     = "Mi 10S"; // 请修改WiFi名称
const char *password = "qwertyur"; // 请修改WiFi密码

struct tm timeinfo;

extern "C" void app_main()
{
    initArduino();
    pinMode(4, OUTPUT);
    digitalWrite(4, HIGH);
    // Do your own thing

    Serial.begin(115200);
    Serial.printf("Connecting to %s ", ssid);
    WiFi.begin(ssid, password);
    while (WiFi.status() != WL_CONNECTED) 
    {
        delay(500);
        Serial.print(".");
    }
    Serial.println(" CONNECTED");
    
    configTime(60*60*8, 0, "ntp7.aliyun.com");    // 用的阿里云的服务器

    while (1)
    {
        /* code */
        struct tm timeinfo;
        if(!getLocalTime(&timeinfo))
        {
            Serial.println("Failed to obtain time");
            return;
        }
        Serial.println(&timeinfo, "%A, %Y-%m-%d %H:%M:%S");
        delay(1000);
    }
    
}
```