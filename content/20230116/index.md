---
emoji:
title: leetcode 1591. Strange Printer II
date: '2023-01-16 03:00:00'
author: inu
tags: leecode topology-sort dfs bfs
categories: algorithm
---

이 문제를 위상정렬로 풀어낼 수 있는 응용력이 중요한 가장 그래프스러운 문제.

```kotlin
class Solution {
fun isPrintable(targetGrid: Array<IntArray>): Boolean {
val m = targetGrid.size
val n = targetGrid[0].size

        val indegree = IntArray(61)
        val graph = Array(61){mutableListOf<Int>()}
        fun search(color:Int){
             val rect = intArrayOf(m,n, -1,-1)
            for(i in 0..m-1)for(j in 0..n-1) if(targetGrid[i][j]==color){
               
                rect[0] = minOf(i,rect[0])
                rect[1] = minOf(j,rect[1])
                rect[2] = maxOf(i,rect[2])
                rect[3] = maxOf(j,rect[3])
            }
            for(i in rect[0]..rect[2])for(j in rect[1]..rect[3])if(targetGrid[i][j]!=color){
                graph[targetGrid[i][j]]!!.add(color)
                indegree[color]+=1
            }
        }
        for(i in 0..60)search(i)
        
        val zeros = ArrayDeque<Int>()
        for(i in 0..60)if(indegree[i]==0)zeros.add(i)
        val seen = hashSetOf<Int>()
        while(zeros.isNotEmpty()){
            val curr=zeros.poll()
            if(curr in seen)continue
            seen.add(curr)
            for(next in graph[curr]){
                indegree[next]-=1
                if(indegree[next]==0)zeros.add(next)
            }
        }
        return seen.size==61
    }
}
```