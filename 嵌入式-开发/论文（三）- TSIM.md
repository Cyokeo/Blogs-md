---
title: 论文（三）- TSIM
categories: 嵌入式-开发
---
## TSIM
1. 没有实现pipeline
2. TSIM is not currently multi-core capable
3. Generic Debug Instrument (GDI) --> 这个好像是tsim的控制接口
4. 使用Tasking界面使用TSIM

## 使用
1. 结合MConfig中的`__IOWRITE`、`__CYCLES`关键字可以把之前的cycle打印出来
2. TSIM User Guide 3.2.1
	- Each instruction has a fixed latency\
	- Parallel execution of instructions - 【superscalar feature】
	- Cache data/code with LRU replacement
	- Variable memory configuration and latency

## 注意
1. 一定要注意数据和代码，尤其是数据放在了哪个DSPR里面
2. 明显，系统时钟中断比较频繁时，会导致更多的抖动！！！
3. 变量的存放位置很关键！！！
4. 必要的时候需要反汇编去看汇编码啊！！！