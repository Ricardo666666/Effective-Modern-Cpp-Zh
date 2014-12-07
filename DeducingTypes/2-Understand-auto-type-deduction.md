条款二：理解`auto`类型推导
=========================

如果你已经阅读了条款1关于模板相关的类型推导，你就已经知道了机会所有关于`auto`的类型推导，因为除了一个例外，`auto`类型推导就是模板类型推导。但是它怎么就会是模板类型推导呢？模板类型推导涉及模板和函数以及参数，但是`auto`和上面的这些没有任何的关系。

这是对的，但是没有关系。模板类型推导和`auto`类型推导是有一个直接的映射。有一个书面上的从一种情况转换成另外一种情况的算法。

在条款1，模板类型推导是使用下面的通用模板函数来解释的：

```cpp
template<typename T>
void f(ParamType param);
```

在这里通常调用：

```cpp
f(expr);                    // 使用一些表达式来当做调用f的参数
```

在调用`f`的地方，编译器使用`expr`来推导`T`和`ParamType`的类型。

当一个变量被声明为`auto`，`auto`相当于模板中的`T`，而对变量做的相关的类型限定就像`ParamType`。这用代码说明比直接解释更加容易理解，所以看下面的这个例子：

```cpp
auto x = 27;
```

这里，对`x`的类型定义就仅仅是`auto`本身。从另一方面，在这个声明中：

```cpp
const auto cx = x;
```

类型被声明成`const auto`，在这儿：

```cpp
const auto& rx = x;
```

类型被声明称`const auto&`。在这些例子中推导`x`，`cx`，`rx`的类型的时候，编译器处理每个声明的时候就和处理对应的表达式初始化的模板：

```cpp
template<typename T>                // 推导x的类型的
void func_for_x(T param);           // 概念上的模板

func_for_x(27);                     // 概念上的调用：
                                    // param的类型就是x的类型

template<typename T>
void func_for_cx(const T param);    // 推导cx的概念上的模板

func_for_cx(x);                     // 概念调用：param的推导类型就是cx的类型

template<typename T>
void func_for_rx(const T& param);   // 推导rx概念上的模板

func_for_rx(x);                     // 概念调用：param的推导类型就是rx的类型
```

正如我所说，对`auto`的类型推导只存在一种情况的例外（这个后面就会讨论），其他的就和模板类型推导完全一样了。