# Rust 多线程共享内存变量的核心方案：`Arc` + 内部可变性

Rust 的核心原则是 **“共享不可变，可变不共享”**，但多线程场景下我们通常需要**“共享且可变”**，这时候需要用 **`Arc`（跨线程共享所有权）** + **线程安全的内部可变性容器**（`Mutex`/`RwLock`/`Atomic`）来实现。

---

# 一、最通用的方案：`Arc<Mutex<T>>`

这是 Rust 多线程共享可变变量的**标配组合**：

- **`Arc<T>`**：原子引用计数，让多个线程共同拥有同一个变量的所有权 ✅
- **`Mutex<T>`**：互斥锁，保证同一时间只有一个线程能修改变量（内部可变性）

## 完整示例：多线程累加计数器
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // 1. 创建一个被 Arc<Mutex> 包裹的计数器
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    // 2. 启动 10 个线程
    for _ in 0..10 {
        // 克隆 Arc，增加引用计数，让新线程也能持有 counter
        let counter = Arc::clone(&counter);
        
        let handle = thread::spawn(move || {
            // 3. 获取锁（阻塞直到拿到锁）
            let mut num = counter.lock().unwrap();
            
            // 4. 修改数据（锁会自动在作用域结束时释放）
            *num += 1; // 语法不对称：没有隐式的自增引用，却有隐式的自动销毁
        });
        
        handles.push(handle);
    }

    // 5. 等待所有线程结束
    for handle in handles {
        handle.join().unwrap();
    }

    // 6. 查看最终结果
    println!("Result: {}", *counter.lock().unwrap());
}
```

### 运行结果
```txt
Result: 10
```

---

# 二、读多写少的优化方案：`Arc<RwLock<T>>`

如果你的场景是**读多写少**（比如配置信息、缓存），用 `RwLock` 比 `Mutex` 更高效：

- **`RwLock`**：读写锁，允许多个线程同时读，但同一时间只能有一个线程写

## 示例：多线程读写配置
```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let config = Arc::new(RwLock::new(String::from("initial config")));
    let mut handles = vec![];

    // 启动 5 个读线程
    for _ in 0..5 {
        let config = Arc::clone(&config);
        let handle = thread::spawn(move || {
            // 获取读锁（多个读线程可以同时持有）
            let cfg = config.read().unwrap();
            println!("Read config: {}", *cfg);
        });
        handles.push(handle);
    }

    // 启动 1 个写线程
    let config = Arc::clone(&config);
    let write_handle = thread::spawn(move || {
        // 获取写锁（会阻塞所有读/写线程）
        let mut cfg = config.write().unwrap();
        *cfg = String::from("updated config");
        println!("Wrote config: {}", *cfg);
    });
    handles.push(write_handle);

    for handle in handles {
        handle.join().unwrap();
    }
}
```

---

# 三、简单整数的无锁方案：`Arc<AtomicT>`

对于**简单的整数类型**（如 `i32`、`usize`、`bool`），可以用**原子类型**（`AtomicUsize`、`AtomicBool` 等），它们是**无锁**的，性能更高：
```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicUsize::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            // 原子加法（无锁）
            counter.fetch_add(1, Ordering::SeqCst);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", counter.load(Ordering::SeqCst));
}
```

---

# 四、核心方案总结

|场景|推荐组合|特点|
|---|---|---|
|**通用场景**|`Arc<Mutex<T>>`|最稳妥，支持任意类型，同一时间只有一个线程访问|
|**读多写少**|`Arc<RwLock<T>>`|允许多读单写，读性能更好|
|**简单整数**|`Arc<AtomicT>`|无锁，性能最高，仅支持原子类型|

---

# 五、注意事项

1. **避免死锁**：不要在持有锁的情况下再申请另一个锁
2. **锁的自动释放**：`Mutex`/`RwLock` 的锁会在 `MutexGuard` 离开作用域时自动释放
3. **错误处理**：`lock()` 会返回 `Result`，如果持有锁的线程 panic 了，锁会被 “污染”，需要用 `unwrap()` 或处理 `PoisonError`