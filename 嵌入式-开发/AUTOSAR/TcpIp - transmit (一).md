---
title: TcpIp - transmit (一)
categories: 嵌入式-开发
---
## 单socket不能并发
TcpIp_UdpTransmit
1. 进入关键区
2. 检查perSocket的bool变量是否被设置
	- 被设置说明有待发送的数据，此次不能发送
	- 否则说明可以发送
3. 若没有待发送的数据，则此次可以发送，并设置bool变量为true
4. 每个socket结构有一个txReqList
	- 每次发送将