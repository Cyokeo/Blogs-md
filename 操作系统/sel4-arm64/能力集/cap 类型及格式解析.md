以下是 seL4 中主要 capability 类型的内存布局格式：

## 通用格式说明
```txt
// 所有 capability 使用 128 位 (2个64位字)
// words[0]: [63:59] = 类型码, [58:0] = 类型特定数据
// words[1]: 类型特定数据继续
```

## 具体能力类型布局

### 1. **Null Capability**

```c
// Null capability:
// words[0]: [ type=Null | 全0 ]
// words[1]: 全0
// 用途：空能力，无任何权限
```

### 2. **Untyped Memory**

```c
// Untyped capability:
// words[0]: [ type=Untyped | 物理地址[43:0] | 保留位 ]
// words[1]: [ 大小位图 | 是否设备内存 | 其他属性 ]
// 用途：未类型化内存，可重类型化为其他对象
```

### 3. **TCB (Thread Control Block)**
```c
// TCB capability:
// words[0]: [ type=TCB | TCB指针[43:0] | 权限位 ]
// words[1]: [ 调度上下文指针 | 优先级 | 状态标志 ]
// 权限位：读、写、挂起、恢复、配置等
```

### 4. **Endpoint**
```c
// words[0]: [ type=Endpoint | Endpoint指针[43:0] | 权限位 ]
// words[1]: [ 消息队列头 | 等待队列 | 状态 ]
// 权限位：发送、接收、轮询
```

### 5. **Notification**
```c
// Notification capability:
// words[0]: [ type=Notification | Notification指针[43:0] | 权限位 ]
// words[1]: [ 信号位图 | 绑定端点 | 等待队列 ]
// 权限位：信号、等待、绑定
```
包含如下信息
1. notification_t 类型的地址；
2. notification badge信息；【徽章信息，用于标识signal的发送者信息】-> 该信息会被传递给waiter
	1. badge为另一个cap的captr，其可以是endpoint或者notification
	2. badge所标识的cap是waiter的一个cap；可以根据badge(captr)信息从waiter的cnode中查询到对应的cap
3. 

### 6. **CNode (Capability Node)**
```c
// CNode capability:
// words[0]: [ type=CNode | CNode指针[43:0] | 权限位 | 守卫大小 ]
// words[1]: [ 守卫 | 大小位图 | 解析深度 ]
// 权限位：读、写、授权、移动等
```

### 7. **Page Table**
```c
// Page Table capability:
// words[0]: [ type=PageTable | PT指针[43:0] | 权限位 | 级别 ]
// words[1]: [ 映射的ASID | 保留字段 ]
// 权限位：映射、取消映射
```

### 8. **Page Directory**
```c
// Page Directory capability:
// words[0]: [ type=PageDirectory | PD指针[43:0] | 权限位 ]
// words[1]: [ 映射的ASID | 保留字段 ]
// 权限位：映射、取消映射
```

### 9. **Frame (Memory Page)**
```c
// Frame capability:
// words[0]: [ type=Frame | 物理地址[43:0] | 权限位 | 页面大小 ]
// words[1]: [ 映射信息 | 缓存属性 | 内存类型 ]
// 权限位：读、写、执行、映射、缓存控制
```
每一个具体的物理page都由一个Frame Cap表征:
1. 其中会存储对应的物理页起始地址在内核空间的虚拟地址；因此内核可以通过其获取真实的物理地址
2. 其还会存储映射到的虚拟地址：当然该虚拟地址的值就是在对应的线程虚拟内存空间中了
3. 存储Frame属性：是否为Device Frame
4. 存储访问权限信息（VMRights）
5. 对应物理页在内核态的虚拟基地址
6. 该Frame的页大小【4K, 2M, ...】
### 10. **IRQ Handler**
```c
// IRQ Handler capability:
// words[0]: [ type=IRQHandler | IRQ号 | 权限位 ]
// words[1]: [ 绑定的Notification | 触发模式 | 保留 ]
// 权限位：绑定、解绑、ACK
```

### 11. **IRQ Control**
```c
// IRQ Control capability:
// words[0]: [ type=IRQControl | 保留 ]
// words[1]: [ 保留 ]
// 用途：创建IRQ Handler能力
```

### 12. **ASID Control**
```c
// ASID Control capability:
// words[0]: [ type=ASIDControl | 保留 ]
// words[1]: [ ASID池信息 ]
// 用途：管理地址空间标识符
```

### 13. **ASID Pool**
```c
// ASID Pool capability:
// words[0]: [ type=ASIDPool | ASID池指针[43:0] | 权限位 ]
// words[1]: [ ASID位图 | 已分配计数 ]
// 权限位：分配、释放ASID
```

### 14. **Reply**
```c
// Reply capability:
// words[0]: [ type=Reply | TCB指针[43:0] | 权限位 ]
// words[1]: [ 原始调用者 | 消息标签 | 状态 ]
// 权限位：回复、授予
```

### 15. **Domain**
```c
// Domain capability:
// words[0]: [ type=Domain | 域ID | 权限位 ]
// words[1]: [ 调度周期 | 时间片 | 保留 ]
// 权限位：设置域调度参数
```

### 16. **Sched Control**
```c
// Sched Control capability:
// words[0]: [ type=SchedControl | 核心ID | 权限位 ]
// words[1]: [ 预算 | 周期 | 标志 ]
// 权限位：配置调度参数
```

### 17. **IOMMU**
```c
// IOMMU capability (ARM SMMU):
// words[0]: [ type=IOMMU | IOMMU指针[43:0] | 权限位 ]
// words[1]: [ 上下文银行 | 设备ID映射 ]
// 权限位：映射、配置设备DMA
```

### 18. **IO Port**
```c
// IO Port capability (x86):
// words[0]: [ type=IOPort | 起始端口号 | 端口数量 | 权限位 ]
// words[1]: [ 保留 ]
// 权限位：读、写端口
```

## 权限位典型定义
```c
// 通用权限标志位位置
#define CAP_READ_BIT     0
#define CAP_WRITE_BIT    1  
#define CAP_GRANT_BIT    2
#define CAP_MAP_BIT      3
#define CAP_EXEC_BIT     4
#define CAP_SEND_BIT     5
#define CAP_RECV_BIT     6
#define CAP_CONFIGURE_BIT 7
// ... 类型特定权限位
```

## 类型码枚举
```c
typedef enum {
    seL4_UntypedObject = 0,
    seL4_TCBObject = 1,
    seL4_EndpointObject = 2,
    seL4_NotificationObject = 3,
    seL4_CapTableObject = 4,
    seL4_ReplyObject = 5,
    seL4_PageTableObject = 6,
    seL4_PageDirectoryObject = 7,
    seL4_FrameObject = 8,
    seL4_IRQHandlerObject = 9,
    seL4_IRQControlObject = 10,
    seL4_ASIDControlObject = 11,
    seL4_ASIDPoolObject = 12,
    seL4_DomainObject = 13,
    seL4_SchedControlObject = 14,
    seL4_IOMMUObject = 15,
    seL4_IOPortObject = 16,
    // ... 其他架构特定类型
} seL4_ObjectType;
```

这种统一但类型特定的编码方案允许 seL4 内核在保持强类型安全的同时，高效地验证和操作各种能力。



