+++
title = "洗牌算法"
date = "2016-03-18"
description = "洗牌算法是一个比较形象的术语，本质上是让一个数组内的元素随机排列"
tags = ["algorithms"]
categories = ["algorithms"]
+++

洗牌算法是一个比较形象的术语，本质上是让一个数组内的元素随机排列。刚在[segmentfault](https://segmentfault.com/)看到这样一个问题[10万个数字无序排列](https://segmentfault.com/q/1010000004628428)，也就是洗牌了。

<!--more-->

### 实现

维基百科上的[Fisher–Yates shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle)对洗牌做了比较详细的介绍，上面伪代码实现如下

```
-- To shuffle an array a of n elements (indices 0..n-1):
for i from n−1 downto 1 do
     j ← random integer such that 0 ≤ j ≤ i
     exchange a[j] and a[i]
```

<!--more-->

### 原理

- 该方法选中数组的最后一个元素：


- 接下来确定挑选随机元素的范围，从数组的第一个元素到上一步选中的元素都属于这一范围


- 确定范围后，从中随机挑选一个数


- 然后交换最后一个元素和随机选中的元素的值


- 上面的交换完成后，相当于我们完成了对数组最后一个元素的随机处理。接下来选中数组内倒数第二的元素


- 之所以从后往前处理，是因为这样便于确定随机选择的范围。


- 剩下的就是一些重复性的工作


### 代码

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import random
N = 100000
data = range(N)
for i in range(len(data)-1, 0, -1):
    t = int(random.random()*i)
    data[t], data[i] = data[i], data[t]
print(data)
```

