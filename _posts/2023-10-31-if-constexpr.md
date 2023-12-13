---
layout: post
title: "if constexpr"
tags: c++, constexpr, if
---

# C++ 17
以 `if constexpr` 开头的语句称为 *constexpr if statement*. 不符合条件的分支会被丢弃(原文discard)。

```c++
template <typename T>
auto get_value(T t){
    if constexpr (std::is_pointer_v<T>)
        return *t;
    else
        return t;
}
```
上面的代码中，判断了 `std::is_pointer_v<T>` 的值，如果为true，返回 `*t` ，否则返回 `t`。
如果不使用 `if constexpr` ，上边的代码无法通过编译，因为此时两个分支的代码都不会丢弃，都要检查，所以有一个分支会导致编译错误。

除此之外，还有几处需要注意：
- 在被丢弃的分支中，可以使用未定义的变量。
- 在模板之外，丢弃的分支也会被检查
- 在模板中，如果实例化之后，条件不是 value-dependent 的，那么被丢弃的语句不会在模板实例化时实例化
- 如果被丢弃的分支对所有的特化都是错误(ill-form)的，也会报错

# C++14
没有 `if constexpr` 需要"模拟"一个。
[source](https://baptiste-wicht.com/posts/2015/07/simulate-static_if-with-c11c14.html)
[boost mailing list](https://lists.boost.org/Archives/boost/2014/08/216607.php)
```c++
namespace static_if_detail {

struct identity {
    template<typename T>
    T operator()(T&& x) const {
        return std::forward<T>(x);
    }
};

template<bool Cond>
struct statement {
    template<typename F>
    void then(const F& f){
        f(identity());
    }

    template<typename F>
    void else_(const F&){}
};

template<>
struct statement<false> {
    template<typename F>
    void then(const F&){}

    template<typename F>
    void else_(const F& f){
        f(identity());
    }
};

} //end of namespace static_if_detail

template<bool Cond, typename F>
static_if_detail::statement<Cond> static_if(F const& f){
    static_if_detail::statement<Cond> if_;
    if_.then(f);
    return if_;
}

```

使用样例：

```
template<typename T>
void assign(T& x, T const& y){
    x = y;
    static_if<boost::has_equal_to<T>::value>([](auto f){
        assert(f(x)==f(y));
        std::cout << "asserted for: "<< typeid(T).name() << std::endl;
    })
    .else_([](auto){std::cout << "cannot assert for: "<<typeid(T).name() << std::endl;});
}
```
## 实现解析
<!-- ### 原理 -->
<!-- SFINAE，如果模板参数为false，就不解析对应的代码 -->
### 为什么要把条件放在模板里？
为了在编译时判断
### 为什么在 `static_if` 最后，返回了构造的 `static_if_statement`?
为了支持下面的用法：
 `static_if<condtition>(f1).else_(f2)`
如果不返回，就不能加else
### 为什么要用 `identity` ？
如果不加，使用时，需要在lambda 中捕获局部变量，这样会导致编译错误。使用bind可
以避免编译错误，但是bind不推荐使用。使用identity函数之后，non-dependent type变成
dependent type，即，实例化或者特化后，才能判断lambda中的语句是否正确。
这个 `identity` 写法类似c++20的 `std::identity`

进阶问题:

- 为什么可以传入 `identity()` 到 `f` ? 

  `identity()` 构造了一个 `identity` 类的对象，它有 `template<T> T operator()` ，不会导致编译错误。
- identity 能不能写成函数模板？

  ```
  template<typename T>
  T identity(T&& t){
      return std::forward<T>(t);
  }
  ```
  我不知道怎么用这种方式写上面的代码。
  原来代码中那种 `identity` 称为"functor" 。用 functor 的写法，在传入函数对象时，不需要传入类型参数。
