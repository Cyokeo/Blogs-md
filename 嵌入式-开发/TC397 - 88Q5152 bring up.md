---
title: TC397 - 88Q5152 bring up
categories: 嵌入式-开发
---
## 问题梳理
1. MAC -> 5152 switch rgmii port `vs` MAC -> Phy
	1. Clause 22/45
	2. 即与配置Transceiver【Phy】的区别
	3. 流程： switch上需要额外的port配置【MII配置为Phy模式？】
2. Autosar/User初始化流程
	1. 结合代码