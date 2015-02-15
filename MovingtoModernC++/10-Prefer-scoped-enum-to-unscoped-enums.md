条款10：优先使用作用域限制的`enmus`而不是无作用域的`enum`
=========================
一般而言，在花括号里面声明的变量名会限制在括号外的可见性。但是这对于`C++98`风格的`enums`中的枚举元素并不成立。枚举元素和包含它的枚举类型同属一个作用域空间，这意味着在这个作用域中不能再有同样名字的定义：
```cpp
	enum Color { black, white, red};                 // black, white, red 和
	                                                 // Color  同属一个定义域


	auto white = false;                              // 错误！因为 white
	                                                 // 在这个定义域已经被声明过

```
事实就是枚举元素泄露到包含它的枚举类型所在的作用域中，对于这种类型的`enum`官方称作无作用域的（`unscoped`）。在`C++11`中对应的使用作用域的enums（`scoped enums`）不会造成这种泄露：
```cpp
	enum class Color { black, white, red};            // black, white, red
	                                                  // 作用域为 Color

	auto white = false;                               // fine, 在这个作用域内
	                                                  // 没有其他的 "white"

	Color c = white;                                  // 错误！在这个定义域中
	                                                  // 没有叫"white"的枚举元素

	Color c = Color::white;                           // fine

	auto c = Color::white;                            // 同样没有问题（和条款5
	                                                  // 的建议项吻合）
```	
因为限制作用域的`enum`是通过"enum class"来声明的，它们有时被称作枚举类（`enum class`）。

限制作用域的`enum`可以减少命名空间的污染，这足以是我们更偏爱它们而不是不带限制作用域的表亲们。除此之外，限制作用域的`enums`还有一个令人不可抗拒的优势：它们的枚举元素可以是更丰富的类型。无作用域的`enum`会将枚举元素隐式的转换为整数类型（从整数出发，还可以转换为浮点类型）。因此像下面这种语义上荒诞的情况是完全合法的：
```cpp
	enum Color { black, white, red };                  // 无限制作用域的enum

	std::vector<std::size_t>                           // 返回x的质因子的函数
	  primeFactors(std::size_t x);

	Color c = red;
	...

	if (c < 14.5 ){                                   // 将Color和double类型比较！

		auto  factors =                               // 计算一个Color变量的质因子
		  primeFactors(c); 
	}
```
在`"enum"`后增加一个`"class"`，就可以将一个无作用域的`enum`转换为一个有作用域的`enum`，变成一个有作用域的`enum`之后，事情就变得不一样了。在有作用域的`enum`中不存在从枚举元素到其他类型的隐式转换：
```cpp
	enum class Color { black, white, red };           // 有作用域的enum

	Color c = Color::red;                             // 和前面一样，但是
	...                                               // 加上一个作用域限定符

	if (c < 14.5){                                    // 出错！不能将Color类型
                                                      // 和double类型比较
      auto factors =                                  // 出错！不能将Color类型传递给
        primeFactors(c);                              // 参数类型为std::size_t的函数
    ...
 	}
```
如果你就是想将`Color`类型转换为一个其他类型，使用类型强制转换（`cast`）可以满足你这种变态的需求:
```cpp
	if(static_cast<double>(c) < 14.5) {              // 怪异但是有效的代码

	auto factors =                                   // 感觉不可靠
	  primeFactors(static_cast<std::size_t(c));      // 但是可以编译
	...
}
```
相较于无定义域的`enum`，有定义域的`enum`也许还有第三个优势，因为有定义域的`enum`可以被提前声明的，即可以不指定枚举元素而进行声明:
```cpp
	enum Color;                                      // 出错！

	enum class Color;                                // 没有问题
```
这是一个误导。在`C++11`中，没有定义域的`enum`也有可能被提前声明，但是需要一点额外的工作。这个工作时基于这样的事实：`C++`中的枚举类型都有一个被编译器决定的潜在的类型。对于一个无定义域的枚举类型像`Color`,
```cpp
    enum Color {black, white, red };
```
编译器有可能选择`char`作为潜在的类型，因为仅仅有三个值需要表达。然而一些枚举类型有很大的取值的跨度，如下：
```cpp
	enum Status { good = 0,
                  failed = 1,
                  incomplete = 100,
                  corrupt = 200,
                  indeterminate = 0xFFFFFFFF
                };
```
这里需要表达的值范围从`0`到`0xFFFFFFFF`。除非是在一个不寻常的机器上（在这台机器上，`char`类型至少有`32`个`bit`），编译器一定会选择一个取值范围比`char`大的整数类型来表示`Status`的类型。

为了更高效的利用内存，编译器通常想为枚举类型选择可以充分表示枚举元素的取值范围但又占用内存最小的潜在类型。在某些情况下，为了代码速度的优化，可以回牺牲内存大小，在那种情况下，编译器可能不会选择占用内存最小的可允许的潜在类型，但是编译器依然希望能过优化内存存储的大小。为了使这种功能可以实现，`C++98`仅仅支持枚举类型的定义（所有枚举元素被列出来），而枚举类型的声明是不被允许的。这样可以保证在枚举类型被用到之前，编译器已经给每个枚举类型选择了潜在类型。

不能事先声明枚举类型有几个不足。最引人注意的就是会增加编译依赖性。再次看看`Status`这个枚举类型：
```cpp
	enum Status { good = 0,
                  failed = 1,
                  incomplete = 100,
                  corrupt = 200,
                  indeterminate = 0xFFFFFFFF
                };
```
这个枚举体可能会在整个系统中都会被使用到，因此被包含在系统每部分都依赖的一个头文件当中。如果一个新的状态需要被引入：
```cpp
	enum Status { good = 0,
                  failed = 1,
                  incomplete = 100,
                  corrupt = 200,
                  audited = 500,
                  indeterminate = 0xFFFFFFFF
                };
```
就算一个子系统——甚至只有一个函数！——用到这个新的枚举元素，有可能导致整个系统的代码需要被重新编译。这种事情是人们憎恨的。在`C++11`中，这种情况被消除了。例如，这里有一个完美的有效的有作用域的`enum`的声明，还有一个函数将它作为参数：
```cpp
	enum class Status;                  // 前置声明

	void continueProcessing(Status s);  // 使用前置声明的枚举体
```
如果`Status`的定义被修改，包含这个声明的头文件不需要重新编译。更进一步，如果`Status`被修改（即，增加`audited`枚举元素），但是`continueProcessing`的行为不受影响（因为`continueProcessing`没有使用`audited`），`continueProcessing`的实现也不需要重新编译。

但是如果编译器需要在枚举体之前知道它的大小，`C++11`的枚举体怎么做到可以前置声明，而`C++98`的枚举体无法实现？原因是简单的，对于有作用域的枚举体的潜在类型是已知的，对于没有作用域的枚举体，你可以指定它。

对有作用域的枚举体，默认的潜在的类型是`int`:
```cpp
	enum class Status;                  // 潜在类型是int
```
如果默认的类型不适用于你，你可重载它：
```cpp
	enum class Status: std::uint32_t;   // Status潜在类型是
	                                    // std::uint32_t
	                                    // （来自<cstdint>）
```
无论哪种形式，编译器都知道有作用域的枚举体中的枚举元素的大小。

为了给没有作用域的枚举体指定潜在类型，你需要做相同的事情，结果可能是前置声明：
```cpp
	enum Color: std::uint8_t;          // 没有定义域的枚举体
	                                   // 的前置声明，潜在类型是
	                                   // std::uint8_t
```
潜在类型的指定也可以放在枚举体的定义处：
```cpp
	enum class Status: std::uint32_t{ good = 0,
	                                  failed = 1,
	                                  incomplete = 100,
	                                  corrupt = 200,
	                                  audited = 500,
	                                  indeterminate = 0xFFFFFFFF
	                                };
```
从有定义域的枚举体可以避免命名空间污染和不易受无意义的隐式类型转换影响的角度看，你听到至少在一种情形下没有定义域的枚举体是有用的可能会感到惊讶。这种情况发生在引用`C++11`的`std::tuples`中的某个域时。例如，假设我们有一个元组，元组中保存着姓名，电子邮件地址，和用户在社交网站的影响力数值：
```cpp
	using UserInfo =                 // 别名，参见条款9
	  std::tuple<std::string,        // 姓名
	  	         std::string,        // 电子邮件
	  	         std::size_t> ;      // 影响力
```	  	         
尽管注释已经说明元组的每部分代表什么意思，但是当你遇到像下面这样的源代码时，可能注释没有什么用：
```cpp
	UserInfo uInfo;                  // 元组类型的一个对象
	...

	auto val = std::get<1>(uInfo);   // 得到第一个域的值
```
作为一个程序员，你有很多事要做。你真的想去记住元组的第一个域对应的是用户的电子邮件地址？我不这么认为。使用一个没有定义域的枚举体来把名字和域的编号联系在一起来避免去死记这些东西：
```cpp
	enum UserInfoFields {uiName, uiEmail, uiReputation };

	UserInfo uInfo;                         // 和前面一样
	...

	auto val = std::get<uiEmail>(uInfo);    // 得到电子邮件域的值
```
上面代码正常工作的原因是`UserInfoFields`到`std::get()`要求的`std::size_t`的隐式类型转换。

如果使用有作用域的枚举体的代码就显得十分冗余：
```cpp
	enum class UserInfoFields { uiName, uiEmail, uiReputaion };

	UserInfo uInfo;                        // 和前面一样
	...

	auto val = 
	  std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(uInfo); 
```
写一个以枚举元素为参数返回对应的`std::size_t`的类型的值可以减少这种冗余性。`std::get`是一个模板，你提供的值是一个模板参数（注意用的是尖括号，不是圆括号），因此负责将枚举元素转化为`std::size_t`的这个函数必须在编译阶段就确定它的结果。就像条款15解释的，这意味着它必须是一个`constexpr`函数。

实际上，它必须是一个`constexpr`函数模板，因为它应该对任何类型的枚举体有效。如果我们打算实现这种一般化，我们需要一般化返回值类型。不是返回`std::size_t`，我们需要返回枚举体的潜在类型。通过`std::underlying_type`类型转换来实现（关于类型转换的信息，参见条款9）。最后需要将这个函数声明为`noexcept`（参见条款14），因为我们知道它永远不会触发异常。结果就是这个函数模板可以接受任何的枚举元素，返回这个元素的在编译阶段的常数值：
```cpp
	template<typename E>
	constexpr typename std::underlying_type<E>::type
	  toUType(E enumerator) noexcept
	{
	  return
	    static_cast<typename 
	                std::underlying_type<E>::type>(enumerator);
    }
```
在`C++14`中，`toUType`可以通过将`std::underlying_type<E>::type`替代为`std::underlying_type_t`（参见条款9）:
```cpp
	template<typename E>                           // C++14
	constexpr std::underlying_type_t<E>
	  toUType(E enumerator) noexcept
	{
	  return static_cast<std::underlying_type_t<E>>(enumerator);
    }
```
更加优雅的`auto`返回值类型（参见条款3）在`C++14`中也是有效的：
```cpp
	template<typename E>
	constexpr auto
	  toUType(E enumerator) noexcept
	{
	  return static_cast<std::underlying_type_t<E>>(enumerator);
	}
```
无论写哪种形式，`toUType`允许我们想下面一样访问一个元组的某个域：
```cpp
	auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```
这样依然比使用没有定义域的枚举体要复杂，但是它可以避免命名空间污染和不易引起注意的枚举元素的的类型转换。很多时候，你可能会决定多敲击一些额外的键盘来避免陷入一个上古时代的枚举体的技术陷阱中。

|要记住的东西|
|:--------- |
|`C++98`风格的`enum`是没有作用域的`enum`|
|有作用域的枚举体的枚举元素仅仅对枚举体内部可见。只能通过类型转换（`cast`）转换为其他类型|
|有作用域和没有作用域的`enum`都支持指定潜在类型。有作用域的`enum`的默认潜在类型是`int`。没有作用域的`enum`没有默认的潜在类型。|
|有作用域的`enum`总是可以前置声明的。没有作用域的`enum`只有当指定潜在类型时才可以前置声明。|