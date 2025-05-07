---
title: Linux相关
categories: 面试八股
---
### linux 中swap
从功能上讲，交换分区主要是在内存不够用的时候，将部分内存上的数据交换到 swap空间上，以便让系统不会因内存不够用而导致oom或者更致命的情况出现。

### 什么是swap cache
换区可以包括一个或多个交换区设备（裸盘、逻辑卷、文件都可以充当交换区设备），每一个交换区设备在内存里都有对应的swap cache，**可以把swap cache理解为交换区设备的”page cache”**

### linux查看端口占用情况并解决端口冲突
1. 查看端口占用情况`netstat -an |grep :80`


### linux中如何查看cpu信息
1. CPU信息存在于/proc/cpuinfo文件中
2. 通过 top 指令查看每个CPU的使用情况