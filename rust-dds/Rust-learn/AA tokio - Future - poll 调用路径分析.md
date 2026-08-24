> 接续selector返回后

1. 能获取到events -> 进而从event中获取scheduleIo的地址 -> 解引用获取ScheduledIo类型实例
2. 调用io.wake()方法 -> 所有的waker都存储在ScheduleIo的waiter成员中，其是一个waiter列表，里面可能含有多个waker
	1. 这里就是说明：Future：：Poll方法中要适当保存waker到合适的位置；<- 因为后续exectuor会遍历waker进行wake()唤醒
	2. 并在唤醒之后，适当时机再调用Future::poll()
3. 获取到waker后，调用waker::wake()方法。这里要注意，wake方法是用虚表实现的，因此会调用实际类型的

# Context
> Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/context/current.rs
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/scheduler/current_thread/mod.rs


# Waker创建
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/util/wake.rs

```rust

/// Simplified waking interface based on Arcs.
pub(crate) trait Wake: Send + Sync + Sized + 'static {
/// Wake by value.
fn wake(arc_self: Arc<Self>);

/// Wake by reference.
fn wake_by_ref(arc_self: &Arc<Self>);
}

/// Creates a reference to a `Waker` from a reference to `Arc<impl Wake>`.
pub(crate) fn waker_ref<W: Wake>(wake: &Arc<W>) -> WakerRef<'_> {
	let ptr = Arc::as_ptr(wake).cast::<()>();
	let waker = unsafe { Waker::from_raw(RawWaker::new(ptr, waker_vtable::<W>())) };
	WakerRef {
		waker: ManuallyDrop::new(waker),
		_p: PhantomData,
	}
}

fn waker_vtable<W: Wake>() -> &'static RawWakerVTable {
	&RawWakerVTable::new(
		clone_arc_raw::<W>,
		wake_arc_raw::<W>,
		wake_by_ref_arc_raw::<W>,
		drop_arc_raw::<W>,
	)
}

unsafe fn clone_arc_raw<T: Wake>(data: *const ()) -> RawWaker {
// Safety: `data` was created from an `Arc::as_ptr` in function `waker_ref`.
	unsafe { Arc::<T>::increment_strong_count(data as *const T);}
	RawWaker::new(data, waker_vtable::<T>())
}

unsafe fn wake_arc_raw<T: Wake>(data: *const ()) {
	// Safety: `data` was created from an `Arc::as_ptr` in function `waker_ref`.
	let arc: Arc<T> = unsafe { Arc::from_raw(data as *const T) };
	Wake::wake(arc);
}
```

## Handle
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/scheduler/current_thread/mod.rs

```rust
impl Wake for Handle {
	fn wake(arc_self: Arc<Self>) {
		Wake::wake_by_ref(&arc_self);
	}
	
	/// Wake by reference
	fn wake_by_ref(arc_self: &Arc<Self>) {
		arc_self.shared.woken.store(true, Release);
		arc_self.driver.unpark();
	}
}
```

# 仔细看下block_on函数的实现

