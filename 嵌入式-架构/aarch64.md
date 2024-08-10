---
title: aarch64
categories: 嵌入式-架构
---
## 异常路由
==问题==： 如果某个异常发生，处理器应该使用那个人异常等级来处理这个异常？

### 中断路由规则
- 路由到EL3：需要设置`SCR_EL3.{EA, IRQ, FIQ}`
	```asm
	MRS X0, SCR_EL3 // 读取SCR_EL3到X0寄存器
	ORR X0, X0, #(1<<3) // 设置EA，将SError路由到EL_3
	ORR X0, X0, #(1<<1) // 设置IRQ
	ORR X0, X0, #(1<<2) // 设置FIQ
	MSR SCR_EL3, X0
	```
- 路由到EL_2：需要设置`HCR_EL2.{AMO, FMO, IMO}`，并清除`SCR_EL3.{}`
- 路由到EL_1：如果中断没有设置路由到EL_3，EL_2，默认路由到EL_1