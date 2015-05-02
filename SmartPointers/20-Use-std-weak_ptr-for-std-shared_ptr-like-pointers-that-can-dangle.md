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
