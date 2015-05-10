##Item 21 优先使用`std::make_unique`和`std::make_shared`而不是直接使用new

我们先给`std::make_unique`以及`std::make_shared`提供一个公平的竞争环境，以此开始。`std::make_shared`是C++ 11标准的一部分，但是，遗憾的是，`std::make_unique`不是的。它刚成为C++ 14的一部分。如果你在使用C++11.不要怕，因为你可以很容易自己写一个基本版的`std::make_unique`，我们瞧：

```cpp
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique<Ts&&... params>
{
	return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```
如你所见，`make_unique`只是完美转发了它的参数到它要创建的对象的构造函数中去，由new出来的原生指针构造一个`std::unique_ptr`，并且将之返回。这中格式的构造不支持数组以及自定义deleter(请看 Item18),但是它说明只需稍加努力，便可自己创造出所需要的`make_unique`(备注：为了尽可能花最小大家创造出一个功能齐全的`make_unique`，搜索产生它的标准文档，并且拷贝一份文档中的实现。这里所需要的文档是日期为2013-04-18，Stephan T.Lavavej所写的N3656).请记住不要把你自己实现的版本放在命名空间std下面，因为假如说日后你升级到C++ 14的标准库市县，你可不想自己实现的版本和标准库提供的版本产生冲突。

`std::make_unique`以及`std::make_shared`是3个make函数的其中2个：make函数接受任意数量的参数，然后将他们完美转发给动态创建的对象的构造函数，并且返回指向那个对象的智能指针。第三个make函数是`std::allocate_shared`,除了第一个参数是一个用来动态分配内存的allocator对象，它表现起来就像`std::make_shared`.

即使是最普通的是否使用make函数来创建智能指针之间的比较，也表明了为什么使用make函数是比较可行的做法。考虑一下代码:

```cpp
auto upw1(std::make_unique<Widget>());//使用make函数
std::unique_ptr<Widget> upw2(new Widget);//不使用make函数

auto spw1(std::make_shared<Widget>());//使用make函数
std::shared_ptr<Widget> spw2(new Widget);//不使用make函数
```
我已经高亮显示了必要的差别(不好意思，它这里高亮的是Widget,在代码里高亮暂时俺还做不到--译者注)：使用new需要重复写一遍type，而使用make函数不需要。重复敲type违背了软件工程中的一项基本原则：代码重复应当避免。源代码里面的重复会使得编译次数增加,导致对象的代码变得臃肿，由此产生出的code base(code base的含义请至http://en.wikipedia.org/wiki/Codebase--译者注)变得难以改动以及维护。它经常会导致产生不一致的代码。一个code base中的不一致代码会导致bug.并且，敲某段代码两遍会比敲一遍更费事，谁不想写程序时敲比较少的代码呢。

第二个偏向make函数的原因是为了保证产生异常后程序的安全。设想我们有一个函数根据某个优先级来处理Widget：

```cpp
void processWidget(std::shared_ptr<Widget> spw,int priority);
```
按值传递`std::shared_ptr`可能看起来很可疑，但是Item41解释了如果processWidget总是要创建一个`std::shared_ptr`的拷贝(例如，存储在一个数据结构中，来跟踪已经被处理过的Widget)，这也是一个合理的设计.

现在我们假设有一个函数来计算相关的优先级

```cpp
int computePriority()
```

如果我们调用processWidget时，使用new而不是`std::make_shared`:

```cpp
processWidget(std::shared_ptr<Widget>(new Widget),computePriority())
//可能会导致内存泄露！
```

就像注释里面所说的，这样的代码会产生因new引发的Widget对象的内存泄露。但是怎么会这样？函数的声明和调用函数的代码都使用了`std::shared_ptr`,设计`std::shared_ptr`的目的就是防止内存泄露。当指向资源的最后一个`std::shared_ptr`即将离去时，资源会自动得到析构。不管是什么地方，每个人都在用`std::shared_ptr`，为什么还会发生内存泄露？

这个问题的答案和编译器将源代码翻译为object code(目标代码，想要知道object code是什么,请看这个问题http://stackoverflow.com/questions/466790/assembly-code-vs-machine-code-vs-object-code)有关系。在运行时(runtime:In computer science, run time, runtime or execution time is the time during which a program is running (executing), in contrast to other phases of a program's lifecycle such as compile time, link time and load time.)。在函数被调用前，函数的参数必须被推算出来，所以在调用processWidget的过程中，processWidget开始执行之前，下面的事情必须要发生：
 
* "new Widget"表达式必须被执行，即，一个Widget必须在堆上被创建
* 负责管理new所创建的指针的`std::shared_ptr<Widget>`的构造函数必须被执行
* computePriority必须被执行

并没有要求编译器产生出对这些操作做到按顺序执行的代码。"new Widget"必须要在std::shared_ptr的构造函数被调用之前执行，因为new的结果作为该构造函数的一个参数,因为computePriority可能在这些调用之前执行，或者之后，更关键的是，或者在它们之间。这样的话，编译器可能按如下操作的顺序产生出代码:

1. 执行"new Widget".
2. 执行computePriority.
3. 执行std::shared_ptr的构造函数.

如果这样的代码在runtime被产生出来,computePriority产生出了一个异常，那么在Step 1中动态分配的Widget可能会产生泄漏.因为它永远不会存储在Step 3中产生的本应负责管理它的`std::shared_ptr`中。

使用`std::make_shared`可以避免这个问题。调用的代码看起来如下所示：

```cpp
processWidget(std::make_shared<Widget>(),computePriority);//不会有内存泄漏的危险
```
在runtime的时候，`std::make_shared`或者computePriority都有可能被第一次调用。如果是`std::make_shared`先被调用,被动态分配的Widget安全的存储在返回的`std::shared_ptr`中(在computePriority被调用之前)。如果computePriority产生了异常，`std::shared_ptr`的析构函数会负责把它所拥有的Widget回收。如果computePriority首先被调用并且产生出一个异常，`std::make_shared`不会被调用，因此也不必担心动态分配的Widget会产生泄漏的问题。

如果我们将std::shared_ptr和std::make_shared替换为std::unique_ptr和对应的std::make_unique,同样的分析也会适用。适用std::make_unique而不使用new的原因和使用std::make_shared的目的相同，都是出于写出异常安全(exception-safe)代码的考虑。

一个使用`std::make_shared`（和直接使用new相比）的显著特性就是提升了效率。使用std::make_shared允许编译器利用简洁的数据结构产生出更简洁，更快的代码。考虑下面直接使用new的效果

```cpp
std::shared_ptr<Widget> spw(new Widget);
```

很明显的情况是代码只需一次内存分配，但实际上它执行了两次。Item 19解释了每一个std::shared_ptr都指向了一个包含被指向对象的引用计数的控制块，控制块的分配工作在std::shared_ptr的构造函数内部完成。直接使用new，就需要一次为Widget分配内存，第二次需要为控制块分配内存。

如果使用的是std::make_shared,

```cpp
auto spw = std::make_shared<Widget>();
```
一次分配足够了。这是因为std::make_shared分配了一整块空间，包含了Widget对象和控制块。这个优化减少了程序的静态大小，因为代码中只包含了一次分配调用，并且加快了代码的执行速度，因为内存只被分配一次。此外，使用std::make_shared避免了在控制块中额外添加的一些记录信息的需要，潜在的减少了程序所需的总内存消耗。

上文的`std::make_shared`效率分析同样使用于std::allocate_shared，所以std::make_shared的性能优点也可以延伸到std::allocate_shared函数。

上面说了这么多偏爱make函数，而不是直接用new的理由，每一个都理直气壮。但是，抛开什么软件工程，异常安全和性能的优点，这个Item教程的目的是偏向make函数，并不是要我们完全依赖它们。这是因为有一些情况下，make函数不能或者不应该被使用。

例如，make函数都不支持指定自定义的deleter(请看Item18和Item19).但是std::unique_ptr以及std::shared_ptr都有构造函数来支持这样做。比如，给定一个Widget的自定义deleter

```cpp
auto  widgetDeleter = [](Widget* pw){...};
```
直接使用new创建一个智能指针来直接使用它

```cpp
std::unique_ptr<Widget, decltype(widgetDeleter)> upw(new Widget, widgetDeleter);
std::shared_ptr<Widget> spw(new Widget, widgetDeleter);
```
用make函数可做不了这种事情。

make函数的第二个限制来自于它们实现的句法细节。Item 7解释了当创建了一个对象，该对象的类型重载了是否以std::initializer_list为参数的两种构造函数，使用大括号的方式来构造对象偏向于使用以std::initializer_list为参数的构造函数。而使用括号来构造对象偏向于调用非std::initializer_list的构造函数。make函数完美转发它的参数给对象的构造函数，但是，它使用的是括号还是大括号方式呢？对于某些类型，这个问题的答案产生的结果大有不同。举个例子，在下面的调用中：

```cpp
auto upv = std::make_unique<std::vector<int>>(10,20)
auto spv = std::make_shared<std::vector<int>>(10,20);
```

产生的智能指针所指向的std::vector是拥有10个元素，每个元素的值都是20，还是拥有两个值，分别是10和20？或者说结果是不确定性的？

好消息是结果是确定性的：两个调用都产生了同样的std::vector:拥有10个元素，每个元素的值被设置成了20.这就意味着在make函数中，完美转发使用的是括号而非大括号格式。坏消息是如果你想要使用大括号格式来构造指向的对象，你必须直接使用new.使用make函数需要完美转发大括号initializer的能力，但是，正如Item 30所说的那样，大括号initializer是没有办法完美转发的。但是，Item 30同时描述了一个变通方案：使用auto类型推导从大括号initializer(请看Item 2)中来创建一个std::initializer_list对象，然后将auto创建出来的对象传递给make函数：


```cpp
//使用std::initializer_list创建
auto initList = {10, 20};
//使用std::initializer_list为参数的构造函数来创建std::vector
auto spv = std::make_shared<std::vector<int>>(initList);
```
对于std::unique_ptr,这里只是存在两个场景(自定义的deleter以及大括号initializer)make函数不适用。但对于std::shared_ptr来说，问题可不止两个了。还有另外两个，但是都可称之为边缘情况，但确实有些程序员会处于这种边缘情况，你也有可能会碰到。

一些对象定义它们自己的new和deleter操作符。这些函数的存在暗示了为这种类型的对象准备的全局的内存分配和回收方法不再适用。通常情况下，这种自定义的new和delete都被设计为只分配或销毁恰好是一个属于该类的对象大小的内存，例如，Widget的new和deleter操作符经常被设计为：只是处理大小就是sizeof(Widget)的内存块的分配和回收。而std::shared_ptr支持的自定义的分配(通过std::allocate_shared)以及回收(通过自定义的deleter)的特性，上文描述的过程就支持的不好了，因为std::allocate_shared所分配的内存大小不仅仅是动态分配对象的大小，它所分配的大小等于对象的大小加上一个控制块的大小。所以，使用make函数创建的对象类型如果包含了此类版本的new以及delete操作符，此时(使用make)确实是个坏主意。

使用std::make_shared相对于直接使用new的大小及性能优点源自于：std::shared_ptr的控制块是和被管理的对象放在同一个内存区块中。当该对象的引用计数变成了0，该对象被销毁（析构函数被调用）。但是，它所占用的内存直到控制块被销毁才能被释放，因为被动态分配的内存块同时包含了两者。

我之前提到过，控制块除了它自己的引用计数，还记录了一些其它的信息。引用计数记录了多少个std::shared_ptr引用了当前的控制块，但控制块还包含了第二个引用计数，记录了多少哥std::weak_ptr引用了当前的控制块。第二个引用计数被称之为weak count（备注：在实际情况中，weak count不总是和引用控制块的std::weak_ptr的个数相等，库的实现往weak count添加了额外的信息来生成更好的代码(facilitate better code generation).但为了本Item的目的，我们忽略这个事实，假设它们是相等的）.当std::weak_ptr检查它是否过期(请看Item 19)时,它看看它所引用的控制块中的引用计数(不是weak count)是否是0(即是否还有std::shared_ptr指向被引用的对象，该对象是否因为引用为0被析构)，如果是0，std::weak_ptr就过期了，否则反之。

只要有一个std::weak_ptr还引用者控制块(即，weak count大于0)，控制块就会继续存在，包含控制块的内存就不会被回收。被std::shared_ptr的make函数分配的内存，直至指向它的最后一个std::shared_ptr和最后一个std::weak_ptr都被销毁时，才会得到回收。

当类型的对象很大，而且最后一个std::shared_ptr的析构于最后一个std::weak_ptr析构之间的间隔时间很大时，该对象被析构与它所占用的内存被回收之间也会产生间隔：

```cpp
class ReallyBigType{...};
auto pBigObj = std::make_shared<ReallyBigType>();
//使用std::make_shared来创建了一个很大的对象

... //创建了一些std::shared_ptr和std::weak_ptr来指向那个大对象

... //最后一个指向对象的std::shared_ptr被销毁了
//但是仍有指向它的std::weak_ptr存在

...//在这段时间内，之前为大对象分配的内存仍未被回收

...//最后一个指向该对象的std::weak_ptr在次被析构了；控制块和对象的内存也在此释放
```

如果直接使用了new，一旦指向ReallyBigType的最后一个std::shared_ptr被销毁，对象所占的内存马上得到回收.（本质上使用了new,控制块和动态分配的对象所处的内存不在一起，可以单独回收）

```cpp
class ReallyBigType{...}; //as before
std::shared_ptr<ReallyBigType> pBigObj(new ReallyBigType);//使用new创建了一个大对象

... //就像之前那样，创建一些std::shared_ptr和std::weak_ptr指向该对象。

... //最后一个指向对象的std::shared_ptr被销毁了
//但是仍有指向它的std::weak_ptr存在
//但是该对象的内存在此也会被回收

...//在这段时间内，只有为控制块分配的内存未被回收

...//最后一个指向该对象的std::weak_ptr在次被析构了；控制块的内存也在此释放
```

你发现自己处于一个使用std::make_shared不是很可行甚至是不可能的境地，你想到了之前我们提到的异常安全的问题。实际上直接使用new时，只要保证你在一句代码中，只做了将new的结果传递给一个智能指针的构造函数，没有做其它事情。这也会阻止编译器在new的使用和调用用来管理new的对象的智能指针的构造函数之间，插入可能会抛出异常的代码。

举个栗子，对于我们之间检查的那个异常不安全的processWidget函数，我们在之上做个微小的修订。这次，我们指定一个自定的deleter:

```cpp
void processWidget(std::shared_ptr<Widget> spw,
						int priority); //as before
void cusDel(Widget *ptr);//自定义的deleter						
```
这里有一个异常不安全的调用方式：

```cpp
processWidget(std::shared_ptr<Widget>(new Widget,cusDel),
computePriority())//as before,可能会造成内存泄露
```
回想：如果computerPriority在"new Widget"之后调用，但是在std::shared_ptr构造函数执行之前，并且如果computePriority抛出了一个异常，那么动态分配的Widget会被泄露。

在此我们使用了自定义的deleter,所以就不能使用std::make_shared了，想要避免这个问题，我们就得把Widget的动态分配以及std::shared_ptr的构造单独放到一句代码中，然后以该句代码得到的std::shared_ptr来调用std::shared_ptr.这就是技术的本质，尽管过会儿你会看到我们对此稍加改进来提升性能。

```cpp
std::shared_ptr<Widget> spw(new Widget, cusDel);
processWidget(spw, computePriority());//正确的，但不是最优的：看下面
```

确实可行，因为即使构造函数抛出异常，std::shared_ptr也已经接收了传给它的构造函数的原生指针的所有权.在本例中，如果spw的构造函数抛出异常(例如，假如因为无力去给控制块动态分配内存)，它依然可以保证cusDel可以在“new Widget”产生的指针上面调用。

在异常非安全的调用中，我们传递了一个右值给processWidget，

```cpp
processWidget(std::shared_ptr<Widget>(new Widget, cusDel),	//arg是一个右值 
computePriority());
```
而在异常安全的调用中，我们传递了一个左值：

```cpp
processWidget(spw, computePriority());//arg是一个左值
```
这就是造成性能问题的原因。

因为processWidget的std::shared_ptr参数按值传递，从右值构造只需要一个move，然而从左值构造却需要一个copy操作。对于std::shared_ptr来说，区别是显著的，因为copy一个std::shared_ptr需要对它的引用计数进行原子加1，然后move一个std::shared_ptr不需要对引用计数做任何操作。对于异常安全的代码来说，若想获得和非异常安全代码一样的性能表现，我们需要对spw用std::move,把它转化成一个右值(看Item 23)：

```cpp
processWidget(std::move(spw), computePriority());
//即异常安全又获得了效率
```

是不是很有趣，值得一看。但是这种情况不是很常见。因为你也很少有原因不使用make函数。如果不是非要用其他的方式不可，我还是推荐你尽量使用make函数。

|要记住的东西|
|:--------- |
|和直接使用new相比，使用make函数减少了代码的重复量，提升了异常安全度，并且，对于std::make_shared以及std::allocate_shared来说，产生的代码更加简洁快速|
|也会存在使用make函数不合适的场景：包含指定自定义的deleter,以及传递大括号initializer的需要|
|对于std::shared_ptr来说，使用make函数的额外的不使用场景还包含(1)带有自定义内存管理的class(2)内存非常紧俏的系统，非常大的对象以及比对应的std::shared_ptr活的还要长的std::weak_ptr|

