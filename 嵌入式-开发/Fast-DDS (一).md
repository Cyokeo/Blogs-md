---
title: Fast-DDS (一)
categories: 嵌入式-开发
---
## 版本说明
- Fast-DDS: 2.12.0
- Fast-CDR: 2.1.0
- Tinyxml2: 6.0.0
- Fast-DDS-Gen: v3.1.0

## Fast-DDS well-known策略
每个参与者都会创建一个接收线程监听well-knwon的ip/port「239.255.0.1/7400」；多线程在创建socket，bind之前使用「SO_REUSEPORT」参数进行设置

>在Linux - UDP中:
>1. 单播的情况下，如果多个进程绑定同一个ip和端口，则只会有一个进程收到请求，具体哪个进程不同的操作系统实现不一样
>2. 多播的情况下，多个绑定同一个ip和端口的进程，同一个请求，每个进程都会收到

## 发送 - reliable
1. 从data中获取key_instancehandle
2. 从changePool中获取change
3. 下面的操作用一把🔒锁住！！
4. 准备change「prepare_change()」
	1. 如果history满了，需要尝试清理history【根据不同的KeppLast/KeepAll QoS采取不同的删除策略】。这里重点看KeepLast【DataWriterHistory::remove_change_pub()】
		1. history中change使用vector管理，且每次新元素插入到尾部，因此这里尝试删除头部change
		2. 如果关联Topic是No_KEY的，直接将change从History的vector中移除即可【当然还要释放对应的资源】
		3. 如果是WITH_KEY的，根据key找到key_changes维护数组：接着将该change从history中删除，接着从key_changes中移除该change的指针
	2. 如果删除了某个change，而且该change还没有被acked，则通知writer
	3. 将此次新的change插入
		1. 如果NO_KEY，简单插入History中即可
		2. 如果WITH_KEY，还要检查该change管理的instance是否满，如果满了，还要把该instance中的旧change删除，肯定还要删除该旧change在Histroy中对应的cache change
		3. 先把新change的指针放入instance相关的change指针数组中
	4. ++seq，并赋值给这个新的change，将change插入history的数组尾部

### remove_min_change()
0. 删除m_changes中的最小值
1. 如果NO_KEY，直接从m_changes中删除即可
2. 如果with_key，要根据该change的key，先将其从instance中删除，之后再从m_changes中删除

### PDPClient
1. 中的pdp_reader使用的就是ReaderHistory的实例；因此statefulReader在调用`mp_history->received_change(a_change, 0)`时调用的就是`ReaderHistory::received_change`改函数仅将change加入到m_changes中，不将其加入到instances中！！！
2. EDPSimple阶段也是这样吗？-> 是的√