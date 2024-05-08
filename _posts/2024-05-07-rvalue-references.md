---
layout: post
title:  "Rvalue References"
author: Allen Sun
tags: C++ Rvalue Move-Semantics
---

# 1 左值与右值

- 左值

    - 可以在赋值符=左边出现。

    - 有地址，即占据内存。

- 右值

    - 可以在赋值符=右边出现。

    - 不可取地址，通常包括：立即数(如：5, '5')、临时变量(如：函数返回值)。

# 2 左值引用与右值引用

引用即变量的别名，底层用指针实现。

- 左值引用

    - 仅能指向左值。

        ```cpp
        int a = 1;
        int& ref1 = a;  // ok
        int& ref2 = 5;  // error: 左值引用不能指向右值

        /* const左值引用不会改变右值，所以其可以指向右值 */
        const int& ref3 = 5;  // ok
        ```

- 右值引用

    - 仅能指向右值。

        ```cpp
        int&& ref1 = 5;  // ok

        int a = 1;
        int&& ref2 = a;  // error: 右值引用不能指向左值
        ```

- Q&A

    1. 左/右值引用自身是左值还是右值？