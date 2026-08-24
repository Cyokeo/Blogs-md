## 一、标签分发（Tag Dispatching）

### 1. **基本概念**

标签分发是一种**编译时多态**技术，通过使用**空结构体标签**来选择不同的函数重载。

### 2. **基本实现**
```cpp
// 定义标签类型
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : input_iterator_tag {};
struct bidirectional_iterator_tag : forward_iterator_tag {};
struct random_access_iterator_tag : bidirectional_iterator_tag {};

// 使用标签分发选择算法
template<typename Iterator>
void advance_impl(Iterator& it, typename std::iterator_traits<Iterator>::difference_type n,
                  input_iterator_tag) {
    // 线性前进
    while (n-- > 0) ++it;
}

template<typename Iterator>
void advance_impl(Iterator& it, typename std::iterator_traits<Iterator>::difference_type n,
                  random_access_iterator_tag) {
    // 随机访问，直接跳转
    it += n;
}

// 主函数
template<typename Iterator>
void advance(Iterator& it, typename std::iterator_traits<Iterator>::difference_type n) {
    using category = typename std::iterator_traits<Iterator>::iterator_category;
    advance_impl(it, n, category{});  // 标签分发
}
```
## 二、特性萃取（Type Traits）
### 1. **基本概念**
特性萃取是一种**编译时类型信息查询**技术，用于获取类型的各种特性。
### 2. **核心类型特性**
```cpp
// 基础类型特性
template<typename T>
struct is_integral : std::false_type {};

template<>
struct is_integral<int> : std::true_type {};
template<>
struct is_integral<short> : std::true_type {};
// ... 其他整型特化

// 复合类型特性
template<typename T>
struct is_pointer : std::false_type {};

template<typename T>
struct is_pointer<T*> : std::true_type {};

// 引用特性
template<typename T>
struct is_lvalue_reference : std::false_type {};

template<typename T>
struct is_lvalue_reference<T&> : std::true_type {};
```

## 三、实际应用案例

### 案例 1：**智能指针的 make_unique 实现**
```cpp
#include <memory>
#include <type_traits>

// 特性萃取：检查是否为数组
template<typename T>
struct is_unbounded_array : std::false_type {};

template<typename T>
struct is_unbounded_array<T[]> : std::true_type {};

template<typename T>
struct is_bounded_array : std::false_type {};

template<typename T, std::size_t N>
struct is_bounded_array<T[N]> : std::true_type {};

// 标签分发实现
namespace detail {
    // 非数组版本
    template<typename T, typename... Args>
    std::unique_ptr<T> make_unique_impl(std::false_type, Args&&... args) {
        return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
    }
    
    // 无界数组版本
    template<typename T>
    std::unique_ptr<T[]> make_unique_impl(std::true_type, std::size_t size) {
        return std::unique_ptr<T[]>(new std::remove_extent_t<T>[size]());
    }
    
    // 有界数组不允许
    template<typename T, typename... Args>
    std::unique_ptr<T> make_unique_impl(is_bounded_array<T>, Args&&...) = delete;
}

// 统一的 make_unique
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return detail::make_unique_impl<T>(
        is_unbounded_array<T>{},  // 标签分发
        std::forward<Args>(args)...);
}

// 使用
auto p1 = make_unique<int>(42);           // 非数组
auto p2 = make_unique<int[]>(10);         // 无界数组
// auto p3 = make_unique<int[10]>();       // 编译错误：有界数组
```
### 案例 2：**序列化框架**
```cpp
#include <iostream>
#include <type_traits>
#include <vector>
#include <string>

// 标签定义
struct arithmetic_tag {};
struct string_tag {};
struct container_tag {};
struct pointer_tag {};
struct custom_tag {};

// 特性萃取：获取类型标签
template<typename T>
struct type_traits {
    using tag = typename std::conditional<
        std::is_arithmetic<T>::value, arithmetic_tag,
        typename std::conditional<
            std::is_same<T, std::string>::value, string_tag,
            typename std::conditional<
                is_container<T>::value, container_tag,
                typename std::conditional<
                    std::is_pointer<T>::value, pointer_tag,
                    custom_tag
                >::type
            >::type
        >::type
    >::type;
};

// 容器检查特性
template<typename T>
struct is_container : std::false_type {};

template<typename T, typename Alloc>
struct is_container<std::vector<T, Alloc>> : std::true_type {};

// 标签分发实现
template<typename T>
void serialize_impl(const T& value, arithmetic_tag) {
    std::cout << "Arithmetic: " << value << std::endl;
}

template<typename T>
void serialize_impl(const T& value, string_tag) {
    std::cout << "String: \"" << value << "\"" << std::endl;
}

template<typename T>
void serialize_impl(const T& container, container_tag) {
    std::cout << "Container[";
    for (const auto& item : container) {
        serialize(item);
        std::cout << ", ";
    }
    std::cout << "]" << std::endl;
}

template<typename T>
void serialize_impl(const T* ptr, pointer_tag) {
    if (ptr) {
        std::cout << "Pointer -> ";
        serialize(*ptr);
    } else {
        std::cout << "Pointer: nullptr" << std::endl;
    }
}

// 主函数
template<typename T>
void serialize(const T& value) {
    using tag = typename type_traits<T>::tag;
    serialize_impl(value, tag{});
}

// 使用
int main() {
    serialize(42);                           // Arithmetic
    serialize(3.14);                         // Arithmetic
    serialize(std::string("Hello"));         // String
    serialize(std::vector<int>{1, 2, 3});    // Container
    int x = 100;
    serialize(&x);                           // Pointer
    return 0;
}
```

### 案例 3：**算法优化（STL风格）**
```cpp
#include <iterator>
#include <type_traits>
#include <algorithm>

// 距离计算优化
namespace detail {
    // 标签：随机访问迭代器（高效）
    template<typename Iterator>
    typename std::iterator_traits<Iterator>::difference_type
    distance_impl(Iterator first, Iterator last, std::random_access_iterator_tag) {
        return last - first;  // O(1)
    }
    
    // 标签：输入迭代器（通用）
    template<typename Iterator>
    typename std::iterator_traits<Iterator>::difference_type
    distance_impl(Iterator first, Iterator last, std::input_iterator_tag) {
        typename std::iterator_traits<Iterator>::difference_type n = 0;
        while (first != last) {
            ++first;
            ++n;
        }
        return n;  // O(n)
    }
}

// 统一的 distance
template<typename Iterator>
typename std::iterator_traits<Iterator>::difference_type
distance(Iterator first, Iterator last) {
    using category = typename std::iterator_traits<Iterator>::iterator_category;
    return detail::distance_impl(first, last, category{});
}

// 复制算法优化
namespace detail {
    // 特性萃取：检查是否可平凡复制
    template<typename T>
    using is_trivially_copyable = std::is_trivially_copyable<T>;
    
    // 标签：可平凡复制（使用 memcpy）
    template<typename InputIt, typename OutputIt>
    OutputIt copy_impl(InputIt first, InputIt last, OutputIt d_first,
                      std::true_type /* trivially_copyable */) {
        std::size_t n = std::distance(first, last);
        if (n > 0) {
            std::memcpy(&*d_first, &*first, n * sizeof(*first));
        }
        return d_first + n;
    }
    
    // 标签：不可平凡复制（使用循环）
    template<typename InputIt, typename OutputIt>
    OutputIt copy_impl(InputIt first, InputIt last, OutputIt d_first,
                      std::false_type /* not trivially_copyable */) {
        while (first != last) {
            *d_first++ = *first++;
        }
        return d_first;
    }
}

// 优化的 copy
template<typename InputIt, typename OutputIt>
OutputIt optimized_copy(InputIt first, InputIt last, OutputIt d_first) {
    using value_type = typename std::iterator_traits<InputIt>::value_type;
    return detail::copy_impl(first, last, d_first,
                            detail::is_trivially_copyable<value_type>{});
}

// 使用示例
int main() {
    // 基础类型数组（使用 memcpy）
    int src1[] = {1, 2, 3, 4, 5};
    int dst1[5];
    optimized_copy(std::begin(src1), std::end(src1), dst1);
    
    // 复杂类型（使用循环）
    std::string src2[] = {"a", "b", "c"};
    std::string dst2[3];
    optimized_copy(std::begin(src2), std::end(src2), dst2);
    
    return 0;
}
```
## 四、高级技巧：编译时策略选择

### 策略模式 + 标签分发
```cpp
#include <iostream>
#include <type_traits>

// 策略标签
struct linear_search_tag {};
struct binary_search_tag {};
struct hash_search_tag {};

// 特性萃取：选择搜索策略
template<typename Container>
struct search_strategy_traits {
    using tag = typename std::conditional<
        has_random_access<Container>::value, binary_search_tag,
        typename std::conditional<
            has_hash_function<Container>::value, hash_search_tag,
            linear_search_tag
        >::type
    >::type;
};

// 容器特性检查
template<typename T>
struct has_random_access : std::false_type {};

template<typename T>
struct has_random_access<std::vector<T>> : std::true_type {};

template<typename T>
struct has_hash_function : std::false_type {};

template<typename Key, typename Value>
struct has_hash_function<std::unordered_map<Key, Value>> : std::true_type {};

// 策略实现
namespace detail {
    template<typename Container, typename Value>
    auto search_impl(const Container& cont, const Value& val, linear_search_tag) {
        std::cout << "Using linear search\n";
        return std::find(cont.begin(), cont.end(), val);
    }
    
    template<typename Container, typename Value>
    auto search_impl(const Container& cont, const Value& val, binary_search_tag) {
        std::cout << "Using binary search\n";
        return std::lower_bound(cont.begin(), cont.end(), val);
    }
    
    template<typename Container, typename Value>
    auto search_impl(const Container& cont, const Value& val, hash_search_tag) {
        std::cout << "Using hash search\n";
        return cont.find(val);
    }
}

// 统一的search
template<typename Container, typename Value>
auto smart_search(const Container& cont, const Value& val) {
    using tag = typename search_strategy_traits<Container>::tag;
    return detail::search_impl(cont, val, tag{});
}

// 使用
int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    auto it1 = smart_search(vec, 3);  // 使用二分查找
    
    std::list<int> lst = {1, 2, 3, 4, 5};
    auto it2 = smart_search(lst, 3);  // 使用线性查找
    
    std::unordered_map<int, std::string> map = {{1, "a"}, {2, "b"}};
    auto it3 = smart_search(map, 2);  // 使用哈希查找
    
    return 0;
}
```

## 五、性能对比和选择指南

|技术|编译时开销|运行时开销|适用场景|
|---|---|---|---|
|**标签分发**|低|零开销|根据类型选择不同算法|
|**特性萃取**|中|零开销|获取类型信息，条件编译|
|**虚函数**|低|有开销（虚表）|运行时多态|
|**if constexpr**|中|零开销|C++17+ 简单条件编译|

## 六、最佳实践
1. **优先使用标签分发**而非运行时多态，当行为在编译时已知时
    
2. **将特性萃取与标签分发结合**实现复杂的编译时策略
    
3. **使用 `if constexpr`** 简化简单的条件编译（C++17+）
    
4. **保持标签类型简单**，使用空结构体
    
5. **为常用类型组合预定义特性**，提高编译速度