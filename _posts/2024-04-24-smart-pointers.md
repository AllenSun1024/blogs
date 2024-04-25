---
layout: post
title:  "Smart Pointers"
author: Allen Sun
tags: C++ Memory-Management Pointers
---

# 1 背景

直接使用C++指针可能会引发下列问题：

1. 内存泄漏(Memory Leak)：不停地手动分配内存，但是从来不回收，系统会因大量内存被消耗而崩溃。

2. 悬浮指针(Dangling Pointer)：指针指向的内存被回收后，指针的指向仍没有改变，稍不注意就会导致非法访问。

3. 野指针(Wild Pointer)：指针从未被初始化，从未指向合法的数据。

4. 数据不一致(Data Inconsistency)：内存中的数据没有按一致性次序更新。

为了解决上述问题，C++提出了“用对象的方式管理内存”，模型如下：
```cpp
template <typename T>
class SmartPointer{
    T* raw_ptr_;
public:
    explicit SmartPointer(T* p = nullptr) : raw_ptr_(p) {}
    ~SmartPointer(){
        /**
         * 智能指针的生存期结束时：
         * 自动调用析构函数释放其所指向的数据对象
         */
        delete raw_ptr_;
    }
    T& operator*(){
        return *raw_ptr_;
    }
};
```

# 2 unique_ptr

同一时刻，只能有一个指针指向数据对象。
```cpp
#include <memory>

void foo(){
    std::unique_ptr<int> p1(new int(38));  // p1指向38
    std::unique_ptr<int> p2;
    p2 = std::move(p1);  // p2指向38，并且，p1指向空
}
```

# 3 shared_ptr

同一时刻，可以有多个指针指向数据对象，基于**引用计数**实现。
```cpp
#include <memory>

void foo(){
    std::shared_ptr<int> p1(new int(38));
    std::shared_ptr<int> p2;
    p2 = p1;
}
```

## 引用计数

- 首先，区分两个概念：

    1. 指针对象：共享指针本身的内存，分配在栈区

    2. 数据对象：一个或多个共享指针所共同指向的数据，由于该数据通过指针动态分配，所以在堆区

- 引用计数器

![](https://github.com/AllenSun1024/blogs/blob/main/docs/assets/smart_pointers_1.png)

    1. 内存位置：因为不同的指针对象需要共享引用计数器(控制块)，所以需要分配在**堆区**；对于每个指针对象，在自己的栈区维护一个指向控制块的指针即可。

    2. 引用计数分强计数和弱计数两类：

        - 强计数：

        - 弱计数：

# 4 weak_ptr


