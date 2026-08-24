# CRTP核心思想
基类模板接受派生类本身作为参数，派生类继承该模板，***从而在编译时获得基类的方法***。
```cpp
template <typename Derived>
class Base {
public:
    void interface() {
        // 调用派生类的方法
        static_cast<Derived*>(this)->implementation();
    }
    // ...
};

class Derived : public Base<Derived> {
public:
    void implementation() {
        // ...
    }
};
```

# 具体案例
## 1. **静态多态（替代虚函数）**
- **目标**: 避免虚函数的动态分派开销，实现编译时多态。
- **示例**: [Shape](https://www.google.com/search?q=Shape&oq=cpp+CRTP%E7%9A%84%E5%85%B7%E4%BD%93%E6%A1%88%E4%BE%8B&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTIKCAEQABiABBiiBDIKCAIQABiiBBiJBTIHCAMQABjvBTIHCAQQABjvBTIKCAUQABiABBiiBNIBCTY1NzhqMGoxNagCCLACAfEF0sY7AcR8RnLxBdLGOwHEfEZy&sourceid=chrome&ie=UTF-8&mstk=AUtExfCtxbeQd0g1x9l-x85gQdPpFRMKtpNNqdAmJpBQG2onedAOF-0nn9G2lQDgi7IU_ZUYz_7E9Rn9FOW394ajsBaEv3KRUU-2piIaSYHb_O0ljTUqnyl_65eouU9Rvnk1c7tEMi8ojRIFtCoe3Fv95vaABSOiJiB0FVDfVVx654o1qKk&csui=3&ved=2ahUKEwj0jKvSuryRAxXVsFYBHcA2DqYQgK4QegQIBxAD)基类，派生类`Circle`, `Rectangle`实现`draw()`。
- **优势**: 无需虚表，性能更高。
```cpp
template<typename Derived>
struct Shape {
    void draw() const {
        static_cast<const Derived*>(this)->draw_impl();
    }
};

struct Circle : Shape<Circle> {
    void draw_impl() const { std::cout << "Drawing Circle\n"; }
};

// 在函数中直接调用
void renderShapes(const std::vector<Shape<void>*>& shapes) { // 传统虚函数
// void renderShapes(const std::vector<Circle>& shapes) { // CRTP
    for (const auto& shape : shapes) {
        shape.draw(); // 编译期确定，无虚函数开销
    }
}
```

## 2. **Mixin（混合模式）**
- **目标**: 共享通用行为，而不必是同一继承体系。
- **示例**: 添加[Observable](https://www.google.com/search?q=Observable&oq=cpp+CRTP%E7%9A%84%E5%85%B7%E4%BD%93%E6%A1%88%E4%BE%8B&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTIKCAEQABiABBiiBDIKCAIQABiiBBiJBTIHCAMQABjvBTIHCAQQABjvBTIKCAUQABiABBiiBNIBCTY1NzhqMGoxNagCCLACAfEF0sY7AcR8RnLxBdLGOwHEfEZy&sourceid=chrome&ie=UTF-8&mstk=AUtExfCtxbeQd0g1x9l-x85gQdPpFRMKtpNNqdAmJpBQG2onedAOF-0nn9G2lQDgi7IU_ZUYz_7E9Rn9FOW394ajsBaEv3KRUU-2piIaSYHb_O0ljTUqnyl_65eouU9Rvnk1c7tEMi8ojRIFtCoe3Fv95vaABSOiJiB0FVDfVVx654o1qKk&csui=3&ved=2ahUKEwj0jKvSuryRAxXVsFYBHcA2DqYQgK4QegQIBxAK) Mixin，让对象可被观察。
```cpp
template<typename T>
class Observable {
public:
    void notify(const std::string& event) {
        std::cout << typeid(T).name() << " notified: " << event << std::endl;
    }
};

class User : public Observable<User> {
    // ...
};

class Product : public Observable<Product> {
    // ...
};
```

## 3. **[对象计数器](https://www.google.com/search?q=%E5%AF%B9%E8%B1%A1%E8%AE%A1%E6%95%B0%E5%99%A8&oq=cpp+CRTP%E7%9A%84%E5%85%B7%E4%BD%93%E6%A1%88%E4%BE%8B&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTIKCAEQABiABBiiBDIKCAIQABiiBBiJBTIHCAMQABjvBTIHCAQQABjvBTIKCAUQABiABBiiBNIBCTY1NzhqMGoxNagCCLACAfEF0sY7AcR8RnLxBdLGOwHEfEZy&sourceid=chrome&ie=UTF-8&mstk=AUtExfCtxbeQd0g1x9l-x85gQdPpFRMKtpNNqdAmJpBQG2onedAOF-0nn9G2lQDgi7IU_ZUYz_7E9Rn9FOW394ajsBaEv3KRUU-2piIaSYHb_O0ljTUqnyl_65eouU9Rvnk1c7tEMi8ojRIFtCoe3Fv95vaABSOiJiB0FVDfVVx654o1qKk&csui=3&ved=2ahUKEwj0jKvSuryRAxXVsFYBHcA2DqYQgK4QegQIBxAO)**
- **目标**: 方便地为不同类添加引用计数/实例计数。
```cpp
template<typename T>
class Counter {
public:
    static int count;
    Counter() { count++; }
    ~Counter() { count--; }
    // ...
};

class MyClass : public Counter<MyClass> {
    // ...
};
```

## 4. **[接口强制实现](https://www.google.com/search?q=%E6%8E%A5%E5%8F%A3%E5%BC%BA%E5%88%B6%E5%AE%9E%E7%8E%B0&oq=cpp+CRTP%E7%9A%84%E5%85%B7%E4%BD%93%E6%A1%88%E4%BE%8B&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTIKCAEQABiABBiiBDIKCAIQABiiBBiJBTIHCAMQABjvBTIHCAQQABjvBTIKCAUQABiABBiiBNIBCTY1NzhqMGoxNagCCLACAfEF0sY7AcR8RnLxBdLGOwHEfEZy&sourceid=chrome&ie=UTF-8&mstk=AUtExfCtxbeQd0g1x9l-x85gQdPpFRMKtpNNqdAmJpBQG2onedAOF-0nn9G2lQDgi7IU_ZUYz_7E9Rn9FOW394ajsBaEv3KRUU-2piIaSYHb_O0ljTUqnyl_65eouU9Rvnk1c7tEMi8ojRIFtCoe3Fv95vaABSOiJiB0FVDfVVx654o1qKk&csui=3&ved=2ahUKEwj0jKvSuryRAxXVsFYBHcA2DqYQgK4QegQIBxAV) (Prototype/Policy)**
- **目标**: 确保派生类实现特定方法，否则编译时报错。
- **示例**: [Printable](https://www.google.com/search?q=Printable&oq=cpp+CRTP%E7%9A%84%E5%85%B7%E4%BD%93%E6%A1%88%E4%BE%8B&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTIKCAEQABiABBiiBDIKCAIQABiiBBiJBTIHCAMQABjvBTIHCAQQABjvBTIKCAUQABiABBiiBNIBCTY1NzhqMGoxNagCCLACAfEF0sY7AcR8RnLxBdLGOwHEfEZy&sourceid=chrome&ie=UTF-8&mstk=AUtExfCtxbeQd0g1x9l-x85gQdPpFRMKtpNNqdAmJpBQG2onedAOF-0nn9G2lQDgi7IU_ZUYz_7E9Rn9FOW394ajsBaEv3KRUU-2piIaSYHb_O0ljTUqnyl_65eouU9Rvnk1c7tEMi8ojRIFtCoe3Fv95vaABSOiJiB0FVDfVVx654o1qKk&csui=3&ved=2ahUKEwj0jKvSuryRAxXVsFYBHcA2DqYQgK4QegQIBxAY)接口。
```cpp
template<typename Derived>
struct Printable {
    void print() const {
        // 确保 Derived 提供了 'print_impl'
        static_cast<const Derived*>(this)->print_impl();
    }
};

struct Data : Printable<Data> {
    void print_impl() const { /* ... */ }
    // void print_impl_wrong() const { /* ... */ } // 缺少print_impl将导致编译错误
};
```

# enable_shared_from_this详解
## 总结
感觉最重要的还是：shared_ptr内部构造函数与enable_shared_from_this进行了感知！！！

## 基本概念
`enable_shared_from_this` 是一个**基类模板**，允许对象**安全地获取指向自己的 `shared_ptr`**，即使该对象已经由 `shared_ptr` 管理。
### 基本用法
```cpp
#include <memory>

class MyClass : public std::enable_shared_from_this<MyClass> {
public:
    std::shared_ptr<MyClass> get_shared() {
        return shared_from_this();  // 安全地获取 shared_ptr
    }
};

int main() {
    auto ptr = std::make_shared<MyClass>();
    auto ptr2 = ptr->get_shared();  // 正确：两个 shared_ptr 共享所有权
    return 0;
}
```
## 核心实现原理
### 友元函数
- 是在类外部定义的函数
- 它不是类的成员函数
- 但是可以访问类的所有成员
### 1. **类的定义**
```cpp
template<typename _Tp>
class enable_shared_from_this
{
protected:
    constexpr enable_shared_from_this() noexcept { }
    
    enable_shared_from_this(const enable_shared_from_this&) noexcept { }
    
    enable_shared_from_this& operator=(const enable_shared_from_this&) noexcept {
        return *this;
    }
    
    ~enable_shared_from_this() { }

public:
    shared_ptr<_Tp> shared_from_this() {
        return shared_ptr<_Tp>(this->_M_weak_this);
    }
    
    shared_ptr<const _Tp> shared_from_this() const {
        return shared_ptr<const _Tp>(this->_M_weak_this);
    }

private:
    // ⭐ 关键：弱引用指针
    mutable weak_ptr<_Tp> _M_weak_this;

    // ⭐ 关键：模板友元函数
    template<typename _Tp1>
    friend void _M_enable_shared_from_this_helper(
        const enable_shared_from_this*,
        const __shared_count<>&,
        enable_shared_from_this* __p) noexcept
    {
        if (__p && __p->_M_weak_this._M_empty()) {
            __p->_M_weak_this = _Tp1(__p, __n);
        }
    }
    
    // 另一个重载版本
    template<typename _Tp1>
    friend void _M_enable_shared_from_this_helper(
        const enable_shared_from_this*,
        const __shared_count<>&) noexcept
    {
        // 用于 const 版本
    }
};
```

## 关键机制
### 1. **弱引用指针 `_M_weak_this`**
```cpp
// 在 enable_shared_from_this 中：
mutable weak_ptr<_Tp> _M_weak_this;

// 作用：
// 1. 存储指向 this 的 weak_ptr
// 2. 当 shared_ptr 构造时被初始化
// 3. 用于创建额外的 shared_ptr
```
### 2. **友元函数 `_M_enable_shared_from_this_helper`**
```cpp
// 这个函数在 shared_ptr 构造函数中被调用
template<typename _Tp1>
friend void _M_enable_shared_from_this_helper(...)
{
    if (__p && __p->_M_weak_this._M_empty()) {
        // ⭐ 关键：初始化弱引用指针
        __p->_M_weak_this = _Tp1(__p, __n);
    }
}
```
## `shared_ptr` 如何与 `enable_shared_from_this` 协作

### `shared_ptr` 构造函数的关键部分
```cpp
template<typename _Tp>
class shared_ptr {
    template<typename _Yp>
    explicit shared_ptr(_Yp* __p)
        : _M_ptr(__p), _M_refcount(__p)
    {
        // ⭐ 关键：检查是否继承自 enable_shared_from_this
        typedef typename std::remove_cv<_Yp>::type _Yp_nc;
        if (__p && std::is_base_of<enable_shared_from_this<_Tp>, _Yp_nc>::value) {
            // 调用友元函数初始化 weak_ptr
            _M_enable_shared_from_this_helper(this->_M_ptr, this->_M_refcount, __p);
        }
    }
    
private:
    // 内部实现函数
    template<typename _Yp>
    static void _M_enable_shared_from_this_helper(
        const _Tp* __ptr,
        const __shared_count<>& __refcount,
        const enable_shared_from_this<_Yp>* __base) noexcept
    {
        if (__base) {
            __base->_M_weak_this._M_assign(const_cast<_Yp*>(__ptr), __refcount);
        }
    }
};
```
## 完整的工作原理流程
### 步骤 1：创建对象
```cpp
class MyClass : public std::enable_shared_from_this<MyClass> {
    // ...
};

int main() {
    // 步骤1：创建对象
    MyClass* raw_ptr = new MyClass();
    // 此时 _M_weak_this 是空的 !!!
}
```
### 步骤 2：创建第一个 `shared_ptr`
```cpp
std::shared_ptr<MyClass> ptr1(raw_ptr);
// shared_ptr 构造函数中：
// 1. 检测到 MyClass 继承自 enable_shared_from_this !!!
// 2. 调用 _M_enable_shared_from_this_helper
// 3. 初始化 raw_ptr->_M_weak_this
```
### 步骤 3：使用 `shared_from_this()`
```cpp
std::shared_ptr<MyClass> ptr2 = ptr1->shared_from_this();
// shared_from_this() 内部：
// return shared_ptr<_Tp>(this->_M_weak_this);
// 从 weak_ptr 创建 shared_ptr（增加引用计数）
```

## 手写简化实现
### 完整的简化实现
```cpp
#include <memory>
#include <iostream>

// 简化的 enable_shared_from_this
template<typename T>
class enable_shared_from_this {
protected:
    enable_shared_from_this() noexcept 
        : weak_this_() {}
    
    enable_shared_from_this(const enable_shared_from_this&) noexcept 
        : weak_this_() {}
    
    ~enable_shared_from_this() = default;

public:
    std::shared_ptr<T> shared_from_this() {
        // 从 weak_ptr 创建 shared_ptr
        if (weak_this_.expired()) {
            throw std::bad_weak_ptr();
        }
        return std::shared_ptr<T>(weak_this_);
    }
    
    std::shared_ptr<const T> shared_from_this() const {
        if (weak_this_.expired()) {
            throw std::bad_weak_ptr();
        }
        return std::shared_ptr<const T>(weak_this_);
    }

private:
    // 关键：存储 weak_ptr
    mutable std::weak_ptr<T> weak_this_;
    
    // 模板友元函数，供 shared_ptr 调用
    template<typename U, typename V>
    friend void internal_enable_shared_from_this(
        const std::shared_ptr<U>& sp,
        const enable_shared_from_this<V>* pe) noexcept;
};

// 友元函数的实现
template<typename U, typename V>
void internal_enable_shared_from_this(
    const std::shared_ptr<U>& sp,
    const enable_shared_from_this<V>* pe) noexcept
{
    if (pe != nullptr) {
        // 关键：将 weak_ptr 指向 shared_ptr 管理的对象
        const std::enable_shared_from_this<V>* p =
            static_cast<const std::enable_shared_from_this<V>*>(pe);
        const_cast<std::weak_ptr<V>&>(p->weak_this_) = sp;
    }
}

// 简化的 shared_ptr 实现（部分）
template<typename T>
class my_shared_ptr {
public:
    template<typename U>
    explicit my_shared_ptr(U* ptr) : ptr_(ptr), ref_count_(new int(1)) {
        // 检查是否继承自 enable_shared_from_this
        typedef typename std::remove_cv<U>::type U_nc;
        if (ptr && std::is_base_of<enable_shared_from_this<T>, U_nc>::value) {
            // 初始化 enable_shared_from_this 的 weak_ptr
            enable_shared_from_this<T>* base = 
                static_cast<enable_shared_from_this<T>*>(ptr);
            internal_enable_shared_from_this(*this, base);
        }
    }
    
private:
    T* ptr_;
    int* ref_count_;
};
```

## 为什么需要这样设计？
### 1. **避免多个独立的 `shared_ptr`**
```cpp
// ❌ 危险：创建多个独立的 shared_ptr
class BadClass {
public:
    std::shared_ptr<BadClass> get_bad() {
        return std::shared_ptr<BadClass>(this);  // 危险！
    }
};

int main() {
    auto p1 = std::make_shared<BadClass>();
    auto p2 = p1->get_bad();  // 💥 两个独立的控制块！
    // 对象会被销毁两次
}
```
### 2. **安全的共享所有权**
```cpp
// ✅ 安全：使用 enable_shared_from_this
class GoodClass : public std::enable_shared_from_this<GoodClass> {
public:
    std::shared_ptr<GoodClass> get_good() {
        return shared_from_this();  // 共享同一个控制块
    }
};

int main() {
    auto p1 = std::make_shared<GoodClass>();
    auto p2 = p1->get_good();  // ✅ 共享所有权
    // 引用计数为 2
}
```

## 重要的限制和注意事项
### 1. **必须在 `shared_ptr` 管理下使用**
```cpp
class MyClass : public std::enable_shared_from_this<MyClass> {};

int main() {
    MyClass obj;
    // ❌ 错误：对象不是由 shared_ptr 管理的
    auto ptr = obj.shared_from_this();  // 抛出 std::bad_weak_ptr
    
    // ✅ 正确：先由 shared_ptr 管理
    auto ptr1 = std::make_shared<MyClass>();
    auto ptr2 = ptr1->shared_from_this();  // 正确
}
```
### 2. **构造函数中不能使用**
```cpp
class MyClass : public std::enable_shared_from_this<MyClass> {
public:
    MyClass() {
        // ❌ 错误：此时 weak_this_ 还未初始化
        // auto ptr = shared_from_this();
    }
};
```
### 3. **多继承问题**
```cpp
// 只能从一个 enable_shared_from_this 继承
class Base1 : public std::enable_shared_from_this<Base1> {};
class Base2 : public std::enable_shared_from_this<Base2> {};

// ❌ 错误：多继承会导致 ambiguous
// class Derived : public Base1, public Base2 {};

// ✅ 正确：使用虚继承（小心使用）
class Derived : public Base1, public virtual Base2 {};
```

## 实现中的技术细节
### 1. **`mutable` 关键字**
```cpp
mutable weak_ptr<_Tp> _M_weak_this;
// mutable 允许在 const 成员函数中修改
// 因为 shared_from_this() 有 const 版本
```
### 2. **类型转换安全**
```cpp
// 在 shared_ptr 构造函数中：
if (std::is_base_of<enable_shared_from_this<_Tp>, _Yp_nc>::value) {
    // 确保类型正确转换
    enable_shared_from_this<_Tp>* base = 
        static_cast<enable_shared_from_this<_Tp>*>(ptr);
}
```
### 3. **线程安全**
```cpp
// weak_ptr 的赋值和读取是线程安全的
// 因为 shared_ptr/weak_ptr 使用原子操作
__p->_M_weak_this = _Tp1(__p, __n);  // 原子操作
```

## 性能考虑
### 1. **空间开销**
```cpp
// 每个 enable_shared_from_this 对象有一个 weak_ptr
// weak_ptr 通常包含两个指针：
// 1. 指向对象的指针
// 2. 指向控制块的指针
// 总计：2 * sizeof(void*) 的额外开销
```
### 2. **时间开销**
```cpp
// 创建 shared_ptr 时：
// 额外检查是否继承自 enable_shared_from_this
// 如果是，初始化 weak_ptr（一次性的）

// 调用 shared_from_this() 时：
// 从 weak_ptr 创建 shared_ptr（原子操作）
```

## 总结
**`enable_shared_from_this` 的核心原理**：
1. **存储弱引用**：在对象中存储 `weak_ptr<_Tp> _M_weak_this`
2. **友元初始化**：`shared_ptr` 构造函数通过友元函数初始化这个 `weak_ptr`
3. **安全创建**：`shared_from_this()` 从 `weak_ptr` 创建新的 `shared_ptr`
**关键设计点**：
- 使用 `weak_ptr` 避免循环引用
- 通过友元函数实现 `shared_ptr` 和 `enable_shared_from_this` 的协作
- 编译时类型检查确保安全
- 一次初始化，多次安全使用
**记住**：`enable_shared_from_this` 是一个"标记接口"，它本身不做太多事情，真正的魔法发生在 `shared_ptr` 的构造函数和 `shared_from_this()` 的协作中。