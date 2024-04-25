---
layout: post
title:  "Smart Pointers"
author: Allen Sun
tags: C++ Memory-Management Pointers
---

# 1 背景

直接使用C++指针可能会引发下列问题：

1. 内存泄漏(Memory Leak)：不停地手动分配内存，但是从来不回收，系统会因内存耗尽而崩溃。

2. 悬浮指针(Dangling Pointer)：指针指向的内存被回收后，指针的指向仍没有改变。

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
    p2 = p1;  // 引用计数增加
}
```

## 引用计数

- 首先，区分两个概念：

    1. 指针对象：共享指针本身的内存，分配在栈区。

    2. 数据对象：一个或多个共享指针所共同指向的数据，由于该数据通过指针动态分配，所以在堆区。只有指向动态分配的对象的指针才能交给shared_ptr对象托管，将指向普通局部变量、全局变量的指针交给shared_ptr托管，编译时不会有问题，但程序运行时会出错，因为不能析构一个没有指向堆区的指针。

- 引用计数器

    1. 内存位置：因为不同的指针对象需要共享引用计数器(控制块)，所以需要分配在**堆区**；对于每个指针对象，在自己的栈区维护一个指向控制块的指针即可。
    ```cpp
    /* 控制块结构 */
    template <typename T>
    struct ControlBlock{
        T* ptr_;  // 指向数据对象的指针
        shared_count;  // 强引用计数
        weak_count;  // 弱引用计数
    };
    ```

    2. 引用计数包括强引用计数和弱引用计数：

        - 强引用计数：shared_ptr数目。最后一个强引用释放时，数据**对象**会被销毁(destroyed, possibly deallocated)。

        - 弱引用计数：weak_ptr数目。最后一个弱引用释放时，数据**内存**与控制块会被彻底释放(deallocated)。

    3. 控制块是由多个指针对象共享的临界资源，写控制块时要注意的问题如下：

        - 增加引用计数：atomic::fetch_add + memory_order_relaxed，对操作内存的次序不需要有严格的要求。

        - 减少引用计数：需要强内存序，以保证安全地析构。

## 使用make_shared构造指针对象

- 构造指针对象的两种方式
```cpp
/* 方式一 */
auto* ptr = new DataObject(/* args */);  // 为数据对象分配内存
std::shared_ptr<DataObject> shptr{ptr};  // 为指针对象及控制块分配内存

/* 方式二 */
auto shptr = std::make_shared<DataObject>(/* args */);
```

- 使用make_shared有什么好处？

    - 减少内存分配次数：一次性分配控制块内存和数据对象内存(两块内存连续)，满足RAII(Resource Acquisition Is Initialization)原则。

    - 提高cache命中率：控制块内存和数据对象内存连续，优化了程序的空间局部性。

    - 异常安全(Exception Safety)：make_shared保证在连续多次make_shared情况下按序执行，以及异常安全。

- 使用make_shared有什么问题？

    - 当最后一个weak_ptr的生命期结束时，make_shared申请的内存块(包括控制块、数据对象)才会彻底销毁(deallocation)。

    - 如果内存块很大，并且weak_ptr的生命期很长，可能会导致不合理的资源占用。

# 4 weak_ptr

主要用于解决shared_ptr在引用计数过程中可能出现的“死锁”问题。


- “死锁”场景：两个shared_ptr相互引用，这两个shared_ptr的引用计数永远不会减为0，对应的资源永远不会释放。
```cpp
class B;
class A
{
public:
    std::shared_ptr<B> pb_;
};
class B
{
public:
    std::shared_ptr<A> pa_;
};
```

- 破除“死锁”：将相互引用的shared_ptr中的一个置为weak_ptr。由前述可知，weak_ptr不会增加强引用计数，所以不会影响资源的释放。
```cpp
class B;
class A
{
public:
    std::weak_ptr<B> pb_weak;
};
class B
{
public:
    std::shared_ptr<A> pa_;
};
```