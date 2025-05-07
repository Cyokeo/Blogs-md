---
title: TCP相关
categories: 面试八股
---
### [三次握手连接阶段，最后一次ACK包丢失会进入什么样的一个状态](https://www.cnblogs.com/jelly12345/p/14769028.html "发布于 2021-05-14 16:24")
在 TCP 建立连接的三次握手连接阶段，如果客户端发送的第三个ACK包丢了，那么客户端和服务端分别进行什么处理呢？  
相信了解 tcp 协议的人，三次握手的过程肯定很了解了。第三次的 ack 包丢失就是说在 client 端接收到 syn + ack 之后，向 server 发送的 ack 包 由于各种原因 server 没有收到。这时 client, server 分别会进行怎样的处理呢？  
**Server 端**  
第三次的ACK在网络中丢失，那么Server 端该TCP连接的状态为SYN_RECV,并且***会根据 TCP的超时重传机制***，会等待3秒、6秒、12秒后重新发送SYN+ACK包，以便Client重新发送ACK包。  
而Server重发SYN+ACK包的次数，可以通过设置/proc/sys/net/ipv4/tcp_synack_retries修改，默认值为5.  
如果重发指定次数之后，仍然未收到 client 的ACK应答，那么一段时间后，Server自动关闭这个连接。  
**Client 端**  
在linux c 中，client 一般是通过 connect() 函数来连接服务器的，而***connect()是在 TCP的三次握手的第二次握手完成后就成功返回值***。***也就是说 client 在接收到 SYN+ACK包，它的TCP连接状态就为 established （已连接），表示该连接已经建立***。那么如果 第三次握手中的ACK包丢失的情况下，***Client 向 server端发送数据，Server端将以 RST包响应，方能感知到Server的错误。***