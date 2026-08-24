
IPC 交互双方是如何通过endpoint联系在一起的？
# 猜测
1. 1个cap在kernel中有唯一的对应实体
2. user在某一个”具名“cap上进行系统调用时，kernel能找到对应在kernel中的原始cap
	1. 进而能够对应通信的双方！

## endpoint-cap 对应 kernel endpoint_t
- 可看到endpoint-cap的word[0]记录了内核对象的地址
- 而派生出的不同的具名cap的word[0]中该字段的值应该是一样的，因此就对应同一个内核endpoint_t类型对象
```c
static inline uint64_t CONST
cap_endpoint_cap_get_capEPPtr(cap_t cap) {
	uint64_t ret;
	/* fail if union does not have the expected tag */
	assert(((cap.words[0] >> 59) & 0x1f) == cap_endpoint_cap);
	ret = (cap.words[0] & 0xffffffffffffull) << 0;
	/* Possibly sign extend */
	if (__builtin_expect(!!(1 && (ret & (1ull << (47)))), 1)) {
		ret |= 0xffff000000000000;
	}
return ret;
}

// this ret will be cast to endpoint_t *
```