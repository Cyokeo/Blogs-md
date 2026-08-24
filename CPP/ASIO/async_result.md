`async_result.hpp`

```cpp
template <typename CompletionToken, ASIO_COMPLETION_SIGNATURE... Signatures> 
class async_result : public conditional_t<  
   detail::are_any_lvalue_completion_signatures<Signatures...>::value || 
   !detail::are_any_rvalue_completion_signatures<Signatures...>::value, 
detail::completion_handler_async_result<CompletionToken, Signatures...>, async_result<CompletionToken, typename detail::simple_completion_signature<Signatures>::type...> >
{
	public: 
		typedef conditional_t< 
			detail::are_any_lvalue_completion_signatures<Signatures...>::value || 
			!detail::are_any_rvalue_completion_signatures<Signatures...>::value, 
		detail::completion_handler_async_result<CompletionToken, Signatures...>, 
		async_result<CompletionToken, typename 
		detail::simple_completion_signature<Signatures>::type...> > base_type;
		using base_type::base_type; private: async_result(const async_result&) = delete; 
		async_result& operator=(const async_result&) = delete;
}; 

template <ASIO_COMPLETION_SIGNATURE... Signatures> 
class async_result<void, Signatures...> { // Empty. }; 

template <typename... Signatures> 
class async_result<detail::async_operation_probe, Signatures...> 
{
public:
	typedef detail::async_operation_probe_result return_type;
	template <typename Initiation, typename... InitArgs>
	static return_type initiate(Initiation&&, detail::async_operation_probe, InitArgs&&...)
	{ return return_type(); }
}; 

template <typename... Signatures> 
class async_result<detail::completion_signature_probe, Signatures...>
{ 
public:
	typedef detail::completion_signature_probe_result<Signatures...> return_type; 
	template <typename Initiation, typename... InitArgs>
	static return_type initiate(Initiation&&, detail::completion_signature_probe, InitArgs&&...)
	{ return return_type();} 
}; 

template <typename Signature> 
class async_result<detail::completion_signature_probe, Signatures...>
{ 
public:
	typedef detail::completion_signature_probe_result<Signatures...> return_type; 
	template <typename Initiation, typename... InitArgs>
	static return_type initiate(Initiation&&, detail::completion_signature_probe, InitArgs&&...)
	{ return return_type();} 
}; 
```

## 1. **主模板：递归继承结构**
### 1.1 **条件继承逻辑**
```cpp
// 关键条件：
detail::are_any_lvalue_completion_signatures<Signatures...>::value
|| !detail::are_any_rvalue_completion_signatures<Signatures...>::value

// 解释：
// 如果 Signatures 中有左值引用签名 → 选择基类A
// 或者 Signatures 中没有右值引用签名 → 选择基类A
// 否则（全是右值引用签名）→ 递归处理
```
### 1.2 **递归处理右值引用签名**
```cpp
// 当全是右值引用签名时，递归调用：
async_result<CompletionToken,
  typename detail::simple_completion_signature<Signatures>::type...>

// simple_completion_signature 去掉引用限定符：
// void(int)&& → void(int)
// void(int)&  → void(int)

// 递归后再次判断：
// 去掉 && 后就没有右值引用签名了
// 所以条件变为 true，选择基类A，递归终止
```
### 1.3 **设计目的**
```cpp
// 统一处理接口：
// 最终所有 async_result 特化都继承自：
// detail::completion_handler_async_result<...>

// 这样用户特化只需要处理一种情况
// 内部自动处理引用限定符的复杂性
```

## 2. **特化1：void 作为 CompletionToken**
```cpp
template <ASIO_COMPLETION_SIGNATURE... Signatures>
class async_result<void, Signatures...>
{
  // Empty.
};
```
**作用**：

- 当 CompletionToken 为 `void` 时提供一个空实现
- 可能是为了**编译时检测**或**占位**使用
- 防止模板实例化错误

## 3. **特化2：async_operation_probe 探测**
```cpp
template <typename... Signatures>
class async_result<detail::async_operation_probe, Signatures...>
{
public:
  typedef detail::async_operation_probe_result return_type;

  template <typename Initiation, typename... InitArgs>
  static return_type initiate(Initiation&&,
      detail::async_operation_probe, InitArgs&&...)
  {
    return return_type();
  }
};
```
### 3.1 **设计目的：编译时检测**
```cpp
// 用于检测某个表达式是否是异步操作
// 配合 is_async_operation trait 使用：

template <typename T, typename... Args>
struct is_async_operation :
  detail::is_async_operation_call<
    T(Args..., detail::async_operation_probe)>
{
};

// 工作原理：
// 1. 尝试调用 T(Args..., async_operation_probe)
// 2. 如果 T 是异步操作，会调用 async_initiate
// 3. async_initiate 查找 async_result<async_operation_probe, ...>
// 4. 匹配此特化，返回 async_operation_probe_result
// 5. 检测成功
```
### 3.2 **使用示例**
```cpp
// 检测 async_read 是否是异步操作
static_assert(
    is_async_operation<
        decltype(&asio::async_read),
        asio::ip::tcp::socket&,
        asio::mutable_buffer
    >::value,
    "async_read should be an async operation"
);
```

## 4. **特化3：completion_signature_probe 探测**
### 4.1 **多个签名的版本**
```cpp
template <typename... Signatures>
class async_result<detail::completion_signature_probe, Signatures...>
{
public:
  typedef detail::completion_signature_probe_result<Signatures...> return_type;
  // ...
};
```
### 4.2 **单个签名的版本**
```cpp
template <typename Signature>
class async_result<detail::completion_signature_probe, Signature>
{
public:
  typedef detail::completion_signature_probe_result<Signature> return_type;
  // ...
};
```
### 4.3 **设计目的：获取完成签名**
```cpp
// 用于 completion_signature_of trait：
template <typename T, typename... Args>
struct completion_signature_of :
  result_of_t<T(Args..., detail::completion_signature_probe)>
{
};

// 工作原理：
// 1. 尝试调用 T(Args..., completion_signature_probe)
// 2. 如果 T 是异步操作，调用 async_initiate
// 3. async_initiate 查找 async_result<completion_signature_probe, ...>
// 4. 匹配特化，返回 completion_signature_probe_result<Signatures...>
// 5. 从中提取 Signatures...
```



