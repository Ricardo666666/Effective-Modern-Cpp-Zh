条款五：优先使用`auto`而非显式类型声明
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

好吧，有三个令人愉悦的地方：声明一个封装好的局部变量的类型带来的快乐。是的，这是没有问题的。一个封装体的类型只有编译器知道，因此不能被显示的写出来。哎，见鬼。

见鬼，见鬼，见鬼！使用`C++`编程并不是它本该有的愉悦体验。

是的，过去的确不是。但是由于`C++11`，得益于`auto`，这些问题都消失了。`auto`变量从他们的初始化推导出其类型，所以它们必须被初始化。这就意味着你可以在现代的`C++`高速公路上对没有初始化的变量的问题说再见了。
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

`std::function`是`C++11`标准库的一个模板，它可以使函数指针普通化。鉴于函数指针只能指向一个函数，然而，`std::function`对象可以应用任何可以被调用的对象，就像函数。就像你声明一个函数指针的时候，必须指明这个函数指针指向的函数的类型，你产生一个`std::function`对象时，你也指明它要引用的函数的类型。你可以通过`std::function`的模板参数来完成这个工作。例如，有声明一个名为`func`的`std::function`对象，它可以引用有如下特点的可调用对象：

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

意识到需要重复参数的类型这种冗余的语法是重要的，使用`std::function`和使用`auto`并不一样。一个使用`auto`声明持有一个封装的变量和封装体有同样的类型，也仅使用和封装体同样大小的内存。持有一个封装体的被`std::function`声明的变量的类型是`std::function`模板的一个实例，并且对任何类型只有一个固定的大小。这个内存大小可能不能满足封装体的需求。出现这种情况时，`std::function`将会开辟堆空间来存储这个封装体。导致的结果就是`std::function`对象一般会比`auto`声明的对象使用更多的内存。由于实现细节中，约束内嵌的使用和提供间接函数的调用，通过`std::function`对象来调用一个封装体比通过`auto`对象要慢。换言之，`std::function`方法通常体积比`auto`大，并且慢，还有可能导致内存不足的异常。就像你在上面一个例子中看到的，使用`auto`的工作量明显小于使用`std::function`。持有一个封装体时，`auto`和`std::function`之间的竞争，对`auto`简直就是游戏。（一个相似的论点也成立对于持有`std::blind`调用结果的`auto`和`std::function`，但是在条款34中，我将竭尽所能的说服你尽可能使用`lambda`表达式，而不是`std::blind`）。

`auto`的优点除了可以避免未初始化的变量，变量声明引起的歧义，直接持有封装体的能力。还有一个就是可以避免“类型截断”问题。下面有个例子，你可能见过或者写过：

```cpp
   std::vector<int> v;
   ...
   unsigned sz = v.size();
```

`v.size()`定义的返回类型是`std::vector<int>::size_type`，但是很少有开发者对此十分清楚。`std::vector<int>::size_type`被指定为一个非符号的整数类型，因此很多程序员认为`unsigned`类型是足够的，然后写出了上面的代码。这将导致一些有趣的后果。比如说在32位`Windows`系统上，`unsigned`和`std::vector<int>::size_type`有同样的大小，但是在64位的`Windows`上，`unsigned`是32bit的，而`std::vector<int>::size_type`是64bit的。这意味着上面的代码在32位`Windows`系统上工作良好，但是在64位`Windows`系统上时有可能不正确，当应用程序从32位移植到64位上时，谁又想在这种问题上浪费时间呢？
使用`auto`可以保证你不必被上面的东西所困扰：

```cpp
    auto sz = v.size()    // sz's type is std::vector<int>::size_type
```

仍然不太确定使用`auto`的高明之处？看看下面的代码：

```cpp
   std::unordered_map<std::string, int> m;
   ...

   for (const std::pair<std::string, int>& p : m)
   {
   ...                  // do something with p
   }
```

这看上去完美合理。但是有一个问题，你看出来了吗？
意识到`std::unorder_map`的`key`部分是`const`类型的，在哈希表中的`std::pair`的类型不是`std::pair<std::string, int>`，而是`std::pair<const std::sting, int>`。但是这不是循环体外变量`p`的声明类型。后果就是，编译器竭尽全力去找到一种方式，把`std::pair<const std::string, int>`对象（正是哈希表中的内容）转化为`std::pair<std::string, int>`对象（`p`的声明类型）。这个过程将通过复制`m`的一个元素到一个临时对象，然后将这个临时对象和`p`绑定完成。在每个循环结束的时候这个临时对象将被销毁。如果是你写了这个循环，你将会感觉代码的行为令人吃惊，因为你本来想简单地将引用`p`和`m`的每个元素绑定的。
这种无意的类型不匹配可以通过`auto`解决

```cpp
    for (const auto& p : m)
    {
      ...                     // as before
    }
```

这不仅仅更高效，也更容易敲击代码。更近一步，这个代码还有一些吸引人的特性，比如如果你要取`p`的地址，你的确得到一个指向`m`的元素的指针。如果不使用`auto`，你将得到一个指向临时对象的指针——这个临时对象在每次循环结束时将被销毁。

上面两个例子中——在应该使用`std::vector<int>::size_type`的时候使用`unsigned`和在该使用`std::pair<const std::sting, int>`的地方使用`std::pair<std::string, int>`——说明显式指定的类型是如何导致你万万没想到的隐式的转换的。如果你使用`auto`作为目标变量的类型，你不必为你声明类型和用来初始化它的表达式类型之间的不匹配而担心。

有好几个使用`auto`而不是显式类型声明的原因。然而，`auto`不是完美的。`auto`变量的类型都是从初始化它的表达式推导出来的，一些初始化表达式并不是我们期望的类型。发生这种情况时，你可以参考条款2和条款6来决定怎么办，我不在此处展开了。相反，我将我的精力集中在你将传统的类型声明替代为`auto`时带来的代码可读性问题。

首先，深呼吸放松一下。`auto`是一个可选项，不是必须项。如果根据你的专业判断，使用显式的类型声明比使用`auto`会使你的代码更加清晰或者更好维护，或者在其他方面更有优势，你可以继续使用显式的类型声明。牢记一点，`C++`并没有在这个方面有什么大的突破，这种技术在其他语言中被熟知，叫做类型推断(`type inference`)。其他的静态类型过程式语言（像`C#`，`D`，`Scala`，`Visual Basic`）也有或多或少等价的特点，对静态类型的函数编程语言（像`ML`，`Haskell`，`OCaml`，`F#`等）另当别论。一定程度上说，这是受到动态类型语言的成功所启发，比如`Perl`，`Python`，`Ruby`，在这些语言中很少显式指定变量的类型。软件开发社区对于类型推断有很丰富的经验，这些经验表明这些技术和创建及维护巨大的工业级代码库没有矛盾。

一些开发者被这样的事实困扰，使用`auto`会消除看一眼源代码就能确定对象的类型的能力。然而，IDE提示对象类型的功能经常能缓解这个问题（甚至考虑到在条款4中提到的IDE的类型显示问题），在很多情况下，一个对象类型的摘要视图和显示完全的类型一样有用。比如，摘要视图足以让开发者知道这个对象是容器还是计数器或者一个智能指针，而不需要知道这个容器，计数器或者智能指针的确切特性。假设比较好的选择变量名字，这样的摘要类型信息几乎总是唾手可得的。

事实是显式地写出类型可能会引入一些难以察觉的错误，导致正确性或者效率问题，或者两者兼而有之。除此之外，`auto`类型会自动的改变如果初始化它的表达式改变后，这意味着通过使用`auto`可以使代码重构变得更简单。举个例子，如果一个函数被声明为返回`int`，但是你稍后决定返回`long`可能更好一些，如果你把这个函数的返回结果存储在一个`auto`变量中，在下次编译的时候，调用代码将会自动的更新。结果如果存储在一个显式声明为`int`的变量中，你需要找到所有调用这个函数的地方然后改写他们。

|要记住的东西|
| :--------- |
| `auto`变量一定要被初始化，并且对由于类型不匹配引起的兼容和效率问题有免疫力，可以简单化代码重构，一般会比显式的声明类型敲击更少的键盘|
| `auto`类型的变量也受限于[条款2](../DeducingTypes/2-Understand-auto-type-deduction.html)和条款6中描述的陷阱|