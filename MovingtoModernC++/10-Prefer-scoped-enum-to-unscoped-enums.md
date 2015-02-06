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
