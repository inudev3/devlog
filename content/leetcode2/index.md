---
emoji: 🧢
title: leetcode 269.Alien Dictionary
date: '2023-01-15 03:00:00'
author: inu
tags: leecode topology-sort bfs
categories: algorithm
---

위상정렬은 맞는데 구현이 좀 까다롭다.

```kotlin
class Solution {
    fun alienOrder(words: Array<String>): String {
        //알파벳 순서가 다름
        val adj = hashMapOf<Char, MutableList<Char>>()
        val indegree = hashMapOf<Char,Int>()
        val set= hashSetOf<Char>()
        for(word in words){
            for(c in word)if(c !in set)set.add(c)
        }
        set.forEach{ch-> adj.put(ch, mutableListOf()); indegree.put(ch,0)}
        for(i in 0..words.size-2){
            if(words[i].length>words[i+1].length && words[i].startsWith(words[i+1]))
            return ""
             for(j in 0..minOf(words[i].length, words[i+1].length)-1){
                 if(words[i][j]!=words[i+1][j]) {
                     
                     indegree.put(words[i+1][j], indegree.getOrDefault(words[i+1][j],0)+1)
                     adj[words[i][j]]!!.add(words[i+1][j])
                     break
                 }
             }
        }
        val queue = LinkedList<Char>()
        for((k,v) in indegree)if(v==0)queue.add(k)
        val sb = StringBuilder()
        while(queue.isNotEmpty()){
            val curr=queue.remove()
            sb.append(curr)
            for(next in adj[curr]!!){
                indegree.put(next,indegree.get(next)!!-1 )
                if(indegree[next]==0)queue.add(next)
            }
        }
        if(sb.length<indegree.size) return ""
        return sb.toString()
    }
    
}
