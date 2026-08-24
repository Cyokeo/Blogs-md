# 8 种内存访问属性
访问属性索引会被写到页表项中

# 地址转换控制寄存器
[tcr_el1](http://developer.arm.com/documentation/ddi0601/2025-09/AArch64-Registers/TCR-EL1--Translation-Control-Register--EL1-)

# 两个页表基址寄存器
```asm
/* Setup page tables */
adrp x8, _boot_pgd_down
msr ttbr0_el1, x8
adrp x8, _boot_pgd_up
msr ttbr1_el1, x8
```
## ttbr0_el1
## ttbr1_el1
`ttbr_el0/1`中的ASID（地址空间标识符）位用于在MMU（内存管理单元）的TLB（[快表](https://www.google.com/search?q=%E5%BF%AB%E8%A1%A8&sca_esv=c4f930c117b1d5e6&sxsrf=AE3TifPnwxHjnWVi_cMJZSRswNJK_VPstw%3A1762592695779&ei=twcPaf6pL8rm1e8PvOS40QE&ved=2ahUKEwikjK6VmuKQAxWOe_UHHZ5yM7YQgK4QegQIARAB&uact=5&oq=ttbr_el0%E4%B8%AD%E7%9A%84ASID%E4%BD%8D%E6%9C%89%E4%BB%80%E4%B9%88%E4%BD%9C%E7%94%A8&gs_lp=Egxnd3Mtd2l6LXNlcnAiJHR0YnJfZWww5Lit55qEQVNJROS9jeacieS7gOS5iOS9nOeUqDIIEAAYiQUYogQyBRAAGO8FMgUQABjvBTIFEAAY7wUyCBAAGIAEGKIESKxKUI4GWLxIcAl4AZABAJgBmQSgAckvqgEMMC4zMC4xLjEuMC4xuAEDyAEA-AEBmAIGoAKcBcICChAAGEcY1gQYsAOYAwCIBgGQBgiSBwMyLjSgB9QtsgcDMC40uAeUBcIHBTAuNS4xyAcM&sclient=gws-wiz-serp&mstk=AUtExfBaI48PlO05ATb3Dtb4Dd7ciqsFsJ4aq94XqOXJuTyDKqm9BGAxbIL5KTn6IGrjfmMGH2MDNId1IG3HtUVYo9Jq5twAVLxKBxynzCDefibadqFKz6nJ-VFgJLobPNUqU7WJY7KOf6j09OTeZn7sfvRvm5jpexqt5Gwy13TRPLVJKBb6OKf5IBVn3mdJwDRQ7Aoq7WsYBz5An17JQCjzsFuSRyYo76fCrx554Z7QUcw7dCCO5n9IdfSP4oOdNnOZOxumCkzCuNXInDWiXIsfywPi&csui=3)）中标记地址空间，以区分不同的进程。它的主要作用是实现TLB的标签化，从而提高地址转换效率并减少TLB的全局刷新。当进程切换时，只需更改ASID，而不必刷新整个TLB，从而有效隔离不同进程的地址空间和提高性能。 

- **地址空间标签**: ASID为一个进程分配一个唯一的标识符，用于标记其地址空间。
- **TLB缓存优化**: MMU可以将具有相同ASID的页表项缓存到TLB中，减少了每次地址转换时访问页表的需求。
- **提高进程切换效率**: 在进程切换时，只需更新ASID，而无需刷新整个TLB。这种方法可以有效隔离不同进程的地址空间，避免TLB的全局刷新开销，从而显著提高系统性能。
- **权限隔离**: ASID通过在TLB中提供权限隔离，防止了在不刷新TLB的情况下其他进程访问当前进程的地址空间。

>在 ARM64 中，**TTBR0_EL1 和 TTBR1_EL1 总是同时启用的**，关键在于如何配置 TCR_EL1 寄存器来控制它们的使用方式。以下是详细的配置方法：

## 关键配置字段
```c
// TCR_EL1 寄存器布局（相关字段）
#define TCR_T0SZ_SHIFT   0       // TTBR0 区域大小偏移
#define TCR_T1SZ_SHIFT   16      // TTBR1 区域大小偏移
#define TCR_TG0_SHIFT    14      // TTBR0 粒度大小
#define TCR_TG1_SHIFT    30      // TTBR1 粒度大小
#define TCR_EPD0_SHIFT   7       // TTBR0 使能位
#define TCR_EPD1_SHIFT   23      // TTBR1 使能位
```

# tcr_el1寄存器
- 控制MMU地址转换
## ASID
```c
// 设置 ASID (Address Space ID) 为 16 位
TCR_ASID16 = (1 << 36)
```
- **ASID 范围**: 0-65535，支持 65536 个不同的地址空间
- **意图**: 支持大量并发进程，减少 TLB 刷新
