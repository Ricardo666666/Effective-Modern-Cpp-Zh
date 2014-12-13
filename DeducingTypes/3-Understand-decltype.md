条款三：理解`decltype`
=========================

`decltype`是一个怪异的发明。给定一个变量名或者表达式，`decltype`会告诉你这个变量名或表达式的类型。`decltype`的返回的类型往往也是你期望的。然而有时候，它提供的结果会使开发者极度抓狂而不得参考其他文献或者在线的Q&A网站。

我们从在典型的情况开始讨论，这种情况下`decltype`不会有令人惊讶的行为。与`templates`和`auto`在类型推导中行为相比（请见条款一和条款二），`decltype`一般只是复述一遍你所给他的变量名或者表达式的类型，如下：

```cpp
   const int i = 0;            // decltype(i) is const int

   bool f(const Widget& w);    // decltype(w) is const Widget&
                               // decltype(f) is bool(const Widget&)
   struct Point{
     int x, y;                 // decltype(Point::x) is int
   };

   Widget w;                   // decltype(w) is Widget

   if (f(w)) ...               // decltype(f(w)) is bool

   template<typename T>        // simplified version of std::vector
   class vector {
   public:
     ...
     T& operator[](std::size_t index);
     ...
   };

   vector<int> v;              // decltype(v) is vector<int>
   ...
   if(v[0] == 0)               // decltype(v[0]) is int&
```
看到没有？毫无令人惊讶的地方。

在C++11中，`decltype`最主要的用处可能就是用来声明一个函数模板，在这个函数模板中返回值的类型取决于参数的类型。举个例子，假设我们想写一个函数，这个函数中接受一个支持方括号索引（也就是"[]"）的容器作为参数，验证用户的合法性后返回索引结果。这个函数的返回值类型应该和索引操作的返回值类型是一样的。

操作子`[]`作用在一个对象类型为`T`的容器上得到的返回值类型为`T&`。对`std::deque`一般是成立的，例如，对`std::vector`，这个几乎是处处成立的。然而，对`std::vector<bool>`，`[]`操作子不是返回`bool&`，而是返回一个全新的对象。发生这种情况的原理将在条款六中讨论，对于此处重要的是容器的`[]`操作返回的类型是取决于容器的。

`decltype`使得这种情况很容易来表达。下面是一个模板程序的部分，展示了如何使用`decltype`来求返回值类型。这个模板需要改进一下，但是我们先推迟一下：

```cpp
    template<typename Container, typename Index>    // works, but
    auto authAndAccess(Container& c, Index i)       // requires
      -> decltype(c[i])                             // refinements
    {
      authenticateUser();
      return c[i];
    }
```
将`auto`用在函数名之前和类型推导是没有关系的。更精确地讲，此处使用了`C++11`的尾随返回类型技术，即函数的返回值类型在函数参数之后声明(“->”后边)。尾随返回类型的一个优势是在定义返回值类型的时候使用函数参数。例如在函数`authAndAccess`中，我们使用了`c`和`i`定义返回值类型。在传统的方式下，我们在函数名前面声明返回值类型，`c`和`i`是得不到的，因为此时`c`和`i`还没被声明。

使用这种类型的声明，`authAndAccess`的返回值就是`[]`操作子的返回值，这正是我们所期望的。

`C++11`允许单语句的`lambda`表达式的返回类型被推导，在`C++14`中之中行为被拓展到包括多语句的所有的`lambda·表达式和函数。在上面`authAndAccess`中，意味着在`C++14`中我们可以忽略尾随返回类型，仅仅保留开头的`auto`。使用这种形式的声明，
意味着将会使用类型推导。特别注意的是，编译器将从函数的实现来推导这个函数的返回类型：

```cpp
    template<typename Container, typename Index>         // C++14;
    auto authAndAccess(Container &c, Index i)            // not quite
    {                                                    // correct
      authenticateUser();
      return c[i];
    }                                 // return type deduced from c[i]
```

<font color='#990000'>条款二</font>解释说，对使用`auto`来表明函数返回类型的情况，编译器使用模板类型推导。但是这样是回产生问题的。正如我们所讨论的，对绝大部分对象类型为`T`的容器，`[]`操作子返回的类型是`&T`, 然而<font color='#990000'>条款一</font>提到，在模板类型推导的过程中,初始表达式的引用会被忽略。思考这对下面代码意味着什么：

```cpp
    std::deque<int> d;
    ...
    authAndAccess(d, 5) = 10;       // authenticate user, return d[5],
                                    // then assign 10 to it;
                                    // this won't compile!
```

此处，`d[5]`返回的是`int&`，但是`authAndAccess`的`auto`返回类型声明将会剥离这个引用，从而得到的返回类型是`int`。`int`作为一个右值成为真正的函数返回类型。上面的代码尝试给一个右值`int`赋值为10。这种行为是在`C++`中被禁止的，所以代码无法编译通过。

为了让`authAndAccess`按照我们的预期工作，我们需要为它的返回值使用`decltype`类型推导，即指定`authAndAccess`要返回的类型正是表达式`c[i]`的返回类型。`C++`的拥护者们预期到在某种情况下有使用`decltype`类型推导规则的需求，并将这个功能在`C++14`中通过`decltype(auto)`实现。这使这对原本的冤家（`decltype`和`auto`）在一起完美地发挥作用：`auto`指定需要推导的类型，`decltype`表明在推导的过程中使用`decltype`推导规则。因此，我们可以重写`authAndAccess`如下：

```cpp
    template<typename Container, typename Index>  // C++14; works,
    decltype(auto)                                // but still
    authAndAccess(Container &c, Index i)          // requires
    {                                             // refinement
      authenticateUser();
      return c[i];
  }
```

现在`authAndAccess`的返回类型就是`c[i]`的返回类型。在一般情况下，`c[i]`返回`T&`，`authAndAccess`就返回`T&`，在不常见的情况下，`c[i]`返回一个对象，`authAndAccess`也返回一个对象。

`decltype(auto)`并不仅限使用在函数返回值类型上。当时想对一个表达式使用`decltype`的推导规则时，它也可以很方便的来声明一个变量：

```cpp
    Widget w;
    const Widget& cw = w;
    auto myWidget1 = cw;                     // auto type deduction
                                             // myWidget1's type is Widget 
    decltype(auto) myWidget2 = cw            // decltype type deduction:
                                             // myWidget2's type is
                                             //  const Widget&
```

我知道，到目前为止会有两个问题困扰着你。一个是我们前面提到的，对`authAndAccess`的改进。我们在这里讨论。

再次看一下`C++14`版本的`authAndAccess`的声明：

```cpp
    template<typename Container, typename Index>
    decltype(auto) anthAndAccess(Container &c, Index i);
```

这个容器是通过非`const`左值引用传入的，因为通过返回一个容器元素的引用是来修改容器是被允许的。但是这也意味着不可能将右值传入这个函数。右值不能和一个左值引用绑定（除非是`const`的左值引用，这不是这里的情况）。

诚然，传递一个右值容器给`authAndAccess`是一种极端情况。一个右值容器作为一个临时对象，在 `anthAndAccess` 所在语句的最后被销毁，意味着对容器中一个元素的引用（这个引用通常是`authAndAccess`返回的）在创建它的语句结束的地方将被悬空。然而，这对于传给`authAndAccess`一个临时对象是有意义的。一个用户可能仅仅想拷贝一个临时容器中的一个元素，例如：

```cpp
    std::deque<std::string> makeStringDeque(); // factory function
    // make copy of 5th element of deque returned
    // from makeStringDeque
    auto s = authAndAccess(makeStringDeque(), 5);
```

支持这样的应用意味着我们需要修改`authAndAccess`的声明来可以接受左值和右值。重载可以解决这个问题（一个重载负责左值引用参数，另外一个负责右值引用参数），但是我们将有两个函数需要维护。避免这种情况的一个方法是使`authAndAccess`有一个既可以绑定左值又可以绑定右值的引用参数，条款24将说明这正是统一引用（`universal reference`）所做的。因此`authAndAccess`可以像如下声明：

```cpp
    template<typename Container, typename Index>  // c is now a
    decltype(auto) authAndAccess(Container&& c,   // universal
                                 Index i);        // reference
```

在这个模板中，我们不知道我们在操作什么类型的容器，这也意味着我们等同地忽略了它用到的索引对象的类型。对于一个不清楚其类型的对象使用传值传递通常会冒一些风险，比如因为不必要的复制而造成的性能降低，对象切片的行为问题，被同事嘲笑，但是对容器索引的情况，正如一些标准库的索引（`std::string, std::vector, std::deque`的`[]`操作）按值传递看上去是合理的，因此对它们我们仍坚持按值传递。

然而，我们需要更新这个模板的实现，将`std::forward`应用给统一引用，使得它和条款25中的建议是一致的。

```cpp
    template<typename Container, typename Index>     // final
    decltype(auto)                                   // C++14
    authAndAccess(Container&& c, Index i)            // version
    {
      authenticateUser();
      return std::forward<Container>(c)[i];
    }
```

这个实现可以做我们期望的任何事情，但是它要求使用支持`C++14`的编译器。如果你没有一个这样的编译器，你可以使用这个模板的`C++11`版本。它出了要你自己必须指定返回类型以外，和对应的`C++14`版本是完全一样的，

```cpp
    template<typename Container, typename Index>    // final
    auto                                            // C++11
    authAndAccess(Container&& c, Index i)           // version
    -> decltype(std::forward<Container>(c)[i])
    {
      authenticateUser();
      return std::forward<Container>(c)[i];
    }
```

另外一个容易被你挑刺的地方是我在本条款开头的那句话：`decltype`几乎所有时候都会输出你所期望的类型，但是有时候它的输出也会令你吃惊。诚实的讲，你不太可能遇到这种以外，除非你是一个重型库的实现人员。

为了彻底的理解`decltype`的行为，你必须使你自己对一些特殊情况比较熟悉。这些特殊情况太晦涩难懂，以至于很少有书会像本书一样讨论，但是同时也可以增加我们对`decltype`的认识。

对一个变量名使用`decltype`得到这个变量名的声明类型。变量名属于左值表达式，但这并不影响`decltype`的行为。然而，对于一个比变量名更复杂的左值表达式，`decltype`保证返回的类型是左值引用。因此说，如果一个非变量名的类型为`T`的左值表达式，`decltype`报告的类型是`T&`。这很少产生什么影响，因为绝大部分左值表达式的类型有内在的左值引用修饰符。例如，需要返回左值的函数返回的总是左值引用。

这种行为的意义是值得我们注意的。但是在下面这个语句中
```cpp
    int x = 0;
```
`x`是一个变量名，因此`decltyper(x)`是`int`。但是如果给`x`加上括号"(x)"就得到一个比变量名复杂的表达式。作为变量名，`x`是一个左值，同时`C++`定义表达式`(x)`也是左值。因此`decltype((x))`是`int&`。给一个变量名加上括号会改变`decltype`返回的类型。

在`C++11`中，这仅仅是个好奇的探索，但是和`C==14`中对`decltype(auto)`支持相结合，函数中返回语句的一个细小改变会影响对这个函数的推导类型。
```cpp
    decltype(auto) f1()
    {
      int x = 0;
      ...
      return x;        // decltype(x) is int, so f1 returns int
    }

    decltype(auto) f2()
    {
      int x = 0;
      return (x);     // decltype((x)) is int&, so f2 return int&
    }
```
`f2`不仅返回值类型与`f1`不同，它返回的是对一个局部变量的引用。这种类型的代码将把你带上一个为定义行为的快速列车-你完全不想登上的列车。

最主要的经验教训就是当使用`decltype(auto)`时要多留心一些。被推导的表达式中看上去无关紧要的细节都可能影响`decltype`返回的类型。为了保证推导出的类型是你所期望的，请使用条款4中的技术。

同时不能更大视角上的认识。当然，`decltype`（无论只有`decltype`或者还是和`auto`联合使用）有可能偶尔会产生类型推导的惊奇行为，但是这不是常见的情况。一般情况下，`decltype`会产生你期望的类型。将`decltype`应用于变量名无非是正确的，因为在这种情况下，`decltype`做的就是报告这个变量名的声明类型。

|要记住的东西|
|:--------- |
|`decltype`几乎总是得到一个变量或表达式的类型而不需要任何修改|
|对于非变量名的类型为`T`的左值表达式，`decltype`总是返回`T&`|
|`C++14`支持`decltype(auto)`，它的行为就像`auto`,从初始化操作来推导类型，但是它推导类型时使用`decltype`的规则|
