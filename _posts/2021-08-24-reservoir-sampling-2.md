---
layout: post
title: "再探蓄水池抽样"
tags: sampling, random algorithm
---

参考： [维基百科](https://en.wikipedia.org/wiki/Reservoir_sampling#Simple_algorithm)

## 起点
[上次的蓄水池抽样]({{ site.baseurl }}{% link
_posts/2021-08-17-reservoir-sampling.md %}) 只是最简单的实现。

维基百科说上次实现的简单算法叫 `Algorithm R`

```c++
// Algorithm R, O(n)
std::vector<int> randomSample(const std::vector<int> &v, const int k) {
    std::vector<int> sample{};
    for (auto i = 0; i < v.size(); i++) {
        if (i < k) {
            sample.push_back(v[i]);
        } else {
            // 对每个输入都要生成一个随机数
            auto x = rand() % i;
            if (i < k) {
                sample[x] = v[i];
            }
        }
    }
    return sample;
}
```

## 计算需要丢弃多少个元素
Algorithm L:

```c++
// 函数签名省略

// 先填满水池
for (auto i = 0; i < k; i++){
    sample.push_back(v[i]);
}

// 假设random() 生成从0到1的随机数
double w = exp(log(random()) / k);

int i = k;
while(i < n){
    // 按几何分布跳过
    i += floor(log(random())/log(1-w)) + 1
    
    if (i < n){
        // 更新水池
        sample[randInt(0,k)] = v[i];
        // 更新w
        w = w * exp(log(random()) / k)
    }
}
```

几何分布此处用来计算需要丢弃多少个元素才能找到下一个元素。即随机事件为，选中当前
的样本，选中概率为k/n，选中下一个元素之前需要进行多少次选择。

大概好像是这个意思，但是为啥这么算，我没看懂，复杂度是 O(k(1 + log(n/k)))，怎么算的我也不知道。此处记个TODO


## 从洗牌到Algorithm R
Fisher-Yates洗牌算法：
```c++
// Algorithm P
std::vector<int> shuffle(const std::vector<int> &v){
    std::vector<int> result(v.size(), v[0]);
    
    result[0] = v[0];
    for (int i = 1; i < v.size(); i++){
        int j = rand(0,i+1);
        result[i] = result[j];
        result[j] = v[i];
    }
}
```
随机抽k个样本可以理解为从洗完的牌堆顶上取k张牌。也就是洗牌算只输出前面k张洗好的牌。
```c++
std::vector<int> shuffleSample(const std::vector<int> &v, int k){
    std::vector<int> sample(k, v[0]);
    
    sample[0] = v[0];
    for (int i = 1; i < k; i++){
        int j = rand(0, i+1);
        sample[i] = sample[j];
        sample[j] = v[i];
    }
    
    // 将 [1,n) 截断成 [1,k) + [k,n),
    // 在后面半段不用关心sample[k,n)，
    // 也就是不用写sample[i] = sample[j]，
    // 只需要在j落在[1,k)的时候，执行sample[j] = v[i];

    for (int i = k; i < v.size(); i++){
        int j = rand(0, i+1);
        if(j < k){
            sample[j] = v[i];
        }
    }
}
```
而且输入的前k个顺序无所谓，所以直接可以保留前k个元素，这样就得到了Algorithm R
