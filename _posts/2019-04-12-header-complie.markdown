---
layout: post
title:      "将文件间编译依存关系降至最低与预编译头"
subtitle:   ""
date:       2019-04-04 00:00:00
author:     "DEEP"
header-img: "img/home-bg.jpg"
catalog: true
tags:
    -   C/C++
---

## 序言

该话题出自「Effective C++」中的条款 31。

一直以来对这个话题以及预编译头的作用都不是很懂，看完该小节后终于相对懂了一点。

篇幅和代码比较多，就不写在读书笔记里了，单独水一发（

## 概述

### 问题出现

我们知道将头文件和实现文件分离的好处之一是降低编译的依存关系。

对于一个未被修改的文件，若其包含的头文件也未发生任何修改，编译器就不会重新编译该文件。

这意味着如果只有 `xxx.cpp` 的实现细节发生变动时，那么其它文件就不会被重新编译。

但文件间的编译依存关系很容易就会被高度耦合，看下面代码：

```c++
// Person.h
#pragma once

#include <string>
#include "Date.h"
#include "Address.h"

class Person
{
public:
    Date fun(Date);
private:
    std::string name;
    Date date;
    Address address;
};
```

如上面代码，如果 `Date.h` 发生了改变，那么 **直接包含了 `Date.h` 的文件都需要重新编译**。

若 `Date` 和 `Address` 是互相对立的话，显然 `Address.cpp` 不需要重新编译。

但此时 `Person.cpp` 却需要重新编译，因为其头文件包含了 `Date.h`。

同理，这 **间接导致所有使用了 `Person.h` 的文件也需要重新编译**！

为了解决这个问题，首先需要思考：为什么 `Person.h` 需要插入 `#include "Date.h"` 呢？

这是因为该头文件出现了 `Date` 对象的声明，此时编译器需要知道该对象的大小才可完成声明，即需要该对象的定义式。

因此便需要引入包含了 `Date` 类定义的头文件 `Date.h`。

但看下面的代码，其无需引入 `Date.h`，转而使用了  `class Date;` 这样的前置声明式：

```c++
// Person.h
#pragma once

#include <string>

class Date;
class Address;

class Person
{
public:
    Date fun(Date);
private:
    Date* date;
    Address* address;
    std::string name;
};
```

上面代码能通过编译，是因为如果只是将类作为：

- 引用或指针。
- 函数形参。
- 函数返回值。

对于这几种用途则只需要 `class Date;` 这样的类前置声明式即可使编译器编译。

因为对于这几种用途，编译器无需使用它的定义。

且需要获取对象引用或对象指针的大小时，编译器会把它们当做指针，而所有类型的指针的大小是固定且一致的。

但通过类的指针或引用访问对象的属性时，就需要该类的定义了：

```c++
// Person.cpp
#include "Person.h"
// #include "Date.h" // 引入该头文件才能找到 Date 的定义

Date Person::fun(Date d)
{
    d.fun(); // 错误！未找到 Date 的定义
}
```

另外需注意，不要对标准程序库组件进行前置声明，正确的标准库组件的前置声明一般比较复杂。

如 `string` 为一个 `typedef`，其定义为 `basic_string<char>`。

且标准头文件一般不太可能成为编译的瓶顶，在支持使用预编译头的建置环境中更是不可能。

另外：

前置声明式是一个非常好用的特性，但面对一些拥有几百个函数声明的头文件时。

需要在头文件中手写每个地方对于用到的类前置声明式，这无疑是十分糟糕的事情。

为了方便，可以将所有声明式都定义在一个 `fwd.h` 中，然后每个头文件都 `#include "fwd.h"`。

这样就无需为每个头文件手工前置声明。

### 设计策略

总结上文，我们需要解决 **改变一个头文件后，直接或间接包含该头文件的文件都需要重新编译** 的问题。

而我们可以遵守下面策略：

- 可以使用对象引用或对象指针时，就不要使用直接对象。
- 头文件中尽量以类前置声明式代替类的定义式。（即头文件尽量不要引入类对应的头文件）
- 不要对标准程序库的组件进行前置声明，可直接引入头文件，更好的方法是使用预编译头。（见下文）
- 将前置声明式写入一个 `fwd.h` 中，然后每个头文件都引入 `fwd.h`。
- cpp 文件中所需的头文件为其对应的头文件，以及定义时所需的组件对应的头文件。

### 预编译头

但一个文件经过更改后，下次编译就会将其重新编译一次，而文件中包含的所有头文件的相关内容都需要重新处理一次。

但我们知道，头文件中的一些内容如标准库组件的引入，其不会再发生任何变动。

但仅仅因为其它地方的变动，导致编译器需要重新处理整个头文件。

对于该场景，我们可以将这些不会再经常变动的代码，写入项目中的预编译头文件中。

这样编译器会生成一个 `xxx.pch` 文件，这是编译这些不常变动的代码后的产物，其可以供给其它文件所用。

使用预编译头可以大幅度提高项目的编译速度。

一般预编译头只引入标准库组件即可，因为标准库的头文件是不可能经常变动的。

需注意：

其它头文件不需要也千万不要引入预编译头。

一方面，本机的开发环境进行了预编译头的设置后，项目内部已经知道哪个是预编译头文件，这时候重复引入是无意义的。

另一方面，若头文件内引入了本机上的预编译头文件，并且其他人引入了本机环境开发的一个头文件后。

此时其它人的开发环境不会知道该头文件引入的文件是一个预编译头。

于是其它人的开发环境会把它当做普通的头文件，开始处理它，那可太糟糕了。

### 问题解决

下面根据上面总结的策略重新设计上面的例子：

```c++
// pch.h
#ifndef PCH_H
#define PCH_H

#include <iostream>
#include <string>
#endif //PCH_H

// -------------------------------------
// -------------------------------------

// pch.cpp
#include "pch.h"
```

```c++
// fwd.h
#pragma once

class Address;
class Date;
class Person;
```

```c++
// Date.h
#pragma once
#include "fwd.h"

class Date
{
public:
    std::string fun();
};

// -------------------------------------
// -------------------------------------

// Date.cpp
#include "pch.h"
#include "Date.h"

std::string Date::fun()
{
    return "fun";
}
```

```c++
// Person.h
#pragma once
#include "fwd.h"

class Person
{
public:
    Date fun(Date);
private:
    std::string name;
    Date* date;
    Address* address;
};

// -------------------------------------
// -------------------------------------

// Person.cpp
#include "pch.h"
#include "Date.h"
#include "Person.h"

Date Person::fun(Date d)
{
    return d;
}
```

现在即使 `Date.h` 发生改变，需要重新编译的文件也只有直接引用了 `Date.h` 的文件，大大降低文件间的编译依存关系。

## 设计

上面的设计只是为了突出总结得出的策略而已，下文总结书中更加精妙的设计手段。

这些设计方式可以解除接口和实现之间的耦合关系，从而进一步降低文件间的编译依存性：

（为行文方便，部分文件的代码如预编译头文件，不再在下文给出）

### Handle classes

该设计的思想是“将对象实现细节隐藏于指针背后”：

```c++
// Person.h
#pragma once
#include "fwd.h"

class Person
{
public:
    Person();
    Person(const std::string& name, const Date& birthday,
		const Address& address);
    std::string getName() const;
    std::string getBirthday() const;
    std::string getAddress() const;
private:
    std::shared_ptr<PersonImpl> pImpl;
};

// -------------------------------------
// -------------------------------------

// Person.cpp
#include "pch.h"
#include "Person.h"
#include "PersonImpl.h"
using namespace std;

Person::Person() : pImpl(new PersonImpl())
{

}
Person::Person(const string& name, const Date& birthday,
	const Address& address)
    : pImpl(new PersonImpl(name, birthday, address))
{

}
string Person::getName() const
{
    return pImpl->getName();
}
string Person::getBirthday() const
{   
    return pImpl->getBirthday();
}
string Person::getAddress() const
{
    return pImpl->getAddress();
}
```

```c++
// PersonImpl.h
#pragma once
#include "fwd.h"
#include "Address.h"
#include "Date.h"

class PersonImpl
{
public:
    PersonImpl();
    PersonImpl(const std::string& name, const Date& birthday,
		const Address& address);
    std::string getName() const;
    std::string getBirthday() const;
    std::string getAddress() const;
private:
    std::string name;
    Date birthday;
    Address address;
};

// -------------------------------------
// -------------------------------------

// PersonImpl.cpp
#include "pch.h"
#include "PersonImpl.h"
using namespace std;

PersonImpl::PersonImpl()
{

}
PersonImpl::PersonImpl(const std::string& name, const Date& birthday,
	const Address& address)
    : name(name), birthday(birthday), address(address)
{

}
string PersonImpl::getName() const
{
    return name;
}
string PersonImpl::getBirthday() const
{
    return birthday.toString();
}
string PersonImpl::getAddress() const
{
    return address.toString();
}

```

该设计要求实现类里面的方法与接口类的方法是一致的。

以后客户端代码使用的将是 `Person`，而不会使用到 `PersonImp`。

`PersonImp` 的实现只面向于软件开发者而不是使用者，从而起到了隔离和隐藏接口实现的作用。

### Interface classes

这种设计模拟 Java 的接口，但与 Java 不同，其可以在接口内定义成员变量和成员函数，具有巨大的弹性：

```c++
// Person.h
#pragma once
#include "fwd.h"

class Person
{
public:
    static std::shared_ptr<Person> create();
    static std::shared_ptr<Person> create(const std::string name,
        const Date& birthday, const Address& addr);
    virtual std::string getName() const = 0;
    virtual std::string getBirthday() const = 0;
    virtual std::string getAddress() const = 0;
    virtual ~Person() {}
};

// -------------------------------------
// -------------------------------------

// Person.cpp
#include "pch.h"
#include "Person.h"
#include "RealPerson.h"
using namespace std;

shared_ptr<Person> Person::create()
{
    return shared_ptr<Person>(new RealPerson());
}
shared_ptr<Person> Person::create(const std::string name,
    const Date& birthday, const Address& addr)
{
    return shared_ptr<Person>(new RealPerson(name, birthday, addr));
}
```

```c++
#pragma once
#include "fwd.h"
#include "Address.h"
#include "Date.h"
#include "Person.h"

class RealPerson: public Person
{
public:
    RealPerson();
    RealPerson(const std::string& name, const Date& birthday,
                const Address& address);
    std::string getName() const;
    std::string getBirthday() const;
    std::string getAddress() const;
    virtual ~RealPerson() {}
private:
    std::string name;
    Date birthday;
    Address address;
};

// -------------------------------------
// -------------------------------------

#include "pch.h"
#include "RealPerson.h"
using namespace std;

RealPerson::RealPerson()
{

}
RealPerson::RealPerson(const std::string& name, const Date& birthday,
                        const Address& address)
    : name(name), birthday(birthday), address(address)
{

}
string RealPerson::getName() const
{
    return name;
}
string RealPerson::getBirthday() const
{
    return birthday.toString();
}
string RealPerson::getAddress() const
{
    return address.toString();
}
```

客户端这样使用它：

```c++
#include "pch.h"
#include "Person.h"
using namespace std;

int main()
{
    std::shared_ptr<Person> pp(Person::create());
    cout << pp->getName() << " " << pp->getBirthday() << " " << pp->getAddress() << endl;
}
```

在该设计中，父类接口只包含虚方法和静态的 `create` 函数声明。

子类将虚方法实现，并实现 `create` 接口利用多态特性。

这样客户不能实例化父类接口对象，但可以通过父类的指针或引用访问子类实现的方法。

同时由于父类不含任何成员变量，所以即使子类更改，父类文件也不会重新编译。

### 成本

- Handle classes 的成本主要在于每次调用接口成员函数都需要通过指针取得实现类对象的数据。

    此外，实现类指针需要初始化，蒙受了动态分配内存的额外开销。

- Inteface classes 的成本主要在于虚函数调用的成本，其派生的对象都含一个 vptr。

    这个指针可能会增加存放对象所需的内存数量。

这些额外成本的付出，使得实现代码变化时对客户造成的冲击最小化，这一般是绝对优越的。

另外：

翻译书中有一句话：

> 不论 Handle classes 和 Interface classes，一旦脱离 inline 函数都无法有太大作为。
>
> 条款 30 解释过为什么函数本体为了被 inlined 必须置于头文件内，
>
> 但 Handle classes 和 Interfaces classes 正式特别被设计用来隐藏实现细节和函数本体。

首先这段话前后矛盾，前面还说需要 `inline`，后面就“但……”转折了。

然后我们假设对这几种设计使用 `inline` ，如果在实现类使用 `inline` 函数。

那么这就意味着使用了实现类接口的文件都需要重新编译，这样不就违背这种设计的本意了吗？

其实应该是翻译有误了，下面看看英文版的原句：

> Finally, neither Handle classes nor Interface classes can get much use
> out of inline functions. Item 30 explains why function bodies must
> typically be in header files in order to be inlined, but Handle and
> Interface classes are specifically designed to hide implementation
> details like function bodies.

get use out of 是“用尽”的意思，这句话应该翻译成：

> 不论 Handle classes 和 Interface classes，都无法发挥 inline 函数的作用。

而不是将 out of 理解为“脱离”的意思。
