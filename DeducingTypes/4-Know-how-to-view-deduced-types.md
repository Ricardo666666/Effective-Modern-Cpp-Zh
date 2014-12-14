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

`printf`到运行的时候可以用来显示类型信息（这并不是我推荐你使用`printf`的原因），但是它提供了对输出格式的完全掌控。挑战就在于你要创造一个你关心的对象的输出的格式控制展示的textual。“这还不容易，”你会这样想，“就是用`typeid`和`std::type_info::name`来救场啊。”在后续的对`x`和`y`的类型推导中，你可以发现你可以这样写：

```cpp
std::cout << typeid(x).name() << '\n'; // display types for
std::cout << typeid(y).name() << '\n'; // x and y
```

这是基于对类似于`x`或者`y`运算`typeid`可以得到一个`std::type_info`对象，`std::type_info`有一个成员函数，`name`可以提供一个C-style的字符串（也就是`const char*`）代表了类型的名字。

调用`std::type_info::name`并不会确定返回有意义的东西，但是实现上是有帮助性质的。帮助是多种多样的。举一个例子，GNU和Clang编译器返回`x`的类型是“`i`”，`y`的类型是“`PKi`”。这些编译器的输出结果你一旦学会就可以理解他们，“`i`”意味着“`int`”，“`PK`”意味着“pointer to ~~konst~~ const”（所有的编译器都支持一个工具，`C++filt`，它可以解析这样的“乱七八糟”的类型。）微软的编译器提供更加直白的输出：“`int`”对`x`，“`int const*`”对`y`。

因为这些结果对`x`和`y`而言都是正确的，你可能认为类型输出的问题就此解决了，但是这并不能轻率。考虑一个更加复杂的例子：

```cpp
template<typename T>                // template function to
void f(const T& param);             // be called

std::vector<Widget> createVec();    // 工厂方法

const auto vw = createVec();        // init vw w/factory return

if (!vw.empty()) {
    f(&vw[0]);                      // 调用f
    …
}
```

在代码中，涉及了一个用户定义的类型（`Widget`），一个STL容器（`std::vector`），一个`auto`变量（`vw`），这对你的编译器的类型推导的可视化是非常具有表现性的。举个例子，想看到模板类型参数`T`和`f`的函数模板参数`param`。

在问题中没有`typeid`是很直接的。在`f`中添加一些代码去展示你想要的类型：

```cpp
template<typename T>
void f(const T& param)
{
    using std::cout;
    cout << "T = " << typeid(T).name() << '\n';         // 展示T
    cout << "param = " << typeid(param).name() << '\n'; // 展示param的类型
    …
}
```

使用GNU和Clang编译器编译会输出如下结果：

    T = PK6Widget
    param = PK6Widget

我们已经知道对于这些编译器，`PK`意味着“pointer to `const`”，所以比较奇怪的就是数字6，这是在后面跟着的类的名字(`Widget`)的字母字符的长度。所以这些编译器就告我我们`T`和`param`的类型都是`const Widget*`。

微软的编译器输出：

    T = class Widget const *
    param = class Widget const *

三种不同的编译器都产出了相同的建议性信息，这表明信息是准确的。但是更加仔细的分析，在模板`f`中，`param`的类型是`const T&`。`T`和`param`的类型是一样的难道不会感到奇怪吗？举个例子，如果`T`是`int`，`param`的类型应该是`const int&`——根本不是相同的类型。

悲剧的是，`std::type_info::name`的结果并不可靠。在这种情况下，举个例子，所有的三种编译器报告的`param`的类型都是不正确的。更深入的话，它们本来就是不正确的，因为`std::type_info::name`的特化指定了类型会被当做它们被传给模板函数的时候的按值传递的参数。正如条款1所述，这就意味着如果类型是一个引用，他的引用特性会被忽略，如果在忽略引用之后存在`const`（或者`volatile`），它的`const`特性（或者`volatile`特性）会被忽略。这就是为什么`param`的类型——`const Widget * const &`——被报告成了`const Widget*`。首先类型的引用特性被去掉了，然后结果参数指针的`const`特性也被消除了。

同样的悲剧，由IDE编辑器显示的类型信息也并不准确——或者说至少并不可信。对之前的相同的例子，一个我知道的IDE的编辑器报告出`T`的类型（我不打算说）：

```cpp
const
std::_Simple_types<std::_Wrap_alloc<std::_Vec_base_types<Widget,
std::allocator<Widget> >::_Alloc>::value_type>::value_type *
```

还是这个相同的IDE编辑器，`param`的类型是：

```cpp
const std::_Simple_types<...>::value_type *const &
```

这个没有`T`的类型那么吓人，但是中间的“...”会让你感到困惑，直到你发现这是IDE编辑器的一种说辞“我们省略所有`T`类型的部分”。带上一点运气，你的开发环境也许会对这样的代码有着更好的表现。

如果你更加倾向于库而不是运气，你就应该知道`std::type_info::name`可能在IDE中会显示类型失败，但是Boost TypeIndex库（经常写做Boost.TypeIndex）是被设计成可以成功显示的。这个库并不是C++标准的一部分，也不是IDE和模板的一部分。更深层的是，事实上Boost库（在[boost.com](http://boost.com/)）是一个跨平台的，开源的，并且基于一个偏执的团队都比较喜欢的协议。这就意味着基于标准库之上使用Boost库的代码接近于一个跨平台的体验。

这里展示了一段我们使用Boost.TypeIndex的函数`f`精准的输出类型信息：

```cpp
#include <boost/type_index.hpp>
template<typename T>
void f(const T& param)
{
    using std::cout;
    using boost::typeindex::type_id_with_cvr;

    // show T
    cout << "T = "
         << type_id_with_cvr<T>().pretty_name()
         << '\n';

    // show param's type
    cout << "param = "
         << type_id_with_cvr<decltype(param)>().pretty_name()
         << '\n';
    …
}
```

这个模板函数`boost::typeindex::type_id_with_cvr`接受一个类型参数（我们想知道的类型信息）来正常工作，它不会去除`const`，`volatile`或者引用特性（这也就是模板中的“`cvr`”的意思）。返回的结果是个`boost::typeindex::type_index`对象，其中的`pretty_name`成员函数产出一个`std::string`包含一个对人比较友好的类型展示的字符串。

通过这个`f`的实现，再次考虑之前使用`typeid`导致推导出现错误的`param`类型信息：

```cpp
std::vector<Widget> createVec();    // 工厂方法

const auto vw = createVec();        // init vw w/factory return

if (!vw.empty()) {
    f(&vw[0]);                      // 调用f
    …
}
```

在GNU和Clang的编译器下面，Boost.TypeIndex输出（准确）的结果：

    T = Widget const*
    param = Widget const* const&

微软的编译器实际上输出的结果是一样的：

    T = class Widget const *
    param = class Widget const * const &

这种接近相同的结果很漂亮，但是需要注意IDE编辑器，编译器错误信息，和类似于Boost.TypeIndex的库仅仅是一个对你编译类型推导的一种工具而已。所有的都是有帮助意义的，但是到目前为止，没有什么关于类型推导法则1-3的替代品。

|要记住的东西|
| :--------- |
| 类型推导的结果常常可以通过IDE的编辑器，编译器错误输出信息和Boost TypeIndex库的结果中得到|
| 一些工具的结果不一定有帮助性也不一定准确，所以对C++标准的类型推导法则加以理解是很有必要的|
