---
layout: post
title: 拓扑排序
date: 2024-03-25 19:50:00 +0800
categories: 算法
author: Lundrea
tags:
- 算法  
---

* content
{:toc}
拓扑排序（Topological Sorting）是一种针对有向无环图（DAG）的排序算法。在拓扑排序中，图中的顶点被安排成一个线性序列，使得如果图中存在一条从顶点 u 到顶点 v 的有向边，则在序列中顶点 u 出现在顶点 v 之前。  

拓扑排序通常用于描述一组任务或事件之间的依赖关系，其中每个任务都有一些前置任务，必须在它们完成后才能执行。通过拓扑排序，我们可以确定一种合理的执行顺序，以确保所有任务都按照其依赖关系得到正确执行。  







在实现上，拓扑排序可以根据下面的步骤实现：  
1. 找到入度为 0 的顶点，把这些节点加入到排序结果中，这些节点没有前置任务  
2. 将已经加入到排序结果中的节点及其相连的边删去，更改图中其他节点的入度  
3. 重复 1，2 步，直到图中没有入度为 0 的节点

如果被拓扑排序的图是一个有向无环图，那么所有顶点都会被排序；而如果图中有环，则拓扑排序只会得到一部分的节点序列。  

因此，拓扑排序也常被用来检测一个图中是否有环。  

下面是拓扑排序入门的一个经典案例及其代码实现（_2024春招字节跳动暑期实习一面代码题_）：

> **题目描述**: 一共有 n 门课你可以选，从课程0，1到课程 n-1 。 有一个数组 p ,  p[i] = [a, b]  表示你必须要先修完课程 b 才可以修课程 a 。 请写一个方程返回 true / false 表示你是否能修完所有的课程。  

```java
package com.ymiir;

import java.util.*;

public class Main {

    public static boolean topologicalSorting(int n, int[][] graph){
        // 创建入度数组，存储每个节点的入度
        int[] inDegree = new int[n];
        // 邻接矩阵
        Map<Integer, List<Integer>> map = new HashMap<>();
        // 入度数组置 0
        for(int i=0;i<n;i++){
            inDegree[i] = 0;
        }
        // 初始化邻接矩阵和入度数组
        for(int i=0;i<graph.length;i++){
            int node = graph[i][0];
            int pre = graph[i][1];
            inDegree[node]++;
            map.computeIfAbsent(pre, k->new ArrayList<>()).add(node);
        }
        // 创建队列并初始化队列，将入度为 0 的节点入队，用于拓扑排序
        Queue<Integer> queue = new ArrayDeque<>();
        for(int i=0;i<n;i++){
            if(inDegree[i]==0){
                queue.offer(i);
            }
        }

        // 拓扑排序
        // count 用于记录拓扑排序遍历到的节点
        int count = 0;
        while(!queue.isEmpty()){
            // 一个元素出队
            int temp = queue.poll();
            count++;
            // 如果邻接矩阵中有
            if(map.containsKey(temp)){
                // 删掉当前元素及与之相连的边，具体表现在子节点入度 -1
                for(int v : map.get(temp)){
                    inDegree[v]--;
                    // 如果调整之后入度为 0 ，则可以放入拓扑排序序列，入队
                    if(inDegree[v]==0){
                        queue.add(v);
                    }
                }
            }
        }
        // 如果拓扑排序遍历到了所有的节点，则说明可以修完所有的课
        return count==n;
    }

    public static void main(String[] args) {
        int[][] graph = ;
        boolean flag = topologicalSorting(4,graph);
        System.out.println(flag);
    }
}
```
