# 参考
1. [virtio 协议原文](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-100002)
2. [较好的blog](https://jiaweihawk.github.io/2024/08/23/virtio%E7%AE%80%E4%BB%8B/#%E5%89%8D%E8%A8%80)

根据[virtio标准2.6.](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-270006)，**virtqueue**由**descriptor table**、**available ring**和**used ring**构成
# descriptor table
**descriptor table**指的是驱动为设备准备的buffer，其中每个元素形式如[virtio标准2.7.5.](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-430005)中定义

```c
struct virtq_desc {   
        /* Address (guest-physical). */   
        le64 addr;   
        /* Length. */   
        le32 len;   
   
/* This marks a buffer as continuing via the next field. */   
#define VIRTQ_DESC_F_NEXT   1   
/* This marks a buffer as device write-only (otherwise device read-only). */   
#define VIRTQ_DESC_F_WRITE     2   
/* This means the buffer contains a list of buffer descriptors. */   
#define VIRTQ_DESC_F_INDIRECT   4   
        /* The flags as indicated above. */   
        le16 flags;   
        /* Next field if flags & NEXT */   
        le16 next;   
};
```

其中，每个描述符描述一个buffer，**addr**是**guest**的物理地址。描述符可以通过**next**进行链式连接，其中每个描述符描述的buffer要么是设备只读guest只写的，要么是设备只写guest只读的(但无论那种其描述符都是设备只读的)，但一个描述符链可以同时包含两种buffer

buffer的具体内容取决于设备类型，最常见的做法是包含一个设备只读头部表明数据类型，并在其后添加一个设备只写尾部以便设备写入

# available ring
驱动使用**available ring**将可用buffer提供给设备，其形式如[virtio标准2.7.6.](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-490006)所示

```c
struct virtq_avail {   
#define VIRTQ_AVAIL_F_NO_INTERRUPT      1   
        le16 flags;   
        le16 idx;   
        le16 ring[ /* Queue Size */ ];   
        le16 used_event; /* Only if VIRTIO_F_EVENT_IDX */   
};
```

其中，**ring**每个元素指向**descriptor table**中的描述符链，其仅由驱动写入，由设备读取

**idx**表示驱动下一个**ring**元素的位置，**仅由驱动维护**。除此之外，设备会维护一个**last_avail_idx**，表示设备使用过的最后一个**ring**元素的位置，即(last_avail_idx, idx)**是所有可用的**ring元素。

# used ring
类似的，设备使用**used ring**将已用buffer提供给设备，其形式如[virtio标准2.7.8.](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-540008)所示

```c
struct virtq_used {   
#define VIRTQ_USED_F_NO_NOTIFY  1   
        le16 flags;   
        le16 idx;   
        struct virtq_used_elem ring[ /* Queue Size */];   
        le16 avail_event; /* Only if VIRTIO_F_EVENT_IDX */   
};   
   
/* le32 is used here for ids for padding reasons. */   
struct virtq_used_elem {   
        /* Index of start of used descriptor chain. */   
        le32 id;   
        /*   
         * The number of bytes written into the device writable portion of   
         * the buffer described by the descriptor chain.   
         */   
        le32 len;   
};
```

其中，**ring**每个元素包含指向**descriptor table**中描述符链的**id**和设备实际写入的字节数**len**，其仅由设备写入，由驱动读取

**idx**表示设备将下一个**ring**元素的位置，仅由设备维护。除此之外，驱动会维护一个**last_used_idx**，表示驱动使用过的最后一个**ring**元素的位置，即**(last_used_idx, idx)**是所有可用的**ring元素

# device configuration space

**设备配置空间**通常用于那些很少更改或在初始化时设定的参数。不同于**PCI设备**的配置空间，其是设备相关的，即不同类型的设备有不同的**设备配置空间**，如**virtio-net**设备的**设备配置空间**如[virtio标准5.1.4.](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-2230004)所示而**virtio-blk**设备的**设备配置空间**如[virtio标准5.2.4.](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-2790004)所示。

# notifications

驱动和**virtio设备**通过**notifications**来向对方表明有信息需要传达，根据[virtio标准2.3.](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-180003)可知，共有三种：

- 设备变更通知
- 可用buffer通知
- 已用buffer通知

这些通知在不同的设备接口下有不同的表现形式

### 设备变更通知

设备变更通知是由设备发送给**guest**，表示前面介绍的[设备配置空间](https://jiaweihawk.github.io/2024/08/23/virtio%E7%AE%80%E4%BB%8B/#device-configuration-space)发生了更改。

一般是Qemu利用硬件机制注入设置改变MSIx中断

### 已用buffer通知

类似的，已用buffer通知也是由设备发送给**guest**，表示前面介绍的[used vring](https://jiaweihawk.github.io/2024/08/23/virtio%E7%AE%80%E4%BB%8B/#used-ring)上更新了新的已用buffer。

一般是Qemu注入对应的MSIx中断

### 可用buffer通知

可用buffer通知则是由**guest**驱动发送给设备的，表示前面介绍的[available vring](https://jiaweihawk.github.io/2024/08/23/virtio%E7%AE%80%E4%BB%8B/#available-ring)上更新了新的可用buffer。

一般是Qemu设置一段特定的**MMIO**空间，驱动访问后触发**vm_exit**退出到**kvm**后利用ioeventfd机制通知Qemu

# virtio transport

根据[virtio标准4.](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-1140004)可知，**virtio协议**可以使用各种不同的总线，因此**virtio协议**被分为通用部分和总线相关部分。即**virtio协议**规定都需要有[前面小节](https://jiaweihawk.github.io/2024/08/23/virtio%E7%AE%80%E4%BB%8B/#virtio%E5%8D%8F%E8%AE%AE)介绍的5个组件，但驱动和**virtio**设备如何设置这些组件就是总线相关的。其主要可分为**Virtio Over PCI Bus**、**Virtio Over MMIO**和**Virtio Over Channel I/O**，而**virtio-net-pci设备**自然属于是**Virtio Over PCI BUS**。

