---
title: aarch64-linux-内存管理（一）
date: 2024-06-26 11:23:05
tags:
categories:
- [Linux相关, 内存管理]
---

## 优质参考博客
1. [Linux内存管理-专栏](https://blog.csdn.net/yhb1047818384/category_10345494.html)
2. [Linux内存模型](http://www.wowotech.net/memory_management/memory_model.html)
3. [arm64架构linux内核地址转换__pa(x)与__va(x)分析](https://www.cnblogs.com/liuhailong0112/p/14465697.html)
4. [底层开发必知的三个内存结构概念](https://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NDQ0NA==&mid=2247485533&idx=1&sn=bf4dc798fc2cbbe0b55dcd0f5360d933&chksm=cee9878ef99e0e98628bd41f0a733f47d955e470d26840d0f07cc5aa7c9f789e746133c39c82&scene=178&cur_album_id=2707075920913924097#rd)


## 内存硬件结构

Linux 把物理内存划分为三个层次来管理: 存储节点(Node)、内存管理区(Zone)和页面(Page)
- Node -> struct pglist_data，包含的重要信息有
    - 该 Node 包含的Zone数目
    - 该node中内存的起始页帧号
    - 该node地址范围内的实际管理的页面数量
    - 该node地址范围内的所有页面数量，包括空洞的页面
    - ZONE_PADDING宏：让前后的成员分布在不同的cache line中, 以空间换取时间
- Zone -> struct Zone
    - 将node拆分成zone主要还是出于Linux为了兼容各种架构和平台，对不同区域的内存需要采用不同的管理方式和映射方式；32位系统中，内核和用户地址空间按1:3划分，内核地址空间只有1GB，所以不能把1GB以上的内存直接映射到内核地址空间，因此就把不能直接映射的内存划分到了高端内存区
    - ZONE_DMA: 只适用于Intel x86架构，ARM架构没有这个区域，用于ISA设备的DMA操作，物理地址范围为0-16MB
    - ZONE_DMA32: 在64位的系统上使用32位地址寻址的适合DMA操作的内存区。例如在AMD64系统上，该区域为低4GB的空间。在32位系统上，本区域通常是空的
    - ZONE_NORMAL: 指的是<u>可以直接映射到内核空间的内存</u>。也常称为“普通区域”“直接映射区域”“线性映射区域”。所谓线性映射就是物理地址和映射后的虚拟地址存在一种简单的关系，即虚拟地址=物理地址+固定偏移。在32位系统上，内核空间和用户空间按1:3划分，那么这个固定偏移就是：`0xC0000000` - 物理内存起始地址。因此可以看到：在32位系统中，将物理内存地址的低1G[物理内存起始地址(start): start+1G]映射到内核空间[0xc0000000:0xffffffff]
    - ZONE_HIGHMEM: 高端内存区，32位时代的产物。在32位系统上，指的是高于`896M`的物理内存。32位系统中，内核和用户地址空间按1:3划分，内核地址空间只有1GB，所以不能把1GB以上的内存直接映射到内核地址空间，因此就把不能直接映射的内存划分到了高端内存区。要将高于896MB的物理内存映射在内核空间的话，需要通过单独的映射来完成，并且这类映射不能保证物理地址和虚拟地址之间存在固定的对应关系（例如ZONE_NORMAL的固定偏移）
    > 64位系统中没有这个区域，即没有高端内存。因为64系统的内核虚拟地址空间非常大，不再需要高端内存区域
    - 指向所属的Node节点
    - 空闲内存链表，用于实现伙伴系统
- Page -> struct page
    - Linux内核使用page结构体来描述一个物理页面，每一个page frame有一个一一对应的page数据结构，系统中定义了page_to_pfn和pfn_to_page的宏用来在page frame number和page数据结构之间进行转换，具体如何转换是和[memory modle](http://www.wowotech.net/memory_management/memory_model.html)相关
    - PFN是page frame number的缩写，所谓page frame，就是针对物理内存而言的，把物理内存分成一个个的page size的区域，并且给每一个page 编号，这个号码就是PFN。假设物理内存从0地址开始，那么PFN等于0的那个页帧就是0地址（物理地址）开始的那个page。假设物理内存从x地址开始，那么第一个页帧号码就是（x>>PAGE_SHIFT）
- 区分系统物理地址空间 VS 内存占据的物理地址空间
    - 整个系统的物理地址空间并不是都用于内存，有些也属于I/O空间（当然，有些cpu arch有自己独立的io address space）。因此，内存所占据的物理地址空间应该是一个有限的区间，不可能覆盖整个物理地址空间

1. UMA 与 NUMA
    - UMA: Uniform Memory Access，统一内存访问，每个CPU共享相同的内存地址空间
    - NUMA: Non-Uniform Memory Access，非统一内存访问。系统中会有很多的内存节点和多个CPU簇， 所有节点中的CPU可以访问全部的物理内存，但是CPU访问本地的节点速度远快于访问远端的内存节点的速度


## 启动阶段

主要问题有：
1. 内核如何知道系统的物理内存信息？
    - DTB 方式，物理内存信息会写到DTB image中，内核在启动初期对DTB进行解析，得到物理内存信息
    - ACPI 方式，会在BIOS中写入物理内存信息

2. 内核启动初期有一部分汇编编写的位置无关码，它们主要做了什么事情？
    - 内核绝大部分的代码都不是位置无关码，且其运行时地址基本为虚拟地址，因此在执行到内核主体部分代码时，需要开启MMU，启动虚拟化。开启虚拟化就需要提供页表
    - 因此，在进入start_kernel()之前的初始阶段汇编代码会进行两个页表映射：
        - `identity mapping`：VA和PA相等的一段映射，主要目的就是为了打开MMU。<u>在打开mmu之前，cpu访问的都是物理地址，打开mmu访问的就是虚拟地址</u>，其实真正打开mmu的操作就是往某个system register的某个bit写1， 如果在开启mmu之前已经下发了某一个数据的操作指令，本来它是想访问物理地址的，结果mmu打开导致访问了虚拟地址，这样会造成混乱。 所以为了解决这一个情况，引入了identity mapping。VA = PA， 打开mmu前后，无论访问物理地址还是虚拟地址，都是对应同一段物理内存
        - `kernel image mapping`：内核镜像映射，主要目的是为了执行内核代码。打开了MMU后，内核需要运行起来，就需要将kernel运行需要的地址（kernel txt、rodata、data、bss等等）进行映射。映射到的虚拟地址为：内核编译时指定（计算出）的虚拟地址（***存疑❓***）
`idmap_pg_dir`是identity mapping用到的页表，`init_pg_dir`是kernel_image_mapping用到的页表。这两个页表定义在arch/arm64/kernel/vmlinux.lds.S中，同样定义在该文件中的还有另外三个页表`reserved_ttbr0`，`tramp_pg_dir`， `swapper_pg_dir`。
- reserved_ttbr0：是内核访问用户空间需要用的页表。
- tramp_pg_dir：适用于映射kaslr的内核区域
- <u>swapper_pg_dir</u>：在内核启动期间进行常规映射后，用作内核页表。（在4.20的内核之前其实是没有init_pg_dir这个概念的，arm64/mm: Separate boot-time page tables from swapper_pg_dir添加了启动时pgd的init_pg_dir）这几个页表的位置、大小在内核链接文件中都有定义。
- 使用init_pg_dir，是因为处理FDT的内核代码，后续的内核代码比较大；但是物理内存还没有扫描完成，进行不了最终swapper页表的建立

3. 初期的kernel image mapping （init_pg_dir）是如何进行（初始化这个页表）的？
    - Linux代码如下
    ```armasm
    /*
     * Map the kernel image (starting with PHYS_OFFSET).
     */
    adrp    x0, init_pg_dir
    mov_q   x5, KIMAGE_VADDR + TEXT_OFFSET   // compile time __va(_text)
    add x5, x5, x23           // add KASLR displacement
    mov x4, PTRS_PER_PGD
    adrp    x6, _end           // runtime __pa(_end)
    adrp    x3, _text          // runtime __pa(_text)
    sub x6, x6, x3            // _end - _text
    add x6, x6, x5            // runtime __va(_end)
 
    map_memory x0, x1, x5, x6, x7, x3, x4, x10, x11, x12, x13, x14
    ```
    ❗️ KIMAGE_VARDDR 即为内核映像的虚拟空间开始地址。这个值也是在编译时指定的【或者可以计算出的】
    ❗️ TEXT_OFFSET 即为内核代码段相对于内核虚拟地址起始位置的偏移
    ❗️ <font color=#DC143C>adrp 指令用于获取标号的运行时物理地址【借助运行当前指令时的PC值】</font>
    - [TEXT_OFFSET](https://stackoverflow.com/questions/51763634/why-physical-address-of-aarch64-kernel-image-is-nonnegative)
    ```
    // in file /arch/arm64/kernel/vmlinux.lds.S
    . = KIMAGE_VADDR + TEXT_OFFSET; 
    .head.text : {                          
    _text = .;
    HEAD_TEXT
    }
    ```

    这里，TEXT_OFFSET是一个随机值，因此每次编译时，内核代码的偏移都不是固定的（出于安全的考虑）。最新Linux代码中已经不使用TEXT_OFFSET了。虚拟地址随机化完全依赖于kaslr_offset。init_pg_dir页表的初始化过程也稍有变化：__primary_switch -> __pi_early_map_kernel()[这个函数似乎就是：early_map_kerne()]
    ```
    . = KIMAGE_VADDR;

	.head.text : {
		_text = .;
		HEAD_TEXT
	}
	.text : ALIGN(SEGMENT_ALIGN) {	/* Real text segment		*/
		_stext = .;  
        ......
    }
    
    // in /arch/arm64/kernel/head.S  

    SYM_FUNC_START_LOCAL(__primary_switch)
	adrp	x1, reserved_pg_dir
	adrp	x2, init_idmap_pg_dir
	bl	__enable_mmu

	adrp	x1, early_init_stack
	mov	sp, x1
	mov	x29, xzr
	mov	x0, x20				// pass the full boot status
	mov	x1, x21				// pass the FDT
	bl	__pi_early_map_kernel		// Map and relocate the kernel

	ldr	x8, =__primary_switched
	adrp	x0, KERNEL_START		// __pa(KERNEL_START)
	br	x8
    SYM_FUNC_END(__primary_switch)
    ```

    可以看到，在初始化init_pg_dir时，mmu已经开启了。需要注意：
    - ❗️`bl __pi_early_map_kernel`: BL: Branch with Link branches to a PC-relative offset
    - ❗️`br x8`: “Adding an L to the B or BR instructions turns them into a branch with link. This means that a
    return address is written into LR (X30) as part of the branch.” 可以看出B，BR是两个不同的指令。
    - ❗️ 因此，`br x8`将__primary_switched标号的虚拟地址赋给PC，从前面的链接文件可以看到，其虚拟地址处于KIMAGE_VADDR开始之后的位置。因此，之后Linux内核将运行于高地址的虚拟内存空间。

    > [from 《Armv8-A Instruction Set Architecture》]： The unconditional branch instruction B <label> performs a direct, PC-relative, branch to <label>. The offset from the current PC to the destination is encoded within the instruction. The range is limited by the space available within the instruction to record the offset and is +/- 128MB. When you use BR <Xn>, BR performs an indirect, or absolute, branch to the address specified in Xn.

    ```
    // in file /arch/arm64/kernel/pi/map_kernel.c

    asmlinkage void __init early_map_kernel(u64 boot_status, void *fdt)
    {
        u64 va_base, pa_base = (u64)&_text;
        u64 kaslr_offset = pa_base % MIN_KIMG_ALIGN;

        map_fdt((u64)fdt);

        /*
        * The virtual KASLR displacement modulo 2MiB is decided by the
        * physical placement of the image, as otherwise, we might not be able
        * to create the early kernel mapping using 2 MiB block descriptors. So
        * take the low bits of the KASLR offset from the physical address, and
        * fill in the high bits from the seed.
        */
        if (IS_ENABLED(CONFIG_RANDOMIZE_BASE)) {
            u64 kaslr_seed = kaslr_early_init(fdt, chosen);

            if (kaslr_seed && kaslr_requires_kpti())
                arm64_use_ng_mappings = true;

            kaslr_offset |= kaslr_seed & ~(MIN_KIMG_ALIGN - 1);
        }

        if (IS_ENABLED(CONFIG_ARM64_LPA2) && va_bits > VA_BITS_MIN)
            remap_idmap_for_lpa2();

        va_base = KIMAGE_VADDR + kaslr_offset;
        map_kernel(kaslr_offset, va_base - pa_base, root_level);
    }
    ```

    - kaslr_offset:
    这里相当于给内核运行时虚拟地址加了一个随机的偏移，因此后续在map_kernel中需要对内核进行重定位。<u>物理地址随机化比较好处理，虚拟地址随机化之后，内核大部分代码都需要重定位</u>。

4. 内核如何去解析这些配置信息？
    - 现在虽然MMU已经打开，kernel image的页表已经建立，但是内核还没有为DTB这段内存创建映射，现在内核还不知道内存的布局，所以内存管理模块还没能初始化。这个时候就需要用到fixmap。即将DTB的物理地址映射到Fixed map中的区域，然后访问该区域中的虚拟地址即可。
    - 解析DTB获取系统的物理内存信息，并保存到 ***memblock*** 结构中，这是一个全局的变量，用于管理内核早期启动阶段过程中的所有物理内存。



## 顺序记录

### aarch64-内核内存布局
- 0-256T -> 用户空间；256-512T -> 内核空间

#### [fixed mappings[4124KB]](https://www.cnblogs.com/alantu2018/p/8447570.html)

固定映射区，这部分的虚拟地址在编译阶段就已经确定。 在内核的启动过程中，有些模块需要使用虚拟内存并mapping到指定的物理地址上。而且，这些模块也没有办法等待完整的内存管理模块初始化之后再进行地址映射。因此，linux kernel固定分配了一些fixmap的虚拟地址，这些地址有固定的用途。使用该地址的模块在初始化的时候，将这些固定分配的地址mapping到指定的物理地址上去，进而可以通过虚拟化访问到需要访问的特定物理内存。[参考](https://www.cnblogs.com/alantu2018/p/8447570.html)

***个人理解***：例如在处理DTB时，内核初始部分代码将启动参数给出的DTB物理地址映射到fixed mappings虚拟空间区域，进而在开启虚拟化之后，实现对DTB的解析

***fixed address***的具体位置，对于aarch64架构，当前内核中这部分对应的虚拟地址范围为`[fffffdfffe5f9000, fffffdfffe9fffff]`，总共4124KB大小

***fixed address***又分为两大类：永久映射 & 临时映射

1. 永久映射：用于具体的某个内核模块，使用关系是永久的。涉及到的模块主要有：
- DTB解析模块：
- early Console 模块：kernel启动阶段初期可以使用的consol，可以用于输出各种调试信息
- 动态打补丁的模块：使用fixed address映射具有RW属性的代码段，进而动态修改这部分代码段的部分内容


2. 临时映射：各个内核模块都可以使用，用完之后就释放。主要用于early ioremap模块

### aarch64虚拟地址->物理地址

aarch64有两个页表基地址寄存器：
- ttbr0：用户空间页表基地址。启动初期，idmap_pg_dir填入ttbr0。这里将内核代码映射到低虚拟地址空间。
- ttbr1：内核空间页表基地址。启动初期，init_pg_dir填入ttbr1。这个页表将内核代码映射到高虚拟地址空间。

64bit的虚拟地址并不是所有bit都被用上的。目前有效的VA_BITS的配置是：36, 39, 42, 47。假设我现在使用64K的页和42bit的虚拟地址空间， 使用三级页表。地址转换过程[举例](https://blog.csdn.net/yhb1047818384/article/details/108210044)：
1. 如果VA[63：42] = 1, 那么就会使用ttbr1的地址作为一级页表的基地址；如果VA[63:42] = 0, 那么就会使用ttbr0的地址作为一级页表的基地址，那么就会使用ttbr0的地址作为一级页表的基地址；
2. VA[41:29]放置Level 1页表中的索引，从而找到对应的描述符地址并获取描述符内容，<u>根据描述符中的内容获取Level 2页表基地址</u>
3. VA[28:16]放置Level 2页表中的索引，从而找到对应的描述符地址并获取描述符内容，根据描述符中的内容获取<u>物理地址的高36位</u>，以4K地址对齐
4. VA[15: 0]放置的是物理地址的偏移，结合获取的物理地址高位，最终得到物理地址


