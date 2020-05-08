---
layout: post
title:      "浅谈 C++ 的右值引用"
subtitle:   ""
date:       2019-04-04 00:00:00
author:     "DEEP"
header-img: "img/home-bg.jpg"
catalog: true
tags:
    -   C/C++
---

## 概述

右值引用是 C++11 加入的新特性。

与常规引用（左值引用）不同，右值引用不可绑定到左值上，其只可绑定到右值上。

（左值是可取地址的变量，右值是指非左值）

```c++
int i = 42;
int &&rr = i; // 错误，右值引用不可绑定到左值
int &r2 = i * 42; // 错误，右边为右值
int &&rr2 = i * 42; // 正确
const int &r3 = i * 42; // 正确，常量引用可绑定到右值上
```

返回非引用类型的函数、后置递增/减运算符、算术、关系、位运算均生成右值。

右值引用也即只能绑定到临时对象上，因此其有特性：

- 所引用的对象将要被销毁。
- 该对象没有其它用户。

右值引用初始化后为一个右值引用类型变量，而变量都是左值，所以右值引用不可绑定到右值引用类型变量：

```c++
int &&rr1 = 42;
int &&rr2 = rr1; // 错误，rr1 是左值
```

## 转换左值为右值引用

我们可以使用 `std::move` 显式地将一个左值转换为右值引用类型：

```c++
int i = 42;
int &r1 = i;
int &&rr3 = std::move(r1);
// 注意不应该对 std::move 使用 using 声明
// 原因在于 std::move 是一个接收右值引用参数的模板函数
// 若用户定义了一个接收普通参数的 move 函数，则使用 move 时编译器会优先使用用户的版本
// （注：没错，std::move 是接收右值引用参数，其利用了引用折叠将右值引用转为左值）
```

此外， `std::move` 也可以接收一个右值参数。

调用 `move` 则意味着承诺除了对 `rr1` 赋值或销毁外，之后不再会使用它，因为其含义是转交资源。

`std::move` 的实现如下：

```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& t)
{
    return static_cast<typename remove_reference<T>::type&&>(t);
}
```

这里有很多语法特性，下面逐一介绍：

- 在模板返回值中使用 `typename` 关键字可指定返回值为类的类型成员。

- `remove_reference` 是标准库的类型转换模板，其有一个名为 `type` 的类型成员，表示被引用的类型。

- 引用折叠：在间接创建一个引用的引用时会有以下规则：

    `X& &`、`X& &&`、`X&& &` => `X&`
    `X&& &&` => `X&&`

    因此在该函数中，其可接受左值或右值参数，并保持参数原来的类型。

    （注意引用折叠只能在间接创建一个引用的引用时才能工作，因此引用折叠只能应用于类型别名或模板参数中）

下面根据几个例子来分析 `std::move` 的工作原理：

```c++
string s1("hi!"), s2;
s2 = std::move(string("bye!"));
// 1. 推断出 T 为 string 类型
// 2. 函数返回类型 remove_reference<T>::type&& 为 string&&
// 3. 函数参数 t 为 string&& 类型
// 4. static_cast 什么都不做

s2 = std::move(s1);
// 1. 推断出T 为string& 类型
// 2. 函数返回类型 remove_reference<T>::type&& 为 string&&
// 3. 函数参数 t 折叠为 string& 类型
// 4. 将一个左值显式地 static_cast 到一个右值引用是允许的
```

## 右值引用的应用

### 移动构造和移动赋值

看看一个简单的动态数组类的实现：

```c++
class Array
{
public:
    const static unsigned int defaultSize = 10000;
    Array() : size(defaultSize), elements(new int[defaultSize])
    {
        std::cout << "default constructor called" << std::endl;
    }
    Array(const Array& arr) : size(arr.size), elements(new int[arr.size])
    {
        std::cout << "copy constructor called" << std::endl;
        memcpy(elements, arr.elements, arr.size * sizeof(int));
    }
    Array& operator=(const Array& rhs)
    {
        std::cout << "equal operator called" << std::endl;
        if (this != &rhs) {
            delete[] elements;
            size = rhs.size;
            elements = new int[rhs.size];
            memcpy(elements, rhs.elements, rhs.size * sizeof(int));
        }
        return *this;
    }
    ~Array()
    {
        std::cout << "destructor called" << std::endl;
        delete[] elements;
    }
    void doSomething()
    {
		// do some thing
    }
private:
    unsigned int size;
    int *elements;
};

int main()
{
    Array arr1;
    arr1.doSomething();
    Array arr2(arr1);
    return 0;
}
```

在上面代码中，`arr1` 在逻辑上只是一个“临时”变量，处理完的 `arr1` 将会被 `arr2` 拷贝。

此时调用的是拷贝赋值，导致 `arr2` 花了额外的时间和空间进行拷贝工作。

但其实逻辑上 `arr1` 再也不会被使用到，其初始化时花费的时间和空间被“浪费”了。

对于此类情况，可为 `Array` 类定义移动构造函数和移动赋值函数，并利用 `std::move` 进行赋值：

```c++
class Array
{
public:
    ... // 其余代码同上
    Array(Array &&arr) noexcept : size(arr.size), elements(arr.elements)
    {
        std::cout << "move constructor called" << std::endl;
        arr.size = 0;
        arr.elements = nullptr;
    }
    Array& operator=(Array &&rhs) noexcept
    {
        std::cout << "move equal operator called" << std::endl;
        if (this != &rhs) {
            delete[] elements;
            size = rhs.size;
            elements = rhs.elements;
            rhs.size = 0;
            rhs.elements = nullptr;
        }
        return *this;
    }
};

int main()
{
    Array arr1;
    arr1.doSomething();
    Array arr2(std::move(arr1));
    return 0;
}
```

在移动构造函数和移动赋值函数中，其不分配任何新内存，只是接管右值的内存。

因此，其一般不会抛出任何异常，因此可以指明 `noexcept`。

通过移动函数，可以重复利用无用对象的内存，从而大幅度提升性能。

但移后源对象具有不确定的状态，对其进行操作时十分危险的行为。

因此使用在 `std::move` 后一定要保证不再使用被移动的对象。

接下来，看看如何在拷贝并交换赋值运算符中利用移动操作：

```c++
class Array
{
    friend void swap(Array&, Array&);
public:
    ... // 其余代码同上
    Array(Array &&arr) noexcept : size(arr.size), elements(arr.elements)
    {
        std::cout << "move constructor called" << std::endl;
        arr.size = 0;
        arr.elements = nullptr;
    }
    Array& operator=(Array rhs)
    {
        std::cout << "copy and swap equal operator called" << std::endl;
        swap(*this, rhs);
        return *this;
    }
};
inline void swap(Array &lhs, Array&rhs)
{
    using std::swap; // 加上此行后
                     // 如果类成员存在类型特定的 swap 版本，则匹配程度会优于 std 版本
    swap(lhs.size, rhs.size);
    swap(lhs.elements, rhs.elements);
}

int main() {
    Array arr1, arr2, arr3;
    arr1.doSomething();
    arr2 = arr1;
    arr3 = std::move(arr1);
    return 0;
}
```

其实该拷贝并交换赋值运算符的定义与常规版本并无区别，但移动拷贝构造函数间接为其提供了移动功能。

当调用运算符时，若传入的实参是一个左值，此时形参初始化时会调用常规的拷贝构造函数。

因此形参会进行申请新内存的拷贝。

但若传入实参初始化是一个右值，此时形参初始化会调用移动拷贝构造函数。

因此形参会接管实参的内存，然后在运算符函数内部通过 `swap` 函数将该内存又转交给 `*this` 。

在小节的最后，整理一下编译器是否进行对移动函数的合成或删除的原则：

- 与拷贝操作不同，移动操作永远不会被隐式定义为删除。

    （在定义移动操作但未定义拷贝操作的类中，拷贝操作被隐式定义为删除）

- 若类中没有定义任何拷贝控制成员，且所有成员均可移动。

    编译器才会为它合成移动操作。

- 在显示要求编译器生成 `= default` 的时候：

    - 若有类成员定义拷贝构造函数而未定义移动构造函数，

        或有类成员未定义拷贝构造函数但编译器不能为它合成移动构造函数，

        则移动构造函数被定义为删除。（赋值运算符的情况类似）

    - 若有类成员的移动构造函数被定义为删除或不可访问，

        则类的移动构造函数被定义为删除。（赋值运算符的情况类似）

    - 类似拷贝构造函数，若类的析构函数被定义为删除或不可访问，

        则移动构造函数也被定义为删除。

    - 类似拷贝赋值运算符，如果有类成员是 `const` 或是引用，

        则类的移动赋值运算符被定义为删除。

### 模板函数参数转发

现编写一个函数，其接受一个二元可调用表达式和两个参数，其返回将两个参数作为可调用表达式的实参调用后的结果值。

下面是一个比较简单的实现方式，但实际使用会出现问题：

```c++
template <typename F, typename T1, typename T2>
void call_fun(F f, T1 t1, T2 t2)
{
    f(t1, t2);
}
void fun1(int &v1, int v2)
{
    // do some thing to v1
    ++v1;
}
int i = 0;
fun1(i, 42); // i 被改变
call_fun(fun1, i, 42); // i 没有被改变
```

在上例可看到，在模板函数 `call_fun` 初始化 `t1` 后得到的是 `i` 的一个拷贝副本。

导致其实际调用 `fun1(t1, t2)` 时，`v1` 变成了是对 `t1` 的引用，从而违背了设计者的本意。

此外，该实现方式也无法保持参数的常量属性，为了能保持参数引用和常量属性，可利用右值引用实现：

```c++
template <typename F, typename T1, typename T2>
void call_fun(F f, T1 &&t1, T2 &&t2)
{
    f(t1, t2);
}
call_fun(fun1, i, 42);
// 该情况下，T1 被推断为 int&，通过引用折叠，t1 的类型折叠为 int&
// 此外，右值引用的形参可保持对应实参的 const 属性
```

该版本解决了左值引用参数的问题，但其不能用于接受右值引用参数的可调用函数：

```c++
void fun2(int &v1, int &&v2)
{
    // do some thing
}
call_fun(fun2, i, 42); // 错误
```

这里之所以会出错，是因为使用 42 初始化的 `t2` 时，`T2` 被推断为 `int`，因此 `t2` 是一个右值引用。

而在本文第一章提到，右值引用为一个左值，因此不可绑定到 `int&&`。

对于该情况，就需要使用 `std::forward` 保持原来参数的类型信息。

`forward` 必须通过显式模板实参调用，其返回显式实参类型的右值引用，即 `forward<T>` 返回 `T&&`。

下面看该函数的最终实现：

```c++
template <typename F, typename T1, typename T2>
void call_fun(F f, T1 &&t1, T2 &&t2)
{
    f(std::forward(t1), std::forward(t2));
}
call_fun(fun2, i, 42);
// 此时，std::forward(t2) 为 int&& &&，折叠为 int &&
```
