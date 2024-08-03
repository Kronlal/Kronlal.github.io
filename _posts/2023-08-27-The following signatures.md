---
layout: post
title: 解决方法The following signatures couldn't be verified because the public key is not available
subtitle: Ubuntu使用过程中的一些错误解决方法记录
date: 2023-08-27 19:50:00 +0800
categories: Linux
author: 月梦
cover: 'https://s1.ax1x.com/2023/09/03/pPDyQit.jpg'
cover_author: 'thehackernews'
cover_author_link: 'https://thehackernews.com/2023/07/gameoverlay-two-severe-linux.html'
tags:
- Linux  
---

## 概述
今天在Ubuntu20.04上执行sudo apt-get update命令时，遇到以下错误:
```
Err:2 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY B53DC80D13EDEF05
```
他说这个签名不能被验证，因为没有可用的公钥，并且表示没有B53DC80D13EDEF05这个密钥
## 解决方法
我们给他加上这个密钥就可以了，使用命令:
```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys B53DC80D13EDEF05
```
apt-key命令用于管理Debian Linux系统中的软件包密钥,每个发布的Debian软件包都是通过密钥认证的。  
adv: 告知apt-key工具使用高级模式  
--keyserver keyserver.ubuntu.com: 指定从哪个GPG密钥服务器获取密钥  
--recv-keys: 指定要接收的GPG密钥ID  

这个命令从指定的服务器获取对应ID的密钥并添加到本地，用于软件包的验证。