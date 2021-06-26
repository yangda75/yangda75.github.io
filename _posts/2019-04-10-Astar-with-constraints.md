---
layout: post
title: "带约束的A*搜索"
tags: algorithm, A*, constraints
---
# 带约束(时间地点人物)的A*搜索
在A\*中，有时需要添加一些约束(constraint)，表示在某个时间点，不能出现在某个地点。为了实现这个功能，需要在A\*中做一些修改。

## `Constraint`类
首先要搞一个类来描述约束。这个类里面要有时间、地点、人物这三个元素。然后还要重载一下`==`，方便比较不同的约束。

```c++
struct Constraint {
    int agent;
    GridPoint point;
    int constraintTimeStamp;
    Constraint() {}
    Constraint(int a, GridPoint p, int t)
        : agent(a), point(p), constraintTimeStamp(t) {}
    Constraint(GridPoint p, int t) : point(p), constraintTimeStamp(t) {}
    bool operator==(const Constraint &cc) const {
        return cc.agent == agent and cc.point == point and
               cc.constraintTimeStamp == constraintTimeStamp;
    }
};
```
## `search`函数
算法的主体部分`search`函数，和不带约束的A\*是一样的。搞一个`openSet`，一个`closedSet`，开始搜，搜到`openSet`空或者到达目标点为止。
每次都进行扩展，往`openSet`里面加东西。一般都把这个扩展的过程整成一个单独的函数。加入约束主要修改扩展的过程。
## 扩展过程
A\*的扩展过程：
1. 输入一个点;
2. 找到从这个点可以一步到达的点，存起来，比如存到`pointList`里面;
3. 从`pointList`里删掉实际上不能到达的点;
4. 返回`pointList`。  

要改的是第三步，不带约束的算法里，实际上不能到达的点就是位置不随时间变化的障碍物，加上约束之后就多了一种情况：被约束的点。由于约束是有时间戳的，所以输入也要加一个时间戳。其实时间戳也可以搞成一个全局变量，这样代码可以少改几行。
```c++
std::vector<GridPoint> AStar::getAdjacentGridPoints(GridPoint &p) {
    std::vector<GridPoint> adjGridPoints;
    int xx[] = {-1, 0, 1};
    int yy[] = {-1, 0, 1};
    for (int dx : xx)
        for (int dy : yy)
            if (std::abs(dx + dy) == 1) {
                int x = dx + p.x;
                int y = dy + p.y;
                if (x >= 0 and x < dimX and y >= 0 and y < dimY) {
                    GridPoint newPoint(x, y);
                    if (std::find(obstacles.begin(), obstacles.end(),
                                  newPoint) == obstacles.end()) {
                        bool flag = true;
                        // 看看有没有被约束
                        for (Constraint c : constraints)
                            if (c.point == newPoint and
                                c.constraintTimeStamp == timeStamp + 1)
                            // 被约束了，不能走
                            {
                                std::cout << "constrained, not adding point "
                                          << newPoint.x << "," << newPoint.y
                                          << "\n";
                                flag = false;
                                break;
                            }
                        if (flag) adjGridPoints.push_back(newPoint);
                    }
                }
            }
    // 没地方去，等着
    if (adjGridPoints.empty()) {
        adjGridPoints.push_back(p);
    }
    return adjGridPoints;
}
```
## 其他
还有一些需要注意的部分，但是实在太细节，就没有写出来。   
这篇文章的内容很短也没有什么难度，也不会有很多人用得到，只是自己在写代码的时候解决的一个小问题，记录一下。
