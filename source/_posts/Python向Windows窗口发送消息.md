---
title: Python向Windows窗口发送消息
date: 2025-01-04 19:27:13
tags:
---
最近有一个小需求，需要将文件复制到剪贴板，然后粘贴到目标文件夹中，在python中需要直接调用Windows的API，还没有直接实现相关功能的轮子，所以在这里记录一下过程中的一些尝试，本文中参考了网上相关资料，主要有以下两篇：

[【python】复制文件到剪贴板](https://juejin.cn/post/7123461124162322445)

[Python win32api.keybd_event模拟键盘输入](https://blog.csdn.net/lihuarongaini/article/details/101298063)
<!-- more -->

# 安装win32

使用如下命令安装win32依赖，理论上我们可以通过这个依赖访问Windows的所有API

```shell
pip install win32
```

在之前Python获取BLR广播的文章中我介绍过一些Windows上BLE相关的API，感兴趣的可以看一看

# 将文件复制到剪贴板

直接上代码：

```python
import os
from glob import glob
import ctypes
import time
import win32clipboard
import win32con
import win32gui
import win32api

class DROPFILES(ctypes.Structure):
    _fields_ = [
        ("pFiles", ctypes.c_uint),
        ("x", ctypes.c_long),
        ("y", ctypes.c_long),
        ("fNC", ctypes.c_int),
        ("fWide", ctypes.c_bool),
    ]

pDropFiles = DROPFILES()
pDropFiles.pFiles = ctypes.sizeof(DROPFILES)
pDropFiles.fWide = True
meta = bytes(pDropFiles)

def clipboard_copy(file):
    data = file.encode("U16")[2:] + b"\0\0" 

    win32clipboard.OpenClipboard()  #打开剪贴板（独占）
    try:
        win32clipboard.EmptyClipboard() #清空当前的剪贴板信息
        win32clipboard.SetClipboardData(win32clipboard.CF_HDROP,meta+data) #设置当前剪贴板数据
        print("复制成功！")
    except Exception as e:
        print(str(e))
    finally:
        win32clipboard.CloseClipboard() #无论什么情况，都关闭剪贴板
```
首先我们使用Python类构建了一个C语言的结构体[DROPFILES](https://learn.microsoft.com/en-us/windows/win32/api/shlobj_core/ns-shlobj_core-dropfiles)，这个结构体是传递给剪贴板数据的一部分，我们将其pFiles字段设为该结构体的大小，实际上也指明了文件路径的开始偏移，把fWide设为True表示我们文件路径使用的是Unicode编码而不是ANSI编码。x和y是另一个结构体[POINT](https://learn.microsoft.com/en-us/windows/win32/api/windef/ns-windef-point)，不过这里我们直接将其展开了，fNC则是用来指示x和y的范围，fNC为True表示非clint区域，False则是在client区域。将结构体转为字节编码以便后续使用

然后我们对传入的路径进行编码，编码类型为UTF-16，并且去掉编码后的开头两个Byte，再在末尾加上两个空字符，具体原因如下：

* 要将字符串中的字符转为UCS-2格式，平时我们使用较多的是UTF-8其存储一个字符只有1个字节，UCS-2是2个字节
* UTF-16就是2个字节的编码，有三种类型：UTF-16，UTF-16BE（大端序），UTF-16LE（小端序），其中UTF-16 通过以名称BOM（字节顺序标记，U + FEFF）启动文件来指示该文件仍然是小端序
* 进行UTF-16编码之后去掉头部的BOM标记即可得到可用的编码
* Windows规定使用2个空字符对路径元组结尾，我们只有一个路径所以直接以2个空字符结尾
* 如果有多个路径，那么将每个路径用1个空字符进行分隔，最后用2个空字符进行结尾即可

多个路径列表的处理方式如下：

```python
    filepaths_list = ['path0', 'path1', 'pathn']
    files = ("\0".join(filepaths_list))
    data = files.encode("U16")[2:] + b"\0\0" 
```

接着调用OpenClipboard()方法打开剪贴板，后续的操作在异常处理中进行，防止发生意外导致剪贴板无法关闭，我们在finally里面关闭剪贴板

由于我们要填数据到剪贴板，所以要先清空剪贴板，如果你想从剪贴板读取数据，则不用清空，直接读取即可。清空之后我们使用[SetClipboardData()](https://learn.microsoft.com/zh-cn/windows/win32/api/winuser/nf-winuser-setclipboarddata)方法，第一个参数是剪贴板格式，可以参阅[剪贴板格式](https://learn.microsoft.com/zh-cn/windows/win32/dataxchg/clipboard-formats)和，我们传入的是标准剪贴板格式的**CF_HDROP**，表示类型**HDROP**的句柄，用于标识文件列表，第二个参数就是我们前面处理好的结构体和UTF-16编码的文件路径

到此，我们就将文件复制到剪贴板中了，当然不是真正的文件，而是文件的绝对路径，这里实际上是我们完成了手动按Ctrl+c过程

# 向目的窗口发送Ctrl+v消息

由于本次的目标路径是同一个，并且由于特殊原因不能通过代码copy文件实现，所以我考虑了两个方案，一个是通过剪贴板+Ctrl+v消息，另一个是通过[WM_DROPFILES消息](https://learn.microsoft.com/zh-cn/windows/win32/shell/wm-dropfiles)，但是发送WM_DROPFILES消息方案没有成功，所以这里只介绍第一个方案

首先我们要定位我们的目标窗口

![目标窗口](/img/python_wm/dest_window.png "目标窗口")

我们可以使用Spy++ Lite进行定位，请自行下载软件，使用管理员身份运行，使用鼠标拖动界面中的指针定位到目标窗口

![Spy++ Lite](/img/python_wm/spylite.png "Spy++ Lite")

从Spy++ Lite窗口中获取窗口类名和标题文本两个值，用于获取窗口句柄

```python
hwnd = win32gui.FindWindow("CabinetWClass", "test")
win32gui.SetForegroundWindow(hwnd)
while not os.path.exists(dest_file_path):
    win32api.keybd_event(win32con.VK_CONTROL, 0, 0, 0)
    win32api.keybd_event(0x56, 0, 0, 0)
    win32api.keybd_event(0x56, 0, win32con.KEYEVENTF_KEYUP, 0)
    win32api.keybd_event(win32con.VK_CONTROL, 0, win32con.KEYEVENTF_KEYUP, 0)
    time.sleep(1)
```

首先我们通过FindWindow获取窗口句柄，传入刚才我们通过Spy++ Lite获取到的两个值，当然如果你没有Spy++ Lite并且确定你的窗口标题唯一的时候，可以只填窗口标题，窗口类名填None即可

获取到窗口句柄之后我们调用SetForegroundWindow()方法将窗口激活，随后进入一个循环，通过检查目标文件是否复制到位来决定是否继续循环。为什么有这个循环呢？是因为我发现有些时候我们只发送一次消息的话有时不能达到目的，所以增加了一个循环进行持续触发，至于原因是什么，这个恐怕得问微软

我们调用[keybd_event()](https://learn.microsoft.com/zh-cn/windows/win32/api/winuser/nf-winuser-keybd_event)方法，第一个参数是我们要提交的按键编码，第三个参数是按键事件，首先我们按下Ctrl，然后按下v(0x56)，再依次释放两个按键

实际上keybd_event()方法已经被[sendInput()](https://learn.microsoft.com/zh-cn/windows/win32/api/winuser/nf-winuser-sendinput)方法取代，但是sendInput方法更复杂一些，所以依旧使用keybd_event方法

# 结语

Windows的API是相当复杂的一个体系，我们能做的只是在我们需要使用的时候进行相应的查询和使用，并且API中存在相当一部分的遗留API，所以我们需要特别注意使用API的使用，尽量寻找一些高级的封装