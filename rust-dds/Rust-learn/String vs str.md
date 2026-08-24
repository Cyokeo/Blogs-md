由于 `String` 实现了 `Deref<Target = str>`，所以 `String` 可以自动调用 `str` 的方法：

## Deref<Target = str>
`Deref<Target = str>` 是 Rust 中 **关联类型（associated type）** 的语法
### 1. **`Deref` trait 定义**
```rust
pub trait Deref {
    type Target;  // 关联类型
    
    fn deref(&self) -> &Self::Target;
}
```
### 2. **`String` 的实现**
```rust
impl Deref for String {
    type Target = str;  // 指定关联类型为 str
    
    fn deref(&self) -> &str {  // 返回 &str
        // 实现细节...
    }
}
```

