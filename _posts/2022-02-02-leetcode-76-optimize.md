---
layout: post
title: "记一道算法题的优化：leetcode.76"
tags: algorithm, leetcode, c++, optimization
---

题目地址 https://leetcode-cn.com/problems/minimum-window-substring/

## 解法
看了第二个提示之后，确定用滑动窗口。
思路：
- 滑动窗口，左右两侧都只能向右滑
- 如果不匹配，右边界向右滑
- 如果匹配，左边界向右滑
- 结束条件，右边界超过字符串末尾

## 实现
解法不是特别有意思。但是这道题的实现，尤其是用c++，可以差出至少1000倍。我尝试了一系列写法，逻辑都是正确的，也都是上面说的解法，但是运行时间差距很大：
- 最慢的一个解法超时了
- 第一次改进之后AC, 256ms
- 第二次改进之后AC, 112ms
- 第三次改进之后AC, 92ms
- 第四次改进之后AC, 56ms
- 第五次改进之后AC, 4ms

最后一次改进参考了用时0ms的C++提交。

## 1. 超时的实现
```c++
class Solution {
public:
    // I
    bool containsAllChars(map<char, int> const& char_map, string str){
        map<char, int> char_map2{};
        for(char& c:str){
            char_map2[c]+=1;
        }
        for(auto const& entry : char_map){
            if(char_map2[entry.first] < entry.second){
                return false;
            }
        }
        return true;
    }

    string minWindow(string s, string t) {
        int left = 0;
        int len = 0;

        // 构造字符表
        map<char, int> char_map;
        for(char& c: t){
            char_map[c] += 1;
        }

        // 滑动窗口
        string result="";
        bool match = false;
        while(true){
            if(match){
                //cout<<"left: "<<left<<"\n";
                left++;len--;
            }else{
                //cout<<"len: "<<len<<"\n";
                len++;
            }
            if(left+len>s.size()){
                return result;
            }
            // II
            string new_str = s.substr(left, len);
            match = containsAllChars(char_map, new_str);
            if(match){
                if(new_str.size() < result.size() || result.size() == 0){
                    // III
                    result = new_str;
                }
            }
        }
        return result;
    }
};
```
具体实现逻辑：
1. 从字符串开头开始滑动
2. 根据上一个滑动窗口是否匹配, `match`判断此次是对窗口左侧还是右侧进行+1，即滑动操作
3. 滑动之后，根据窗口，构造新的字符串, `new_str`
4. 判断`new_str` 是否匹配，更新 `match`
5. 如果匹配，判断是否需要更新最短的匹配，`result`

有三个主要的优化点，代码中用 `I` ,`II`, `III`注释标注：
1. `containsAllChars` 用来判断新的滑动窗口是否匹配，需要输入目标字符计数表`char_map`，待检测字符串`str`。遍历字符串，生成实际字符计数表，再遍历字符计数表，判断两个表是否匹配
2. 每次滑动窗口都构造了新的字符串，可以不构造，直接用左右下标，操作原字符串
3. 每次最短匹配都更新了字符串，可以记录左右下标，最后返回时再构造

## 2. 256ms 的实现
```c++
class Solution {
public:
    // bool containsAllChars(map<char, int> const& char_map, string str){
    //     map<char, int> char_map2{};
    //     for(char& c:str){
    //         char_map2[c]+=1;
    //     }
    //     for(auto const& entry : char_map){
    //         if(char_map2[entry.first] < entry.second){
    //             return false;
    //         }
    //     }
    //     return true;
    // }
    bool debug = false;
    string minWindow(string s, string t) {
        if(debug) cout<<"s:"<<s<<", t:"<<t<<endl;
        int left = 0;
        int right = 0;

        // 构造字符表
        map<char, int> char_map;
        for(char& c: t){
            char_map[c] += 1;
        }
        map<char,int> char_map2;
        // 滑动窗口
        string result="";
        bool match = false;
        while(true){
            if(match){
                if(debug) cout<<"left: "<<left<<"\n";
                char_map2[s[left]]--;
                left++;
            }else{
                if(debug) cout<<"right: "<<right<<"\n";
                char_map2[s[right]]++;
                right++;
            }
            if(right>s.size()){
                return result;
            }
            // string new_str = s.substr(left, len);

            // match = containsAllChars(char_map, new_str);
            match = true;
            for(auto& entry: char_map){
                if(char_map2[entry.first]<entry.second){
                    if(debug) cout<<entry.first<<","<<entry.second<<","<<char_map2[entry.first]<<"\n";
                    match = false;
                    break;
                }
            }
            if(match){
                string new_str = s.substr(left, right-left);
                if(debug) cout<<"match"<<left<<","<<right<<","<<new_str<<"\n";
                if(new_str.size() < result.size() || result.size() == 0){
                    result = new_str;
                }
            }
        }
        return result;
    }
};
```
优化了三个可优化项中的前两项:
1. 去掉了`containsAllChars` 这个专门用来判断滑动窗口是否匹配的函数，直接在`while` 循环中维护待滑动窗口的字符计数表
2. 不再对每个窗口构造新的字符串，改为滑动时维护字符计数表，也能完成判断是否匹配的动作

但是还是很慢，运行时间和内存占用都排在后5%

## 3. 112ms 的实现
```c++
class Solution {
public:
    bool debug = false;
    string minWindow(string s, string t) {
        if(debug) cout<<"s:"<<s<<", t:"<<t<<endl;
        int left = 0;
        int right = 0;

        // 构造字符表
        map<char, int> char_map;
        for(char& c: t){
            char_map[c] += 1;
        }
        map<char,int> char_map2;
        // 滑动窗口
        string result="";
        bool match = false;
        while(true){
            if(match){
                if(debug) cout<<"left: "<<left<<"\n";
                char_map2[s[left]]--;
                left++;
            }else{
                if(debug) cout<<"right: "<<right<<"\n";
                char_map2[s[right]]++;
                right++;
            }
            if(right>s.size()){
                return result;
            }
            // string new_str = s.substr(left, len);

            // match = containsAllChars(char_map, new_str);
            match = true;
            for(auto& entry: char_map){
                if(char_map2[entry.first]<entry.second){
                    if(debug) cout<<entry.first<<","<<entry.second<<","<<char_map2[entry.first]<<"\n";
                    match = false;
                    break;
                }
            }
            if(match && (right-left<result.size() || result.size()==0)){
                string new_str = s.substr(left, right-left);
                if(debug) cout<<"match"<<left<<","<<right<<","<<new_str<<"\n";
                result = new_str;
            }
        }
        return result;
    }
};
```
主要的优化的项目是匹配之后的处理：

之前的实现中，每次`match`为真时，都构造了一个新的字符串，其实可以根据长度判断是否需要更新`result` 字符串
```c++
// 从：
            if(match){
                string new_str = s.substr(left, right-left);
                if(debug) cout<<"match"<<left<<","<<right<<","<<new_str<<"\n";
                if(new_str.size() < result.size() || result.size() == 0){
                    result = new_str;
                }
            }
// 改为：
            if(match && (right-left<result.size() || result.size()==0)){
                string new_str = s.substr(left, right-left);
                if(debug) cout<<"match"<<left<<","<<right<<","<<new_str<<"\n";
                result = new_str;
            }
```
内存占用也从360m 降到了16m

## 4. 56ms的实现
```c++
class Solution {
public:
    bool debug = false;
    string minWindow(string s, string t) {
        if(debug) cout<<"s:"<<s<<", t:"<<t<<endl;
        int left = 0;
        int right = 0;

        // 构造字符表
        array<int, 128> char_map{};
        for(char& c: t){
            char_map[c] += 1;
        }
        // 滑动窗口
        int min_start=0;
        int min_len=0;
        bool match = false;
        while(true){
            if(match){
                if(debug) cout<<"left: "<<left<<"\n";
                char_map[s[left]]++;
                left++;
            }else{
                if(debug) cout<<"right: "<<right<<"\n";
                char_map[s[right]]--;
                right++;
            }
            if(right>s.size()){
                break;
            }
            if(right<t.size()) continue;
            // string new_str = s.substr(left, len);

            // match = containsAllChars(char_map, new_str);
            match = true;
            for(int i=0; i<128; i++){
                if(char_map[i]>0){
                    match = false;
                    if(debug) cout<<i<<","<<char_map[i]<<endl;
                    break;
                }
            }
            if(match && (right-left<min_len || min_len==0)){
                min_len = right-left;
                min_start = left;
                if(debug) cout<<"match "<<min_start<<","<<min_len<<","<<s.substr(min_start, min_len)<<"\n";
            }
        }
        return min_len == 0 ? "" : s.substr(min_start, min_len);
    }
};
```
改进了第三个可优化项，在循环过程中不构造字符串，而是记录结果字符串的开始下表和长度，跳出循环后再构建。
此外，顺便将`map` 改为了`array` ，并且去掉了`s`的字符计数表，直接操作`t`的字符计数表，也能判断是否匹配。

## 5. 4ms的实现
```c++
class Solution {
public:
    bool debug = false;
    string minWindow(string s, string t) {
        if(debug) cout<<"s:"<<s<<", t:"<<t<<endl;
        int left = 0;
        int right = 0;

        // 构造字符表
        array<int, 128> char_map{};
        array<bool, 128> char_flag{};
        for(char& c: t){
            char_map[c] += 1;
            char_flag[c] = true;
        }
        // 滑动窗口
        int min_start=0;
        int min_len=0;
        bool match = false;
        int cnt=0; // cnt == t.size() 时匹配，需要配合bool数组标记t中是否有这个字母
        while(true){
            if(match){
                if(debug) cout<<"left: "<<left<<"\n";
                char_map[s[left]]++;
                if(char_map[s[left]]>0 && char_flag[s[left]]) cnt--;
                left++;
            }else{
                if(debug) cout<<"right: "<<right<<"\n";
                char_map[s[right]]--;
                // 没有flag数组无法判断是否有这个字母
                if(char_map[s[right]]>=0 && char_flag[s[right]]) cnt++;
                right++;                
            }
            if(right>s.size()){
                break;
            }
            if(right<t.size()) continue;
            // string new_str = s.substr(left, len);

            // match = containsAllChars(char_map, new_str);
            match = cnt>=t.size();
            // for(int i=0; i<128; i++){
            //     if(char_map[i]>0){
            //         match = false;
            //         if(debug) cout<<i<<","<<char_map[i]<<endl;
            //         break;
            //     }
            // }
            if(match && (right-left<min_len || min_len==0)){
                min_len = right-left;
                min_start = left;
                if(debug) cout<<"match "<<min_start<<","<<min_len<<","<<s.substr(min_start, min_len)<<"\n";
            }
        }
        return min_len == 0 ? "" : s.substr(min_start, min_len);
    }
};
```
56ms的实现中，有一个明显的问题：每次循环都要遍历字符计数表
```c++
            match = true;
            for(int i=0; i<128; i++){
                if(char_map[i]>0){
                    match = false;
                    if(debug) cout<<i<<","<<char_map[i]<<endl;
                    break;
                }
            }
```
想了一会儿没想到怎么去掉这个遍历，就去看了0ms的c++提交，力扣这个功能很赞。

思路是用一次大小比较来判断是否匹配：`match = cnt>=t.size()`，这个`cnt`，描述起来比较麻烦：

当前滑动窗口中和目标字符串匹配的字符数，两个相同的字符算作两个不同的字符

如：滑动窗口为 `"ABC"` ,t为`"CCC"` 时，cnt=1

### 维护cnt:
滑动时，需要维护`cnt`
- 左侧边界++时，如果之前左侧边界的字符在`t` 中，且字符计数表中对应项++之后大于零了，就需要对`cnt`减一
- 右侧边界++时，如果新右侧边界的字符在`t` 中，且字符计数表中对应项--之后仍然大于等于零，需要对`cnt`加一

字符串计数表中对应项++表示增加一个待匹配字符，--表示减少一个待匹配字符，对所有字符，待匹配数量都<=0 了，两个字符串就匹配了

例子：
当前滑动窗口为 `"ATBC"` ，`t` 为`"ABCCC"`

- 此时`char_map[C]=2`, 因为窗口中只有一个`C`, t中有三个，所以还需要两个`C` ，待匹配数量是2
- `char_map[A]=0`, 因为窗口有一个`A`, t中也有一个，不再需要了
- `cnt` = 3, 窗口中和`t`匹配的字母有三个,`A`,`B`,`C`

窗口右侧向右滑动后，为 `"ATBCC"` 
- `char_map[C]=1`
- `cnt` = 4, `A`,`B`,`C`,`C`

如果窗口为 `"AATBCCCC"` , `t` 仍为 `"ABCCC"`, 此时`cnt`为5,`match`为true

窗口左侧向右滑动，得到`"ATBCCCC"`
- `char_map[A]=0` 从-1 变成0，不需要对cnt--，因为`A` 还是匹配的
所以有上面的大于零条件

窗口右侧向右滑动，得到`"ATBCCCCC"`
- `char_map[C]=-2` 从-1变成-2，不需要对cnt++，因为`C` 已经匹配了

如果窗口从`"ATBCC"` 滑动到 `"ATBCCC"` 
- `char_map[C]=0` 从1变成0，因为缺少的`C`的数目从1变成0了，所以有上面的大于等于零条件

此外，还需要一个`flag`数组，标识字符是否在 `t` 中，因为通过修改后的`char_map`是无法得到这个信息的。

## 总结
相当于自己对自己进行了一个code review。手法还得练