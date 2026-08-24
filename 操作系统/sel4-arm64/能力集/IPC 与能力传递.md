# doIPCTransfer 
called by 
## doReplyTransfer
called by 
### handleReply
called by 
#### handleSyscall
when syscall is
1. SysReply
2. SysReplyRecv

this two syscalls exist only when MCS

### performInvocation_Reply
called by
#### decodeInvocation
only when the called cap is `cap_reply_cap`for MCS and NON-MCS

## sendIPC
called by
#### performInvocation_Endpoint
called by 
##### decodeInvocation
only when the called cap is `cap_endpoint_cap` for MCS and NON-MCS

#### sendFaultIPC
called by 
##### handleFault
called multiple in kernel when there is something wrong, and the only single param usually `NODE_STATE(ksCurThread)`

## receiveIPC
called by 
### handleRecv
called by 
#### handleSyscall
when the syscall is
1. `SysRecv`
2. `SysReplyRecv` -> only for NON-MCS
3. `SysWait` -> only MCS
4. `SysNBWait` -> MCS
5. `SysReplyRecv` -> MCS
6. `SysNBSendRecv` -> MCS
7. `SysNBSendWait` -> MCS
8. `SysNBRecv` -> MCS and NON-MCS

# prototype
```c
void doIPCTransfer(tcb_t *sender, endpoint_t *endpoint, word_t badge, bool_t grant, tcb_t *receiver)
```

# code reading
```c
if (cap_get_capType(cap) == cap_endpoint_cap &&
EP_PTR(cap_endpoint_cap_get_capEPPtr(cap)) == endpoint) {
	/* If this is a cap to the endpoint on which the message was sent,
	* only transfer the badge, not the cap. */
	setExtraBadge(receiveBuffer,
	cap_endpoint_cap_get_capEPBadge(cap), i);
	info = seL4_MessageInfo_set_capsUnwrapped(info,
	seL4_MessageInfo_get_capsUnwrapped(info) | (1 << i));
} else {
	// derive the cap
	dc_ret = deriveCap(slot, cap);
	// and insert it to receiver.capSlot
	cteInsert(dc_ret.cap, slot, destSlot);
	// and this receiver.capSlot should be previously set by receiver in IPCBuffer struct
}
```



