---
layout: post
title:      "浅谈 C++ 的常量和引用"
subtitle:   ""
date:       2019-03-12 00:00:00
author:     "DEEP"
header-img: "img/home-bg.jpg"
catalog: true
tags:
    -   C/C++
---

## 序言

大一学这块知识的时候一直不是很清晰，莫名其妙地混过去了。

今日重温了一下 primer，试着尽量简要地整理一下 ~~（结果整理完后发现还是跟书上一样）~~。

有些地方混入了个人的片面理解，如有错误的话希望能指出～

## 概念

### 引用

**引用** 为对象起了另外一个名字，定义引用时，程序把引用和对象的初始值绑定在一起。

一旦初始化完成，引用将和它的初始值对象一直绑定在一起，所以：

- 引用无法重新绑定到另一个对象，因此引用必须初始化。
- 因为引用不是对象，所以不能定义引用的引用。
- 引用只可引用对象，不可引用常量。
- 除 **初始化常量引用** 和 **继承类间关系** 两种情况外，其它情况引用类型和对象类型需严格匹配。

整理一些特殊的代码：

```c++
const int const_int = 1024;
int &normal_int_r1 = const_int; // 错误！普通引用不可引用常量对象
const int &const_int_r2 = 1024; // 正确，常量引用可“引用常量”，下文「对常量的引用」会解释
const int &const_int_r3 = 3.14; // 正确，情况1，常量引用允许隐式转型
```

### 常量

**常量** 是一旦创建后其值不能再改变的对象，所以：

- 常量必须初始化，初始值可为任意复杂的表达式。

> 附：若希望一个 `const` 变量只在一个文件定义，其它多个文件声明并使用它。
>
> 解决办法是：对于该 `const` 对象不管声明还是定义都添加 `extern` 关键字：
>
> ```c++
> // file_1.cc 定义并初始化了一个常量
> extern const int bufSize = fcn();
> // file_1.h 头文件
> extern const int bufSize;
> ```

## 常量、引用和对象间的组合

### 对常量的引用

> 对常量的引用可简称为常量引用。
>
> 但记得其实不是指引用本身是「常量」，因为引用不是一个对象。

常量引用是一个特殊的组合。

常量引用对引用可参与的操作做出了限定，但对其引用的对象是否常量未作限定。（即下文的底层 `const`）

下面整理常量引用的初始化代码：

```c++
int nomral_int = 512;
const int const_int = 1024;
double normal_double = 3.14;
const int &const_int_r1 = const_int;     // 正确，较为标准的一个例子
const int &const_int_r2 = normal_int;    // 正确，常量引用不限定对象必须为常量
                                         // 原因见下文的底层 const
const int &const_int_r3 = 1024;          // 正确，但不是说引用不可引用常量吗？
const int &const_int_r4 = normal_double; // 正确，常量引用允许隐式转型，但为什么？
```

为什么常量引用可以引用常量？为什么常量引用允许隐式转型？这也就是常量引用的特殊性。

其原理是相似的，编译器会把代码变成下面形式，使常量引用绑定了一个 **临时量** 对象：

```c++
const int temp_1 = 1024;
const int &const_int_r3 = temp_1; // 引用常量
const int temp_2 = normal_double;
const int &const_int_r4 = temp_2; // 隐式转型
```

> 因为常量引用可以绑定大部分类型的值，所以函数形参一般为常量引用类型。
>
> 但注意！正如上文产生临时值的原理，导致有些情况下对常量引用取地址是无意义的！
>
> 所以若函数中需要对引用形参取地址时，请忽将函数形参声明为常量引用类型。

### 定义或拷贝的合法性

下面讨论顶层 `const` 和底层 `const` 的内容。

利用从右向左阅读变量的定义的方法，有利于区分顶层 `const` 和底层 `const`：

```c++
int normal_int = 1024;
int *const top_const_p1 = &normal_int;           // 顶层const，可模糊理解为(int (*const))
const int const_int = 0;                         // 顶层const，可模糊理解为((const) int)
const int &const_ref = const_int;                // 底层const
                                                 // 用于声明引用的 const 都是底层const
const int *bottom_const_p2 = &const_int;         // 底层const，可模糊理解为(const int (*))
const int *const mix_const_p3 = bottom_const_p2; // 既有顶层const 也有底层const
                                                 // 可模糊理解为(const int (*const))
```

- 顶层 `const` 限制不可改变自身的值。（引用本身就不可改变绑定对象，所以没有顶层 `const` 的概念）

    因此只含顶层 `const` 的指针可以改变其指向的对象内部的值。

- 底层  `const` 限制其不能改变所指对象的值，或调用对象的非  `const` 函数。

    在对象拷贝时，限制双方的底层  `const` 资格必须相同。

    或拷出的对象类型可转换为拷入的对象类型。（即非常量可转为常量）

看几个定义或拷贝例子：

```c++
bottom_const_p2 = mix_const_p3;  // 正确，p2、p3 均包含底层const 的定义
                                 // p3 的顶层const 不影响拷贝
bottom_const_p2 = normal_int;    // 正确，int* 可转换为 const int*
const int &const_r = normal_int; // 正确，int 可转换为 const int
int *normal_p = mix_const_p3;    // 错误，p3 包含底层const 的定义
int &normal_r = const_int;       // 错误，const int 不可转换为 int 的引用
```

总结得出规律：

**假设表达式左右两边不考虑 const 时的基本数据类型相同时**：

- 引用/指针初始化时，若右边含顶层  `const`，则左边需含底层 `const`。
- 拷贝时，若右边含底层  `const`，则左边也应该含底层  `const`。
- 其它情况通常合法。

## 特殊类型符号中的常量和引用

原本想自我拓展一下这块滴，然后认真读读写写，发现书本的例子其实已经很全面了……

就当复习吧（

### constexpr

**常量表达式** 指值不会改变且在编译过程就能得到计算结果的表达式。

```c++
const int max_files = 20;       // 是常量表达式
const int limit = max_file + 1; // 是常量表达式
int staff_size = 27;            // 不是常量表达式，它的数据类型是普通的int
const int sz = get_size();      // 不是常量表达式
```

在系统中，很难分辨一个初始值是否常量表达式。

C++11 提供了 `constexpr` 类型，将变量声明为 `constexpr`，则变量一定是常量，且必须用常量表达式初始化：

```c++
constexpr int mf = 20;        // 常量表达式
constexpr int limit = mf + 1; // 常量表达式
constexpr int sz = size();    // 只有 size 是一个 constexpr 函数时，声明才正确
```

在声明变量时，`constexpr` 包含顶层 `const` 的意义：

```c++
const int *p = nullptr;        // p 是一个指向整形常量的指针
constexpr int q = nullptr;     // q 是一个指向常量的常量指针
const int num = 42;
constexpr const int *z = &num; // z 是一个指向整形常量的常量指针
```

### typedef

类型别名实际是声明了一种新的数据类型，而非简单的类型别名替换，看下面代码：

```c++
typedef char *pstring;
const pstring cstr = 0; // cstr 是一个常量指针，const 是一个顶层const
const char *cstr = 0;   // cstr 是一个指向常量的指针，可见直接代换别名，实际意义并不一样
```

### auto

`auto` 要求同一句语句定义的变量类型一致，`auto` 的推演结果类型取决于第一个变量的初始值。

对于 `const`，auto 会忽略顶层 `const` 而保留底层 `const`。

```c++
int normal_int = 1;
const int const_int = normal_int, &const_ref = const_int;
auto b = const_int;                     // b 是一个整数，因为顶层const 特性被忽略掉了
auto c = const_ref;                     // c 是一个整数，const_ref 是 const_int 的别名
auto d = &normal_int;                   // d 是一个整型指针
auto e = &const_int;                    // e 是一个指向整数常量的指针
                                        // 因为底层const 特性被保留

const auto &f = ci;                     // f 是一个整型常量，若需顶层const，则要显式写出
auto &g = ci;                           // g 是一个整型常量引用，auto 推演成了 const int
auto &h = 42;                           // 错误，推演出常量引用
auto auto &j = 42;                      // 正确，auto 推演为 int

auto k = const_int, &l = normal_int;    // 正确，k 是整数，l 是整型引用
auto &m = const_int, *p = &const_int;   // 正确，m 是整型常量引用，p 是指向整数常量的指针
                                        // auto 推演为 const int
auto &n = normal_int, *p2 = &const_int; // 错误，auto 推演为 int，p2 的定义不合法
```

### decltype

`decltype` 选择并返回操作数的数据类型，此过程中，编译器分析表达式并得到它的类型，而不实际计算表达式的值。

还有一些比较特殊的情况，见下面例子：

```c++
const int const_int = 0, &const_ref = const_int;
decltype(const_int) x = 0;             // 类型是 const int
decltype(const_ref) y = x;             // 类型是 const int&
decltype(const_ref) z;                 // 错误，引用必须初始化

int normal_int = 1, *p = &normal_int, &normal_ref = normal_int;
decltype(normal_ref + 0) b;            // b 是整型，因为引用参与运算，表达式结果为右值
decltype(*p) c = normal_int;           // c 是整型引用，因为解引用操作返回左值引用

decltype((normal_int)) d = normal_int; // d 是整型引用
                                       // 因为给变量加上一层括号，编译器当成是表达式
                                       // (变量) 表达式返回结果为左值
```
