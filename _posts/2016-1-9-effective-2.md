---
layout: blog
categories: C++
title: Item 2：尽量以const，enum，inline替换#define
subtitle: Effective C++ 读书笔记
tags: C++ const enum inline define
excerpt: 尽量以const、enum、inline替换#define
---

> Prefer consts, enums, and inlines to #defines

该条款可以解释为**用编译器替换预处理器**。

#常量替换宏

比如下面的代码，

```cpp
#define ASPECT_RATIO 1.653
```

记号名称`ASPECT_RATIO`可能并未被编译器所看到，它可能在预处理期间就被1.653替换，因而并未进入记号表（symbol table）中。
这样不利于后面的调试，因为捕捉到的错误信息可能是1.653而不是`ASPECT_RATIO`。

解决方法是用常量来替换上面的宏：

```cpp
const double AspectRatio = 1.653; //全大写名称常用于宏，这里采用驼峰写法
```

这里的AspectRatio肯定会被编译器看到，并进入记号表中。当采用常量替换宏（#define）时，需考虑以下两种情况，

一是定义常量指针时。若要在头文件中定义的是一个常量的char\*-based字符串，你必须使用const两次：

```cpp
const char* const authorName = "Scott Meyers";
```

这里使用string对象比char\*-based更合适些，所以最好采用如下方式：

```cpp
const std::string authorName("Scott Meyers");
```

二是定义类专属常量。为了将常量的作用域限制在类内，需要让它成为类的一个成员；为了确保该常量只有一份副本，你需要把它定义成static成员：

```cpp
class GamePlayer {
private:
    static const int NumTurns = 5;
    int scores[NumTurns];
    ...
};
```

上面其实是`NumTurns`的声明式（declaration）而非定义式（definition）。
一般来说，C++要求你提供定义式，但类专属常量又是static和整数类型（int、char、bool）时可以特殊处理。
即在不需要取该常量地址的情况下，只提供声明就好。若需要取该常量地址，则需提供如下定义式：

```cpp
const int GamePlayer::NumTurns; //NumTurns的定义式，声明中已设初值，定义中不可以再设初值
```
这个式子需放在源文件中，而不是头文件中。

顺带一提，无法使用#define来创建类专属常量，因为#define并不尊重作用域。
一旦宏被定义，它在后面的编译过程中就一直有效（除非被#undef）。因此也不能提供封装性，即没有所谓的private #define这样的东西。
但const成员变量是可以被封装的（NumTurns就是）。

旧的编译器也许不支持static成员在声明时获得初值，这时你可以在定义式中赋初值：

```cpp
class CostEstimate {
private:
    static const double FudgeFactor; //放在头文件中
    ...
};

const double CostEstimate::FudgeFactor = 1.35; //放在源文件中
```

#enum替换宏

当你在类编译期间需要一个类常量值，比如上面的GamePlayer::scores的数组声明式中（编译器必须在编译期间知道数组的大小），
我们还可以采用`the enum hack`的补偿做法代替static常量。因为枚举类型的数值可以当做ints来使用，于是GamePlayer可定义如下：

```cpp
class GamePlayer {
private:
    enum { NumTurns = 5 };
    int scores[NumTurns];
    ...
};
```

`the enum hack`的以下特性也值得我们认识一番：

1. `the enum hack`在某些方面表现的更像#define而不像const，有时这正式我们想要的。
比如，取一个const 的地址是合法的，但取一个enum或#define是不合法的。
如果我们不希望别人获取该常量的指针或引用，就可以使用enum。
又比如，虽然优秀的编译器不会为整数型const对象设定另外的内存空间（除非你创建了指向该对象的pointer或reference），不够优秀的编译器却未必如此。
但enum和#define绝不会导致不必要的内存分配。
2. `the enum hack`是模板元编程的基础技术。 

#inline替换宏

形似函数的宏虽不会产生函数调用（function call）带来的额外开销，但却很容易被误用：

```cpp
//a和b中的较大值调用f
//不要忘记给宏中的所有实参加上小括号
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b)) 
```

然后这样使用了它，

```cpp
int a = 5, b = 0;
CALL_WITH_MAX(++a, b);      //a累加二次
CALL_WITH_MAX(++a, b+10);   //a累加一次
```

在这里，调用f之前，a的递增次数竟取决于**它和谁进行比较！**

幸运的是我们还可以使用template inline函数来代替宏，这样既不会失去宏带来的效率，又能拥有一般函数的所有可预料行为和类型安全性。

```cpp
template<typename T>                            //由于不知道
inline void callWithMax(const T& a, const T& b) //T是什么，采用
{                                               //pass by reference-to-const
    f(a > b ? a : b);
}
```

这个template产生了一个函数群，每个函数都接受两个同型对象，并以其中较大者调用f。

当然const、enum和inline确实降低了预处理器（特别是#define）的需求，但并非完全消除。
\#include仍然是必需品，#ifdef/#ifndef也继续扮演控制编译的重要角色。

