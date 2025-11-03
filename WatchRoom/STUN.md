# 参考文档
- https://www.cnblogs.com/ssyfj/p/14797389.html
- [p2p通信原理——STUN 协议详解](https://zhuanlan.zhihu.com/p/1890724663333736762)：精品文章
- https://zhuanlan.zhihu.com/p/583022591
- https://ipw.cn/doc/ipv6/user/enable_ipv6.html

最初的解决方案是 `RFC 3489` —— STUN (Simple Traversal of User Datagram Protocol (UDP)Through Network Address Translators (NATs)) 一个简单的基于 UDP 进行 NAT 穿越通信的工具。

但是 `RFC 3489` 毕竟只是一个 `Simple` 的玩具，不足以应用于诸多复杂的生产环境，比如对 NAT 类型进行的归类 (full core, restricted cone, port restricted cone, symmetric ) 并不能够完全描述现实中存在的 NAT 设备行为。

于是重新设计了 `RFC 5389` —— STUN (Session Traversal Utilities for NAT)。从一个 `Simple Traversal` 简单的穿越 变成了 `Session Traversal Utilities` 会话穿越工具集。而 `RFC 8489` 只是在 `RFC 5389` 的基础上增加了一些补充，比如新协议支持，额外的安全保障等等。

`STUN` 下有很多的工具，比如：

- `RFC 5780`: NAT 行为发现 (NAT Behavior Discovery)
- `RFC 8445`: 交互式连接建立 (ICE -- [Interactive Connectivity Establishment](https://zhida.zhihu.com/search?content_id=255896212&content_type=Article&match_order=1&q=Interactive+Connectivity+Establishment&zhida_source=entity))
- `RFC 8656`: 基于中继的 NAT 穿越 (TURN -- Traversal Using Relays around NAT)
- ......

# 一：STUN协议介绍

## （一）STUN协议简介

![](https://img2020.cnblogs.com/blog/1309518/202105/1309518-20210521192002366-300585819.png)

STUN 存在的目的就是进行NAT穿越，NAT有四种类型，每种类型如何穿越，它的基本原理是什么，都是属于STUN协议中的一部分。  

## （二）RFC STUN规范

RFC STUN规范中，实际上有两套STUN规范：
目前，又有一个新的规范了RFC 8489 (2020)

### 规范一：RFC3489（2003）

![](https://img2020.cnblogs.com/blog/1309518/202105/1309518-20210521192256344-1903236356.png)

**STUN的全称是**Simple Traversal of User Datagram Protocol (UDP) Through Network Address Translators (NATs)**，即穿越NAT的简单UDP传输，**

**是一个轻量级的协议，允许应用程序发现自己和公网之间的中间件类型，同时也能允许应用程序发现自己被NAT分配的公网IP。**

**它就是将STUN定义成简单的通过UDP进行NAT穿越的一套规范，也就是告诉你如何一步一步通过UDP进行穿越**，但是这套规范在穿越的过程中还是存在很多问题，尤其是现在的网络

路由器对UDP的限制比较多，有的路由器甚至不允许进行UDP传输，所以这就导致了我们通过RFC3489这套规范进行NAT穿越的时候它的失败率会非常高。所以为了解决这个问题，

又定义了另一套标准，RFC5389.

### 规范二：RFC5389（2008）

![](https://img2020.cnblogs.com/blog/1309518/202105/1309518-20210521194056150-1602839262.png)

1.RFC5389中，STUN的全称为**Session Traversal Utilities for NAT**，即NAT环境下的会话传输工具，是一种处理NAT传输的协议，但主要作为一个工具来服务于其他协议。

和STUN/RFC3489类似，可以被终端用来发现其公网IP和端口，同时可以检测端点间的连接性，也可以作为一种保活（keep-alive）协议来维持NAT的绑定。

和RFC3489最大的不同点在于，**STUN本身不再是一个完整的NAT传输解决方案，而是在NAT传输环境中作为一个辅助的解决方法，同时也增加了TCP的支持。**

RFC5389废弃了RFC3489，因此后者通常称为**classic STUN**，但依旧是后向兼容的。而完整的NAT传输解决方案则使用STUN的工具性质，[ICE](http://www.rfc-editor.org/info/rfc5245)就是一个基于[offer/answer](http://www.rfc-editor.org/info/rfc3264)方法的完整NAT传输方案，如[SIP](http://www.rfc-editor.org/info/rfc3261)。

2.RFC5389是在RFC3489的基础上又增加了一些功能，但是它对整个STUN的描述就不一样了, 它是把STUN描述成一系列穿越NAT的工具，所以都叫STUN，但是他们的含义完全就不一样了。

**RFC5389在UDP尝试可能失败的情况下，尝试使用TCP，也就是说RFC5389是包括UDP和TCP的两种协议进行NAT穿越的，这是两套规范最本质的区别**。当然在协议的具体内容上，包括协议头还有协议体中的属性都有很多的变化，但是那些都不是最关键的，最关键的是RFC5389里面将TCP纳入进来。你可以通过TCP进行穿越。

# 二：STUN协议详解

STUN是一个C/S架构的协议，支持**两种传输类型**：

一种是请求/响应（request/respond）类型，由客户端给服务器发送请求，并等待服务器返回响应；

另一种是指示类型（indication transaction），由服务器或者客户端发送指示，另一方不产生响应。

两种类型的传输都包含一个96位的随机数作为**事务ID**（transaction ID），

对于请求/响应类型，事务ID允许客户端将响应和产生响应的请求连接起来；
对于指示类型，事务ID通常作为debugging aid使用。

所有的STUN报文信息都含有一个**固定头部（类型+长度+事务ID，3个字段），包含了方法，类和事务ID**。

**方法表示是具体哪一种传输类型---由type字段剩下的12位决定**（两种传输类型<RFC3489中有两种>又分了很多具体类型），STUN/RFC5389中只定义了一个方法，即binding（绑定），其他的方法可以由使用者自行拓展；

Binding方法可以用于请求/响应类型和指示类型，用于前者时可以用来确定一个NAT给客户端分配的具体绑定，用于后者时可以保持绑定的激活状态。**类表示报文类型是请求/成功响应/错误响应/指示---由（C0C1）决定**。

在固定头部之后是零个或者多个属性（attribute），长度也是不固定的。

下面我们就具体看看这个STUN协议：

![](https://img2020.cnblogs.com/blog/1309518/202105/1309518-20210521194444819-362205428.png)

STUN这个协议它是包括了消息头和消息体，消息头是***20字节固定***的消息头，Body中可以有0个或者多个Attribute属性，后面我们会介绍属性的作用。

### 消息头的格式：
![[Pasted image 20251012113743.png]]

上图的格式是最新的RFC5389的格式，刚刚我们上面说三个的是RFC3489，那么RFC5389和RFC3489之间有什么区别呢？
1. RFC5389最新的协议要求**消息类型的最低两位必须是0 0** ；
2. 事物ID：RFC3489里面是128位事物ID；而RFC5389里面是 96位，其中有32位单独划出来了单独作为 Magic Cookie，一个魔法数。
**以上两点**就是RFC3489和5389的STUN消息头的区别。

### 消息体

下面再看STUN Message Body 消息体，***消息头后有0或多个属性***。每个属性都使用TLV(动态)编码：Type,Length,Value

![](https://img2020.cnblogs.com/blog/1309518/202105/1309518-20210521205824187-69610064.png)
### Length字段：
Value这个值是可以变化的，变化的程度怎么知道有多长呢？**通过这个Length，这个Length标示了Value的长度，最终的消息是一个32位对齐的，如果最后不是32位对齐，要通过补0来达到对齐，这是整个Body。**
![[Pasted image 20251012113645.png]]


字段存储了信息的长度，以字节为单位，不包括20字节的STUN头部。由于所有的STUN属性都是都是4字节对齐（填充）的，因此这个字段最后两位应该恒等于零，这也是辨别STUN包的一个方法之一。

### Type字段：

为属性的类型。任何属性类型都有可能在一个STUN报文中出现超过一次。

除非特殊指定，否则其出现的顺序是有意义的：即只有第一次出现的属性会被接收端解析，而其余的将被忽略。为了以后版本的拓展和改进，属性区域被分为两个部分。

Type值在**0x0000-0x7FFF之间的属性被指定为强制理解，意思是STUN终端必须要理解此属性，否则将返回错误信息；**

**而0x8000-0xFFFF之间的属性为选择性理解，即如果STUN终端不识别此属性则将其忽略。**

目前STUN的属性类型由IANA维护。

### 这里列举定义的属性Type：

![](https://img2020.cnblogs.com/blog/1309518/202105/1309518-20210521210011089-1051018264.png)