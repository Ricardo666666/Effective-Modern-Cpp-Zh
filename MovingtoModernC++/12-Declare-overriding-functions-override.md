条款12：使用override关键字声明覆盖的函数
=========================
`C++`中的面向对象的变成都是围绕类，继承和虚函数进行的。其中最基础的一部分就是，派生类中的虚函数会覆盖掉基类中对应的虚函数。但是令人心痛的意识到虚函数重载是如此容易搞错。这部分的语言特性甚至看上去是按照墨菲准则设计的，它不需要被遵从，但是要被膜拜。

因为覆盖“`overriding`”听上去像重载“`overloading`”，但是它们完全没有关系，我们要有一个清晰地认识，虚函数（覆盖的函数）可以通过基类的接口来调用一个派生类的函数：
```cpp
	class Base{
	public:
	  virtual void doWork();                          // 基类的虚函数
	  ...
	};

	class Derived: public Base{
	public:
	  virtual void doWork();                         // 覆盖 Base::doWork
	                                                 // ("virtual" 是可选的)
	  ...
	};

	std::unique_ptr<Base> upb =                      // 产生一个指向派生类的基类指针
	                                                 // 关于 std::make_unique 的信息参考条款21
	  std::make_unique<Derived>();

	...

	upb->doWork();                                   // 通过基类指针调用 doWork()，
	                                                 // 派生类的对应函数别调用
```
如果要使用覆盖的函数，几个条件必须满足：
- 基类中的函数被声明为虚的。
- 基类中和派生出的函数必须是完全一样的（出了虚析构函数）。
- 基类中和派生出的函数的参数类型必须完全一样。
- 基类中和派生出的函数的常量特性必须完全一样。
- 基类中和派生出的函数的返回值类型和异常声明必须使兼容的。

以上的约束仅仅是`C++98`中要求的部分，`C++11`有增加了一条：

- 函数的引用修饰符必须完全一样。成员函数的引用修饰符是很少被提及的`C++11`的特性，所以你之前没有听说过也不要惊奇。这些修饰符使得将这些函数只能被左值或者右值使用成为可能。成员函数不需要声明为虚就可以使用它们：
```cpp
	class Widget{
	public:
	  ...
	  void doWork() &;                               // 只有当 *this 为左值时
	                                                 // 这个版本的 doWorkd()
	                                                 // 函数被调用

	  void doWork() &&;                              // 只有当 *this 为右值
	                                                 // 这个版本的 doWork()
	                                                 // 函数被调用
	};
	...
	Widget makeWidget();                             // 工厂函数，返回右值

	Widget w;                                        // 正常的对象（左值）

	...

	w.doWork();                                      // 为左值调用 Widget::doWork() 
	                                                 //（即 Widget::doWork &）

	makeWidget().doWork();                           // 为右值调用 Widget::doWork() 
	                                                 //（即 Widget::doWork &&）
```
稍后我们会更多介绍带有引用修饰符的成员函数的情况，但是现在，我们只是简单的提到：如果一个虚函数在基类中有一个引用修饰符，派生类中对应的那个也必须要有完全一样的引用修饰符。如果不完全一样，派生类中的声明的那个函数也会存在，但是它不会覆盖基类中的任何东西。

对覆盖函数的这些要求意味着，一个小的错误会产生一个很大不同的结果。在覆盖函数中出现的错误通常还是合法的，但是它导致的结果并不是你想要的。所以当你犯了某些错误的时候，你并不能依赖于编译器对你的通知。例如，下面的代码是完全合法的，乍一看，看上去也是合理的，但是它不包含任何虚覆盖函数——没有一个派生类的函数绑定到基类的对应函数上。你能找到每种情况里面的问题所在吗？即为什么派生类中的函数没有覆盖基类中同名的函数。
```cpp
	class Base {
	public:
	  virtual void mf1() const;
	  virtual void mf2(int x);
	  virtual void mf3() &;
	  void mf4() const;
	};

	class Derived: public Base {
	 public:
	   virtual void mf1();
	   virtual void mf2(unsigned int x);
	   virtual void mf3() &&;
	   void mf4() const;
	};
```
需要什么帮助吗？
- `mf1`在`Base`中声明常成员函数，但是在`Derived`中没有
- `mf2`在`Base`中以`int`为参数，但是在`Derived`中以`unsigned int`为参数
- `mf3`在`Base`中有左值修饰符，但是在`Derived`中是右值修饰符
- `mf4`没有继承`Base`中的虚函数
