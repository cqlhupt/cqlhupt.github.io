---
title: Hexo博客迁移问题解决指南
date: 2024-11-03 09:45:40
tags:
---
每次长期搁置博客之后恢复博客都会遇到一些坑，最好的方式当然是督促自己稳定更新，但是这就是另外一个问题了😂，还是来看看博客恢复的坑吧。
<!--more-->
# 目录
* 可迁移Hexo
* 迁移到Windows
  * 配置npm源
  * 重新安装hexo和hexo-cli
  * hexo g失败
  * public文件夹
  * 配置git global
  * 配置github公私钥
  * git timeout fatal
  * icarus主题打开Latex支持
  * 直接从其他计算机复制博客文件夹
  * 密钥问题
* 结语

# 可迁移Hexo
当我们搭建完Hexo博客之后，会面临这样一个问题，如果搭建Hexo的本地环境出现了损坏，或者换了新的电脑，就需要将原来的Hexo迁移到新的平台，这是一个需要尽早考虑的问题，否则在积累一定的数据量之后迁移可能会造成数据的丢失。

可迁移的准备起始非常简单，由于在搭建Hexo时已经在github新建了一个用于托管博客的仓库，所以我们可以将我们博客的重要文件，例如主题、md文档，图片等资源都托管到Hexo仓库的另一个分支中，每次hexo  deploy之后把相关的资料也git push到这个新分支中，以后如果需要迁移，那么直接从github将这个仓库clone下来，切换到该分支，然后重新安装hexo相关module即可完成平台迁移。

创建新分支：
``` shell
git clone "Your Hexo github repository" # 克隆你的Hexo仓库
git checkout -b hexo                    # 创建并切换到名为hexo的分支，分支名称可以不同
```

将原来搭建hexo的目录内容复制到刚才创建的新分支目录下，并在.gitignore文件中填入以下内容
``` shell
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
_multiconfig.yml
package*
```
通过以下命令提交并推送到github
``` shell
git add .               # 添加所有非ignore文件到git管理
git commit -a           # 提交所有修改和添加的文件
git push origin hexo    # push新分支到github
```

# 迁移到Windows
其实不管是Windows还是Linux，都可以迁移，不过目前我们日用的电脑一般都是Windows，所以这里着重讲迁移到Windows

首先你至少需要在Windows上安装以下软件
* git
* node和npm

你可以访问他们的官网下载安装包进行安装
* https://git-scm.com/downloads/win
* https://npm.nodejs.cn/cli/v8/configuring-npm/install

如果你的网络不能正常访问，你也可以考虑通过清华源下载：
* https://mirrors.tuna.tsinghua.edu.cn/github-release/git-for-windows/git/
* https://mirrors.tuna.tsinghua.edu.cn/nodejs-release/

## 配置npm源
由于国内网络问题，可能对npm模块仓库访问很慢，甚至不能访问，所以需要配置npm国内镜像，以提高访问速度。这里我们可以选择不同的镜像，比如[清华源npm](https://mirrors.tuna.tsinghua.edu.cn/help/nodejs-release/)或者[npmmirror 镜像站](https://npmmirror.com/)，这里我选择了npmmirror 镜像站，执行如下指令

以下命令请在git bash中执行，打开git bash的目录是你clone下来的hexo仓库的hexo分支（或者你自己命名的新分支）
``` shell
npm install -g cnpm --registry=https://registry.npmmirror.com
```

执行完之后，后续我们可以用cnpm代替npm进行module的安装
## 重新安装hexo和hexo-cli
通过前一节的配置我们可以通过以下命令安装hexo和hexo-cli
``` shell
cnpm install hexo
cnpm install hexo-cli
```
## hexo g失败
理论上讲这时我们的迁移已经完成了，我们可以开始generate博客
``` shell
hexo g
```
此时你可能会遇到以下问题：

1. Script load failed: themes\icarus\scripts\index.js
   
   解决办法是先执行一次hexo clean再执行hexo g
2. 提示hexo版本不对
   
   该问题一般在错误信息中会给出安装正确版本的命令，比如：npm install hexo^6.0.0
## public文件夹
public文件夹是在hexo s时生成的临时文件存放文件夹，在长期搁置博客之后你可能会忘记这件事，然后根据public的结构把资源放在其中，但是这会导致你编写完下一个博客后生成静态资源查看时丢失所有资源。正确的资源路径是你的主题文件夹下的source文件夹
## 配置git global
当你编写好一个hexo博客之后想add和commit到本地仓库时，git会提示你配置global信息，以便确认你的身份，配置命令如下：
``` shell
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```
将双引号内的内容替换为你的github用户名和邮箱即可
## 配置github公私钥
当你希望进行hexo d或者将本地改动push到hexo分支（你的新分支）时，会出现以下错误：
``` shell
On branch master
nothing to commit, working tree clean
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
FATAL Something's wrong. Maybe you can find the solution here: https://hexo.io/docs/troubleshooting.html
```
这是由于没有配置github公私钥导致的，我们可以生成一个新的密钥对，把公钥上传到github，将私钥配置关联到本地仓库，然后就可以进行正常的deploy和push了

生成密钥对
``` shell
ssh-keygen –t rsa –C "密钥注释"
```
根据提示完成密钥的存储路径选择和密码输入，如果不需要密码可以直接回车两次

配置github公钥：
1. 点击右上角你的头像并选择Settings，进入github网页的设置页面
   ![进入设置页面](/img/github/setting_entrance.png "进入设置页面")
2. 点击SSH and GPG keys项再点击右侧的New SSH key，将你刚才生成的带_pub后缀的文件内容复制到表单中，并给这个公钥取一个名字
   ![添加新的密钥](/img/github/public_key_set.png "添加新的密钥")

配置本地仓库关联私钥
``` shell
# 在你的hexo分支目录下打开git bash
ssh-agent bash
ssh-add "你的私钥文件地址"
```
随后即可开始进行博客部署和资源的提交

## git timeout fatal

在执行git push/pull时出现报错信息：
```shell
ssh: connect to host github.com port 22: Connection refused
fatal: Could not read from remote repository.
​
Please make sure you have the correct access rights
and the repository exists.
```
原因：GitHub的22端口（ssh默认端口）被屏蔽

解决方案：修改访问端口为443（https默认端口）

在用户目录下的.ssh文件夹中找到config文件（没有的话请新建一个），配置如下内容
```shell
Host github.com
HostName ssh.github.com 
User git
Port 443
```
可以使用如下命令进行测试：

```shell
$ ssh -T git@github.com
Hi xxxxx! You've successfully authenticated, but GitHub does not
provide shell access.
$ 
```

## icarus主题打开Latex支持

在你的主题配置文件中增加如下配置：
```yml
plugins:
    mathjax: true
```
如果你使用的是主题默认配置文件，只需要在该默认文件中找到mathjax项，将其从false改成true即可

## 直接从其他计算机复制博客文件夹

当我们更换计算机或者重装系统后，想复制以前的博客备份文件夹过来直接使用时，可能会遇到如下问题：
```shell
admin@DESKTOP-TNI5ROV MINGW64 /d/git/rt-thread/rt-thread_pm2
$ git log
fatal: detected dubious ownership in repository at 'D:/git/rt-thread/rt-thread_pm2'
'D:/git/rt-thread/rt-thread_pm2' is owned by:
        'S-1-5-21-1045045257-1974506225-3199486363-500'
but the current user is:
        'S-1-5-21-1045045257-1974506225-3199486363-1001'
To add an exception for this directory, call:

        git config --global --add safe.directory D:/git/rt-thread/rt-thread_pm2
```

这是由于这个文件夹的所有者不是当前我们登录的用户，我们需要修改这个文件夹及其内部的所有文件/文件夹的所有者为当前我们登录的账户

![修改文件夹所有者1](/img/hexo_mig/change_owner_of_dictory1.png "修改文件夹所有者1")
![修改文件夹所有者2](/img/hexo_mig/change_owner_of_dictory2.png "修改文件夹所有者2")

根据以上两张图中的步骤修改你的博客文件夹所有者，注意第9步中的选框是在第8步结束后才会出现的，并且第9和第10步必须选上才能生效

## 密钥问题
当我们将文件夹复制到另一台曾经使用SSH访问过GitHub的电脑时可能会遇到以下问题：
```shell
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the RSA key sent by the remote host is
SHA256:uNiVztksCsDhcc0u9e8BujQXVUpKZIDTMczCvj3tD2s.
Please contact your system administrator.
Add correct host key in /c/Users/LH/.ssh/known_hosts to get rid of this message.
Offending RSA key in /c/Users/LH/.ssh/known_hosts:3
RSA host key for github.com has changed and you have requested strict checking.
Host key verification failed.
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
```
这是由于我们的known_hosts文件中记录存在问题，需要将其删除，然后重新尝试连接即可，使用如下命令删除：
```shell
ssh-keygen -R github.com
```

# 结语
保持更新确实是一件很困难的事情，尤其是更新这件事本身对于自己可有可无的时候，毕竟人类的本质是鸽子和复读机。希望以后能够按时更新，记录自己的成长和生活。