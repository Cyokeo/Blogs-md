
```c
#define NUM_READY_QUEUES (CONFIG_NUM_DOMAINS * CONFIG_NUM_PRIORITIES)
// threads ready to be scheduled
UP_STATE_DEFINE(tcb_queue_t, ksReadyQueues[NUM_READY_QUEUES]);

#ifdef CONFIG_KERNEL_MCS
/* Head of the queue of threads waiting for their budget to be replenished */
UP_STATE_DEFINE(tcb_queue_t, ksReleaseQueue);
#endif
```