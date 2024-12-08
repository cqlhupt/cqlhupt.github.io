---
title: Android逆向实践
date: 2024-12-08 12:53:46
tags:
---
最近在研究逆向Android apk，就找了之前玩过的一个单机游戏[《艾彼》](https://abi.lilith.com/en/pc/index.html)进行学习，本文中将会使用到Java、Java虚拟机、Java字节码、Android killer、Android开发相关知识，本文不涉及如上知识讲解，如果你具备如上知识，那么这篇文章可以作为快速入门Android逆向指南，本文中我们将会逆向《艾彼》的支付流程，并进行修改。当我们掌握Android逆向之后，能够在Android虚拟机，Java语言底层实现上获得更深刻的认识，可以帮助我们进行Android代码审计，获取app中的调度逻辑等
<!-- more -->

# 莉莉丝游戏
[上海莉莉丝科技股份有限公司](https://ancient.lilith.com/)（Shanghai Lilith Technology Corporation），简称莉莉丝游戏（Lilith Games）。莉莉丝游戏创立于2013年5月，2019年入选领英“顶尖创业公司排行榜” ，同年被毕马威评为“2019中国出海品牌50强”；总部位于中国上海。莉莉丝游戏致力为全世界玩家提供超越预期的游戏体验，研发和发行了《小冰冰传奇》《剑与家园》《艾彼》《迷失岛2：时间的灰烬》《剑与远征》《南瓜先生2》等多款作品。（以上内容来自百度百科）

![莉莉丝Logo](/img/apk_reverse/abi_lilith_banner.jpg "莉莉丝Logo")

# 声明
本文仅供学习参考，其中涉及的一切资源均来源于网络，请勿用于任何非法行为，否则您将自行承担相应后果

本文不提供任何工具，修改前后文件下载

![莉莉丝警告](/img/apk_reverse/abi_warning.jpg "莉莉丝警告") 

# 艾彼启动
![艾彼启动](/img/apk_reverse/abi_startup.jpg "艾彼启动")

2016年，《艾彼》第一次对外亮相，当时并没有太多提及游戏内容。2017年，《艾彼》正式于全球上线。在本体发布4年之后（2022），宣布更新DLC内容“大都市”，并且在7月8日到8月8日期间限时免费体验全部内容（含DLC）。目前限免已经结束，所以我们进入游戏体验完免费章节之后，就会出现付费购买提示

![去往大都市的飞船](/img/apk_reverse/abi_outside_of_spaceship.jpg "去往大都市的飞船") 

进入去往大都市的飞船后，爬上二层平台，接通飞船内部电源，然后回到地面尝试启动举升机械臂

![尝试启动机械臂](/img/apk_reverse/abi_lift_start.jpg "尝试启动机械臂") 

付费提示出现，我们点击确定

![付费提示](/img/apk_reverse/abi_payment_notify.jpg "付费提示") 

点击确定之后，弹出选择支付方式，我们选择支付宝

![付费方式选择](/img/apk_reverse/abi_payment_select.jpg "付费方式选择") 

选择支付宝之后，游戏会拉起我们手机上的支付宝进行支付，价格是6元

![付费6元](/img/apk_reverse/abi_pay.jpg "付费6元") 

我们的目的不是付费购买，所以我们直接进入下一个章节，由于整个逆向过程较长，导致下一章的篇幅较长，请做好准备

# Android killer启动

本文使用Android killer进行apk反编译，Android killer是一个apk反编译集成环境，支持对apk文件进行解包，将dex反编译为jar，分析apk结构，反编译smali为java等功能，我下载的是portable版本，解压之后直接点击运行

![Android killer启动](/img/apk_reverse/android_kill_startup.png "Android killer启动") 

选择我们要逆向的apk文件，比如abi_v5.0.3.apk，这里的apk名称是我重命名的，因为我在网上下载的时候介绍上写的是5.0.3版本，但是我实际反编译之后发现这个其实是5.0.2版本的

![打开APK文件](/img/apk_reverse/android_kill_open_file1.png "打开APK文件") 

Android killer反编译文件

![Android Killer反编译](/img/apk_reverse/android_kill_decompile.png "Android Killer反编译") 

反编译完成后提示是否分析文件，我们选择是

![Android Killer分析文件](/img/apk_reverse/android_kill_analysis.png "Android Killer分析文件")  

文件分析完成

![Android Killer分析完成](/img/apk_reverse/android_kill_analysis_done.png "Android Killer分析完成") 

我们尝试搜索付费提示

![搜索付费提示](/img/apk_reverse/android_kill_search_info.png "搜索付费提示") 

没搜到，我们尝试将中文转成Unicode编码再搜索

![文字转换成Unicode](/img/apk_reverse/android_kill_search_unicode.png "文字转换成Unicode")

![搜索付费提示的Unicode](/img/apk_reverse/android_kill_search_info2.png "搜索付费提示的Unicode") 

还是没搜到，这可能是因为提示词不是直接完整存储的字符串，而是在运行过程中临时拼接在一起的，我们尝试进行部分字符串的搜索，比如搜索解锁

![搜索解锁](/img/apk_reverse/android_kill_search_info3.png "搜索解锁") 

依然没有结果，我们转成Unicode再试一下

![搜索解锁的Unicode](/img/apk_reverse/android_kill_search_info4.png "搜索解锁的Unicode") 

终于找到了，我们找到了一个叫lilithPay的函数，根据这个名称我们可以确定这个函数是我们支付时需要调用的，另一方面也说明这个apk中的代码没有被混淆和加固，这为我们节约了很多时间，不需要进行apk脱壳和反混淆，Android killer在反编译代码成Java时可以在jadx界面进行反混淆

![lilithPay函数](/img/apk_reverse/android_kill_lilith_pay.png "lilithPay函数") 

我们直接将lilithPay所在的Activity类（UnityPlayerActivity）进行反编译

![反编译lilithPay所在的Activity成Java](/img/apk_reverse/android_kill_decompile_activity.png "反编译lilithPay所在的Activity成Java") 

反编译完成，我们可以看到lilithPay的Java代码如下，首先构建了一个PayRequest.Builder对象，并进行了支付请求的属性设置，其中有一个setPrice()函数，传入的参数是UnitPrice，我们找到UnitPrice的定义发现它的值是600（0x258），由于浮点数的精度问题，所以一般在进行价格相关的计算和存储时直接使用int或者long类型，避免由于误差导致出现账单不平问题，由于目前最低的价格单位是0.01元，所以在存储时直接将数据乘以100得到int数，类似于定点数，不过这个定点是定在十进制的，而不是通常的二进制定点数

![UnityPlayerActivity代码](/img/apk_reverse/android_kill_activity_java.png "UnityPlayerActivity代码")

我们在smali文件中搜索0x258，并将其改成0，注意我们可能会搜索到很多文件，要注意区别包名路径，smali\android\support是Android提供的依赖，不要乱改

![搜索金额](/img/apk_reverse/android_kill_search_info5.png "搜索金额") 

![修改smali代码中的头部定义价格](/img/apk_reverse/android_kill_change_price1.png "修改smali代码中的头部定义价格") 

![修改smali代码中立即数价格](/img/apk_reverse/android_kill_change_price2.png "修改smali代码中立即数价格") 

在构建好PayRequest.Builder对象之后，lilithPay方法调用了UniIndeSDK的方法，我们直接搜索UniIndeSDK，找到它的定义

![UniIndeSDK类](/img/apk_reverse/android_kill_search_UniIndeSDK.png "UniIndeSDK类")

这里我们遇到了lilith设下的第一个障眼法，我们要寻找UniIndeSDK中的后续调用，先来看lilithPay调用的函数getInstance和startPayWithApplyingOrderId，getInstance比较简单，就是获取一个UniIndeSDK对象，我们着重看startPayWithApplyingOrderId

![反编译UniIndeSDK](/img/apk_reverse/android_kill_decompile_UniIndeSDK.png "反编译UniIndeSDK")

可以看到函数中创建了一个变量iUniIndeInner，该变量来自本类的mProxy变量，如果该变量不空则调用变量的startPayWithApplyingOrderId函数，那么我们寻找一下mProxy的来源

![mProxy的初始化](/img/apk_reverse/android_kill_UniIndeSDK_mProxy.png "mProxy的初始化")

可以很明显的看到，这里使用了Java中的反射，构建了一个以getMetaValue()的返回值为类名的对象

可以看到调用getMetaValue()时传入的参数为lilith_uni_inde_proxy，搜索一下这个字符串

![AndroidManifest文件中的meta-data标签](/img/apk_reverse/android_kill_lilith_uni_inde_proxy_meta_data.png "AndroidManifest文件中的meta-data标签")

完整的xml标签如下，meta-data是Android提供的一种配置数据映射方式，这个标签可能是lilith将游戏上架到TapTap平台时平台反编译这个apk并注入的（跟我们现在做的事情很像，不过目的是相反的），这里有一个完整的类路径，我们可以直接去找到UniInappTapTap类，也可以进行搜索

```xml
<meta-data android:name="lilith_uni_inde_proxy" android:value="com.lilith.sdk.uni.in.app.taptap.UniInappTapTap"/>
```

![UniInappTapTap类](/img/apk_reverse/android_kill_UniInallTapTap.png "UniInappTapTap类")

![UniInappTapTap类反编译](/img/apk_reverse/android_kill_UniInappTapTap_decompile.png "UniInappTapTap类反编译")

让我们找一下被调用的startPayWithApplyingOrderId函数，发现在这个类中没有发现这个函数，然后我们发现UniInappTapTap的父类是BaseUniWechatAlipayProxy，我们可以到其父类中寻找这个函数

![BaseUniWechatAlipayProxy类反编译](/img/apk_reverse/android_kill_BaseUniWechatAlipayProxy.png "BaseUniWechatAlipayProxy类反编译")

可以发现BaseUniWechatAlipayProxy类中也没有startPayWithApplyingOrderId函数，我们继续找BaseUniWechatAlipayProxy的父类BaseUniIndeProxy

![BaseUniIndeProxy类](/img/apk_reverse/android_kill_BaseUniIndeProxy.png "BaseUniIndeProxy类")

我们终于找到了startPayWithApplyingOrderId函数

startPayWithApplyingOrderId函数最后一行调用了postLilithOrderIdRequest函数，这个函数的最后一个参数是LilithRequestListener对象，我们观察到传入的参数是

```Java
new 14(this, copyToBuilder, activity, uIConfig)
```

这是smali的内部类反编译出来的代码问题，正常的代码中是不允许使用数字开头的标识符的，它代表的是BaseUniIndeProxy$14这个类，它是BaseUniIndeProxy的内部类，有一个单独的smali文件

![postLilithOrderIdRequest函数](/img/apk_reverse/android_kill_BaseUniIndeProxy_postLilithOrderIdRequest.png "postLilithOrderIdRequest函数")

![BaseUniIndeProxy$14类](/img/apk_reverse/android_kill_BaseUniIndeProxy_inner14.png "BaseUniIndeProxy$14类")
BaseUniIndeProxy$14类是一个监听器，有onFail和onSuccess两个函数，重要的代码是onSuccess的：

```Java
// onSuccess
if (jSONObject != null && jSONObject.has("serial")) {
    this.val$copyedBuilder.setOrderId(jSONObject.optString("serial"));
    new Handler(Looper.getMainLooper()).post(new 1(this));
    return;
}
```
在成功的时候将json对象中的serial字段设置到val$copyedBuilder实例的orderId中，并创建了一个Handler对象，调用了Handler对象的post方法，post函数的参数是一个BaseUniIndeProxy$14的内部类，并且是Runnable的子类（post的参数是Runnable对象），根据前面的经验该类的名字应该是BaseUniIndeProxy$14$1

![BaseUniIndeProxy$14$1类](/img/apk_reverse/android_kill_BaseUniIndeProxy_inner14_inner1.png "BaseUniIndeProxy$14$1类")

可以发现，该类实现了Runnable接口，并且调用了其上两个层级的startPay函数，也就是BaseUniIndeProxy类中的startPay函数

![BaseUniIndeProxy类](/img/apk_reverse/android_kill_BaseUniIndeProxy.png "BaseUniIndeProxy类")

startPay函数就在startPayWithApplyingOrderId函数上方，在startPay中又post了一个BaseUniIndeProxy$13对象

![BaseUniIndeProxy$13类](/img/apk_reverse/android_kill_BaseUniIndeProxy_inner13.png "BaseUniIndeProxy$13类")

在这里我们陷入了死循环，BaseUniIndeProxy$13类的run方法中也调用了BaseUniIndeProxy类的startPay函数，这意味着我们应该要回头了，那么我们回到哪里呢，回想一下我们是追踪startPayWithApplyingOrderId函数到这里的，不妨回到入口第一层级的BaseUniWechatAlipayProxy类

![BaseUniWechatAlipayProxy类反编译](/img/apk_reverse/android_kill_BaseUniWechatAlipayProxy.png "BaseUniWechatAlipayProxy类反编译")

我们观察doStartPay函数，发现这个函数是我们选择支付方式的对话框弹出函数，它内部构建了两个BasePayStrategy对象，分别用于支付宝和微信支付，后续就是支付方式对话框展示逻辑，我们可以去看一下BasePayStrategy类的getPayStrategy()函数

![BasePayStrategy类](/img/apk_reverse/android_kill_BasePayStrategy.png "BasePayStrategy类")

这就是第二层障眼法了，可以发现getPayStrategy()首先调用了initPayStrategy()函数，该函数中put了两个类名字符串，很明显是要使用反射来构建对象了，在getPayStrategy()函数的最后调用了createStrategy()函数，在createStrategy()中就使用了反射创建对象，由于我使用支付宝更多，所以我直接追踪"com.lilith.sdk.uni.in.app.wechatAlipay.alipay.AliPayStrategy"类，微信的支付类读者可以自行尝试追踪和修改

![AliPayStrategy类](/img/apk_reverse/android_kill_AliPayStrategy.png "AliPayStrategy类")

AliPayStrategy类是BasePayStrategy类的子类，有一个内部类对象mPaySignObserver，这个对象最终被添加到了mThirdPayObservable对象中，这个对象是BasePayStrategy类的一个静态变量，作为支付观察者对象的容器，最终的结果判断仍需要其内部的对象进行，所以我们继续看mPaySignObserver的来源AliPayStrategy$1类

![AliPayStrategy$1类](/img/apk_reverse/android_kill_AliPayStrategy_inner1.png "AliPayStrategy$1类")

该类中存在onFail和onSuccess两个函数，其中onSuccess函数进行了一大堆的参数合法性判断，最后尝试构建了一个AliPayStrategy$1$1类并启动其内部进程，结束后从mThirdPayObservable中删除了mPaySignObserver

这里的fail和success不是支付结果的状态，而是支付过程是否成功结束，结果要在AliPayStrategy$1$1类中进行判断

![AliPayStrategy$1$1类](/img/apk_reverse/android_kill_AliPayStrategy_inner1_inner1.png "AliPayStrategy$1$1类")

可以发现该类的run函数中进行了PayTask的发起，并接收返回的字符串为pay变量，然后构建了一个ResultParser对象对返回码进行解析，通过ResultParser的getResultStatus()函数获取结果返回码resultStatus，随后进行返回码的判断，可以发现其中9000就是支付成功的返回码，所以我们要修改getResultStatus()函数的返回值为9000(0x2328)，进入success条件分支后，进行了二次确认，通过ResultParser的getSuccess()和getSign()函数获取success和sign字符串，并判断success是否为"true"，判断sign是否为空，所以我们要修改getSuccess()函数的返回值为"true"，修改getSign()函数的返回值为非空字符串，比如"sign"，然后run就会调用notifyPayFinish()函数通知上层代码支付已经成功，并打开后续游戏关卡，我们就可以继续玩耍了

![修改getResultStatus()函数](/img/apk_reverse/android_kill_resultparser_getResultstatus.png "修改getResultStatus()函数")

![修改getSign()函数](/img/apk_reverse/android_kill_resultparser_getsign.png "修改getSign()函数")

![修改getSuccess()函数](/img/apk_reverse/android_kill_resultparser_getsuccess.png "修改getSuccess()函数")

修改完之后记得保存所有被修改过的文件，没保存的文件名前会有一个*号，所以，保存之后我们进行重新编译和签名

![重新编译](/img/apk_reverse/android_kill_recompile.png "重新编译") 

![重新编译结束](/img/apk_reverse/android_kill_recompile_done.png "重新编译结束") 

点击蓝色的路径会自动打开文件所在目录，我们将该文件传输到手机上

# 重新安装

由于我们进行了重新编译和签名，所以我们的签名和这个apk的原始签名会不一致，如果我们直接覆盖安装的话会提示签名不一致问题，导致安装失败，所以我们要先卸载原来安装的应用，然后使用新的安装包重新安装

![艾彼安装](/img/apk_reverse/abi_install.jpg  "艾彼安装") 

由于我们卸载之后重新安装了应用，所以我们需要重新将游戏打到大都市飞船章节，重新触发支付流程，选择支付宝支付

![支付失败](/img/apk_reverse/abi_pay_fail.jpg "支付失败") 

由于我们将订单金额改成了0，所以这里支付宝会提示我们支付订单金额异常，这表明我们的修改成功了，我们点击确定，之后返回到机械臂启动界面，我们再次尝试启动机械臂

![再次启动机械臂成功](/img/apk_reverse/abi_lift_start_success.jpg "再次启动机械臂成功") 

机械臂启动成功，我们切换游戏角色为DD，让它抓住机械臂

![抓住机械臂](/img/apk_reverse/abi_capture_lift.jpg  "抓住机械臂")

通过机械臂，我们成功进入下一个关卡

![付费关的下一关](/img/apk_reverse/abi_next_stage_after_pay.jpg "付费关的下一关") 

# 结语

本文描述的过程是正向的，但是实际上我们利用常识可以跳过一些步骤，因为我们有一些常识，但是我准备写本文时必须要有一个完整的逻辑链才去了寻找整个正向的调用链条，比如众所周知微信和支付宝是国内最大的支付平台，我们可以直接找到这两者的依赖代码，并进行修改让其返回正确的结果就可以了

最后再强调一下，为他人的知识付费是完全正当合理的，不要窃取他人的知识进行牟利