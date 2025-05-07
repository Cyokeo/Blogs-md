---
title: 论文（六）- ESP_ILP
categories: 嵌入式-开发
---
## 论文写作
1. 建立、给出每个约束时，除了说出他的作用，还要举出具体的例子进行解释
2. `Order protection band constraint`似乎需要包含端节点的发送抖动！！！
3. OPB约束的描述中引入了M3/M4与其它约束的描述不一致 -> 需要修正一下，保持一致
	1. 论文中的描述是对的，但是Gurobi中没有或的约束；需要将`或`转化为`且`，就需要用到大常数M3/M4等
	2. 要注意适当增大M3/M4等大常量的值

## 深入复习
1. 之前的版本是对的；考虑了不同的传输延时、传播延迟
2. 暂时将帧间隔的长度设置为0 -> 还是不行，效果反而更差了（~/Workspace/omnetpp/inet4/src/inet/linklayer/ethernet/Ethernet.h/line:33）
3. 抖动更小时 - 2us级，Degrade对Qbv的影响越显著；Debug原因分析如下：
	1. 端节点发Qbv包时被普通的包打断造成上述问题

## 证明解空间增大
1. FIC + Forwarding Constraint可以推导出OPB
2. 但是OPB推导不出FIC
3. 从仿真结果来看，“顺序保护带约束”是有用处的！！！