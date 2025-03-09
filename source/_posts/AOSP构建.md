---
title: AOSP构建
date: 2025-03-08 20:47:20
tags:
---
Android系统无疑给我们的生活带来了深远的影响，目前我们的移动设备使用的操作系统只剩下iOS和Android两个。Android在嵌入式设备中也得到广泛的应用，比如你可以看见各种广告机，POS机等设备也在使用Android系统。提到Android就不得不提到Google，一家在全球具有相当影响力的公司，它在2005年收购了Android系统，并在后续将其开源，使得各个手机制造商能够免费获取到Android源代码，并进行自己的修改，也催生了如今精彩的移动互联网世界。我无意在此讨论关于Google与Oracle的纠纷，以及当年阿里巴巴的YunOS如何衰落，目前关于Harmony的争端，抑或是Nokia和微软的WP系统的兴衰。本文的目的是从更深的层面认识我们广泛应用的操作系统，并尝试构建它。
<!-- more -->

# 环境准备
在一开始准备尝试构建Android的时候，在互联网上找到的教程中提到的资源需求还是让我有点望而却步的，比如Google提到的硬件要求：

* 一个 64 位 x86 系统。

    注意：您可以在 32 位系统上编译低于 2.3.x 的 AOSP 版本。

* 如果要检出和构建代码，至少需要 400 GB 可用磁盘空间（250 GB 空间用于检出代码 + 150 GB 空间用于构建代码）。

    注意：如果您要检出镜像，则需要更多空间，因为完整的 Android 开源项目 (AOSP) 镜像包含所有使用过的 Git 代码库。

* 至少 64 GB RAM。Google 使用 72 核机器和 64 GB RAM 来构建 Android。采用此硬件配置时，一个完整的 Android build 大约需要 40 分钟；Android 增量 build 大约需要几分钟的时间。相比之下，使用 6 核机器和 64 GB RAM 构建一个完整 build 大约需要 6 个小时。

第三条配置要求在普通的个人PC上不是很容易实现，一开始我是有考虑过购买一台服务器的，不过一方面考虑到硬件成本，另一方面也要考虑到后期使用成本，就暂时放弃买服务器的想法了，不过既然有困难我们就要解决困难，资源有限我们可以用时间来弥补，我直接使用了我的笔记本来进行构建，实际尝试后发现情况并没有想象中的那么糟糕，具体配置如下：

1. 联想拯救者Y7000 2018款
2. CPU: Intel i7-8750H 6核12线程
3. RAM: 8GB + 16GB 非对称双通道DDR4 2666
4. Disk: 忆联128G SSD + 希捷 2T HDD + 西数 1T SSD
5. System: Kali Linux 2024

其中RAM中16GB是大学的时候购买的，为了编译AOSP，本次购买了一个西数的SN770 1T SSD，用于存储AOSP代码和构建AOSP。戏剧化的一点是，在我购买硬盘后没几天，西数就宣布彻底退出SSD市场，所有SSD相关业务将由SanDisk接手，看起来西数的做法和Intel拆分Altera很像，想必和Kioxia合并的方案被Hynix否决有一定的关系。

加装硬盘时遇到一个小插曲，拆开电脑发现根本没有多余的硬盘位给我加装硬盘，我之前认为空的一个硬盘位置上实际插着我的Windows系统盘（忆联 128G SSD），不得已又购买了一个移动硬盘盒来进行硬盘扩展，不过移动硬盘盒要过几天才能到，所以我先将系统盘拆下来然后把新硬盘插上去进行AOSP代码检出和构建尝试，主要是因为移动硬盘盒只有PCIe x2，而主板上的硬盘位则是PCIe x4的，理论上具有更高的数据带宽（这可能是后续编译时间短于预期的原因之一），本来准备后续直接将Windows系统盘放在移动硬盘盒上启动，后来发现不行，只能把新硬盘和Windows系统盘的位置进行交换，不过好在那时我已经在PCIe x4接口上完成了AOSP的首次构建。由于Windows系统盘只有128G，所以我当时装Linux时就将系统装在了机械硬盘上，这也是为什么我将Windows系统盘拔出后仍可以进入Linux完成AOSP构建。

# 新硬盘分区
强烈建议在SSD上构建AOSP，HDD实在太慢。

我对新硬盘的分区是这样的：

1. 460GB 用于AOSP代码存储和构建，使用ext4文件系统，因为有人提到如果使用NTFS可能会失败，不过我没有进行验证
2. 40GB 挂在为Linux swap分区，用于弥补RAM的不足，不过我没有监控构建过程中的swap占用情况，不知道是否真的需要40GB
3. 剩下的空间保留，等待后续使用

根据我的分区使用情况来看，其实购买512G的硬盘应该也是足够的，不过现在SSD价格也相当便宜了，所以可以买大一点

# 环境配置
上文提到，我使用的是Kali Linux，实际上Google官方和绝大多数网络教程都使用的是Ubuntu，也有部分使用Debian的，不过Kali也是一个基于Debian的发行版，所以也是可以的，另一方面，如果你不想安装物理Linux系统，也可以考虑使用虚拟机或者WSL，不过这两者可能存在一些额外的资源开销，所以最推荐的还是使用Linux物理机。

## 依赖安装
根据Google官方教程安装依赖：
```shell
sudo apt update
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev libc6-dev-i386 x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig -y
```

在更早的Android版本中，你可能需要使用到更多的依赖，比如以下命令中提到的依赖：
``` shell
sudo apt install unzip zip libssl-dev  libffi-dev gnupg flex bison gperf build-essential  curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 x11proto-core-dev libx11-dev libz-dev ccache libgl1-mesa-dev libxml2-utils xsltproc git python2 openjdk-8-jdk aptitude repo libncurses5-dev -y 
sudo apt install lib32ncurses5-dev 
sudo aptitude install libncurses5-dev -y
```
两个单独列出的依赖在最新的系统中已经不存在，所以如果你需要编译早期版本Android的话，需要考虑使用旧版的操作系统，如Ubuntu18

## 信息配置和目录创建
执行如下命令配置git信息：
```shell
git config --global user.email "你的email"
git config --global user.name "你的用户名"
```
用户名和email不需要真实信息，只用于表明我们的身份，如果没有配置，后续进行代码同步时会报错

执行如下命令创建工作目录：
```shell
mkdir android
mkdir android/bin
cd android/bin
```
为了更好的管理项目，建议新建一个目录进行操作，避免出现混乱

执行如下命令更新.bashrc
```shell
echo "PATH=$(pwd):\$PATH" >> ~/.bashrc # 当前处于android/bin中
echo "export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/'" >> ~/.bashrc
source ~/.bashrc
```
我们将当前目录加入PATH变量中，并设置了REPO_URL变量为清华大学的镜像源

## 下载monthly代码压缩包
由于国内网络环境打的原因，我们不能直接访问Google提供的AOSP代码仓库，只能通过国内的镜像进行访问，比如[清华源](https://mirrors.tuna.tsinghua.edu.cn/)，[中科大源](https://mirrors.ustc.edu.cn/)，[重大源](https://mirrors.cqu.edu.cn/#/)，[重邮源](https://mirrors.cqupt.edu.cn/)等。

由于直接传输大量文件夹和文件的速度远低于直接诶传输一个完整的文件，所以镜像源给我们提供了一种更友好的方式，先下载一个打包好的压缩包，然后在解压后的压缩包基础上进行代码同步，这样就不需要传输大量的文件夹和文件了

下载monthly压缩包：
1. 通过网页下载： https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/
2. 通过wget下载： https://mirrors.ustc.edu.cn/aosp-monthly/aosp-latest.tar

推荐使用wget方式，执行如下命令即可，由于每个月压缩包命名不同，所以镜像中还提供了一个aosp-latest.tar指向最新的一个压缩包，所以我们直接下载最新压缩包即可
```shell
wget -c https://mirrors.ustc.edu.cn/aosp-monthly/aosp-latest.tar
wget -c https://mirrors.ustc.edu.cn/aosp-monthly/aosp-latest.tar.md5

my_md5=$(md5sum aosp-latest.tar)
md5=$(cat aosp-latest.tar.md5)
if [ "${my_md5:0:31}" != "${md5:0:31}" ]; then
    echo "md5sum check failed, please try again."
    exit 1
fi

tar -xvf aosp-latest.tar # aosp/.repo
cd aosp/
```

wget -c表示我们需要断点续传，避免出现网络问题导致下载失败，我们不仅下载了代码压缩包，还下载了压缩包对应的md5文件，用于文件的完整性校验，当前最新的代码压缩包大小为84GB，所以完整性校验是必要的，在if判断中只判断了md5部分，后面的文件名部分则忽略掉。由于每个人的网络环境不同，所以需要的等待时间也不同，我下载代码压缩包花费了大概8个小时，另外如果你的网络是按流量计费的，请注意流量消耗。md5的计算大概需要十几分钟，压缩包解压则需要半个小时到一个小时。解压完成后我们进入aosp文件夹爱，准备开始进行代码同步

## 代码同步
以下是进行代码同步的命令，通常情况下建议定期进行代码同步，避免在长时间不同步之后产生各种各样的问题

```shell
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-15.0.0_r3
repo sync -j4 
```
首先我们使用repo init指定我们需要的分支和代码仓库的地址，然后进行repo sync，由于镜像的限制，我们只使用4个线程进行同步，避免被封IP

## 构建代码
读入环境设置
```shell
source build/envsetup.sh
```
查看可用产品版本
```shell
list_products
list_releases [PRODUCT]
list_variants [PRODUCT] [RELEASE]
```
以上三个命令必须在读入环境设置后执行，否则会找不到命令，这里产品版本列出命令和早期的命令不一样了，早期版本使用lunch命令即可直接查看产品版本，现在直接使用lunch则会提示你使用以上三个命令查看

指定产品版本
```shell
# lunch <product>-<release>-<variant>
lunch sdk_phone64_x86_64-ap3a-eng
```
根据刚才查看的产品和版本呢，使用lunch命令指定我们需要构建的产品版本，我们选择编译一个x86_64的enginer版本

开始构建
```shell
make -j10
```
Intel i7-8750H有6个核共12线程，这里我使用了10个线程进行编译，最终经过4.5个小时完成了第一次构建，如果使用12线程的话可能还有下降空间，构建过程中可能会遇到大量Warning，但是这还不是我们需要关心的东西

## 启动模拟器
使用如下命令启动我们编译好的模拟器
```shell
emulator
```
![模拟器界面](/img/aosp_build/android15_emulator_screen_shoot.png "模拟器界面")

![模拟器日志](/img/aosp_build/android15_emulator_log.png "模拟器日志")

# 重新回到目录
当我们完成工作后，退出工作目录，下次重新回到目录时需要做以下操作才能恢复工作环境上下文
```shell
cd android/bin/aosp
source build/envsetup.sh
lunch sdk_phone64_x86_64-ap3a-eng
```

# 结语

终于完成了AOSP的构建，还是比较顺利的，后续可能会进行一些代码调试和阅读的工作，更深入的理解Android系统，希望后期能够进行一些代码的定制和修改。我将以上命令整理成一个脚本并在github开源：[AOSP CHECKER](https://github.com/cqlhupt/AOSP-CHECKER/)