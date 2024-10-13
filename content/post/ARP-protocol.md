---
title: "ARP Protocol"
description: 
categories:
  - "C_C++"
tags:
  - "ARP"
  - "Protocol"
date: 2024-05-09T00:28:32-07:00
slug: "ARP-protocol"
math: 
license: 
hidden: false
draft: false
---

# 什么是ARP？

> ARP是**A**ddress **R**esolution **P**rotocol的首字母缩写，即**地址解析协议**。

如果一台主机要将一个帧发送到另一台主机，只知道这台主机的IP地址是不够的，还需要知道主机的硬件地址。

> 对于以太网而言，硬件地址即48位的MAC地址。

对于采用以太网的TCP/IP网络，ARP协议提供从IPv4地址到MAC地址的动态映射。

> 动态是指它会自动执行和随时间变化，而不需要系统管理员重新配置。比如一台主机因更换网卡改变了MAC地址，ARP在一定延时之后继续正常运作。

**ARP不能跨网络使用！**

> ARP仅用于IPv4，IPv6使用ICMPv6中的邻居发现协议实现类似功能。

# ARP帧的格式

> 注：这里所说的ARP帧实际上是指ARP消息封装成的以太网帧。

## 以太网帧的格式

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/ARP-protocol/structure-of-ethernet-frame.png)

上图描述了链路层以太网帧的基本格式及各个字段的大小：

- DST：填写目的MAC地址，占6字节。
- SRC：填写源MAC地址，占6字节；
- 长度或类型：用于确定数据部分（上层协议PDU）来自哪种协议；
- 上层协议PDU：以太网帧的有效载荷部分，存放上层协议发来的PDU；
- FCS：帧校验序列，提供了对帧完整性的检查。

> 1. DST：可以填写广播地址或组播地址，广播功能用于ARP协议，组播功能用于ICMPv6，分别实现IPv4地址和IPv6地址到MAC地址的映射；
> 2. 长度或类型：TCP/IP网络常见值包括IPv4（0x0800）、IPv6（0x86DD）、ARP（0x0806）。

以太网帧有最小和最大尺寸的规定，除有效载荷（上层协议PDU）部分，其他四部分固定占有18字节：

- 最小帧为64字节：这要求上层协议PDU最小为46字节，如果不够就在有效载荷尾部填充0，以保证达到最小帧要求；
- 最大帧为1518字节：这要求上层协议PDU最大为1500字节，对于IP分组，如果超过1500字节，则需要进行IP分片。

> 有效载荷部分的最大长度，即1500字节，也被称为以太网的MTU（最大传输单元）。

## ARP消息的格式

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/ARP-protocol/structure-of-ARP-message.png)

上图描述了ARP消息的格式及各个字段的大小：

- 硬件类型：指出硬件地址类型，占2字节。对于以太网，该值为1；
- 协议类型：指出要映射的协议地址类型，占2字节。对于IPv4地址，该值为0x0800；
- 硬件大小：指出硬件地址的字节数，占1字节。对于以太网MAC地址，该值为6；
- 协议大小：指出要映射的协议地址的字节数，占1字节。对于IPv4地址，该值为4；
- Op：指出该ARP消息是ARP请求（该值为1）还是ARP应答（该值为2），占2字节；
- 源硬件地址：对于以太网，就是填发送方的MAC地址，占6字节；
- 源协议地址：对于IPv4网络，就是填发送方的IPv4地址，占4字节；
- 目的硬件地址：对于ARP请求，设为0；对于ARP应答，填接收方的MAC地址；占6字节；
- 目的协议地址：对于IPv4网络，就是填接收方的IPv4地址，占4字节。

> 对于一个ARP请求，它的任务是寻找目的协议地址（已知）对应的目的硬件地址（未知），所以除了目的硬件地址设为0，其他字段均需填写；
>
> 当所请求的系统接收到ARP请求，它填充自己的硬件地址，将两个源地址和两个目的地址互换，将Op字段设置为2，然后发送生成的应答。

## ARP帧的格式

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/ARP-protocol/ARP-frame-structure.png)

> ARP帧中存在重复信息：以太网头部和ARP消息中均包含**发送方的MAC地址**。

# ARP缓存

ARP高效运行的关键是每个主机和路由器上的**ARP缓存**。

在Linux下，我们可以使用arp命令查看本机的ARP缓存。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/ARP-protocol/arp-command.png)

> Flags字段的C表示该条目是由ARP协议动态学习而来；若为M表示是手工输入的；若为P表示“发布”。
>
> 从上图看，我的云服务器上只有其默认网关的ARP缓存。

当系统接收到发送给它的ARP请求时，除了发送ARP应答，它还会在其ARP缓存中保存请求者的IP地址和MAC地址。

ARP缓存是有超时时间的，通常，完整条目的超时为20分钟，不完整条目的超时为3分钟。并且通常在每次使用一个条目后为它重新启动20分钟的超时。

> 对一个不存在的主机进行ARP请求，就会在ARP缓存中生成一个**不完整条目**。

# ARP如何工作？

下面通过一个实际场景描述ARP的工作流程。

场景：主机A要给同一子网内的主机B发送消息，我们假设主机A和主机B的ARP缓存中中均没有对方的IP地址和MAC地址映射信息。主机A和主机B的IP地址和MAC地址如下。

> 主机A的IP地址：10.0.0.56；
>
> 主机A的MAC地址：00:00:c0:6f:2d:40；
>
> 主机B的IP地址：10.0.0.3；
>
> 主机B的MAC地址：00:00:c0:c2:9b:26。

## ARP请求

主机A发现自己不知道主机B的MAC地址，无法封装以太网帧，所以发出如下ARP请求帧：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/ARP-protocol/ARP-asking.png)

这个ARP请求帧相当于主机A在子网内**广播**了这样一条消息：IP地址为10.0.0.3的主机，请把你的MAC地址告诉我！

由于是广播，所以子网内的所有主机的以太网接口都可以接收到该ARP请求帧，IP地址不是10.0.0.3的主机将主动丢弃该帧。

## ARP应答

而主机B接收到ARP请求后，发现自己的IP地址与请求中的IP地址一致，所以它做出反应：发送ARP应答帧：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/ARP-protocol/ARP-response.png)

同时，主机B还会把主机A的IP地址及对应的MAC地址添加到自己的ARP缓存中。

主机A收到ARP应答帧后，就有了主机B的MAC地址，加上之前已知的主机B的IP地址，也一同添加到自己的ARP缓存中。

> 这就是一个完整的ARP请求/应答流程。有了主机B的MAC，主机A就可以给主机B发送消息了。

# 代理ARP

代理ARP的原理就是当出现跨网段的ARP请求时，路由器将自己的MAC地址返回给发送ARP请求的发送者，实现MAC地址代理（善意的欺骗），最终使得主机能够通信。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/ARP-protocol/proxy-ARP.png)

# 免费ARP

免费ARP的特殊之处在于，ARP请求中的源IP地址和目的IP地址均为请求发送者的IP地址，即：主机发送ARP请求寻找自己的MAC地址。

免费ARP有两个用处：

- 检测IP冲突。如果这个ARP请求收到了应答，说明请求者的IP地址已经被其他主机用了。
- 更新自己的MAC地址。如果主机的MAC地址变了（如更换了网卡），而IP地址没变。则可以发送一个免费ARP，告诉其他主机：我换MAC地址了，你们注意下。收到免费ARP的其他主机，就可以在自己的ARP缓存里找IP地址与免费ARP里的IP地址一致的条目，然后把这个IP地址对应的MAC地址更新为免费ARP里指明的新MAC地址。

如上述主机A检测自己的IP地址是否冲突，可发送免费ARP如下：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/ARP-protocol/free-ARP.png)

# ACD

ACD是IPv4地址冲突检测的简称，用途顾名思义，就是检测IP地址是否冲突的，一般在通过DHCP获取IP地址后，DHCP客户机通过ACD技术检测分配的IP地址是否冲突。

> ACD技术是通过ARP协议来完成的。

ACD定义了ARP探测消息和ARP通告消息。

## ARP探测消息

ARP探测消息是一个特殊的ARP请求，特殊在其源IP地址字段被设置为0。ARP探测分组用于查看IP地址是否被其他主机占用。

还是以上述的主机A为例，ARP探测消息如下：

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/ARP-protocol/ARP-detect.png)

> ARP探测消息的源IP地址设置为0，是为了避免候选IP地址（10.0.0.56）被另一台主机使用时的缓存污染。

当主机A发送自己的ARP探测时，它可能接收到ARP请求或应答：

- 若收到针对它自己ARP探测的应答，说明IP地址10.0.0.56被其他主机占用了；
- 若收到ARP请求（免费ARP或ARP探测），请求中的目的IP地址字段也是候选IP地址10.0.0.56，说明有其他主机也正在尝试获得该IP地址。

以上两种情况，主机A都会生成一个地址冲突消息，如果是DHCP分配的该IP地址，则发送DHCPDECLINE消息拒绝该IP地址。

## ARP通告消息

如果通过ARP探测消息发现，10.0.0.56没有被其他主机占用，即没有出现IP地址冲突，则主机A会间隔2秒向子网广播发送2个ARP通告消息，以表明它现在占用这个IP地址（10.0.0.56）。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/ARP-protocol/ARP-announcement.png)

> 可以看到ARP通告消息其实跟免费ARP没有区别。

ACD是一个持续的过程，这是它与免费ARP的区别。当主机A通告了它正在使用IP地址10.0.0.56后，它会继续接收ARP请求/应答消息，查看自己的IP地址（10.0.0.56）是否出现在这些ARP请求/应答消息的源IP地址字段中。如果是的话，说明其他主机正在与自己使用相同的IP地址。对于这种情况，有三种解决方案：

- 停止使用该地址；
- 保留该地址，但发送一个防御性的ARP通告，如果冲突继续，则停止使用该地址；
- 不理会冲突，继续使用。
