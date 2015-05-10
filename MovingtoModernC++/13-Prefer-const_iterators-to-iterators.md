条款13：优先使用const_iterator而不是iterator
===============

`const_iterator`在STL中等价于指向`const`的指针。被指向的数值是不能被修改的。标准的做法是应该使用`const`的迭代器的地方，也就是尽可能的在没有必要修改指针所指向的内容的地方使用`const_iterator`。

这对于C++98和C++11是正确，但是在C++98中，`const_iterator`s只有部分的支持。一旦有一个这样的迭代器，创建它们并非易事，使用也会受限。举一个例子，假如你希望从`vector<int>`搜索第一次出现的1983(这一年"C++"替换"C + 类"而作为一个语言的名字)，然iterator后在搜到的位置插入数值1998(这一年第一个ISO C++标准被接受)。如果在vector中并不存在1983，插入操作的位置应该是vector的末尾。在C++98中使用`iterator`，这会非常容易：

```cpp
std::vector<int> values;
…
std::vector<int>::iterator it =
std::find(values.begin(),values.end(), 1983);
values.insert(it, 1998);
```

在这里`iterator`并不是合适的选择，因为这段代码永远都不会修改`iterator`指向的内容。重新修改代码，改成`const_iterator`s是不重要的，但是在C++98中，有一个改动看起来是合理的，但是仍然是不正确的：

```cpp
typedef std::vector<int>::iterator IterT;    // typetypedef
std::vector<int>::const_iterator ConstIterT; // defs
std::vector<int> values;
…
ConstIterT ci =
std::find(static_cast<ConstIterT>(values.begin()), // cast
static_cast<ConstIterT>(values.end()), 1983);      // cast
values.insert(static_cast<IterT>(ci), 1998);       // 可能无法编译
                                                   // 参考后续解释
```

`typedef`并不是必须的，当然，这会使得代码更加容易编写。（如果你想知道为什么使用`typedef`而不是使用规则9中建议使用的别名声明，这是因为这个例子是C++98的代码，别名声明的特性是C++11的。）

在`std::find`中的强制类型转换是因为`values`是在C++98中是非`const`的容器，但是并没有比较好的办法可以从一个非`const`容器中得到一个`const_iterator`。强制类型转换并非必要的，因为可以从其他的办法中得到`const_iterator`（比如，可以绑定`values`到一个`const`的引用变量，然后使用这个变量代替代码中的`values`），但是不管使用哪种方式，从一个非`const`容器中得到一个`const_iterator`牵涉到太多。

一旦使用了`const_iterator`，麻烦的事情会更多，因为在C++98中，插入或者删除元素的定位只能使用`iterator`，`const_iterator`是不行的。这就是为什么在上面的代码中，我把`const_iterator`（从`std::find`中小心翼翼的拿到的）有转换成了`iterator`：`insert`给一个`const_iterator`会编译不过。

老实说，我上面展示的代码可能就编译不过，这是因为并没有合适的从`const_iterator`到`interator`的转换，甚至是使用`static_cast`也不行。甚至最暴力的`reinterpret_cast`也不成。（这不是C++98的限制，同时C++11也同样如此。`const_iterator`转换不成`iterator`，不管看似有多么合理。）还有一些方法可以生成类似`const_iterator`行为的`iterator`，但是它们都不是很明显，也不通用，本书中就不讨论了。除此之外，我希望我所表达的观点已经明确：`const_iterator`在C++98中非常麻烦事，是万恶之源。那时候，开发者在必要的地方并不使用`const_iterator`，在C++98中`const_iterator`是非常不实用的。

所有的一切在C++11中发生了变化。现在`const_iterator`既容易获得也容易使用。容器中成员函数`cbegin`和`cend`可以产生`const_iterator`，甚至非`const`的容器也可以这样做，STL成员函数通常使用`const_iterator`来进行定位（也就是说，插入和删除insert and erase）。修订原来的C++98的代码使用C++11的`const_iterator`替换原来的`iterator`是非常的简单的事情：

```cpp
std::vector<int> values; // 和之前一样
…
auto it = // use cbegin
std::find(values.cbegin(),values.cend(), 1983); // and cend
values.insert(it, 1998);
```

现在代码使用`const_iterator`非常的实用！

在C++11中只有一种使用`const_iterator`的短处就是在编写最大化泛型库的代码的时候。代码需要考虑一些容器或者类似于容器的数据结构提供`begin`和`end`（加上cbegin, cend, rbegin等等）作为非成员函数而不是成员函数。例如这种情况针对于内建的数组，和一些第三方库中提供一些接口给自由无约束的函数来使用。最大化泛型代码使用非成员函数而不是使用成员函数的版本。