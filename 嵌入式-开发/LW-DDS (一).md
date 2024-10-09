---
title: LW-DDS (一)
categories: 嵌入式-开发
---
## stack-refactor 分支 简要说明
1. 期望实现支持多participant，多核多线程。限制🚫如下：
	- 每个pp有一个接收线程；后续应该也要有一个timer线程
	- 每个pp只会监听两个地址：
		1. well-known address: 239.255.0.1:7400 -> 可以配置不监听
		2. specific unicast address
2. 🈲止所有的内存动态申请与释放
	- Linux等平台下，sem/mutex等还是会使用到堆上内存
	- 但是MCU平台，AUTOSAR/FreeRTOS将全面禁用所有动态内存申请与释放
3. 更改数据结构，全面参考AUTOSAR，仅使用数组进行
	- 存储participants, writers, readers
	- 存储远端参与者、实体
4. 增加简易开源log组件，方便后续log

## 24-10-01分支更新说明
1. 当前重构仅进展到：
	- 接收RTPS Msg，并对RTPS Header进行处理
3. 提供了一个简单的程序，`main`为主入口；可以简单运行，并处理接收到的RTPS Msg， 但是只能处理头部
4. 后续步骤
	1. process_cdr，完成数据子消息的处理
	2. 发现阶段SPDP -> SEDP
	3. ......