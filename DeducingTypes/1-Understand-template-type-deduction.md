条款1：理解模板类型推导
====================
##Understand template type deduction.

当一个复杂系统的用户忽略这个系统是如何工作的，那就再好不过了，因为“如何”扯了一堆系统的设计细节。从这个方面来度量，C++的模板类型推导是个巨大的成功。成百上万的程序猿给模板函数传递完全类型匹配的参数，尽管有很多的程序猿会更加苛刻的给于这个函数推导的类型的严格描述。

如果上面的描述包括你，那我有好消息也有坏消息。好消息就是模板的类型推导是现代C++的最引人注目的特性：`auto`。如果你喜欢C++98模板的类型推导，那么你会喜欢上C++11的`auto`对应的模板类型推导。坏消息就是模板类型推导的法则是受限于`auto`的上下文的，有时候看起来应用到模板上不是那么直观。因为这个原因，真正的理解模板`auto`的类型推导是很重要的。这条条款囊括了你的所需。

如果你想大致看一段伪码，一段函数模板看起来会是这样：

```cpp
template<typename T>
void f(ParamType param);
```

调用会是这样：

```cpp
f(expr);                    // 用一些表达式来调用f
```

在编译的时候，编译器通过`expr`来进行推导出两个类型：一个是`T`的，另一个是`ParamType`。通常来说这些类型是不同的，因为`ParamType`通常包含一些类型的装饰，比如`const`湖综合引用特性。举个例子，模板通常采用如下声明：

```cpp
template<typename T>
void f(const T& param);		// ParamType 是 const T&
```

如果有这样的调用：

```cpp
int x = 0；

f(x)；						// 使用int调用f
```

`T`被推导成`int`，`ParamType`被推导成`const int&`。

一般会很自然的期望`T`的类型和传递给他的参数的类型一致，也就是说`T`的类型就是`expr`的类型。在上面的例子中，`x`是一个`int`，`T`也就被推导成`int`。但是并不是所有的情况都是如此。`T`的类型不仅和`expr`的类型独立，而且还和`ParamType`的形式独立。下面是三个例子：

* `ParamType`是一个指针或者是一个应用类型，但并不是一个通用的引用类型（通用的引用类型的内容在条款24。此时，你要知道例外情况会出现的，他们的类型并不和左值应用或者右值引用）。

* `ParamType`是一个通用的引用

* `ParamType`既不是指针也不是引用

这样的话，我们就有了三种类型需要检查的类型推导场景。每一种都是基于我们队模板的通用的调用封装：

```cpp
template<typename T>
void f(ParamType param);

f(expr);					// 从expr推导出T和ParamType的类型
```

## 第一种情况：`ParamType`是个非通用的引用或者是一个指针

最简单的情况是当`ParamType`是一个引用类型或者是一个指针，但并非是通用的引用。在这种情况下，类型推导的过程如下：

1. 如果`expr`的类型是个引用，忽略引用的部分。

2. 然后利用`expr`的类型和`ParamType`对比去判断`T`的类型。

举一个例子，如果这个是我们的模板，

```cpp
template<typename T>
void f(T& param);           // param是一个引用类型
```

我们有这样的代码变量声明：

```cpp
int x = 27;                 // x是一个int
const int cx = x;           // cx是一个const int
const int& rx = x;          // rx是const int的引用
```

`param`和`T`在不同的调用下面的类型推导如下：

```cpp
f(x);                       // T是int，param的类型时int&

f(cx);                      // T是const int，
                            // param的类型是const int&
f(rx);                      // T是const int
                            // param的类型时const int&
```

在第二和第三部分的调用，注意`cx`和`rx`由于被指定为`const`类型变量，`T`被推导成`const int`，这也就导致了参数的类型被推导为`const int&`。这对调用者非常重要。当传递一个`const`对象给一个引用参数，他们期望对象会保留常量特性，也就是说，参数变成了`const`的引用。这也就是为什么给一个以`T&`为参数的模板传递一个`const`对象是安全的：对象的`const`特性是`T`类型推导的一部分。

在第三个例子中，注意尽管`rx`的类型是一个引用，`T`任然被推导成了一个非引用的。这是因为`rx`的引用特性会被类型推导所忽略。

这些例子展示了左值引用参数的处理方式，但是类型推导在右值引用上也是如此。当然，右值参数只可能传递给右值引用参数，但是这个限制和类型推导没有关系。

如果我们把`f`的参数类型从`T&`变成`const T&`，情况就会发生变化，但是并不会令人惊讶。由于`param`的声明是`const`引用的，`cx`和`rx`的`const`特性会被保留，这样的话`T`的`const`特性就没有必要了。

```cpp
template<typename T>
void f(const T& param);     // param现在是const的引用

int x = 27;                 // 和之前一样
const int cx = x;           // 和之前一样
const int& rx = x;          // 和之前一样

f(x);                       // T是int，param的类型是const int&

f(cx);                      // T是int，param的类型是const int&

f(rx);                      // T是int，param的类型是const int&
```

和之前一样，`rx`的引用特性在类型推导的过程中会被忽略。

如果`param`是一个指针（或者指向`const`的指针）而不是引用，情况也是类似：

```cpp
template<typename T>
void f(T* param);           // param是一个指针

int x = 27;                 // 和之前一样
const int *px = &x;         // px是一个指向const int x的指针

f(&x);                      // T是int，param的类型是int*

f(px);                      // T是const int
                            // param的类型时const int*
```

到目前为止，你或许瞌睡了，因为C++在引用和指针上的类型推导法则是如此的自然，我写出来读者看显得很傻比。所有的事情都这么明显！这就是读者所期望的的类型推导系统吧。

## 第二种情况：ParamType是个通用的引用（Universal Reference）