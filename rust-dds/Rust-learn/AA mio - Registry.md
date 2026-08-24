## 重要文件
1. `pub struct Registry` in poll.rs

```text
mio 库结构：
├── src/
│   ├── sys/
│   │   ├── unix/           # Unix-like 系统（Linux, macOS, BSD）
│   │   │   ├── epoll.rs    # Linux: epoll
│   │   │   ├── kqueue.rs   # macOS/BSD: kqueue
│   │   │   └── selector.rs # Unix 公共抽象
│   │   ├── windows/        # Windows 系统
│   │   │   └── iocp.rs     # Windows: IOCP
│   │   └── mod.rs          # 平台选择
│   ├── poll.rs             # Poll 结构体（公共 API）
│   └── registry.rs         # 注册表
```


## impl event::Source for UdpSocket {}
/Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/mio-1.1.1/src/net/udp.rs
```rust
impl event::Source for UdpSocket {

	fn register(
	&mut self,
	registry: &Registry,
	token: Token,
	interests: Interest,
	) -> io::Result<()> {
		self.inner.register(registry, token, interests)
	}

	pub(crate) fn register(
	&mut self,
	registry: &Registry,
	token: Token,
	interests: Interest,
	fd: RawFd,
	) -> io::Result<()> {
		// Pass through, we don't have any state.
		registry.selector().register(fd, token, interests)
	}

#[cfg_attr(not(feature = "os-ext"), allow(dead_code))]
pub fn register(&self, fd: RawFd, token: Token, interests: Interest) -> io::Result<()> {
	let flags = libc::EV_CLEAR | libc::EV_RECEIPT | libc::EV_ADD;
	// At most we need two changes, but maybe we only need 1.
	let mut changes: [MaybeUninit<libc::kevent>; 2] =
	[MaybeUninit::uninit(), MaybeUninit::uninit()];
	let mut n_changes = 0;
	if interests.is_writable() {
		let kevent = kevent!(fd, libc::EVFILT_WRITE, flags, token.0);
		changes[n_changes] = MaybeUninit::new(kevent);
		n_changes += 1;
	}
	
	if interests.is_readable() {
		let kevent = kevent!(fd, libc::EVFILT_READ, flags, token.0);
		changes[n_changes] = MaybeUninit::new(kevent);
		n_changes += 1;
	}

// Older versions of macOS (OS X 10.11 and 10.10 have been witnessed)
// can return EPIPE when registering a pipe file descriptor where the
// other end has already disappeared. For example code that creates a
// pipe, closes a file descriptor, and then registers the other end will
// see an EPIPE returned from `register`.
//
// It also turns out that kevent will still report events on the file
// descriptor, telling us that it's readable/hup at least after we've
// done this registration. As a result we just ignore `EPIPE` here
// instead of propagating it.
//
// More info can be found at tokio-rs/mio#582.
	let changes = unsafe {
		// This is safe because we ensure that at least `n_changes` are in
		// the array.
		slice::from_raw_parts_mut(changes[0].as_mut_ptr(), n_changes)
		};
		kevent_register(self.kq.as_raw_fd(), changes, &[libc::EPIPE as i64])
	}
}


#[track_caller]
#[cfg_attr(feature = "signal", allow(unused))]
pub(crate) fn new_with_interest(io: E, interest: Interest) -> io::Result<Self> {
	Self::new_with_interest_and_handle(io, interest, scheduler::Handle::current())   ！！！！
}


tokio_thread_local! {  /// 线程本地变量
	static CONTEXT: Context = const {
	Context {} // /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/context.rs
	// Handle是与当前线程绑定的一个结构
}
```

通过阅读代码，发现：
在创建udpsocket时，就已经把其注册到了epoll/kqueue中，且注册了READABLE/WRITABLE事件

## kevent
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/mio-1.1.1/src/sys/unix/selector/kqueue.rs
```rust

```
在 Tokio 中，**selector 的 `select` 方法（或对应的系统调用）是在 I/O 驱动（Driver）的事件循环中调用的**
### 1. **主调用路径（单线程运行时）**
```text
Runtime::block_on()
    └── Runtime::enter()
        └── Driver::turn()
            └── Driver::poll()
                └── mio::Poll::poll()  ←─ 内部调用 selector 的 select
```

