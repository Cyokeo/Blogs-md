---
title: Linux - 中断下半部的三种机制
categories: 操作系统
tags:
  - Linux
---

## 参考博客
- [# Linux tasklet 机制的理解](https://bbs.aw-ol.com/topic/2470/%E5%88%86%E6%9E%90%E7%AC%94%E8%AE%B0-linux-tasklet-%E6%9C%BA%E5%88%B6%E7%9A%84%E7%90%86%E8%A7%A3)

## Top half vs Bottom Half
- 上半部关中断 VS 下半部开中断


## softirq
- 内核目前配置了10+个softirq，最多只能注册32个
- 使用前必须使用`open_softirq`注册对应的处理函数；使用时调用`raise_softirq()`触发/标记软中断的发生；内核使用`softirq_init`对软中断系统进行初始化
- 在中断上下文执行；因此不能睡眠、触发调度
- ***同一个softirq可以在不同CPU上并行执行；在同一个CPU上只能串行执行***


## tasklet
- 使用比较灵活，可以动态创建：目前驱动里面用得比较多
- 使用`tasklet_init`进行初始化；在中断上半部调用`tasklet_schedule`就可以触发其在下半部执行
- 实现依赖于`TASKLET_SOFTIRQ`这一种软中断机制
- 在softirq上下文中执行，也即在中断上下文中执行
- ***同一个tasklet不论在同一个CPU还是多个CPU上都只能串行执行***，当第二个CPU检测到当前tasklet正在其它CPU上执行时，会将该tasklet重新挂到CPU的tasklet链表中，并重新触发softirq，等待下次执行
- 因此使用时不用考虑复杂的重入、并行等问题


## work queue
- 在进程上下文执行；要执行的工作交给一个内核线程进行。因此其在处理过程中可以睡眠，可以触发调度。而上面两种机制都不可以