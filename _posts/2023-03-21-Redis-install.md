---
layout: post
title: Redis安装(使用Docker)
date: 2023-03-21 15:50:00 +0800
categories: Redis
author: Lundrea
tags: 
- Redis  
- Docker
---

* content
{:toc}










### 使用Docker部署Redis
#### 1. 拉取镜像
```
docker pull redis#默认下载最新版本
```
#### 2. 创建本地目录（用于挂载）
创建本地映射目录用于挂载Redis配置文件和数据文件  
```
#递归创建目录
mkdir -p /home/docker/redis/data
mkdir -p /home/docker/redis/conf
#递归修改文件权限为可编辑
chmod -R 777 /home/docker/
```
#### 3.下载配置文件
下载redis.conf,上传到服务器。  
在[官网](http://www.redis.cn/download.html)下整个文件，找到redis.conf,上传到服务器刚刚创建的conf文件夹下。  

#### 4. 修改配置文件
注释掉下面这句，使redis可以外部访问。
```
bind 127.0.0.1
```
```
#设置密码
requirepass 密码
#改为yes，持久化
appendonly yes
```
#### 5. 创建docker容器
```
sudo docker run -p 6380:6379 --name redis -v /home/docker/redis/conf:/etc/redis/redis.conf -v /home/docker/redis/data:/data -d redis redis-server /etc/redis/redis.conf --appendonly yes
-p端口映射，前面主机，后面容器，本机6379端口被占用，所以换了个
--name指定容器的名称
-v挂载文件或目录，前面表示主机目录，后面表示容器部分
-d后台启动redis
```
#### 6. 通过客户端连接Redis
进入Redis容器
```
#我的容器名称是redis1
docker exec -it redis1 /bin/bash
#通过redis-cli连接Redis
redis-cli
```
输入ping命令，若输出PONG，表示目前处在一个正常的连通状态。  

