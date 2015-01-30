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

在编译的时候，编译器通过`expr`来进行推导出两个类型：一个是`T`的，另一个是`ParamType`。通常来说这些类型是不同的，因为`ParamType`通常包含一些类型的装饰，比如`const`或引用特性。举个例子，模板通常采用如下声明：

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

* `ParamType`是一个指针或者是一个引用类型，但并不是一个通用的引用类型（通用的引用类型的内容在条款24。此时，你要知道例外情况会出现的，他们的类型并不和左值应用或者右值引用）。

* `ParamType`是一个通用的引用

* `ParamType`既不是指针也不是引用

这样的话，我们就有了三种类型需要检查的类型推导场景。每一种都是基于我们队模板的通用的调用封装：

```cpp
template<typename T>
void f(ParamType param);

f(expr);					// 从expr推导出T和ParamType的类型
```

###第一种情况：`ParamType`是个非通用的引用或者是一个指针

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

在第三个例子中，注意尽管`rx`的类型是一个引用，`T`仍然被推导成了一个非引用的。这是因为`rx`的引用特性会被类型推导所忽略。

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

到目前为止，你或许瞌睡了，因为C++在引用和指针上的类型推导法则是如此的自然，我写出来读者看显得很没意思。所有的事情都这么明显！这就是读者所期望的的类型推导系统吧。

###第二种情况：`ParamType`是个通用的引用（Universal Reference）

对于通用的引用参数，情况就变得不是那么明显了。这些参数被声明成右值引用（也就是函数模板使用一个类型参数`T`，一个通用的引用参数的申明类型是`T&&`），但是当传递进去右值参数情况变得不一样。完整的讨论请参考条款24，这里是先行版本。

* 如果`expr`是一个左值，`T`和`ParamType`都会被推导成左值引用。这有些不同寻常。第一，这是模板类型`T`被推导成一个引用的唯一情况。第二，尽管`ParamType`利用右值引用的语法来进行推导，但是他最终推导出来的类型是左值引用。

* 如果`expr`是一个右值，那么就执行“普通”的法则（第一种情况）

举个例子：

```cpp
template<typename T>
void f(T&& param);			// param现在是一个通用的引用

int x = 27;                 // 和之前一样
const int cx = x;           // 和之前一样
const int& rx = x;          // 和之前一样

f(x);						// x是左值，所以T是int&
							// param的类型也是int&

f(cx);						// cx是左值，所以T是const int&
							// param的类型也是const int&

f(rx);						// rx是左值，所以T是const int&
							// param的类型也是const int&

f(27);						// 27是右值，所以T是int
							// 所以param的类型是int&&
```

条款23解释了这个例子推导的原因。关键的地方在于通用引用的类型推导法则和左值引用或者右值引用的法则大不相同。特殊的情况下，当使用了通用的引用，左值参数和右值参数的类型推导大不相同。这在非通用的类型推到上面绝对不会发生。

###第三种情况：`ParamType`既不是指针也不是引用

当`ParamType`既不是指针也不是引用，我们把它处理成pass-by-value：

```cpp
template<typename T>
void f(T param);			// param现在是pass-by-value
```

这就意味着`param`就是完全传给他的参数的一份拷贝——一个完全新的对象。基于这个事实可以从`expr`给出推导的法则：

1. 和之前一样，如果`expr`的类型是个引用，将会忽略引用的部分。

2. 如果在忽略`expr`的引用特性，`expr`是个`const`的，也要忽略掉`const`。如果是`volatile`，照样也要忽略掉（`volatile`对象并不常见。它们常常被用在实现设备驱动上面。查看更多的细节，请参考条款40。）

这样的话：

```cpp
int x = 27;                 // 和之前一样
const int cx = x;           // 和之前一样
const int& rx = x;          // 和之前一样

f(x);                       // T和param的类型都是int

f(cx);                      // T和param的类型也都是int

f(rx);                      // T和param的类型还都是int
```

注意尽管`cx`和`rx`都是`const`类型，`param`却不是`const`的。这是有道理的。`param`是一个和`cx`和`rx`独立的对象——一个`cx`和`rx`的拷贝。`cx`和`rx`不能被修改和`param`能不能被修改是没有关系的。这就是为什么`expr`的常量特性（或者是易变性）（在很多的C++书籍上面`const`特性和`volatile`特性被称之为CV特性——译者注）在推导`param`的类型的时候被忽略掉了：`expr`不能被修改并不意味着它的一份拷贝不能被修改。

认识到`const`（和`volatile`）在按值传递参数的时候会被忽略掉。正如我们所见，引用的`const`或者是指针指向`const`，`expr`的`const`特性在类型推导的过程中会被保留。但是考虑到`expr`是一个`const`的指针指向一个`const`对象，而且`expr`被通过按值传递传递给`param`：

```cpp
template<typename T>
void f(T param);            // param仍然是按值传递的（pass by value）

const char* const ptr =     // ptr是一个const指针，指向一个const对象
  "Fun with pointers";

f(ptr);                     // 给参数传递的是一个const char * const类型
```

这里，位于星号右边的`const`是表明指针是常量`const`的：`ptr`不能被修改指向另外一个不同的地址，并且也不能置成`null`。（星号左边的`const`表明`ptr`指向的——字符串——是`const`的，也就是说字符串不能被修改。）当这个`ptr`传递给`f`，组成这个指针的内存bit被拷贝给`param`。这样的话，指针自己（`ptr`）本身是被按值传递的。按照按值传递的类型推导法则，`ptr`的`const`特性会被忽略，这样`param`的推导出来的类型就是`const char*`，也就是一个可以被修改的指针，指向一个`const`的字符串。`ptr`指向的东西的`const`特性被加以保留，但是`ptr`自己本身的`const`特性会被忽略，因为它要被重新复制一份而创建了一个新的指针`param`。

##数组参数

这主要出现在mainstream的模板类型推导里面，但是有一种情况需要特别加以注意。就是数组类型和指针类型是不一样的，尽管它们通常看起来是可以替换的。一个最基本的幻觉就是在很多的情况下，一个数组会被退化成一个指向其第一个元素的指针。这个退化的代码常常如此：

```cpp
const char name[] = "J. P. Briggs";     // name的类型是const char[13]

const char * ptrToName = name;          // 数组被退化成指针
```

在这里，`const char*`指针`ptrToName`使用`name`初始化，实际的`name`的类型是`const char[13]`。这些类型（`const char*`和`const char[13]`）是不一样的，但是因为数组到指针的退化规则，代码会被正常编译。

但是如果一个数组传递给一个安置传递的模板参数里面情况会如何？会发生什么呢？

```cpp
template<typename T>
void f(T param);            // 模板拥有一个按值传递的参数

f(name);                    // T和param的类型会被推到成什么呢？
```

我们从一个没有模板参数的函数开始。是的，是的，语法是合法的，

```cpp
void myFunc(int param[]);       // 和上面的函数相同
```

但是以数组声明，但是还是把它当成一个指针声明，也就是说`myFunc`可以和下面的声明等价：

```cpp
void myFunc(int* param);        // 和上面的函数是一样的
```

这样的数组和指针等价的声明经常会在以C语言为基础的C++里面出现，这也就导致了数组和指针是等价的错觉。

因为数组参数声明会被当做指针参数，传递给模板函数的按值传递的数组参数会被退化成指针类型。这就意味着在模板`f`的调用中，模板参数`T`被推导成`const char*`：

```cpp
f(name);                    // name是个数组，但是T被推导成const char*
```

但是来一个特例。尽管函数不能被真正的定义成参数为数组，但是可以声明参数是数组的引用！所以如果我们修改模板`f`的参数成引用，

```cpp
template<typename T>
void f(T& param);           // 引用参数的模板
```

然后传一个数组给他

```cpp
f(name);                    // 传递数组给f
```

`T`最后推导出来的实际的类型就是数组！类型推导包括了数组的长度，所以在这个例子里面，`T`被推导成了`const char [13]`，函数`f`的参数（数组的引用）被推导成了`const char (&)[13]`。是的，语法看起来怪怪的，但是理解了这些可以升华你的精神（原文knowing it will score you mondo points with those few souls who care涉及到了几个宗教词汇——译者注）。

有趣的是，声明数组的引用可以使的创造出一个推导出一个数组包含的元素长度的模板：

```cpp
// 在编译的时候返回数组的长度（数组参数没有名字，
// 因为只关心数组包含的元素的个数）
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
    return N;                   // constexpr和noexcept在随后的条款中介绍
}
```

（`constexpr`是一种比`const`更加严格的常量定义，`noexcept`是说明函数永远都不会抛出异常——译者注）

正如条款15所述，定义为`constexpr`说明函数可以在编译的时候得到其返回值。这就使得创建一个和一个数组长度相同的一个数组，其长度可以从括号初始化：

```cpp
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };     // keyVals有七个元素

int mappedVals[arraySize(keyVals)];             // mappedVals长度也是七
```

当然，作为一个现代的C++开发者，应该优先选择内建的`std::array`：

```cpp
std::array<int, arraySize(keyVals)> mappedVals; // mappedVals长度是七
```

由于`arraySize`被声明称`noexcept`，这会帮助编译器生成更加优化的代码。可以从条款14查看更多详情。

##函数参数

数组并不是C++唯一可以退化成指针的东西。函数类型可以被退化成函数指针，和我们之前讨论的数组的推导类似，函数可以被推华城函数指针：

```cpp
void someFunc(int， double);    // someFunc是一个函数
                                // 类型是void(int, double)

template<typename T>
void f1(T param);               // 在f1中 参数直接按值传递

template<typename T>
void f2(T& param);              // 在f2中 参数是按照引用传递

f1(someFunc);                   // param被推导成函数指针
                                // 类型是void(*)(int, double)

f2(someFunc);                   // param被推导成函数指针
                                // 类型时void(&)(int, double)
```

这在实践中极少有不同，如果你知道数组到指针的退化，或许你也就会就知道函数到函数指针的退化。

所以你现在知道如下：`auto`相关的模板推导法则。我把最重要的部分单独在下面列出来。在通用引用中对待左值的处理有一点混乱，但是数组退化成指针和函数退化成函数指针的做法更加混乱呢。有时候你要对你的编译器和需求大吼一声，“告诉我到底类型推导成啥了啊！”当这种情况发生的时候，去参考条款4，因为它致力于让编译器告诉你是如何处理的。

|要记住的东西|
| :--------- |
| 在模板类型推导的时候，有引用特性的参数的引用特性会被忽略|
| 在推导通用引用参数的时候，左值会被特殊处理|
| 在推导按值传递的参数时候，`const`和/或`volatile`参数会被视为非`const`和非`volatile`|
| 在模板类型推导的时候，参数如果是数组或者函数名称，他们会被退化成指针，除非是用在初始化引用类型|