条款9：优先使用声明别名而不是`typedef`
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

`C++11`给你提供了工具来完成这类转换的工作，表现的形式是`type traits`,它是`<type_traits>`中的一个模板的分类工具。在这个头文件中有数十个类型特征，但是并不是都可以提供类型转换，不提供转换的也提供了意料之中的接口。给定一个你想竞选类型转换的类型`T`，得到的类型是`std::transformation<T>::type`。例如：
```cpp
    std::remove_const<T>::type                 // 从 const T 得到 T
    std::remove_reference<T>::type             // 从 T& 或 T&& 得到 T
    std::add_lvalue_reference<T>::type         // 从 T 得到 T&
```
注释仅仅总结了这些转换干了什么，因此不需要太咬文嚼字。在一个项目中使用它们之前，我知道你会参考准确的技术规范。

无论如何，我在这里不是只想给你大致介绍一下类型特征。反而是因为注意到，类型转换总是以`::type`作为每次使用的结尾。当你对一个模板中的类型参数（你在实际代码中会经常用到）使用它们时，你必须在每次使用前冠以`typename`。这其中的原因是`C++11`的类型特征是通过内嵌`typedef`到一个模板化的`struct`来实现的。就是这样的，他们就是通过使用类型同义技术来实现的，就是我一直在说服你远不如模板别名的那个技术。

这是一个历史遗留问题，但是我们略过不表（我打赌，这个原因真的很枯燥）。因为标准委员会姗姗来迟地意识到模板别名是一个更好的方式，对于`C++11`的类型转换，委员会使这些模板也成为`C++14`的一部分。别名有一个统一的形式：对于`C++11`中的每个类型转换`std::transformation<T>::type`，有一个对应的`C++14`的模板别名`std::transformation_t`。用例子来说明我的意思：
```cpp
	std::remove_const<T>::type                  // C++11: const T -> T
	std::remove_const_t<T>                      // 等价的C++14

	std::remove_reference<T>::type              // C++11: T&/T&& -> T
	std::remove_reference_t<T>                  // 等价的C++14

	std::add_lvalue_reference<T>::type          // C++11: T -> T&
	std::add_lvalue_reference_t<T>              // 等价的C++14
```
`C++11`的结构在`C++14`中依然有效，但是我不知道你还有什么理由再用他们。即便你不熟悉`C++14`，自己写一个模板别名也是小儿科。仅仅`C++11`的语言特性被要求，孩子们甚至都可以模拟一个模式，对吗？如果你碰巧有一份`C++14`标准的电子拷贝，这依然很简单，因为需要做的即使一些复制和粘贴操作。在这里，我给你开个头：
```cpp
	template<class T>
	using remove_const_t = typename remove_const<T>::type;

	template<class T>
	using remove_reference_t = typename remove_reference<T>::type;

	template<class T>
	using add_lvalue_reference_t = 
	  typename add_lvalue_reference<T>::type;
```
看到没有？不能再简单了。

|要记住的东西|
|:--------- |
|`typedef`不支持模板化，但是别名声明支持|
|模板别名避免了`::type`后缀，在模板中，`typedef`还经常要求使用`typename`前缀|
|`C++14`为`C++11`中的类型特征转换提供了模板别名|