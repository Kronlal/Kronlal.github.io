---
layout: post
title: unordered_map如何排序?
subtitle: unordered_map笔记
date: 2023-03-22 19:50:00 +0800
categories: 数据结构与算法
author: 月梦
cover: 'https://s1.ax1x.com/2023/09/04/pPrl0pt.jpg'
cover_author: 'WilliamFiset'
cover_author_link: 'https://www.youtube.com/@WilliamFiset-videos'
tags: 
- 数据结构 
- 算法 
---

### 1. 简介
- unordered_map是一种key-value关联容器，可以高效地通过单个key值查找对应的value  
- key值是唯一的
- unordered_map不按序存储，其底层实现是哈希表，根据key的哈希值，将元素存在特定位置，所以根据key查找value时非常高效，时间复杂度为O(1)

### 2. 使用
需要使用哈希表的场景都可以使用一个unordered_map来实现  

### 3. unordered_map排序
如何对无序的unordered_map进行排序？  

使用pair容器，每个pair存一对键值对，作为元素放入一个vector，重写sort()的compare函数，就可以对vector进行排序了。  

具体实现(针对 [[HOT100]347前k个高频元素](https://ymiir.top/leetcode/hot100-347)这道题)：  
```
class Solution {
public:
    vector<int> topKFrequent(vector<int>& nums, int k) {
        unordered_map<int,int> m;
        for(auto i:nums){
            if(m.count(i))
            m[i]++;
            else
            m[i]=1;
        }
        vector<pair<int,int>> v;
        for(auto i:m){
            v.push_back(i);
        }
        sort(v.begin(),v.end(),[](auto &p1,auto &p2){return p1.second>p2.second;});
        vector<int>re;
        for(int i=0;i<k;i++)
        {
            re.push_back(v[i].first);
        }
        return re;
    }
};
```
