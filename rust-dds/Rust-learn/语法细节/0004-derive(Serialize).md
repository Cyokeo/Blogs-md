# `#[derive(Serialize)]` 完整工作原理：结合 `serde_derive` 源码与生成代码

`#[derive(Serialize)]` 是 Rust **过程宏（Proc Macro）** 的经典应用，核心流程是：

1. **编译期**：`serde_derive` 宏读取你的结构体 / 枚举定义，生成 `impl Serialize for YourType` 的代码；
2. **运行期**：调用生成的 `serialize` 方法，通过 `Serializer` trait 完成序列化。

我们结合 **`serde_derive` 源码逻辑**和**实际生成的代码**，一步步拆解。

---

## 一、前置知识：Rust Derive 宏基础

`#[derive(Trait)]` 是 Rust 的**派生宏**，属于过程宏的一种，规则是：

- 必须定义在单独的 `proc-macro` crate 中（`serde_derive` 就是这样的 crate）；
- 输入：结构体 / 枚举的 AST（抽象语法树）；
- 输出：生成的 `impl Trait for YourType` 代码。

---

## 二、核心流程：`#[derive(Serialize)]` 的 4 个步骤

我们用你熟悉的 `HelloWorldData` 作为例子：
```rust
// 你的代码
#[derive(Serialize)]
struct HelloWorldData {
    user_id: i32,
    message: String,
}
```

`serde_derive` 处理这段代码的完整流程如下：

---

### 步骤 1：解析输入 AST（用 `syn` 库）

`serde_derive` 的入口函数会接收编译器传来的 `TokenStream`（原始代码流），然后用 **`syn`** 库解析成结构化的 AST。

#### `serde_derive` 入口逻辑（简化版源码）
```rust
// serde_derive/src/lib.rs（简化示意）
extern crate proc_macro;
use proc_macro::TokenStream;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(Serialize, attributes(serde))]
pub fn derive_serialize(input: TokenStream) -> TokenStream {
    // 1. 用 syn 解析输入的 TokenStream 为 DeriveInput（结构化的 AST）
    let input = parse_macro_input!(input as DeriveInput);
    
    // 2. 分析 AST，生成 Serialize impl 的代码
    let expanded = serialize::expand_derive_serialize(&input);
    
    // 3. 返回生成的代码
    TokenStream::from(expanded)
}
```

这里的 `DeriveInput` 就是解析后的结构体 / 枚举定义，包含：

- 类型名称：`HelloWorldData`
- 字段信息：`user_id: i32`、`message: String`
- 属性：`#[serde(...)]` 等

---

### 步骤 2：分析字段与属性（处理 `#[serde(...)]`）

`serde_derive` 会遍历每个字段，检查是否有 `#[serde(...)]` 属性（比如 `rename`、`skip`、`default` 等），并记录下来。

#### 核心分析逻辑（简化示意）
```rust
// serde_derive/src/ser.rs（简化示意）
pub fn expand_derive_serialize(input: &DeriveInput) -> TokenStream {
    // 1. 提取结构体/枚举的信息
    let ident = &input.ident; // "HelloWorldData"
    let data = &input.data;
    
    // 2. 分析字段：提取字段名、类型、serde 属性
    let fields = match data {
        syn::Data::Struct(data_struct) => {
            analyze_fields(&data_struct.fields)
        }
        _ => panic!("Only structs are supported in this example"),
    };
    
    // 3. 生成 impl Serialize 的代码
    quote! {
        impl Serialize for #ident {
            fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
            where
                S: Serializer,
            {
                // ... 生成的序列化逻辑 ...
            }
        }
    }
}
```

---

### 步骤 3：生成序列化代码（用 `quote` 库）

这是最关键的一步：`serde_derive` 用 **`quote`** 库生成实际的 `impl Serialize` 代码。

对于你的 `HelloWorldData`，**实际生成的代码**（可以用 `cargo expand` 命令看到）如下：!!!!!

#### 生成的 `impl Serialize for HelloWorldData`（真实代码）
```rust
// 这是 #[derive(Serialize)] 自动生成的代码！
impl serde::Serialize for HelloWorldData {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        // 1. 告诉 Serializer：我要序列化一个结构体
        // 结构体名："HelloWorldData"，字段数：2
        let mut state = serializer.serialize_struct("HelloWorldData", 2)?;
        
        // 2. 序列化第 1 个字段：user_id
        serde::ser::SerializeStruct::serialize_field(
            &mut state,
            "user_id",
            &self.user_id,
        )?;
        
        // 3. 序列化第 2 个字段：message
        serde::ser::SerializeStruct::serialize_field(
            &mut state,
            "message",
            &self.message,
        )?;
        
        // 4. 结束序列化
        serde::ser::SerializeStruct::end(state)
    }
}
```

---

### 步骤 4：运行期调用（结合你之前的 `CdrSerializer`）

当你实际调用 `data.serialize(serializer)` 时，执行的就是上面生成的代码，流程如下：
1. **生成的代码**调用 `serializer.serialize_struct("HelloWorldData", 2)`；
2. 你的 `CdrSerializer` 实现了 `Serializer` trait，所以返回一个实现了 `SerializeStruct` 的对象（就是 `&mut CdrSerializer` 自己）；
3. **生成的代码**调用 `SerializeStruct::serialize_field`，也就是你之前看到的：    
    ```rust
    fn serialize_field<T>(&mut self, value: &T) -> Result<()> {
        value.serialize(&mut **self) // 把字段序列化到底层 CdrSerializer
    }
    ```
4. 最后调用 `end`，完成序列化。

---

## 三、关键工具库：`syn` 和 `quote`

`serde_derive` 能工作，完全依赖这两个核心库：

| 库           | 作用               | 在 `serde_derive` 中的用途                      |
| ----------- | ---------------- | ------------------------------------------ |
| **`syn`**   | 解析 Rust 代码为 AST  | 把你的 `struct HelloWorldData` 解析成结构化数据       |
| **`quote`** | 把 AST 转回 Rust 代码 | 生成 `impl Serialize for HelloWorldData` 的代码 |

---

## 四、如何自己看生成的代码？（`cargo expand`）

你可以用 `cargo expand` 命令直接看到 `#[derive(Serialize)]` 生成的代码：

### 1. 安装 `cargo expand`

```bash
cargo install cargo-expand
```

### 2. 在你的项目中运行

```bash
cargo expand --lib your_module
```

它会输出所有宏展开后的代码，包括 `serde_derive` 生成的 `impl Serialize`。

---

## 五、总结：`#[derive(Serialize)]` 完整流程

|阶段|工具|动作|
|---|---|---|
|**编译期**|`serde_derive` + `syn`|解析你的结构体 / 枚举定义|
|**编译期**|`serde_derive` + `quote`|生成 `impl Serialize for YourType` 的代码|
|**运行期**|生成的代码 + `Serializer`|调用 `serialize_struct`/`serialize_field` 完成序列化|