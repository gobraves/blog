---
title: "mysql 索引"
date: 2018-12-25
tags: ["数据库"]
draft: true
---

## 0. 序--没有索引是怎样的情况
如果没有索引，可能就是每次在对应的表中，一条记录一条记录的去匹配，这样会很慢。
索引就像书签，方便查找对应的数据，但是数据库应该如何设计这个索引，使得查找数据时高效、方便？
所以我们可能会想用什么高效的数据结构和算法呢？数组？折半查找？二叉搜索树？红黑树？哈希表？  
算法和数据结构都是根据具体的场景而选择的。那么数据库的场景是怎样的呢？
1. 数据量大，不可能一次全部加载到内存中，需要多次io访问
2. io访问耗时，因此应尽量减少io访问的频率  

所以问题在于：
1. io查找数据操作相比于在内存中数据操作而言，已经非常耗时 
2. 同时，结合磁盘加载数据的方式考虑，若一次查找的数据相互之前相隔较远，也会延长io的访问时间  

所以是否有方式降低io的访问频率，且保证一次所要查询的数据在一块呢？

答案当然是有的，B+树就这么诞生了。它解决了两个问题

## 1. mysql索引的作用
## 2. mysql索引的逻辑实现
### B+树
- B+树是什么
- B+树做索引的实现结构有什么优点
- B+树如何实现索引

## 3. mysql索引的分类(主要讲innoDB)
### mysql索引
1. 唯一索引
2. 全文索引
3. 稀疏索引

### innoDB索引
- 聚集索引
- 辅助索引

### 数据结构划分
- b+ tree
- hash table

## 4. mysql如何使用索引
