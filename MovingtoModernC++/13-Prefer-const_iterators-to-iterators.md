条款13：优先使用const_iterator而不是iterator
===============

`const_iterator`在STL中等价于指向`const`的指针。被指向的数值是不能被修改的。标准的做法是应该使用`const`的迭代器的地方，也就是尽可能的在没有必要修改指针所指向的内容的地方使用`const_iterator`。

这对于C++98和C++11是正确，但是在C++98中，`const_iterator`s只有部分的支持。一旦有一个这样的迭代器，创建它们并非易事，使用也会受限。举一个例子，假如你希望从`vector<int>`搜索第一次出现的1983(这一年"C++"替换"C + 类"而作为一个语言的名字)，然后在搜到的位置插入数值1998(这一年第一个ISO C++标准被接受)。如果在vector中并不存在1983，插入操作的位置应该是vector的末尾。在C++98中使用`iterator`，这会非常容易：

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