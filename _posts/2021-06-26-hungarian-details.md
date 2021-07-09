---
layout: post
title: "匈牙利匹配实现细节"
mathjax: true
tags: hungarian method, assignment problem, bipartite matching
---

最近又写了一遍匈牙利匹配，总结了一些细节
[参考](http://www.cse.ust.hk/~golin/COMP572/Notes/Matching.pdf)

## 一些概念和算法流程

### 一些概念
- Hungarian Method (Kuhn-Munkres Algorithm) 复杂度为 $$ O (\lvert V\rvert^3) $$ 
- 匹配, matching：一个边集合 $$ M \in E $$ 若满足：所有的点最多是其中一条边的端点，则$$ M $$为一个匹配
- 最大匹配, maximum matching：对所有的 匹配$$ M', \lvert M'\rvert \leq \lvert M\rvert$$ 的匹配 $$ M $$
- 完美匹配, perfect matching：$$ 2\lvert M \rvert = \lvert E\rvert $$ 的匹配$$ M $$
- 交替路, alternating path：$$ E $$ 为边集，$$ M $$ 为一个匹配，即 $$ E $$的一个子集，一条路径由一组边构成，若构成路径的这组边在 $$ M $$ 和 $$ E-M $$ 之间交替，则这条路径称为交替路
- 增广路, augmenting path：两个端点都不在匹配中的交替路
- 增广, augmenting：将增广路中的匹配边变为非匹配边，非匹配边变为匹配边的操作，每次增广都会使匹配中的点和边数量各加一
- 标记, labeling：带权图中的一个函数，定义域为点集，值域为实数集: $$ L : V \rightarrow R $$
- 可行标记, feasible labeling：满足 $$ L(x) + L(y) \geq W(x,y) $$ 的函数，（或者 $$ L(x) + L(y) \leq W(x,y) $$，求最大或最小匹配）
- 相等子图, equality graph：对标记 $$ L $$ 相等子图为: $$ G = (V, E_L), E_L=\{(x,y) : L(x) +L(y) = W(x,y)\} $$
- 一个定理：如果 $$ L $$ 是可行标记，且 $$ M $$ 是 $$ E_L $$ 中的完美匹配，则 $$ M $$为最大（或最小，由可行标记中的不等号方向决定）匹配。

### 算法
1. 寻找一个初始可行标记 $$ L $$，在 $$ E_L $$ 中寻找一个匹配 $$ M $$
2. 当 $$ M $$不是完美匹配时，重复：

    a. 在 $$E_L$$中寻找 $$ M $$的增广路，进行增广

    b. 如果找不到增广路，改进 $$ L $$扩大相等子图，返回 a.

## 完全图和无穷权重边
匈牙利匹配中要求一个完全二分图，但是实际使用中，有时需要处理不是完全二分图的情况，比如分配任务给机器人的时候，某些机器人不能执行任务。
处理方法是把不存在边设为无穷权重，这样既是完全图，也通过无穷大这个特殊值存储了不连通的信息。
需要注意一个细节，在构造相等子图（两点的标记值之和等于连同他们的边的权重的子图）时，不应该加入权重为无穷大的边，这样最后的匹配结果中就不会存在无穷权重的边了。
## 更新标记
在更新标记值时，需要计算一个最小变化量 alpha，对于求最大权重匹配和求最小权重匹配的两种情况，alpha的计算方法和更新方法是不一样的，因为两种情况下标记值和边权重的关系不同。

点集合为 $$ V = X \cup Y $$ , 边集合为 $$ E $$, 边的权重函数为 $$ W: X \times Y \rightarrow R $$, 标记为 $$ L: V \rightarrow R $$,

求最大权重匹配时的标记值L满足： $$ \forall x \in X, \forall y \in Y,  L(x) + L(y) \geq W(x,y) $$;

而求最小权重匹配时，L满足： $$ L(x) + L(y) \leq W(x,y) $$

所以对应的alpha计算方式也不同：

求最大权重匹配时：
$$ \alpha_L = min_{x\in S, y \notin T} \{L(x) + L(y) - W(x,y)\} $$

新的标记值 $$ L'(v) = \left\{ \begin{array}{ll}
L(v) + \alpha_L & \mbox{if }  v \in S \\ 
L(v) - \alpha_L & \mbox{if }  v \in T \\
L(v) & \mbox{otherwise} 
\end{array}\right.$$

求最小权重匹配时：
$$ \alpha_L = min_{x\in S, y \notin T} \{W(x,y) - L(x) - L(y)\} $$

新的标记值 $$ L'(v) = \left\{ \begin{array}{ll}
L(v) - \alpha_L & \mbox{if }  v \in S \\ 
L(v) + \alpha_L & \mbox{if }  v \in T \\
L(v) & \mbox{otherwise} 
\end{array}\right.$$

## 不平衡的处理
对于不平衡的问题，常用的处理方法是增加一个反向的图，使得问题变成对称的:

不平衡的二分图匹配问题： $$ G = (X, Y, E), \lvert X\rvert \neq \lvert Y \rvert $$, 增加X,Y的副本$$ X_c, Y_c $$变为平衡问题 $$ G' = ( X\cup Y_c , Y \cup X_c, E') $$ 。新增加的边的权重为:

 $$  \forall x \in X, \forall x' \in X_c, \forall y \in Y, \forall y' \in Y_c, \\
 \begin{array}{rcl} 
 W(x,x') & =& \infty \\ 
 W(y,y')& =& 0 \\ 
 W(y', x') & = &W(x, y) 
 \end{array}$$

