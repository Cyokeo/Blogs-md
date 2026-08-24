
## 三种语义
1. 移动：在没有Copy时，赋值/函数调用传参时都默认移动语义；原先变量所有权被转移；后续不能再使用原变量
2. 引用：借用的方式；原变量仍有效
3. 复制（Copy trait），必须也实现Clone
	a. Copy是一种给编译器的提示，无需实现Copy函数
```rust
// 手动实现 Copy（通常用 derive）
struct MyType {
    data: [u8; 16],
}

impl Copy for MyType {}  // 需要先实现 Clone

impl Clone for MyType {
    fn clone(&self) -> Self {
        *self  // 依赖 Copy 实现
    }
}

// 或者更简单：
#[derive(Copy, Clone)]
struct MyDerivedType {
    data: [u8; 16],
}
```
**记住这个简单规则**
- **有 `Copy` trait → 复制语义**
- **无 `Copy` trait → 移动语义**

## Copy的局限性
### **Copy 不能有 Drop**
```cpp
// 重要限制：Copy 和 Drop 互斥
struct Resource {
    handle: i32,
}

impl Drop for Resource {  // 有 Drop 实现
    fn drop(&mut self) {
        println!("Cleaning up {}", self.handle);
    }
}

// impl Copy for Resource {}  // ❌ 编译错误！
// 因为 Copy + Drop 会导致双重清理
```
### **Copy 是静态决定**
```rust
// Copy 是在编译时决定的特性
// 不能在运行时改变行为

fn process<T>(value: T) {
    // 在编译时就知道 T 是否是 Copy
    // 编译器会为每种情况生成不同的代码
}

process(42_i32);    // 使用复制语义
process("hello".to_string());  // 使用移动语义
```
### **何时应该实现 Copy？**
```rust
// 考虑实现 Copy 当：
// 1. 类型很小（<= 64字节是个好经验值）
// 2. 复制成本低（按位复制安全）
// 3. 不需要自定义析构逻辑（Drop）
// 4. 用户期望值语义（而不是引用语义）

#[derive(Copy, Clone, Debug)]
struct Pixel {
    r: u8,
    g: u8,
    b: u8,
    a: u8,
} // 4字节，适合 Copy

// 使用起来很自然：
let p1 = Pixel { r: 255, g: 0, b: 0, a: 255 };
let p2 = p1;  // 复制，不是移动
let p3 = p1;  // 可以继续复制
```
### **何时不应该实现 Copy？**
```rust
// 避免实现 Copy 当：
// 1. 类型很大（复制昂贵）
// 2. 包含需要特殊清理的资源
// 3. 具有唯一性语义（如文件句柄）
// 4. 需要自定义复制逻辑（使用 Clone 代替）!!!!!

// 应该用 Clone 而不是 Copy：
#[derive(Clone)]  // 但不 Copy
struct BigData {
    buffer: Vec<u8>,  // 可能很大
}

let data1 = BigData { buffer: vec![0; 1024*1024] };
let data2 = data1.clone();  // 显式克隆，用户知道成本
```

# 如何正确的管理资源
### **FFI 中的资源句柄**
```rust
// 从 C 库获得的资源句柄
#[repr(transparent)]
#[derive(Copy, Clone)]
struct CHandle(pub i32);  // C 文件描述符

// 问题：谁负责关闭文件？
// 因为是 Copy，不能有 Drop 实现！
// 解决方案1：手动管理
unsafe fn use_chandle() {
    let handle = CHandle(open_file("test.txt"));
    
    // 使用文件...
    
    // 必须手动关闭！
    close_file(handle.0);
}

// 解决方案2：包装在非 Copy 类型中     !!!!!
struct ManagedHandle(CHandle);

impl Drop for ManagedHandle {
    fn drop(&mut self) {
        unsafe { close_file(self.0.0) };
    }
}

impl ManagedHandle {
    fn as_raw(&self) -> CHandle {
        self.0  // 复制 CHandle（i32 是 Copy）
    }
}
```
## **正确的设计模式**
### **模式1：分离 Copy 的数据和需要管理的资源**
```rust
// 数据部分：可 Copy
#[derive(Copy, Clone)]
struct HandleId(u64);  // 纯 ID，可复制

// 管理部分：不可 Copy，有 Drop
struct ResourceManager {
    id: HandleId,
    // 内部状态、连接等需要清理的资源
}

impl Drop for ResourceManager {
    fn drop(&mut self) {
        println!("Cleaning up resource {}", self.id.0);
        // 实际的清理逻辑
    }
}
```
### **模式2：使用引用计数（Rc/Arc）**
```cpp
use std::rc::Rc;

// Rc 本身是 Copy！但指向的数据可以有 Drop
struct SharedResource {
    data: Rc<String>,  // Rc 是 Copy，但管理堆内存
}

// 可以自由复制 SharedResource
let resource1 = SharedResource {
    data: Rc::new(String::from("hello")),
};
let resource2 = resource1;  // 复制 Rc，增加引用计数

// 最后一个 SharedResource 被 drop 时，
// Rc 会清理 String
```
### **模式3：Clone（深拷贝）代替 Copy**
```cpp
// 当需要"复制但需要清理"时，使用 Clone
struct NetworkConnection {
    socket: i32,  // 假设是系统套接字
}

impl Clone for NetworkConnection {
    fn clone(&self) -> Self {
        // 创建新的套接字连接
        Self {
            socket: unsafe { duplicate_socket(self.socket) },
        }
    }
}

impl Drop for NetworkConnection {
    fn drop(&mut self) {
        unsafe { close_socket(self.socket) };
    }
}

// 使用：
let conn1 = NetworkConnection { socket: 42 };
let conn2 = conn1.clone();  // 显式深拷贝
// 每个都有自己的清理逻辑
```

### **决策流程图**
```text
需要管理的资源？
    ├── 是 → 能自动化管理吗？
    │       ├── 能（如 Rc） → 使用智能指针
    │       └── 不能 → 需要 Copy 吗？
    │               ├── 是 → 外部管理器 + 显式清理
    │               └── 否 → 非 Copy + Drop（RAII）
    │
    └── 否 → 是小型值类型吗？
            ├── 是 → 实现 Copy（自动管理）
            └── 否 → 可能不需要 Copy
```