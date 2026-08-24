
# pthread_create
1. 进行标准的TLS处理，TLS相关结构都挂在pthread内
```c
void *__copy_tls(unsigned char *mem)
{
pthread_t td;
struct tls_module *p;
size_t i;
uintptr_t *dtv;
#ifdef TLS_ABOVE_TP
dtv = (uintptr_t*)(mem + libc.tls_size) - (libc.tls_cnt + 1);
mem += -((uintptr_t)mem + sizeof(struct pthread)) & (libc.tls_align-1);
td = (pthread_t)mem;
mem += sizeof(struct pthread);
for (i=1, p=libc.tls_head; p; i++, p=p->next) {
	dtv[i] = (uintptr_t)(mem + p->offset) + DTP_OFFSET;
	memcpy(mem + p->offset, p->image, p->len);
}
#else
dtv = (uintptr_t *)mem;
mem += libc.tls_size - sizeof(struct pthread);
mem -= (uintptr_t)mem & (libc.tls_align-1);
td = (pthread_t)mem;
for (i=1, p=libc.tls_head; p; i++, p=p->next) {
	dtv[i] = (uintptr_t)(mem - p->offset) + DTP_OFFSET;
	memcpy(mem - p->offset, p->image, p->len);
}
#endif
dtv[0] = libc.tls_cnt;
td->dtv = dtv;
return td;
}

int pthread_create(void *(*entry)(void *), pthread_attr_t *restrict attrp)
{
	if (attr._a_stackaddr) {
		size_t need = libc.tls_size + __pthread_tsd_size;
		/* Use application-provided stack for TLS only when
		 * it does not take more than ~12% or 2k of the
	     * application's stack space. */
		if (need < size/8 && need < 2048) {
			tsd = stack - __pthread_tsd_size;
			stack = tsd - libc.tls_size;
			memset(stack, 0, need);
		} else {
			size = ROUND(need);
		}
	}
	
	if (!tsd) {
		map = __mmap(0, size, ...)
		tsd = map + size - __pthread_tsd_size;
	}
	// here, tls与pthread有很大的关联关系
	struct *pthread new = new = __copy_tls(tsd - 
		libc.tls_size);
	
	/**
	 * 从clone调用的参数，大致猜测：
	 * 1.clone系统调用内要创建OS-specific thread结构
	 * 2.且要将pthread *new与该OS-specific Thread结构进行关联
	 * 3.Thread ID（tid）也是由具体的OS层管理 **/
	ret = __clone((c11 ? start_c11 : start), stack, flags,
		args, &new->tid, TP_ADJ(new), &__thread_list_lock);
}
```
1. 内部会调用clone系统调用