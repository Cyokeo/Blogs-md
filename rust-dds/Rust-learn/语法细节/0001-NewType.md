## 一、Newtype 模式的 4 个核心用法（除了封装）

Newtype 模式（`struct X(Y);`）是 Rust 中**零成本、高安全**的设计模式，除了之前说的「隐藏内部实现」，还有这 4 个最常用的场景：

---

### 1. **类型安全：防止参数混淆（最常用！）**

这是 Newtype 模式**最有价值**的用途 —— 用编译器帮你「防手抖」，避免把不同含义的同类型数据传错。

#### 反面例子（没有 Newtype，容易出错）

```rust
// 都是 String，但含义完全不同
fn create_user(user_id: String, product_id: String) {
    // 万一写反了参数顺序？编译器不会报错！
    println!("创建用户：{}，关联产品：{}", user_id, product_id);
}

fn main() {
    let uid = "user_123".to_string();
    let pid = "product_456".to_string();
    
    create_user(pid, uid); // ❌ 传反了！编译器不报错，逻辑错误
}
```

#### 正面例子（用 Newtype，编译器帮你把关）
```rust
// 用 Newtype 包装，创建两个完全不同的类型
#[derive(Debug, Clone)]
struct UserId(String);

#[derive(Debug, Clone)]
struct ProductId(String);

// 参数类型明确，传错直接编译报错
fn create_user(user_id: UserId, product_id: ProductId) {
    println!("创建用户：{:?}，关联产品：{:?}", user_id, product_id);
}

fn main() {
    let uid = UserId("user_123".to_string());
    let pid = ProductId("product_456".to_string());
    
    // create_user(pid, uid); // ❌ 编译报错！类型不匹配
    create_user(uid, pid);   // ✅ 正确
}
```

---

### 2. **绕过孤儿规则：为外部类型实现外部 Trait**

Rust 有个**孤儿规则（Orphan Rule）**：

> 你不能为「外部库的类型」实现「外部库的 Trait」。

但用 Newtype 包装一下，就能绕过这个限制！

#### 例子：给 `Vec<String>` 实现 `Display`

假设你想让 `Vec<String>` 能直接用 `println!("{}")` 打印，但孤儿规则不允许直接 `impl Display for Vec<String>`（因为 `Vec` 和 `Display` 都是标准库的）。

用 Newtype 解决：

```rust
// 用 Newtype 包装 Vec<String>
struct PrintableVec(Vec<String>);

// 现在可以为 PrintableVec 实现 Display 了！
impl std::fmt::Display for PrintableVec {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "[{}]", self.0.join(", ")) // self.0 访问内部的 Vec
    }
}

fn main() {
    let names = PrintableVec(vec!["Alice".to_string(), "Bob".to_string()]);
    println!("{}", names); // ✅ 输出：[Alice, Bob]
}
```

---

### 3. **零成本抽象：自定义类型的默认行为**

Newtype 是**零成本**的（编译后和直接用内部类型性能完全一样），你可以用它给原生类型「换个默认行为」。

#### 例子：给 `i32` 换个默认值

原生 `i32` 的默认值是 `0`，但你想要一个默认值是 `100` 的 `i32`：

```rust
#[derive(Debug, Clone, Copy)]
struct DefaultHundred(i32);

// 自定义 Default 实现
impl Default for DefaultHundred {
    fn default() -> Self {
        DefaultHundred(100) // 默认值是 100，不是 0
    }
}

fn main() {
    let num: DefaultHundred = Default::default();
    println!("{:?}", num); // ✅ 输出：DefaultHundred(100)
}
```

---

### 4. **隐藏内部实现细节，控制对外 API**

你可以用 Newtype 包装一个复杂的内部类型，**只暴露你想暴露的方法**，避免用户直接依赖内部实现。

#### 例子：包装一个内部的 HTTP 客户端

```rust
// 内部用的是 reqwest::Client，但对外不暴露
pub struct HttpClient(reqwest::Client);

impl HttpClient {
    // 对外只暴露「创建客户端」和「发送 GET 请求」两个方法
    pub fn new() -> Self {
        HttpClient(reqwest::Client::new())
    }

    pub async fn get(&self, url: &str) -> Result<String, reqwest::Error> {
        self.0.get(url).send().await?.text().await // self.0 访问内部的 reqwest::Client
    }
}
```

这样用户只能用你暴露的 API，你后续想把 `reqwest` 换成别的库，用户代码完全不用改！

---

## 二、`IntoInner` 及相关 Trait 的设计惯例

当你用 Newtype 包装类型时，通常需要提供「访问内部类型」的方法，Rust 社区有一套**约定俗成的 Trait 设计惯例**，我们按「所有权转移程度」从强到弱排列：

---

### 1. **`IntoInner`：彻底取出内部类型（转移所有权）**

这是你代码里用的模式，**核心特点**：

- 方法签名：`fn into_inner(self) -> Inner`（拿走 `self` 的所有权）；
- 效果：包装器被销毁，内部类型的所有权完全转移给调用者；
- 适用场景：用户不再需要包装器，只想用内部的原始类型。

#### 标准库中的例子
- `std::sync::Mutex<T>`：`fn into_inner(self) -> Result<T, PoisonError<T>>`
- `std::cell::RefCell<T>`：`fn into_inner(self) -> T`
- `std::boxed::Box<T>`：`fn into_inner(self) -> T`

### 2. **`AsRef` / `AsMut`：借用内部类型（不转移所有权）**

如果你想让用户**只读 / 修改内部类型，但不拿走所有权**，就实现 `AsRef`（只读借用）和 `AsMut`（可变借用）。

#### 例子：给 `UdpSocket` 实现 `AsRef` 和 `AsMut`

```rust
pub struct UdpSocket(net_imp::UdpSocket);

// 只读借用：&UdpSocket -> &net_imp::UdpSocket
impl AsRef<net_imp::UdpSocket> for UdpSocket {
    fn as_ref(&self) -> &net_imp::UdpSocket {
        &self.0
    }
}

// 可变借用：&mut UdpSocket -> &mut net_imp::UdpSocket
impl AsMut<net_imp::UdpSocket> for UdpSocket {
    fn as_mut(&mut self) -> &mut net_imp::UdpSocket {
        &mut self.0
    }
}

// 使用示例
fn main() {
    let mut socket: UdpSocket = ...;
    
    // 只读借用
    let inner_ref: &net_imp::UdpSocket = socket.as_ref();
    
    // 可变借用
    let inner_mut: &mut net_imp::UdpSocket = socket.as_mut();
    
    // socket 仍然有效，所有权没转移
}
```

### 3. **`Deref` / `DerefMut`：自动解引用（让包装器「像」内部类型）**

如果你想让包装器**用起来和内部类型一模一样**（自动调用内部类型的方法），就实现 `Deref`（解引用为不可变引用）和 `DerefMut`（解引用为可变引用）。

#### ⚠️ 注意：不要滥用 `Deref`！

`Deref` 是为「智能指针」设计的（比如 `Box<T>`、`Rc<T>`），如果你用它来做 Newtype 的「自动转换」，可能会**破坏类型安全**（比如之前的 `UserId` 和 `ProductId`，如果实现了 `Deref`，又能互相传了）。

#### 例子：给 `HttpClient` 实现 `Deref`（仅当包装器是「智能指针」时用）

```rust
use std::ops::{Deref, DerefMut};

pub struct HttpClient(reqwest::Client);

impl Deref for HttpClient {
    type Target = reqwest::Client; // 解引用后的目标类型

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl DerefMut for HttpClient {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}

// 使用示例：可以直接调用 reqwest::Client 的方法！
fn main() {
    let client = HttpClient::new();
    
    // 不用 client.0.get()，直接 client.get()！
    // 因为 Deref 自动把 &HttpClient 解引用为 &reqwest::Client
    let response = client.get("https://example.com");
}
```

---

### 4. **对比总结：什么时候用哪个？**

|Trait / 方法|所有权变化|适用场景|示例|
|---|---|---|---|
|`IntoInner`|转移所有权（包装器销毁）|彻底取出内部类型，不再用包装器|`socket.into_inner()`|
|`AsRef`|只读借用（不转移）|只需要看内部类型，不修改|`socket.as_ref()`|
|`AsMut`|可变借用（不转移）|需要修改内部类型，但保留包装器|`socket.as_mut()`|
|`Deref`|自动解引用（借用）|包装器是智能指针，想直接用内部类型的方法|`*client`|
