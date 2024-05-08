---
layout: post
title:  "Rvalue References"
author: Allen Sun
tags: C++ Rvalue Move-Semantics
---

# 1 左值与右值

1. 左值

    - 可以在赋值符=左边出现。

    - 有地址，即占据内存。

2. 右值

    - 可以在赋值符=右边出现。

    - 没有地址。

    - 通常包括：立即数(5, '5')、临时变量(函数返回值)

