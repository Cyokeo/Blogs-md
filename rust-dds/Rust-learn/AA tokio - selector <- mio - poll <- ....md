## kevent
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/mio-1.1.1/src/sys/unix/selector/kqueue.rs
```rust
pub fn select(&self, events: &mut Events, timeout: Option<Duration>) -> io::Result<()> {
	let timeout = timeout.map(|to| libc::timespec {
		tv_sec: cmp::min(to.as_secs(), libc::time_t::MAX as u64) as libc::time_t,
		// `Duration::subsec_nanos` is guaranteed to be less than one
		// billion (the number of nanoseconds in a second), making the
		// cast to i32 safe. The cast itself is needed for platforms
		// where C's long is only 32 bits.
		tv_nsec: libc::c_long::from(to.subsec_nanos() as i32),
	});
	
	let timeout = timeout.as_ref().map(|s| s as *const _).unwrap_or(ptr::null_mut());
	
	events.clear();
	syscall!(kevent(self.kq.as_raw_fd(),
		ptr::null(), 0,
		events.as_mut_ptr().cast(),
		events.capacity() as Count, timeout,)).map(|n_events| {
			// This is safe because `kevent` ensures that `n_events` are
			// assigned.
			unsafe { events.set_len(n_events as usize) };
		})
}
```
在 Tokio 中，**selector 的 `select` 方法（或对应的系统调用）是在 I/O 驱动（Driver）的事件循环中调用的**
### 1. **主调用路径（单线程运行时）**
```text
Runtime::block_on()
    └── Runtime::enter()
		  ---- 在向上的调用路径就有些复杂了
		    Driver::park()/park_timeout()
	        └── Driver::turn()
	            └── Driver::poll()
	                └── mio::Poll::poll()  ←─ 内部调用 selector 的 select
```

### 2. mio::Poll::poll()
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/mio-1.1.1/src/poll.rs

```rust
pub fn poll(&mut self, events: &mut Events, timeout: Option<Duration>) -> io::Result<()> {
	self.registry.selector.select(events.sys(), timeout)
}
```

### 3. tokio::Driver::turn()
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/io/driver.rs

```rust
/// I/O driver, backed by Mio.
pub(crate) struct Driver {
	/// True when an event with the signal token is received
	signal_ready: bool,
	/// Reuse the `mio::Events` value across calls to poll.
	events: mio::Events,
	/// The system event queue.
	poll: mio::Poll,
}

/// I/O 驱动句柄的引用
/// 
/// 该结构体提供了对 I/O 驱动功能的访问接口，
/// 用于注册和管理异步 I/O 资源。
pub(crate) struct Handle {
    /// 注册 I/O 资源的核心注册表
    /// 
    /// 这是 `mio` 提供的注册表，用于在系统事件多路复用器
    ///（如 epoll、kqueue、IOCP）中注册文件描述符。
    registry: mio::Registry,

    /// 跟踪所有注册的 I/O 资源
    /// 
    /// 维护已注册 I/O 资源的集合，包括它们的生命周期管理和状态跟踪。
    registrations: RegistrationSet,

    /// 需要同步访问的状态
    /// 
    /// 使用互斥锁保护需要线程安全访问的内部状态。
    synced: Mutex<registration_set::Synced>,

    /// 用于从 `turn` 调用中唤醒反应器
    /// 
    /// 在 `Wasi` 目标平台上不支持，因为 WASI 缺乏线程支持。
    #[cfg(not(target_os = "wasi"))]
    waker: mio::Waker,

    /// I/O 驱动性能指标收集器
    /// 
    /// 收集和跟踪 I/O 操作的性能指标，如就绪事件数量、等待时间等。
    pub(crate) metrics: IoDriverMetrics,

    /// io_uring 上下文（仅适用于 Linux）
    /// 
    /// 使用互斥锁保护的 io_uring 上下文，用于 Linux 上的高性能异步 I/O。
    #[cfg(all(
        tokio_unstable,      // 需要启用不稳定特性
        feature = "io-uring", // 启用 io_uring 特性
        feature = "rt",      // 启用运行时特性
        feature = "fs",      // 启用文件系统特性
        target_os = "linux", // 仅限 Linux 平台
    ))]
    pub(crate) uring_context: Mutex<UringContext>,

    /// io_uring 状态（仅适用于 Linux）
    /// 
    /// 原子操作维护的 io_uring 状态，用于高效的状态同步和标记。
    #[cfg(all(
        tokio_unstable,
        feature = "io-uring",
        feature = "rt",
        feature = "fs",
        target_os = "linux",
    ))]
    pub(crate) uring_state: AtomicUsize,
}

/// 执行一次 I/O 事件循环迭代
fn turn(&mut self, handle: &Handle, max_wait: Option<Duration>) {
    // 确保注册表未关闭（调试断言）
    debug_assert!(!handle.registrations.is_shutdown(&handle.synced.lock()));

    // 释放待处理的注册项
    handle.release_pending_registrations();

    // 获取事件缓冲区引用
    let events = &mut self.events;

    // 阻塞等待事件发生，获取事件数量
    match self.poll.poll(events, max_wait) {  !!!!!
        // 成功获取事件
        Ok(()) => {}
        
        // 系统调用被中断 - 正常情况，继续处理
        Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {}
        
        // WASI 目标平台的特殊错误处理
        #[cfg(target_os = "wasi")]
        Err(e) if e.kind() == io::ErrorKind::InvalidInput => {
            // 在 wasm32_wasi 上，当尝试轮询但没有订阅时会发生此错误
            // 直接从 park 返回，因为没有东西会唤醒我们
        }
        
        // 其他错误 - 视为致命错误
        Err(e) => panic!("unexpected error when polling the I/O driver: {e:?}"),
    }

    // 处理所有到达的事件，进行适当的分发
    let mut ready_count = 0;
    
    for event in events.iter() {
        let token = event.token();

        // 根据令牌类型进行不同处理
        match token {
            // 唤醒令牌 - 用于解除 I/O 驱动的阻塞
            TOKEN_WAKEUP => {
                // 无需操作，事件仅用于解除阻塞
            }
            
            // 信号令牌 - 标记信号就绪
            TOKEN_SIGNAL => {
                self.signal_ready = true;
            }
            
            // 普通 I/O 事件令牌
            _ => {
                // 1. 将 mio 事件转换为就绪状态
                let ready = Ready::from_mio(event);
                
                // 2. 从令牌中获取指针地址
                let ptr = super::EXPOSE_IO.from_exposed_addr(token.0);

                // 安全性说明：
                // 我们确保用作令牌的指针在被释放之前：
                // 1) 已从 mio 注销
                // 2) 我们知道 I/O 驱动没有并发轮询
                // I/O 驱动持有 `Arc<ScheduledIo>` 的所有权，
                // 因此可以安全地将此指针转换为引用
                let io: &ScheduledIo = unsafe { &*ptr };

                // 3. 更新就绪状态
                io.set_readiness(Tick::Set, |curr| curr | ready);
                
                // 4. 唤醒等待此事件的任务
                io.wake(ready);

                // 5. 统计处理的事件数量
                ready_count += 1;
            }
        }
    }

    // io_uring 特定处理（Linux 平台）
    #[cfg(all(
        tokio_unstable,
        feature = "io-uring",
        feature = "rt",
        feature = "fs",
        target_os = "linux",
    ))]
    {
        // 获取 io_uring 上下文锁
        let mut guard = handle.get_uring().lock();
        let ctx = &mut *guard;
        
        // 分发已完成的 io_uring 操作
        ctx.dispatch_completions();
    }

    // 更新度量统计：增加就绪事件计数
    handle.metrics.incr_ready_count_by(ready_count);
}
```

### 4. event事件分发
```rust
// /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/libc-0.2.179/src/unix/bsd/apple/mod.rs
#[repr(packed(4))]
pub struct kevent {
pub ident: crate::uintptr_t,
pub filter: i16,
pub flags: u16,
pub fflags: u32,
pub data: intptr_t,
pub udata: *mut c_void,   // 这里是kevent结构中运行存储的用户数据
}

pub fn token(event: &Event) -> Token {
	Token(event.0.udata as usize)  // 这里可以看到，使用了这个用户数据，与[AA mio - Registry]中的注册对应
}

{
	static EXPOSE_IO: PtrExposeDomain<ScheduledIo> = PtrExposeDomain::new();
	let ptr = super::EXPOSE_IO.from_exposed_addr(token.0); // 从turn()函数中的这句调用可以知道，udata用于存储shdceduleIo实例的地址
	let io: &ScheduledIo = unsafe { &*ptr };
	
	// 后续调用scheduleIo的两个函数
	io.set_readiness(Tick::Set, |curr| curr | ready);
	io.wake(ready);
}
```

# 正向调用

## main
```rust
fn main2() {
	tokio::runtime::Builder::new_current_thread()
	.enable_all()
	.build()
	.unwrap()
	.block_on(async {
		println!("Hello world");})
}
```

## runtime
```rust
impl Runtime{
	#[track_caller]
	fn block_on_inner<F: Future>(&self, future: F, _meta: SpawnMeta<'_>) -> F::Output {
		match &self.scheduler {
			Scheduler::CurrentThread(exec) => exec.block_on(&self.handle.inner, future),
			#[cfg(feature = "rt-multi-thread")]
			Scheduler::MultiThread(exec) => exec.block_on(&self.handle.inner, future),
		}
	}
	
#[track_caller]
pub fn block_on<F: Future>(&self, future: F) -> F::Output {
	let fut_size = mem::size_of::<F>();
	if fut_size > BOX_FUTURE_THRESHOLD {
		self.block_on_inner(Box::pin(future), SpawnMeta::new_unnamed(fut_size))
	} else {
		self.block_on_inner(future, SpawnMeta::new_unnamed(fut_size))
	}
}
}
```

## current_thread impl
```rust
impl CurrentThread {
	#[track_caller]
	pub(crate) fn block_on<F: Future>(&self, handle: &scheduler::Handle, future: F) -> F::Output {
		core.block_on(...);
	}
}
```
## current_thread
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/scheduler/current_thread/mod.rs

```rust
impl CoreGuard<'_> {
#[track_caller]
fn block_on<F: Future>(self, future: F) -> F::Output {
	core = if !context.defer.is_empty() {
		context.park_yield(core, handle)  // !!!!!
		} else {
		context.park(core, handle) // !!!
	};
}
```

## Context
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/scheduler/current_thread/mod.rs

```rust
impl Context {
	// ...
	/// Blocks the current thread until an event is received by the driver,
	/// including I/O events, timer events, ...
	fn park(&self, mut core: Box<Core>, handle: &Handle) -> Box<Core> {
		core = self.park_internal(core, handle, &mut driver, None);
	}


	fn park_internal(...) {
		let (core, ()) = self.enter(core, || {
			match duration {
				Some(dur) => driver.park_timeout(&handle.driver, dur),
				None => driver.park(&handle.driver),
			}
			self.defer.wake();
		});
		core
	}
}
```

## top Driver
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/driver.rs

```rust
#[derive(Debug)]
pub(crate) struct Driver {
inner: TimeDriver,
}

pub(crate) fn park(&mut self, handle: &Handle) {
	self.inner.park(handle);
}

impl TimeDriver {
	pub(crate) fn park(&mut self, handle: &Handle) {
	match self {
		TimeDriver::Enabled { driver, .. } => driver.park(handle),
		TimeDriver::EnabledAlt(v) => v.park(handle),
		TimeDriver::Disabled(v) => v.park(handle),
	}
	//...
}

impl IoStack {
	pub(crate) fn park(&mut self, handle: &Handle) {
	match self {
		IoStack::Enabled(v) => v.park(handle),
		IoStack::Disabled(v) => v.park(),
	}
	//...
}
```

## signal Driver
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/process.rs

```rust
impl Driver {

/// Creates a new signal `Driver` instance that delegates wakeups to `park`.
// ...
	pub(crate) fn park(&mut self, handle: &driver::Handle) {
		self.park.park(handle);
		GlobalOrphanQueue::reap_orphans(&self.signal_handle);
	}
	// ...
}
```

## io Driver
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/signal/mod.rs

```rust
pub(crate) struct Driver {
/// Thread parker. The `Driver` park implementation delegates to this.
io: io::Driver,

/// A pipe for receiving wake events from the signal handler
receiver: UnixStream,

/// Shared state. The driver keeps a strong ref and the handle keeps a weak
/// ref. The weak ref is used to check if the driver is still active before
/// trying to register a signal handler.
inner: Arc<()>,
}

impl Driver {
	pub(crate) fn park(&mut self, handle: &driver::Handle) {
		self.io.park(handle);
		self.process();
	}
}
```

## I/O driver, backed by Mio.
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/io/driver.rs

```rust
/// I/O driver, backed by Mio.
pub(crate) struct Driver {
/// True when an event with the signal token is received
signal_ready: bool,

/// Reuse the `mio::Events` value across calls to poll.
events: mio::Events,

/// The system event queue.
poll: mio::Poll,
}

impl Driver {
	pub(crate) fn park(&mut self, rt_handle: &driver::Handle) {
		let handle = rt_handle.io();
		self.turn(handle, None); // !@!这里就跟前面 #3 的Driver::turn接上了
	}
}
```




