### 各种运行时的选择：

|运行时|特点|适用场景|
|---|---|---|
|**tokio**|功能完整，性能最优|生产服务器、高性能网络应用|
|**async-std**|API 类似 std，易上手|快速原型、学习|
|**smol**|轻量级，可嵌入|嵌入式、资源受限环境|
|**futures**|只有执行器，无 I/O|纯计算任务、测试|

## ## **`std::io::Read` 和 `std::io::Write`**

# async探索
> 有一个普通的main函数，我要使用std::thread在其中创建一个线程，并在新创建的线程中调用async main_entry函数，请帮我实现

## 方法1：使用 `futures` crate（最常用）
> futures也是`tokio`库中的内容

```rust
use std::thread;

// Cargo.toml 需要添加依赖
// [dependencies]
// futures = "0.3"

async fn main_entry() {
    println!("异步函数开始执行");
    
    // 模拟异步工作
    println!("做一些异步操作...");
    
    // 注意：这里不能用真正的异步 I/O，因为缺少运行时
    // 但可以做纯计算或者模拟异步
    
    println!("异步函数执行完成");
}

fn main() {
    println!("主线程开始");
    
    // 创建新线程
    let handle = thread::spawn(|| {
        println!("新线程开始");
        
        // 使用 futures::executor::block_on 阻塞执行 async 函数
        futures::executor::block_on(async {
            main_entry().await;
        });
        
        println!("新线程结束");
    });
    
    // 主线程可以继续做其他事情
    println!("主线程做其他工作...");
    
    // 等待新线程完成
    handle.join().unwrap();
    
    println!("主线程结束");
}
```

## 方法2：使用 `tokio` 运行时
```rust
use std::thread;

// Cargo.toml 需要添加依赖
// [dependencies]
// tokio = { version = "1.0", features = ["full"] }

async fn main_entry() {
    println!("异步函数开始执行");
    
    // 这里可以使用真正的异步 I/O
    println!("可以在这里使用 tokio 的异步 API");
    
    println!("异步函数执行完成");
}

fn main() {
    println!("主线程开始");
    
    // 创建新线程
    let handle = thread::spawn(|| {
        println!("新线程开始");
        
        // 创建独立的 tokio 运行时
        let rt = tokio::runtime::Runtime::new().unwrap();
        
        // 在这个线程中运行异步代码
        rt.block_on(async {
            main_entry().await;
        });
        
        println!("新线程结束");
    });
    
    // 主线程可以继续做其他事情
    println!("主线程做其他工作...");
    
    // 等待新线程完成
    handle.join().unwrap();
    
    println!("主线程结束");
}
```

## 方法3：使用 `async-std` 运行时
```rust
use std::thread;

// Cargo.toml 需要添加依赖
// [dependencies]
// async-std = "1.12"

async fn main_entry() {
    println!("异步函数开始执行");
    
    // 这里可以使用 async-std 的异步 API
    println!("使用 async-std 的异步功能");
    
    println!("异步函数执行完成");
}

fn main() {
    println!("主线程开始");
    
    // 创建新线程
    let handle = thread::spawn(|| {
        println!("新线程开始");
        
        // 使用 async_std::task::block_on
        async_std::task::block_on(async {
            main_entry().await;
        });
        
        println!("新线程结束");
    });
    
    // 主线程可以继续做其他事情
    println!("主线程做其他工作...");
    
    // 等待新线程完成
    handle.join().unwrap();
    
    println!("主线程结束");
}
```

## 方法4：只使用标准库（功能有限）
```rust
use std::thread;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// 自定义简单的 Future
struct SimpleFuture {
    completed: bool,
}

impl Future for SimpleFuture {
    type Output = ();
    
    fn poll(mut self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.completed {
            Poll::Ready(())
        } else {
            self.completed = true;
            Poll::Pending
        }
    }
}

async fn main_entry() {
    println!("异步函数开始执行");
    
    // 只能使用自定义的简单 Future
    let fut = SimpleFuture { completed: false };
    fut.await;
    
    println!("异步函数执行完成");
}

// 简单的执行器
fn block_on<F: Future>(mut future: F) -> F::Output {
    use std::task::{RawWaker, RawWakerVTable};
    
    // 创建一个不做任何事的 waker
    unsafe fn clone(_data: *const ()) -> RawWaker {
        RawWaker::new(std::ptr::null(), &VTABLE)
    }
    
    unsafe fn wake(_data: *const ()) {}
    
    static VTABLE: RawWakerVTable = RawWakerVTable::new(clone, wake, wake, drop);
    
    let waker = unsafe { std::task::Waker::from_raw(RawWaker::new(std::ptr::null(), &VTABLE)) };
    let mut cx = Context::from_waker(&waker);
    
    loop {
        // 安全地 pin future
        let pinned = unsafe { Pin::new_unchecked(&mut future) };
        
        match pinned.poll(&mut cx) {
            Poll::Ready(result) => return result,
            Poll::Pending => {
                // 忙等待（实际中应该 yield）
                thread::yield_now();
            }
        }
    }
}

fn main() {
    println!("主线程开始");
    
    // 创建新线程
    let handle = thread::spawn(|| {
        println!("新线程开始");
        
        // 使用自定义的 block_on
        block_on(async {
            main_entry().await;
        });
        
        println!("新线程结束");
    });
    
    // 主线程可以继续做其他事情
    println!("主线程做其他工作...");
    
    // 等待新线程完成
    handle.join().unwrap();
    
    println!("主线程结束");
}
```

## 总结
虽然标准库提供了`async` `await`，但是其没有运行时
```text
// Rust标准库提供了：
// 1. async/await 语法 ✅
// 2. Future trait ✅  
// 3. 任务调度原语（Waker, Context, Poll）✅
// 4. 异步运行时 ❌
// 5. 异步I/O支持 ❌

// 所以我们要自己实现一个最简单的"运行时"
```
