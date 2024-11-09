---
title: Python获取BLE广播
date: 2024-11-06 21:10:28
tags:
---
蓝牙协议从1999年发布第一版以来已经二十几个年头，也随着移动通信的发展普及到我们生活的方方面面，并且在可见的未来将会随着物联网和边缘计算渗透到我们肉眼可见不可见的各个角落。在蓝牙4.0中，[蓝牙技术联盟（Bluetooth SIG）](https://www.bluetooth.com/zh-cn/)引入了BLE（Bluetooth Low Energy）协议，该协议以极低的功耗优势进入物联网设备领域。本篇就一起看一下，如何在Windows使用Python读取周围BLE设备发出的广播（advertisement）。
<!--more-->
# nRF Connect
如果你调试过BLE设备，那么nRF Connect软件你应该并不陌生，它是由[NORDIC SEMICONDUCTOR](https://www.nordicsemi.com/)公司开发的一款跨平台BLE连接性测试应用。他们也设计各种射频相关的Soc，你可以购买他们的评估或者开发套件用于测试与开发。

通过移动版的nRF Connect应用可以轻松的对附近的蓝牙设备进行扫描，并且不依赖于任何外部硬件设备。如果你使用的是Android设备，你可以前往[Google Play](https://play.google.com/store/apps/details?id=no.nordicsemi.android.mcp&hl=en&gl=US)或者[Github](https://github.com/NordicSemiconductor/Android-nRF-Connect/releases)下载相应的安装包文件进行安装，如果你使用的是苹果设备请前往[App Store](https://apps.apple.com/gb/app/nrf-connect-for-mobile/id1054362403)下载应用。

nRF Connect应用启动页面如下，软件需要打开定位和蓝牙方可开始扫描

![nRF Connect首页](/img/python_ble/nRF_connect_startup.jpg "nRF Connect首页")

我们打开定位和蓝牙就可以看到右上角的SCAN按钮，然后点击按钮即可对周围的蓝牙设备进行扫描

![nRF Connect扫描设备](/img/python_ble/nRF_connect_scan.jpg "nRF Connect扫描设备")

点击设备列表中的设备可以展开查看设备的广播信息

![nRF Connect广播信息](/img/python_ble/nRF_connect_advertisement.jpg "nRF Connect广播信息")

点击信息的RAW按钮，即可查看该设备广播的RAW数据是什么

![nRF Connect广播RAW](/img/python_ble/nRF_connect_advertisement_raw.jpg "nRF Connect广播RAW")

Raw data解析之后就是Details表格中的内容，实际上raw data的组织是很简单的，raw data是按照多个固定顺序排列的数据结构构成的：<br>
* 第1字节是type和value的字节数量
* 第2字节是type，是蓝牙技术联盟指定的内容类型
* 后续字节是value，即数据载荷

重复以上数据结构，即组成raw data，我们也可以根据规则对raw data进行解析，这个我们后续会用到。

nRF Connect的功能非常强大，不过今天不准备再介绍它了。nRF Connect虽然功能强大，不过目前较为普遍的使用都是手机端的，桌面端的需要一个他们公司的适配器才能进行周围的设备扫描，另外一个缺点是手机端应用需要开启定位，虽然我也不知道它为什么要定位，不过对于需要定位的应用我总是觉得不是很放心。所以接下来我们开始介绍Python的蓝牙库，最终实现在Windows上直接获取BLE的RAW广播。

# Python的蓝牙库
如果你希望使用Python实现某个功能，那么你要先寻找一个相应的库，然后在代码中引入这个库。接下来我们看一下各位大神都给我们造好了哪些轮子吧。

## pybluez
pybluez支持Windows和Linux下的经典蓝牙，仅在Linux下支持BLE。由于不同操作系统对于硬件的驱动方式不同，所以一个处于高层的库需要更底层的库依赖，或者对底层的库进行封装。你可以前往[PyBluez官方文档](https://pybluez.readthedocs.io/en/latest/install.html)查看如何进行安装。

pybluez的安装相当麻烦，在windows上需要安装[Microsoft Visual C++ 14.0](https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk)，这个其实我难以理解，为什么某些操作系统上层的应用会以操作系统的应用开发工具为依赖，这种依赖方式在你安装了各种不同的应用之后会发现你的程序管理器中多出一大堆不同版本的Microsoft Visual C++，有些应用甚至直接让你安装一个Microsoft Visual Studio开发环境，如果你遇到不同应用之间的依赖版本冲突则堪称噩梦，因为此时你虽然只想把当前这个应用安装好，但是你必须弄清楚以前安装的应用的依赖分别是哪些，以及会不会有版本冲突。

![不同版本的Microsoft Visual C++依赖](/img/python_ble/microsoft_cpp_dependency.png "不同版本的Microsoft Visual C++依赖")

安装完成后我们可以使用如下代码进行测试

```python
#server端
from bluetooth import *

server_sock=BluetoothSocket( RFCOMM )
server_sock.bind(("",PORT_ANY))
server_sock.listen(1)

port = server_sock.getsockname()[1]

uuid = "94f39d29-7d6d-437d-973b-fba39e49d4ee"

advertise_service( server_sock, "SampleServer",
                   service_id = uuid,
                   service_classes = [ uuid, SERIAL_PORT_CLASS ],
                   profiles = [ SERIAL_PORT_PROFILE ], 
#                   protocols = [ OBEX_UUID ] 
                    )
                   
print("Waiting for connection on RFCOMM channel %d" % port)

client_sock, client_info = server_sock.accept()
print("Accepted connection from ", client_info)

try:
    while True:
        data = client_sock.recv(1024)
        if len(data) == 0: break
        print("received [%s]" % data)
except IOError:
    pass

print("disconnected")

client_sock.close()
server_sock.close()
```

```python
# client端
from bluetooth import *
import sys

if sys.version < '3':
    input = raw_input

addr = None

if len(sys.argv) < 2:
    print("no device specified.  Searching all nearby bluetooth devices for")
    print("the SampleServer service")
else:
    addr = sys.argv[1]
    print("Searching for SampleServer on %s" % addr)

# search for the SampleServer service
uuid = "94f39d29-7d6d-437d-973b-fba39e49d4ee"
service_matches = find_service( uuid = uuid, address = addr )

if len(service_matches) == 0:
    print("couldn't find the SampleServer service =(")
    sys.exit(0)

first_match = service_matches[0]
port = first_match["port"]
name = first_match["name"]
host = first_match["host"]

print("connecting to \"%s\" on %s" % (name, host))

# Create the client socket
sock=BluetoothSocket( RFCOMM )
sock.connect((host, port))

print("connected.  type stuff")
while True:
    data = input()
    if len(data) == 0: break
    sock.send(data)

sock.close()
```

## bluepy
仅支持Linux的BLE蓝牙库，不需要底层操作系统依赖，直接通过pip安装即可：
```shell
sudo pip install bluepy
```
[官方文档](https://ianharvey.github.io/bluepy-doc/)中有一些DEMO可以参考，也可以参考[bluepy_examples_nRF51822_mbed](https://github.com/rlangoy/bluepy_examples_nRF51822_mbed)

有一点需要注意，在Linux下使用bluepy要使用sudo前缀，由于bluepy只支持Linux，所以先不做介绍。

## bleak
来到本篇的重点，bleak是一个异步的BLE库，支持Windows和Linux，也不依赖任何Windows开发环境，直接依赖于Windows提供的Runtime环境。

直接通过pip安装

```shell
pip install bleak
```

直接扫描周围的BLE设备
```python
import asyncio
from bleak import BleakScanner

async def main():
    devices = await BleakScanner.discover()
    for d in devices:
        print(d)

asyncio.run(main())
```
使用client类连接设备进行操作
```python
import asyncio
from bleak import BleakClient

address = "24:71:89:cc:09:05"
MODEL_NBR_UUID = "2A24"

async def main(address):
    async with BleakClient(address) as client:
        model_number = await client.read_gatt_char(MODEL_NBR_UUID)
        print("Model Number: {0}".format("".join(map(chr, model_number))))

asyncio.run(main(address))
```
如果你想获取一些广播中的信息，你可以在discover方法中传入一个参数：
```python
    devices = await scanner.discover(return_adv=True)
```
将return_adv参数设为True之后，discover将会返回一个Dictionary，其key是蓝牙设备的地址字符串，其value是一个Tuple，该Tuple包含两个对象，第0个对象是BLEDevice，第1个对象是AdvertisementData，也许你觉得到此我们就可以获取到地址对应的广播数据了（advertisement），但是bleak的作者告诉我，并不能。

我们可以先看一下python安装目录\Lib\site-packages目录中的bleak的AdvertisementData类

![AdvertisementData类](/img/python_ble/bleak_AdvertisementData_classs.png "AdvertisementData类")

观察类的内部属性，对比蓝牙技术联盟提供的[Assigned Numbers文档2.3节](https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/Assigned_Numbers/out/en/Assigned_Numbers.pdf?v=1731073793951)，发现该类中的属性是将部分重要的Assigned Numbers对应的数据。我们还可以在目录中找到一个assigned_numbers.py文件，这里面就是bleak可以解析的Assigned Numbers

![bleak支持解析的Assigned Numbers](/img/python_ble/bleak_valid_assigned_numbers.png "bleak支持解析的Assigned Numbers")

可以发现，bleak支持解析的Assigned Numbers其实很少，如果我们直接通过AdvertisementData类获取advertisement，就可能会出现bleak不支持的类型无法获取的问题，如下是目前较为完整的Assigned Numbers列表
```python
adv_type_list = {0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x08,0x09,0x0A
                ,0x0D,0x0E,0x0F,0x10,0x11,0x12,0x14,0x15,0x16,0x17
                ,0x18,0x19,0x1A,0x1B,0x1C,0x1D,0x1E,0x1F,0x20,0x21
                ,0x22,0x23,0x24,0x25,0x26,0x27,0x28,0x29,0x2A,0x2B
                ,0x2C,0x2D,0x2E,0x2F,0x30,0x31,0x32,0x34,0x3D,0xFF}
```
那么我们如何才能获取到完整的广播呢，不如直接到Github上提个issue问一下吧，看看作者如何解答：[bleak/issues/1678](https://github.com/hbldh/bleak/issues/1678)

![bleak作者如是说](/img/python_ble/bleak_get_adv_issue.png "bleak作者如是说")

非常遗憾，bleak并不提供这样的接口，不如来看看bleak是如何解析Windows返回的advertisement的吧。打开bleak\backends\winrt\scanner.py文件，找到BleakScannerWinRT类中的_received_handler方法

```python
    def _received_handler(
        self,
        sender: BluetoothLEAdvertisementWatcher,
        event_args: BluetoothLEAdvertisementReceivedEventArgs,
    ):
        """Callback for AdvertisementWatcher.Received"""
        # TODO: Cannot check for if sender == self.watcher in winrt?
        logger.debug("Received %s.", _format_event_args(event_args))

        # REVISIT: if scanning filters with BluetoothSignalStrengthFilter.OutOfRangeTimeout
        # are in place, an RSSI of -127 means that the device has gone out of range and should
        # be removed from the list of seen devices instead of processing the advertisement data.
        # https://learn.microsoft.com/en-us/uwp/api/windows.devices.bluetooth.bluetoothsignalstrengthfilter.outofrangetimeout

        bdaddr = _format_bdaddr(event_args.bluetooth_address)

        # Unlike other platforms, Windows does not combine advertising data for
        # us (regular advertisement + scan response) so we have to do it manually.

        # get the previous advertising data/scan response pair or start a new one
        raw_data = self._advertisement_pairs.get(bdaddr, _RawAdvData(None, None))

        # update the advertising data depending on the advertising data type
        if event_args.advertisement_type == BluetoothLEAdvertisementType.SCAN_RESPONSE:
            raw_data = _RawAdvData(raw_data.adv, event_args)
        else:
            raw_data = _RawAdvData(event_args, raw_data.scan)

        self._advertisement_pairs[bdaddr] = raw_data

        uuids = []
        mfg_data = {}
        service_data = {}
        local_name = None
        tx_power = None

        for args in filter(lambda d: d is not None, raw_data):
            for u in args.advertisement.service_uuids:
                uuids.append(str(u))

            for m in args.advertisement.manufacturer_data:
                mfg_data[m.company_id] = bytes(m.data)

            # local name is empty string rather than None if not present
            if args.advertisement.local_name:
                local_name = args.advertisement.local_name

            try:
                if args.transmit_power_level_in_d_bm is not None:
                    tx_power = args.transmit_power_level_in_d_bm
            except AttributeError:
                # the transmit_power_level_in_d_bm property was introduce in
                # Windows build 19041 so we have a fallback for older versions
                for section in args.advertisement.get_sections_by_type(
                    AdvertisementDataType.TX_POWER_LEVEL
                ):
                    tx_power = bytes(section.data)[0]

            # Decode service data
            for section in args.advertisement.get_sections_by_type(
                AdvertisementDataType.SERVICE_DATA_UUID16
            ):
                data = bytes(section.data)
                service_data[normalize_uuid_str(f"{data[1]:02x}{data[0]:02x}")] = data[
                    2:
                ]
            for section in args.advertisement.get_sections_by_type(
                AdvertisementDataType.SERVICE_DATA_UUID32
            ):
                data = bytes(section.data)
                service_data[
                    normalize_uuid_str(
                        f"{data[3]:02x}{data[2]:02x}{data[1]:02x}{data[0]:02x}"
                    )
                ] = data[4:]
            for section in args.advertisement.get_sections_by_type(
                AdvertisementDataType.SERVICE_DATA_UUID128
            ):
                data = bytes(section.data)
                service_data[str(UUID(bytes=bytes(data[15::-1])))] = data[16:]

        # Use the BLEDevice to populate all the fields for the advertisement data to return
        advertisement_data = AdvertisementData(
            local_name=local_name,
            manufacturer_data=mfg_data,
            service_data=service_data,
            service_uuids=uuids,
            tx_power=tx_power,
            rssi=event_args.raw_signal_strength_in_d_bm,
            platform_data=(sender, raw_data),
        )

        device = self.create_or_update_device(
            bdaddr, local_name, raw_data, advertisement_data
        )

        self.call_detection_callbacks(device, advertisement_data)
```

观察_received_handler方法的定义发现它有3个参数，self是当前调用该方法的对象，剩下两个分别是[BluetoothLEAdvertisementWatcher](https://learn.microsoft.com/zh-cn/uwp/api/windows.devices.bluetooth.advertisement.bluetoothleadvertisementwatcher?view=winrt-26100)对象和[BluetoothLEAdvertisementReceivedEventArgs](https://learn.microsoft.com/zh-cn/uwp/api/windows.devices.bluetooth.advertisement.bluetoothleadvertisementreceivedeventargs?view=winrt-26100)对象。除了self之外，另外两个参数都是Windows Runtime环境提供的对象，表明该函数是直接与Windows系统交互的。

在代码第15行，定义了bdaddr变量，从BluetoothLEAdvertisementReceivedEventArgs对象中取得了BLE设备地址并进行了格式化。再看第17、18行作者写的注释：不像其他的平台，Windows不会组织advertisement信息，所以我们需要自己处理。这行注释表明该方法将会处理Windows系统返回的advertisement信息。第21行到第29行尝试获取对应bdaddr地址的BLE advertisement raw，并更新本对象的_advertisement_pairs属性中bdaddr键对应的值。第31到35行创建了一些临时变量，可以发现这些变量都是Assigned Numbers对应的数据，这些变量在第83行用于创建了一个我们前面提到的AdvertisementData对象（discover方法的部分返回值）。接下来是一个嵌套的for循环，在循环中获取了BluetoothLEAdvertisementReceivedEventArgs对象内部的advertisement属性，该属性是[BluetoothLEAdvertisement类](https://learn.microsoft.com/zh-cn/uwp/api/windows.devices.bluetooth.advertisement.bluetoothleadvertisement?view=winrt-26100)对象，其内部构造如下

![BluetoothLEAdvertisement类](/img/python_ble/BluetoothLEAdvertisement_class.png "BluetoothLEAdvertisement类")

bleak在嵌套的for循环中获取了BluetoothLEAdvertisement类内部的service_uuids，manufacturer_data，local_name属性，又通过get_sections_by_type方法获取了tx_power（发射功率），service_data，最终将填充好的变量和raw_data一起构造了一个AdvertisementData对象，随后在93行调用了create_or_update_device方法，更新了self对象属性，最后调用用户传入的回调函数

到这里我们可以发现，当我们在discover方法中传入return_adv=True参数后我们其实可以拿到AdvertisementData对象，而AdvertisementData对象中platform_data属性就是在刚才_received_handler方法中传入的，platform_data是一个Tuple，其第[1]个元素就是raw_data，raw_data是一个_RawAdvData对象，_RawAdvData对象中有adv和scan属性

```python
class _RawAdvData(NamedTuple):
    """
    Platform-specific advertisement data.

    Windows does not combine advertising data with type SCAN_RSP with other
    advertising data like other platforms, so se have to do it ourselves.
    """

    adv: BluetoothLEAdvertisementReceivedEventArgs
    """
    The advertisement data received from the BluetoothLEAdvertisementWatcher.Received event.
    """
    scan: Optional[BluetoothLEAdvertisementReceivedEventArgs]
    """
    The scan response for the same device as *adv*.
    """
```
adv是BluetoothLEAdvertisementReceivedEventArgs对象，具有advertisement属性，advertisement属性是BluetoothLEAdvertisement对象，具有[DataSections属性](https://learn.microsoft.com/zh-cn/uwp/api/windows.devices.bluetooth.advertisement.bluetoothleadvertisement.datasections?view=winrt-26100#windows-devices-bluetooth-advertisement-bluetoothleadvertisement-datasections)，这个属性在winrt转成python对象之后变成data_sections，该属性包含了BLE设备advertisement的所有字段，我们可以直接解析这个属性，它是一个包含[BluetoothLEAdvertisementDataSection类](https://learn.microsoft.com/zh-cn/uwp/api/windows.devices.bluetooth.advertisement.bluetoothleadvertisementdatasection?view=winrt-26100)对象的列表，BluetoothLEAdvertisementDataSection类中有两个属性data和data_type，其中data_type就是Assigned Number，data则是Assigned Number对应的Byte内容

到此我们通过一个超长的对象引用链，最终拿到了我们想要的advertisement数据

## bleak获取advertisement示例
```python
import asyncio
import bleak
from bleak import BleakScanner
import binascii

async def ble_scan():
    scanner = BleakScanner()
    devices = await scanner.discover(return_adv=True)

    for d in devices:
        print(f"mac addr: {devices[d][0].address}, name: {devices[d][0].name}")
        for args in filter(lambda d: d is not None, devices[d][1].platform_data[1]):
            sec_set = []
            for j in args.advertisement.data_sections:
                j_set = [int(j.data_type), bytes(j.data)]
                flag = False
                for s in sec_set:
                    flag = (s == j_set)
                if (flag == False):
                    sec_set.append(j_set)
            for sec in sec_set:
                print(f"Type: {'{:02X}'.format(sec[0])}\tPayload: {''.join(['%02X' % b for b in sec[1]])}")

asyncio.run(ble_scan())
```
```shell
# output
mac addr: 81:26:00:00:07:03, name: LuYuan-Smart
Type: 01        Payload: 06
Type: 19        Payload: C003
Type: 03        Payload: 1218
Type: FF        Payload: 812600000703
Type: 0A        Payload: 00
Type: 09        Payload: 4C755975616E2D536D617274
mac addr: 5A:53:C9:75:58:C7, name: None
Type: 01        Payload: 1A
Type: 0A        Payload: 0C
Type: FF        Payload: 4C0010060D1D2229AB38    
mac addr: 4D:69:42:97:E4:E4, name: None
Type: 01        Payload: 1A
Type: 0A        Payload: 0C
Type: FF        Payload: 4C001005051C237DA3
mac addr: D7:C4:BA:B5:3F:6C, name: None
Type: FF        Payload: 4C0012020002
mac addr: 66:84:61:36:6D:6F, name: None
Type: 01        Payload: 1A
Type: 0A        Payload: 0B
Type: FF        Payload: 4C001005711C7D9A6A
mac addr: D0:62:2C:CC:F4:0E, name: None
Type: 01        Payload: 06
Type: 19        Payload: C200
Type: 16        Payload: 95FE1059282E000EF4CC2C62D0
Type: 16        Payload: ABFD0103
```
在for循环中输出了data_sections的内容，我们也可以直接将数据重新组装为raw。

# 结论
通过如上的分析，我们从bleak到Windows Runtime环境，最终获取到我们想要的数据。到此，也许如何获取advertisement RAW已经不再重要，重要的是我们了解到一条可以让python通向Windows操作系统底层的路径，该条路径以后可以帮助我们直接操作Windows的API，用以自己构建相应的python库。

在这里也祝贺川建国同志赢得大选，继续为世界人民带来乐子😂