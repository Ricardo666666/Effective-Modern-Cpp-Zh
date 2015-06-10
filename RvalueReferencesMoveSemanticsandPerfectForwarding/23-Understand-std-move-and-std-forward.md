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

std::forward的情况和std::move类似，但是和std::move__无条件地__将它的参数转化为rvalue不同,std::forward在特定的条件下才会执行转化。std::forward是一个__有条件__的转化。为了理解它何时转化何时不转化，我们来回想一下std::forward的典型的使用场景。最常见的场景是：一个函数模板(function template)接受一个universal reference参数，将它传递给另外一个函数(作参数):

```cpp
void process(const Widget& lvalArg);	//process lvalues
void process(Widget&& rvalArg);	//process rvalues

template<typename T>
void logAndProcess(T&& param)		//template that passes
										//param to process
{
	auto now = std::chrono::system_clock::now(); //get current time
	makeLogEntry("Calling 'process'", now);
	process(std::forward<T>(param));
}
```

请看下面对logAndProcess的两个调用，一个使用的lvalue,另一个使用的rvalue:

```cpp
Widget w;
logAndProcess(w);	//call with lvalue
logAndProcess(std::move(w));	//call with rvalue
```

在logAndProcess的实现中，参数param被传递给了函数process.process按照参数类型是lvalue或者rvalue都做了重载。当我们用lvalue调用logAndProcess时，我们自然地期望：forward给process的也是一个lvalue,当我们用rvalue来调用logAndProcess时，我们希望process的rvalue重载版本被调用。

但是就像所有函数的参数一样，param可能是一个lvalue.logAndProcess内的每一个对process的调用因此想要调用process的lvalue重载版本。为了让以上代码的行为表现正确，我们需要一个机制，param转化为rvalue当且仅当：传递给logAndProcess的用来初始化param的参数必须是一个rvalue.这正是std::forward做的事情。这就是为什么std::forward被称作是一个__条件__转化(conditional cast)：当参数被rvalue初始化时，才将参数转化为rvalue.

你可能想知道std::forward怎么知道它的参数是否被一个rvalue初始化。比如说，在以上的代码中，std::forward怎么知道param被一个lvalue或者rvalue初始化？答案很简单，这个信息蕴涵在logAndProcess的模板参数T中。这个参数传递给了std::forward，然后std::forward来从中解码出此信息。欲知详情，请参考Item 28。

std::move和std::forward都可以归之为cast.唯一的一点不同是，std::move总是在执行casts，而std::forward是在某些条件满足时才做。你可能觉得我们不用std::move,只使用std::forward会不会好一些。从一个纯粹是技术的角度来说，答案是肯定的：std::forward是可以都做了，std::move不是必须的。当然，可以说这两个函数都不是必须的，因为我们可以在任何地方都直接写cast代码，但是我希望我们在此达成共识：这样做很恶心。

std::move的魅力在于:方便，减少了错误的概率，而且更加简洁。举个栗子，有这样的一个class,我们想要跟踪，它的move构造函数被调用了多少次，我们这次需要的是一个static的counter，它在每次move构造函数被调用时递增。假设该class还有一个std::string类型的非静态成员，下面是一个实现move constructor(使用std::move)的常见的例子：

```cpp
class Widget{
public:
	Widget(Widget&& rhs)
	: s(std::move(rhs.s))
	{ ++moveCtorCalls; }
	...
private:
	static std::size_t moveCtorCalls;
	std::string s;	
}
```
如果要使用std::forward来实现同样的行为，代码像下面这样写：

```cpp
class Widget{
public:
	Widget(Widget&& rhs)				//unconventional,
	: s(std::forward<std::string>(rhs.s)) //undesirable
	{ ++moveCtorCalls; }				//implementation
	
	...
}
```
请注意到：首先，std::move只需要一个函数参数(rhs.s), std::forward不只需要一个函数参数(rhs.s),还需要一个模板类型参数(std::string).然后,注意到我们传递给std::forward的类型是非引用类型(non-reference)，因为这就意味着传递的那个参数是一个rvalue（请看Item 28）。综上，这就意味着std::move比std::forward用起来更方便(至少少敲了不少字),免去了让我们传递一个表示函数参数是否是一个rvalue的类型参数。消除了传递错误类型(比如说,传一个std::string&,可以导致数据成员s被拷贝构造，而不是想要的move构造)的可能性。

更重要的是，std::move的使用表明了对rvalue的无条件的转换，然而，当std::forward只对被绑定了rvalue的reference进行转换。这是两个非常不同的行为。std::move就是为了move操作而生，而std::forward,就是将一个对象转发(或者说传递)给另外一个函数，同时保留此对象的左值性或右值性(lvalueness or rvalueness)。所以我们需要这两个不同的函数(并且是不同的函数名字)来区分这两个操作。

|要记住的东西|
|:--------- |
|std::move执行一个无条件的对rvalue的转化。对于它自己本身来说，它不会move任何东西|
|std::forward在参数被绑定为rvalue的情况下才会将它转化为rvalue|
|std::move和std::forward在runtime时啥都不做|