---
title: LW-DDS (二)
categories: 嵌入式-开发
---
## 线程资源
在LW-DDS的设计中，每个participnt配置了一个专属的线程「pp_thread」，用于处理网络中发送给该participant的所有RTPS报文。除了处理接收的RTPS报文，即接收事件，pp线程还会处理协议栈内部的定时事件。因此总的来说，pp线程采用事件触发的方式进入就绪状态。

🆚：原先使用一个timer线程专门处理定时事件，因此协议栈涉及到了两个线程，进一步的需要考虑线程并发/并行情况下，共享数据的同步问题。改进为使用一个线程进行处理，避免了复杂的关键数据同步「加锁」的处理逻辑，简化了协议栈的开发。

目前协议栈pp线程需要的处理的事件有：
1. eRxEvent：接收事件，即有新的接收报文需要处理。接收报文缓存在一个环形队列中，可配置该环形队列的内存大小。由于该环形队列只有一个消费者：即pp线程；一般只有一个生产者线程：即「MCU：一般在接收中断服务例程中」，即「Linux：一般为执行epool/kqueue的主线程」，因此可以进行无锁化设计
2. eTxEvent：发送事件。发送过程中会涉及到协议栈内部数据的临界区保护问题。***如果发送接口采用eTxEvent的方式***：即将待发送数据挂到writer的发送队列当中，之后激活pp线程进行后续的处理；那么，只需要保护writer发送队列的插入和取出操作即可。
3. eTimerEvent：定时事件处理。表示协议栈内部有超时事件等待处理。需要配合OS提供的系统timer进行工作；当该timer到期时，向pp线程注册eTimerEvent事件
4. eTickEvent：由于eTimerEvent的方式需要配合os提供的系统定时器，有些系统上可能没有提供，或者不好用。因此提供了eTickEvent，由外部周期性地设置该事件。

## 内存资源
LW-DDS采用初始化时，根据本地writer/reader个数及相应的QoS策略配置方式，从一个内存池中申请需要的内存。需要注意⚠️的是：
- 这些内存只会在初始化阶段：即创建pp，writer，reader时被调用
- 这些内存一旦被申请就不会被释放
- 以上两点基于这样一个普遍的现象：本地的writer/reader在创建之后，一般不会被释放掉；只有远端的writer，reader才会被释放。

## 移植接口
LW-DDS设计初期就考虑到了三种OS系统
- AUTOSAR CP
- FreeRTOS
- Linux
LW-DDS依赖的操作系统资源很少，且考虑到了上述三种平台的特性。不同平台下移植时，需要改动的文件基本都在ddsrt文件夹下，主要包括：
- sockets
- task
- compiler
- event_group
- 
另外有一个`rtps_event.h`需要注意，由于AUTOSAR下，eventType采用配置的方式生成，且eventType的值不由用户控制，因此需要修改`rtps_event.h`中event_mask宏定义的值，与实际的配置相符。后续可以将这部配置添加的配置文件中去。


## 未实现
1. 两种built-in writer/reader还没有实现
	1. Topic Detector/Announcer
	2. Participant Message Writer/Reader
2. 变长度的消息类型
	1. LocatorList：目前均只支持一个Locator
	2. UserData QoS：需要的内存大小未知，尚未实现


## 后续开发计划
0. `rtps_write`接口的并发/行操作实现
1. 远端实体退出时，相关的内存归还操作
2. RPC框架
3. Client-Server发现机制


## 通用Timer接口设计
1. libevent配合SIGALARM信号进行处理
	- OK

## 资源回收♻️
1. 仅进行pp级别的存活性判别
	- 基于这样一个可以接受的规则：***pp存活时，如果writer/reader退出链接，则其应该发出具体的断开链接事件报文***
	- writer/reader的主动退出，均由SEDP的writer发出消息进行申明！！！

## 发送接口并发处理
1. reliable的发送全部交给pp_thread处理

## Debug: callback接到的报文数目小于实际发送的报文数量
1. 拿到了旧数据？
2. commit 12b7a269b9d75db62da31732a5e4d24d02be0d97 -> 在macos上，总是触发`EXC_BAD_ACCESS (code=2, address=0x100050000) <- memset() : rtps_task_entry()`