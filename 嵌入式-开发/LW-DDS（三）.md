---
title: LW-DDS（三）
categories: 嵌入式-开发
---
## 对Fast-DDS中的whiteList理解有问题
1. 添加了whiteList后，只允许跟在该List中的ip通信
2. 但是尝试添加更多的whitelist，却添加不进去 ？？？

## 待做
1. 把app文件夹里面的源文件 `#if 0` -> 先不移除，但是后续不再使用这里的文件；只是提供一个应用层的参考实现
2. Project_LW_Lib编译成库后，放到Autosar-Project中竟然发不了周期性的spdp报文！！！需要解决一下
3. app中参考实现，`id.eid.u = ` ->应改正为 `id.idx = `