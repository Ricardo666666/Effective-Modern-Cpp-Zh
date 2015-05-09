Item 20：Use `std::weak_ptr` for `std::shared_ptr` like pointers that can dangle.
=========================
说起来有些矛盾，可以很方便的创建一个表现起来想`std::shared_ptr`的智能指针，但是它却不会参于被指向资源的共享式管理。换句话说，一个类似于`std::shared_ptr`的指针不影响它所指对象的引用计数。这种类型的智能指针必须面临一个`std::shared_ptr`未曾面对过的问题:它所指向的对象可能已经被析构。一个真正的智能指针通过持续跟踪判断它是否已经悬挂(dangle)来处理这种问题，悬挂意味着它指向的对象已经不复存在。这就是`std::weak_ptr`的功能所在

你可能怀疑`std::weak_ptr`怎么会有用，当你检查了下`std::weak_ptr`的API之后，你会觉得更奇怪。它的API看起来一点都不智能。`std::weak_ptr`不能被解引用，也不能检测判空。这是因为`std::weak_ptr`不能被单独使用，它是`std::shared_ptr`作为参数的产物。

这种关系与生俱来，`std::weak_ptr`通常由一个`std::shared_ptr`来创建，它们指向相同的地方，`std::shared_ptr`来初始化它们，但是`std::weak_ptr`不会影响到它所指向对象的引用计数:

```cpp
auto spw = std::make_shared<Widget>();//spw 被构造之后
							//被指向的Widget对象的引用计数为1
							//(欲了解std::make_shared详情，请看Item21)
...
std::weak_ptr<Widget> wpw(spw);//wpw和spw指向了同一个Widget,但是RC(这里指引用计数，下同)仍旧是1
...
spw = nullptr;//RC变成了0，Widget也被析构，wpw现在处于悬挂状态
```
悬挂的std::weak_ptr可以称作是过期了(expired),可以直接检查是否过期：

```cpp
if(wpw.expired())...		//如果wpw悬挂...
```

但是我们最经常的想法是：查看`std::weak_ptr`是否已经过期，如果没有过期的话，访问它所指向的对象。想的容易做起来难啊。因为`std::weak_ptr`缺少解引用操作，也就没办法写完成这样操作的代码。即使又没法做到，将检查和解引用分开的写法也会引入一个竞态存在：在调用expired以及解引用操作之间，另外一个线程可能对被指向的对象重新赋值或者摧毁了最后一个指向对象的`std::shared_ptr`,这样就导致了被指向的对象的析构。这种情况下，你的解引用操作会产生未定义行为。

我们需要的是将检查`std::weak_ptr`是否过期，以及如果未过期的话获得访问所指对象的权限这两种操作合成一个原子操作。这是通过由`std::weak_ptr`创建出一个`std::shared_ptr`来完成的。根据当`std::weak_ptr`已经过期，仍以它为参数创建`std::shared_ptr`会发生的情况的不同，这种创建有两种方式。一种方式是通过`std::weak_ptr::lock`,它会返回一个`std::shared_ptr`,当`std::weak_ptr`已经过期时，`std::shared_ptr`会是null：

```cpp
std::shared_ptr<Widget> spw1 = wpw.lock();//如果wpw已经过期
//spw1的值是null
auto spw2 = wpw.lock();//结果同上，这里使用了auto
```
另外一种方式是以`std::weak_ptr`为参数，使用`std::shared_ptr`构造函数。这种情况下，如果`std::weak_ptr`过期的话，会有异常抛出：

```cpp
std::shared_ptr<Widget> spw3(wpw);//如果wpw过期的话
//抛出std::bad_weak_ptr异常
```
你可能会产生疑问，`std::weak_ptr`到底有啥用。下面我们举个例子，假如说现在有一个工厂函数，根据一个唯一的ID，返回一个指向只读对象的智能指针。根据Item 18关于工厂函数返回类型的建议，它应该返回一个`std::unique_ptr`:

```cpp
std::unique_ptr<const Widget> loadWidget(WidgetID id);
```
如果loadWidget调用的代价不菲(比如，它涉及到了文件或数据库的I/O操作)，而且ID的使用也比较频繁，一个合理的优化就是再写一个函数，不仅完成loadWidget所做的事情，而且要缓存loadWidget的返回结果。把每一个请求过的Widget对象都缓存起来肯定会导致缓存自身的性能出现问题，所以，一个合理的做法是当被缓存的Widget不再使用时将它销毁。

对于这样的一个带有缓存的工厂函数，返回`std::unique_ptr`类型不是一个很好的选择。可以确定的两点是：调用者接收指向缓存对象的智能指针，调用者来决定这些缓存对象的生命周期；但是，缓存也需要一个指向所缓存对象的指针。因为当工厂函数的调用者使用完了一个工厂返回的对象，这个对象会被销毁，对应的缓存项会悬挂，所以缓存的指针需要有检测它现在是否处于悬挂状态的能力。因此缓存使用的指针应该是std::weak_ptr类型，它有检测悬挂的能力。这就意味着工厂函数的返回类型应该是`std::shared_ptr`,因为只有当一个对象的生命周期被`std::shared_ptr`所管理时，`std::weak_ptr`才能检测它自身是否处于悬挂状态。

下面是一个较快却欠缺完美的缓存版本的loadWidget的实现：

```cpp
std::shared_ptr<const Widget> fastLoadWidget(WidgetId id)
{
	static std::unordered_map<WidgetID,
	std::weak_ptr<const Widget>> cache;
	auto objPtr = cache[id].lock();//objPtr是std::shared_ptr类型
	//指向了被缓存的对象(如果对象不在缓存中则是null)
	
	if(!objPtr){
		objPtr = loadWidget(id);
		cache[id] = objPtr;
	}//如果不在缓存中，载入并且缓存它
	return objPtr;
}
```
C++11利用了hash表容器(`std::unordered_map`),尽管它没有提供所需的WidgetID哈希算法以及相等比较函数。

我为啥要说fastLoadWidget实现欠缺完美，因为它忽略了一个事实，缓存可能把一些已经过期的`std::weak_ptr`(对应的Widget不会被使用了，已经被销毁了)。所以它的实现还可以再改善下，但是我们还是不要深究了，因为深究对我们继续深入了解`std::weak_ptr`没有用处。我们下面探究第二个使用`std::weak_ptr`的场景：在观察者模式中，主要的组成部分是：状态可能会发生变化的subjects，以及当状态变化时需要得到通知的observers.在大多数实现中，每一个subject包含了指向它的observers的数据成员.这就使得subject很容易发送出状态变化的通知。subject对于控制他们的observer的生命周期(observer何时被析构)毫无兴趣.但是，它们必须知道，如果一个observer析构了，subject就不能尝试去访问它了。一个合理的设计是：每一个subject拥有一个`std::weak_ptr`，指向了它的observer,这样在可以在访问之间，先检查一下指针是否处于悬挂状态。

下面讲到最后一个`std::weak_ptr`的例子，有这样一个数据结构，包含A,B和C。A和C共享B的所有权，它们各自包含了一个`std::shared_ptr`指向B

![20-1.png]

如果现在有需要使B拥有反向指针指向A,那么指针应该是什么类型？

![20-2.png]

下面有三种选择：

* __一个原生指针__。如果这么做，A如果被析构了，但是C会继续指向B,B包含的指向A的指针现在处于悬挂状态。而B对此毫不知情，所以B有可能不小心反引用了那个悬挂指针，这样会产生未定义的行为。
* __一个`std::shared_ptr`__。在这种设计下，A和B包含了`std::shared_ptr`互相指向对方。结果就引发了一个`std::shared_ptr`的环(A指向B,B指向A),这个环会使得A和B都不能得到析构。即使程序其他的数据结构都不能访问到A和B(例如，C如果不再指向B)，A和B的引用计数仍然是1.如果这种情况发生了，A和B都会是内存泄露的情况,实际上，程序永远无法再访问到它们，它们也永远无法得到回收。
* __一个`std::weak_ptr`__。这样避免了以上所有的问题。如果A被回收，B指向它的指针将会悬挂，B也有能力检测到这一状态。此外，就算A和B互相指向对方，B的指针也不会影响到A的引用计数。当没有`std::shared_ptr`指向A时，也不会阻止A的析构。

使用`std::weak_ptr`毫无疑问是最好的选择。然而，值得注意的是，使用`std::weak_ptr`来破坏预期的`std::shared_ptr`形成的环不是那么普遍。在定义的比较严格的数据结构，比如说树,子节点一般被父节点所拥有。当父节点被析构时，子节点也应该会被析构。从父节点指向子节点的链接因此最好使用std::unique_ptr.因为子节点不应该比父节点存在的时间过长，从子节点指向父节点的链接可以安全的使用原生指针来实现。因此也不会出现子节点解引用一个指向父节点的悬挂指针。

当然，并不是所有的以指针为基础的数据结构都是严格的层级关系。如果不是的话，就像刚才所说的缓存以及观察者列表的情形，使用`std::weak_ptr`是最棒的选择了。

从效率的观点来看，`std::weak_ptr`和`std::shared_ptr`的情况基本相同，。`std::weak_ptr`对象的大小和`std::shared_ptr`对象相同，它们都利用了同样的控制块(请看Item 19),并且诸如构造，析构以及赋值都涉及到引用计数的原子操作。这可能让你吃了一惊，因为我在本章开始的时候说`std::weak_ptr`不参与引用计数的操作。可能没有表达完整我的意思。我要写的意思是`std::weak_ptr`不参与对象的共享所有权，因此不影响被指向对象的引用计数。但是，实际上在控制块中存在第二个引用计数，`std::weak_ptr`来操作这个引用计数。欲知详情，请看Item 21.

|要记住的东西|
|:--------- |
|`std::weak_ptr`用来模仿类似std::shared_ptr的可悬挂指针|
|潜在的使用`std::weak_ptr`的场景包括缓存，观察者列表，以及阻止`std::shared_ptr`形成的环|


