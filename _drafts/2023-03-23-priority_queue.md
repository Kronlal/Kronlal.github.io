---
layout: post
title: priority_queue
subtitle: priority_queue笔记
date: 2023-03-23 19:50:00 +0800
categories: 数据结构与算法
author: 月梦
cover: 'https://s1.ax1x.com/2023/09/04/pPrl0pt.jpg'
cover_author: 'WilliamFiset'
cover_author_link: 'https://www.youtube.com/@WilliamFiset-videos'
tags:
- 数据结构
- 算法
---

### 1. priority_queue简介
- priority_queue优先队列，其底层用堆来实现。  
- 在优先队列中，队首元素一定是当前队列中优先级最高的一个。  
- 插入元素，堆结构调整，保证队首元素优先级最高。  
- 不同于队列，优先队列只能通过top()函数来访问队首元素。  

### 2. 基本使用
- push(); 入队  
- top(); 获取队首元素  
- pop(); 令队首元素出队  

凡是需要用堆结构的地方，都可以考虑优先队列。  
### 3. 如何定义优先
#### 3.1 对于基本数据类型
数字越大优先级越高，priority_queue默认使用vector作为存储数据的容器。  
以下两种写法等价：  
```
priority_queue<int> p;
priority_queue<int,vector<int>,less(int)> p;
```
#### 3.2 对于自定义数据类型
需要自己定义比较方式......  
未完待续
