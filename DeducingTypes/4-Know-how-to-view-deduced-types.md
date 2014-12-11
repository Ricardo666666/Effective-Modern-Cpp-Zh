条款4：知道如何查看类型推导
==========================

对类型推导结果的查看的工具的选择和你在软件开发过程中的相关信息有关系。我们要探讨三种可能：在你编写代码的时候，在编译的时候和在运行的时候得到类型推导的信息。

###IDE编辑器

在IDE里面的代码编辑器里面当你使用光标悬停在实体之上，常常可以显示出程序实体（例如变量，参数，函数等等）的类型。举一个例子，下面的代码：

```cpp
const int theAnswer = 42;
auto x = theAnswer;
auto y = &theAnswer;
```

一个IDE的编辑器很可能会展示出`x`的推导的类型是`int`，`y`的类型是`const int*`。

对于这样的情况，你的代码必须处在一个差不多可以编译的状态，因为这样可以使得IDE接受这种在IDE内部运行这的一个C++编译器（或者至少是一个前端）的信息。如果那个编译器无法能够有足够的能力去感知你的代码并且parse你的代码然后去执行类型推导，他就无法展示对应推导的类型了。

对于简单的类型例如`int`，IDE里面的信息是正常的。但是我们随后会发现，涉及到更加复杂的类型的时候，从IDE里面得到的信息并不一定是有帮助性的。

###编译器诊断

一个有效的让编译器展示类型的办法就是故意制造编译问题。编译的错误输出会报告会和捕捉到的类型相关错误。

假设，举个例子，我们希望看在上面例子中的`x`和`y`被推导的类型。我们首先声明一个类模板，但是并不定义这个模板。就像下面优雅的做法：

```cpp
template<typename T>                    // 声明TD
class TD;                               // TD == "Type Displayer"
```

尝试实例化这个模板会导致错误信息，因为没有模板的定义实现。想看`x`和`y`被推导的类型，只要尝试去使用这些类型去实例化`TD`：

```cpp
TD<decltype(x)> xType;                  // 引起的错误
TD<decltype(y)> yType;                  // 包含了x和y的类型
```
我使用的变量名字的形式`variableNameType`是因为这样有利于输出的错误信息可以帮助我定位我要寻找的信息。对上面的代码，我的一个编译器输出了诊断信息，其中的一部分如下：（我把我们关注的类型信息高亮了（原文中高亮了模板中的`int`和`const int*`，但是Markdown在代码block中操作粗体比较麻烦，译文中没有加粗——译者注））：

    error: aggregate 'TD<int> xType' has incomplete type and cannot be defined
    error: aggregate 'TD<const int *> yType' has incomplete type and cannot be defined

另一个编译器提供相同的信息，但是格式不太一样：

    error: 'xType' uses undefined class 'TD<int>'
    error: 'yType' uses undefined class 'TD<const int *>'

排除格式的区别，我测试了所有的编译器都会在这种代码的技术中输出有用的错误信息。

###运行时输出