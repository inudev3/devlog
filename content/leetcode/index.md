---
emoji: 🧢
title: leetcode 1136.Parallel Courses
date: '2023-01-15 03:00:00'
author: inu
tags: leecode topology-sort dfs bfs
categories: algorithm
---
1.BFS
가장 간단하게 위상정렬과 BFS를 사용한 솔루션인데 한 학기에 들을 수 있는 과목의 수가 무제한이므로 queue size에 대해 모두 순회해야 한다. 

```kotlin
class Solution {
    fun minimumSemesters(n: Int, relations: Array<IntArray>): Int {
        //queue를 바꿔가면서 하는 풀이
        val courses = Array(n+1){mutableListOf<Int>()}
        val indegree= IntArray(n+1)
        for((prev,next) in relations){
            courses[prev].add(next)
            indegree[next]+=1
        }
        var queue = LinkedList<Int>()
        for(i in 1..n)if(indegree[i]==0)queue.add(i)
        var step=0
        var count=0
        while(queue.isNotEmpty()){
            step+=1
            val nextQueue = LinkedList<Int>()
            for(node in queue){
                count+=1
                for(next in courses[node]){
                    indegree[next]-=1
                    if(indegree[next]==0) nextQueue.add(next)
                }
            }
            queue=nextQueue
        }
        return if(count==n) step else -1
    }
}
```

2.BFS(cycle check+longest path)

The number of semesters needed is equal to the **length of the longest path**
 in the graph.

```kotlin
class Solution {
    fun minimumSemesters(n: Int, relations: Array<IntArray>): Int {

        val courses = Array(n+1){mutableListOf<Int>()}
        for((prev,next) in relations){
            courses[prev].add(next)
        }
         fun dfsCycleCheck(node:Int, visited:IntArray):Int{
            if(visited[node]!=-1)return visited[node]
            else visited[node]=-1//visiting, currently in stack
            for(next in courses[node])if(dfsCycleCheck(next,visited)==-1)return -1
            visited[node]=1
            return 1
        }
        fun dfsLongestPath(node:Int, visitedLength:IntArray):Int{
            if(visitedLength[node]!=0)return visitedLength[node]
            var max=1
            for(next in courses[node])max=maxOf(max,dfsLongestPath(next,visitedLength)+1)
            visitedLength[node]=max
            return max
        }
        val visited = IntArray(n+1)
        for(i in 1..n){
            if(dfsCycleCheck(i, visited)==-1)return -1
        }
        val visitedLength = IntArray(n+1)
        var max=1
        for(i in 1..n){
            max=maxOf(max, dfsLongestPath(i,visitedLength)+1)
        }
        return max
       
    }
}
```

stackoverflow 발생

1. 둘을 합친 one-pass 솔루션(dfs로 사이클체크하면서 동시에 최대길이 갱신)

```kotlin
class Solution {
    fun minimumSemesters(n: Int, relations: Array<IntArray>): Int {
        val courses = Array(n+1){mutableListOf<Int>()}
        for((prev,next) in relations){
            courses[prev].add(next)
        }
     
        fun dfsLongestPath(node:Int, visitedLength:IntArray):Int{
            if(visitedLength[node]!=0)return visitedLength[node]
            else visitedLength[node]=-1
            var max=1
            for(next in courses[node]){
                val length=dfsLongestPath(next,visitedLength)
                if(length==-1)return -1
                else max=maxOf(max,length+1)
            }
            visitedLength[node]=max
            return max
        }
      
        val visitedLength = IntArray(n+1)
        var max=1
        for(i in 1..n){
             val length=dfsLongestPath(i,visitedLength)
             if(length==-1)return -1
             max=maxOf(max,length)

        }
      
        return max
       
    }
}
```
