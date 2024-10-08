---
title: TCP学习-超时与重传
date: 2024-07-24 19:49:45
tags:
---

# TCP超时与重传

## 核心问题
* 发送时设置一个定时器来解决**数据**和**确认**可能丢失的问题
* 如何确定超时判断间隔？
* 如何确定重传的频率？
    - 1，2，4，8， 16，32，64......


## 典型实现
* 测量TCP报文段的往返时间
    - 这样才能比较好的确定超时时间
    - 发出数据 -> 收到ACK的时间间隔？
* 使用测量结果对下一个将要传输的报文段建立重传超时时间


## 四个定时器
* 对每个连接，TCP管理四个不同的定时器
    - **重传定时器**使用于当希望收到<u>另一端的确认</u>
    - **坚持(persist)定时器**使<u>窗口大小信息</u>保持不断流动，即使另一端关闭了其接收窗口
    - **保活(keepalive)定时器**可检测到一个<u>空闲连接的另一端何时崩溃或重启</u>
    - **2MSL定时器**测量一个连接处于TIMEWAIT状态的时间


## 往返(RTT)时间测量
* 该时间在民用网络中可能是会经常变化的，因此TCP需要跟踪这些变化并相应地调整其超时时间

### 测量演进
* 测量某报文段和其响应之间地时间差`M`，并动态更新其RTT值；`RTT = k*RTT + (1-k)*M`
* 超时间隔设置为`RTO = b*RTT`，b地推荐值为2
* `缺陷`：在RTT变化很大地场景下，无法跟踪变化，因此会引起不必要的重传
* `改进`：跟踪RTT的方差并进行平滑，放到RTT估计值中去
* `Karn算法`：当一个分组重传发生时，在重传数据的确认最后到达之前，不能更新RTT的值；这是因为不知道收到的确认不知道对应于哪一次重传

## 拥塞避免算法
* 应对达到中间路由器转发极限的算法
* 假设：分组丢失意味着在源主机和目的主机之间的某处网络上发生了拥塞
* 两种**分组丢失的指示**
    - 发生超时
    - 接收到重复的确认
* **慢启动算法**可以降低分组进入网络中的速率，当拥塞发生时可以使用慢启动来做这一动作以减少网络拥塞。实际中，这两个算法通产一起实现：
    - 拥塞避免相关参数：cwnd（本地拥塞窗口大小）
    - 慢启动相关参数：ssthresh（慢启动门限）
### 流程细节
* 未发生分组丢失时，cwnd逐渐增加，增加的速率取决于cwnd和ssthresh的关系
    - 如果cwnd <= ssthresh，说明此时处于慢启动阶段，cwnd以指数速率增长
    - 如果cwnd > ssthresh，说明此时处于拥塞避免阶段，cwnd以固定速率增长【且一个往返时间内cwnd最多增加1】
* 如果某一时刻发生了分组丢失
    - ssthresh被设置为当前窗口的一半: `min(2, min(cwnd, 通告窗口大小))/2`
    - cwnd的设置与分组丢失的原因有关：
        * 如果是超时引起，则cwnd = 1
        * 否则，被设置为？？？ -> **快速恢复**，设置为新的ssthresh值+3个报文段大小


## 快速重传与快速恢复
* 承接上面最后的问题，当收到**3个及以上**重复的确认，就非常有可能发生了报文丢失，因此重传丢失的数据报文段，无需等待超时器溢出 -> **快速重传**
    - 进入拥塞避免阶段，而不是慢启动  -> **快速恢复**
* 收到重复的确认时，接收方一定收到了大于该确认号的数据段；否则接收方不会有任何响应，只能等待发送方超时器超时进行重传

### 流程细节【这里需要进一步的思考，记忆】
* 当收到第3个重复的ACK时，将ssthresh设置为当前拥塞窗口cwnd的一半。重传丢失的报文段。设置cwnd为ssthresh加上3倍的报文段大小
* 每次收到另一个重复的ACK时，cwnd增加1个报文段大小并发送1个分组（如果新的cwnd允许发送）
* 当下一个确认新数据的ACK到达时，设置cwnd为ssthresh（在第1步中设置的值）。这个ACK应该是在进行重传后的一个往返时间内对步骤1中重传的确认。另外，这个ACK也应该
是对丢失的分组和收到的第1个重复的ACK之间的所有中间报文段的确认。这一步采用的是拥塞避免，因为当分组丢失时我们将当前的速率减半

## 坚持(persist)定时器

* 当接收方通告窗口为0时，发送方将不再发送数据，直到接收方通告窗口大于0；
    - 但是后续接收方通告窗口大小是通过ACK进行的，然而TCP不对未带数据的ACK进行确认
    - 如果这个ACK丢包了怎么办？-> 可能会进入“死锁”
* 为了解决前述提到的问题，TCP在发送端维护一个**坚持(persist)定时器**以周期性地向接收端查询通告窗口的大小

## 保活定时器
* 保活不是TCP规范中的一部分，但是许多实现提供了保活定时器
    - 保活功能主要为服务器程序提供的；但客户机也可以开启这个功能
    - 用于主动关闭服务器上存在的半连接【客户机非正常宕机引起】

* 服务器端保活探测的结果
    - 如果客户机正常，则保活正常，应用程序无感
    - 如果客户机崩溃，服务器将收不到对探查的响应，服务器应用程序将收到read接口返回的差错信息
    - 如果客户机关闭后重启，这时客户机会给服务器发回探查响应，但是该响应为RST，使得服务器终止该连接
    - 如果客户机对于服务器来说不可达，类似于第二种情况

* 这里注意到：
    - 客户机程序被终止，并且发出了一个FIN，如果该FIN丢包，会重传
    - 但是整个时间可能会比较长，而客户宿主机上对客户机程序的关闭就导致TCP连接终止异常
    - 进而可能导致服务器上存在半连接
