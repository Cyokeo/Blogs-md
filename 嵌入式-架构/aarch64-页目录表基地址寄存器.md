---
title: aarch64-页目录表基地址寄存器
categories: 嵌入式-架构
---
每个异常等级【EL1，EL2，EL3】都有自己的页目录表基地址寄存器：
`EL0/EL1`：有两个寄存器TTBR0，TTBR1；其中当虚拟地址的前几位都是0时，TTBR0所指向的映射表被选中（对应Linux系统的用户空间）；当VA的前几位都是1时，TTBR1指向的映射表被选中（对应Linux系统的内核空间）
`EL2`：只有一个TTBR0，只能使用范围：0x0 ~ 0x0000ffff_ffffffff
`EL3`：只有一个TTBR0，一样