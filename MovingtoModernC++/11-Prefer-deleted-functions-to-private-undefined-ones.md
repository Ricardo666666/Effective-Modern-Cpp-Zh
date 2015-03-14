条款11：优先使用delete关键字删除函数而不是private却又不实现的函数
=========================
如果你要给其他开发者提供代码，并且还不想让他们调用特定的函数，你只需要不声明这个函数就可以了。没有函数声明，没有就没有函数可以调用。这是没有问题的。但是有时候`C++`为你声明了一些函数，如果你想阻止客户调用这些函数，就不是那么容易的事了。

这种情况只有对“特殊的成员函数”才会出现，即这个成员函数是需要的时候`C++`自动生成的。条款17详细地讨论了这种函数，但是在这里，我们仅仅考虑复制构造函数和复制赋值操作子。这一节致力于`C++98`中的一般情况，这些情况可能在`C++11`中已经不复存在。在`C++98`中，如果你想压制一个成员函数的使用，这个成员函数通常是复制构造函数，赋值操作子，或者它们两者都包括。

在`C++98`中阻止这类函数被使用的方法是将这些函数声明为`private`，并且不定义它们。例如，在`C++`标准库中，`IO`流的基础是类模板`basic_ios`。所有的输入流和输出流都继承（有可能间接地）与这个类。拷贝输入和输出流是不被期望的，因为不知道应该采取何种行为。比如，一个`istream`对象，表示一系列输入数值的流，一些已经被读入内存，有些可能后续被读入。如果一个输入流被复制，是不是应该将已经读入的数据和将来要读入的数据都复制一下呢？处理这类问题最简单的方法是定义这类问题不存在，`IO`流的复制就是这么做的。

为了使`istream`和`ostream`类不能被复制，`basic_ios`在`C++98`中是如下定义的（包括注释）：
```cpp
	template <class charT, class traits = char_traits<charT> >
	class basic_ios :public ios_base {
	public:
	  ...

	  private:
	    basic_ios(const basic_ios& );                  // 没有定义
	    basic_ios& operator(const basic_ios&);         // 没有定义
    };
```
将这些函数声明为私有来阻止客户调用他们。故意不定义它们是因为，如果有函数访问这些函数（通过成员函数或者友好类）在链接的时候会导致没有定义而触发的错误。

在`C++11`中，有一个更好的方法可以基本上实现同样的功能：用`= delete`标识拷贝复制函数和拷贝赋值函数为删除的函数`deleted functions`。在`C++11`中`basic_ios`被定义为：
```cpp
	template <class charT, class traits = char_traits<charT> >
	class basic_ios : public ios_base {
	public:
	  ...
	  basic_ios(const basic_ios& ) = delete;
	  basic_ios& operator=(const basic_ios&) = delete;
	  ...
    };
```
删除的函数和声明为私有函数的区别看上去只是时尚一些，但是区别比你想象的要多。删除的函数不能通过任何方式被使用，即便是其他成员函数或者友好函数试图复制`basic_ios`对象的时候也会导致编译失败。这是对`C++98`中的行为的升级，因为在`C++98`中直到链接的时候才会诊断出这个错误。

方便起见，删除函数被声明为公有的，而不是私有的。这样设计的原因是，当客户端程序尝试使用一个成员函数的时候，`C++`会在检查删除状态之前检查可访问权限。当客户端代码尝试访问一个删除的私有函数时，一些编译器仅仅会警报该函数为私有，尽管这里函数的可访问性并不本质上影响它是否可以被使用。当把私有未定义的函数改为对应的删除函数时，牢记这一点是很有意义的，因为使这个函数为公有的可以产生更易读的错误信息。

删除函数一个重要的优势是任何函数都可以是删除的，然而仅有成员函数才可以是私有的。举个例子，加入我们有个非成员函数，以一个整数位参数，然后返回这个参数是不是幸运数字：
```cpp
	bool isLucky(int number);
```
`C++`继承于`C`意味着，很多其他类型被隐式的转换为`int`类型，但是有些调用可以编译但是没有任何意义：
```cpp
	if(isLucky('a')) ...                  // a 是否是幸运数字？

	if(isLucky(ture)) ...                 // 返回true?

	if(isLucky(3.5)) ...                  // 我们是否应该在检查它是否幸运之前裁剪为3？
```
如果幸运数字一定要是一个整数，我们希望能到阻止上面那种形式的调用。

完成这个任务的一个方法是为想被排除出去的类型的重载函数声明为删除的：
```cpp
	bool isLucky(int number);           // 原本的函数

	bool isLucky(char) = delete;        // 拒绝char类型

	bool isLucky(bool) = delete;        // 拒绝bool类型

	bool isLucky(double) = delete;      // 拒绝double和float类型
```
（对`double`的重载的注释写到：`double`和`float`类型都讲被拒绝可能会令你感到吃惊，当时当你回想起来，如果给`float`一个转换为`int`或者`double`的可能性，`C++`总是倾向于转化为`double`的，就不会感到奇怪了。以`float`类型调用`isLucky`总是调用对应的`double`重载，而不是`int`类型的那个重载。结果就是将`double`类型的重载删除将会组织`float`类型的调用编译。）

尽管删除函数不能被使用，但是它们仍然是你程序的一部分。因此，在重载解析的时候仍会将它们考虑进去。这也就是为什么有了上面的那些声明，对`isLucky`不被期望的调用会被拒绝：
```cpp	
	if (isLucky('a')) ...               // 错误！调用删除函数

	if (isLucky(true)) ...              // 错误！

	if (isLucky(3.5f)) ...              // 错误！
```
还有一个删除函数可以完成技巧（而私有成员函数无法完成）是可以阻止那些应该被禁用的模板实现。举个例子，假设你需要使用一个内嵌指针的模板（虽然第4章建议使用智能指针而不是原始的指针）：
```cpp
	template<typename T>
	void processPointer(T* ptr);
```
在指针的家族中，有两个特殊的指针。一个是`void*`指针，因为没有办法对它们解引用，递增或者递减它们等操作。另一个是`char*`指针，因为它们往往表示指向`C`类型的字符串，而不是指向独立字符的指针。这些特殊情况经常需要特殊处理，在`processPointer`模板中，假设对这些特殊的指针合适的处理方式拒绝调用。也就是说，不可能以`void*`或者`char*`为参数调用`processPointer`。

这是很容易强迫实现的。仅仅需要删除这些实现：
```cpp
	template<>
	void processPointer<void>(void*) = delete;

	template<>
	void processPointer<char>(char*) = delete;
```
现在，使用`void*`或者`char*`调用`processPointer`是无效的，使用`const void*`或者`const char*`调用也需要是无效的，因此这些实现也需要被删除：
```cpp
	template<>
	void processPointer<const void>(const void*) = delete;

	template<>
	void processPointer<const char>(const char*) = delete;
```
如果你想更彻底一点，你还要删除对`const volatile void*`和`const volatile char*`的重载，你就可以在其他标准的字符类型的指针`std::wchar_t, std::char16_t`和`std::char32_t`上愉快的工作了。

有趣的是，如果你在一个类内部有一个函数模板，你想通过声明它们为私有来禁止某些实现，但是你通过这种方式做不到，因为赋予一个成员函数模板的某种特殊情况下拥有不同于模板主体的访问权限是不可能。举个例子，如果`processPointer`是`Widget`内部的一个成员函数模板，你想禁止使用`void*`指针的调用，下面是一个`C++98`风格的方法，下面代码依然无法通过编译：
```cpp
	class Widget{
	public:
	  ...
	  template<typename T>
	  void processPointer(T* ptr)
	  { ... }

	private:
	  template<>                                       // 错误！
	  void processPointer<void>(void*)

    };
```
这里的问题是，模板的特殊情况必须要写在命名空间的作用域内，而不是类的作用域内。这个问题对于删除函数是不存在的，因为它们不再需要一个不同的访问权限。它们可以再类的外面被声明为是被删除的（也就是在命名空间的作用域内）：
```cpp
	class Widget{
	public:
	  ...
	  template<typename T>
	  void processPointer(T* ptr)
	  { ... }
	  ...

    };

    template<>                                              // 仍然是公用的，但是已被删除
    void Widget::processPointer<void>(void*) = delete;
```
真相是，`C++98`中声明私有函数但是不定义是想达到`C++11`中删除函数同样效果的尝试。作为一个模仿品，`C++98`的方式并不如它要模仿的东西那么好。它在类的外边和内部都是是无法工作的，当它工作时，知道链接的时候可能又不工作了。所以还是坚持使用删除函数吧。

|要记住的东西|
|:--------- |
|优先使用删除函数而不是私有而不定义的函数|
|任何函数都可以被声明为删除，包括非成员函数和模板实现|
