---
title: Python访问samba
date: 2024-11-16 16:03:01
tags:
---
1987年，微软公司和英特尔公司共同制定了SMB（server Messages Block）协议用来解决局域网内的文件或打印机等的资源共享问题。但是这时后还是解决不了跨系统之间的文件共享。直到1991年，在读大学的Tridgwell 基于SMB协议开发能够解决Linux系统和windows系统之间的文件的问题——也就是SMBServer服务测序。后来被命名为samba（根据一个拉丁舞名字）。如今，samba服务测序成为了在Linux和windows系统之间共享文件的最佳选择，在工作中常使用Linux/UNIX服务器，而开发人员的终端可能是任意操作系统，所以在公司中常用samba进行文件共享；在生活中我们也可以用samba搭建家庭NAS，进行家庭内部的文件共享。
<!-- more -->

# Samba搭建

Linux和Windows都可以开启Samba服务，作为服务端向局域网内的其他主机提供文件共享服务


## Linux开启Samba
我们可以通过如下命令安装samba
```shell
sudo apt update
sudo apt install samba
```
编辑samba配置文件
``` shell
sudo vim /etc/samba/smb.conf
```
配置文件内容为
```conf
[global]
   workgroup = WORKGROUP
   server string = Samba Server
   security = user      # 指定 Samba 使用用户级别安全性，要求每个连接都有指定的用户名和密码
   map to guest = never
   min protocol = NT1  # 指定SMB版本
   max protocol = NT1  # 指定SMB版本

[shared]   # 添加共享文件夹的配置
   path = /srv/samba/shared
   browseable = yes
   writable = yes
   valid users = sambauser  # 限制只有 sambauser 用户可以访问共享
   create mask = 0755
```

创建共享文件夹，并指定用户和权限
```shell
sudo mkdir -p /srv/samba/shared
sudo chown nobody:nogroup /srv/samba/shared
sudo chmod 0777 /srv/samba/shared

# 创建系统用户（如果尚未创建）
sudo useradd -M -s /sbin/nologin sambauser

# 设置 Samba 用户密码
sudo smbpasswd -a sambauser
```

重启samba并放行端口
```shell
# 重启samba
sudo systemctl restart smbd
# 防火墙放行
sudo ufw allow Samba
```

到这里你可以通过IP地址访问Samba，如果你想通过主机名称访问Samba，则需要配置NetBios

```shell
sudo systemctl start nmbd
sudo systemctl enable nmbd
```
增加配置文件内容
```shell
[global]
netbios name = debian  # 将此处的名称更改为你的主机名
```
## Linux挂载Samba
我们可以通过以下步骤将一个Samba服务器挂载到本地文件系统，将Samba作为一个本地硬盘：
```shell
# 安装cifs-utils
sudo apt-get install cifs-utils
# 创建文件夹作为挂载点
mkdir /mnt/share
# 挂载，输入以下命令后回车会要求输入密码
mount.cifs //IP地址或者NetBios name/共享文件夹名 /mnt/share -o username=sambauser
```
如需开机自动挂载，请在/etc/fstab文件中添加
```shell
//IP地址或者NetBios name/share(远程共享目录)   /mnt/share(本地那个目录)    cifs(文件系统类型)  defaults,uid=1000,gid=1000,username=登陆用户名, passwd=登陆密码      0   0 
```
## Windows开启SMB
通过开始菜单搜索并打开控制面板

![开始菜单](/img/samba/controll_pannel.png "开始菜单")

从控制面板进入程序与功能

![程序与功能](/img/samba/controll_pannel_sel.png "程序与功能")

从程序与功能进入开启或关闭Windows功能

![开启或关闭Windows功能](/img/samba/enable_disable_win_function.png "开启或关闭Windows功能")

找到SMB选项，全部勾选，点击确认

![勾选SMB功能](/img/samba/SMB_enable.png "勾选SMB功能")

最后重启Windows系统，重启后即可通过该电脑的IP地址访问到SMB服务了
## Windows访问Samba
### 文件管理器直接访问
直接在文件管理器的地址栏输入Samba服务器IP和目录，回车访问，如果是初次访问则会弹出用户名和密码输入框，输入用户名和密码后勾选记住凭据即可

![文件管理器访问samba](/img/samba/samba_access1.png "文件管理器访问samba")

### 挂载samba为本地磁盘
在此电脑页面选择计算机选项卡后选择映射网络驱动器，在弹出的菜单中选择盘符并输入Samba链接，点击确认后Windows会将Samba映射到网络位置栏

![网络位置映射](/img/samba/network_location_map.png "网络位置映射")

### 通过浏览器访问Samba
在浏览器地址栏输入Samba链接即可看见共享文件列表，可以在这里点击文件进行下载，点击txt文件则会直接在浏览器中预览

![浏览器访问](/img/samba/browser_access.png "浏览器访问")

# Python的SMB库
Python中有很多SMB依赖库，目前主流的有如下几个
## pysmb
安装
```shell
pip install pysmb
```
在程序中导入smb，由于pysmb安装后在它的文件夹下有很多个文件，我们只需要引入我们需要的SMBConnection，该类继承自SMB
```python
from smb.SMBConnection import *
```
构建一个SMBConnection对象
```python
client = SMBConnection(username='用户名', password='密码', my_name="TEST", remote_name='FW_SERVER', is_direct_tcp=True)
```
username和password无需多言，my_name是你本机的NetBios name，remote_name是SMB服务器的NetBios name，my_name可以在自己的计算机上查看，但是remote_name我们可能并不知道，也不可能接触到SMB服务器，我们只能通过IP地址访问SMB服务，这时其实my_name和remote_name其实都不重要，你可以传入两个空字符串，只需要将is_direct_tcp设置成True即可

启动SMB连接
```python
client.connect(ip='SMB服务器IP地址', port=445)
```
当我们使用IP地址直接访问SMB服务时，需要将访问端口设为445，如果使用NetBios name访问则使用139端口

获取SMB的服务列表和文件列表
```python
services = client.listShares() # SharedDevice对象列表/服务列表
files = client.listPath(service_name='服务名称', path='服务名称之后的文件夹索引/') # SharedFile对象列表
```
在实际情况下，我们直接在资源管理器里面打开IP地址（没有后续索引层级），就会看见该服务器上所有的服务列表，类似于下图，每个服务的用户名和密码可能是不同的，请确认你的用户名和密码正确以及拥有对应目录的访问权限

![服务列表示例](/img/samba/service_list.png "服务列表示例")

从服务列表中双击进去，地址栏的IP地址后会跟上服务的名称，这就是listPath方法的service_name参数，再选择服务中的文件夹，这就是listPath方法的path参数，path参数可以索引多个层级

当我们获取到SharedDevice对象列表后可以访问SharedDevice对象的name属性获取到服务名称，SharedFile对象中的filename属性是文件的文件名，注意这里获取的SharedFile对象列表中包含文件和文件夹，文件夹中包含本文件夹./和上层文件夹../

文件下载
```python
for i in files:
    if((i.file_attributes & ATTR_DIRECTORY) != ATTR_DIRECTORY): # 筛选文件而非文件夹
        f = open(f'本地文件夹路径/{i.filename}', 'wb') # 打开一个本地文件用于接收将要下载的文件
        client.retrieveFile(service_name='服务名称', path=f'服务名称之后的文件夹索引/{i.filename}', file_obj=f) # 下载文件，传入服务名称，文件路径和本地文件描述符
        f.close() # 接收完毕关闭本地文件
        print(i.filename)
```

首先我们迭代了刚才获取到的SharedFile对象列表，通过SharedFile对象的file_attributes，判断ATTR_DIRECTORY指示位是否被置起，该指示位为1时表示该对象对应的是一个文件夹，指示位为0则表示该对象对应的是一个文件，判断SharedFile对象是文件的话，就在本地打开一个文件，使用retrieveFile方法下载文件

文件上传
```python
f = open(f'本地文件路径/本地文件名称', 'rb')
client.storeFile(service_name='服务名称', path='服务名称之后的文件夹索引/上传到服务器的文件名称', file_obj=f)
f.close()
```

首先在打开本地要上传的文件，使用storeFile方法上传，可以将上传到服务器的文件名称指定为与本地不一致

关闭SMB连接
```python
client.close()
```
## smbprotocol
安装smbprotocol
```shell
pip install smbprotocol
```
安装完毕后有两个库可供调用：
* smbprotocol: 能完成任何工作的低级接口，但是会有些繁琐
* smbclient: 内建支持SMB的os和os.path文件系统函数的高级接口

由于底层协议比较复杂，所以我们只介绍smnclient

引入smbclient
```python
import smbclient
```
连接到SMB服务器
```python
server_address = 'your_server_ip'
username = 'your_username'
password = 'your_password'
smbclient.register_session(server_address, username=username, password=password)
```
设置共享目录和文件路径，share_name就是前面我们提到的共享服务名称
```python
share_name = 'shared_folder'
local_file_path = 'local_example.txt'
remote_file_path = 'remote_example.txt'
```
上传文件，由于smbclient内建了os和os.path文件系统函数，所以我们可以直接open SMB服务器上的数据，然后进行读写，直接按照二进制读写数据，从而将数据上传到服务器或者将服务器上的文件下载下来
```python
with open(local_file_path, 'rb') as local_file:
    with smbclient.open_file(r"\\{server_address}\{share_name}\{remote_file_path}", mode='wb') as remote_file:
        remote_file.write(local_file.read())
```
下载文件，与上传文件类似
```python
with smbclient.open_file(r"\\{server_address}\{share_name}\{remote_file_path}", mode='rb') as remote_file:
    with open(local_file_path, 'wb') as local_file:
        local_file.write(remote_file.read())
```


# 曲线救国

当我们将samba服务器挂载为本地磁盘后，我们就可以使用本地文件操作方法对samba服务器中的文件进行操作，操作系统会帮我们处理samba的协议（用户只需要操作文件就好，操作系统要考虑的就很多了&#128517;）

# 结语
Samba是一个非常有用的服务，在工作中可以进行本地和服务器的数据共享，在家里可以用于构建NAS，甚至可以直接作为网络公开文件共享服务，进行文件发布。使用Python操作Samba有助于提高我们对Samba的管理效率，节约我们的宝贵时间。