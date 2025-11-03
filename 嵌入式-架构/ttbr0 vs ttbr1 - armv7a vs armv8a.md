
### 如何确定ttbr0/1的管理范围
1. armv7a通过ttbcr寄存器的N[链接](https://developer.arm.com/documentation/ddi0406/c/System-Level-Architecture/Virtual-Memory-System-Architecture--VMSA-/Short-descriptor-translation-table-format/Selecting-between-TTBR0-and-TTBR1--Short-descriptor-translation-table-format?lang=en)
2. armv8a使用tcr寄存器的t0sz/t1sz字段[链接](https://blog.csdn.net/dai_xiangjun/article/details/120132635)


### armv8a
1. ttbr0 寄存器的管理范围总是从0开始
	1. t0sz == 16，在0x0000_ffff_ffff_ffff处结束，总大小为256T
2. ttbr1 寄存器的管理范围总是在0xffff_ffff_ffff_ffff处结束
	1. t1sz == 16，在0xffff_0000_0000_0000处开始，总大小256T
3. txsz增大时，管理的虚拟内存范围将减小，例如等于16时，管理的虚拟内存范围为
	`64 - 16 = 48`位，即48位的虚拟地址有效