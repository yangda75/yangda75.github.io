---
layout: post
title: "C++ for循环的一个有趣的例子"
tags: c++, for, fun
---

## for(xx : xxx)
c++ 11 开始的一个特性。可以冒号来遍历序列(range)中的元素。比如
```c++
for (int a : int_arr){
  // ...
}
```

c++的range-based for loop 翻译称序列应该合理。

## 什么是序列(range)?
之父的书 "The C++ Programming Language" 第四版第9章中说：
- 能够通过 `v.begin()` , `v.end()` 组合或者 `begin(v)` `end(v)` 获取迭代器的对象叫做range

一个很好的SO帖子：
https://stackoverflow.com/questions/8164567/how-to-make-my-custom-type-to-work-with-range-based-for-loops

## 不使用`vector` `array` 甚至数组，通过 `v.begin()` `v.end()` 的组合实现一个好玩的序列
比如打印几个朝代

作为序列的对象的类叫 `Dynasty`,迭代器类型为 `Dynasty::iterator`, 用`std::string` 表示朝代
需要实现:
- `Dynasty::iterator Dynasty::begin()`
- `Dynasty::iterator Dynasty::end()`
- `Dynasty::iterator Dynasty::iterator::operator++()`
- `bool Dynasty::iterator::operator==(const Dynasty::iterator& other)`
- `std::string& Dynasty::iterator::operator*()`

使用时:
```c++
    Dynasty dynasty_list{};
    for(auto& dynasty:dynasty_list){
        std::cout<<dynasty<<std::endl;
    }
```
打印出:
```
tang
song
yuan
ming
```

挺好玩，但是实现代码很难看:
```c++
#include<string>
using std::string;
// 不用vector, array, [], 实现range-based for loop

// 几个朝代
string tang{"tang"};
string song{"song"};
string yuan{"yuan"};
string ming{"ming"};

// “容器”
class Dynasty {
  public:
    class iterator {
      public:
        iterator(int i) : index(i) {}
        iterator operator++() {
            ++index;
            return *this;
        }
        bool operator!=(const iterator &other) const { return index != other.index; }
        // 关键部分，通过index判断该返回哪个朝代，决定了begin() end() 里面该怎么传index
        string &operator*() const {
            switch (index) {
            case 0:
                return tang;
            case 1:
                return song;
            case 2:
                return yuan;
            case 3:
                return ming;
            default:
                throw "WRONG INDEX";
            }
        }

      private:
        int index;
    };
    iterator begin() {return iterator(0);} // index 的第一个合法值是0
    iterator end() {return iterator(4);} // 4 个朝代
};
```

