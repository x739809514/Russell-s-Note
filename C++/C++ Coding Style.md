## 命名规范

只能给个大致的、合理的、清晰的命名规范。全球范围内重要的标准机构或者企业，基本都有自己的一套，谁也说服不了谁，这个也没法强制，能做到清晰明了就行。坚决抵制拼音、abcd等名字。全首字母缩写（acronym）也要是一个共识的时候才使用，如RAII，RVO，CRTP这些多数人都能明白；自己随便想出来的，别用，没人看得懂。

| **代码元素** | **命名风格** | **注释** |
| --- | --- | --- |
| Namespace | under\_scored | 为了跟类名做区分 |
| Class name | CamelCase | 为了跟标准库的类名做区分 (建议不要使用大写"C" 或者 "T" 作为前缀) |
| Function name | camelCase | 小写开头的函数名基本是通用的，除了.Net自成一格 |
| Parameters/Locals | under\_scored | 这个在c++世界里面是占大多数的 |
| Member variables | under\_scored\_with\_ | "\_"作为前缀在c++标准里面是不建议的，所以使用后缀 |
| Enums and its members | CamelCase | 除了很老的标准外，大多数都支持这种风格 |
| Globals | g\_under\_scored | 你一开始就不该写全局变量！ |
| File names | 与类名相符 | 优劣参半 |

## 代码规范

### 条款1——重要的Specifiers

1. override：重写基类的方法时，必须在函数后面加上override。
2. const：类的方法没有修改类成员时，必须在函数后面加上const。比较典型的是所有的Getter函数。
3. noexcept：当确定类的方法不会抛出异常时，尽可能加上noexcept。

### 条款2——try catch
当你不知道具体要catch什么异常时，就不要try了。

### 条款3——常量
定义常量，尤其是数字常量，禁止使用#define，一律用constexpr或者static constexpr。
Difference between `const` & ` constexpr`:

	1. const variable can't be changed in runtime
	1. ` constexpr` variable is already knowed in compile time.

```c++
// this variable is already knew when compile
constexpr float STEP = 0.02f;
constexpr size_t MAX_SIZE = 100000;
```

### 条款4——尽可能使用显式类型转换
```c++
int a = -10;
size_t b = 5;
auto c = a / b;
std::cout << "c = " << c << std::endl;
// c = 3689348814741910321

auto d = a / static_cast<int>(b);
std::cout << "d = " << d << std::endl;
// d = -2
```
在C++中，有四种主要的类型转换操作符，每种都有其特定的目的和适用场景：
1. static_cast
	* 用途：主要用于非多态类型的转换，包括基本类型之间的转换（如int到float），以及类层次结构中的向上或向下转型（但不包括安全地向下转型到具体派生类）。它还可以用于去除const、volatile属性，但在这种情况下不是类型的安全转换。
	* 示例：
```c++
int i = 10;
// 基本类型转换
float f = static_cast<float>(i); 
// 向下转型，需要保证basePtr实际指向的是Derived类型对象
Derived* d = static_cast<Derived*>(basePtr); 
const int& cref = ...;
// 不安全地去除了const属性
int& nonConstRef = static_cast<int&>(cref); 
```
2. dynamic_cast
	* 用途：用于运行时检查的类型转换，常用于多态类型的对象之间安全地向下转型。如果转换无法成功，动态_cast会返回nullptr（对于指针）或抛出std::bad_cast异常（对于引用）。
	* 示例：
```c++
class Base {};
class Derived : public Base {};

Base* basePtr = new Derived();
// 安全向下转型，如果basePtr指向的是Base以外的类型，则derivedPtr为nullptr
Derived* derivedPtr = dynamic_cast<Derived*>(basePtr); 
```
3. reinterpret_cast
	* 用途：最不安全的类型转换之一，它主要用于底层的比特级转换，例如将一个指针转换为整数或者不同类型的指针之间转换。这种转换可能导致未定义行为，除非你知道你正在做什么并且满足一定的条件（如两个指针类型具有兼容的底层类型）。
	* 示例：
```c++
void* voidPtr = &someValue;
// 风险转换，仅当voidPtr原本就是int型变量的地址时才是安全的
int* intPtr = reinterpret_cast<int*>(voidPtr); 
```
4. const_cast
	* 用途：唯一用来改变表达式或对象的常量性或挥发性的类型转换。它可以添加或移除const、volatile属性，但不能改变对象的实际类型。
	* 示例：
```c++
const int* constIntPtr = ...;
// 移除了const属性，但这并不意味着现在可以修改通过mutableIntPtr指向的数据
int* mutableIntPtr = const_cast<int*>(constIntPtr); 
```
总结来说：
* static_cast用于编译期已知且通常安全的转换，但需确保转换合理有效。
* dynamic_cast用于在运行时检查类型并安全地进行多态类型的转换。
* reinterpret_cast用于低级别的比特级转换，非常危险，应谨慎使用。
* const_cast仅用于改变常量属性，但不能更改类型，并且改变常量属性后的操作必须遵守原有的逻辑约束。
### 条款5——auto关键字
能用auto的地方，多用auto，特别是很长的、一层套一层的类型，比如迭代器。
```c++
std::vector<OpenMesh::Point<float, 4>>::Iterator iter = vec.begin(); // no good
// prefer
auto iter = vec.begin();
```

### 条款6——Type Alias
使用using关键词来创建别名，停止使用typedef，just for consistency.
```c++
using AliasType = Type;
// typedef Type AliasType;
```

### 条款7——stl算法
尽可能使用stl里面的算法，不要自己手写相同功能的算法，不仅很难写的比stl好，还容易有bug，不好排查。
```c++
// 如求和
int sum = 0;
for(auto i : vec){
	sum += i;
}
// maybe prefer std::accumulate or std::reduce
std::accumulate(vec.begin(), vec.end(), 0, [](auto value, auto item){return value + item;});
// or
std::reduce(vec.begin(), vec.end(), 0, [](auto value, auto item){return value + item;});
// or even parallelism
std::reduce(std::execution::par, vec.begin(), vec.end(), 0, [](auto value, auto item){return value + item;});

```

### 条款8——stl容器

1. 使用合适的容器做合适的事情。
2. 如果你不知道用哪个容器合适，就用vector。
3. 如果知道vector将会用到的大小，先reserve。
   1. <u>reserve used to get a specifical num of space, it call "alloc" once. `Reserve(1000)` This alloc a `continue` space for 1000 elements.</u>
   2. <u>In contract, if use vector. Push_back(), `push_back(1000)` it calls alloc 1000 times. It's cost.</u>
   3. <u>Resize(n), this func alloc n elements of space, and it has initial val, their all 0. But `Reserve(n)` doesnt have initial val.</u>

4. 尽量使用std::array，而不是原生数组，std::array可以直接使用stl算法，并且没有额外开销。
5. 如果键值顺序无所谓，优先使用unordered_map。

### 条款9——传值
```c++
// 需要修改
auto func(T& v) -> void;

// 只读，sizeof(T) >= 16，经验值
// 当T是float时，如果传const T&就很搞笑了，本身就只占4byte，现在变成了8byte（64位计算机）
auto func(const T& v) -> void; 

// 只读，sizeof(T) < 16
auto func(T v) -> void;

// 虽然没错，但没必要
auto func(const T v) -> void;
```

### 条款10——返回值
```c++
// 返回局部变量，禁止返回引用。
auto func() -> T
{
	T val;
	// ...
	// RVO
	return val;
}

// 返回成员变量
struct A
{
	T val_;
	// 如果sizeof(T) < 16，经验值
	auto getValue() const noexcept -> T
	{
		return val_;
	}

	// 根据接收类型，会选择拷贝或者引用。
	auto getValue2() const noexcept -> const T&
	{
		return val_;
	}
};
```

### <u>条款11——默认参数</u>
虽然个人不建议使用默认参数，但是不可否认默认参数的确一定程度上提供了不少方便。但是有一种情况下，坚决抵制使用默认参数，那就是基类虚函数。
```c++
// 动态绑定会出现问题，不明白原理，会造成难以排查的错误。
class Base
{
public:
	virtual auto setValue(int v = 10) noexcept -> void
	{
		// ...
	}
}
class Derived: public Base
{
public:
	auto setValue(int v = 20) noexcept -> void override
	{
		// ...
	}
}
```

此外，可变引用不要带默认参数，虽然msvc能编过，但是标准不支持，是未定义行为。
```c++
auto func(double& in_out = 5.f); // 达咩！！！
```

### 条款12——内存
1. 优先栈分配，除非你能找到充分的理由向操作系统申请堆内存。
2. 当申请堆内存时，尽量避免裸指针，除非你很强。
3. 使用智能指针，unique_ptr优于shared_ptr，除非你找到充分的理由使用shared_ptr。

### 条款13——Trailing Comma
在数组或者枚举中，建议加上Trailing Comma。
```c++
std::vector<int> v{0,1,2,3,};

// 特别是枚举，强烈建议加上Trailing Comma，会方便很多。
enum class Color
{
	Red,    ///< comment
	Green,  ///< comment
	Blue,   ///< comment
	Alpha,  ///< comment
}
```

### 条款14——使用nullptr而不是NULL
在c++代码中，空指针应该尽可能使用nullptr，而非NULL。
```c++
#define NULL 1 // 使用NULL你就G了。

// 如果有两个重载函数，一个接收integer，一个接收指针。
auto fun(int) -> { std::print("{}", "This is an integer");}
auto fun(int*) -> { std::print("{}", "This is a pointer to an integer");}

auto p_int = NULL;
fun(p_int); // the first overload will be called.

auto p_int = nullptr;
fun(p_int); // the second overload will be called, which is generally expected.
```

### 条款15——不要把子线程detach掉，无论出于何种原因。
你找不到任何合情合理，并且负责任的理由，把子线程detach掉。将线程detach掉只会增加后续维护的困难度。C++ Core Guidelines也不建议使用detach。

>  CP.26: Don’t detach() a thread
> This rule sounds strange. The C++11 standard supports detaching a thread, but we should not do it! The reason is that detaching a thread can be quite challenging. As rule, C.25 said: CP.24: Think of an thread as a global container. Of course, this means you are magnificent if you use only variables with global scope in the detached threads. NO! Even objects with static duration can be critical. For example, look at this small program with undefined behavior.
```c++
#include <iostream>  
#include <string>  
#include <thread>  
  
void func(){
  std::string s{"C++11"};
  std::thread t([&s]{ std::cout << s << std::endl;});
  t.detach();
}  
  
int main(){  
  func();  
}
```
> This is easy. The lambda function takes s by reference. This is undefined behavior because the child thread t uses the variable s, which goes out of scope. STOP! This is the apparent problem but the hidden issue is std::cout. std::cout has a static duration. This means the lifetime of std::cout ends with the end of the program, and we have, additionally, a race condition: thread t may use std::cout at this time. 

### <u>条款16——使用RAII的lock，不要直接lock和unlock</u>
一般来说，要找到一个合适的unlock时机是不容易的，如果直接调用lock和unlock，很可能会导致程序死锁。
```c++
// 不推荐的方式
auto fun() -> int
{
	mutex.lock();
	// your code
	// 此处可能会提前return，可能会抛出异常，这种情况下，mutex没有unlock，程序就死锁了。
	mutex.unlock();
	return 0;
}

// 推荐的方式
auto fun() -> int
{
	std::lock_guard<Mutex> guard(mutex);
	// your code
	// 此处无论是否提前return，或者抛出异常，都可以保证unlock会被调用。
	return 0;
}

// 需要手动解锁的时候
auto fun() -> int
{
	std::unique_lock<Mutex> u_lock(mutex);
	// your code
	u_lock.unlock();
	// your code
	u_lock.lock();
	// your code
	return 0;
}

// c++17 or later
auto fun() -> int
{
	// scoped_lock相较于lock_guard，支持多把锁。
	std::scoped_lock s_lock(mutex1, mutex2, mutex3);
	// your code

	return 0;
}
```

### 条款17——重视编译器警告
要特别重视编译器警告，不能当做没有看到。有些警告特别致命，相当于编译器告诉你，这里十有八九出错了，但是根据标准我不能管你，只能提示你，你自己看着办。这种类型的警告建议直接当做错误处理，我在这里列出了一些：
```cmake
/we4172 # 返回局部变量或临时变量的地址
/we4715 # 不是所有的控件路径都返回值
/we4265 # 类包含虚函数，但其不常用的析构函数不是虚函数；该类的实例可能无法进行正确析构
/we4390 # 找到空的受控语句；这是否是有意的?
/we4146 # 一元负运算符应用于无符号类型，结果仍为无符号类型
/we4308 # 负整型常量转换为无符号类型
/we4700 # 使用了未初始化的局部变量
/we4703 # 使用了可能未初始化的本地指针变量
/we4365 # “参数”: 从“int”转换到“size_t”，有符号/无符号不匹配
/we4245 # 从常量“int”转换到“size_t”，有符号/无符号不匹配
```
举例：
```c++
auto func() -> int&
{
	//we4172 # 返回局部变量或临时变量的地址
	int a = 0;
	return a;
}

auto func(bool b) -> int
{
	//we4715 # 不是所有的控件路径都返回值
	if(b){
		return 1;
	}
}

class VClass
{
public:
	virtual auto help() -> void = 0;
	~VClass(){}
	//we4265 # 类包含虚函数，但其不常用的析构函数不是虚函数；该类的实例可能无法进行正确析构
};

auto func(bool b)
{
	if(b);
	...
	//we4390 # 找到空的受控语句；这是否是有意的?
}

auto func()
{
	size_t a = 10; // unsigned
	int b = -9;
	//we4146 # 一元负运算符应用于无符号类型，结果仍为无符号类型
	if(b > -a) 
	{
		
	}
}

auto func()
{
	unsigned int a = -10;
	//we4308 # 负整型常量转换为无符号类型
}

auto func()
{
	int a;
	++a;
	//we4700 # 使用了未初始化的局部变量
}

auto func(size_t size)
{
	int* p;
	if(size > 255)
	{
		p = new int(10);
	}
	if(p)
	{
		delete p;
	}
	//we4703 # 使用了可能未初始化的本地指针变量
	// 解决：int* p{nullptr};
}

auto func()
{
	int a = -10;
	unsigned int b = a;
	//we4365 # “参数”: 从“int”转换到“size_t”，有符号/无符号不匹配
}

auto func()
{
	const int a = -10;
	unsigned int b = a;
	//we4245 # 从常量“int”转换到“size_t”，有符号/无符号不匹配
}
```

### 条款18——尽量使用标准c++，避免使用编译器的扩展功能，除非几个主流的编译器都支持的扩展功能。
使用MSVC的时候，可以加上/permissive-编译选项，强制让编译器遵守c++标准。

### 条款19——重写虚函数必须使用override关键字
```c++
class Base
{
public:
	virtual auto listen(int k) -> void = 0;
	virtual ~Base(){}
};

class Derived: public Base
{
public:
	auto listen(int k) -> void override
	{
		// do something
	}
};
```
<u>如果不写override，那么当Base类把listen的接口改变的时候，编译器不会报错，甚至多数编译器连警告都不会有，这个时候程序运行的结果就会出错。一旦写override，那么当Base类把listen的接口改变的时候，编译就会失败，就很容易修改了。</u>

### 条款20——使用PImpl惯用法
使用Pimpl（Pointer to implementation）惯用法有两个显而易见的好处：
1. 起到编译防火墙的作用，减少编译期的依赖。
2. 增强二进制兼容（Binary Compatibility）能力。在PImpl类里面增加新的字段或者改变字段的顺序，都不会影响二进制兼容。

<u>Seems like a component system, that uses a unique_ptr to store a class reference, then implements other components in this implement class, in order to hide references.</u>



### 条款21——命名空间规范
1. 原则上禁止在公共接口头文件里面使用using namespace std；
2. 使用using namespace std要遵循“最小影响范围原则”，即要尽可能缩小using namespace std的影响范围。
3. 建议用到什么，using什么。如要使用vector和shared_ptr，那么应该尽量使用using std::vector和using shared_ptr，而不是直接using namespace std。

### 条款22——enum和enum class
1. 作用域（Scoping）
	1. enum：枚举成员（枚举值）的作用域与枚举类型是共享的。这意味着，一旦定义了一个枚举类型，其枚举值可以在包含它的整个作用域内被直接访问，这可能导致命名冲突
	2. enum class：每个枚举成员的作用域限定在枚举类型内部，因此<u>必须通过枚举类型名进行限定来访问，如 EnumClass::ValueName。这样可以避免不同枚举类型之间的命名冲突</u>
2. 类型安全
	1. enum：默认情况下，枚举值可以隐式转换为整数类型，并且整数可以隐式转换回对应的枚举值。这种行为可能会导致潜在的类型安全问题，尤其是在比较、赋值操作时。
	2. <u>enum class：不支持隐式类型转换。枚举值不能隐式地转换为整数，反之亦然。如果需要进行这样的转换，必须显式地进行类型转.</u>
	
	   Different betwen enum and enum class
	   	1. Enum allows implicit conversion , but enum class does not
	   	1. Enum doesn't has scope, but enum class has
	   	1. Enum class can avoid name conflict
	   	1. Enum is not type-safe, but enum is not type-safe
3. 语法和可读性
	1. 使用 enum class 可以提供更好的代码可读性和意图表达，因为它明确指出了枚举值属于哪个枚举类型，增强了代码的自文档化能力。
	推荐用法：
* 如果你希望枚举值在整个作用域内可见并且能够方便地与其他整数值交互，同时对类型安全要求不高，可以选择使用传统的 enum。
* 对于大多数现代编程实践和面向对象设计原则来说，更推荐使用 enum class，因为它能帮助防止命名空间污染并提供更强的类型安全性，从而减少程序中的潜在错误。尤其当你的枚举类型可能与其它地方使用的整数值有重叠或交互风险时，应当优先考虑 enum class。

### 条款23——#include<>和#include""
在C++中，#include指令用于将一个头文件的内容插入到当前源文件中。使用尖括号 <> 和双引号 "" 区别如下：
1. 尖括号 < >：
	1. 当你想要包含标准库提供的头文件时，应该使用尖括号。编译器会在预定义的系统目录列表中查找这些头文件。
2. 双引号 " "：
	1. 当你希望包含自定义的项目特定或第三方库的头文件时，通常使用双引号。编译器首先会在当前源文件所在的目录下搜索，然后按照编译命令指定的其他包含路径进行查找
	2. 示例：如果你有一个自定义头文件 my_header.h，你可能会这样包含它：#include "my_header.h"。
	建议：
1. 对于所有标准库的头文件，始终使用尖括号形式。
2. 对于项目内部和自定义的头文件，使用双引号形式。
3. 如果你的项目依赖于外部库，并且该库提供了自定义头文件，按照该库文档的指示来决定是否使用尖括号（如果库文档明确指出需要在系统路径中查找）或者双引号（如果库要求将其安装在特定目录并在编译选项中添加相应包含路径）。
4. 在项目结构设计上，尽量组织好自定义头文件的目录结构，并通过编译选项 -I 或者 CMakeLists.txt 中的 include_directories() 指令正确地设置包含路径，确保即使使用双引号也能找到对应的头文件。

### 条款24——Lakos Include Order
C++ 的Lakos Include Order是由Steve Lakos在其著作《Large-Scale C++ Software Design》中提出的一种头文件包含顺序的策略。该策略的主要目的是减少编译时间和避免编译错误，特别是由于循环依赖和未定义符号导致的问题。
Lakos提出的头文件包含顺序原则可以概括为以下几点：
1. 自包含原则：每个头文件应尽可能地自包含（self-contained），即它所依赖的所有内容都应当通过内部包含或前向声明来解决。
2. 项目内部头文件优先：
	1. 首先包含当前模块（源文件对应的头文件）
	2. 然后按模块间的依赖关系，从低层次到高层次依次包含其他项目内部头文件。
3. 标准库与系统头文件：
	1. 在包含所有项目内部头文件之后，再包含C++标准库和其他系统提供的头文件。
4. 第三方库头文件：
	1. 最后才包含第三方库的头文件。

Lakos Include Order的主要好处在于减少编译时间、避免不必要的重新编译，并且解决由于头文件依赖关系引起的潜在问题，具体包括：
1. 降低编译依赖性：
	1. 通过确保每个头文件自包含或者先于使用它的头文件包含其依赖的其他头文件，可以最小化模块间的耦合度。这有助于防止因无意更改一个头文件而触发大量不相关源文件的重新编译。
2. 预防编译循环和未定义引用错误：
	1. 如果遵循从项目内部到外部（第三方库）再到系统头文件的顺序，可以更容易地发现并解决循环依赖问题。同时，由于在使用某个类型或函数之前，对应的声明已经存在，所以可以避免“未定义引用”的链接错误。
3. 提高代码可读性和维护性：
	1. 统一的头文件组织和包含顺序使得阅读和理解代码更加容易，同时也便于团队成员遵循一致的编码规范，从而提高项目的整体维护性。
4. 支持增量编译：
	1. 合理的头文件包含顺序能够最大程度地利用现代构建工具（如Makefile、CMake等）的增量编译特性，当仅修改了项目内部的头文件时，只重新编译直接或间接依赖这些头文件的源文件，而不必全局重新编译。
5. 简化预处理器宏管理：
	1. 避免预处理器宏的冲突或意外覆盖，因为系统和第三方库通常会定义一些全局作用域的宏，如果这些宏在包含它们的头文件之前被项目内部代码修改，可能会导致不可预见的问题。
	尽管随着C++语言特性的增加（比如命名空间、模板以及模块化编程等），以及编译器性能和智能程度的提升，Lakos提出的策略可能需要结合实际情况进行调整，但其基本原则对于大型软件项目的设计和维护仍然具有重要的指导意义。

### 条款25——switch和[[fallthrough]]
`[[fallthrough]]`是`C++17`标准引入的一个编译器注解，用于处理`switch`语句中的fallthrough行为。它允许程序员明确表示在执行完某个`case`的代码后，希望程序继续执行下一个`case`而不是停止。这有助于避免因为忘记添加`break`语句而引起的意外情况，同时增加了代码的可读性和清晰度。以下是关于`[[fallthrough]]`的一些关键点：
1. 明确意图：
   - 每个`case`语句块的末尾都应该有一个`break`语句，除非你确实想要fallthrough到下一个`case`。
   - 如果需要fallthrough，使用`[[fallthrough]]`注解来明确声明这一意图。这样可以消除因为忘记`break`而产生的意外行为，并提高代码可读性。
2. 避免无条件fallthrough：
	- 除非在设计上是必要的，否则避免无条件的fallthrough，因为这可能导致逻辑混淆。每个`case`应该视为独立的逻辑分支，除非有明确的注解说明它们应该一起执行。
3. 使用default case：
	- 总是包含一个`default`case作为最后的备选，以处理未匹配到任何`case`的情况。
4. 保持case简洁：
	- 尽量避免在`case`体内包含复杂的逻辑，尤其是跨越多个语句的逻辑。如果`case`代码过于复杂，考虑重构为函数。
5. 注释：
	- 对于使用`[[fallthrough]]`的地方，提供简短的注释说明为什么这样做以及fallthrough后会发生什么。

```C++
   switch (expr) {
       case value1:
           // do something
           [[fallthrough]]; // 显式声明 fallthrough
       case value2:
           // 这里将被执行，因为前面有 fallthrough
   }
```

## 代码优化建议
### 条款1——Ahmdal定律
$$ SpeedUp=\frac{1}{(1-func_{cost})+func_{cost}/func_{speedup}} $$
也就是说，如果某个函数占50%的运行时间，你把他速度提升到了原来的两倍，那么根据这个公式，$$ SpeedUp = \frac{1}{(1-0.5)+0.5/2}=\frac{1}{0.75}=\frac{4}{3}=1.33\overline{3} $$也就是整体提速了1.333倍。

### 条款2——先保证正确，再进行优化
> Premature optimization is the root of all evil.  —— Donald Knuth
> 过早优化是万恶之源

### 条款3——使用likely和unlikely
如果无法避免地要使用条件分支，那么做好用上likely和unlikely。
```c++
if( a > 3 ) [[likely]] {
    std::cout<<"a is greater then 3"<<std::endl;
}else[[unlikely]]{
    std::cout<<"a is smaller then 3"<<std::endl;
}
```
上面的例子中，如果a>3的概率远大于a<=3的概率，那么加上likely和unlikely会帮助编译器更好地优化。

### 条款4——把多次访问的本地变量放到register中
```c++
auto func()
{
	register int a = 1;
	while(condition)
	{
		++a;
	}
}
```
当然，编译器可能已经帮你优化了，don't bother.
