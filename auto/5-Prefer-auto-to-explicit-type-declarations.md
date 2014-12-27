条款三：相比于显式的类型声明更倾向于使用`auto`
=========================

使用下面语句是简单快乐的
```cpp
    int x;
```
等等。见鬼，我忘记初始化`x`了，因此它的值是无法确定的。也许，它会被初始化为0。但是这根据上下文语境决定。这真令人叹息。

不要介意。我们来看看一个要通过迭代器解引用初始化的局部变量声明的简单与快乐。
```cpp
    template<typename It>
    void dwim(It b, It e)
    {
      while(b != e){
      typename std::iterator_traits<It>::value_type
        currValue = *b;
      ...
      }
    }
```
额。`typename std::iterator_traits<It>::value_type`来表示被迭代器指向的值的类型？真的是这样吗？我必须努力不去想这是多么有趣的一件事。见鬼。等等，难道我已经说出来了。

好吧，有三个令人愉悦的地方：声明一个封装好的局部变量的类型带了的快乐。是的，这是没有问题的。一个封装体的类型只有编译器知道，因此不能被显示的写出来。哎，见鬼。

见鬼，见鬼，见鬼！使用`C++`编程并不是它本该有的愉悦体验。

是的，过去的确不是。但是由于`C++11`，这些问题都消失了得益于`auto`。`auto`变量从他们的初始化推导出其类型，所以它们必须被初始化。这就意味着你可以在现代的`C++`高速公路上对没有初始化的变量的问题说再见了。
``` cpp
    int x1;                   // potentially uninitialized
    auto x2;                  // error! initializer required
    auto x3 = 0;              // fine, x's value is well-defined
```
如上所述，高速公路上不再有由于解引用迭代器的声明局部变量而引起的坑坑洼洼。
``` cpp
    template<typename It>
    void dwim(It b, It e)
    {
      while(b != e){
        auto currValue = *b;
      ...
      }
    }
``` 
由于`auto`使用类型推导（参见条款2），它可以表示那些仅仅被编译器知晓的类型：
``` cpp
    auto dereUPLess =                         // comparison func.
      [](const std::unique_ptr<Widget>& p1,   // for Widgets
         const std::unique_ptr<Widget>& p2)   // pointed to by
      { return *p1 < *p2};                     // std::unique_ptrs
```
非常酷。在`C++14`中，模板（原文为temperature）被进一步丢弃，因为使用`lambda`表达式的参数可以包含`auto`：
``` cpp
  auto derefLess =                            // C++14 comparison
  [](const auto& p1,                          // function for
     const auto& p2)                          // values pointed
  { return *p1 < *p2; };
```
尽管非常酷，也许你在想，我们不需要使用`auto`去声明一个持有封装体的变量，因为我们可以使用一个`std::function`对象。这是千真万确的，我们可以这样干，但是也许那不是你正在思考的东西。也许你在思考“`std::function`是什么东东？”。因此让我们解释清楚。

`std::function`是`C++11`标准库的一个模板，它可以是函数指针普通化。鉴于函数指针只能指向一个函数，然而，`std::function`对象可以应用任何可以被调用的对象，就像函数。就像你声明一个函数指针的时候，必须指明这个函数指针指向的函数的类型，你产生一个`std::function`对象时，你也指明它要引用的函数的类型。你可以通过`std::function`的模板参数来完成这个工作。例如，有声明一个名为`func`的`std::function`对象，它可以引用有如下特点的可调用对象：
``` cpp
    bool(const std::unique_ptr<Widget> &,    // C++11 signature for
         const std::unique_ptr<Widget> &)    // std::unique_ptr<Widget>
                                             // comparison funtion
```
你可以这么写：
``` cpp
    std::function<bool(const std::unique_ptr<Widget> &,
                       const std::unique_ptr<Widget> &)> func; 
```
因为`lambda`表达式得到一个可调用对象，封装体可以存储在`std::function`对象里面。这意味着，我们可以声明不适用`auto`的`C++11`版本的`dereUPLess`如下：
``` cpp
    std::function<bool(const std::unique_ptr<Widget>&,
                       const std::unique_ptr<Widget>&)>
      derefUPLess = [](const std::unique_ptr<Widget>& p1,
                       const std::unique_ptr<Widget>& p2)
                      {return *p1 < *p2; };
```
意识到需要重复参数的类型这种冗余的语法是重要的，使用`std::function`和使用`auto`并不一样。一个使用`auto`声明持有一个封装的变量和封装体有同样的类型，也仅使用和封装体同样大小的内存。