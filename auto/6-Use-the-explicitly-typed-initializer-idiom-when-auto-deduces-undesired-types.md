条款6：当auto推导出非预期类型时应当使用显式的类型初始化
===============================================

条款5解释了使用`auto`关键字去声明变量，这样就比直接显示声明类型提供了一系列的技术优势，但是有时候`auto`的类型推导会和你想的南辕北辙。举一个例子，假设我有一个函数接受一个`Widget`返回一个`std::vector<bool>`，其中每个`bool`表征`Widget`是否接受一个特定的特性：

```cpp
std::vector<bool> features(const Widget& w);
```

进一步的，假设第五个bit表示`Widget`是否有高优先级。我们可以这样写代码：

```cpp
Widget w;
…
bool highPriority = features(w)[5];         // w是不是个高优先级的？
…
processWidget(w, highPriority);             // 配合优先级处理w
```

这份代码没有任何问题。它工作正常。但是如果我们做一个看起来无伤大雅的修改，把`highPriority`的显式的类型换成`auto`：

```cpp
auto highPriority = features(w)[5];         // w是不是个高优先级的？
```

情况变了。所有的代码还是可以编译，但是他的行为变得不可预测：

```cpp
processWidget(w, highPriority);             // 未定义行为
```

正如注释中所提到的，调用`processWidget`现在会导致未定义的行为。但是为什么呢？答案是非常的令人惊讶的。在使用`auto`的代码中，`highPriority`的类型已经不是`bool`了。尽管`std::vector<bool>`从概念上说是`bool`的容器，对`std::vector<bool>`的`operator[]`运算符并不一定是返回容器中的元素的引用（`std::vector::operator[]`对所有的类型都返回引用，就是除了`bool`）。事实上，他返回的是一个`std::vector<bool>::reference`对象（是一个在`std::vector<bool>`中内嵌的class）。

`std::vector<bool>::reference`存在是因为`std::vector<bool>`是对`bool`数据封装的模板特化，一个bit对应一个`bool`。这就给`std::vector::operator[]`带来了问题，因为`std::vector<T>`的`operator[]`应该返回一个`T&`，但是C++禁止bits的引用。没办法返回一个`bool&`，`std::vector<T>`的`operator[]`于是就返回了一个行为上和`bool&`相似的对象。想要这种行为成功，`std::vector<bool>::reference`对象必须能在`bool&`的能处的语境中使用。在`std::vector<bool>::reference`对象的特性中，是他隐式的转换成`bool`才使得这种操作得以成功。（不是转换成`bool&`，而是`bool`。去解释详细的`std::vector<bool>::reference`对象如何模拟一个`bool&`的行为有有些偏离主题，所以我们就只是简单的提一下这种隐式转换只是这种技术中的一部。）

在大脑中带上这种信息，再次阅读原先的代码：

```cpp
bool highPriority = features(w)[5];         // 直接显示highPriority的类型
```

这里，`features`返回了一个`std::vector<bool>`对象，在这里`operator[]`被调用。`operator[]`返回一个`std::vector<bool>::reference`对象，这个然后隐式的转换成`highPriority`需要用来初始化的`bool`类型。于是就以`features`返回的`std::vector<bool>`的第五个bit的数值来结束`highPriority`的数值，这也是我们所预期的。

和使用`auto`的`highPriority`声明进行对比：

```cpp
auto highPriority = features(w)[5];         // 推导highPriority的类型
```

这次，`features`返回一个`std::vector<bool>`对象，而且，`operator[]`再次被调用。`operator[]`继续返回一个`std::vector<bool>::reference`对象，但是现在有一个变化，因为`auto`推导`highPriority`的类型。`highPriority`根本并没有`features`返回的`std::vector<bool>`的第五个bit的数值。

数值和`std::vector<bool>::reference`是如何实现的是有关系的。一种实现是这样的对象包含一个指向包含bit引用的机器word的指针，在word上面加上偏移。考虑这个对`highPriority`的初始化的意义，假设`std::vector<bool>::reference`的实现是恰当的。

调用`features`会返回一个临时的`std::vector<bool>`对象。这个对象是没有名字的，但是对于这个讨论的目的，我会把它叫做`temp`，`operator[]`是在`temp`上调用的，`std::vector<bool>::reference`返回一个由`temp`管理的包含一个指向一个包含bits的数据结构的指针，在word上面加上偏移定位到第五个bit。`highPriority`也是一个`std::vector<bool>::reference`对象的一份拷贝，所以`highPriority`也在`temp`中包含一个指向word的指针，加上偏移定位到第五个bit。在这个声明的结尾，`temp`被销毁，因为它是个临时对象。因此，`highPriority`包含一个野指针，这也就是调用`processWidget`会造成未定义的行为的原因：

```cpp
processWidget(w, highPriority);         // 未定义的行为，highPriority包含野指针
```

`std::vector<bool>::reference`是代理类的一个例子：一个类的存在是为了模拟和对外行为和另外一个类保持一致。代理类在各种各样的目的上被使用。`std::vector<bool>::reference`的存在是为了提供一个对`std::vector<bool>`的`operator[]`的错觉，让它返回一个对bit的引用，而且标准库的智能指针类型（参考第4章）也是一些对托管的资源的代理类，使得他们的资源管理类似于原始指针。代理类的功能是良好确定的。事实上，“代理”模式是软件设计模式中的最坚挺的成员之一。

一些代理类被设计用来隔离用户。这就是`std::shared_ptr`和`std::unique_ptr`的情况。另外一些代理类是为了一些或多或少的不可见性。`std::vector<bool>::reference`就是这样一个“不可见”的代理，和他类似的是`std::bitset`，对应的是`std::bitset::reference`。

同时在一些C++库里面的类存在一种被称作表达式模板的技术。这些库最开始是为了提高数值运算的效率。提供一个`Matrix`类和`Matrix`对象`m1, m2, m3 and m4`，举一个例子，下面的表达式：

```cpp
Matrix sum = m1 + m2 + m3 + m4;
```

可以计算的更快如果`Matrix`的`operator+`返回一个结果的代理而不是结果本身。这是因为，对于两个`Matrix`，`operator+`可能返回一个类似于`Sum<Matrix, Matrix>`的代理类而不是一个`Matrix`对象。和`std::vector<bool>::reference`一样，这里会有一个隐式的从代理类到`Matrix`的转换，这个可能允许`sum`从由`=`右边的表达式产生的代理对象进行初始化。（其中的对象可能会编码整个初始化表达式，也就是，变成一种类似于`Sum<Sum<Sum<Matrix, Matrix>, Matrix>, Matrix>`的类型。这是一个客户端需要屏蔽的类型。）

作为一个通用的法则，“不可见”的代理类不能和`auto`愉快的玩耍。这种类常常它的生命周期不会被设计成超过一个单个的语句，所以创造这样的类型的变量是会违反库的设计假定。这就是`std::vector<bool>::reference`的情况，而且我们可以看到这种违背约定的做法会导致未定义的行为。

因此你要避免使用下面的代码的形式：

```cpp
auto someVar = expression of "invisible" proxy class type;
```

但是你怎么能知道代理类被使用呢？软件使用它们的时候并不可能会告知它们的存在。它们是不可见的，至少在概念上！一旦你发现了他们，难道你就必须放弃使用`auto`加之条款5所声明的`auto`的各种好处吗？

我们先看看怎么解决如何发现它们的问题。尽管“不可见”的代理类被设计用来fly beneath programmer radar in day-to-day use，库使用它们的时候常常会撰写关于它们的文档来解释为什么这样做。你对你所使用的库的基础设计理念越熟悉，你就越不可能在这些库中被代理的使用搞得狼狈不堪。