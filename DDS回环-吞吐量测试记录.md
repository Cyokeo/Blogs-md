---
title: DDS回环-吞吐量测试记录
date: 2024-06-22 19:42:05
updated: 2024-06-24
categories: 
- [日常记录]
---

## LW DDS
- 240607-可以用于毕业小实验数据采集
- 240624-用于项目中期考察时的补充实验：回环/吞吐量


## Cyclone DDS
- 240624-用于项目中期考察时的补充实验：回环/吞吐量



## RTI DDS
- 


## 调试记录
- Cyclone 和 Fast[2.12.0]都定义了0x8007这个PID，但是含义不一样，因此Fast在接收到Cyclone的data(p)报文，进行处理时出错，不会创建参与者代理，因此匹配不了
- FastDDS调试时常用的断点：
  - on_new_cache_change_added() ["Ignore announcement from own RTPSParticipant"] at file: PDPListener.cpp -> 用于处理data(p)报文
  - EDPSimplePUBListener::on_new_cache_change_added() at file: EDPSimpleListeners.cpp -> 用于处理data(w)报文
