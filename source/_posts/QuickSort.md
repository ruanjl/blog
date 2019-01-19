title: 'QuickSort Application in java'
date: 2018-11-25 23:07:22
tags: Algorithm
categories: 数据结构与算法
top: True
---
## 前言：
**对各种_基本排序_有些了解的人都会知道，各种单一的排序都有他自己合适的使用场景，快速排序是综合表现最好的。
而实际应用中的排序可要考虑的实在是太多了，看jdk的排序是怎么做的**
![I love it when a plan comes together.](http://ww1.sinaimg.cn/large/006Cwrd9gy1fxskn2tpksj31hc0u0guq.jpg)

    java中的Arrays.Sort()方法是我们常用的排序方法，有心的人肯定点进去源码里面看过的，随着jdk的变化
    这个排序也有持续的变动，说明维护的人还是很愿意花精力在这个方法上的，特别是jdk1.8的Arrays.Sort()，
    对一些基本算法没有足够了解的人看起来还是很吃力的(我说的是以前的我)，我花了点时间整理下这个算法。
### 涉及的算法和思路
   
    
    
    