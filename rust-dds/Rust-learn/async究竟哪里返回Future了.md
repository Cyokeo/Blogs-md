## 参考文档
- http://huangjj27.github.io/async-book/02_execution/04_executor.html
- https://huangjj27.github.io/async-book/01_getting_started/04_async_await_primer.html

## 源码分析

```rust
pub async fn bind<A: ToSocketAddrs>(addr: A) -> io::Result<UdpSocket> {
	let addrs = to_socket_addrs(addr).await?;
	let mut last_err = None;
	for addr in addrs {
		match UdpSocket::bind_addr(addr) {
			Ok(socket) => return Ok(socket),
			Err(e) => last_err = Some(e),
		}
	}
	Err(last_err.unwrap_or_else(|| {
		io::Error::new(
		io::ErrorKind::InvalidInput,
		"could not resolve to any address",
		)	
	}))
}
```

1. 通过这个函数，我们知道，其之所以成为async，还是因为其调用了`to_socket_addr()`这个异步函数，并调用了.await？
2. 而`to_socket_addr()`就会返回一个Future
3. 而且`to_socket_addr()`是一个非异步的函数
```rust
type ReadyFuture<T> = future::Ready<io::Result<T>>;

cfg_net! {
	pub(crate) fn to_socket_addrs<T>(arg: T) -> T::Future
		where T: ToSocketAddrs,
	{
		arg.to_socket_addrs(sealed::Internal)
	}
}

pub trait ToSocketAddrsPriv {
	type Iter: Iterator<Item = SocketAddr> + Send + 'static;
	type Future: Future<Output = io::Result<Self::Iter>> + Send + 'static;
	fn to_socket_addrs(&self, internal: Internal) -> Self::Future;
}

/// ready.rs
pub struct Ready<T>(Option<T>);
#[stable(feature = "future_readiness_fns", since = "1.48.0")]
impl<T> Unpin for Ready<T> {}
#[stable(feature = "future_readiness_fns", since = "1.48.0")]
impl<T> Future for Ready<T> {
	type Output = T;
	#[inline]
	fn poll(mut self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<T> {
		Poll::Ready(self.0.take().expect("`Ready` polled after completion"))
	}
}
```

`async fn`函数返回实现了`Future`的类型。为了执行这个`Future`，我们需要执行器（executor）
```rust
// `block_on` blocks the current thread until the provided future has run to
// completion. Other executors provide more complex behavior, like scheduling
// multiple futures onto the same thread.   !!!!!
use futures::executor::block_on;

async fn hello_world() {
    println!("hello, world!");
}

fn main() {
    let future = hello_world(); // Nothing is printed
    block_on(future); // `future` is run and "hello, world!" is printed
}

```