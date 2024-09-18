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

我们需要一个没有被墙的IP地址，购买一台海外VPS可以满足我们这个需求。VPS其实就是一台位于海外的云服务器。

以Vultr为例，注册并登陆Vultr。

## 选择VPS类型和位置

![select_type_and_location](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/select_type_and_location.png)

## 选择系统镜像

![choose_image](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/choose_image.png)

## 选择Plan

![choose_plan](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/choose_plan.png)

## 其他设置

去掉Auto Backups和IPv6。

![additional_features](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/additional_feature.png)

可以SSH key

![server_settings](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/server_setting.png)

## 下单

![order](https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/BuildYourOwnLadder/order.png)

# 配置VPS

## 启用ufw防火墙

## 更改SSH默认端口

## 禁止密码登陆

## 新建普通用户

## 禁止root登录

## 安装Fail2ban

# 安装XRay-core

# Warp解锁chatGPT
