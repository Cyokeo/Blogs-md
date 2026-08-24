> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/task/mod.rs
> /Users/kevin/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tokio-1.49.0/src/runtime/task/core.rs

# 多任务
```rust
#[tokio::main(flavor = "current_thread")]
async fn main() {
    println!("主线程开始");
    
    // 创建多个并发任务
    let task1 = tokio::spawn(async {
        println!("任务1开始");
        tokio::time::sleep(std::time::Duration::from_secs(1)).await;
        println!("任务1结束");
        "任务1结果"
    });
    
    let task2 = tokio::spawn(async {
        println!("任务2开始");
        tokio::time::sleep(std::time::Duration::from_secs(2)).await;
        println!("任务2结束");
        "任务2结果"
    });
    
    let task3 = tokio::spawn(async {
        println!("任务3开始");
        tokio::time::sleep(std::time::Duration::from_secs(3)).await;
        println!("任务3结束");
        "任务3结果"
    });
    
    // 等待所有任务完成
    let (result1, result2, result3) = tokio::join!(task1, task2, task3);
    
    println!("任务1结果: {:?}", result1.unwrap());
    println!("任务2结果: {:?}", result2.unwrap());
    println!("任务3结果: {:?}", result3.unwrap());
    
    println!("所有任务完成");
}
```


