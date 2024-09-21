---
title: AUTOSAR - Automatic Tcp Client
categories: 嵌入式-开发
---
## MainFunction
- 周期性可配置
- 里面进行socket connection的连接工作

## Maintain Conn
会对如下三种情况下的socket connection进行`open`操作
- `SoAdSocketAutomaticSoConSetup == True`
	- 这种类型的管理全部交由SoAd模块进行，不允许用户主动使用open/close
- `in reconnect state`
	- 处于这种状态的socket conn，可能是 ==AUTOMATIC== 的，也可能不是
	- 对于非AUTOMATIC，可以主动关闭。但是关闭后，仍有可能处于RECONNECT状态，因此可能需要手动将状态切换为***offline*** 【使用案例 - DoIP】
- **主动调用open()请求建立连接**

使用event queue，向SoConState queue插入事件；每次MainFunction均会检查队列是否空闲；不空闲的话，就会进行Conn的维护工作。显然，当某个Conn open 失败时，会向该queue中插入事件，这样下次周期性SoAd_MainFunction就会继续尝试open and connect。插入的事件信息中包含SoConId信息。

如果该状态事件对应的SoCon上没有Close请求，则会检查并尝试Open：
- 处于RECONNECT状态，或者配置为AUTOMATIC
- 不处于ONLINE状态，且配置了local address
继续检查：
- SoCon对应的socket处于关闭状态，且remote address的 (ip, port)都已经设置
如果检查均通过：
1. 调用底层接口，获取socket
2. bind
3. connect -> 设置socket状态为：SOAD_SOCK_STATE_CONNECT
4. 不成功则进行socket的清理，并插入SO_CON_STATE事件，在下一次MainFunction中重试
如果open成功，且调用connect成功:
1. 如果SoCon的状态不为RECONNECT，则设置SoCon的状态为==RECONNECT==，且通知应用层状态改变【changeCbk】
2. 这些接口都是同步非阻塞的，如何通知连接状态？



## 两个相关回调 - 主要被TcpIp模块调用
### SoAd_TcpConnected
连接成功

### SoAd_TcpIpEvent
对于TCP，仅有如下几种事件：
- SOAD_TCP_RESET
- SOAD_TCP_CLOSED
	- 将tcp client设置为SOAD_CLOSE_SOCKET_RECONNECT状态
	- 并设置SoCon state 事件，等待MainFunc处理: 后续会被设置为RECONNECT状态
- SOAD_TCP_FIN_RECEIVED
	- 对于tcp client 处理类似

## SoAd SoCon有三种状态
###  `OFFLINE`:

### `RECONNECT`:
每个SoCon有几个关闭状态，包括：
- SOAD_CLOSE_SOCKET_RECONNECT
- SOAD_CLOSE_RECONNECT
此时，如果关闭状态对应上面两个，则SoCon会被设置为RECONNECT状态

设置remote address也可能导致SoCon的状态被设置为RECONNECT。只有在OFFLINE状态下才能设置remote。

### `ONLINE`:

## 使用案例 - DoIP
DoIP在调用SoAd模块的关闭连接API后，会继续判断当前连接的状态是否为RECONNECT，如果是，则***主动将状态变更为OFFLINE***

## 启发 - GUID -> LocalIdx
- 只能for所有本地idxs，检查其绑定的GUID信息是否与当前查找的GUID相同！！！


## 参考资料
- [SoAd_CloseSoCon / ](https://www.autosar.org/fileadmin/standards/R20-11/CP/AUTOSAR_SWS_SocketAdaptor.pdf)