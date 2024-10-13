---
title: "DNS Protocol"
description: 
categories:
  - "C_C++"
tags:
  - "DNS"
  - "Protocol"
date: 2021-05-06T00:44:05-07:00
slug: "DNS-protocol"
math: 
license: 
hidden: false
draft: false
---

# 什么是DNS？

> DNS是**D**omain **N**ame **S**ystem的首字母缩写，即**域名系统**。

网络上的主机有两种标识方法：

- 域名：如www.baidu.com。优点是人们喜欢，容易记；缺点是机器不喜欢，路由器无法处理。
- IP地址：如39.156.69.79。优点是机器喜欢，容易处理；缺点是人们不喜欢，不好记。

为了折中人类和机器不同的偏好，我们需要一种能**从域名转换到IP地址**的服务，这就是DNS的主要任务。

DNS协议采用客户端/服务器模型，DNS协议位于五层网络模型中的应用层，在进行域名解析时其传输层采用UDP协议，其知名端口号为53。

> DNS协议由两部分组成：
>
> - 域名解析，用于执行对DNS特定名称查询的查询/响应协议；
> - 区域传输：用于交换数据库记录的协议。
>
> 对于域名解析，由于数据量小，DNS协议采用UDP协议，知名端口号为53；
>
> 对于区域传输，由于数据量大，DNS协议采用TCP协议，知名端口号为53。

# DNS服务器层次结构

DNS使用了大量的DNS服务器，它们以**层次方式**组织并且分布在全世界范围内。不存在一台DNS服务器拥有因特网上所有域名的映射。

大致说来，有3种类型的DNS服务器：

- 根DNS服务器：根DNS服务器提供顶级域名DNS服务器的IP地址；
- 顶级域名DNS服务器：顶级域名DNS服务器提供权威DNS服务器的IP地址；
- 权威DNS服务器：权威DNS服务器提供该组织的可访问域名的IP地址。

> 比如百度在因特网上有很多可以访问的域名，如百度首页baidu.com、百度贴吧tieba.baidu.com、百度新闻news.baidu.com，百度的权威服务器负责提供这些域名的IP地址。百度可以自己建权威DNS服务器，也可以花钱让中国电信这样的ISP帮它维护权威DNS服务器。

这三种类型的DNS服务器以下图的层次结构组织起来。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/DNS-protocol/hier-structure.png)

还有一种DNS服务器，叫**本地DNS服务器**，不在上述的DNS服务器层次结构中。本地DNS服务器就是用DHCP获得的配置信息中的DNS服务器。本地DNS服务器通常离我们的主机比较近，通常不超过几个路由器的距离。

> 当主机发岀DNS请求时，该请求被发往本地DNS服务器，它起着代理的作用，并将该请求转发到DNS服务器层次构中，后边将详细讨论。

# DNS缓存

为了改善时延性并减少在因特网上到处传输的DNS报文数，DNS广泛使用了**缓存技术**。

在一个请求链中，当某DNS服务器接收一个DNS回答 (例如，包含某域名到IP地址的映射)时，它能将映射缓存在本地存储器中。这样下次如果有对同样域名的DNS查询到达该DNS服务器时，它可以从自己的缓存中找出该域名对应的IP地址，直接返回给请求者，而不用再去问其他DNS服务器。

> 由于域名与IP的映射不是永久的，所以DNS缓存有一个生存时间（TTL）。

# DNS资源记录类型

虽然DNS通常用来将域名转换为对应的IP地址，但是它也有其他功能，区分这些功能的方法就是使用不同的**资源记录类型**。这里我们只关注以下四种资源记录类型：

- A：提供域名到IPv4地址的映射；
- AAAA：提供域名到IPv6地址的映射；
- NS：用来指定该域名由哪个DNS服务器来进行解析；
- CNAME：允许多个域名映射到同一IP地址。

> 对于CNAME，我记得，当时用GitHub建个人博客时用到过，默认的地址xgx127.github.io用CNAME映射到我买的域名xushark.com，然后用A记录把xushark.com映射到我的云服务器的IP地址。

# DNS工作流程

下面以一个简单例子来描述DNS的工作流程。本例中我们假设没有DNS缓存。

假设主机cse.nyu.edu想知道主机gaia.cs.umass.edu的IP地址。

1. 主机cse.nyu.edu首先向它的本地DNS服务器dns. nyu. edu发送一个DNS查询报文。 DNS查询报文内封装有要查询的域名gaia.cs.umass.edu；

2. 由于假定无缓存，本地DNS服务器将查询报文转发到根DNS服务器；

3. 根DNS服务器注意到其edu前缀，向本地DNS服务器返回负责edu的顶级域名DNS服务器的IP地址列表；

4. 本地DNS服务器则再次向这些顶级域名DNS服务器之一发送查询报文；

5. 顶级域名DNS服务器（edu）注意到umass.edu前缀，并返回权威DNS服务器的IP地址进行响应，该权威DNS服务器是负责马

   萨诸塞大学的dns.umass.edu；

6. 本地DNS服务器向dns.umass.edu发送查询报文；

7. 权威DNS服务器（dns.umass.edu）用gaia. cs. umass. edu的IP地址进行响应；

8. 本地DNS服务器将gaia.cs.umass.edu的IP地址反馈给主机cse.nyu.edu。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/DNS-protocol/DNS-server-interaction.png)

> 理论上讲，任何DNS查询既可以是迭代的也可以是递归的；
>
> 实践上，通常遵循上图的模式，请求主机到本地DNS服务器的查询是递归的，其余的查询是迭代的。

# DNS消息格式

基本的DNS消息以固定的12字节头部开始，其后跟随4个可变长度的区段：问题（或查询）、回答、授权记录和额外信息。

![](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/DNS-protocol/DNS-message.png)

> - 事务ID：由客户端设置，由服务器返回。客户端用它来匹配响应与查询；
> - QR：0表示查询消息，1表示响应消息；
> - OpCode：0表示标准查询（查询和响应），4表示通知，5表示更新，其他值弃用；
> - AA：表示授权回答（与缓存回答相对）；
> - TC：表示“可截断的”。使用UDP时，它表示当应答的总长度超过512字节时，只返回前512个字节；
> - RD：表示“期望递归”。在查询中设置，在响应中返回。它告诉服务器执行递归查询；
> - RA：表示“递归可用”。如果服务器支持递归查询，则在响应中设置该字段；
> - Z：目前Z字段必须为0，为将来而保留；
> - AD：若包含的信息是已授权的，则AD置1；
> - CD：若禁用安全检查，则CD置1；
> - RCODE：返回码字段，常见值如0（NoError）、3（NXDomain）；
> - QDCOUNT、ANCOUNT、NSCOUNT、ARCOUNT：说明了组成DNS消息的问题、回答、授权和额外信息区段中的条目数目。对于查询消息，QDCOUNT通常为1，其他三个为0；对于应答消息，ANCOUNT至少为1；
