#Item 19:使用std::shared_ptr来管理共享式的资源
使用垃圾回收机制的程序员指责并且嘲笑C++程序员阻止内存泄露的做法。“你们tmd是原始人!”他们嘲笑道。“你们有没有看过1960年Lisp语言的备忘录？应该用机器来管理资源的生命周期，而不是人类。”C++程序员开始翻白眼了:"你们懂个屁，如果备忘录的内容意味着唯一的资源是内存而且回收资源的时机是不确定性的，那么我们宁可喜欢具有普适性和可预测性的析构函数."但是我们的回应部分是虚张声势。垃圾回收确实非常方便，手动来控制内存管理周期听起来像是用原始工具来做一个记忆性的内存回路。为什么我们不两者兼得呢？做出一个既可以想垃圾回收那样自动，且可以运用到所有资源，具有可预测的回收时机(像析构函数那样)的系统。

`std::shared_ptr`就是C++11为了达到上述目标推出的方式。一个通过`std::shared_ptr`访问的对象被指向它的指针通过共享所有权(shared ownership)方式来管理.没有一个特定的`std::shared_ptr`拥有这个对象。相反，这些指向同一个对象的`std::shared_ptr`相互协作来确保该对象在不需要的时候被析构。当最后一个`std::shared_ptr`不再指向该对象时(例如，因为`std::shared_ptr`被销毁或者指向了其他对象)，`std::shared_ptr`会在此之前摧毁这个对象。就像GC一样，使用者不用担心他们如何管理指向对象的生命周期，而且因为有了析构函数，对象析构的时机是可确定的。

一个`std::shared_ptr`可以通过查询资源的引用计数(reference count)来确定它是不是最后一个指向该资源的指针，引用计数是一个伴随在资源旁的一个值，它记录着有多少个`std::shared_ptr`指向了该资源。`std::shared_ptr`的构造函数会自动递增这个计数，析构函数会自动递减这个计数，而拷贝构造函数可能两者都做(比如，赋值操作`sp1=sp2`,sp1和sp2都是`std::shared_ptr`类型，它们指向了不同的对象，赋值操作使得sp1指向了原来sp2指向的对象。赋值带来的连锁效应使得原来sp1指向的对象的引用计数减1，原来sp2指向的对象的引用计数加1.)如果`std::shared_ptr`在执行减1操作后发现引用计数变成了0，这就说明了已经没有其他的`std::shared_ptr`在指向这个资源了，所以`std::shared_ptr`直接析构了它指向的空间。

引用计数的存在对性能会产生部分影响

 * `std::shared_ptrs`是原生指针的两倍大小，因为它们内部除了包含了一个指向资源的原生指针之外，同时还包含了指向资源的引用计数
 
 * 引用计数的内存必须被动态分配.概念上来说，引用计数会伴随着被指向的对象，但是被指向的对象对此一无所知。因此，他们没有为引用计数准备存储空间。(一个好消息是任何对象，即使是内置类型，都可以被`std::shared_ptr`管理。)Item21解释了用`std::make_shared`来创建`std::shared_ptr`的时候可以避免动态分配的开销，但是有些情况下`std::make_shared`也是不能被使用的。不管如何，引用计数都是存储为动态分配的数据

 * 引用计数的递增或者递减必须是原子的，因为在多线程环境下，会同时存在多个写者和读者。例如，在一个线程中，一个`std::shared_ptr`指向的资源即将被析构(因此递减它所指向资源的引用计数)，同时，在另外一个线程中，一个`std::shared_ptr`指向了同一个对象，它此时正进行拷贝操作(因此要递增同一个引用计数)。原子操作通常要比非原子操作执行的慢，所以尽管引用计数通常只有一个word大小，但是你可假设对它的读写相对来说比较耗时。

当我写到:`std::shared_ptr`构造函数在构造时"通常"会增加它指向的对象的引用计数时，你是不是很好奇？创建一个新的指向某对象的`std::sharedptr`会使得指向该对象的`std::sharedptr`多出一个，为什么我们不说构造一个`std::sharedptr`总是会增加引用计数？

Move构造函数是我为什么那么说的原因。从另外一个`std::shared_ptr` move构造(Move-constructing)一个`std::shared_ptr`会使得源`std::shared_ptr`指向为null,这就意味着新的`std::shared_ptr`取代了老的`std::shared_ptr`来指向原来的资源，所以就不需要再修改引用计数了。Move构造`std::shared_ptr`要比拷贝构造`std::shared_ptr`快：copy需要修改引用计数，然而拷贝缺不需要。对于赋值构造也是一样的。最后得出结论，move构造要比拷贝构造快，Move赋值要比copy赋值快。

像`std::unique_ptr`(Item 18)那样,`std::shared_ptr`也把delete作为它默认的资源析构机制。但是它也支持自定义的deleter.然后，它支持这种机制的方式不同于`std::unique_ptr`.对于`std::unique_ptr`,自定义的deleter是智能指针类型的一部分，对于`std::shared_ptr`,情况可就不一样了:

```cpp
auto loggingDel = [](widget *pw)
					{	
						makeLogEntry(pw);
						delete pw;
					}//自定义的deleter(如Item 18所说)
std::unique_ptr<Widget, decltype(loggingDel)>upw(new Widget, loggingDel);//deleter类型是智能指针类型的一部分

std::shared_ptr<Widget> spw(new Widget, loggingDel);//deleter类型不是智能指针类型的一部分
```

std::shared_prt的设计更加的弹性一些，考虑到两个std::shared_ptr<Widget>,每一个都支持不同类型的自定义deleter(例如，两个不同的lambda表达式):

```cpp
auto customDeleter1 = [](Widget *pw) {...};
auto customDeleter2 = [](Widget *pw) {...};//自定义的deleter,属于不同的类型

std::shared_prt<Widget> pw1(new Widget, customDeleter1);
std::shared_prt<Widget> pw2(new Widget, customDeleter2);
```

因为pw1和pw2属于相同类型，所以它们可以放置到属于同一个类型的容器中去:

```cpp
std::vector<std::shared_ptr<Widget>> vpw{ pw1, pw2 };
```

它们之间可以相互赋值，也都可以作为一个参数类型为`std::shared_ptr<Widget>`类型的函数的参数。所有的这些特性，具有不同类型的自定义deleter的`std::unique_ptr`全都办不到，因为自定义的deleter类型会影响到`std::unique_ptr`的类型。

与`std::unique_ptr`不同的其他的一点是，为`std::shared_ptr`指定自定义的deleter不会改变`std::shared_ptr`的大小。不管deleter如何，一个`std::shared_ptr`始终是两个pointer的大小。这可是个好消息，但是会让我们一头雾水。自定义的deleter可以是函数对象，函数对象可以包含任意数量的data.这就意味着它可以是任意大小。涉及到任意大小的自定义deleter的`std::shared_ptr`如何保证它不使用额外的内存呢？

它肯定是办不到的，它必须使用额外的空间来完成上述目标。然而，这些额外的空间不属于`std::shared_ptr`的一部分。额外的空间被分配在堆上，或者在`std::shared_ptr`的创建者使用了自定义的allocator之后，位于该allocator管理的内存中。我之前说过，一个`std::shared_ptr`对象包含了一个指针，指向了它所指对象的引用计数。此话不假，但是却有一些误导性，因为引用计数是一个叫做控制块(control block)的很大的数据结构。每一个由`std::shared_ptr`管理的对象都对应了一个控制块。改控制块不仅包含了引用计数，还包含了一份自定义deleter的拷贝(在指定好的情况下).如果指定了一个自定义的allocator,也会被包含在其中。控制块也可能包含其他的额外数据，比如Item 21条所说，一个次级(secondary)的被称作是weak count的引用计数，在本Item中我们先略过它。我们可以想象出`std::shared_ptr<T>`的内存布局如下所示:

一个对象的控制块被第一个创建指向它的`std::shared_ptr`的函数来设立.至少这也是理所当然的。一般情况下，函数在创建一个`std::shared_ptr`时，它不可能知道这时是否有其他的`std::shared_ptr`已经指向了这个对象，所以在创建控制块时，它会遵循以下规则：

* `std::make_shared`(请看Item 21)总是会创建一个控制块。它制造了一个新的可以指向的对象，所以可以确定这个新的对象在`std::make_shared`被调用时肯定没有相关的控制块。
* 当一个`std::shared_ptr`被一个独占性的指针(例如，一个`std::unique_ptr`或者`std::auto_ptr`)构建时，控制块被相应的被创建。独占性的指针并不使用控制块，所以被指向的对象此时还没有控制块相关联。(构造的一个过程是，由`std::shared_ptr`来接管了被指向对象的所有权，所以原来的独占性指针被设置为null).
* 当一个`std::shared_ptr`被一个原生指针构造时，它也会创建一个控制块。如果你想要基于一个已经有控制块的对象来创建一个`std::shared_ptr`，你可能传递了一个`std::shared_ptr`或者`std::weak_ptr`作为`std::shared_ptr`的构造参数，而不是传递了一个原生指针。`std::shared_ptr`构造函数接受`std::shared_ptr`或者`std::weak_ptr`时，不会创建新的控制块，因为它们(指构造函数)会依赖传递给它们的智能指针是否已经指向了带有控制块的对象的情况。

当使用了一个原生的指针构造多个`std::shared_ptr`时，这些规则的存在会使得被指向的对象包含多个控制块，带来许多负面的未定义行为。多个控制块意味着多个引用计数，多个引用计数意味着对象会被摧毁多次(每次引用计数一次)。这就意味着下面的代码着实糟糕透顶：

```cpp
auto pw = new Widget;			//pw是一个原生指针
...
std::shared_ptr<Widget> spw1(pw, loggingDel);//为*pw创建了一个控制块
...
std::shared_ptr<Widget> spw2(pw, loggingDel);//为pw创建了第二个控制块!
```

创建原生指针pw的行为确实不太好，这样违背了我们一整章背后的建议(请看开章那几段话来复习)。但是先不管这么多，创建pw的那行代码确实不太建议，但是至少它没有产生程序的未定义行为.

现在的情况是，因为spw1的构造函数的参数是一个原生指针，所以它为指向的对象(就是pw指向的对象:`*pw`)创造了一个控制块(伴随着一个引用计数)。到目前为止，代码还没有啥问题。但是随后，spw2也被同一个原生指针作为参数构造,它也为`*pw`创造了一个控制块(还有引用计数).`*pw`因此拥有了两个引用计数。每一个最终都会变成0，最终会引起两次对`*pw`的析构行为。第二次析构就要对未定义的行为负责了。

对于`std::shared_ptr`在这里总结两点.首先，避免给std::shared_ptr构造函数传递原生指针。通常的取代做法是使用std::make_shared(请看Item 21).但是在上面的例子中，我们使用了自定义的deleter,这对于std::make_shared是不可能的。第二，如果你必须要给std::shared_ptr构造函数传递一个原生指针，那么请直接传递new语句，上面代码的第一部分如果被写成下面这样：
```cpp
std::shared_ptr<Widget> spw1(new Widget,loggingDel);//direct use of new
```
这样就不大可能从同一个原生指针来构造第二个`std::shared_ptr`了。而且，创建spw2的代码作者会用spw1作为初始化(spw2)的参数(即，这样会调用std::shared_ptr的拷贝构造函数)。这样无论如何都不有问题:

```cpp
std::shared_ptr<Widget> spw2(spw1);//spw2 uses same control block as spw1
```

使用this指针时，有时也会产生因为使用原生指针作为`std::shared_ptr`构造参数而导致的产生多个控制块的问题。假设我们的程序使用`std::shared_ptr`来管理Widget对象，并且我们使用了一个数据结构来管理跟踪已经处理过的Widget对象：

```cpp
std::vector<std::shared_ptr<Widget>> processedWidgets;
```
进一步假设Widget有一个成员函数来处理:

```cpp
class Widget{
public:
	...
	void process();
	...
};
```
这有一个看起来很合理的Widget::process实现

```cpp
void Widget::process()
{
	...						//process the Widget
	processedWidgets.emplace_back(this);//add it to list
										//processed Widgets;
										//this is wrong!	   
}
```
注释里面说这样做错了，指的是传递this指针，并不是因为使用了`emplace_back`(如果你对`emplace_back`不熟悉，请看Item 42.)这样的代码会通过编译，但是给一个`std::shared_ptr`传递this就相当于传递了一个原生指针。所以`std::shared_ptr`会给指向的Widget(*this)创建了一个新的控制块。当你意识到成员函数之外也有`std::shared_ptr`早已指向了Widget，这就粗大事了，同样的道理，会导致发生未定义的行为。

`std::shared_ptr`的API包含了修复这一问题的机制。这可能是C++标准库里面最诡异的方法名字了：`std::enabled_from_this`.它是一个基类的模板，如果你想要使得被std::shared_ptr管理的类安全的以this指针为参数创建一个`std::shared_ptr`,就必须要继承它。在我们的例子中，Widget会以如下方式继承`std::enable_shared_from_this`：

```cpp
class Widget: public std::enable_shared_from_this<Widget>{
	public:
		...
		void process();
		...
};
```
正如我之前所说的，`std::enable_shared_from_this`是一个基类模板。它的类型参数永远是它要派生的子类类型，所以widget继承自`std::enable_shared_from_this<widget>`。如果这个子类继承自以子类类型为模板参数的基类的想法让你觉得不可思议，先放一边吧，不要纠结。以上代码是合法的，并且还有相关的设计模式，它有一个非常名字，虽然像`std::enable_shared_from_this`一样古怪,名字叫The Curiously Recurring Template Pattern(CRTP).欲知详情请使用你的搜索引擎。我们下面继续讲`std::enable_shared_from_this`.

`std::enable_shared_from_this`定义了一个成员函数来创建指向当前对象的`std::shared_ptr`,但是它并不重复创建控制块。这个成员函数的名字是`shared_from_this`,当你实现一个成员函数，用来创建一个`std::shared_ptr`来指向this指针指向的对象,可以在其中使用`shared_from_this`。下面是Widget::process的一个安全实现:

```cpp
void Widget::process()
{
	//as before, process the Widget
	...
	//add std::shared_ptr to current object to processedWidgets
	processedWidgets.emplace_back(shared_from_this());
}
```
`shared_from_this`内部实现是，它首先寻找当前对象的控制块，然后创建一个新的`std::shared_ptr`来引用那个控制块。这样的设计依赖一个前提，就是当前的对象必须有一个与之相关的控制块。为了让这种情况成真，事先必须有一个`std::shared_ptr`指向了当前的对象(比如说，在这个调用`shared_from_this`的成员函数的外面)，如果这样的`std::shared_ptr`不存在(即，当前的对象没有相关的控制块)，虽然shared_from_this通常会抛出异常，产生的行为仍是未定义的。

为了阻止用户在没有一个`std::shared_ptr`指向该对象之前，使用一个里面调用`shared_from_this`的成员函数，继承自`std::enable_shared_from_this`的子类通常会把它们的构造函数声明为private,并且让它们的使用者利用返回`std::shared_ptr`的工厂函数来创建对象。举个栗子，对于Widget来说，可以像下面这样写：

```cpp
class Widget: public std::enable_shared_from_this<Widget>{
public:
	//工厂函数转发参数到一个私有的构造函数
	template<typename... Ts>
	static std::shared_ptr<Widget> create(Ts&&... params);
	...
	void process();			//as before
	...
private:
	...							//构造函数
}
```
直到现在，你可能只能模糊的记得我们关于控制块的讨论源自于想要理解`std::shared_ptr`性能开销的欲望。既然我们已经理解如何避免创造多余的控制块，下面我们回归正题吧。

一个控制块可能只有几个字节大小，尽管自定义的deleters和allocators可能会使得它更大。通常控制块的实现会比你想象中的更复杂。它利用了继承，甚至还用到虚函数(确保指向的对象能正确销毁。)这就意味着使用`std::shared_ptr`会因为控制块使用虚函数而导致一定的机器开销。

当我们读到了动态分配的控制块，任意大小的deleters和allocators,虚函数机制，以及引用计数的原子操纵，你对`std::shared_ptr`的热情可能被泼了一盆冷水，没关系.它做不到对每一种资源管理的问题都是最好的方案。但是相对于它提供的功能，`std::shared_ptr`性能的耗费还是很合理。通常情况下，`std::shared_ptr`被`std::make_shared`所创建，使用默认的deleter和默认的allocator,控制块也只有大概三个字节大小。它的分配基本上是不耗费空间的(它并入了所指向对象的内存分配，欲知详情，请看Item 21.)解引用一个`std::shared_ptr`花费的代价不会比解引用一个原生指针更多。执行一个需要操纵引用计数的过程(例如拷贝构造和拷贝赋值，或者析构)需要一直两个原子操作，但是这些操作通常只会映射到个别的机器指令，尽管相对于普通的非原子指令他们可能更耗时，但它们终究仍是单个的指令。控制块中虚函数的机制在被`std::shared_ptr`管理的对象的生命周期中一般只会被调用一次：当该对象被销毁时。

花费了相对很少的代价，你就获得了对动态分配资源生命周期的自动管理。大多数时间，想要以共享式的方式来管理对象，使用`std::shared_ptr`是一个大多数情况下都比较好的选择。如果你发现自己开始怀疑是否承受得起使用`std::shared_ptr`的代价时，首先请重新考虑是否真的需要使用共享式的管理方法。如果独占式的管理方式可以或者可能实用，`std::unique_ptr`或者是更好的选择。它的性能开销于原生指针大致相同，并且从`std::unique_ptr`“升级”到s`td::shared_ptr`是很简单的，因为`std::shared_ptr`可以从一个`std::unique_ptr`里创建。

反过来可就不一定好用了。如果你把一个资源的生命周期管理交给了`std::shared_ptr`，后面没有办法在变化了。即使引用计数的值是1，为了让`std::unique_ptr`来管理它，你也不能重新声明资源的所有权。资源和指向它的`std::shared_ptr`之间的契约至死方休。不许离婚，取消或者变卦。

还有一件事情`std::shared_ptr`不好用，那就是用在数组上面。可`std::unique_ptr`不同的一点就是，`std::shared_ptr`的API设计为指向单个的对象。没有像`std::shared_ptr<T[]>`这样的用法。经常有一些自作聪明的程序员使用`std::shared_ptr<T>`来指向一个数组,指定了一个自定义的deleter来做数组的删除操作(即delete[]).这样做可以通过编译，但是却是个坏主意，原因有二，首先，`std::shared_ptr`没有重载操作符[],所以如果是通过数组访问需要通过丑陋的基于指针的运算来进行，第二，`std::shared_ptr` supports derived-to-base pointer conversions that make sense for single objects, but that open holes in the type system when applied to arrays. (For this reason, the `std::unique_ptr<T[]>` API prohibits such conversions.)更重要的一点是，鉴于C++11标准给了比原生数组更好的选择(例如，`std::array`,`std::vector`,`std::string`),给数组来声明一个智能指针通常是不当设计的表现。


|要记住的东西|
|:--------- |
|`std::shared_ptr`为了管理任意资源的共享式内存管理提供了自动垃圾回收的便利|
|`std::shared_ptr`是`std::unique_ptr`的两倍大，除了控制块，还有需要原子引用计数操作引起的开销|
|资源的默认析构一般通过delete来进行，但是自定义的deleter也是支持的。deleter的类型对于`std::shared_ptr`的类型不会产生影响|
|避免从原生指针类型变量创建`std::shared_ptr`|