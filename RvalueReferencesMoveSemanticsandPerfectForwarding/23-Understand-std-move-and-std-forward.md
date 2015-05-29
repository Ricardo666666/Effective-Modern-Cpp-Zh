#Item 23:Understand std::move and std::forward

首先通过了解它们(指std::move和std::forward)不做什么来认识std::move和std::forward是非常有用的。std::move不move任何东西。std::forward也不转发任何东西。在运行时，他们什么都不做。不产生可执行代码，一个比特/Users/shikunfeng/Documents/neteaseWork/timeline_15_05_18/src/main/webapp/tmpl/web2/widget/event2.ftl的代码也不产生。

std::move和std::forward只是执行转换的函数(确切的说应该是函数模板)。std::move无条件的将它的参数转换成一个右值，而std::forward当特定的条件满足时，才会执行它的转换。这就是它们本来的样子.这样的解释产生了一些新问题，但是，基本上，就是这么一回事。

为了让这个故事显得更加具体，下面是C++ 11的std::move的一种实现样例，虽然不能完全符合标准的细节，但也非常相近了。

```cpp
template<typename T>
typename remove_reference<T>::type&&
move(T&& param)
{
	using ReturnType = 						//alias declaration;
	typename remove_reference<T>::type&&;//see Item 9
	return static_cast<ReturnType>(param);
}
```
我为你高亮的两处代码(我做不到啊!--菜b的译者注)。首先是函数的名字move，因为返回的类型非常具有迷惑性，我可不想让你一开始就晕头转向。另外一处是最后的转换，包含了move函数的本质。正如你所看到的，std::move接受了一个对象的引用做参数(准确的来说，应该是一个universal reference.请看Item 24。这个参数的格式是T&& param，但是请不要误解为move接受的参数类型就是右值引用，请继续往下看----菜b译者注)，并且返回指向同一个对象的引用。

函数返回值的"&&"部分表明std::move返回的是一个右值引用。但是呢，正如Item 28条解释的那样，如果T的类型恰好是一个左值引用，T&&的类型就会也会是左值引用。为了阻止这种事情的发生，我们用到了type trait(请看Item 9),在T上面应用std::remove_reference，它的效果就是“去除”T身上的引用，因此保证了"&&"应用到了一个非引用的类型上面。这就确保了std::move真正的返回的是一个右值引用(rvalue reference)，这很重要，因为函数返回的rvalue reference就是右值(rvalue).因此，std::move就做了一件事情：将它的参数转换成了右值(rvalue).

说一句题外话，std::move可以更优雅的在C++14中实现。感谢返回函数类型推导(function return type deduction 请看Item 3),感谢标准库模板别名(alias template)`std::remove_reference_t`(请看Item 9),`std::move`可以这样写：

```cpp
template<typename T>				//C++14; still in
decltype(auto) move(T && param)	//namespace std
{
	using ReturnType = remove_reference_t<T>&&;
	return static_cast<ReturnType>(param);
}
```
看起来舒服多了，不是吗？

因为std::move除了将它的参数转换成右值外什么都不做，所以有人说应该给它换个名字，比如说叫`rvalue_cast`可能会好些。话虽如此，它现在的名字仍然就是`std::move`.所以记住`std::move`做什么不做什么很重要。它只作转换，不做move.

当然了，rvalues是对之执行move的合格候选者，所以对一个对象应用std::move告诉编译器，该对象很合适对之执行move操作，所以std::move的名字就有意义了：标示出那些可以对之执行move的对象。

事实上，rvalues并不总是对之执行move的合格候选者。假设你正在写一个类，它用来表示注释。此类的构造函数接受一个包含注释的std::string做参数，并且将此参数的值拷贝到一个数据成员上.受到Item 41的影响，你声明一个接收by-value参数的构造函数:

```cpp
class Annotation{
	public:
		explicit Annotation(std::string text);//param to be copied,
		...			//so per Item 41, pass by value
};
```

但是Annotation的构造函数只需要读取text的值。并不需要修改它。根据一个历史悠久的传统:能使用const的时候尽量使用。你修改了构造函数的声明，text改为const：

```cpp
class Annotation{
	public:
		explicit Annotation(const std::string text);//param to be copied,
		...			//so per Item 41, pass by value
};
```
为了避免拷贝text到对象成员变量带来拷贝代价。你继续忠实于Item 41的建议，对text应用std::move,因此产生出一个rvalue:

```cpp
class Annotation{
	public:
		explicit Annotation(const std::string text)
		: value(std::move(text))//"move" text into value; this code
		{...}			//doesn't do what it seems to!
		...
		
	private:
		std::string value;
};
```
这样的代码通过了编译，链接，最后运行。而且把成员变量value设置成text的值。代码跟你想象中的完美情况唯一不同的一点是，它没有对text执行move到value，而是拷贝了text的值到value.text确实被std::move转化成了rvalue，但是text被声明为const std::string.所以在cast之前，text是一个const std::string类型的lvalue.cast的结果是一个const std::string的rvalue,但是自始至终，const的性质一直没变。

代码运行时，编译器要选择一个std::string的构造函数来调用。有以下两种可能:

```cpp
class string{				//std::string is actually a 
	public:				//typedef for std::basic_string<char>
		...
		string(const string& rhs);		//copy ctor
		string(string&& rhs);			//move ctor
};	
```

在Annotation的构造函数的成员初始化列表(member initialization list),`std::move(text)`的结果是const std::string的rvalue.这个rvalue不能传递给std::string的move构造函数,因为move构造函数接收的是非const的std::string的rvalue引用。然而，因为lvalue-reference-to-const的参数类型可以被const rvalue匹配上，所以rvalue可以被传递给拷贝构造函数.因此即使text被转换成了rvalue，上文中的成员初始化仍调用了std::string的拷贝构造函数!这样的行为对于保持const的正确性是必须的。从一个对象里move出一个值通常会改变这个对象，所以语言不允许将const对象传递给像move constructor这样的会改变次对象的函数。

从本例中你可以学到两点。首先，如果你想对这些对象执行move操作，就不要把它们声明为const.对const对象的move请求通常会悄悄的执行到copy操作上。