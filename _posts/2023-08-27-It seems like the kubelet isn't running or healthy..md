---
layout: post
title: 解决方法-It seems like the kubelet isn't running or healthy.
subtitle: 在部署k8s集群时遇到的一些问题
date: 2023-08-27 15:50:00 +0800
categories: 云原生
author: 月梦
cover: 'https://s1.ax1x.com/2023/09/03/pPDsm40.png'
cover_author: 'Yevgeniy Brikman'
cover_author_link: 'https://medium.com/@brikis98'
tags:
- Kubernetes
- 云原生
---

## 概述
如果在使用kubeadm初始化(kubeadm init)或者添加节点到k8s集群(kubeadm join)中时，遇到类似下图的错误:
[![pPUr9JJ.png](https://s1.ax1x.com/2023/08/27/pPUr9JJ.png)](https://imgse.com/i/pPUr9JJ)
只要出现了**It seems like the kubelet isn't running or healthy.** 这句提示，都可以尝试使用下面的解决方法来解决。
## 解决方法
依次执行:
```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
即可。