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