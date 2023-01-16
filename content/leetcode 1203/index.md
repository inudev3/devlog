---
emoji:
title: leetcode 1203.Sort Items by Groups Respecting Dependencies
date: '2023-01-16 03:00:00'
author: inu
tags: leecode topology-sort bfs
categories: algorithm
---


```kotlin
class Solution {
fun sortItems(n: Int, m: Int, group: IntArray, beforeItems: List<List<Int>>): IntArray {
//n아이템개수, m은 그룹개수

        fun topOrder(graph:Array<MutableList<Int>>,indegree:IntArray):IntArray{
            val res= mutableListOf<Int>()
            val queue=ArrayDeque<Int>()
            for(i in indegree.indices)if(indegree[i]==0)queue.add(i)
            while(queue.isNotEmpty()){
                val curr=queue.poll()
                res.add(curr)
                for(next in graph[curr])if(--indegree[next]==0)queue.add(next)
            }
            return if(res.size==indegree.size)res.toIntArray() else intArrayOf()
        }
        var groupCnt=m
        for(i in 0..n-1)if(group[i]==-1)group[i]=groupCnt++
        val graphGroups = Array(groupCnt){mutableListOf<Int>()}
        val graphItems = Array(n){mutableListOf<Int>()}
        val indegreeGroups = IntArray(groupCnt)
        val indegreeItems = IntArray(n)
        for(i in 0..n-1)beforeItems[i].forEach{item->
            graphItems[item].add(i)
            indegreeItems[i]+=1
            if(group[i]!=group[item]){
                graphGroups[group[item]].add(group[i])
                indegreeGroups[group[i]]+=1
            }
        }
        val itemOrder = topOrder(graphItems,indegreeItems)
        val groupOrder = topOrder(graphGroups,indegreeGroups)
        if(itemOrder.isEmpty() || groupOrder.isEmpty())return intArrayOf()
        val order=itemOrder.groupBy{ item->group[item] }

        return groupOrder.flatMap{ it-> order.getOrDefault(it, listOf())}.toIntArray()
        

    }
}
```