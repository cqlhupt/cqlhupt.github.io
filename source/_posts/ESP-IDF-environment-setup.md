---
title: ESP-IDF环境搭建
date: 2024-11-01 17:40:56
tags:
---
ESP-IDF(ESP IoT Development Framework)是乐鑫公司推出的开源ESP32开发框架，支持通过VS code插件安装，具有完善的文档，并且支持配置Arduino为项目组件，可以极大降低开发难度，使初学者快速上手。但是在安装过程中仍遇到一些问题，主要与国内的网络环境相关，编写这篇博客记录一下安装使用过程中遇到的坑和解决办法。
<!--more-->
# 目录
* 起因
* 第一次搭建
* Hello World工程
* 更便宜的ESP

# 起因
前一段时间在研究蓝牙打卡时心血来潮斥巨资购买了一块esp32c3开发板，用于研究蓝牙广播（后来发现嘉立创新用户购买更便宜，后面再说）。我参考的教程是[钉钉蓝牙打卡](https://github.com/skysilksock/dingding "钉钉蓝牙打卡")，根据作者的描述，我们需要两块开发板，我就先买了一块准备进行学习，一方面节约资金，一方面我在考虑另一套方案。如果你是一个有素养的嵌入式攻城狮，那么你大概率手里有USB转串口设备，此时你可以选择购买简约版（不自带串口），当然你也可以购买经典版，它有即插即用的优势。

![购买订单](/img/esp_idf/esp32_c3_order.jpg "购买订单")

# 第一次搭建
作为忠实的VS code用户，能用插件解决的问题都用插件解决，所以我准备参考网上的教程安装。首先需要准备VS code，如果你的计算机上没有VS code，你可以前往[官网](https://code.visualstudio.com)下载安装，如果你使用的是Windows，那么你还可以前往微软商店搜索安装。

![VS code官网](/img/esp_idf/vscode_official_download.png "VS code官网")
![微软商店链接](/img/esp_idf/vscode_ms_strore.png "微软商店链接")

安装完VS code之后，打开VS code并进入插件搜索界面

![ESP-IDF插件安装](/img/esp_idf/vscode_esp_idf_extention.png)

点击安装，等待安装完成，安装完成后左侧快速访问栏中出现插件的logo，点击插件logo，如果安装完成后没有看见logo，请点击快速访问中的三个点然后选择<kbd>ESP-IDF：资源管理器</kbd>

![ESP-IDF：资源管理器](/img/esp_idf/vscode_esp_idf_extention_gui.png "ESP-IDF：资源管理器")

点击<kbd>Configure extentions</kbd>，进入ESP-IDF的真正安装界面

![ESP-IDF安装选项](/img/esp_idf/vscode_esp_idf_install_startup.png "ESP-IDF安装选项")

界面中有三个选项，从上到下分别是
* 快速安装：选择ESP-IDF，ESP-IDF Tool以及python安装路径进行安装
* 高级安装：选择ESP-IDF，ESP-IDF Tool以及python安装路径以及每个ESP-IDF Tool的安装路径
* 使用已经安装的ESP-IDF配置

我们选择快速安装即可，进入安装界面

![配置安装](/img/esp_idf/vscode_esp_idf_directory_sel.png "配置安装")

安装说明：
* Select download server：选择Espressif(Better speed for China)，如果你的网络可以正常访问Github也可以选择Github
* Select ESP-IDF version：选择ESP-IDF版本，目前建议选择v5.1.4，由于我已经安装了v5.1.4，所以这里用v5.2.3作为示例
* Enter ESP-IDF container directory：选择ESP-IDF容器安装文件夹
* Enter ESP-IDF Tools directory(IDF_TOOLS_PATH)：选择ESP-IDF工具安装文件夹

两个安装文件夹尽量选择非C盘，因为Windows的某些机制导致系统内的应用都疯狂的想占领C盘，但C盘的空间往往并不富裕，当然如果你恰好颇有家资请当我没说。配置完成后点击右下角<kbd>Install</kbd>即可，等待安装完毕。

![安装中](/img/esp_idf/esp_idf_installing.png)

# Hello World工程
安装完毕后回到插件的起始界面，点击<kbd>Show examples</kbd>，编辑器上方弹出EDF-IDF工具选择，点击图中的2处我们安装ESP-IDF的路径，这个路径是我们刚才安装的，如果你安装了多个版本的ESP-IDF，可以点击<kbd>Configure extentions</kbd>在通过界面中的使用已经安装的ESP-IDF配置进行选择

![建立Hello World工程](/img/esp_idf/esp_idf_example_project.png "建立Hello World工程")

随后进入示例工程选择界面，这里包含官方提供的各种例程，包括例程的介绍和支持的芯片，后续可以通过阅读这些例程深入学习相关的使用方法，现在我们就选择hello_world工程进行创建

![hello_world例程](/img/esp_idf/esp_hello_example.png "hello_world例程")

选择我们的开发板芯片esp32c3，然后点击<kbd>Create project using example hello_world</kbd>，随后选择一个文件夹，创建时工具会自动在该文件夹下创建一个新文件夹并将例程文件放在其中，所以建议新建一个文件夹专门用于放置ESP-IDF的项目

![工具新建的hello_world窗口](/img/esp_idf/new_hello_world_project.png "工具新建的hello_world窗口")

点击底部的小扳手构建项目，然后出现了报错，提示build.ninja文件找不到，我们点击扳手左边的垃圾桶来清除构建，然后再点击扳手重新构建项目即可成功构建

![hello_world构建成功](/img/esp_idf/hello_world_build_success.png "hello_world构建成功")

第一次搭建环境即告完成，现在我们可以使用USB数据线将开发板连接到计算机，然后通过左下角的COM口配置选择开发板的串口号，串口号可能不一样，如果你购买的是简约版，那么你需要选择你的USB转串口设备的串口号，并将USB转串口设备通过杜邦线链接到开发板的串口引脚

![esp32c3开发板连接](/img/esp_idf/esp32c3_board_link.jpg "esp32c3开发板连接")
![开发板串口选择](/img/esp_idf/esp32c3_com_sel.png "开发板串口选择")

点击图中3处标记的闪电，初次烧录程序会弹出烧录方式选择，我们选择UART即可，后续如果需要改变烧录方式可以通过点击闪电左边的五角星来改变。随后工具开始烧录程序

![开发板程序烧录](/img/esp_idf/esp32c3_flash.png "开发板程序烧录")

烧录完成后会有如下提示：

```shell
Flash Done ⚡️
Flash has finished. You can monitor with ESP-IDF: Monitor your device command
```

点击闪电右边的显示器按钮，就可以看到开发板打印出来的信息

![开发板串口打印](/img/esp_idf/esp32c3_hello_print.png "开发板串口打印")

# 更便宜的ESP
在购买了esp32c3开发板之后，突然发现立创商城提供新用户优惠，可以1元购买ESP32-WROOM-32模块，虽然需要付邮费，但是可以选择比较经济的邮寄方式，并且有邮费减免，最终实付3元。希望嘉立创看到的话能够给我打钱😉

![立创商城订单](/img/esp_idf/lichuang_store_order.jpg "立创商城订单")