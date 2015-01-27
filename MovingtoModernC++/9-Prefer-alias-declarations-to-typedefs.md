条款9：优先使用声明别名而不是'typedef'
=========================
我有信心说，大家都同意使用`STL`容器是个好的想法，并且我希望，条款18可以说服你使用`std::unique_ptr`也是个好想法，但是我想绝对我们中间没有人喜欢写像这样`std::unique_ptr<std::unordered_map<std::string, std::string>>`的代码多余一次。这仅仅是考虑到这样的代码会增加得上“键盘手”的风险。

为了避免这样的医疗悲剧，推荐使用一个`typedef`:
```cpp
	typedef
	  std::unique_ptr<std::unordered_map<std::string, std::string>>
	  UPtrMapSS;
```
但是`typedef`家族是有如此浓厚的`C++98`气息。他们的确可以在`C++11`下工作，但是`C++11`也提供了别名声明（`alias declarations`）：
```cpp
	using UptrMapSS = 
	  std::unique_ptr<std::unordered_map<std::string, std::string>>;
```
考虑到`typedef`和别名声明具有完全一样的意义，推荐其中一个而排斥另外一个的坚实技术原因是容易令人质疑的。这是合理的。

技术原因当让存在，但是在我提到之前。我想说的是，很多人发现使用别名声明可以使涉及到函数指针的类型的声明变得容易理解：
```cpp
	// `FP`等价于一个函数指针，这个函数的参数是一个`int`类型和`
	// std::string`常量类型，没有返回值
	typedef void (*FP)(int, const std::string&);      // typedef

	// 同上
	using FP = void (*)(int, const std::string&);     // 别名声明
```
当然，上面任何形式都不是特别让人容易下咽，并且很少有人会花费大量的时间在一个函数指针类型的标识符上，所以这很难当做选择别名声明而不是`typedef`的不可抗拒的原因。

但是，一个不可抗拒的原因是真实存在的：模板。尤其是别名声明有可能是模板化的（这种情况下，它们被称为别名模板（`alias template`）），然而`typedef`只能说句“臣妾做不到”。这给`C++11`程序员提供了一个明确的机制来表达在`C++98`中需要黑客式的将`typedef`嵌入在模板化的`struct`中才能完成的东西。举个栗子，给一个使用个性化的分配器`MyAlloc`的链接表定义一个标识符。使用别名模板，这就是小菜一碟：
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
