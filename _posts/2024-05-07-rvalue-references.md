---
layout: post
title:  "Rvalue References"
author: Allen Sun
tags: C++ Rvalue Move-Semantics
---

# 1 值的类型(The value categories)

- 左值(lvalues)

    - have an identity and can't be moved from

- 右值(prvalues)

    - don't have an identity and can be moved from

    - 通常包括：字面常量(如：5)、临时变量(如：函数返回值)。

- 过期值(xvalues，eXpiring values)

    - have an identity and can be moved from

    - manually designate name as expiring by using the std::move cast

# 2 引用

引用即变量的别名，底层用指针实现。引用存在的意义是：**传参时避免拷贝**。

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

- Q & A

    1. 左/右值引用自身是左值还是右值？

        A：左值引用是左值。右值引用既可以是左值(被直接声明时)，也可以是右值(作为函数返回值时)。

    2. 右值引用是否可以指向左值？

        A：可以通过std::move()。

        ```cpp
        int a = 1;
        int&& ref1 = a;  // error
        int&& ref2 = std::move(a);  // ok
        ```

        另外，**右值引用可以指向右值的底层行为逻辑**是：首先，将右值提升为左值；然后，定义一个右值引用；最后，通过std::move()使得右值引用指向对应左值。

    3. 左值引用和右值引用有什么区别？

        - 函数传参时使用二者没有**性能**差异，都可以避免拷贝赋值。

        - 右值引用作为函数形参比左值引用更**灵活**，既可以接受右值，也可以接受过期值。如果采用左值引用作为函数形参并且要求左右值实参都接受，那么左值引用必须声明为const。

    4. 为什么需要右值引用？

        - 比左值引用更灵活。

        - 实现移动语义、完美转发。

# 3 移动语义(Move Semantices)

## std::move()

虽然名字叫move，但背地里只做了类型转换，将move的参数转换成右值。

```cpp
template <class T>
typename remove_reference<T>::type&&
move(T&& a){
    return a;
}
```

## 问题分析

```cpp
/* 代码A */
class Array{
public:
    Array(int size): size_(size){
        data_ = new int[size_];
    }
    ~Array(){
        delete []data_;
    }
    Array(const Array& other){
        size_ = other.size_;
        data_ = new int[size_];
        for(int i = 0; i < size_; i++){
            data_[i] = other.data_[i];
        }
    }
    Array& operator=(const Array& other){
        delete []data_;
        size_ = other.size_;
        data_ = new int[size_];
        for(int i = 0; i < size_; i++){
            data_[i] = other.data_[i];
        }
    }
public:
    int* data_;
    int size_;
};
```

- 在**代码A**中，存在如下性能缺陷：

    - 虽然通过左值引用传参避免了一次拷贝，但是在拷贝构造函数、拷贝赋值函数的内部实现中，还是进行了**深拷贝**。

    - 如果**被拷贝者在拷贝后不再被需要**(expiring value)，那么是否可以直接进行**浅拷贝**以提高程序性能？

```cpp
/* 代码B */
class Array{
public:
    // 左值引用——拷贝构造
    Array(const Array& other){
        size_ = other.size_;
        data_ = new int[size_];
        for(int i = 0; i < size_; i++){
            data_[i] = other.data_[i];
        }
    }
    // 左值引用——拷贝构造函数重载
    Array(const Array& other, bool move){
        size_ = other.size_;
        data_ = other.data_;
        other.data_ = nullptr;  // error：无法修改const修饰的参数
    }
};
```

- 在**代码B**中，存在如下问题：

    - const左值引用虽然可以接受右值，但是无法实现浅拷贝。

    - 还需要加个参数move，实现得不优雅。

```cpp
/* 代码C */
class Array{
public:
    // 深拷贝，左值引用，《拷贝构造》
    Array(const Array& other){
        size_ = other.size_;
        data_ = new int[size_];
        for(int i = 0; i < size_; i++){
            data_[i] = other.data_[i];
        }
    }
    // 浅拷贝，右值引用，《移动构造》
    Array(Array&& other){
        size_ = other.size_;
        data_ = other.data_;
        other.data_ = nullptr;  // 防止other析构data_指向的资源
    }
};
```

- 在**代码C**中，存在如下优点：

    - 实现优雅，不需要额外增加参数move。

    - 如果用户传入的是临时对象(右值)，则直接进行浅拷贝以提高程序性能。

## 典例赏析

提醒：不是std::move()提高了性能，而是通过std::move()做类型转换后，调用到**移动**构造/赋值函数而非**拷贝**构造/赋值函数，从而提高了性能。

1. case1：移动构造

    ```cpp
    std::string str = "I've already learned what is Move Semantics.";
    std::vector<std::string> vec;

    vec.push_back(str);  // 调用拷贝构造

    /* 调用移动构造，str失去原有值变成空串 */
    vec.push_back(std::move(str));
    ```

2. case2：移动赋值

    ```cpp
    std::unique_ptr<A> ptr = std::make_unique<A>();

    /* 调用移动赋值，ptr失去原有资源的所有权变成空指针 */
    std::unique_ptr<A> ptr1 = std::move(ptr);

    std::unique_ptr<A> ptr2 = ptr;  // error：unique_ptr不支持拷贝赋值
    ```


# 4 完美转发(Perfect Forwarding)

## std::forward()

并不做转发，做类型转换，比std::move()的功能更强大。主要用于模板编程的参数转发。

- 类型转换逻辑：

    ```cpp
    std::forward<T>(u)
    ```

    - 如果T是左值引用类型，则将u转换为T类型的左值；

    - 否则，将u转换为T类型的右值。

## 典例赏析

```cpp
void func1(int& ref_l){
    ref_l = 1;
}

void func2(int&& ref_r){
    ref_r = 1;
}

void foo(int&& ref_r){
    func1(ref_r);  // ok, 右值引用ref_r是左值
    func1(std::forward<int&>(ref_r));  // ok, std::forward会将ref_r转换为左值

    func2(ref_r); // error：func2需要的参数是右值
    func2(std::move(ref_r));  // ok, std::move将ref_r转换为右值
    func2(std::forward<int&&>(ref_r));  // ok, std::forward将ref_r转换为右值
}
```
