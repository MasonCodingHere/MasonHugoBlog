---
title: "私人代理——搭建专属于你的梯子"
description: 
categories:
    - OverTheWall
tags:
    - VPS
date: 2024-09-18T16:26:03+08:00
image: https://xtls.github.io/LogoX2.png
math: 
license: 
hidden: false
comments: false
draft: false
---

# 前言

大家都知道，像Google、Twitter这类的海外网站在中国大陆是无法直接访问的。大家还知道，连上VPN就可以访问这些网站了。至于其中的细节，大多数人就不知道了，确实也没必要知道。但是有一点我们要明白，VPN能够起到效果的重要原因是GFW没有封锁一切海外网站。正是由于这个原因，我们才能有空子可钻。

> 技术上讲，VPN≠翻墙。不过在大陆VPN已经成为翻墙的代名词，这里就不咬文嚼字了。

在互联网中，IP地址是我们的“身份证”。这些被禁的海外网站也有自己的IP地址，可惜被墙了，所以我们无法直接访问。还存在一些IP地址，没有被墙，我们可以访问。一系列的翻墙技术正是建立在这个基础上。

假设有这样一个IP地址A，A没有被GFW墙掉，并且A作为互联网的一员，可以访问那些被墙的网站。我们可以把要访问Google的请求发给A，由A帮我们访问，再把结果发给我们，这就是**代理**的原理，也是一系列翻墙技术得以实现的基础。

# 购买VPS

我们需要一个没有被墙的IP地址，购买一台海外VPS可以满足这个需求。VPS其实就是一台位于海外的云服务器。

有很多VPS厂商，比如BandWagon、Vultr、CloudCone等等，本文以Vultr为例，其他的厂商也大同小异。

注册并登陆[Vultr](https://www.vultr.com/)，点击右上角的Deploy按钮按以下步骤部署一台VPS。

## 选择VPS类型和位置

类型选Shared CPU即可。

位置看自己需求，这里以美国Los Angeles为例。

![select_type_and_location](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/select_type_and_location.png)

## 选择系统镜像

选一个操作系统，服务器嘛，自然是要选Linux，推荐选择Debian，比较稳定。

![choose_image](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/choose_image.png)

## 选择Plan

根据需求，选择一个配置，配置越高越贵。这里选择最便宜的5$/month的Plan。

![choose_plan](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/choose_plan.png)

## 其他设置

去掉Auto Backups，费钱。

去掉IPv6，不需要。

![additional_features](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/additional_feature.png)

这里可以把SSH key上传一下，把本地电脑`~/.ssh/`的**公钥**传到Vultr，这样等部署完可以直接密钥登陆，不用输密码，方便很多。

在Server Hostname设置一下主机名，自己起个名即可，本文以Test为例。

![server_settings](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/server_setting.png)

## 下单

这样就配好了，确认一下数量和价格，点击Deploy Now就开始部署了。

> 部署过程或许需要几分钟，耐心等待一下。

![order](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/order.png)

## 部署完成

Status变成Running就部署完成了，打码处为这台VPS的公网IP地址，下文以`100.200.300.400`来表示。

![deploy](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/deploy.png)

## 检查IP是否被墙

在[站长工具-Ping检测](https://ping.chinaz.com/)输入你的IP地址，点击Ping检测，如果大部分是绿的，说明IP没被墙，可以放心使用。

如果全是红色，说明此IP地址已被墙，无法使用，根据你的VPS厂商政策，想办法更换IP，直至可用。

![pingcheck](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/pingcheck.png)



# 配置VPS

## SSH登陆VPS

要配置VPS，首先通过SSH登陆VPS。打开你的终端工具，输入以下命令登陆VPS。

```bash
ssh root@100.200.300.400
```

> SSH登陆Linux服务器是基本操作，不细说了。

## 启用ufw防火墙

登陆后第一件事，就是启用ufw防火墙。首先确认ufw是否已经安装，执行下述命令，若返回ufw版本号，，则说明已经安装。

```bash
ufw --version
```

> 没有安装的自己安装一下。

执行下述命令设置ufw。

```bash
#设置ufw默认值
sudo ufw default deny incoming
sudo ufw default allow outgoing
#允许SSH连接
sudo ufw allow ssh
#允许http连接
sudo ufw allow http
#允许https连接
sudo ufw allow https
#开机启动
sudo ufw enable
```

ufw其他常用命令：

```bash
#查看ufw状态
sudo ufw status
#ufw状态带行号输出，删除某一行
sudo ufw status numbered
sudo ufw delete 5
#重载ufw配置
sudo ufw reload
```

## 更改SSH默认端口

1. 打开nano或vim等编辑器打开文件`/etc/ssh/sshd_config`

```bash
vim /etc/ssh/sshd_config
```

2. 在打开文件中找到`Port`这一项，去掉注释符`#`，并将22修改为新的端口号（1025-65535），本文以9753为例。

![changeport](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/changeport.png)

3. 重启ssh服务，使变更生效。

```bash
service sshd restart
```

4. 防火墙开放新端口号


```bash
ufw allow 9753/tcp comment "SSH"
```

5. 防火墙中删除原来的22端口，下图删除了`22/tcp`，同样的方法把`22/tcp(v6)`也删除掉。

![delete_port22](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/delete_port22.png)

6. 测试新端口号登陆

   > 注意：为了保证你不会失联，请不要关闭当前的ssh登录窗口！而是另外开一个窗口来测试！

   在新的终端窗口，用新的端口号登陆VPS。

   ```bash
   ssh root@100.200.300.400 -p 9753
   ```

## 设置仅密钥登陆

1. 把公钥上传到VPS。

```bash
# 在家目录下新建.ssh文件夹
mkdir ~/.ssh
# 修改.ssh文件夹权限
chmod 700 ~/.ssh
# 上传公钥。将引号内的内容替换为公钥值
echo "Your public key content" >> ~/.ssh/authorized_keys
# 修改authorized_keys文件权限
chmod 600 ~/.ssh/authorized_keys
```

2. 禁止密码登录

```bash
vim /etc/ssh/sshd_config
```

找到`PasswordAuthentication`这一项，解注释并改为`no`，然后保存并退出。

![banpasswdlogin](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/banpasswdlogin.png)

有些系统只改这一个没用，还要改下面这个。如果系统没有这个文件就忽略这一步。

```bash
vim /etc/ssh/sshd_config.d/50-cloud-init.conf
```

同样，将`PasswordAuthentication`这一项，解注释并改为`no`，然后保存并退出。

3. 重启ssh服务，使变更生效。

```bash
service sshd restart
```

4. 测试密钥登陆。

打开一个新的终端，用以下命令登陆VPS。

```bash
ssh -i path/to/yourprivatekeyname -p 9753 root@100.200.300.400
```

> 将`path/to/yourprivatekeyname`修改为VPS使用的公钥的对应的私钥路径。通常在`~/.ssh/`下。

## 新建普通用户

1. 执行以下命令，新建一个用户。根据提示完成即可。

```
adduser leo
```

> 给新用户起个名字，本文以`leo`为例。

2. 将新用户`leo`加入`sudo`。

```bash
visudo
```

![addsudo](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/addsudo.png)

3. 把公钥给新用户。

```bash
su leo
mkdir ~/.ssh
chmod 700 ~/.ssh
echo "公钥内容" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

这样新用户就可以用密钥登陆了。

## 禁止root登录

打开`/etc/ssh/sshd_config`，找到`PermitRootLogin`这一项，解注释并将`yes`改为`no`。

```bash
vim /etc/ssh/sshd_config
```

重启sshd服务，使变更生效。

```bash
sudo service sshd restart
```

这样，root用户就无法登陆了，以后用`leo`用户管理VPS。

## 安装Fail2ban

```bash
sudo apt update && sudo apt install fail2ban
cd /etc/fail2ban # 进入fail2ban目录
sudo cp fail2ban.conf fail2ban.local  # 复制一份配置文件 
nano fail2ban.local #打开配置文件，以下内容粘贴在最下方

******粘贴内容开始******
[sshd]
backend = systemd
enable = ture
port = 9753   # 注意改成自己对应的端口
filter =sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = -1 # 永久封禁
******粘贴内容结束******

$ sudo service fail2ban restart #重启
$ sudo fail2ban-client status #查看状态
$ sudo fail2ban-client status sshd #查看sshd的详细状态

$ sudo fail2ban-client set sshd unbanip 192.0.0.1 #解禁指定IP
```



# 安装XRay-core

# Warp解锁chatGPT
