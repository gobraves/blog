---
layout: post  
title: celery_learning.md 
date: 2018/03/24 
tags: 

categories: 
 - python
 
---

# Celery学习小结

## Celery是什么
在我最初的理解中，它是一个由消息队列，任务队列和一个backend(db or other for saving data)组成的调度工具。且具有以下功能：
1. 定时任务
2. 任务调度
3. 任务并行
4. 监控

## 学习过程
### 入坑
最开始没有理解它的task，以为它的工作机制是一个任务从取数据到计算，再到最终存储都在Task流中完成，当时就疑惑应该怎样去调用它。
### 踩坑
因此当看到flower时特别开心,因为可以通过api调用任务。
### 挣扎
在试用中遇到两个问题：
1. 调用相应任务时，调用成功，但无反应
    1. task_queue设置是否正确
    2. 相应的queue是否启动
2. 在task中使用chain时，无法获得结果，且那个task会显示失败
### 脱坑
但在尝试使用chain时，文档注明不能在task中get结果，意味着使用chain的部分不能是一个task，因此完全可以将其写在一个service中，然后通过api去访问

## 技巧
1. ```from celery.tasks.control import inspect```可查看相关任务信息
2. 启动任务后，通过```celery -A proj events```可查看每个任务的执行情况

## 疑问
1. 一个worker具体定义？
2. celery中task_queue有什么作用？
3. celery中exchange有什么作用?是否和rabbitmq作用一样
