条款9：优先使用声明别名而不是'typedef'
=========================
我有信心说，大家都同意使用`STL`容器是个好的想法，并且我希望，条款18可以说服你使用`std::unique_ptr`也是个好想法，但是我想绝对我们中间没有人喜欢写像这样`std::unique_ptr<std::unordered_map<std::string, std::string>>`的代码多于一次。这仅仅是考虑到这样的代码会增加得上“键盘手”的风险。

为了避免这样的医疗悲剧，推荐使用一个`typedef`:
```cpp
	typedef
	  std::unique_ptr<std::unordered_map<std::string, std::string>>
	  UPtrMapSS;
```
但是`typedef`家族是有如此浓厚的`C++98`气息。他们的确可以在`C++11`下工作，但是`C++11`也提供了声明别名（`alias declarations`）：
```cpp
	using UptrMapSS = 
	  std::unique_ptr<std::unordered_map<std::string, std::string>>;
```
考虑到`typedef`和声明别名具有完全一样的意义，推荐其中一个而排斥另外一个的坚实技术原因是容易令人质疑的。这样的质疑是合理的。

技术原因当然存在，但是在我提到之前。我想说的是，很多人发现使用声明别名可以使涉及到函数指针的类型的声明变得容易理解：
```cpp
	// FP等价于一个函数指针，这个函数的参数是一个int类型和
	// std::string常量类型，没有返回值
	typedef void (*FP)(int, const std::string&);      // typedef

	// 同上
	using FP = void (*)(int, const std::string&);     // 声明别名
```
当然，上面任何形式都不是特别让人容易下咽，并且很少有人会花费大量的时间在一个函数指针类型的标识符上，所以这很难当做选择声明别名而不是`typedef`的不可抗拒的原因。

但是，一个不可抗拒的原因是真实存在的：模板。尤其是声明别名有可能是模板化的（这种情况下，它们被称为模板别名（`alias template`）），然而`typedef`这是只能说句“臣妾做不到”。模板别名给`C++11`程序员提供了一个明确的机制来表达在`C++98`中需要黑客式的将`typedef`嵌入在模板化的`struct`中才能完成的东西。举个栗子，给一个使用个性化的分配器`MyAlloc`的链接表定义一个标识符。使用别名模板，这就是小菜一碟：
```cpp
	template<typname T>                             // MyAllocList<T>
	using MyAllocList = std::list<T, MyAlloc<T>>;   // 等同于
                                                    // std::list<T,
                                                    //   MyAlloc<T>>

	MyAllocList<Widget> lw;                         // 终端代码 
```
使用`typedef`，你不得不从草稿图开始去做一个蛋糕：
```cpp
	template<typename T>                            // MyAllocList<T>::type
	struct MyAllocList {                            // 等同于
	  typedef std::list<T, MyAlloc<T>> type;        // std::list<T, 
    };                                              // MyAlloc<T>>

    MyAllocList<Widget>::type lw;                   // 终端代码
```
如果你想在一个模板中使用`typedef`来完成创建一个节点类型可以被模板参数指定的链接表的任务，你必须在`typedef`名称之前使用`typename`：
```cpp
	template<typename T>                            // Widget<T> 包含
	class Widget{                                   // 一个 MyAloocList<T>
	private:                                        // 作为一个数据成员
	  typename MyAllocList<T>::type list;
	  ...
    };
```
此处，`MyAllocList<T>::type`表示一个依赖于模板类型参数`T`的类型，因此`MyAllocList<T>::type`是一个依赖类型（`dependent type`），`C++`中许多令人喜爱的原则中的一个就是在依赖类型的名称之前必须冠以`typename`。

如果`MyAllocList`被定义为一个声明别名，就不需要使用`typename`（就像笨重的`::type`后缀）：
```cpp
	template<typname T>                             
	using MyAllocList = std::list<T, MyAlloc<T>>;   // 和以前一样

	template<typename T>
	class Widget {
	private:
	  MyAllocList<T> list;                         // 没有typename
	  ...                                          // 没有::type
	};
```
对你来说，`MyAllocList<T>`（使用模板别名）看上去依赖于模板参数`T`，正如`MyAllocList<T>::type`（使用内嵌的`typdef`）一样，但是你不是编译器。当编译器处理`Widget`遇到`MyAllocList<T>`（使用模板别名），编译器知道`MyAllocList<T>`是一个类型名称，因为`MyAllocList`是一个模板别名：它必须是一个类型。`MyAllocList<T>`因此是一个非依赖类型（`non-dependent type`），指定符`typename`是不需要和不允许的。

另一方面，当编译器在`Widget`模板中遇到`MyAllocList<T>`（使用内嵌的`typename`）时，编译器并不知道它是一个类型名，因为有可能存在一个特殊化的`MyAllocList`，只是编译器还没有扫描到，在这个特殊化的`MyAllocList`中`MyAllocList<T>::type`表示的并不是一个类型。这听上去挺疯狂的，但是不要因为这种可能性而怪罪于编译器。是人类有可能会写出这样的代码。

例如，一些被误导的鬼魂可能会杂糅出像这样代码：
```cpp
	class Wine {...};

	template<>                                 // 当T时Wine时
	class MyAllocList<Wine>{                   // MyAllocList 是特殊化的
	private:
	  enum class WineType                      // 关于枚举类参考条款10
	  { White, Red, Rose };

	  WineType type;                           // 在这个类中，type是个数据成员
	  ...
	};
```
正如你看到的，`MyAllocList<Wine>::type`并不是指一个类型。如果`Widget`被使用`Wine`初始化，`Widget`模板中的`MyAllocList<T>::type`指的是一个数据成员，而不是一个类型。在`Wedget`模板中，`MyAllocList<T>::type`是否指的是一个类型忠实地依赖于传入的`T`是什么，这也是编译器坚持要求你在类型前面冠以`typename`的原因。

如果你曾经做过模板元编程（`TMP`），你会强烈地额反对使用模板类型参数并在此基础上修改为其他类型的必要性。例如，给定一个类型`T`，你有可能想剥夺`T`所包含的所有的`const`或引用的修饰符，即你想将`const std::string&`变成`std::string`。你也有可能想给一个类型加上`const`或者将它变成一个左值引用，也就是将`Widget`变成`const Widget`或者`Widget&`。（如果你没有做过`TMP`,这太糟糕了，因为如果你想成为一个真正牛叉的`C++`程序员，你至少需要对`C++`这方面的基本概念足够熟悉。你可以同时看一些TMP的例子，包括我上面提到的类型转换，还有条款23和条款27。）

