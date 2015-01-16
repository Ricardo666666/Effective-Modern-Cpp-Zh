条款五：优先使用`nullptr`而不是`0`或者`NULL`
=========================
`0`字面上是一个`int`类型，而不是指针，这是显而易见的。`C++`扫描到一个`0`，但是发现在上下文中仅有一个指针用到了它，编译器将勉强将`0`解释为空指针，但是这仅仅是一个应变之策。`C++`最初始的原则是`0`是`int`而非指针。

经验上讲，同样的情况对`NULL`也是存在的。对`NULL`而言，仍有一些细节上的不确定性，因为赋予`NULL`一个除了`int`（即`long`）以外的整数类型是被允许的。这不常见，但是这真的是没有问题的，因为此处的焦点不是`NULL`的确切类型而是`0`和`NULL`都不属于指针类型。

在`C++98`中，这意味着重载指针和整数类型的函数的行为会令人吃惊。传递`0`或者`NULL`作为参数给重载函数永远不会调用指针重载的那个函数：
```cpp
	void f(int);          // 函数f的三个重载
	void f(bool);
	void f(void*);

	f(0);                // 调用 f(int)，而非f(void*)

	f(NULL);             // 可能无法编译，但是调用f(int)
	                     // 不可能调用 f(void*)
```
`f(NULL)`行为的不确定性的确反映了在实现`NULL`的类型上存在的自由发挥空间。如果`NULL`被定为`0L`（即`0`作为一个`long`整形），函数的调用是有歧义的，因为`long`转化为`int`，`long`转化为`bool`，`0L`转换为`void*`都被认为是同样可行的。关于这个函数调用有意思的事情是在源代码的字面意思（使用`NULL`调用`f`，`NULL`应该是个空指针）和它的真实意义（一个整数在调用`f`，`NULL`不是空指针）存在着冲突。这种违背直觉的行为正是`C++98`程序员不被允许重载指针和整数类型的原因。这个原则对于`C++11`依然有效，因为尽管有本条款的力荐，仍然还有一些开发者继续使用`0`和`NULL`，虽然`nullptr`是一个更好的选择。

`nullptr`的优势是它不再是一个整数类型。诚实的讲，它也不是一个指针类型，但是你可以把它想象成一个可以指向任意类型的指针。`nullptr`的类型实际上是`std::nullptr_t`，`std::nullptr_t`定义为`nullptr`的类型，这是一个完美的循环定义。`std::nullptr_t`可以隐式的转换为所有的原始的指针类型，这使得`nullptr`表现的像可以指向任意类型的指针。

使用`nullptr`作为参数去调用重载函数`f`将会调用`f(void*)`重载体，因为`nullptr`不能被视为整数类型的：
```cpp
	f(nullptr);             //调用f(void*)重载体
```
使用`nullptr`而不是`0`或者`NULL`，可以避免重载解析上的令人吃惊行为，但是它的优势不仅限于此。它可以提高代码的清晰度，尤其是牵扯到`auto`类型变量的时候。例如，你在一个代码库中遇到下面代码：
```cpp
	auto result = findRecord( /* arguments */);

	if(result == 0){
		...
	}
```
如果你不能轻松地的看出`findRecord`返回的是什么，要知道`result`是一个指针还是整数类型并不是很简单的。毕竟，`0`（被用来测试`result`的）即可以当做指针也可以当做整数类型。另一方面，你如果看到下面的代码：
```cpp
	auto result = findRecord( /* arguments */);

	if(reuslt == nullptr){
		...
	}
```
明显就没有歧义了：`result`一定是个指针类型。

当模板进入我们考虑的范围，`nullptr`的光芒则显得更加耀眼了。假想你有一些函数，只有当对应的互斥量被锁定的时候，这些函数才可以被调用。每个函数的参数是不同类型的指针：
```cpp
	int    f1(std::shared_ptr<Widget> spw);  // 只有对应的
	double f2(std::unique_ptr<Widget> upw);  // 互斥量被锁定
	bool   f3(Widget* pw);                   // 才会调用这些函数
```
想传递空指针给这些函数的调用看上去像这样：
```cpp
	std::mutex f1m, f2m, f3m;            // 对应于f1, f2和f3的互斥量 

	using MuxGuard =                     // C++11 版typedef；参加条款9
      std::lock_guard<std::mutex>;
    ...
    {
      MuxGuard g(f1m);                   // 为f1锁定互斥量
      auto result = f1(0);               // 将0当做空指针作为参数传给f1
    }                                    // 解锁互斥量

    ...

    {
      MuxGuard g(f2m);                   // 为f2锁定互斥量
      auto result = f2(NULL);            // 将NULL当做空指针作为参数传给f2
    }                                    // 解锁互斥量

    ...

    {
      MuxGuard g(f3m);                   // 为f3锁定互斥量
      auto result = f3(nullptr);         // 将nullptr当做空指针作为参数传给f3
    }                                    // 解锁互斥量
```
在前两个函数调用中没有使用`nullptr`是令人沮丧的，但是上面的代码是可以工作的，这才是最重要的。然而，代码中的重复模式——锁定互斥量，调用函数，解锁互斥量——才是更令人沮丧和反感的。避免这种重复风格的代码正是模板的设计初衷，因此，让我们使用模板化上面的模式：
```cpp
	template<typename FuncType,
	         typename MuxType,
	         typename PtrType>
	auto lockAndCall(FuncType func,
	                 MuxType& mutex,
	                 PtrType ptr) -> decltype(func(ptr))
	{
      MuxGuard g(mutex);
      return func(ptr);
    }
```