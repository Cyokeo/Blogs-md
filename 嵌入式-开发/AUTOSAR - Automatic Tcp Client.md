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
- **主动调用open()请求建立连接**

使用event queue，向SoConState queue插入事件；每次MainFunction均会检查队列是否空闲；不空闲的话，就会进行Conn的维护工作

显然，当某个Conn open 失败时，会向该queue中插入事件，这样下次周期性SoAd_MainFunction就会继续尝试open and connect


## 参考资料
- [SoAd_CloseSoCon / ](https://www.autosar.org/fileadmin/standards/R20-11/CP/AUTOSAR_SWS_SocketAdaptor.pdf)