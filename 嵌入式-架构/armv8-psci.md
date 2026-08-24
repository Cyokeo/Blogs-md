## 参考
1. https://www.cnblogs.com/arnoldlu/p/13993375.html
2. https://www.cnblogs.com/arnoldlu/p/14175126.html

## PSCI
参考：[PSCI的安全唤醒与TrustZone的密室协约](https://kernel.meizu.com/2025/04/10/Linux%20SMP%E5%90%AF%E5%8A%A8%E7%BD%97%E6%9B%BC%E5%8F%B2(%E4%B8%8B):PSCI%E7%9A%84%E5%AE%89%E5%85%A8%E5%94%A4%E9%86%92%E4%B8%8ETrustZone%E7%9A%84%E5%AF%86%E5%AE%A4%E5%8D%8F%E7%BA%A6/)
1. psci是运行在安全世界的EL3固件，非安全世界通过`SMC`（secure monitor call）调用陷入其中，实现ARM核的安全状态与非安全状态之间的切换，执行PSCI调用
2. 陷入Hypervisior的指令为`HVC`
3. 陷入EL1的指令为`SVC`
4. 需要注意的是EL3固件运行在MMU关闭的环境

从具体的PSCI调用可以看到，AP的执行入口为`secondary_startup()->core_entry()->non_boot_main()`

# EL执行等级与安全态/非安全态的区别
在ARMv8架构中使用执行等级（Execution Level，EL）**EL0～EL3来定义ARM核的运行等级**，其中***EL0～EL2等级分为安全态和非安全态***。ARMv8架构与ARMv7架构中ARM核运行权限的对应关系如图所示。EL3只有安全态。
需要注意的是：作为一个可选特性，*Armv8.4-A增加了安全世界下EL2的支持*。支持安全世界EL2的处理器，需配置EL3下的SCR_EL3.EEL2比特位来开启这一特性。设置了这一比特位，才允许使用安全状态下的虚拟化功能。
1. **非安全世界想要切换到安全世界，必须执行SMC指令**，陷入到EL3。安全世界状态和正常世界状态之间的切换是由bl31的固件完成;
2. ***无论从安全切换到非安全，还是非安全切换到安全都需要经过EL3***
3. 非安全部分包括：NS.EL0、NS.EL1、NS.EL2；安全部分包括：EL3、S.EL2(ARMv8.4新增)、S.EL1、S.EL0。

![[Pasted image 20260223123038.png]]

![[Pasted image 20260223125115.png]]
## 安全状态切换
无论从安全切换到非安全，还是非安全切换到安全都需要经过EL3
![[Pasted image 20260223125237.png]]

- Rich OS通过FIQ或者SMC异常进入EL3；
- EL3中执行对应的异常处理函数，并进行SCR_EL3.NS位设置1->0；保存非安全寄存器状态；恢复安全寄存器状态；
- ***从异常处理中退出并将CPU从EL3切换到S.EL1***。
## SMC异常
执行SMC产生SMC异常进入EL3。SMC被用于从EL3 Firmware或者TEE中请求服务。SMC  dispatcher将SMC请求发送给Firmware(PSCI)或者TOS处理。
![[Pasted image 20260223125632.png]]
## 问题
> 既然安全态也有EL1/EL2的区分，那怎么实现切换呢？
> 