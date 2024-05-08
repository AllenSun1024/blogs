---
layout: post
title:  "Rvalue References"
author: Allen Sun
tags: C++ Rvalue Move-Semantics
---

# 1 什么是左值与右值？

1. 定义

    - 左值(Lvalue)

        - 有一块内存属于自己。

        - 可以通过取地址符`&`获取左值的内存位置。

    - 右值(Rvalue)

        - 不是左值。通常是一个临时变量，比如：函数的返回值。

2. 例子

    - 不可对右值进行赋值操作：

    ```cpp
    int a = 1, b = 2;  // a与b都是左值
    a = a * b;  // ok
    a * b = 3;  // error：a * b是右值，不可以对右值赋值
    ```

    - 不可对右值进行取址操作：

    ```cpp
    int& foo();  // 函数声明，返回值类型为int引用
    int foobar();  // 函数声明

    int main(){
        int j = foo();  // ok
        int* p1 = &foo();  // ok：foo()的返回值是左值
        foo() = 4;  // ok：foo()的返回值是左值

        int k = foobar();  // ok
        /* error： foobar()的返回值是右值(临时变量)
        *  存储foobar()返回值的寄存器在foobar()调用结束后立即回收，
        *  致使无法对foobar()的返回值(根本不会被写入内存)取址
        */
        int* p2 = &foobar();
        foobar() = 5;  // error
    }
    ```

# 2 什么是左值引用与右值引用？

1. 定义

    - 左值引用(Lvalue References)

        - 变量的别名，可以通过引用读写变量的内存。

    - 右值引用(Rvalue References)

        - 和左值引用类似。

        - 不同之处在于，右值引用可以绑定到临时变量(Rvalue)上。

2. 例子

    - 定义左值引用：

    ```cpp
    A a;
    A& a_ref1 = a;  // an lvalue reference
    ```

    - 定义右值引用：

    ```cpp
    // 右值引用可以绑定到左值上
    A a;
    A&& a_ref2 = a;  // an rvalue reference
    ```

    - 右值引用可以绑定到临时变量：

    ```cpp
    A& a_ref3 = foo();  // error: 左值引用不可以绑定到临时变量！
    A&& a_ref4 = foo();  // ok
    ```

# 为什么要有右值引用？

## 移动语义(Move Semantics)

## 完美传播(Perfect Forwarding)

# 右值的底层实现逻辑是什么？