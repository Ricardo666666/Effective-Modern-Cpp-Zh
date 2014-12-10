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

