#Item 24: Distinguish universal references from rvalue references.

大多数人说真相可以让我们感到自由，但是在某些情况下，一个巧妙的谎言也可以让人觉得非常轻松。这个Item就是要编制一个“谎言”。因为我们是在和软件打交道。所以我们避开“谎言”这个词：我们是在编制一种“抽象”的意境。

为了声明一个类型T的右值引用，你写下了T&&。下面的假设看起来合理：你在代码中看到了一个"T&&"时，你看到的就是一个右值引用。但是，它可没有想象中那么简单：

```cpp
void f(Widget&& param);			//rvalue reference
Widget&& var1 = Widget();		//rvalue reference
auto&& var2 = var1;				//not rvalue reference

template<typename T>
void f(std::vector<T>&& param)		//rvalue reference

template<typename T>
void f(T&& param);				//not rvalue reference
```

实际上，“T&&”有两个不同的意思。首先,当然是作为rvalue reference,这样的引用表现起来和你预期一致：只和rvalue做绑定，它们存在的意义就是表示出可以从中move from的对象。

“T&&”的另外一个含义是：既可以是rvalue reference也可以是lvalue reference。这样的references在代码中看起来像是rvalue reference（即"T&&"）,但是它们_也_可以表现得就像他们是lvalue refernces（即"T&"）那样.它们的dual nature允许他们既可以绑定在rvalues(like rvalue references)也可以绑定在lvalues(like lvalue references)上。进一步来说，它们可以绑定到const或者non-const,volatile或者non-volatile，甚至是const + volatile对象上面。它们几乎可以绑定到任何东西上面。为了对得起它的全能，我决定给它们起个名字：universal reference.(Item25将会解释universal references总是可以将std::forward应用在它们之上，本书出版之时，C++委员会的一些人开始将universal references称之为forward references).

两种上下文中会出现universal references。最普通的一种是function template parameters，就像上面的代码所描述的例子那样:

```cpp
template<typename T>
void f(T&& param); 			//param is a universal reference
```
第二种context就是auto的声明方式，如下所示：

```cpp
auto&& var2 = var1;			//var2 is a universal reference
```

这两种context的共同点是:都有type deduction的存在。在template  fucntion f中,参数param的类型是被deduce出来的，在var2的声明中，var2的类型也是被deduce出来的。和接下来的例子(也可以和上面的栗子一块儿比)对比我们会发现，下面栗子是不存在type deduction的。如果你看到"T&&",却没有看到type deduction.那么你看到的就是一个rvalue reference:


```cpp
void f(Widget&& param);			//no type deduction
									//param is an rvalue reference
									
Widget&& var1 = Widget();		//no type deduction
									//var1 is an rvalue reference
```

因为universal references是references，所以它们必须被初始化。universal reference的initializer决定了它表达的是rvalue reference或者lvalue reference。如果initializer是rvalue，那么universal reference对应的是rvalue reference.如果initializer是lvalue,那么universal reference对应的就是lvalue reference.对于身为函数参数的universal reference，initializer在call site（调用处）被提供：

```cpp
template<typename T>
void f(T&& param);		//param is a universal reference
									
Widget w;
f(w);					//lvalue passed to f;param's type is Widget&(i.e., an lvalue reference)
f(std::move(w));		//rvalue passed to f;param's type is Widget&&(i.e., an rvalue reference)
```
对universal的reference来说，type deduction是必须的，但还是不够，它要求的格式也很严格，必须是"T&&".再看下我们之前写过的栗子：

```cpp
template<typename T>
void f(std::vector<T>&& param);	//param is an rvalue reference
```

当f被调用时，类型T会被deduce（除非调用者显式的指明类型，这种边缘情况我们不予考虑）。param声明的格式不是T&&，而是std::vector<T>&&.这就说明它不是universal reference,而是一个rvalue reference.如果你传一个lvalue给f，那么编译器肯定就不高兴了。

```cpp
std::vector<int> v;
f(v);				//error! can't bind lvalue to rvalue reference
```

即使一个最简单前缀const.也可以把一个reference成为universal reference的可能抹杀：


```cpp
template<typename T>
void f(const T&& param);		//param is an rvalue reference
```
如果你在一个template里面，并且看到了T&&这样的格式，你可能就会假设它就是一个universal reference.但是并非如此，因为还差一个必要的条件：type deduction.在template里面可不保证一定有type deduction.看个例子，std::vector里面的push_back方法。

```cpp
template<class T, class Allocator = allocator<T>>
class vector{
public:
	void push_back(T&& x);
	...
}
```

以上便是只有T&&格式却没有type deduction的例子，push_back的存在依赖于一个被instantiation的vector.用于instantiation的type就完全决定了push_back的函数声明。也就是说

```cpp
std::vector<Widget> v;
```

使得std::vector的template被instantiated成为如下格式:

```cpp
class vector<Widget, allocator<Widget>>{
public:
	void push_back(Widget&& x);	//rvalue reference
	...
};
```
如你所见，push_back没有用到type deduction.所以这个vector<T>的$push_back$（有两个overload的$push_back$）所接受的参数类型是rvalue-reference-to-T.

与之相反，std::vector中概念上相近的$emplace_back$函数确实用到了type deduction:

```cpp
template<class T,class Allocator=allocator<T>>

class vector{
public:
	template <class... Args>
	void emplace_back(Args&&... args);
	...
};
```

type parameter```Args```独立于vector的type parameter ```T```,所以每次调用```emplace_back```的时候，```Args```就要被deduce一次。（实际上，```Args```是一个parameter pack.并不是type parameter.但是为了讨论的方便，我们姑且称之为type parameter）。

我之前说universal reference的格式必须是```T&&```, 事实上，emplace_back的type parameter名字命名为Args，但这不影响args是一个universal reference,管它叫做T还是叫做Args呢，没啥区别。举个例子，下面的template接受的参数就是universal reference.一是因为格式是"type&&",二是因为param的type会被deduce(再一次提一下，除非caller显示的指明了type这种边角情况).

```cpp
template<typename MyTemplateType>			//param is a 
void someFunc(MyTemplateType&& param); //universal reference
```
我之前提到过auto变量可以是universal references.更准确的说，声明为auto&&的变量就是universal references.因为类型推导发生并且它们也有正确的格式("T&&").auto类型的universal references并不想上面说的那种用来做function template parameters的universal references那么常见，在最近的C++ 11和C++ 14中，它们变得非常活跃。C++ 14中的lambda expression允许声明auto&&的parameters.举个栗子，如果你想写一个C++ 14的lambda来记录任意函数调用花费的时间，你可以这么写:

```cpp
auto timeFuncInvocation = 
	[](auto&& func, auto&&... params)		//C++ 14
	{
		start timer;
		std::forward<decltype(func)>(func)(
			std::forward<decltype(params)>(params)... //invoke func on params
		);
		stop timer and record elapsed time
	}	
```

如果你对于"std::forward<decltype(blah blah blah)"的代码的反应是“这是什么鬼!？”，这只是意味着你还没看过Item33，不要为此担心。
func是一个可以绑定到任何callable object(lvaue或者rvalue)的universal reference.args是0个或多个universal references(即a universal reference parameter pack)，它可以绑定到任意type，任意数量的objects.所以，多亏了auto universal reference,timeFuncInvocation可以记录pretty much any的函数调用(any和pretty much any的区别，请看Item 30).


本Item中universal reference的基础，其实是一个谎言，呃，或者说是一种"abstraction".隐藏的事实被称之为_reference collapsing_,Item 28会讲述。但是该事实并不会降低它的用途。rvalue references以及universal references会帮助你更准确第阅读source code(“Does that T&& I’m looking at bind to rvalues only or to everything?”),和同事沟通时避免歧义(“I’m using a universal reference here, not an rvalue reference...”),它也会使得你理解Item 25和Item 26,这两条都依赖于此区别。所以，接受理解这个abstraction吧。牛顿三大定律(abstraction)比爱因斯坦的相对论(truth)一样有用且更容易应用，universal reference的概念也可以是你理解reference collapsing 的细节。


|要记住的东西|
|:--------- |
|如果一个函数的template parameter有着T&&的格式，且有一个deduce type T.或者一个对象被生命为auto&&,那么这个parameter或者object就是一个universal reference.|
|如果type的声明的格式不完全是type&&,或者type deduction没有发生，那么type&&表示的是一个rvalue reference.|
|universal reference如果被rvalue初始化，它就是rvalue reference.如果被lvalue初始化，他就是lvaue reference.|