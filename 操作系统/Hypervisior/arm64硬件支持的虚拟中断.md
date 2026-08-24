# 重要参考
1. https://www.scs.stanford.edu/~zyedidia/docs/arm/virtualization.pdf
2. [aarch64虚拟化简介](https://calinyara.github.io/technology/2019/11/03/armv8-virtualization.html)
3. [arm instruction set and system registers](https://arm.jonpalmisc.com/)

# 重要解读：
从下面的描述可以看出：EL2的Hyp写List Register触发中断之前；需要根据当前的中断要路由到哪个vCPU进行context switch，然后再写List Register触发vINIT。否则vINIT可能会被不期望的vCPU响应。
***Hyp在切换vCPU时要主动保存/恢复List Registers***
显然，vINIT的响应，会在切换到vCPU（即恢复到EL1）时进行；显然，vINIT发生时，使用的vbar_el1就由vCPU提供了（vbar_el1也是其需要保存恢复的上下文）

# 总结
virtio前后端交互逻辑：真实网卡物理硬件触发物理中断，进入Hyp进行网卡接收事件的处理。【这里有bridge进行消息的分发，bridge是运行在Hyp内的server】，bridge通过接收报文中的DST MAC信息将报文forwarding给某个virtio后端，virtio后端执行pp_cb回调函数，这里面主要完成与前端的交互准备，即将报文数据写到与前端共享的virtqueue中，之后进行中断注入的第1阶段：将软件维护的pending位等进行置位【这里需要知道对应的vCPU】。**Hyp在退出并恢复某个vCPU执行之前，会进行2阶段的中断注入：遍历pending位将pending的中断写入到硬件“虚拟CPU Interface”的List Register中。之后当PE切换到EL1模式的vCPU时，硬件“虚拟CPU Interface”会触发中断，该中断会被路由到EL1的vCPU进行处理，异常向量表入口由该vCPU的vbar_el1进行指定。**
这里还有不同核通知的处理：即Hyp当前运行的核与前端virtio的vCPU不在同一个物理CPU时，会通过ipi核间中断进行跨核通知
还有一个内容点需要注意：由于是vSPI，1阶段的中断注入是写到VM模拟的GIC相应pending数组的对应bit位中；一个VM可以有很多的vCPU
对于软件维护的vGIC，每个IRQ都对应一个virq_data，其中保存了中断状态等信息，其中就有affinity信息；对于PPI，SGI，他们是vCPU独有的，因此里面affinity是固定的；对于vSPI，他们是多vCPUs共享的；需要VM在初始化时进行vSPI的路由设置，进而设置virtirq_data中的affinity信息。
vCPU似乎就是对sel4内核vcpu对象的分装；virq_data中还有一个irq_handler cap。Hyp中的中断注入，最终会调用到sel4的内核接口seL4_ARM_VCPU_InjectIRQ，在内核中作进一步的中断注入：将irq状态写入内核维护vGicInf的pending list中。后续内核进行恢复时也是根据该pending List更新虚拟CPU接口中的硬件List Register寄存器

