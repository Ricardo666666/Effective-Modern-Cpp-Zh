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

就像注释里面所说的，