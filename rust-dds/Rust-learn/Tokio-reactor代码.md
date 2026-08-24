#[tokio::main]是如何工作的！！

## 关键代码
```rust
/// /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/io/scheduled_io.rs
impl Future for Readiness<'_>
	type Output = ReadyEvent;
	fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
	
	}
```

## .await
对future.await操作时，编译器生成的代码会调用一次future的poll