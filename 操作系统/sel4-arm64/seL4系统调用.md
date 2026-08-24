## API接口
```c
LIBSEL4_INLINE seL4_Error

seL4_CNode_Copy(seL4_CNode _service, seL4_Word dest_index, seL4_Uint8 dest_depth, seL4_CNode src_root, seL4_Word src_index, seL4_Uint8 src_depth, seL4_CapRights_t rights)

{
	// 第一个参数是该次操作的目标cap
	seL4_Error result;
	seL4_MessageInfo_t tag = seL4_MessageInfo_new(CNodeCopy, 0, 1, 5);
	seL4_MessageInfo_t output_tag;
	seL4_Word mr0;
	seL4_Word mr1;
	seL4_Word mr2;
	seL4_Word mr3;
	
	/* Setup input capabilities. */
	// 这里传入额外的cap给到内核
	seL4_SetCap(0, src_root);
	
	// 进入内核后，会通过lookupExtraCaps操作获取extraCap信息；有多少拿多少
	
	// 因此，应该：内核拥有所有线程的cnode以及cap信息！！！
	
	// ...
}
```

```c
handleInvocation(){
	// ...
	
	
	status = lookupExtraCaps(thread, buffer, info);
	// ...
}
```
