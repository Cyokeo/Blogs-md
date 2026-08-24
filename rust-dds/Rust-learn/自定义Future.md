## 参考博客
- https://huangjj27.github.io/async-book/02_execution/02_future.html
	- future的高级用法
- 
## **自定义 Future 的继续条件**
### 示例：等待文件创建
```rust
use std::path::Path;
use tokio::fs;

struct WaitForFile {
    path: &'static Path,
    interval: Duration,
}

impl Future for WaitForFile {
    type Output = io::Result<()>;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
        // 继续条件：文件存在
        if Path::exists(self.path) {
            Poll::Ready(Ok(()))
        } else {
            // 注册定时器，稍后重试
            let waker = cx.waker().clone();
            let path = self.path;
            let interval = self.interval;
            
            tokio::spawn(async move {
                tokio::time::sleep(interval).await;
                waker.wake();  // 唤醒再次检查
            });
            
            Poll::Pending
        }
    }
}

async fn wait_for_file() {
    WaitForFile {
        path: Path::new("/tmp/ready.txt"),
        interval: Duration::from_secs(1),
    }.await;
    println!("文件已存在！");
}
```

## 多个 future 可以一个接一个执行
```rust
/// A SimpleFuture that runs two futures to completion, one after another.
//
// Note: for the purposes of this simple example, `AndThenFut` assumes both
// the first and second futures are available at creation-time. The real
// `AndThen` combinator allows creating the second future based on the output
// of the first future, like `get_breakfast.and_then(|food| eat(food))`.
pub struct AndThenFut<FutureA, FutureB> {
    first: Option<FutureA>,
    second: FutureB,
}

impl<FutureA, FutureB> SimpleFuture for AndThenFut<FutureA, FutureB>
where
    FutureA: SimpleFuture<Output = ()>,
    FutureB: SimpleFuture<Output = ()>,
{
    type Output = ();
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if let Some(first) = &mut self.first {
            match first.poll(wake) {
                // We've completed the first future -- remove it and start on
                // the second!
                Poll::Ready(()) => self.first.take(),
                // We couldn't yet complete the first future.
                Poll::Pending => return Poll::Pending,
            };
        }
        // Now that the first future is done, attempt to complete the second.
        self.second.poll(wake)
    }
}

```