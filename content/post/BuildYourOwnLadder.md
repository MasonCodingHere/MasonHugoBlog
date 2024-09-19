---
title: "私人代理——搭建专属于你的梯子"
description: 
categories:
    - OverTheWall
tags:
    - VPS
    - XRay
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
# 切换至新用户leo
su leo
# 在新用户leo家目录下创建.ssh文件夹
mkdir ~/.ssh
# 修改.ssh文件夹权限
chmod 700 ~/.ssh
# 上传公钥。将引号内的内容替换为公钥值
echo "公钥内容" >> ~/.ssh/authorized_keys
# 修改authorized_keys文件权限
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

## 安装Fail2Ban

```bash
# 安装Fail2Ban
sudo apt update && sudo apt install fail2ban
# 建配置文件
sudo nano /etc/fail2ban/jail.local
# 将以下内容粘贴在打开文件中
******粘贴内容开始******
[sshd]
enabled = true
port = 9753
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
bantime = 7200
findtime = 600
******粘贴内容结束******
# 启动Fail2Ban
sudo systemctl start fail2ban
```

> 上述配置的意思是：600秒内5次尝试登陆ssh的IP地址会被判7200秒有期徒刑。

`Fail2Ban`常用命令:

```bash
# 重启Fail2Ban
sudo systemctl restart fail2ban
# 查看fail2ban服务状态
sudo systemctl status fail2ban
# 将Fail2Ban设为开机启动
sudo systemctl enable fail2ban
# 查看状态
sudo fail2ban-client status
# 查看sshd jail的详细状态
sudo fail2ban-client status sshd
# 解禁指定IP
sudo fail2ban-client set sshd unbanip 192.0.0.1
```

至此，VPS安全配置已经完成。接下开是真正的搭梯子环节。

# XRay安装与配置

不推荐用任何一键脚本，谁知道里边有没有藏什么东西。官方文档足够详细，安装配置足够简单。此部分详细内容可查阅[官方文档](https://xtls.github.io/document/)。

## Install Xray

GitHub仓库[Xray-install](https://github.com/XTLS/Xray-install)是[Xray官方](https://github.com/XTLS)提供的安装方式，在仓库`README`页可以找到官方安装脚本：

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

> 后期更新新版本也是这个命令。

一键安装，足够简单吧。

## Config Xray

1. 生成一个合法的 `UUID` 并保存备用（`UUID`可以简单粗暴的理解为像指纹一样几乎不会重复的 ID）。

   ```bash
   xray uuid
   ```

   ![uuidgenerate](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/uuidgennerate.png)

   这个`uuid`要留存，后边配置会用到。

2. 修改Xray配置文件。

   ```bash
   sudo nano /usr/local/etc/xray/config.json
   ```

   将以下内容粘贴进去。只需修改`port`和`id`两项。本文`port`以`12345`为例，`id`以上一步生成的为例。

   ```json
   # 原始版本，VMess+Kcp，访问不了ChatGPT客户端
   {
     "log": {
       "loglevel": "warning",
       "access": "/var/log/xray/access.log", // 这是 Linux 的路径
       "error": "/var/log/xray/error.log"
     },
     "inbounds": [
       {
         "port": 12345, // 服务器监听端口
         "protocol": "vmess",    // 主传入协议
         "settings": {
           "clients": [
             {
               "id": "0061c282-49f7-41d7-b223-0a9b6d8675dd",  // uuid，客户端与服务器必须相同
               "alterId": 0
             }
           ]
         },
         "streamSettings": {
           "network": "kcp", //此处的 kcp 也可写成 mkcp，两种写法是起同样的效果
           "kcpSettings": {
             "uplinkCapacity": 15,
             "downlinkCapacity": 100,
             "congestion": true,
             "readBufferSize": 1,
             "writeBufferSize": 1,
             "header": {
               "type": "wireguard"
             }
           }
         }
       }
     ],
     "outbounds": [
       {
         "protocol": "freedom",  // 主传出协议
         "settings": {}
       }
     ]
   }
   ```

3. 开放在配置文件中指定的端口号。

   ```bash
   sudo ufw allow 12345/tcp comment "XRay"
   sudo ufw allow 12345/udp comment "XRay"
   ```

   ![openport12345](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/openport12345.png)

4. 启动Xray

   ```bash
   # 启动XRay
   sudo systemctl start xray
   # 将XRay设为开机启动
   sudo systemctl enable xray
   # 查看XRay运行状态
   sudo systemctl status xray
   ```

   看到下图这样绿色的`active(running)`就说明XRay已成功启动，这样服务端就配置好了。

   ![startxray](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/startxray.png)

5. 开启BBR加速。

   给 Debian 10 添加官方 `backports` 源，获取更新的软件库。

   ```bash
   sudo nano /etc/apt/sources.list
   ```

   然后把下面这一条加在最后，并保存退出。

   ```text
   deb http://archive.debian.org/debian buster-backports main
   ```

   刷新软件库并查询 Debian 官方的最新版内核并安装。请务必安装你的 VPS 对应的版本（本文以比较常见的【amd64】为例）。

   ```bash
   sudo apt update && sudo apt -t buster-backports install linux-image-amd64
   ```

   修改 `kernel` 参数配置文件 `sysctl.conf` 并指定开启 `BBR`

   ```bash
   sudo nano /etc/sysctl.conf
   ```

   把下面的内容添加进去

   ```text
   net.core.default_qdisc=fq
   net.ipv4.tcp_congestion_control=bbr
   ```

   重启 VPS、使内核更新和`BBR`设置都生效。

   ```bash
   sudo reboot
   ```

   确认`BBR`开启。

   ```bash
   lsmod | grep bbr
   ```

   此时应该返回`tcp_bbr`这样的结果。

   如果你想确认 `fq` 算法是否正确开启，可以使用下面的命令：

   ```bash
   lsmod | grep fq
   ```

   此时应该返回`sch_fq`这样的结果。

6. 配置客户端。

   服务端配置结束，接下来就是在你的客户端进行配置。根据客户端用的代理工具不同，配置方法也不太一样。但全都是根据服务端的配置进行。主要是把`IP:Port`和`uuid`配置为与服务端一致。

   > 客户端配置就不细讲了。

   这样，就已经可以科学上网了。只是无法使用ChatGPT客户端，因为ChatGPT对这些VPS厂商的IP进行了封锁。

# Warp解锁ChatGPT

> ChatGPT封锁了VPS厂商的IP地址，而Cloudflare的Warp可以提供供我们使用的IP地址，所以我们用Warp提供的IP地址去访问ChatGPT。但是，Warp提供的IPv4地址已经被滥用，也无法访问ChatGPT客户端了，所以我们用Warp IPv6。目标是访问寻常网站时用VPS的公网IPv4地址，访问ChatGPT时用Warp的IPv6地址。

执行下面这个脚本安装WGCF。

```bash
wget -N https://gitlab.com/fscarmen/warp/-/raw/main/menu.sh && bash menu.sh
```

根据需要选择语言，本文选中文了。

![chooselanguage](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/choose.png)

因为我们这台VPS是IPv4 only的机器，没有IPv6地址，我们仅需要Warp提供IPv6地址，所以选择2. 为IPv4 only添加Warp IPv6网络接口。

![addv6forv4only](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/add.png)

工作模式选1.全局（默认）即可。

![workmode](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/workmode.png)

选择账户类型。这里根据自己的需要选择，我自己有WARP的团队版，即Zero Trust，所以选择3. Teams。

![accountchoose](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/loginaccount.png)

接下来就是登陆选择的账户。Teams账户选择2.通过组织名和邮箱验证码来登陆是比较方便的。

![login](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/logincf.png)

优先级别选择3.使用VPS初始设置（默认）即可。

![46](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/choose46priority.png)

这样WGCF会开始安装，等待安装完成，运行下面的命令，可以看到有一个叫`warp`的网卡。

```bash
sudo ifconfig
```

![warpcard](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/warpinter.png)

这样VPS就有IPv6地址了。

修改XRay的配置文件

```bash
sudo nano /usr/local/etc/xray/config.json
```

用下面的内容覆盖。将`IPv4`地址替换为你的VPS公网IPv4地址，将`IPv6`地址替换为warp网卡的地址。`uuid`和`port`也要替换为自己的。其他的不用更改。

```json
# Warp接管IPv6流量版本，VMess+Kcp，解锁ChatGPT客户端
{
  "log": {
    "loglevel": "warning",
    "access": "/var/log/xray/access.log", // 这是 Linux 的路径
    "error": "/var/log/xray/error.log"
  },
  "inbounds": [
    {
      "port": 12345, // 服务器监听端口
      "protocol": "vmess",    // 主传入协议
      "settings": {
        "clients": [
          {
            "id": "0061c282-49f7-41d7-b223-0a9b6d8675dd",  // uuID，客户端与服务器必须相同
            "alterId": 0
          }
        ]
      },
      "streamSettings": {
        "network": "kcp", //此处的 kcp 也可写成 mkcp，两种写法是起同样的效果
        "kcpSettings": {
          "uplinkCapacity": 15,
          "downlinkCapacity": 100,
          "congestion": true,
          "readBufferSize": 1,
          "writeBufferSize": 1,
          "header": {
            "type": "wireguard"
          }
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",  // 主传出协议
      "settings": {
        "domainStrategy": "UseIPv4"
      },
      "sendThrough": "100.200.300.400" //VPS公网IPv4地址
    },
    {
      "tag": "warp",
      "protocol": "freedom",
      "settings": {
        "domainStrategy": "UseIPv6"
      },
      "sendThrough": "2606:4700:110:88b8:5141:4387:4a3:20d1" //warp网卡IPv6地址
    }
  ],
  "routing": {
    "rules": [
      {
        "domain": [
          "geosite:openai",
          "geosite:bing",
          "geosite:netflix"
        ],
        "outboundTag": "warp",
        "type": "field"
      }
    ]
  }
}
```

重启XRay，并确认Xray已成功运行。

```bash
sudo systemctl start xray
sudo systemctl status xray
```

在客户端连接该节点，可以发现ChatGPT客户端已经可以正常使用。

# References

1. [Project X官方文档](https://xtls.github.io/document/)
2. [保护好你的小鸡！保姆级服务器安全教程！](https://blog.laoda.de/archives/how-to-secure-a-linux-server)
3. [vps 解锁 chatgpt](https://cn.v2ex.com/t/942922)
4. [注册Cloudflare并加入ZeroTrust教程](https://surge.tel/13/2116/)
