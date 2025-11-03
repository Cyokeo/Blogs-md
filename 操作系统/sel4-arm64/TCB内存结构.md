### TCB结构前面部分是几个slot
![[d0f4ab3d687ef10e483d006ff94b1912.png]]

### TCB有哪几个slot（cnode table entry）
![[b9a7eeb748d7bb85ff634688891c60ba.png]]

#### initial thread cnode -> tcbCTable
try_init_kernel()中会将initial thread 的root cnode插入到TCB结构的tcbCTable能力槽上。其中从root cnode可以访问到初始线程的所有能力，其能力槽结构如下：
```c
// 初始线程的根 CNode 是一个能力数组
cap_t root_cnode[seL4_NumInitialCaps] = {
    [0]  = Null Capability,           // seL4_CapNull
    [1]  = InitThread TCB Cap,        // seL4_CapInitThreadTCB  
    [2]  = InitThread CNode Cap,      // seL4_CapInitThreadCNode
    [3]  = InitThread VSpace Cap,     // seL4_CapInitThreadVSpace
    [4]  = IRQ Control Cap,           // seL4_CapIRQControl
    [5]  = ASID Control Cap,          // seL4_CapASIDControl
    [6]  = InitThread ASID Pool Cap,  // seL4_CapInitThreadASIDPool
    [7]  = IO Port Control Cap,       // seL4_CapIOPortControl
    [8]  = IO Space Cap,              // seL4_CapIOSpace
    [9]  = BootInfo Frame Cap,        // seL4_CapBootInfoFrame
    [10] = InitThread IPC Buffer Cap, // seL4_CapInitThreadIPCBuffer
    [11] = Domain Control Cap,        // seL4_CapDomain
    [12] = SMMU SID Control Cap,      // seL4_CapSMMUSIDControl
    [13] = SMMU CB Control Cap,       // seL4_CapSMMUCBControl
    [14] = InitThread SC Cap,         // seL4_CapInitThreadSC
    [15] = SMC Cap,                   // seL4_CapSMC
};
```
因此，后续可以从tcbCTable node 访问到这些能力槽，进而获取到能力cap

### 这几个slot是配置相关的
1. CTable指明了TCB的Cnode（也即thread的CSpace）

### TCB也可以表示为一个cap
- 且cap中记录了tcb_t (b)的地址
- 一个cap由两个字组成

### 调度
1. 可抢占、tickless