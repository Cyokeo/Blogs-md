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

## 24-10-16 发送架构
1. 从writer的history获取空闲chang buff -> 需要保护起来【而且要考虑多核】
2. 对该buff进行序列化，这里就不需要保护了
3. 遍历writer的【有效：是否有效设置应该采用原子操作保护起来】matched reader进行发送，将数据交给底层 -> 这里应该也不需要保护
	1. 对于删除matched reader操作，应该需要延迟进行！！！；当该reader处于使用状态时，就需要延迟删除该matched reader！！！，连带的可能需要延迟删除该matched reader所属的proxy participant 
	- matched reader 三个状态：ok, sending, deleting
4. 底层是否支持多核发送，这就是底层需要考虑的事情了！！！

5. ***还是需要实现一个多核通信queue机制，使用spin lock保护***
	1. 队列满了直接警告 - 并丢弃此次事件！！
	2. 是否需要提供同步等待接口？
6. Tx event提供QoS
	- Direct Send 不经过Tx Event，直接在dds_write的调用上下文环境中将报文发送出去
	- In Direct Send 要经过Tx Event而且可以设置不同的传输优先级队列！
		- 优先发送高优先级队列中的数据
### local reader and remote writer
1. changes_received 存储所有收到的不连续的数据，不能无限制的收这样的数据；因为这样的数据如果确认接收了，那么它和low_marker之间的所有change也要被缓存；因此如果一个change与change_low_marker不连续，则还需要判断是否超过了极限；才能接收


## 24-10-17 思路
1. 流识别是必须要有的；当识别出这个流已经错过了时隙，或者到达的太早？应该怎么处理？ -> 参考文章《Deadline-Aware Online Scheduling of TSN Flows for Automotive Applications》
2. 基本等同于我要将高优先级的TT流重新映射到哪个稍低优先级的队列中去！是一种实时在线调度的方法
3. 对比：Qbv+Degrade vs raw Qbv vs AVB
4. WCRT - AVB计算方法！！！

### LW-DDS
1. 既然每个核部署一个pp，那就没必要考虑dds_write/read的多核调用；！！！
	- 可以提供一个配置选项：是否允许跨核调用dds_write

## 24-10-18 timer实现
- 精度不用很高
- 提供一个接口，供外部周期性调用【5ms/10ms/】
	- 这个接口设置tick时间，并激活pp_thread
- thread内部，如果有tick事件，则将内部维护的tick++
- 后续处理timer事件【继续使用timer_thread中的fibheap数据结构】

### 还有其它更简单/优秀的实现方式吗？
目的：让pp_thread处理所有的事件，包括「接收/延迟发送/Timer/其它事件」
1. 如果期望可变的【需要的】timer事件导致pp_thread唤醒，而不是像上面那样周期性处理
2. 则需要pp_thread内部根据计算得到的睡眠时间，利用os_timer设定一个触发器，到期后，设置timer事件，并尝试唤醒pp_thread
3. 这样的话，pp_thread内部就不需要维护时间了，但是需要操作系统「外部」提供的时间接口，以获取系统时间

### 总结
这两种实现方式还是有很多相似之处的 -> 也是比较容易兼容的。但是AUTOSAR的时间系统真的很难使用！！！


## 24-10-19 
### 多核通信使用queue来处理事件的缺点
- 只能按序进行事件的处理，不能优先处理接收/发送事件
- AUTOSAR ClearEvent在开始处理前进行清理，可以仅清除待处理的事件类型，因此不会出现「丢事件」的情况
- AUTOSAR：隶属于同一个Task的EventMask是从1开始计数的

***采用Event Group配合相应的Queue「或者链表」来处理***

### 发送
1. reliable：交由pp_thread进行处理，用户需保证数据的有效性；需提供一个回调函数，当序列化完成后，回调通知应用data可以释放了！！！
2. bestEffort：细心处理，可以多核并行/单核并发！！！

### 有哪些事件需要处理
- 接收事件 -> 需要优先响应
- 延迟删除事件 -> 延迟删除WriterMatch，ReaderMatch，「WriterProxy，ReaderProxy，ProxyParticipant」，这三个好像不需要延迟删除
- Write事件：用于发送reliable的data，BestEffort的可以在app上下文中完成发送？


## 24-10-20
1. timer处理还是没有想好啊！！！
	1. 如何获取时间sys_now();
	2. 如何实现：经过一段时间后设置TimerEvent !
2. 可选资源
	- Linux：
		- timerfd
		- eventfd
	- Macos:
		- kqueue 「BSD」
3. 参考libevent/asio如何使用epoll/kqueue

4. Linux/Macos估计得使用kqueue/epoll

## 24-10-21
1. 引入libevent解决定时器问题
2. TODO -> 需要pp_thread与main_event_loop_thread之间通信以期：按照pp_thread的需求设置超时定时器