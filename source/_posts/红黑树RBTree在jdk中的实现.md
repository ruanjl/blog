---
title: 红黑树在jdk中的实现
date: 2019-01-26 10:51:33
tags: [数据结构]
categories: [数据结构]
---
### 前言
jdk里主要有TreeMap和HashMap里有用到红黑树的数据结构，我觉得TreeMap的实现看起来友好一点，但是长时间不看也容易忘记，在这一并整理一下。本次源码是基于jdk1.8的TreeMap的插入和删除的方法。

### 基本概念
红黑树的**基本概念**还是贴上来好点，该数据结构必须**同时**满足下面5点条件,[参见wiki](https://zh.wikipedia.org/wiki/%E7%BA%A2%E9%BB%91%E6%A0%91)：
>1. 节点是红色或黑色。
>2. 根是黑色。
>3. 所有叶子都是黑色（叶子是NIL节点）。
>4. 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
>5. 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点    

>其实如果你能细心看完wiki的介绍并理解这里就不用看了，由于情况较多我这里是跟着源码做的总结，个人觉得删除相对难理解一点。

### 插入
插入相对简单点，主要的工作是修复插入带来的影响，梳理下**核心逻辑**：
>父节点是红色的时候才会需要调整
1. 获取到叔节点的颜色  
    1. 为红色：将父节点和叔节点染黑，爷节点染红，问题推向爷节点(**回到开始**)
    2. 为黑色：
        1. 先将当前节点方向变为和父节点同向(即当前节点，父节点，爷节点摆成一条直线)
        2. 换色(交换父，爷节点颜色) + 旋转(爷节点为中心) **调整结束**

```` java
    /** From CLR */
    private void fixAfterInsertion(Entry<K,V> x) {
        x.color = RED;
        // 父节点是红色的时候才会需要调整
        while (x != null && x != root && x.parent.color == RED) {
            //  父节点是左节点的情况
            if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
                // 1. 获取到叔节点U(uncle)
                Entry<K,V> y = rightOf(parentOf(parentOf(x)));
                if (colorOf(y) == RED) {
                // 如果U为红色:将父节点和叔节点染黑，爷节点染红，问题向爷节点递推(进入下一个循环)
                    setColor(parentOf(x), BLACK);
                    setColor(y, BLACK);
                    setColor(parentOf(parentOf(x)), RED);
                    x = parentOf(parentOf(x));
                } else {
                    // U 为黑色 不用递推可以直接解决
                    // 如果当前节点是父节点的右节点，左旋，将自己变成父节点的左节(变得和父节点同向)
                    if (x == rightOf(parentOf(x))) {// 等价转换
                        x = parentOf(x);
                        rotateLeft(x);
                    }
                    // 换色(交换父，爷节点颜色) + 旋转(爷节点为中心)
                    setColor(parentOf(x), BLACK); // 父节点已经为黑
                    setColor(parentOf(parentOf(x)), RED);
                    rotateRight(parentOf(parentOf(x)));
                    // 此时父节点为黑色，调整结束
                }
            } else { // 对称的，和上面思路一样
                ......
            }
        }
        root.color = BLACK;
    }
````

### 删除
删除相对复杂，主要的工作是修复删除带来的影响，梳理下**核心逻辑**：
>当前节点是黑色的时候才需要修复，前面先使用后继节点删除，所以需要向当前节点这里**补一个黑色节点**
1. 得到兄弟节点
    1. 将兄弟节点变为黑色(如果兄弟节点为红的话)
    2. **兄弟节点**是否有**红色子节点**
      1. 没有: 问题移向父节点(**回到开始1**)
      2. 有: 先将和当前节点反方向上的**兄弟节点的子节点N**变为红色，将N染黑，交换兄弟和父节点颜色，以父节点为中心，向当前节点方向旋转(**结束**)
        
```` java
    /** From CLR */
    private void fixAfterDeletion(Entry<K,V> x) {
        while (x != root && colorOf(x) == BLACK) {
            // 当前为左节点
            if (x == leftOf(parentOf(x))) {
                // 得到兄弟节点 
                Entry<K,V> sib = rightOf(parentOf(x));
                // 如果兄弟节点为红色， 交换兄弟节点和父节点颜色并左旋(目的是将兄弟节点变为黑色) 
                if (colorOf(sib) == RED) {
                    setColor(sib, BLACK);
                    setColor(parentOf(x), RED);
                    rotateLeft(parentOf(x));
                    sib = rightOf(parentOf(x));
                }
                // 兄弟节点是否有红色子节点
                if (colorOf(leftOf(sib))  == BLACK &&
                    colorOf(rightOf(sib)) == BLACK) {
                    setColor(sib, RED);
                    x = parentOf(x);
                } else {
                    // 将兄弟节点的右子节点变为红色
                    if (colorOf(rightOf(sib)) == BLACK) {
                        setColor(leftOf(sib), BLACK);
                        setColor(sib, RED);
                        rotateRight(sib);
                        sib = rightOf(parentOf(x));
                    }
                    // 将兄弟节点右子节点染黑，交换兄弟和父节点的颜色
                    setColor(sib, colorOf(parentOf(x)));
                    setColor(parentOf(x), BLACK);
                    setColor(rightOf(sib), BLACK);
                    // 以父节点为中心左旋，这样自己这边就多了一个黑色节点，补偿结束！
                    rotateLeft(parentOf(x));
                    x = root;
                }
            } else { // 对称,逻辑一模一样
                ......
            }
        }
        setColor(x, BLACK);
    }
````
## 总结  
>wiki里的c实现使用的是**尾递归**,但是java使用的是**非递归的while**实现的，好像目前java没有对尾递归做优化。
### 插入的修复
![](/images/红黑树插入调整.png)
> 图片双击放大
--- 
### 删除的修复
![](/images/红黑树插入调整.png)