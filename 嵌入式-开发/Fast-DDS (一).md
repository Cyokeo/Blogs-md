---
title: Fast-DDS (一)
categories: 嵌入式-开发
---
## Fast-DDS well-known策略
每个参与者都会创建一个接收线程监听well-knwon的ip/port「239.255.0.1/7400」；多线程在创建socket，bind之前使用「SO_REUSEPORT」参数进行设置

>在Linux - UDP中:
>1. 单播的情况下，如果多个进程绑定同一个ip和端口，则只会有一个进程收到请求，具体哪个进程不同的操作系统实现不一样
>2. 多播的情况下，多个绑定同一个ip和端口的进程，同一个请求，每个进程都会收到

