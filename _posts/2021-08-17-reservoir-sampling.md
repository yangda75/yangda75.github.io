---
layout: post
title: "蓄水池随机采样"
mathjax: true
tags: reservoir sampling, simple random sample, randomized algorithm
---

问题：
从输入流中随机选取k个元素，保证输入流中所有元素被选取的概率是一样的。

## 输入流，或者说，不知道输入到底有多大
如果有全部的输入，那么随机生成k个下标就可以了。

## 时间复杂度O(n), 空间复杂度O(k)
只能遍历一次流，或者说，只能按输入次序处理，并且不能保存之前的所有输入。

## 函数签名
用了`vector`表示流，这样代码简单一些。
```c++
std::vector<int> randomSample(std::vector<int> &v, const int k)
```
## 算法：
- 前`k`个元素直接放在样本里。
- 从第 `k+1`个元素开始，有概率替换样本中已有的元素。
在遍历到第`x`个元素时，(x > k)，选中这个元素的概率应为 `k/x`，用代码表示就是
```c++
if (rand() % x < k){
	// 选中
}
```
- 选中之后，替换样本中每个元素的概率都是一样的，所以随机生成一个下标，然后替换，下标可以直接用上面生成的随机数。

## 概率分析:
前提: x > k, 此时已经处理完了前x个元素
- 前k个元素此时在样本中的概率，即从k+1遍历到x的过程中，都没被替换掉的概率

$$p=\prod_{n = k+1}^{n = x}{(1- \frac{1}{k} * \frac{k}{n})} = \prod_{n = k+1}^{n=x}{\frac{n-1}{n}} = \frac{k}{x}$$

- 第k+1个元素到第x个元素在样本中的概率，即遍历到时被选中，且后续没有被替换的概率，设元素序号为m

$$p=\frac{k}{m}*\prod_{n = m+1}^{n = x}{(1 - \frac{1}{k} * \frac{k}{n})} = \frac{k}{m} * \frac{m}{x} = \frac{k}{x}$$

也就是说输入中每个元素被选中的概率都是 k/x
## 代码：
```c++
std::vector<int> randomSample(const std::vector<int> &v, const int k) {
    std::vector<int> sample{};
    for (auto i = 0; i < v.size(); i++) {
        if (i < k) {
            sample.push_back(v[i]);
        } else {
            auto x = rand() % i;
            if (i < k) {
                sample[x] = v[i];
            }
        }
    }
    return sample;
}
```