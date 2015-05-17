#Item 22:当使用Pimpl的时候在实现文件中定义特殊的成员函数

如果你曾经因为程序过多的build次数头疼过，你肯定对于Pimpl(pointer to implementation)做法很熟悉。它的做法是：把对象的成员变量替换为一个指向已经实现类(或者是结构体)的指针.将曾经在主类中的数据成员转移到该实现类中，通过指针来间接的访问这些数据成员。举个列子，假设Widget看起来像是这样：

```cpp
class Widget{			//in header "widget.h"
public:
	Widget();
	...
private:
	std::string name;
	std::vector<double> data:
	Gadget g1,g2,g3;			
	//Gadget is some user-defined  type
}
```

因为Widget的数据成员是std::string, std::vector以及Gadget类型，为了编译Widget,这些类型的头文件必须被包含进来,使用Widget的客户必须#include <string>,<vector>,以及gadget.h。这些头文件增加了使用Widget的客户的编译时间，并且让客户依赖这些头文件的内容。如果一个头文件里的内容发生了改变，使用Widget的客户必须被重新编译。虽然标准的头文件<string>和<vector>不怎么发生变化，但gadget.h的头文件可能经常变化。

应用C++ 98的Pimpl做法，我们将数据成员变量替换为一个原生指针，指向了一个只是声明并没有被定义的结构体

```cpp
class Widget{			//still in header "widget.h"
public:
	Widget();
	~Widget();	//dtor is needed-see below
	...
private:
	struct Impl;		//declare implementation struct
	Impl *pImpl;		//and pointer to it
}
```
Widget已经不再引用std::string,std::vector以及gadget类型，使用Widget的客户就可以不#include这些头文件了。这加快了编译速度，并且Widget的客户也不受到影响。

一种只声明不定义的类型被称作incomplete type.Widget::Impl就是incomplete type。对于incomplete type,我们能做的事情很少，但是我们可以声明一个指向它的指针。Pimpl做法就是利用了这一点。

Pimpl做法的第一步是：声明一个成员变量，它是一个指向incomplete type的指针。第二步是，为包含以前在原始类中的数据成员的对象(本例中的*pImpl)做动态分配内存和回收内存。分配以及回收的代码在实现文件中。本例中，对于Widget而言,这些操作在widget.cpp中进行：

```cpp
#include "widget.h"			//in impl,file "widget.cpp"
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl{
	std::string name;			//definition of Widget::Impl with data members formerly in Widget
	std::vector<double> data;
	Gadget g1,g2,g3;
}

Widget::Widget():pImpl(new Impl)	//allocate data members for this Widget object
{}

Widget::~Widget()	//destroy data members for this object
{
	delete pImpl;
}
```

在上面的代码中，我还是使用了#include指令，表明对于std::string,std::vector以及Gadget头文件的依赖还继续存在。然后，这些依赖从widget.h(被使用Widget的客户所使用，对它们可见)转移到了widget.cpp(只对Widget的实现者可见)中，我已经高亮了(不好意思，在代码中高亮这种语法俺做不到啊---译者注)分配和回收Impl对象的代码。因为需要在Widget析构是，对Impl对象的内存进行回收，所以Widget的析构函数是必须要写的。

但是我给你展示的是C++ 98的代码，散发着上个世纪的上古气息.使用了原生指针，原生的new和原生的delete，全都是原生的啊！本章的内容是围绕着“智能指针大法好,退原生指针保平安”的理念。如果我们想要在一个Widget的构造函数中动态分配一个Widget::Impl对象，并且在Widget析构时，也析构Widget::Impl对象。那么std::unique_ptr(请看Item 18)就是我们想要的最合适的工具。将原生pImpl指针替换为std::unique_ptr。头文件的内容变为这样：

```cpp
class Widget{
public:
	Widget();
	...
private:
	struct Impl;
	std::unique_ptr<Impl> pImpl;//use smart pointer
	//instead of raw pointer
}
```
实现文件的内容则变成了这样：

```cpp
#include "widget.h"			//in "widget.cpp"
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl{
	std::string name;			//as before
	std::vector<double> data;
	Gadget g1,g2,g3;
}

Widget::Widget()
:pImpl(std::make_unique<Impl>())	//per Item 21,create
{}										//std::unique_ptr
										//via std::make_unique
```
你会发现Widget的析构函数不复存在。这是因为我们不需要在析构函数里面写任何代码了。std::unique_ptr在自身销毁时自动析构它指向的区域，所以我们不需要自己回收任何东西。这就是智能指针的一项优点：它们消除了我们需要手动释放资源的麻烦。

但是呢，使用Widget的客户的一句很平凡的用法，就编译出错了啊

```cpp
#include "widget.h"
Widget w; //error
```

你所受到的错误信息内容依赖于你所使用的编译器类型，但是产生的内容大致都是：在incomplete type上使用了sizeof和delete.这些操作在该类型上是禁止的。

Pimpl做法结合std::unique_ptr竟然会产生错误，这很让人震惊啊。因为(1)std::unique_ptr自身标榜支持incomplete type.(2)Pimpl做法是std::unique_ptr众多的使用场景之一。幸运的是，让代码工作起来也是很简单地。我们首先需要理解为啥会出错。


在执行w被析构(如当出作用域时)的代码时，报了错误。在此时，Widget的析构函数被调用。在定义使用std::unique_ptr的Widget的时候，我们并没有声明析构函数，因为我们不需要在Widget的析构函数内写任何代码。依据编译器自动生成特殊成员函数(请看Item 17)的普通规则，编译器为我们生成了一个析构函数。在那个自动生成的析构函数中，编译器插入代码，调用Widget的数据成员pImpl的析构函数。pImpl是一个`std::unique_ptr<Widget::Impl>`,即，一个使用默认deleter的std::unique_ptr.默认deleter是一个函数，对std::unique_ptr里面的原生指针调用delete.然而，在调用delete之前，编译器通常会让默认deleter先使用C++ 11的static_assert来确保原生指针指向的类型不是imcomplete type(staticassert编译时候检查,assert运行时检查---译者注).当编译器生成Widget w的析构函数时，调用的static_assert检查就会失败，导致出现了错误信息。在w被销毁时，这些错误信息才会出现，但是因为与其他的编译器生成的特殊成员函数相同，Widget的析构函数也是inline的。出错指向w被创建的那一行，因为改行创建了w,导致后来(w出作用域时)w被隐性销毁。

为了修复这个问题，你需要确保，在产生销毁`std::unique_ptr<Widget::Impl>`的代码时，Widget::Impl是完整的类型。当它的定义被编译器看到时，它就是完整类型了。而Widget::Impl在widget.cpp中被定义。所以编译成功的关键在于，让编译器只在widget.cpp内，在widget::Impl被定义之后，看到Widget的析构函数体(该函数体就是放置编译器自动生成销毁std::unique_ptr数据成员的代码的地方)。

像那样安排很简单，在widget.h中声明Widget的析构函数，但是不要在其中定义：
```cpp
class Widget{			//as before, in "widget.h"
public:
	Widget();
	~Widget();		//declaration only
	...
private:				//as before
	struct Impl;
	std::unique_ptr<Impl> pImpl;	
}
```
在widget.cpp里面的Widget::Impl定义之后再定义析构函数：


```cpp
#include "widget.h"			//as before in "widget.cpp"
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl{
	std::string name;			//as before definition of
	std::vector<double> data;	//Widget::Impl
	Gadget g1,g2,g3;
}

Widget::Widget()
:pImpl(std::make_unique<Impl>())	//as before

Widget::~Widget(){}	//~Widget definition
```

这样的话就没问题了，增加的代码量也很少。但是如果你想要强调，编译器生成的析构函数会做正确的事情，你声明析构函数的唯一原因是，想要在Widget.cpp中生成它的定义，你可以用“=default”定义析构函数体：


```cpp
Widget::~Widget()=default;		//same effect as above
```

使用Pimpl做法的类是自带支持move语义的候选者，因为编译器生成的move操作正是我们想要的：对潜在的std::unique_ptr上执行move操作。就像Item 17解释的那样，Widget声明了析构函数，编译器就不会自动生成move操作了。所以如果你想要支持move,你必须自己去声明这些函数。鉴于编译器生成的版本就可以胜任，你可能会像下面那样实现：

```cpp
class Widget{			//still in "widget.h"
public:
	Widget();
	~Widget();
	...
	Widget(Widget&& rhs) = default;		//right idea,
	Widget& operator=(Widget&& rhs) = default //wrong code!
	...
private:				//as before
	struct Impl;
	std::unique_ptr<Impl> pImpl;	
};
```

这样的做法会产生和声明一个没有析构函数的class一样，产生同样的问题，产生问题的原因本质也一样。对于编译器生成的move赋值操作符，它对pImpl再赋值之前，需要先销毁它所指向的对象，然而在Widget头文件中，pImpl指向的仍是一个incomplete type.对于编译器生成的move构造函数。问题在于编译器会在move构造函数内抛出异常的事件中，生成析构pImpl的代码，对pImpl析构(destroying pImpl)需要Impl的类型是完整的。

问题一样，解决方法自然也一样：将move操作的定义写在实现文件widget.cpp中：
widget.h:

```cpp
class Widget{			//still in "widget.h"
public:
	Widget();
	~Widget();
	...
	Widget(Widget&& rhs);				//declarations
	Widget& operator=(Widget&& rhs); //only
	...
private:				//as before
	struct Impl;
	std::unique_ptr<Impl> pImpl;	
};
```
widget.cpp

```cpp
#include <string>			//as before,
...								//in "widget.cpp"
sturct:Widget::Impl {...}; //as before

Widget::Widget()				//as before
:pImpl(std::make_unique<Impl>())
{}

Widget::~Widget() = default; //as before

Widget::Widget(Widget&& rhs) = default;
Widget& Widget::operator=(Widget&& rhs) = default:
//definitions
```

Pimpl做法是一种减少class的实现和class的使用之间编译依赖的一种方式，但是，从概念上来讲，这种做法并不改变类的表现方式。原来的Widget类包含了std::string,std::vector以及Gadget数据成员，并且，假设Gadget，像std::string和std::vector那样，可以被拷贝。所以按理说Widget也要支持拷贝操作。我们必须要自己手写这些拷贝函数了，因为(1)对于带有move-only类型(像std::unique_ptr)的类，编译器不会生成拷贝操作的代码.(2)即使生成了，生成的代码只会拷贝std::unique_ptr(即，执行浅拷贝)，而我们想要的是拷贝指针所指向的资源(即，执行深拷贝)。

现在的做法我们已经熟悉了，在头文件中声明这些函数，然后在实现文件中实现这些函数；

widget.h:

```cpp
class Widget{			//still in "widget.h"
public:
	...					//other funcs, as before
	Widget(const Widget& rhs);				//declarations
	Widget& operator=(const Widget& rhs); //only
private:				//as before
	struct Impl;
	std::unique_ptr<Impl> pImpl;	
};
```
widget.cpp

```cpp
#include <string>			//as before,
...								//in "widget.cpp"
sturct:Widget::Impl {...}; //as before

Widget::~Widget() = default; //other funcs,as before

Widget::Widget(const Widget& rhs); //copy ctor
: pImpl(std::make_unique<Impl>(*rhs.pImpl))
{}

Widget& Widget::operator=(const Widget& rhs)//copy operator=
{
	*pImpl = *rhs.pImpl;
	return *this;
}
```

两个函数的实现都比较常见。每种情况下，我们都是从源对象(rhs)到目的对象(*this)，简单的拷贝了Impl的结构体的内容.我们利用了这样的事实：编译器会为Impl生成拷贝操作的代码，这些操作会自动将stuct的内容逐项拷贝，就不需要我们手动来做了。我们因此通过调用Widget::Impl的编译器生成的拷贝操作符来实现了Widget拷贝操作符。在copy构造函数中，注意到我们遵循了Item 21的建议，不直接使用new，而是优先使用了std::make_unique.

在上面的例子中，为了实现Pimpl做法，std::unique_ptr是我们使用的智能指针类型，因为对象内(在Widget内)的pImpl对对应的实现对象(Widget::Impl对象)拥有独占所有权。但是，很有意思的是，当我们对pImpl使用std::shared_ptr来替代std::unique_ptr,我们发现本Item的建议不再适用了。没必要在Widget.h中声明析构函数，没有了用户自己声明的析构函数，编译器会很乐意生成move操作代码，而且生成的代码表现的行为正合我们意。widget.h变得如下所示：

```cpp
class Widget{					//in "widget.h"
public:
	Widget();
	...						//no declarations for dtor
							//or move operations
private:
	struct Impl;						//std::shared_ptr
	std::shared_ptr<Impl> pImpl;	//instead of std::unique_ptr
};
```
`#include widget.h`的客户代码：

```cpp
Widget w1;
auto w2(std::move(w1));			//move-consturct w2

w1 = std::move(w2);				//move-assign w1
```

所有代码都会如我们所愿通过编译，w1会被默认构造，它的值会被move到w2,之后w2的值又被move回w1.最后w1和w2都得到析构(这使得指向的Widget::Impl对象被析构)

对pImpl应用std::unique_ptr和std::shared_ptr的表现行为不同的原因是：它们之间支持自定义的deleter的方式不同。对于std::unique_ptr，deleter的类型是智能指针的一部分，这就使得编译器生成更小的运行时数据结构以及更快的运行时代码成为可能。更好的效率的结果是要求当编译器生成特殊函数(如析构以及move操作)被使用时，std::unique_ptr所指向的类型必须是完整的。对于std::shared_ptr来说，deleter的类型不是智能指针的一部分。虽然会造成比较大的运行时数据结构和慢一些的代码。但是在调用编译器生成的特殊函数时，指向的类型不需要是完整的。

对于Pimpl做法，在`std::unique_ptr`和`std::shared_ptr`的特点之间，其实并没有一个真正的权衡。因为Widget和Widget::Impl之间是独占的拥有关系，`std::unique_ptr`在此项工作中很合适。然而，在一些其他的场景中，共享式拥有关系存在，`std::shared_ptr`才是一个合适的选择，就没必要像依靠`std::unique_ptr`这样的函数定义的做法了。

|要记住的东西|
|:--------- |
|Pimpl做法通过减少类的实现和类的使用之间的编译依赖减少了build次数|
|对于`std::unique_ptr` pImpl指针，在class的头文件中声明这些特殊的成员函数，在class的实现文件中定义它们。即使默认的实现方式(编译器生成的方式)可以胜任也要这么做|
|上述建议适用于`std::unique_ptr`,对`std::shared_ptr`无用|