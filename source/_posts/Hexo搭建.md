---
title: Hexo搭建
---

## 环境搭建

```
 * git
 * Node.js
 * hexo
```

Github是著名的开源，同性交友社区，为广大用户提供了一个网上代码、博客托管空间，大约有300M空间，当然我们需要先在[GitHub官网](https://github.com/)注册一个账户，在官网上就可以注册。当你注册好了，就可以开始搭建属于你自己的Hexo博客了。

### 安装git

__git__是一个版本控制工具，和__github__搭配使用可以实现强大的功能。

Ubuntu系统：
```
 sudo apt install git -y
```
Centos系统：
```
 sudo yum install git -y
```
### 安装Node.js

__Node.js__ 是一个基于Chrome JavaScript 运行时建立的一个平台。

__Node.js__是一个事件驱动I/O服务端JavaScript环境，基于Google的V8引擎，V8引擎执行Javascript的速度非常快，性能非常好。

Ubuntu系统：
```
 sudo apt install nodejs -y
```
Centos系统：
```
 sudo yum install nodejs -y
```
### 安装Hexo

首先新建并进入一个文件夹：
```
 mkdir 文件夹名

 cd 文件夹名
```
安装Hexo：
```
 sudo npm install -g hexo-cli
```
初始化Hexo：
```
 hexo init
```
到这里，我们的博客本地基本上就搭建完毕了，你新建的文件夹就是博客的根目录，这个文件夹非常重要。

现在我们去看看我们的博客页面:
```
 hexo g                          //生成静态页面

 hexo s                          //启动本地服务，进行文章预览调试
```
这两个命令以后会比较常用，执行完后，不会结束，会提示你到：http://localhost:4000 查看你的博客页面



## 配置Github

刚才我们在GitHub上注册的账户现在派上用场了，我们开始吧。

### 建立Repository

进入[Github官网](https://github.com/)登陆你的账户，点击右上角“+”号，选择New repository，第一项是仓库名，这个仓库将用于托管我们的博客静态页面，所以命名有讲究：
```
 你的用户名.github.io
```
然后点击Create repository即可。

回到我们的博客根目录，如果刚才__"hexo s"__命令没有退出请按__"Ctrl+c"__退出

编辑hexo配置文件：
```
 vim _config.yml
```
当然你也可以用其他的编辑器打开这个文件，修改文件的最后部分成如下：
```
 deploy:

 type: git

 repo: https://github.com/你的用户名/你的用户名.github.io.git

 branch: master
```
这样修改之后我们就将本地博客和我们的GitHub仓库关联在一起了，接下来的命令可以将我们的本地静态文件（前面：hexo g 生成的静态文件）推到GitHub仓库中，然后我们就可以通过：__https://你的用户名.github.io__ 来访问你的博客了。

推送静态文件
```
 hexo d
```
到此我们的博客就部署完毕了，关于其他的像主题、头像什么的，这里就先不说了。不过我想说一说域名的绑定。

## 域名的绑定

域名是为了帮助解决人们对于IP地址的记忆难题的一套机制，如[www.baidu.com](https://www.baidu.com)，[www.alibaba.com](https://www.alibaba.com)，[www.cqlh.ml](http://www.cqlh.ml)都是域名，而IP地址是一串点分十进制数，如14.255.177.39，106.11.223.101，185.199.108.153等，显然前面两者更好记忆，我们可以给我们的博客绑定一个属于我们自己的域名，方便大家访问。

域名也是一种互联网资源，通常域名都是需要购买的，而且国内的域名大部分都是要备案的，而备案就需要有服务器，而且必须要使用期有三个月的服务器才可以用来申请备案，所以作为学生，我们就要想尽办法白嫖啊。

### 科学上网

请使用Firefox浏览器，访问：[https://addons.mozilla.org](https://addons.mozilla.org)，在右上角搜索setup，找到__SetupVPN Lifetime Free VPN__，点击__安装__，然后这个插件将会出现在浏览器右上角，点击插件，然后注册一个免费帐户，然后选择一个节点连接。这里科学上网主要是为了访问下一步中的网站。

### 申请域名

前往：[www.freenom.com](https://www.freenom.com/zh/index.html?lang=zh) 

* 注册一个账户用来申请免费域名
* 在首页检测自己想要的域名，由于是买免费的，所以后缀名可能不太常见，当然这个网站也出售收费域名

免费域名的期限是一年，在域名到期前15天可申请免费续订，如果要继续使用需要进行续订

### 将DNS设置到国内

DNS是域名服务器，其实在网络中计算机使用的还是IP地址，DNS就是专门用来将域名翻译为IP地址的服务器，由于freenom是国外的的网站，所以其DNS在国外，在国内不能解析，所以我们要修改域名的DNS。

前往：[https://www.dnspod.cn](https://www.dnspod.cn/)，注册一个账户或者使用QQ或微信登陆，进入__域名解析-->添加域名__然后将域名写在下面的列表中，点击添加的域名，你将会看见两条记录，不要关闭该页面。

在freenom登陆账户后的首页，点击顶部的__Service-->My Domains__，将看见你的域名，点击对应域名的__Manage Domain-->Management Tools-->Nameservers-->Use custom nameservers (enter below)__。下面要求我们填写域名服务器，那么我们的域名服务器在哪呢？

返回到DNSPOD页面，分别复制两条记录中的__记录值__项，填到Nameserver1和Nameserver2下面，然后点击__Change Nameservers__即可。

这样我们就吧我们域名的DNS服务器改为国内的DNSPOD，在国内我们就可以访问这个域名了，当然还要绑定我们的博客才能访问到博客。这个域名是在国外网站申请的，所以不用在国内进行备案，这样就简单了不少，备案至少需要15天，还不一定能申请下来。

### 创建CNAME文件

回到博客根目录，进入source文件夹：
```
 cd source/
```
创建CNAME文件：
```
 touch CNAME
```
将域名写到CNAME文件中，写的时候记得写域名前面的www

生成静态文件并推上GitHub：
```
 hexo g

 hexo d
```
### GitHub网页绑定域名

在浏览器中访问GitHub官网并登录我们注册的账号，然后点击左边的repository中我们为博客建立的那个repository，点击在页面的中部右边的__Setting__，下滑找到__GitHub Pages__项，将你的域名填写在__Custom domain__中并点击__Save__。

### 添加域名解析记录

> * 通过__host__命令查看我们博客部署的IP地址：
> ```
   host 你的用户名.github.io
 ```
> * 回到DNSPOD __域名解析__ 中点击我们刚才添加的 __域名__
> * 点击__添加记录__，主机记录选择__“@”__，记录类型选择__“A”__，记录值填刚才host出来的IP地址，TTL设置为__600__（可变），然后保存。
> * 点击__添加记录__，主机记录选择__“www”__，记录类型选择__“A”__，记录值填刚才host出来的IP地址，TTL设置为__600__（可变），然后保存。



到此，我们搭建好我们的博客，推上了GitHub并绑定了我们自己的域名。Hexo博客需要使用Markdown编写，不过Markdown语法比较少，也比较简单，可以参考：[Markdown教程](https://www.runoob.com/markdown/md-tutorial.html)


