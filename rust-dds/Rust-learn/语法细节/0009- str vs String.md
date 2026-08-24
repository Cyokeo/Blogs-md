这是 Rust 中最基础也最容易混淆的一对类型，核心区别一句话就能讲透：

> **`str` 是「字符串切片（引用）」，没有所有权，只是借用数据；`String` 是「堆分配的字符串结构体」，拥有所有权，管理着堆上的内存。**

简单类比：

- `&str` 像 `&[T]`（数组切片），只是一个「视图」；
- `String` 像 `Vec<T>`，是一个「拥有数据的容器」。

## 一、核心对比表（一目了然）

|特性|`str`（通常以 `&str` 形式出现）|`String`|
|---|---|---|
|**所有权**|❌ 没有所有权，是借用（引用）|✅ 拥有所有权，管理堆内存|
|**内存布局**|胖指针：(指向数据的指针 + 长度)|结构体：(指向堆的指针 + 长度 + 容量)|
|**可变性**|默认不可变（`&mut str` 很少用）|可变，可以 push、修改|
|**存储位置**|字符串字面量在只读数据段；也可指向 String 的切片|数据存储在堆上|
|**大小**|`&str` 在栈上占 16 字节（64 位系统）|`String` 在栈上占 24 字节（64 位系统）|
|**常见形式**|`&'static str`（字符串字面量）、`&s[..]`（String 的切片）|`String::from("...")`、`"..."`|

---

## 二、代码示例：用法与区别

### 1. `&str`（字符串切片）的常见用法
```rust
fn main() {
    // 1. 字符串字面量：类型是 &'static str
    // 数据硬编码在程序的只读数据段，存活整个程序期间
    let s_literal: &'static str = "Hello, world!";
    println!("String literal: {}", s_literal);

    // 2. 从 String 创建切片：借用 String 的数据
    let s_string = String::from("Hello, Rust!");
    let s_slice: &str = &s_string[0..5]; // 取前5个字符
    println!("Slice from String: {}", s_slice);

    // 3. 整个 String 的切片：&s_string 或 &s_string[..]
    let full_slice: &str = &s_string;
    println!("Full slice: {}", full_slice);
}
```

### 2. `String`（堆分配字符串）的常见用法
```rust
fn main() {
    // 1. 创建 String
    let mut s = String::from("Hello");
    
    // 2. String 是可变的：可以 push、修改
    s.push_str(", world!"); // 追加字符串
    s.push('!'); // 追加字符
    println!("Modified String: {}", s); // 输出 "Hello, world!"

    // 3. 从 &str 创建 String
    let s_from_slice: String = "Hello, slice!".to_string();
    let s_from_str: String = String::from("Hello, from!");

    // 4. String 拥有所有权，离开作用域会自动释放堆内存
    {
        let temp = String::from("I will be dropped");
        println!("{}", temp);
    } // temp 在这里被 drop，堆内存被释放
}
```

---

## 三、内存布局详解（深入理解）
### 1. `&str` 的内存布局（胖指针）

`&str` 是一个**胖指针（Fat Pointer）**，在栈上占 16 字节（64 位系统），包含两个部分：

- 8 字节：指向字符串数据的指针；
- 8 字节：字符串的长度（字节数，不是字符数）。

```txt
栈上：&str
┌─────────────────┐
│  指针 (8字节)   │ ──→ 指向数据（只读数据段 或 堆）
├─────────────────┤
│  长度 (8字节)   │
└─────────────────┘
```

### 2. `String` 的内存布局

`String` 是一个结构体，在栈上占 24 字节（64 位系统），包含三个部分：

- 8 字节：指向堆上字符串数据的指针；
- 8 字节：字符串的长度（已使用的字节数）；
- 8 字节：字符串的容量（堆上分配的总字节数）。
```txt
栈上：String                          堆上：
┌─────────────────┐                  ┌─────────────────┐
│  指针 (8字节)   │ ───────────────→ │  H e l l o ...  │
├─────────────────┤                  └─────────────────┘
│  长度 (8字节)   │
├─────────────────┤
│  容量 (8字节)   │
└─────────────────┘
```

---

## 四、常见类型转换
### 1. `&str` → `String`
```rust
let s_str: &str = "Hello";
// 方法 1：to_string()
let s_string1: String = s_str.to_string();
// 方法 2：String::from()
let s_string2: String = String::from(s_str);
// 方法 3：into()（如果类型能推断）
let s_string3: String = s_str.into();
```

### 2. `String` → `&str`
```rust
let s_string: String = String::from("Hello");
// 方法 1：直接取引用（最常用，利用 Deref 强制转换）
let s_str1: &str = &s_string;
// 方法 2：取全切片
let s_str2: &str = &s_string[..];
// 方法 3：as_str()
let s_str3: &str = s_string.as_str();
```
