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
- seL4_CapInitThreadVSpace
	- 可以通过这个能力槽获取PGD table的virtual address，进而获得其PGD表的物理基地址，进而可以设置ttbr寄存器，进行对应进程的专属内存映射行为
	- 同样的，对应不同层级的转换表，都有一个对应的vspace_cap。都可以从该cap中获取对应表的virtual address
- seL4_CapBootInfoFrame
	- 存储了bootinfo frame的userland virtual addr和kernelland virtual address「pptr」
- seL4_CapInitThreadASIDPool
	- 记录了init thread的ASID
	- 和一片区域【one Page】的 pptr_t 地址信息；这片区域后续会被当做asid_pool类型进行解引用；其中含有一个asid_map_t类型数组，用于记录所有ASID对应的asid_map_t信息
	```c
	ap->array[ASID_LOW(IT_ASID)] = asid_map;
	armKSASIDTable[ASID_HIGH(IT_ASID)] = ap;
	```
	- ASID与地址转换的TLB相关
	- asid_map_t中记录了PGD基地址信息
	

### 这几个slot是配置相关的
1. CTable指明了TCB的Cnode（也即thread的CSpace）

### TCB也可以表示为一个cap
- 且cap中记录了tcb_t (b)的地址
- 一个cap由两个字组成

### 调度
1. 可抢占、tickless