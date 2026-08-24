# 完整实例：带精准报错的 `#[derive(AssertDebug)]` 宏

我们来写一个**实用的过程宏**：

- 功能：自动检查结构体的**所有字段都实现了 `Debug`**
- 亮点：如果某个字段没实现 `Debug`，**报错红线会精准画在那个字段上**（靠 `Spanned` + `quote_spanned!` 实现）

---

## 第一步：创建过程宏 Crate

### `Cargo.toml`
```toml
[package]
name = "assert-debug"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true

[dependencies]
syn = { version = "2.0", features = ["full", "parsing"] }
quote = "1.0"
proc-macro2 = "1.0"
```

---

## 第二步：完整的宏实现代码（`src/lib.rs`）
```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Data, DeriveInput, Fields};
use syn::spanned::Spanned; // 【关键】导入 Spanned trait

/// 自动检查结构体所有字段都实现了 Debug
///
/// # Example
///
/// ```
/// use assert_debug::AssertDebug;
///
/// #[derive(AssertDebug)]
/// struct MyStruct {
///     name: String, // 实现了 Debug ✅
///     ptr: *const (), // 没实现 Debug ❌ 报错会精准画在这里
/// }
/// ```
#[proc_macro_derive(AssertDebug)]
pub fn derive_assert_debug(input: TokenStream) -> TokenStream {
    // 1. 用 syn 解析用户输入的代码
    let input = parse_macro_input!(input as DeriveInput);
    
    // 2. 拿到结构体的名字（Ident）
    let struct_name = input.ident;

    // 3. 提取结构体的字段列表
    let fields = match input.data {
        Data::Struct(data_struct) => match data_struct.fields {
            Fields::Named(fields_named) => fields_named.named,
            Fields::Unnamed(fields_unnamed) => fields_unnamed.unnamed,
            Fields::Unit => {
                // 单元结构体没有字段，直接返回空
                return TokenStream::from(quote! {});
            }
        },
        _ => {
            // 只支持结构体，不支持 enum/union
            panic!("AssertDebug only supports structs");
        }
    };

    // 4. 遍历每个字段，生成对应的 Debug 约束
    let mut asserts = Vec::new();
    for field in fields {
        // 【核心 1】获取当前字段的 Span！
        // 这是 Spanned trait 提供的方法
        let field_span = field.span();
        
        // 拿到字段的类型
        let field_ty = field.ty;

        // 【核心 2】用 quote_spanned! 生成代码
        // 把 span 绑定到 field_span，这样报错会精准落在这个字段上
        let assert = quote_spanned!(field_span=>
            // 生成一个私有结构体，用 where 约束检查 Debug
            struct _AssertDebugField where #field_ty: std::fmt::Debug;
        );
        
        asserts.push(assert);
    }

    // 5. 把所有生成的断言拼在一起，返回
    let generated = quote! {
        #(#asserts)*
    };

    TokenStream::from(generated)
}
```

---

## 第三步：用户如何使用这个宏

在另一个普通 Rust 项目里：

### `Cargo.toml`
```rust
[dependencies]
assert-debug = { path = "../assert-debug" }
```

### `src/main.rs`
```rust
use assert_debug::AssertDebug;

// 加上我们的宏
#[derive(AssertDebug)]
struct MyStruct {
    name: String,        // ✅ 实现了 Debug
    age: u32,            // ✅ 实现了 Debug
    ptr: *const (),      // ❌ 没实现 Debug！
}

fn main() {}
```

---

## 第四步：运行效果（精准报错！）

当你编译用户项目时，会看到：
```txt
error[E0277]: `*const ()` doesn't implement `Debug`
 --> src/main.rs:8:5
  |
8 |     ptr: *const (),
  |     ^^^^^^^^^^^^^^ `*const ()` cannot be formatted using `{:?}`
  |
  = help: the trait `Debug` is not implemented for `*const ()`
  = note: required for `*const ()` to implement `AssertDebug`
```

### 关键点：

**报错红线精准画在了 `ptr: *const (),` 这一行！**

这就是 `field.span()` + `quote_spanned!` 的威力。

---

# 代码里的核心知识点回顾

|代码片段|作用|
|---|---|
|`use syn::spanned::Spanned;`|导入 Spanned trait，让我们能调用 `.span()`|
|`let field_span = field.span();`|从字段节点获取它的源代码位置|
|`quote_spanned!(field_span=> ...)`|生成代码，并强制把 Span 设为字段的位置|

---

# 总结

1. **`Spanned`**：一键获取任何语法树节点的 Span
2. **`quote_spanned!`**：用指定的 Span 生成代码，控制报错位置
3. **完整流程**：解析 → 提取字段 → 获取 Span → 生成约束 → 返回

这就是一个**工业级质量**的过程宏的核心结构！