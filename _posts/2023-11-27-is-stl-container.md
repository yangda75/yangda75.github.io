---
layout: post
title: "is_stl_container"
tags: c++, template, stl, container
---

# 如何判断模板类型参数是否是STL容器？
目标：实现 `is_stl_container_v<T>`, 如果 `T` 是一个stl容器，返回true，否则返回false
## STL中有哪些容器
[cppreference](https://en.cppreference.com/w/cpp/container)

根据文档，C++ 11 开始，STL中的容器分为三类，
- 顺序，Squence，包括 `vector`, `array`, `deque`, `forward_list`, `list`
- 关联，Associative，包括 `set`, `map`, `multiset`, `multimap`
- 无序关联，Unordered associative，包括 `unordered_set`, `unordered_map`, `unordered_multiset`, `unordered_multimap` .

- 除去上面三种容器，还有 “容器适配器”，Container Adaptors, 它们提供访问顺序容器的接口，包括 `stack`, `queue`, `priority_queue`
C++23中还加入了 `flat` 前缀的一组，包括 `flat_set`, `flat_map`, `flat_mulitset`, `flat_multimap`
- 此外，在C++20中新增的 `span`, C++23中新增的`mdspan` ,属于 “视图”，View，提供对不持所有权的一组数据的访问接口


## Named requirements
[cppreference](https://en.cppreference.com/w/cpp/named_req)

在C++标准中，描述对标准库实现的期望。
C++20中新增了 `concept`。Named requirements比concept早，可以用 `concept` 来写 named requirements.
目前（23年）标准库没有描述容器的概念，不过可以根据 named requirements 中的内容自己定义 [SO](https://stackoverflow.com/q/60449592)

## 标准库容器的 named requirements
[cppreference](https://en.cppreference.com/w/cpp/named_req/Container)

假设容器类型(`vector`, `list`等)为 `C`, 容器的类型参数即容器中元素的类型为 `T`，即，
容器的完整类型为 `C<T>`

标准库容器的named requirements，即，标准对于标准库的实现要求包括下面几个部分：
1. 类型定义：
   - 容器需要有`value_type`, `reference`,
     `const_reference`,`iterator`,`const_iterator`,`difference_type`,`size_type`
     这样的类型定义，即，对于容器 `C<T>` ，需要有`C<T>::value_type`, `C<T>::等`reference
   - 类型定义需要符合要求，如：`C<T>::value_type` 需要等于 `T` ； `T` 需要能够拷
     贝构造(另一个named requirements: `CopyConstructible`)； `C<T>::size_type` 要
     是无符号整数，且要能够表示 `C<T>::difference_type`的所有正数取值，等
2. 成员函数和操作符：
   - 函数/操作符的签名、返回值、前后条件、时间复杂度、行为都有规定
   - 构造、赋值、析构：支持无参构造，支持拷贝构造、移动构造，支持拷贝赋值、移动赋值，析构时销毁容器内所有元素并释放内存
   - 迭代器：支持获取begin/end, cbegin/cend
   - 状态、判断：支持判断两个容器是否相同，支持size,max_size,empty
   - 支持 swap
3. 线程安全
4. 对容器和元素类型需要满足的 named requirements


## C++14中的一个初步实现
使用SFINAE，配合标准对于容器的实现要求，即named requirements: Container 中的内容，可以实现一个粗糙的版本：
```c++
template <typename... T>
using void_t = void;

template <typename T, typename U>
constexpr bool is_same_v = std::is_same<T, U>::value;

#define I2T_HAS_XXX_TRAIT_DEF(name)                                                                                    \
    template <typename T, typename = void>                                                                             \
    constexpr bool has_##name = false;                                                                                 \
    template <typename T>                                                                                              \
    constexpr bool has_##name<T, void_t<typename T::name>> = true;

I2T_HAS_XXX_TRAIT_DEF(value_type)
I2T_HAS_XXX_TRAIT_DEF(iterator)
I2T_HAS_XXX_TRAIT_DEF(const_iterator)
I2T_HAS_XXX_TRAIT_DEF(size_type)

template <typename T, typename = void>
constexpr bool is_iterable = false;
template <typename T>
constexpr bool
    is_iterable<T,
                void_t<std::enable_if_t<is_same_v<decltype(std::declval<T>().begin()), typename T::iterator>>,
                       std::enable_if_t<is_same_v<decltype(std::declval<T>().end()), typename T::iterator>>>> = true;

template <typename T, typename = void>
constexpr bool is_const_iterable = false;
template <typename T>
constexpr bool is_const_iterable<
    T,
    void_t<std::enable_if_t<is_same_v<decltype(std::declval<T>().cbegin()), typename T::const_iterator>>,
           std::enable_if_t<is_same_v<decltype(std::declval<T>().cend()), typename T::const_iterator>>>> = true;

template <typename T, typename = void>
constexpr bool is_sizeable = false;
template <typename T>
constexpr bool
    is_sizeable<T, void_t<std::enable_if_t<is_same_v<decltype(std::declval<T>().size()), typename T::size_type>>>> =
        true;

template <typename T>
constexpr bool is_stl_container_like_v = has_value_type<T>&& has_iterator<T>&& has_const_iterator<T>&&
    has_size_type<T>&& is_iterable<T>&& is_const_iterable<T>&& is_sizeable<T>;
```

上面的实现中，判断了下列内容：
- 是否有 `value_type`
- 是否有 `iterator`
- 是否有 `const_iterator`
- 是否有 `size_type`
- 是否有 `begin()` `end()` 方法, 返回值是否为 `iterator`
- 是否有 `cbegin()` `cend()` 方法，返回值是否为 `const_iterator`
- 是否有 `size()` 方法，返回值是否为 `size_type`

一些实现细节：
- `I2T_HAS_XXX_TRAIT_DEF` 是一个 x-macro, 参考了boost库里实现类似trait判断的代码，
- `void_t` 和 `is_same_v` 都在 C++17才加入std，所以抄过来了

## TODO
- 补全对于类型的测试
- 尝试使用C++20的concept实现
