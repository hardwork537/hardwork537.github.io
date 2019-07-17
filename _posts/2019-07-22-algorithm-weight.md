---
layout:     post
title:      加权平滑轮询算法
category: algorithm
tags: [algorithm]
description: 介绍一种GBN中使用到的、脱胎于nginx的weighed round robin算法
---


# 需求

当我们需要把一份数据发送到一个Set中的任意机器的时候，很容易想到的一个问题是，如何挑Set中的机器作为数据的接收方？显然算法需要符合以下要求：

1. 支持加权，以便在机器故障时可以降低其权重
2. 在加权的前提下，尽可能地把请求平摊到每台机器上

第一点很好理解，而第二点的意思是，比如说我们现在有a, b, c三个选择，权重分别是5, 1, 1，我们希望输出的结果是类似于a, a, b, a, c, a, a，而不是a, a, a, a, a, b, c。


# 可选方案

## 随机

非常容易想到的第一个算法是随机，以上面的5、1、1为例，我们挑选一个随机数x，取值范围是[0, 7)，那么当x属于[0, 5)则返回a，当x属于[5, 6)返回b，当x属于[6, 7)则返回c，以此类推。

随机算法的优点是简单容易理解，缺点是具有不可预测性，在一次完整的轮询中，有可能权重为1的b完全没被选中，而权重为5的a则返回大于5次，当然如果轮询的次数足够大则每个元素被选中的次数占比会符合其权重。随机算法符合我们的要求1，勉强符合要求2。

## LVS的算法

LVS中实现了一种基于最大公约数的轮询算法，伪代码大致如下：

```
/*
Supposing that there is a server set S = {S0, S1, …, Sn-1};
W(Si) indicates the weight of Si;
i indicates the server selected last time, and i is initialized with -1;
cw is the current weight in scheduling, and cw is initialized with zero; 
max(S) is the maximum weight of all the servers in S;
gcd(S) is the greatest common divisor of all server weights in S;
*/
while (true) {
    i = (i + 1) mod n;
    if (i == 0) {
        cw = cw - gcd(S); 
        if (cw <= 0) {
            cw = max(S);
            if (cw == 0)
            return NULL;
        }
    } 
    if (W(Si) >= cw) 
        return Si;
}
```

该算法对于权重为4, 3, 2的A, B, C可以得到这样一个序列：AABABCABC。我们可以从数学上简单证明一下该算法的正确性。

```
假设有三个服务器ABC，并且
W(A) = x
W(B) = y
W(C) = z

假设xyz的最大公约数为g，且有
x = ag
y = bg
z = cg

那么对于一次完整的轮询（x + y + z），由算法可以知道，只有当

cw <= y = (a - b) g

算法才会选择B，同理只有当

cw <= z = (a - c) g
算法才会选择C，并且在一次选择B的循环中，必定会选择A，在一次会选择C的循环中，必定会选择AB，于是可以得到在一次完整的轮询中，

选择A的次数为 (a - b)g + (b - c)g + cg = ag = x
选择B的次数为 (b - c)g + cg = bg = y
选择C的次数为 cg = z
```

LVS的算法非常简洁明了，相比随机算法有很好的可预测性，然而在5, 1, 1这种极端case中，该算法返回的序列是

```
AAAAABC
```

秉着精益求精的精神，我们希望可以找到更好的算法，并在偶然中发现Nginx中的

## 平滑加权轮询算法(smooth weighted round robin algorithm)

从Github上面可以看到，Nginx以前也是使用和LVS类似的算法，并在某一次提交中修改为当前的算法，该算法大致思想如下：

```
Upstream: smooth weighted round-robin balancing.

For edge case weights like { 5, 1, 1 } we now produce { a, a, b, a, c, a, a }
sequence instead of { c, b, a, a, a, a, a } produced previously.

Algorithm is as follows: on each peer selection we increase current_weight
of each eligible peer by its weight, select peer with greatest current_weight
and reduce its current_weight by total number of weight points distributed
among peers.

In case of { 5, 1, 1 } weights this gives the following sequence of
current_weight's:

     a  b  c
     0  0  0  (initial state)

     5  1  1  (a selected)
    -2  1  1

     3  2  2  (a selected)
    -4  2  2

     1  3  3  (b selected)
     1 -4  3

     6 -3  4  (a selected)
    -1 -3  4

     4 -2  5  (c selected)
     4 -2 -2

     9 -1 -1  (a selected)
     2 -1 -1

     7  0  0  (a selected)
     0  0  0
```

该算法除了有权重weight，还引入了另一个变量current_weight，在每一次遍历中会把current_weight加上weight的值，并选择current_weight最大的元素，对于被选择的元素，再把current_weight减去所有权重之和。

同样地我们可以从数学上证明该算法的正确性：

```
假设有三个服务器ABC，并且
W(A) = x
W(B) = y
W(C) = z

那么在一次完整的轮询中，ABC的current_weight总共被加的数值分别是：
c(A) = x (x + y + z)
c(B) = z (x + y + z)
c(C) = y (x + y + z)

由于每次被选中都会被减去total = x + y + z，所以ABC在一次轮询中被选中的次数是：
A = c(A) / total = x (x + y + z) / (x + y + z) = x
B = c(B) / total = y (x + y + z) / (x + y + z) = y
C = c(C) / total = z (x + y + z) / (x + y + z) = z

另外由于每次选中current_weight都会减去total，所以可以保证对于权重最大的A，不会被连续选中x次。证明如下：

假设前k次选择都是选择A，那么会有

(k+1) x - k (x + y + z) > 0

即 x - k (y + z) > 0

即 k < x / (y + z)

因为y + z大于1，所以k < x，所以A不可能被连续选择x次
```

该算法也是非常简单明了，可以说是非常天才的算法了，值得注意的是，此算法的时间复杂度是O(N)，对比LVS则是最差O(N)，而随机算法恒定为O(1)，当Set中的机器超过一定数量（比如说有1万台），那么此算法的时间复杂度可能就无法接受了。

最后附上GBN实现的代码片段：

```
#include <iostream>
#include <vector>

template <typename T>
class RoundRobin {
public:
    struct Element {
        T value_;
        int weight_;
        int currentWeight_;
        Element(T v, int w): value_(v), weight_(w), currentWeight_(0) {}
    };
    void Add(const T& t, int w);
    const T* Next();
private:
    std::vector<Element> elements_;
};

template <typename T>
void RoundRobin<T>::Add(const T& t, int w) {
    elements_.push_back({t, w});
}

template <typename T>
const T* RoundRobin<T>::Next() {
    if (elements_.empty())
        return nullptr;
    if (elements_.size() == 1)
        return &(elements_.front().value_);

    int total = 0;
    Element* p = &elements_[0];
    for (size_t i = 0; i < elements_.size(); ++i) {
        total += elements_[i].weight_;
        elements_[i].currentWeight_ += elements_[i].weight_;
        if (elements_[i].currentWeight_ > p->currentWeight_) {
            p = &elements_[i];
        }
    }
    p->currentWeight_ -= total;
    return &p->value_;
}


int main() {
    RoundRobin<char> rr;
    rr.Add('a', 5);
    rr.Add('b', 1);
    rr.Add('c', 1);

    for (int i = 0; i < 7; ++i) {
        std::cout << *rr.Next() << std::endl;
    }
}
```

该程序输出如下：

```
a
a
b
a
c
a
a
```
