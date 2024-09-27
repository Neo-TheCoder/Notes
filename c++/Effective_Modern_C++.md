# 第1章 类型推导
C++98有一套`类型推导`的规则：用于`函数模板`的规则。
C++11修改了其中的一些规则并增加了两套规则，一套用于`auto`，一套用于`decltype`。
`C++14`扩展了`auto`和`decltype`可能使用的范围。
类型推导的广泛应用，让你从拼写那些或明显或冗杂的类型名的暴行中脱离出来。
它让C++程序更具适应性，因为`在源代码某处修改类型会通过类型推导自动传播到其它地方`。
但是类型推导也会让代码更复杂，因为由编译器进行的类型推导并不总是如我们期望的那样进行。
如果对于类型推导操作没有一个扎实的理解，要想写出有现代感的C++程序是不可能的。
类型推导随处可见：在函数模板调用中，在大多数auto出现的地方，在decltype表达式出现的地方，以及C++14中令人费解的应用decltype(auto)的地方。
这一章是每个C++程序员都应该掌握的知识。它解释了`模板类型推导是如何工作的`，`auto是如何依赖类型推导的`，以及`decltype是如何按照它自己那套独特的规则工作的`。
它甚至解释了你该如何强制编译器使类型推导的结果可视，这能让你确认编译器的类型推导是否按照你期望的那样进行。

# 条款1：理解模板类型推导
对于一个复杂系统的用户来说，很多时候他们最关心的是它做了什么而不是它怎么做的。
在这一点上，C++中的模板类型推导表现得非常出色。
数百万的程序员只需要向模板函数传递实参，就能通过编译器的类型推导获得令人满意的结果，尽管他们中的大多数在被逼无奈的情况下，对于传递给函数的那些实参是如何引导编译器进行类型推导的，也只能给出非常模糊的描述。
如果那些人中包括你，我有一个好消息和一个坏消息。
好消息是现在C++最重要最吸引人的特性auto是建立在模板类型推导的基础上的。
如果你满意C++98的模板类型推导，那么你也会满意C++11的auto类型推导。坏消息是当模板类型推导规则应用于auto环境时，有时不如应用于template时那么直观。
由于这个原因，真正理解auto基于的模板类型推导的方方面面非常重要。这项条款便包含了你需要知道的东西。
如果你不介意浏览少许伪代码，我们可以考虑像这样一个函数模板：
```cpp
template<typename T>
void f(ParamType param);
```
它的调用看起来像这样
```cpp
f(expr);                        //使用表达式调用f
```
在编译期间，编译器使用expr进行两个类型推导：
一个是针对`T`的，另一个是针对`ParamType`的。
这两个类型通常是不同的，因为ParamType包含一些`修饰`，比如`const`和`引用修饰符`。
举个例子，如果模板这样声明：
```cpp
template<typename T>
void f(const T& param);         //ParamType是const T&
```
然后这样进行调用

```cpp
int x = 0;
f(x);                           //用一个int类型的变量调用f
```
`T`被推导为`int`，ParamType却被推导为`const int&`

我们可能很自然的期望T和传递进函数的实参是相同的类型，也就是，T为expr的类型。
在上面的例子中，事实就是那样：x是int，T被推导为int。
但有时情况并非总是如此，T的类型推导不仅取决于`expr的类型`，也取决于`ParamType的类型`。这里有三种情况：
* `ParamType`是一个指针或引用，但不是通用引用（关于通用引用请参见Item24。在这里你只需要知道它存在，而且不同于左值引用和右值引用）
* `ParamType`是一个通用引用
* `ParamType`既不是指针也不是引用
我们下面将分成三个情景来讨论这三种情况，每个情景的都基于我们之前给出的模板：

```cpp
template<typename T>
void f(ParamType param);

f(expr);                        //从expr中推导T和ParamType
```

## 情景一：ParamType是一个`指针或引用`，但不是通用引用
最简单的情况是ParamType是一个指针或者引用，但非通用引用。
在这种情况下，类型推导会这样进行：
1. 如果expr的类型是一个引用，**忽略引用部分**
2. 然后expr的类型与ParamType进行`模式匹配`来决定T
举个例子，如果这是我们的模板，
```cpp
template<typename T>
void f(T& param);               //param是一个引用
```
我们声明这些变量，

```cpp
int x = 27;                       //x是int
const int cx = x;                 //cx是const int
const int& rx = x;                //rx是指向作为const int的x的引用
```
在不同的调用中，对param和T推导的类型会是这样：

```cpp
f(x);                           //T是int，param的类型是int&
f(cx);                          //T是const int，param的类型是const int&     ！！！
f(rx);                          //T是const int，param的类型是const int&
```
在第二个和第三个调用中，注意因为cx和rx被指定为const值，所以T被推导为`const int`，从而产生了`const int&`的形参类型。
这对于调用者来说很重要。
当他们传递一个const对象给一个引用类型的形参时，他们期望对象保持不可改变性，也就是说，形参是`reference-to-const`的。
这也是为什么将一个const对象传递给以T&类型为形参的模板安全的：**对象的`常量性constness`会被保留为`T`的一部分**。

在第三个例子中，注意即使rx的类型是一个引用，T也会被推导为一个非引用，这是因为**rx的`引用性`（reference-ness）在`类型推导`中会被`忽略`**。
这些例子只展示了左值引用，但是类型推导会如左值引用一样对待右值引用。
当然，右值只能传递给右值引用，但是在类型推导中这种限制将不复存在。

如果我们将f的形参类型T&改为`const T&`，情况有所变化，但不会变得那么出人意料。
cx和rx的constness依然被遵守，但是因为`现在我们假设param是reference-to-const，const不再被推导为T的一部分`：
```cpp
template<typename T>
void f(const T& param);         //param现在是reference-to-const

int x = 27;                     //如之前一样
const int cx = x;               //如之前一样
const int& rx = x;              //如之前一样

f(x);                           //T是int，param的类型是const int&
f(cx);                          //T是int，param的类型是const int&
f(rx);                          //T是int，param的类型是const int&
```
同之前一样，rx的reference-ness在类型推导中被忽略了。

如果param是一个指针（或者指向const的指针）而不是引用，情况本质上也一样：
```cpp
template<typename T>
void f(T* param);               //param现在是指针

int x = 27;                     //同之前一样
const int *px = &x;             //px是指向作为const int的x的指针

f(&x);                          //T是int，param的类型是int*
f(px);                          //T是const int，param的类型是const int*
```
到现在为止，你会发现你自己打哈欠犯困，因为C++的类型推导规则对引用和指针形参如此自然，书面形式来看这些非常枯燥。
所有事情都那么理所当然！那正是在类型推导系统中你所想要的。


## 情景二：ParamType是一个`通用引用`
模板使用`通用引用形参`的话，那事情就不那么明显了。
这样的形参被声明为像右值引用一样（也就是，在函数模板中假设有一个类型形参T，那么通用引用声明形式就是`T&&`)，它们的行为在传入左值实参时大不相同。
完整的叙述请参见Item24，在这有些最必要的你还是需要知道：
如果`expr`是`左值`，`T`和`ParamType`都会被推导为`左值引用`。
这非常不寻常：
- 第一，这是模板类型推导中唯一一种`T`被推导为`引用`的情况。
- 第二，虽然ParamType被声明为右值引用类型，但是最后推导的结果是左值引用。
如果`expr`是右值，就使用正常的（也就是情景一）推导规则
举个例子：
```cpp
template<typename T>
void f(T&& param);              //param现在是一个通用引用类型
        
int x = 27;                       //如之前一样
const int cx = x;                 //如之前一样
const int & rx = cx;              //如之前一样

f(x);                           //x是左值，所以T是int&，
                                //param类型也是int&

f(cx);                          //cx是左值，所以T是const int&，
                                //param类型也是const int&

f(rx);                          //rx是左值，所以T是const int&，
                                //param类型也是const int&

f(27);                          //27是右值，所以T是int，
                                //param类型就是int&&
```
`Item24`详细解释了为什么这些例子是像这样发生的。
这里关键在于 `通用引用的类型推导规则` 是 不同于普通的左值或者右值引用的。
尤其是，当通用引用被使用时，类型推导会区分`左值实参`和`右值实参`，但是对非通用引用时不会区分。

## 情景三：ParamType既不是指针也不是引用
当ParamType既不是指针也不是引用时，我们通过`传值（pass-by-value）`的方式处理：

```cpp
template<typename T>
void f(T param);                //以传值的方式处理param
```
这意味着无论传递什么param都会成为它的一份`拷贝`————一个完整的新对象。
事实上param成为一个新对象这一行为会影响T如何从expr中推导出结果。

和之前一样，如果expr的类型是一个引用，忽略这个引用部分
如果忽略expr的引用性（reference-ness）之后，expr是一个const，那就再忽略const。
如果它是volatile，也忽略volatile（volatile对象不常见，它通常用于驱动程序的开发中。关于volatile的细节请参见Item40）
因此
```cpp
int x = 27;                       //如之前一样
const int cx = x;                 //如之前一样
const int & rx = cx;              //如之前一样

f(x);                           //T和param的类型都是int
f(cx);                          //T和param的类型都是int
f(rx);                          //T和param的类型都是int
```
注意即使cx和rx表示const值，param也不是const。这是有意义的。
`param是一个完全独立于cx和rx的对象`————是cx或rx的一个拷贝。
具有常量性的cx和rx不可修改并不代表param也是一样。
这就是为什么expr的常量性constness（或易变性volatileness)在推导param类型时会被忽略：因为`expr不可修改 并不意味着 它的拷贝也不能被修改`。

认识到只有在传值给形参时才会忽略const（和volatile）这一点很重要，正如我们看到的，对于reference-to-const和pointer-to-const形参来说，expr的常量性constness在推导时会被保留。
但是考虑这样的情况，expr是一个const指针，指向const对象，expr通过传值传递给param：
```cpp
template<typename T>
void f(T param);                //仍然以传值的方式处理param

const char* const ptr =         //ptr是一个常量指针，指向常量对象 
    "Fun with pointers";

f(ptr);                         //传递const char * const类型的实参
```
在这里，解引用符号（`*`）的右边的const表示ptr本身是一个const：ptr不能被修改为指向其它地址，也不能被设置为null（解引用符号左边的const表示ptr指向一个字符串，这个字符串是const，因此字符串不能被修改）。
当ptr作为实参传给f，组成这个指针的每一比特都被拷贝进param。
像这种情况，ptr自身的值会被传给形参，根据类型推导的第三条规则，ptr自身的常量性constness将会被省略，所以`param`是`const char*`，也就是 `一个可变指针指向const字符串`。
在类型推导中，这个指针指向的数据的常量性constness将会被保留，但是当拷贝ptr来创造一个新指针param时，ptr自身的常量性constness将会被忽略。


## 数组实参
上面的内容几乎覆盖了模板类型推导的大部分内容，但这里还有一些小细节值得注意，比如`数组类型`不同于`指针类型`，虽然它们两个有时候是可互换的。
关于这个错觉最常见的例子是，**在很多上下文中数组会退化为指向它的第一个元素的指针**。
这样的退化允许像这样的代码可以被编译：
```cpp
const char name[] = "J. P. Briggs";     //name的类型是const char[13]

const char * ptrToName = name;          //数组退化为指针
```
在这里`const char*`指针ptrToName会由name初始化，而name的类型为`const char[13]`，这两种类型（const char*和const char[13]）是不一样的，但是由于数组退化为指针的规则，编译器允许这样的代码。

但要是一个数组传值给一个模板会怎样？会发生什么？
```cpp
template<typename T>
void f(T param);                        //传值形参的模板

f(name);                                //T和param会推导成什么类型?
```
我们从一个简单的例子开始，这里有一个函数的形参是数组，是的，这样的语法是合法的，
```cpp
void myFunc(int param[]);
```
但是数组声明会被视作指针声明，这意味着myFunc的声明和下面声明是等价的：
```cpp
void myFunc(int* param);                //与上面相同的函数
```
数组与指针形参这样的等价是C语言的产物，C++又是建立在C语言的基础上，它让人产生了一种数组和指针是等价的的错觉。

因为数组形参会视作指针形参，所以传值给模板的一个`数组类型`会被`推导为`一个`指针类型`。
这意味着在模板函数f的调用中，它的类型形参T会被推导为`const char*`：
```cpp
f(name);                        //name是一个数组，但是T被推导为const char*
```
但是现在难题来了，虽然函数不能声明形参为真正的数组，但是`可以接受指向数组的引用`！所以我们修改f为传引用：
```cpp
template<typename T>
void f(T& param);                       //传引用形参的模板
```
我们这样进行调用，
```cpp
f(name);                                //传数组给f
```
T被推导为了真正的数组！这个类型包括了`数组的大小`，在这个例子中T被推导为`const char[13]`，f的形参（该数组的引用）的类型则为`const char (&)[13]`。是的，这种语法看起来又臭又长，但是知道它将会让你在关心这些问题的人的提问中获得大神的称号。
有趣的是，可声明指向数组的引用的能力，使得我们可以创建一个模板函数来`推导出数组的大小`：
```cpp
//在编译期间返回一个数组大小的常量值（//数组形参没有名字，
//因为我们只关心数组的大小）
template<typename T, std::size_t N>                     //关于
constexpr std::size_t arraySize(T (&)[N]) noexcept      //constexpr
{                                                       //和noexcept
    return N;                                           //的信息
}                                                       //请看下面
```
在Item15提到将一个函数声明为`constexpr`使得结果在`编译期间可用`。
这使得我们可以用一个花括号声明一个数组，然后第二个数组可以使用第一个数组的大小作为它的大小，就像这样：
```cpp
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };             //keyVals有七个元素

int mappedVals[arraySize(keyVals)];                     //mappedVals也有七个
// 当然作为一个现代C++程序员，你自然应该想到使用std::array而不是内置的数组：

std::array<int, arraySize(keyVals)> mappedVals;         //mappedVals的大小为7
```
至于arraySize被声明为`noexcept`，会使得编译器生成更好的代码，具体的细节请参见Item14。

## 函数实参
在C++中不只是数组会退化为指针，`函数类型`也会退化为一个`函数指针`，我们对于数组类型推导的全部讨论都可以应用到函数类型推导和退化为函数指针上来。结果是：
```cpp
void someFunc(int, double);         //someFunc是一个函数，
                                    //类型是void(int, double)

template<typename T>
void f1(T param);                   //传值给f1

template<typename T>
void f2(T& param);                 //传引用给f2

f1(someFunc);                       // param被推导为指向函数的指针，
                                    // 类型是 void(*)(int, double)
f2(someFunc);                       // param被推导为指向函数的引用，
                                    // 类型是 void(&)(int, double)
```
这个实际上没有什么不同，但是如果你知道数组退化为指针，你也会知道函数退化为指针。
这里你需要知道：`auto`依赖于模板类型推导。
正如我在开始谈论的，在大多数情况下它们的行为很直接。
在通用引用中对于左值的特殊处理使得本来很直接的行为变得有些污点，然而，数组和函数退化为指针把这团水搅得更浑浊。
有时你只需要编译器告诉你推导出的类型是什么。这种情况下，翻到item4,它会告诉你如何让编译器这么做。

请记住：
* 在模板类型推导时，有引用的实参会被视为无引用，他们的引用会被忽略
* 对于通用引用的推导，左值实参会被特殊对待
* 对于传值类型推导，const和/或volatile实参会被认为是non-const的和non-volatile的
* 在模板类型推导时，数组名或者函数名实参会退化为指针，除非它们被用于初始化引用




# 条款2：理解auto类型推导
如果你已经读过Item1的模板类型推导，那么你几乎已经知道了auto类型推导的大部分内容，至于为什么不是全部，是因为这里有一个auto不同于模板类型推导的例外。
但这怎么可能？模板类型推导包括模板，函数，形参，但auto不处理这些东西啊。
你是对的，但没关系。
`auto类型推导` 和 `模板类型推导`有一个直接的`映射关系`。
它们之间可以通过一个 非常规范 非常系统化的 转换流程来转换彼此。
在Item1中，模板类型推导使用下面这个函数模板
```cpp
template<typename T>
void f(ParmaType param);
```
和这个调用来解释：
```cpp
f(expr);                        //使用一些表达式调用f
```
在f的调用中，编译器使用`expr`推导`T`和`ParamType`的类型。
当一个变量使用`auto`进行声明时，`auto`扮演了`模板中T`的角色，`变量`的`类型说明符`扮演了`ParamType`的角色。
废话少说，这里便是更直观的代码描述，考虑这个例子：
```cpp
auto x = 27;
```
这里x的类型说明符是`auto`自己，另一方面，在这个声明中：
```cpp
const auto cx = x;
```
类型说明符是`const auto`。另一个：
```cpp
const auto& rx = x;
```
类型说明符是`const auto&`。
在这里例子中要推导x，cx和rx的类型，编译器的行为看起来就像是`认为这里每个声明都有一个模板`，然后使用`合适的初始化表达式`进行调用：
以下代码试图说明的是：
auto与模板类型推导的映射：
```cpp
template<typename T>            //概念化的模板用来推导x的类型
void func_for_x(T param);

func_for_x(27);                 //概念化调用：
                                //param的推导类型是x的类型

template<typename T>            //概念化的模板用来推导cx的类型
void func_for_cx(const T param);

func_for_cx(x);                 //概念化调用：
                                //param的推导类型是cx的类型

template<typename T>            //概念化的模板用来推导rx的类型
void func_for_rx(const T& param);

func_for_rx(x);                 //概念化调用：
                                //param的推导类型是rx的类型
```
正如我说的，auto类型推导除了一个例外（我们很快就会讨论），其他情况都和模板类型推导一样。

Item1基于ParamType————在函数模板中param的类型说明符————的不同特征，把模板类型推导分成三个部分来讨论。
在使用auto作为类型说明符的变量声明中，类型说明符代替了ParamType，因此Item1描述的三个情景稍作修改就能适用于auto：
* 情景一：类型说明符是一个指针或引用但不是通用引用
* 情景二：类型说明符一个通用引用
* 情景三：类型说明符既不是指针也不是引用
我们早已看过情景一和情景三的例子：
```cpp
auto x = 27;                    //情景三（x既不是指针也不是引用）
const auto cx = x;              //情景三（cx也一样）
const auto & rx = cx;             //情景一（rx是非通用引用）, cx是const int类型，因此rx是const int&类型的左值
```
情景二像你期待的一样运作：
```cpp
auto&& uref1 = x;               //x是int左值，
                                //所以uref1类型为int&
auto&& uref2 = cx;              //cx是const int左值，
                                //所以uref2类型为const int&
auto&& uref3 = 27;              //27是int右值，
                                //所以uref3类型为int&&
```
Item1讨论并总结了对于`non-reference`类型说明符，数组和函数名如何退化为指针。
那些内容也同样适用于auto类型推导：
```cpp
const char name[] =             //name的类型是const char[13]
 "R. N. Briggs";

auto arr1 = name;               //arr1的类型是const char*，已经退化了，只有引用能够完整地保证数组具体类型
auto& arr2 = name;              //arr2的类型是const char (&)[13]

void someFunc(int, double);     //someFunc是一个函数，
                                //类型为void(int, double)

auto func1 = someFunc;          //func1的类型是void (*)(int, double)，退化了，变成指向函数的指针
auto& func2 = someFunc;         //func2的类型是void (&)(int, double)
```
就像你看到的那样，auto类型推导和模板类型推导几乎一样的工作，它们就像一个硬币的两面。
讨论完相同点接下来就是不同点，前面我们已经说到 auto类型推导 和 模板类型推导 有一个例外使得它们的工作方式不同，接下来我们要讨论的就是那个例外。

我们从一个简单的例子开始，如果你想声明一个带有初始值27的int，C++98提供两种语法选择：
```cpp
int x1 = 27;
int x2(27);
```

C++11由于也添加了用于支持`统一初始化（uniform initialization）`的语法：

```cpp
int x3 = { 27 };
int x4{ 27 };
```
总之，这四种不同的语法只会产生一个相同的结果：变量类型为int值为27

但是Item5解释了使用auto说明符代替指定类型说明符的好处，所以我们应该很乐意把上面声明中的int替换为auto，我们会得到这样的代码：
```cpp
auto x1 = 27;
auto x2(27);
auto x3 = { 27 };
auto x4{ 27 };
```
这些声明都能通过编译，但是他们不像替换之前那样有相同的意义。前面两个语句确实声明了一个类型为int值为27的变量，
！！！但是后面两个声明了一个`存储一个元素27`的 `std::initializer_list<int>`类型的变量。
```cpp
auto x1 = 27;                   //类型是int，值是27
auto x2(27);                    //同上
auto x3 = { 27 };               //类型是std::initializer_list<int>，
                                //值是{ 27 }
auto x4{ 27 };                  //同上
```
这就造成了auto类型推导不同于模板类型推导的特殊情况。
！！！当用`auto`声明的变量使用`花括号`进行初始化，auto类型推导推出的类型则为`std::initializer_list`。
如果这样的一个类型不能被成功推导（比如花括号里面包含的是不同类型的变量），编译器会拒绝这样的代码：
```cpp
auto x5 = { 1, 2, 3.0 };        //错误！无法推导std::initializer_list<T>中的T
```
就像注释说的那样，在这种情况下类型推导将会失败，但是对我们来说认识到这里确实发生了两种类型推导是很重要的。
一种是由于auto的使用：x5的类型不得不被推导。
因为x5使用花括号的方式进行初始化，x5必须被推导为`std::initializer_list`。
但是`std::initializer_list`是一个模板。`std::initializer_list<T>`会被某种类型T实例化，所以这意味着`T`也会被推导。
推导落入了这里发生的`第二种类型推导`————`模板类型推导的范围`。
在这个例子中推导之所以失败，是因为在花括号中的值并不是同一种类型。
**对于花括号的处理是auto类型推导和模板类型推导唯一不同的地方。**
当 使用auto声明的变量 使用 花括号的语法 进行初始化的时候，会推导出`std::initializer_list<T>`的实例化，但是对于模板类型推导这样就行不通：
```cpp
auto x = { 11, 23, 9 };         //x的类型是std::initializer_list<int>

template<typename T>            //带有与x的声明等价的
void f(T param);                //形参声明的模板

f({ 11, 23, 9 });               //错误！不能推导出T
```cpp
然而如果在模板中指定T是`std::initializer_list<T>`而留下未知T,模板类型推导就能正常工作：
```cpp
template<typename T>
void f(std::initializer_list<T> initList);

f({ 11, 23, 9 });               //T被推导为int，initList的类型为
                                //std::initializer_list<int>
```
**因此auto类型推导和模板类型推导的真正区别在于，`auto`类型推导假定`花括号`表示`std::initializer_list`而模板类型推导不会这样（确切的说是不知道怎么办）。**
你可能想知道为什么auto类型推导和模板类型推导对于花括号有不同的处理方式。
我也想知道。哎，我至今没找到一个令人信服的解释。
但是规则就是规则，这意味着你必须记住如果你使用auto声明一个变量，并用花括号进行初始化，auto类型推导总会得出`std::initializer_list`的结果。
如果你使用**uniform initialization（花括号的方式进行初始化）**用得很爽你就得记住这个例外以免犯错，在C++11编程中一个典型的错误就是偶然使用了`std::initializer_list<T>`类型的变量，这个陷阱也导致了很多C++程序员抛弃花括号初始化，只有不得不使用的时候再做考虑。（在Item7讨论了必须使用时该怎么做）
对于C++11故事已经说完了。但是对于C++14故事还在继续，`C++14允许auto用于函数返回值`并会被推导（参见Item3），而且C++14的lambda函数也允许在形参声明中使用auto。但是在这些情况下auto实际上使用模板类型推导的那一套规则在工作，而不是auto类型推导，所以说下面这样的代码不会通过编译：
```cpp
auto createInitList()
{
    return { 1, 2, 3 };         //错误！不能推导{ 1, 2, 3 }的类型
}
```
同样在C++14的lambda函数中这样使用auto也不能通过编译：
```cpp
std::vector<int> v;
// …
auto resetV = [&v](const auto& newValue){ v = newValue; };        //C++14
// …
resetV({ 1, 2, 3 });            //错误！不能推导{ 1, 2, 3 }的类型
```
请记住：
- auto类型推导通常和模板类型推导相同，但是auto类型推导假定花括号初始化代表`std::initializer_list`，而模板类型推导不这样做
- 在C++14中auto允许出现在函数返回值或者lambda函数形参中，但是它的工作机制是`模板类型推导`那一套方案，而不是auto类型推导




# 条款3：理解decltype
`decltype`是一个奇怪的东西。
给它一个名字或者表达式decltype就会告诉你这个名字或者表达式的类型。
通常，它会精确的告诉你你想要的结果。但有时候它得出的结果也会让你挠头半天，最后只能求助网上问答或参考资料寻求启示。
我们将从一个简单的情况开始，没有任何令人惊讶的情况。相比模板类型推导和auto类型推导（参见Item1和Item2），decltype只是简单的返回名字或者表达式的类型：
```cpp
const int i = 0;                //decltype(i)是const int

bool f(const Widget& w);        //decltype(w)是const Widget&
                                //decltype(f)是bool(const Widget&)

struct Point {
    int x,y;                    //decltype(Point::x)是int
};                              //decltype(Point::y)是int

Widget w;                       //decltype(w)是Widget

if (f(w))
// …                            //decltype(f(w))是bool

template<typename T>            //std::vector的简化版本
class vector{
public:
    // …
    T& operator[](std::size_t index);
    // …
};

vector<int> v;                  //decltype(v)是vector<int>
…
if (v[0] == 0)
// …                            //decltype(v[0])是int&
```
看见了吧？没有任何奇怪的东西。
**在C++11中，`decltype`最主要的用途就是用于`声明函数模板`，而这个`函数返回类型`依赖于`形参类型`**
。举个例子，假定我们写一个函数，一个形参为容器，一个形参为索引值，这个函数支持使用方括号的方式（也就是使用“[]”）访问容器中指定索引值的数据，然后在返回索引操作的结果前执行认证用户操作。
`函数的返回类型`应该和`索引操作返回的类型`相同。
对一个T类型的容器使用`operator[]`通常会返回一个`T&`对象，比如`std::deque`就是这样。
但是`std::vector`有一个例外，对于`std::vector<bool>`，operator[]不会返回bool&，它会返回一个全新的对象（译注：MSVC的STL实现中返回的是`std::_Vb_reference<std::_Wrap_alloc<std::allocator<unsigned int>>>`对象）。
关于这个问题的详细讨论请参见Item6，这里重要的是我们可以看到对一个容器进行operator[]操作返回的类型取决于容器本身。
使用decltype使得我们很容易去实现它，这是我们写的第一个版本，使用decltype计算返回类型，这个模板需要改良，我们把这个推迟到后面：
```cpp
template<typename Container, typename Index>    //可以工作，
auto authAndAccess(Container& c, Index i)       //但是需要改良
    ->decltype(c[i])
{
    authenticateUser();
    return c[i];
}
```
函数名称前面的auto不会做任何的类型推导工作。
相反的，他只是暗示使用了C++11的`尾置返回类型语法`，即在函数形参列表后面使用一个”->“符号指出函数的返回类型，（！！！）**`尾置返回类型`的好处是我们可以`在函数返回类型中 使用 函数形参 相关的信息`**。
在authAndAccess函数中，我们使用c和i指定返回类型。如果我们按照传统语法把函数返回类型放在函数名称之前，c和i就未被声明所以不能使用。
在这种声明中，authAndAccess函数返回operator[]应用到容器中返回的对象的类型，这也正是我们期望的结果。

C++11允许自动推导单一语句的lambda表达式的返回类型， `C++14`扩展到允许自动推导所有的lambda表达式和函数，甚至它们内含多条语句。对于authAndAccess来说这意味着在C++14标准下我们可以忽略尾置返回类型，只留下一个auto。使用这种声明形式，auto标示这里会发生类型推导。更准确的说，`编译器 将会从 函数实现中 推导出 函数的返回类型`。
```cpp
template<typename Container, typename Index>    //C++14版本，
auto authAndAccess(Container& c, Index i)       //不那么正确
{
    authenticateUser();
    return c[i];                                //从c[i]中推导返回类型
}
```
Item2解释了函数返回类型中使用auto，编译器实际上是使用的`模板类型推导`的那套规则。如果那样的话这里就会有一些问题。
正如我们之前讨论的，`operator[]`对于大多数T类型的容器会返回一个T&，但是Item1解释了**在`模板类型推导`期间，表达式的`引用性（reference-ness）`会被`忽略`**。
基于这样的规则，考虑它会对下面用户的代码有哪些影响：
```cpp
std::deque<int> d;
// …
authAndAccess(d, 5) = 10;               //认证用户，返回d[5]，
                                        //然后把10赋值给它
                                        //无法通过编译器！
```
在这里d[5]本该返回一个`int&`，但是模板类型推导会剥去引用的部分，因此产生了int返回类型。
函数返回的那个int是一个右值，上面的代码尝试把10赋值给右值int，C++11禁止这样做，所以代码无法编译。

要想让authAndAccess像我们期待的那样工作，我们需要使用`decltype类型推导`来推导它的返回值，即指定authAndAccess应该返回一个和c[i]表达式类型一样的类型。
C++期望在某些情况下 当类型被暗示时需 要使用decltype类型推导的规则，C++14通过使用decltype(auto)说明符使得这成为可能。
我们第一次看见`decltype(auto)`可能觉得非常的矛盾（到底是decltype还是auto？），实际上我们可以这样解释它的意义：**`auto`说明符表示这个类型将会被推导，`decltype`说明`decltype的规则`将会被用到这个推导过程中**。
归根到底要看具名`表达式`来推导。
因此我们可以这样写authAndAccess：
```cpp
template<typename Container, typename Index>    //C++14版本，
decltype(auto)                                  //可以工作，
authAndAccess(Container& c, Index i)            //但是还需要
{                                               //改良
    authenticateUser();
    return c[i];
}
```
现在authAndAccess将会真正的返回`c[i]`的类型。
现在事情解决了，一般情况下`c[i]`返回`T&`，authAndAccess也会返回T&，特殊情况下`c[i]`返回一个对象，authAndAccess也会返回一个对象。
`decltype(auto)`的使用不仅仅局限于函数返回类型，当你想对初始化表达式使用decltype推导的规则，你也可以使用：
PS: `decltype(auto)`是特定语法，表示在auto类型推导的基础上，保留`引用`性质，以及`const`和`volatile`
```cpp
Widget w;

const Widget& cw = w;

auto myWidget1 = cw;                    //auto类型推导
                                        //myWidget1 的类型为 Widget，！！！既忽略引用，又忽略const
decltype(auto) myWidget2 = cw;          //decltype类型推导
                                        //myWidget2 的类型是 const Widget&，因为cw的类型为const Widget&类型

// 可以这样理解：
template<typename T>            //概念化的模板用来推导myWidget1的类型
void func_for_myWidget1(T param);

func_for_myWidget1(cw);
```

但是这里有两个问题困惑着你。一个是我之前提到的authAndAccess的改良至今都没有描述。让我们现在加上它。
再看看C++14版本的`authAndAccess`声明：
```cpp
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i);
```
容器通过传引用的方式传递非常量左值引用（lvalue-reference-to-non-const），因为返回一个引用允许用户可以修改容器。
但是这意味着 不能给这个函数传递右值容器，右值不能被绑定到左值引用上（除非这个左值引用是一个const（lvalue-references-to-const），但是这里明显不是）。

公认的向authAndAccess传递一个`右值`是一个edge case（译注：在极限操作情况下会发生的事情，类似于会发生但是概率较小的事情）。
`一个右值容器，是一个临时对象，通常会在authAndAccess调用结束被销毁，这意味着authAndAccess返回的引用将会成为一个悬置的（dangle）引用。`
但是使用向authAndAccess传递一个临时变量也并不是没有意义，有时候 用户可能只是想简单的 获得临时容器中的一个元素的拷贝，比如这样：
```cpp
std::deque<std::string> makeStringDeque();      //工厂函数

//从makeStringDeque中获得第五个元素的拷贝并返回
auto s = authAndAccess(makeStringDeque(), 5);
```
要想支持这样使用authAndAccess我们就得修改一下当前的声明使得它支持左值和右值。
`重载`是一个不错的选择（一个函数重载声明为左值引用，另一个声明为右值引用），但是我们就不得不维护两个重载函数。
另一个方法是使authAndAccess的引用可以绑定左值和右值，Item24解释了那正是`通用引用`能做的，所以我们这里可以使用通用引用进行声明：
```cpp
template<typename Containter, typename Index>   //现在c是通用引用
decltype(auto) authAndAccess(Container&& c, Index i);
```
在这个模板中，我们不知道我们操纵的容器的类型是什么，那意味着我们同样不知道它使用的索引对象（index objects）的类型，`对一个未知类型的对象使用传值通常会造成不必要的拷贝`，对程序的性能有极大的影响，还会造成`对象切片`行为（参见item41 对象切片是指当一个子类对象被赋值给一个父类对象时，只会复制子类对象中属于父类的部分，而子类特有的部分会被“切片”掉，丢失掉子类特有的信息），以及给同事落下笑柄。
但是就`容器索引`来说，我们遵照标准模板库对于索引的处理是有理由的（比如std::string，std::vector和std::deque的operator[]），所以我们坚持传值调用。
然而，我们还需要更新一下模板的实现，让它能听从Item25的告诫应用`std::forward`实现`通用引用`：
```cpp
template<typename Container, typename Index>    //最终的C++14版本
decltype(auto)
authAndAccess(Container&& c, Index i)
{
    authenticateUser();
    return std::forward<Container>(c)[i];   // 如果传入的是右值/临时变量，返回出右值
}
```
这样就能对我们的期望交上一份满意的答卷，但是这要求编译器支持C++14。
如果你没有这样的编译器，你还需要使用C++11版本的模板，它看起来和C++14版本的极为相似，除了你不得不指定函数返回类型之外：
```cpp
template<typename Container, typename Index>    //最终的C++11版本
auto
authAndAccess(Container&& c, Index i)
->decltype(std::forward<Container>(c)[i])
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```
另一个问题是就像我在条款的开始唠叨的那样，decltype通常会产生你期望的结果，但并不总是这样。
在极少数情况下它产生的结果可能让你很惊讶。老实说如果你不是一个大型库的实现者你不太可能会遇到这些异常情况。
为了完全理解decltype的行为，你需要熟悉一些特殊情况。
它们大多数都太过晦涩以至于几乎没有书进行有过权威的讨论，这本书也不例外，但是其中的一个会让我们更加理解decltype的使用。
将`decltype`应用于`变量名`会产生该变量名的声明类型。
虽然变量名都是左值表达式，但这不会影响decltype的行为。（译者注：这里是说对于单纯的变量名，decltype只会返回变量的声明类型）
然而，对于比单纯的变量名更复杂的左值表达式，decltype可以确保报告的类型始终是`左值引用`。
**也就是说，如果一个 不是单纯变量名的 左值表达式的 类型是`T`，那么`decltype`会把这个表达式的类型报告为`T&`**。
这几乎没有什么太大影响，因为大多数左值表达式的类型天生具备一个左值引用修饰符。
例如，返回左值的函数总是返回左值引用。
这个行为暗含的意义值得我们注意，在：
```cpp
int x = 0;
```
中，x是一个变量的名字，所以`decltype(x)`是`int`。
但是如果用一个小括号包覆这个名字，比如这样`(x)` ，就会产生一个比名字更复杂的表达式。
对于名字来说，`x`是一个`左值`（但是不是引用），C++11定义了表达式`(x)`也是一个`左值`。
因此`decltype((x))`是`int&`。用小括号覆盖一个名字可以改变decltype对于名字产生的结果。
在C++11中这稍微有点奇怪，但是由于C++14允许了decltype(auto)的使用，这意味着你在函数返回语句中细微的改变就可以影响类型的推导：
```cpp
decltype(auto) f1()
{
    int x = 0;
    // …
    return x;                            //decltype(x）是int，所以f1返回int
}

decltype(auto) f2()
{
    int x = 0;
    return (x);                          //decltype((x))是int&，所以f2返回int&
}
```
注意不仅f2的返回类型不同于f1，而且它还引用了一个局部变量！这样的代码将会把你送上`未定义行为`的特快列车，一辆你绝对不想上第二次的车。
当使用`decltype(auto)`的时候一定要加倍的小心，在表达式中看起来无足轻重的细节将会影响到decltype(auto)的推导结果。为了确认类型推导是否产出了你想要的结果，请参见Item4描述的那些技术。
同时你也不应该忽略decltype这块大蛋糕。
没错，decltype（单独使用或者与auto一起用）可能会偶尔产生一些令人惊讶的结果，但那毕竟是少数情况。
通常，decltype都会产生你想要的结果，尤其是当你对一个变量使用decltype时，因为在这种情况下，decltype只是做一件本分之事：它产出变量的声明类型。
请记住：
* decltype总是`不加修改的`产生变量或者表达式的类型。
* 对于T类型的`不是单纯的变量名的` `左值表达式`，decltype总是产出T的引用即T&。
* C++14支持decltype(auto)，就像auto一样，推导出类型，但是它使用decltype的规则进行推导。



# 条款4：学会查看类型推导结果
选择使用工具查看类型推导，取决于软件开发过程中你想在哪个阶段显示类型推导信息。
我们探究三种方案：在你`编辑`代码的时候获得类型推导的结果，在`编译期间`获得结果，在`运行时`获得结果。


## 编译器诊断
另一个获得推导结果的方法是使用编译器出错时提供的错误消息。
这些错误消息无形的提到了造成我们编译错误的类型是什么。
举个例子，假如我们想看到之前那段代码中x和y的类型，我们可以首先声明一个类模板但不定义。就像这样：
```cpp
template<typename T>                //只对TD进行声明
class TD;                           //TD == "Type Displayer"
```
如果尝试实例化这个类模板就会引出一个错误消息，因为这里没有用来实例化的类模板定义。
为了查看x和y的类型，只需要使用它们的类型去实例化TD：
```cpp
TD<decltype(x)> xType;              //引出包含x和y
TD<decltype(y)> yType;              //的类型的错误消息
```
我使用`variableNameType`的结构来命名变量，因为这样它们产生的错误消息可以有助于我们查找。
对于上面的代码，我的编译器产生了这样的错误信息，我取一部分贴到下面：
```sh
error: aggregate 'TD<int> xType' has incomplete type and cannot be defined
error: aggregate 'TD<const int *> yType' has incomplete type and cannot be defined
``` 
另一个编译器也产生了一样的错误，只是格式稍微改变了一下：
```sh
error: 'xType' uses undefined class 'TD<int>'
error: 'yType' uses undefined class 'TD<const int *>'
```
除了格式不同外，几乎所有我测试过的编译器都产生了这样有用的错误消息。


## 运行时输出
使用printf的方法使类型信息只有在运行时才会显示出来（尽管我不是非常建议你使用printf），但是它提供了一种格式化输出的方法。
现在唯一的问题是只需对于你关心的变量使用一种优雅的文本表示。
“这有什么难的，“你这样想，”这正是`typeid`和`std::type_info::name`的价值所在”。
为了实现我们想要查看x和y的类型的需求，你可能会这样写：
```cpp
std::cout << typeid(x).name() << '\n';  //显示x和y的类型
std::cout << typeid(y).name() << '\n';
```
这种方法对一个对象如x或y调用typeid产生一个`std::type_info`的对象，然后`std::type_info`里面的成员函数`name()`来产生一个C风格的字符串（即一个`const char*`）表示变量的名字。
调用`std::type_info::name`不保证返回任何有意义的东西，但是库的实现者尝试尽量使它们返回的结果有用。
实现者们对于“有用”有不同的理解。
举个例子，GNU和Clang环境下x的类型会显示为”i“，y会显示为”PKi“，这样的输出你必须要问问编译器实现者们才能知道他们的意义：”i“表示”int“，”`PK`“表示”`pointer to konst const`“（指向常量的指针）。
（这些编译器都提供一个工具c++filt，解释这些“混乱的”类型）
Microsoft的编译器输出得更直白一些：对于x输出”int“ 对于y输出”int const *“

因为对于x和y来说这样的结果是正确的，你可能认为问题已经接近了，别急，考虑一个更复杂的例子：
```cpp
template<typename T>                    //要调用的模板函数
void f(const T& param);

std::vector<Widget> createVec();        //工厂函数

const auto vw = createVec();            //使用工厂函数返回值初始化vw

if (!vw.empty()) {
    f(&vw[0]);                          //调用f，f内的表达式是const Base*类型
    // …
}
```
在这段代码中包含了一个用户定义的类型Widget，一个STL容器std::vector和一个auto变量vw，
这个更现实的情况是你可能在会遇到的并且想获得他们类型推导的结果，比如模板类型形参T，比如函数f形参param。
从这里中我们不难看出typeid的问题所在。
我们在f中添加一些代码来显示类型：
```cpp
template<typename T>
void f(const T& param)  // ！！！对于函数形参而言，不管const在哪里，都是顶层引用，也就是param不可以重新赋值
{
    using std::cout;
    cout << "T =     " << typeid(T).name() << '\n';             //显示T

    cout << "param = " << typeid(param).name() << '\n';         //显示
    // …                                                        //param
}                                                               //的类型
```
GNU和Clang执行这段代码将会输出这样的结果
```cpp
T =     PK6Widget
param = PK6Widget
```
我们早就知道在这些编译器中PK表示“pointer to const”，所以只有数字6对我们来说是神奇的。
其实`数字`是`类名称（Widget）`的`字符串长度`，所以这些编译器告诉我们T和param都是const Widget*。

Microsoft的编译器也同意上述言论：
```cpp
T =     class Widget const *
param = class Widget const *
```
这三个独立的编译器产生了相同的信息并表示信息非常准确，当然看起来不是那么准确。
在模板f中，param的声明类型是const T&。难道你们不觉得T和param类型相同很奇怪吗？比如T是int，param的类型应该是const int&而不是相同类型才对吧。
遗憾的是，事实就是这样，`std::type_info::name`的结果并不总是可信的，就像上面一样，三个编译器对param的报告都是`错误的`。
因为它们本质上可以不正确，因为`std::type_info::name`规范批准 像传值形参一样 来对待这些类型。
正如Item1提到的，如果传递的是一个`引用`，那么引用部分（reference-ness）将被忽略，如果忽略后还具有const或者volatile，那么常量性constness或者易变性volatileness也会被忽略。
那就是为什么`param`的类型`const Widget* const &`会输出为`const Widget *`，首先引用被忽略，然后这个指针自身的常量性constness被忽略，剩下的就是指针指向一个常量对象。
同样遗憾的是，IDE编辑器显示的类型信息也不总是可靠的，或者说不总是有用的。
还是一样的例子，一个IDE编辑器可能会把T的类型显示为（我没有胡编乱造）：
```cpp
const
std::_Simple_types<std::_Wrap_alloc<std::_Vec_base_types<Widget, std::allocator<Widget>>::_Alloc>::value_type>::value_type *
```
同样把param的类型显示为
```cpp
const std::_Simple_types<...>::value_type *const &
```
这个比起T来说要简单一些，但是如果你不知道“...”表示编译器忽略T的部分类型那么可能你还是会产生困惑。如果你运气好点你的IDE可能表现得比这个要好一些。

比起运气如果你更倾向于依赖库，那么你乐意被告知std::type_info::name和IDE不怎么好，`Boost TypeIndex`库（通常写作Boost.TypeIndex）是更好的选择。
这个库不是标准C++的一部分，也不是IDE或者TD这样的模板。
Boost库（可在boost.com获得）是跨平台，开源，有良好的开源协议的库，这意味着使用Boost和STL一样具有高度可移植性。
这里是如何使用`Boost.TypeIndex`得到f的类型的代码
```cpp
#include <boost/type_index.hpp>

template<typename T>
void f(const T& param)
{
    using std::cout;
    using boost::typeindex::type_id_with_cvr;

    //显示T
    cout << "T =     "
         << type_id_with_cvr<T>().pretty_name()
         << '\n';
    
    //显示param类型
    cout << "param = "
         << type_id_with_cvr<decltype(param)>().pretty_name()
         << '\n';
}
```
`boost::typeindex::type_id_with_cvr`获取一个类型实参（我们想获得相应信息的那个类型），它不消除实参的const，volatile和引用修饰符（因此模板名中有“with_cvr”）。
结果是一个`boost::typeindex::type_index`对象，它的`pretty_name`成员函数输出一个`std::string`，包含我们能看懂的类型表示。
基于这个f的实现版本，再次考虑那个使用`typeid`时获取param类型信息出错的调用：
```cpp
std::vetor<Widget> createVec();         //工厂函数
const auto vw = createVec();            //使用工厂函数返回值初始化vw
if (!vw.empty()) {
    f(&vw[0]);                          //调用f
    // …
}
```
在GNU和Clang的编译器环境下，使用Boost.TypeIndex版本的f最后会产生下面的（准确的）输出：
```cpp
T =     Widget const *
param = Widget const * const&
```
在Microsoft的编译器环境下，结果也是极其相似：
```cpp
T =     class Widget const *
param = class Widget const * const &
```
这样近乎一致的结果是很不错的，但是请记住IDE，编译器错误诊断或者像Boost.TypeIndex这样的库只是用来帮助你理解编译器推导的类型是什么。它们是有用的，但是作为本章结束语我想说它们根本不能替代你对Item1-3提到的类型推导的理解。

请记住：
* 类型推断可以从IDE看出，从编译器报错看出，从Boost TypeIndex库的使用看出
* 这些工具可能既不准确也无帮助，所以理解C++类型推导规则才是最重要的

# 第2章 `auto`
从概念上来说，auto要多简单有多简单，但是它比看起来要微妙一些。
使用它可以存储类型，当然，它也会犯一些错误，而且比之手动声明一些复杂类型也会存在一些性能问题。
此外，从程序员的角度来说，如果按照符合规定的流程走，那auto类型推导的一些结果是错误的。
当这些情况发生时，对我们来说引导auto产生正确的结果是很重要的，因为严格按照说明书上面的类型写声明虽然可行但是最好避免。
本章简单的覆盖了auto的里里外外。



# 条款5：优先考虑auto而非显式类型声明
哈，开心一下：
```cpp
int x;
```
等等，该死！我忘记了初始化x，所以x的值是不确定的。
它可能会被初始化为0，这得取决于工作环境。哎。

别介意，让我们转换一个话题， 对一个`局部变量`使用`解引用迭代器`的方式`初始化`：
```cpp
template<typename It>           //对从b到e的所有元素使用
void dwim(It b, It e)           //dwim（“do what I mean”）算法
{
    while (b != e) {
        typename std::iterator_traits<It>::value_type currValue = *b;   // 这样写太复杂了
        // …
    }
}
```

嘿！`typename std::iterator_traits<It>::value_type`是想表达迭代器指向的元素的值的类型吗？
    我无论如何都说不出它是多么有趣这样的话，该死！等等，我早就说过了吗？

好吧，声明一个局部变量，类型是一个`闭包`，闭包的类型只有编译器知道，因此我们写不出来，该死!
该死该死该死，C++编程不应该是这样不愉快的体验。
别担心，它只在过去是这样，到了C++11所有的这些问题都消失了，这都多亏了auto。
`auto变量`从`初始化表达式`中推导出类型，所以我们必须初始化。这意味着当你在现代化C++的高速公路上飞奔的同时你不得不对只声明不初始化变量的老旧方法说拜拜：
```cpp
int x1;                         //潜在的未初始化的变量
    
auto x2;                        //错误！必须要初始化

auto x3 = 0;                    //没问题，x已经定义了
```
而且即使 使用 解引用迭代器 初始化局部变量 也不会对你的高速驾驶有任何影响
```cpp
template<typename It>           //如之前一样
void dwim(It b,It e)
{
    while (b != e) {
        auto currValue = *b;
        // …
    }
}
```
因为使用Item2所述的auto类型推导技术，它甚至能表示一些`只有编译器才知道`的类型：
使用auto带来便利：
```cpp
auto derefUPLess = 
    [](const std::unique_ptr<Widget> &p1,       //用于std::unique_ptr
       const std::unique_ptr<Widget> &p2)       //指向的Widget类型的
    { return *p1 < *p2; };                      //比较函数
```
很酷对吧，如果使用C++14，将会变得更酷，因为lambda表达式中的`形参也可以使用auto`(也就是说，具体类型尚未确定)：

```cpp
auto derefLess =                                //C++14版本
    [](const auto& p1,                          //被任何像指针一样的东西
       const auto& p2)                          //指向的值的比较函数
    { return *p1 < *p2; };
```

尽管这很酷，但是你可能会想我们完全不需要使用auto声明局部变量来保存一个闭包，因为我们可以使用`std::function`对象。
没错，我们的确可以那么做，但是事情可能不是完全如你想的那样。当然现在你可能会问，std::function对象到底是什么。让我来给你解释一下。
`std::function`是一个C++11标准模板库中的一个模板，它`泛化了``函数指针`的概念。
与函数指针只能指向函数不同，std::function可以指向任何`可调用对象`，也就是那些像函数一样能进行调用的东西。
当你`声明函数指针时` 你 必须 `指定函数类型（即函数签名）`，同样当你创建`std::function`对象时你也需要提供函数签名，由于它是一个模板 所以你需要在它的模板参数里面提供。
举个例子，假设你想声明一个std::function对象func使它指向一个可调用对象，比如一个具有这样函数签名的函数，
```cpp
bool(const std::unique_ptr<Widget> &,           //C++11
     const std::unique_ptr<Widget> &)           //std::unique_ptr<Widget>
                                                //比较函数的签名
```
你就得这么写：
```cpp
std::function<bool(const std::unique_ptr<Widget> &,
                   const std::unique_ptr<Widget> &)> func;
```
因为lambda表达式能产生一个可调用对象，所以我们现在可以把闭包存放到std::function对象中。这意味着我们可以不使用auto写出C++11版的derefUPLess：
```cpp
std::function<bool(const std::unique_ptr<Widget> &,
                   const std::unique_ptr<Widget> &)>
derefUPLess = [](const std::unique_ptr<Widget> &p1,
                 const std::unique_ptr<Widget> &p2)
                { return *p1 < *p2; };
```
语法冗长不说，还需要重复写很多形参类型，使用std::function还不如使用auto。
用`auto声明的变量` 保存一个和闭包一样类型的（新）闭包，因此使用了与闭包`相同大小存储空间`。
实例化std::function 并声明一个对象 这个对象将会有`固定的大小`。
这个大小可能不足以存储一个闭包，这个时候`std::function`的构造函数将会在`堆`上面分配内存来存储，这就造成了：
！！！使用`std::function`比`auto声明变量`会`消耗更多的内存`。因为`auto`实际得出的是一个匿名类型，是不可名状的。
并且通过具体实现 我们得知 通过`std::function`调用一个闭包几乎无疑比auto声明的对象调用要慢。
换句话说，std::function方法比auto方法要更耗空间且更慢，还可能有`out-of-memory`异常。
并且正如上面的例子，比起写std::function实例化的类型来，使用auto要方便得多。
在这场存储闭包的比赛中，auto无疑取得了胜利(也可以使用`std::bind`来生成一个闭包，但在Item34我会尽我最大努力说服你使用`lambda表达式`代替`std::bind`)

使用auto除了可以避免未初始化的无效变量，省略冗长的声明类型，直接保存闭包外，它还有一个好处是可以避免一个问题，我称之为与`类型快捷方式（type shortcuts）`有关的问题。你将看到这样的代码————甚至你会这么写：
```cpp
std::vector<int> v;
// …
unsigned sz = v.size();
```
`v.size()`的标准返回类型是`std::vector<int>::size_type`，但是只有少数开发者意识到这点。`std::vector<int>::size_type`实际上被指定为无符号整型，所以很多人都认为用unsigned就足够了，写下了上述的代码。
这会造成一些有趣的结果。举个例子，在Windows 32-bit上`std::vector<int>::size_type`和`unsigned`是一样的大小，但是在Windows 64-bit上`std::vector<int>::size_type`是64位，unsigned是32位。
这意味着这段代码在Windows 32-bit上正常工作，但是当把应用程序移植到Windows 64-bit上时就可能会出现一些问题。谁愿意花时间处理这些细枝末节的问题呢？
！！！所以使用auto可以确保你不需要浪费时间(使得代码平台无关了)：
```cpp
auto sz = v.size();                      //sz的类型是std::vector<int>::size_type
```
你还是不相信使用auto是多么明智的选择？考虑下面的代码：
```cpp
std::unordered_map<std::string, int> m;
// …

for(const std::pair<std::string, int>& p : m)
{
    // …                                   //用p做一些事
}
```
看起来好像很合情合理的表达，但是这里有一个问题，你看到了吗？
要想看到错误你就得知道`std::unordered_map`的key是`const`的，所以hash table（std::unordered_map本质上的东西）中的std::pair的类型不是`std::pair<std::string, int>，而是std::pair<const std::string, int>`。
但那不是在循环中的变量p声明的类型。
编译器会努力的找到一种方法把`std::pair<const std::string, int>`（即hash table中的东西）转换为`std::pair<std::string, int>`（p的声明类型）。
它会成功的，因为它会通过`拷贝`m中的对象创建一个`临时对象`，这个临时对象的类型是p想绑定到的对象的类型，即m中元素的类型，然后把p的引用绑定到这个临时对象上。
在每个循环迭代结束时，临时对象将会销毁，如果你写了这样的一个循环，你可能会对它的一些行为感到非常惊讶，因为你确信你只是让成为p指向m中各个元素的引用而已。
使用auto可以避免这些很难被意识到的类型不匹配的错误：
```cpp
for(const auto& p : m)
{
    …                                   //如之前一样
}
```
这样无疑更具效率，且更容易书写。
而且，这个代码有一个非常吸引人的特性，如果你获取p的地址，你确实会得到一个`指向m中元素的指针`。
在没有auto的版本中p会指向一个临时变量，这个临时变量在每次迭代完成时会被销毁。
前面这两个例子————应当写`std::vector<int>::size_type`时写了`unsigned`，应当写`std::pair<const std::string, int>`时写了`std::pair<std::string, int>`————说明了 `显式的指定类型`可能会导致`你不想看到的类型转换`。
如果你使用auto声明目标变量你就不必担心这个问题。
基于这些原因 我建议你 优先考虑auto而非显式类型声明。

然而auto也不是完美的。每个auto变量都从`初始化表达式`中推导类型，有一些表达式的类型和我们期望的大相径庭。
关于在哪些情况下会发生这些问题，以及你可以怎么解决这些问题我们在Item2和6讨论，所以这里我不再赘述。我想把注意力放到你可能关心的另一点：`使用auto代替传统类型声明对源码可读性的影响`。
首先，深呼吸，放松，auto是可选项，不是命令，在某些情况下如果你的专业判断告诉你使用显式类型声明比auto要更清晰更易维护，那你就不必再坚持使用auto。
但是要牢记，C++没有在其他众所周知的语言所拥有的类型推导（type inference）上开辟新土地。其他静态类型的过程式语言（如C#、D、Sacla、Visual Basic）或多或少都有等价的特性，更不必提那些静态类型的函数式语言了（如ML、Haskell、OCaml、F#等）。
在某种程度上，这是因为动态类型语言，如Perl、Python、Ruby等的成功；在这些语言中，几乎没有显式的类型声明。软件开发社区对于类型推导有丰富的经验，他们展示了在维护大型工业强度的代码上使用这种技术没有任何争议。
一些开发者也担心使用auto就不能瞥一眼源代码便知道对象的类型，然而，IDE扛起了部分担子（也考虑到了Item4中提到的IDE类型显示问题），在很多情况下，少量显示一个对象的类型对于知道对象的确切类型是有帮助的，这通常已经足够了。举个例子，要想知道一个对象是容器还是计数器还是智能指针，不需要知道它的确切类型。一个`适当的变量名称`就能告诉我们大量的抽象类型信息。

事实是显式指定类型通常只会引入一些微妙的错误，无论是在正确性还是效率方面。而且，如果初始化表达式的类型改变，则auto推导出的类型也会改变，这意味着`使用auto可以帮助我们完成一些重构工作`。
    举个例子，如果一个函数返回类型被声明为int，但是后来你认为将它声明为long会更好，调用它作为初始化表达式的变量会自动改变类型，但是如果你不使用auto你就不得不在源代码中挨个找到调用地点然后修改它们。

请记住：
* auto变量必须`初始化`，通常它可以避免一些移植性和效率性的问题，也使得重构更方便，还能让你少打几个字。
* 正如Item2和6讨论的，auto类型的变量可能会踩到一些陷阱。






# 条款6：auto推导若非己愿，使用显式类型初始化惯用法
在Item5中解释了比起显式指定类型使用auto声明变量有若干技术优势，但是有时当你想向左转auto却向右转。
举个例子，假如我有一个函数，参数为`Widget`，返回一个`std::vector<bool>`，这里的bool表示Widget是否提供一个独有的特性。
```cpp
std::vector<bool> features(const Widget& w);
```
更进一步假设第5个bit表示Widget是否具有高优先级，我们可以写这样的代码：
```cpp
Widget w;
// …
bool highPriority = features(w)[5];     //w高优先级吗？
// …
processWidget(w, highPriority);         //根据它的优先级处理w
```
这个代码没有任何问题。
它会正常工作，但是如果我们使用auto代替highPriority的显式指定类型做一些看起来很无害的改变：
```cpp
auto highPriority = features(w)[5];     //w高优先级吗？
```
情况变了。所有代码仍然可编译，但是行为不再可预测：
```cpp
processWidget(w,highPriority);          //未定义行为！
```
就像注释说的，这个processWidget是一个未定义行为。
为什么呢？答案有可能让你很惊讶，使用auto后`highPriority`不再是`bool`类型。
虽然从概念上来说`std::vector<bool>`意味着存放bool，但是 ！！！`std::vector<bool>`的`operator[]`不会返回容器中元素的引用（！！！这就是`std::vector::operator[]`可返回除了`bool`以外的任何类型），取而代之它返回一个`std::vector<bool>::reference`的对象（一个嵌套于`std::vector<bool>`中的类）。
`std::vector<bool>::reference`之所以存在是因为`std::vector<bool>`规定了使用一个打包形式（packed form）表示它的bool，每个bool占一个bit。
那给`std::vector的operator[]`带来了问题，因为`std::vector<T>`的operator[]应当返回一个T&，但是！！！`C++禁止对bits的引用`。
无法返回一个`bool&`，`std::vector<bool>`的`operator[]`返回一个行为类似于`bool&`的对象。
要想成功扮演这个角色，`bool&`适用的上下文`std::vector<bool>::reference`也必须一样能适用。
在`std::vector<bool>::reference`的特性中，使这个原则可行的特性是一个可以向bool的隐式转化。
（不是`bool&`，是**`bool`**。要想完整的解释`std::vector<bool>::reference`能模拟`bool&`的行为所使用的一堆技术可能扯得太远了，所以这里简单地说隐式类型转换只是这个大型马赛克的一小块）

有了这些信息，我们再来看看原始代码的一部分：
```cpp
bool highPriority = features(w)[5];     //显式的声明highPriority的类型
```
这里，features返回一个`std::vector<bool>`对象后再调用`operator[]`，`operator[]`将会返回一个`std::vector<bool>::reference`对象，然后再通过`！！！隐式转换`赋值给bool变量highPriority。highPriority因此表示的是features返回的std::vector<bool>中的第五个bit，这也正如我们所期待的那样。
然后再对照一下当使用auto时发生了什么：
```cpp
auto highPriority = features(w)[5];     //推导highPriority的类型
```
同样的，features返回一个`std::vector<bool>`对象，再调用`operator[]`，`operator[]`将会返回一个`std::vector<bool>::reference`对象，但是现在这里有一点变化了，auto推导highPriority的类型为`std::vector<bool>::reference`，但是highPriority对象没有第五bit的值。
这个值取决于`std::vector<bool>::reference`的具体实现。
其中的一种实现是这样的（`std::vector<bool>::reference`）对象包含一个指向`机器字（word）`的指针，然后加上方括号中的偏移实现被引用bit这样的行为。
然后再来考虑highPriority初始化表达的意思，注意这里假设`std::vector<bool>::reference`就是刚提到的实现方式。

调用features将返回一个`std::vector<bool>`临时对象，这个对象没有名字，为了方便我们的讨论，我这里叫他temp。
`operator[]`在`temp`上调用，它返回的`std::vector<bool>::reference`包含一个指向存着这些bits的一个数据结构中的一个word的`指针`（temp管理这些bits），还有相应于第5个bit的偏移。
highPriority是这个`std::vector<bool>::reference`的(浅)拷贝，所以highPriority也包含一个指针，指向temp中的这个word，加上相应于第5个bit的偏移。
在这个语句结束的时候temp将会被销毁，因为它是一个临时变量。
因此highPriority包含一个`悬置的（dangling）指针`，如果用于processWidget调用中将会造成未定义行为：
```cpp
processWidget(w, highPriority);         //未定义行为！
                                        //highPriority包含一个悬置指针！
```
`std::vector<bool>::reference`是一个`代理类（proxy class）`的例子：
    ！！！所谓`代理类`就是`以 模仿 和 增强 一些类型的行为 为目的`而存在的类。很多情况下都会使用代理类，`std::vector<bool>::reference`展示了对`std::vector<bool>`使用`operator[]`来实现引用bit这样的行为。
    另外，C++标准模板库中的`智能指针`（见第4章）也是用`代理类`实现了对`原始指针`的资源管理行为。
    代理类的功能已被大家广泛接受。事实上，“Proxy”设计模式是软件设计这座万神庙中一直都存在的高级会员。
    一些代理类被设计于用以`对客户可见`。
    比如std::shared_ptr和std::unique_ptr。
    其他的代理类则`或多或少不可见`，比如`std::vector<bool>::reference`就是不可见代理类的一个例子，还有它在`std::bitset`的胞弟`std::bitset::reference`。

在后者的阵营（注：指不可见代理类）里一些C++库也是用了`表达式模板（expression templates）`的黑科技。
这些库通常被用于提高数值运算的效率。
给出一个矩阵类Matrix和矩阵对象m1，m2，m3，m4，举个例子，这个表达式
```cpp
Matrix sum = m1 + m2 + m3 + m4;
```
可以使计算更加高效，只需要使让`operator+`返回一个代理类代理结果而不是返回结果本身。
也就是说，对两个`Matrix`对象使用`operator+`将会返回如`Sum<Matrix, Matrix>`这样的代理类作为结果而不是直接返回一个Matrix对象。在`std::vector<bool>::reference`和`bool`中存在一个隐式转换，同样对于Matrix来说也可以存在一个隐式转换允许Matrix的代理类转换为Matrix，这让表达式等号“=”右边能产生代理对象来初始化sum。
（这个对象应当编码整个初始化表达式，即类似于`Sum< Sum< Sum<Matrix, Matrix>, Matrix>, Matrix>`的东西。客户应该避免看到这个实际的类型。）
**作为一个通则，不可见的代理类通常不适用于`auto`**。
这样类型的对象的生命期通常不会设计为能活过一条语句，所以创建那样的对象你基本上就走向了违反程序库设计基本假设的道路。·std::vector<bool>::reference·就是这种情况，我们看到违反这个基本假设将导致未定义行为。
因此你想避开这种形式的代码：
```cpp
auto someVar = expression of "invisible" proxy class type;
```
但是你怎么能意识到你正在使用代理类？
应用他们的软件不可能宣告它们的存在。`它们被设计为不可见`，至少概念上说是这样！每当你发现它们，你真的应该舍弃Item5演示的auto所具有的诸多好处吗？
让我们首先回到如何找到它们的问题上。
虽然“不可见”代理类都在程序员日常使用的雷达下方飞行，但是很多库都证明它们可以上方飞行。
当你越熟悉你使用的库的基本设计理念，你的思维就会越活跃，不至于思维僵化认为代理类只能在这些库中使用。

当缺少文档的时候，可以去看看`头文件`。
很少会出现源代码全都用代理对象，它们通常用于一些函数的返回类型，所以通常能从 `函数签名`中看出它们的存在。
这里有一份`std::vector<bool>::operator[]`的说明书：
```cpp
namespace std {                                  //来自于C++标准库
    template<class Allocator>
    class vector<bool, Allocator> {
    public:
        // …
        class reference { … };

        reference operator[](size_type n);
        // …
    };
}
```
假设你知道对`std::vector<T>`使用`operator[]`通常会返回一个`T&`（而vector本身是·vector<T, Allocator>`类型），在这里`operator[]`不寻常的返回类型提示你它使用了代理类。
多关注你使用的接口可以暴露代理类的存在。
实际上，很多开发者都是在跟踪一些令人困惑的复杂问题或在单元测试出错进行调试时才看到代理类的使用。
不管你怎么发现它们的，一旦看到auto推导了代理类的类型而不是被代理的类型，解决方案并不需要抛弃auto。
auto本身没什么问题，问题是auto不会推导出你想要的类型。
！！！解决方案是 `强制 使用 一个不同的 类型推导形式`，这种方法我通常称之为`显式类型 初始器 惯用法（the explicitly typed initialized idiom)`。
显式类型初始器惯用法使用auto声明一个变量，然后对表达式强制类型转换（cast）得出你期望的推导结果。
举个例子，我们该怎么将这个惯用法施加到highPriority上？
```cpp
auto highPriority = static_cast<bool>(features(w)[5]);
```
这里，`features(w)[5]`还是返回一个`std::vector<bool>::reference`对象，就像之前那样，但是这个转型使得表达式类型为bool，然后auto才被用于推导highPriority。
在运行时，对std::vector<bool>::operator[]返回的std::vector<bool>::reference执行它支持的向bool的转型，在这个过程中指向std::vector<bool>的指针已经被解引用。这就避开了我们之前的未定义行为。然后5将被用于指向bit的指针，bool值被用于初始化highPriority。
对于Matrix来说，显式类型初始器惯用法是这样的：
```cpp
auto sum = static_cast<Matrix>(m1 + m2 + m3 + m4);
```
应用这个惯用法不限制初始化表达式产生一个代理类。
它也可以用于强调你声明了一个变量类型，它的类型不同于初始化表达式的类型。
举个例子，假设你有这样一个表达式计算公差值：
```cpp
double calcEpsilon();                           //返回公差值
```
calcEpsilon清楚的表明它返回一个double，但是假设你知道对于这个程序来说使用float的精度已经足够了，而且你很关心double和float的大小。你可以声明一个float变量储存calEpsilon的计算结果。
```cpp
float ep = calcEpsilon();                       //double到float隐式转换
```
但是这几乎没有表明“我确实要减少函数返回值的精度”。使用显式类型初始器惯用法我们可以这样：
```cpp
auto ep = static_cast<float>(calcEpsilon());
```
出于同样的原因，如果你故意想用整数类型存储一个表达式返回的浮点数类型的结果，你也可以使用这个方法。
假如你需要计算一个`随机访问迭代器`（比如`std::vector，std::deque或者std::array`）中某元素的下标，你被提供一个0.0到1.0的double值表明这个元素离容器的头部有多远（0.5意味着位于容器中间）。
进一步假设你很自信结果下标是int。
如果容器是c，d是double类型变量，你可以用这样的方法计算容器下标：
```cpp
int index = d * c.size();   // double * int
// 发生隐式类型转换
```
但是这种写法并没有明确表明你想将右侧的double类型转换成int类型，显式类型初始器可以帮助你正确表意：
```cpp
auto index = static_cast<int>(d * c.size());
```
请记住：
* 不可见的代理类可能会使auto从表达式中推导出“错误的”类型
* 显式类型初始器惯用法强制auto推导出你想要的结果



# 第3章 移步现代C++
说起知名的特性，C++11/14有一大堆可以吹的东西，`auto，智能指针（smart pointer），移动语义（move semantics），lambda，并发（concurrency）`————每个都是如此的重要，这章将覆盖这些内容。
掌握这些特性是必要的，要想成为高效率的现代C++程序员需要小步迈进。
在从C++98小步迈进到现代C++过程中遇到的每个问题，本章都会一一回答。
- 你什么时候应该用`{}`而不是`()`创建对象？
- 为什么别名（`alias`）声明比`typedef`好？
- `constexpr`和`const`有什么不同？
-常量（`const`）`成员函数`和`线程安全`有什么关系？
这个列表越列越多，这章将会逐个回答这些问题。

# 条款7：区别使用()和{}创建对象
取决于你看问题的角度，`C++11 对象 初始化的语法`可能会让你觉得丰富的让人难以选择，亦或是乱的一塌糊涂。
一般来说，初始化值要用圆括号()或者花括号{}括起来，或者放到等号"="的右边：
```cpp
int x(0);               //使用圆括号初始化

int y = 0;              //使用"="初始化

int z{ 0 };             //使用花括号初始化
```
在很多情况下，你可以使用"="和花括号的组合：
```cpp
int z = { 0 };          //使用"="和花括号
```
在这个条款的剩下部分，我通常会忽略`"="和花括号组合 初始化 的语法，因为C++通常把它视作和只有花括号一样`。

“乱的一塌糊涂” 是指 在初始化中使用"="可能会误导C++新手，使他们以为这里发生了赋值运算，然而实际并没有。
对于像int这样的内置类型，研究两者区别就像在做学术，但是对于用户定义的类型而言，区别赋值运算符和初始化就非常重要了，因为它们涉及不同的函数调用：
```cpp
Widget w1;              //调用默认构造函数

Widget w2 = w1;         //不是赋值运算，调用拷贝构造函数！！！看起来是赋值，其实不是

w1 = w2;                //是赋值运算，调用拷贝赋值运算符（copy operator=）
```
甚至对于一些初始化语法，在一些情况下C++98没有办法表达预期的初始化行为。
举个例子，要想 直接创建 并 初始化一个存放一些特殊值的STL容器是不可能的（比如1,3,5）。

C++11使用`统一初始化（uniform initialization）`来整合这些混乱且不适于所有情景的初始化语法，所谓 统一初始化 是指 在任何涉及初始化的地方都使用单一的初始化语法。
它基于花括号，出于这个原因我更喜欢称之为括号初始化。（译注：注意，这里的括号初始化指的是花括号初始化，在没有歧义的情况下下文的括号初始化指的都是用花括号进行初始化；当与圆括号初始化同时存在并可能产生歧义时我会直接指出。）
`统一初始化`是一个`概念上的`东西，而`花括号初始化`是一个`具体语法结构`。
括号初始化让你可以表达以前表达不出的东西。使用花括号，创建并指定一个容器的初始元素变得很容易：
```cpp
std::vector<int> v{ 1, 3, 5 };  //v初始内容为1,3,5
```
括号初始化也能被用于为 `类成员变量`的`非静态数据成员`指定`默认初始值`。
C++11允许"="初始化不加花括号也拥有这种能力：
```cpp
class Widget {
    // …

private:
    int x{ 0 };                 //没问题，x初始值为0
    int y = 0;                  //也可以
    int z(0);                   //错误！
}
```
另一方面，`不可拷贝的`对象（例如`std::atomic`————见Item40）可以使用`花括号初始化`或者`圆括号初始化`，但是不能使用"="初始化：
```cpp
std::atomic<int> ai1{ 0 };      //没问题
std::atomic<int> ai2(0);        //没问题
std::atomic<int> ai3 = 0;       //错误！
```
因此我们很容易理解为什么括号初始化又叫统一初始化，在C++中这三种方式都被看做是初始化表达式，但是`只有花括号任何地方都能被使用`！！！。

`花括号表达式`还有一个少见的特性，即它`不允许内置类型间隐式的变窄转换`（narrowing conversion），也就是比较严格。
如果一个使用了括号初始化的表达式的值，不能保证由被初始化的对象的类型来表示，代码就不会通过编译：
```cpp
double x, y, z;

int sum1{ x + y + z };          //错误！double的和可能不能表示为int
```
使用圆括号和"="的初始化不检查是否转换为变窄转换，因为由于历史遗留问题它们必须要兼容老旧代码：
```cpp
int sum2(x + y + z);             //可以（表达式的值被截为int）

int sum3 = x + y + z;           //同上
```
另一个值得注意的特性是 `括号表达式`对于C++最令人头疼的`解析问题`有天生的免疫性。
（译注：所谓最令人头疼的解析即`most vexing parse`，更多信息请参见https://en.wikipedia.org/wiki/Most_vexing_parse。）
C++规定 `任何可以被解析为一个声明的东西 必须被解析为声明`。
这个规则的副作用是让很多程序员备受折磨：他们可能想创建一个使用默认构造函数构造的对象，却不小心变成了函数声明。问题的根源是如果你调用带参构造函数，你可以这样做：
```cpp
Widget w1(10);                  //使用实参10调用Widget的一个构造函数
```
！！！但是如果你尝试使用相似的语法调用`Widget无参构造函数`，它就会变成函数声明，而不是一个对象：
```cpp
Widget w2();                    //最令人头疼的解析！声明一个函数w2，返回Widget
```
由于函数声明中形参列表不能带花括号，所以使用花括号初始化表明你想调用默认构造函数构造对象就没有问题：
```cpp
Widget w3{};                    //调用没有参数的构造函数构造对象
```
关于括号初始化还有很多要说的。
它的语法能用于各种不同的上下文，它防止了隐式的变窄转换，而且对于C++最令人头疼的解析也天生免疫。
既然好到这个程度那为什么这个条款不叫“优先考虑括号初始化语法”呢？

括号初始化的`缺点`是有时它有一些令人惊讶的行为。
这些行为使得`花括号初始化`、`std::initializer_list`和`构造函数`参与`重载决议`时本来就不清不楚的暧昧关系进一步混乱。
把它们放到一起会让看起来应该左转的代码右转。
举个例子，Item2解释了当`auto`声明的变量使用花括号初始化，变量类型会被推导为`std::initializer_list`，但是使用相同内容的其他初始化方式会产生更符合直觉的结果。
所以，你越喜欢用auto，你就越不能用括号初始化。
在构造函数调用中，只要不包含`std::initializer_list`形参，那么花括号初始化和圆括号初始化都会产生一样的结果：
```cpp
class Widget { 
public:  
    Widget(int i, bool b);      //构造函数未声明std::initializer_list这个形参 
    Widget(int i, double d);
    // …
};

Widget w1(10, true);            //调用第一个构造函数
Widget w2{10, true};            //也调用第一个构造函数
Widget w3(10, 5.0);             //调用第二个构造函数
Widget w4{10, 5.0};             //也调用第二个构造函数
```
然而，如果有一个或者多个构造函数的声明包含一个`std::initializer_list`形参，那么使用`括号初始化`语法的调用更倾向于选择带`std::initializer_list`的那个构造函数。
如果编译器遇到一个括号初始化并且有一个带std::initializer_list的构造函数，那么它一定会选择该构造函数。
如果上面的Widget类有一个`std::initializer_list<long double>`作为参数的构造函数，就像这样：
```cpp
class Widget { 
public:  
    Widget(int i, bool b);      //同上
    Widget(int i, double d);    //同上
    Widget(std::initializer_list<long double> il);      //新添加的
    // …
}; 
```
w2和w4将会使用新添加的构造函数，即使另一个非std::initializer_list构造函数和实参更匹配：
```cpp
Widget w1(10, true);    //使用圆括号初始化，同之前一样
                        //调用第一个构造函数

Widget w2{10, true};    //使用花括号初始化，但是现在
                        //调用带std::initializer_list的构造函数
                        //(10 和 true 转化为long double)

Widget w3(10, 5.0);     //使用圆括号初始化，同之前一样
                        //调用第二个构造函数 

Widget w4{10, 5.0};     //使用花括号初始化，但是现在调用带std::initializer_list的构造函数 (10 和 5.0 转化为long double，std::initializer_list<long double> il会包含这两个元素)
```
也就是使用圆括号初始化，调用的构造函数更为确定
甚至普通构造函数和移动构造函数都会被带`std::initializer_list`的构造函数劫持：
```cpp
class Widget { 
public:  
    Widget(int i, bool b);                              //同之前一样
    Widget(int i, double d);                            //同之前一样
    Widget(std::initializer_list<long double> il);      //同之前一样
    operator float() const;                             // 这里特意提供了转为float类型的接口，用于说明问题
    // …
};

Widget w5(w4);                  //使用圆括号，调用拷贝构造函数

Widget w6{w4};                  //？？？使用花括号，调用std::initializer_list构造函数（w4先转换为float，float转换为long double）

Widget w7(std::move(w4));       //使用圆括号，调用移动构造函数

Widget w8{std::move(w4)};       //使用花括号，调用std::initializer_list构造
                                //函数（与w6相同原因）
```
编译器一遇到括号初始化就选择带`std::initializer_list`的构造函数的决心是如此强烈，以至于就算带`std::initializer_list`的构造函数不能被调用，它也会硬选。
```cpp
class Widget { 
public: 
    Widget(int i, bool b);                      //同之前一样
    Widget(int i, double d);                    //同之前一样
    Widget(std::initializer_list<bool> il);     //现在元素类型为bool
    // …                                           //没有隐式转换函数
};

Widget w{10, 5.0};              //错误！要求变窄转换
```
这里，编译器会直接忽略前面两个构造函数（其中第二个构造函数是所有实参类型的最佳匹配），然后尝试调用`std::initializer_list<bool>`构造函数。
调用这个函数将会把`int(10)`和`double(5.0)`转换为`bool`，由于会产生变窄转换（bool不能准确表示其中任何一个值），括号初始化拒绝变窄转换，所以这个调用无效，代码`无法通过编译`。
！！！只有当`没办法`把括号初始化中实参的类型转化为`std::initializer_list`时，编译器才会回到`正常的函数决议流程`中。
比如我们在构造函数中用`std::initializer_list<std::string>`代替`std::initializer_list<bool>`，这时非`std::initializer_list`构造函数将再次成为函数决议的候选者，因为没有办法把int和bool转换为`std::string`:
```cpp
class Widget { 
public:  
    Widget(int i, bool b);                              //同之前一样
    Widget(int i, double d);                            //同之前一样
    //现在std::initializer_list元素类型为std::string
    Widget(std::initializer_list<std::string> il);
    …                                                   //没有隐式转换函数
};

Widget w1(10, true);     // 使用圆括号初始化，调用第一个构造函数
Widget w2{10, true};     // 使用花括号初始化，现在调用第一个构造函数
Widget w3(10, 5.0);      // 使用圆括号初始化，调用第二个构造函数
Widget w4{10, 5.0};      // 使用花括号初始化，现在调用第二个构造函数
```
代码的行为和我们刚刚的论述如出一辙。
这里还有一个有趣的边缘情况。
假如你使用的花括号初始化是`空集`，并且你欲构建的对象有`默认构造函数`，也有`std::initializer_list`构造函数。
你的空的花括号意味着什么？
如果它们意味着没有实参，就该使用默认构造函数，但如果它意味着一个空的std::initializer_list，就该调用`std::initializer_list`构造函数。
`最终会调用默认构造函数`。
空的花括号意味着没有实参，不是一个空的`std::initializer_list`：
```cpp
class Widget { 
public:  
    Widget();                                   //默认构造函数
    Widget(std::initializer_list<int> il);      //std::initializer_list构造函数

    // …                                        //没有隐式转换函数
};
Widget w1;                      //调用默认构造函数
Widget w2{};                    //也调用默认构造函数
Widget w3();                    //最令人头疼的解析！声明一个函数
```
如果你想用`空std::initializer`来调用std::initializer_list构造函数，你就得创建一个空花括号作为函数实参————把空花括号放在圆括号或者另一个花括号内来界定你想传递的东西。
```cpp
Widget w4({});                  //使用空花括号列表调用std::initializer_list构造函数
Widget w5{{}};                  //同上
```
此时，括号初始化，std::initializer_list和构造函数重载的晦涩规则就会一下子涌进你的脑袋，你可能会想研究半天这些东西在你的日常编程中到底占多大比例。
可能比你想象的要有用。因为std::vector作为受众之一会直接受到影响。
`std::vector`有一个`非std::initializer_list构造函数`允许你去指定容器的初始大小，以及使用一个值填满你的容器。
但它也有一个`std::initializer_list构造函数`允许你使用花括号里面的值初始化容器。
如果你创建一个数值类型的`std::vector`（比如std::vector<int>），然后你传递两个实参，把这两个实参放到圆括号和放到花括号中有天壤之别：
```cpp
std::vector<int> v1(10, 20);    //使用非std::initializer_list构造函数
                                //创建一个包含10个元素的std::vector，
                                //所有的元素的值都是20
std::vector<int> v2{10, 20};    //使用std::initializer_list构造函数
                                //创建包含两个元素的std::vector，
                                //元素的值为10和20
```
让我们回到之前的话题。
从以上讨论中我们得出两个重要结论。
* 第一，作为一个类库作者，你需要意识到如果一堆重载的构造函数中有一个或者多个含有`std::initializer_list`形参，用户代码如果使用了括号初始化，可能只会看到你std::initializer_list版本的重载的构造函数。因此，`你最好 把你的 构造函数 设计为 不管用户是使用圆括号 还是 使用花括号 进行初始化 都不会有什么影响`。换句话说，了解了std::vector设计缺点后，你以后设计类的时候应该避免诸如此类的问题。
这里的暗语是如果一个类没有std::initializer_list构造函数，然后你添加一个，用户代码中如果使用括号初始化，可能会发现过去被决议为非std::initializer_list构造函数 而现在被决议为新的函数。
当然，这种事情也可能发生在你添加一个函数到那堆重载函数的时候：过去被决议为旧的重载函数而现在调用了新的函数。
`std::initializer_list`重载不会和其他重载函数比较，它`直接盖过`了其它重载函数，其它重载函数几乎不会被考虑。
**所以如果你要加入std::initializer_list构造函数，请三思而后行。**

* 第二，作为一个类库使用者，你必须认真的在花括号和圆括号之间选择一个来创建对象。
大多数开发者都使用其中一种作为默认情况，只有当他们不能使用这种的时候才会考虑另一种。
默认使用花括号初始化的开发者主要被适用面广、禁止变窄转换、免疫C++最令人头疼的解析这些优点所吸引。
这些开发者知道在一些情况下（比如给定一个容器大小和一个初始值创建std::vector）要使用圆括号。
默认使用圆括号初始化的开发者主要被C++98语法一致性、避免std::initializer_list自动类型推导、避免不会不经意间调用std::initializer_list构造函数这些优点所吸引。
这些开发者也承认有时候只能使用花括号（比如创建一个包含着特定值的容器）。关于花括号初始化和圆括号初始化哪种更好大家没有达成一致，所以我的建议是选择一种并坚持使用它。
如果你是一个模板的作者，花括号和圆括号创建对象就更麻烦了。通常不能知晓哪个会被使用。
举个例子，假如你想创建一个接受任意数量的参数来创建的对象。使用`可变参数模板（variadic template）`可以非常简单的解决：
```cpp
template<typename T,            //要创建的对象类型
         typename... Ts>        //要使用的实参的类型
void doSomeWork(Ts&&... params)
{
    create local T object from params...
    // …
} 
```
在现实中我们有两种方式实现这个伪代码（关于`std::forward`请参见Item25）：
```cpp
T localObject(std::forward<Ts>(params)...);             //使用圆括号
// 二选一
T localObject{std::forward<Ts>(params)...};             //使用花括号
```
考虑这样的调用代码：
```cpp
std::vector<int> v; 
// …
doSomeWork<std::vector<int>>(10, 20);   // 外层传进来两个实参
```
如果`doSomeWork`创建localObject时使用的是`圆括号`，`std::vector`就会包含10个元素。
如果`doSomeWork`创建localObject时使用的是`花括号`，std::vector就会包含2个元素。
哪个是正确的？doSomeWork的作者不知道，只有调用者知道。
这正是标准库函数`std::make_unique`和`std::make_shared`（参见Item21）面对的问题。它们的解决方案是使用`圆括号`，并被记录在文档中作为接口的一部分。（注：更灵活的设计————允许调用者决定从模板来的函数应该使用圆括号还是花括号————是有可能的。详情参见Andrzej’s C++ blog在2013年6月5日的文章，“Intuitive interface — Part I.”）

请记住：
* `花括号初始化`是`最广泛`使用的初始化语法，它`防止变窄转换`，并且对于C++最令人头疼的解析有天生的免疫性
* 在构造函数重载决议中，编译器会尽最大努力将括号初始化与`std::initializer_list`参数匹配，即便其他构造函数看起来是更好的选择
* 对于数值类型的`std::vector`来说使用花括号初始化和圆括号初始化会造成巨大的不同
* 在`模板类`选择使用圆括号初始化或使用花括号初始化创建对象是一个挑战。

# 条款9：优先考虑别名声明而非typedef
我相信每个人都同意使用STL容器是个好主意，并且我希望Item18能说服你让你觉得使用std:unique_ptr也是个好主意，但我猜没有人喜欢写上几次`std::unique_ptr<std::unordered_map<std::string, std::string>>`这样的类型，它可能会让你患上腕管综合征的风险大大增加。
避免上述医疗悲剧也很简单，引入typedef即可：
```cpp
typedef std::unique_ptr<std::unordered_map<std::string, std::string>> UPtrMapSS; 
```
但typedef是C++98的东西。
虽然它可以在C++11中工作，但是C++11也提供了一个`别名声明`（alias declaration）：
```cpp
using UPtrMapSS = std::unique_ptr<std::unordered_map<std::string, std::string>>;
```
由于这里给出的typedef和别名声明做的都是完全一样的事情，我们有理由想知道会不会出于一些技术上的原因两者有一个更好。

这里，在说它们之前我想提醒一下很多人都发现当声明一个函数指针时别名声明更容易理解：
```cpp
// FP 是一个 指向函数的 指针 的同义词，它指向的函数 带有 int 和 const std::string& 形参，不返回任何东西
typedef void (*FP)(int, const std::string&);    //typedef

//含义同上
using FP = void (*)(int, const std::string&);   //别名声明
```
当然，两个结构都不是非常让人满意，没有人喜欢花大量的时间处理函数指针类型的别名（译注：指FP），所以至少在这里，没有一个吸引人的理由让你觉得别名声明比typedef好。
不过有一个地方使用别名声明吸引人的理由是存在的：`模板`。
特别地，别名声明可以被模板化（这种情况下称为别名模板alias templates）但是typedef不能。
这使得C++11程序员可以很直接的表达一些C++98中只能把typedef嵌套进模板化的struct才能表达的东西。
考虑一个`链表的 别名`，链表使用自定义的内存分配器，MyAlloc。
使用别名模板，这真是太容易了：
```cpp
template<typename T>                            //MyAllocList<T>是
using MyAllocList = std::list<T, MyAlloc<T>>;   //std::list<T, MyAlloc<T>>
                                                //的同义词

MyAllocList<Widget> lw;                         //用户代码
```
使用`typedef`，你就只能从头开始（必须在class内部，定义一个`typedef xxx type`）：
```cpp
template<typename T>                            //MyAllocList<T>是
struct MyAllocList {                            //std::list<T, MyAlloc<T>>
    typedef std::list<T, MyAlloc<T>> type;      //的同义词  
};

MyAllocList<Widget>::type lw;                   //用户代码
```
更糟糕的是，如果你想使用在一个模板内使用typedef声明一个链表对象，而这个对象又使用了`模板形参`，你就不得不在typedef前面加上typename：
```cpp
template<typename T>
class Widget {                              //Widget<T>含有一个MyAllocLIst<T>对象作为数据成员
private:
    typename MyAllocList<T>::type list;     // 必须加typename是因为：这种::语法结构带来了歧义：因为也有可能是 静态成员变量 或 静态成员函数
    // …
}; 
```
这里`MyAllocList<T>::type`使用了一个类型，这个类型依赖于模板参数T。

在C++中，当使用`模板类中的嵌套类型`时，编译器无法确定`MyAllocList<T>::type`是一个类型还是一个静态成员变量或函数，因为`MyAllocList<T>`是一个依赖于`模板参数T`的作用域。
！！！编译器在`实例化模板时`才能确定`MyAllocList<T>::type`的具体类型！！！

因此`MyAllocList<T>::type`是一个`依赖类型（dependent type）`，在C++很多讨人喜欢的规则中的一个提到必须要在`依赖类型名`前加上`typename`。
如果使用别名声明定义一个MyAllocList，就不需要使用typename（同时省略麻烦的“::type”后缀）：
```cpp
template<typename T> 
using MyAllocList = std::list<T, MyAlloc<T>>;   //同之前一样

template<typename T> 
class Widget {
private:
    MyAllocList<T> list;                        //没有“typename”
    // …                                        //没有“::type”
};
```
对你来说，`MyAllocList<T>`（使用了模板别名声明的版本）可能看起来和`MyAllocList<T>::type`（使用typedef的版本）一样都应该依赖模板参数T，但是你不是编译器。
！！！当编译器处理Widget模板时遇到`MyAllocList<T>`（使用模板别名声明的版本），***它们知道`MyAllocList<T>`是一个类型名**，因为`MyAllocList`是一个别名模板：它一定是一个类型名。因此`MyAllocList<T>`就是一个非依赖类型（non-dependent type），就不需要也不允许使用typename修饰符。
当编译器在Widget的模板中看到`MyAllocList<T>`::type（使用typedef的版本），它不能确定那是一个类型的名称。
因为可能存在一个MyAllocList的它们没见到的`特化版本`，那个版本的`MyAllocList<T>::type`指代了一种不是类型的东西。
那听起来很不可思议，但不要责备编译器穷尽考虑所有可能。因为人确实能写出这样的代码。
举个例子，一个误入歧途的人可能写出这样的代码：
```cpp
class Wine { … };

template<>                          //当T是Wine
class MyAllocList<Wine> {           //特化MyAllocList
private:  
    enum class WineType             //参见Item10了解  
    { White, Red, Rose };           //"enum class"

    WineType type;                  //在这个类中，type是
    // …                            //一个数据成员！
};
```
就像你看到的，`MyAllocList<Wine>::type`不是一个类型。
如果Widget使用Wine实例化，在Widget模板中的`MyAllocList<Wine>::type`将会是一个数据成员，不是一个类型。
在Widget模板内，`MyAllocList<T>::type`是否表示一个类型`取决于T是什么`，这就是为什么编译器会坚持要求你在前面加上typename。

如果你尝试过`模板元编程（template metaprogramming，TMP）`， 你一定会碰到`取模板类型参数`然后基于它创建另一种类型的情况。
举个例子，给一个类型T，如果你想`去掉T的常量修饰和引用修饰（const- or reference qualifiers）`，比如你想把`const std::string&`变成`std::string`。
又或者你想给一个类型加上`const`或变为`左值引用`，比如把Widget变成`const Widget`或`Widget&`。
（如果你没有用过模板元编程，太遗憾了，因为如果你真的想成为一个高效C++程序员，你需要至少熟悉C++在这方面的基本知识。你可以看看在Item23，27里的TMP的应用实例，包括我提到的类型转换）。
C++11在`type traits（类型特性）`中给了你一系列工具去实现类型转换，如果要使用这些模板请包含`头文件<type_traits>`。
里面有许许多多type traits，也不全是类型转换的工具，也包含一些可预测接口的工具。给一个你想施加转换的类型T，结果类型就是`std::transformation<T>::type`，比如：
```cpp
std::remove_const<T>::type          //从const T中产出T
std::remove_reference<T>::type      //从T&和T&&中产出T
std::add_lvalue_reference<T>::type  //从T中产出T&
```
注释仅仅简单的总结了类型转换做了什么，所以不要太随便的使用。在你的项目使用它们之前，你最好看看它们的详细说明书。
尽管写了一些，但我这里不是想给你一个关于type traits使用的教程。注意类型转换尾部的`::type`。如果你在一个模板内部将他们施加到类型形参上（实际代码中你也总是这么用），你也需要在它们前面加上`typename`。至于**为什么要加`typename`，是因为这些C++11的type traits是通过在struct内嵌套`typedef`来实现的**。是的，它们使用类型同义词（译注：根据上下文指的是使用typedef的做法）技术实现，而正如我之前所说这比别名声明要差。
关于为什么这么实现是有`历史原因`的，但是我们跳过它（我认为太无聊了），因为标准委员会没有及时认识到别名声明是更好的选择，所以直到C++14它们才提供了使用别名声明的版本。
这些别名声明有一个通用形式：对于C++11的类型转换std::transformation<T>::type在C++14中变成了std::transformation_t。举个例子或许更容易理解：
```cpp
std::remove_const<T>::type          //C++11: const T → T 
std::remove_const_t<T>              //C++14 等价形式

std::remove_reference<T>::type      //C++11: T&/T&& → T 
std::remove_reference_t<T>          //C++14 等价形式

std::add_lvalue_reference<T>::type  //C++11: T → T& 
std::add_lvalue_reference_t<T>      //C++14 等价形式
```
C++11的的形式在C++14中也有效，但是我不能理解为什么你要去用它们。
就算你没办法使用C++14，使用别名模板也是小儿科。只需要C++11的语言特性，甚至每个小孩都能仿写，对吧？如果你有一份C++14标准，就更简单了，只需要复制粘贴：
```cpp
template <class T> 
using remove_const_t = typename remove_const<T>::type;

template <class T> 
using remove_reference_t = typename remove_reference<T>::type;

template <class T> 
using add_lvalue_reference_t =
  typename add_lvalue_reference<T>::type; 
```
看见了吧？不能再简单了。
请记住：
* typedef不支持模板化，但是别名声明支持。
* 别名模板避免了使用“::type”后缀，而且在模板中使用typedef还需要在前面加上typename
* C++14提供了C++11所有type traits转换的别名声明版本































# 第五章 右值引用，移动语义，完美转发
当你第一次了解到`移动语义（move semantics）`和`完美转发（perfect forwarding）`的时候，它们看起来非常直观：
移动语义使编译器有可能用廉价的移动操作来代替昂贵的拷贝操作。
正如拷贝构造函数和拷贝赋值操作符给了你控制拷贝语义的权力，移动构造函数和移动赋值操作符也给了你控制移动语义的权力。
移动语义也允许创建`只可移动（move-only）的类型`，例如`std::unique_ptr`，`std::future和std::thread`。
`完美转发`使 `接收任意数量实参的 函数模板`成为可能，它可以将实参转发到其他的函数，使目标函数接收到的实参与被传递给转发函数的实参保持一致。
`右值引用`是连接这两个截然不同的概念的胶合剂。
`它是使移动语义和完美转发变得可能的基础语言机制。`
你对这些特点越熟悉，你就越会发现，你的初印象只不过是冰山一角。移动语义、完美转发和右值引用的世界比它所呈现的更加微妙。
举个例子，`std::move`并不移动任何东西，完美转发也并不完美，也会失败。
`移动操作并不永远比复制操作更廉价；即便如此，它也并不总是像你期望的那么廉价`。
而且，它也并不总是被调用，即使在当移动操作可用的时候。构造“type&&”也并非总是代表一个右值引用。
无论你挖掘这些特性有多深，它们看起来总是还有更多隐藏起来的部分。
幸运的是，它们的深度总是有限的。本章将会带你到最基础的部分。一旦到达，C++11的这部分特性将会具有非常大的意义。
比如，你会掌握`std::move`和`std::forward`的惯用法。
你能够适应“type&&”的歧义性质。你会理解移动操作的令人惊奇的不同表现的背后真相。这些片段都会豁然开朗。在这一点上，你会重新回到一开始的状态，因为移动语义、完美转发和右值引用都会又一次显得直截了当。
但是这一次，它们不再使人困惑。
在本章的这些小节中，非常重要的一点是要牢记`形参永远是左值`，即使它的类型是一个`右值引用`。比如，假设
```cpp
void f(Widget&& w);
```
形参w是一个左值，即使它的类型是一个`rvalue-reference-to-Widget`。（如果这里震惊到你了，请重新回顾从本书简介开始的关于左值和右值的总览。）

# 条款23：理解std::move和std::forward
为了了解`std::move`和`std::forward`，一种有用的方式是从 `它们不做什么` 这个角度来了解它们。
std::move不移动（move）任何东西，std::forward也不转发（forward）任何东西。在运行时，它们不做任何事情。它们不产生任何可执行代码，一字节也没有。
`std::move`和`std::forward`仅仅是执行`转换（cast）的函数（事实上是函数模板）`。
**！！！`std::move`无条件的将它的`实参`转换为`右值`，而`std::forward`只`在特定情况满足时`进行转换**。它们就是如此。
这样的解释带来了一些新的问题，但是从根本上而言，这就是全部内容。
为了使这个故事更加的具体，这里是一个C++11的std::move的示例实现。它并不完全满足标准细则，但是它已经非常接近了。
```cpp
template<typename T>                            //在std命名空间
typename remove_reference<T>::type&&
move(T&& param)
{
    using ReturnType = typename remove_reference<T>::type&&;    //别名声明，见条款9

    return static_cast<ReturnType>(param);
}
```
我为你们高亮了这段代码的两部分（译者注：高亮的部分为函数名`move`和`static_cast<ReturnType>(param)）`。
一个是函数名字，因为函数的返回值非常具有干扰性，而且我不想你们被它搞得晕头转向。
另外一个高亮的部分是包含这段函数的本质的转换。
正如你所见，std::move接受一个对象的引用（准确的说，一个`通用引用`（universal reference），见Item24)，返回一个指向同对象的引用。
该函数返回类型的&&部分表明std::move函数返回的是一个`右值引用`，但是，正如Item28所解释的那样，如果类型T恰好是一个左值引用，那么T&&将会成为一个左值引用。
为了避免如此，`type trait`（见Item9）`std::remove_reference`应用到了类型T上，因此确保了&&被正确的应用到了一个不是引用的类型上。这保证了std::move返回的`真的是右值引用`，这很重要，因为！！！**`函数返回的右值引用`是`右值`**。
因此，std::move将它的实参转换为一个右值，这就是它的全部作用。

此外，std::move在C++14中可以被更简单地实现。
多亏了函数返回值类型推导（见Item3）和标准库的模板别名std::remove_reference_t（见Item9），std::move可以这样写：
```cpp
template<typename T>
decltype(auto) move(T&& param)          //C++14，仍然在std命名空间
{
    using ReturnType = remove_referece_t<T>&&;
    return static_cast<ReturnType>(param);
}
```
看起来更简单，不是吗？
因为`std::move`除了转换它的实参到`右值`以外什么也不做，有一些提议说它的名字叫`rvalue_cast`之类可能会更好。
虽然可能确实是这样，但是它的名字已经是std::move，所以记住std::move做什么和不做什么很重要。`它只进行转换，不移动任何东西`。
当然，右值本来就是移动操作的候选者，所以，对一个对象使用std::move就是告诉编译器，这个对象很适合被移动。
所以这就是为什么`std::move`叫现在的名字：更容易指定可以被移动的对象。
事实上，右值只不过经常是移动操作的候选者。
假设你有一个类，它用来表示一段注解。这个类的构造函数接受一个包含有注解的`std::string`作为形参，然后它复制该形参到数据成员。
假设你了解Item41，你声明一个值传递的形参：

```cpp
class Annotation {
public:
    explicit Annotation(std::string text);  //将会被复制的形参，
    …                                       //如同条款41所说，
};                                          //值传递
```
但是Annotation类的构造函数仅仅是需要读取text的值，它并不需要修改它。
为了和历史悠久的传统：`能使用const就使用const`保持一致，你修订了你的声明以使text变成const：
```cpp
class Annotation {
public:
    explicit Annotation(const std::string text);
    …
};
```
当复制text到一个数据成员的时候，为了避免一次复制操作的代价，你仍然记得来自Item41的建议，把std::move应用到text上，因此产生一个右值：
```cpp
class Annotation {
public:
    explicit Annotation(const std::string text)
    ：value(std::move(text))    //“移动”text到value里；这段代码执行起来
    { … }                       //并不是看起来那样
    
    // …

private:
    std::string value;
};
```
这段代码可以编译，可以链接，可以运行。
这段代码将数据成员value设置为text的值。
这段代码与你期望中的完美实现的唯一区别，是text并不是被移动到value，而是被拷贝。？？？因为`形参text`(类型为`const std::string`)先绑定到实参，然后触发`std::string(const std::string&)`函数
诚然，text通过std::move被转换到右值，但是text被声明为const std::string，所以在转换之前，text是一个`左值的const std::string`，而转换的结果是一个`右值的 const std::string`，但是纵观全程，const属性一直保留。
当编译器决定哪一个`std::string`的构造函数被调用时，考虑它的作用，将会有两种可能性：
```cpp
class string {                  //std::string事实上是std::basic_string<char>的类型别名
public:
    // …
    string(const string& rhs);  //拷贝构造函数
    string(string&& rhs);       //移动构造函数
    // …
};
```
在类Annotation的构造函数的`成员初始化列表中`，`std::move(text)`的结果是一个`const std::string的右值`。
！！！这个右值不能被传递给std::string的`移动构造函数`，因为**移动构造函数只接受一个指向`non-const`的`std::string的右值引用`**。！！！因为移动构造函数的实现往往涉及到这种操作来实现资源的移动或者说窃取：例如将其指针置为空，避免资源被重复释放，所以C++这么规定：右值引用就没法绑定到一个const类型的右值上。
然而，该右值却可以被传递给std::string的`拷贝构造函数`，因为`lvalue-reference-to-const`允许被绑定到一个`const右值`上。因此，std::string在成员初始化的过程中调用了拷贝构造函数，即使text已经被转换成了右值。
这样是为了确保维持const属性的正确性。从一个对象中移动出某个值通常代表着修改该对象，所以语言不允许const对象被传递给可以修改他们的函数（例如移动构造函数）。
从这个例子中，可以总结出两点：
* 第一，不要在你希望能移动对象的时候，声明他们为const。对const对象的移动请求 会悄无声息的被转化为拷贝操作。
* 第二点，`std::move`不仅不移动任何东西，而且它也`不保证它执行转换的对象可以被移动`。
关于std::move，你能确保的唯一一件事就是将它应用到一个对象上，你能够`得到一个右值`。

关于`std::forward`的故事与`std::move`是相似的，但是与`std::move`总是无条件的将它的实参为`右值`不同，`std::forward`只有在满足一定条件的情况下才执行转换。
`std::forward`是有条件的转换。
要明白什么时候它执行转换，什么时候不，想想`std::forward`的典型用法：`最常见的情景是一个 模板函数 ，接收一个 通用引用形参 ，并将它 传递给 另外的函数`：
```cpp
void process(const Widget& lvalArg);        //处理左值
void process(Widget&& rvalArg);             //处理右值

template<typename T>                        //用以转发 param 到 process 的模板
void logAndProcess(T&& param)
{
    auto now = std::chrono::system_clock::now();    // 获取现在时间
    
    makeLogEntry("Calling 'process'", now);
    process(std::forward<T>(param));    // 把形参进行转型，再调用给其他函数
}
```
考虑两次对`logAndProcess`的调用，一次左值为实参，一次右值为实参：
```cpp
Widget w;

logAndProcess(w);               //用左值调用
logAndProcess(std::move(w));    //用右值调用
```
在logAndProcess函数的内部，形参`param`被传递给函数`process`。
函数process分别对左值和右值做了重载。
当我们使用`左值`来调用`logAndProcess`时，自然我们期望`该左值被当作左值`转发给process函数；
而当我们使用`右值`来调用`logAndProcess`函数时，我们期望`process函数的右值重载版本`被调用。
但是`param`，正如所有的其他函数形参一样，是一个`左值`。！！！**一切的函数形参，本身都是左值**
每次在函数logAndProcess内部对函数process的调用，都会因此调用函数process的`左值重载版本`。
为防如此，我们需要一种`机制`：`当且仅当 传递给 函数logAndProcess的 用以初始化param的 实参是一个 右值 时，param会被转换为一个右值`。这就是std::forward做的事情。这就是为什么std::forward是一个有条件的转换：它的`实参`用右值初始化时，转换为一个右值。
你也许会想知道std::forward是怎么知道它的实参是否是被一个右值初始化的。
举个例子，在上述代码中，std::forward是怎么分辨param是被一个左值还是右值初始化的？ 
简短的说，该信息藏在函数logAndProcess的模板`参数T`中。
该参数被传递给了函数std::forward，它解开了含在其中的信息。该机制工作的细节可以查询Item28————`引用折叠机制`。

考虑到`std::move`和`std::forward`都可以归结于`转换`，它们唯一的区别就是std::move总是执行转换，而std::forward偶尔为之。
你可能会问 是否我们可以 免于使用std::move而在任何地方只使用std::forward。
从纯技术的角度，答案是yes：std::forward是可以完全胜任，std::move并非必须。
当然，其实两者中没有哪一个函数是真的必须的，因为我们可以到处直接写转换代码，但是我希望我们能同意：这将相当的，嗯，让人恶心。
`std::move`的吸引力在于它的便利性：减少了出错的可能性，增加了代码的清晰程度。
考虑一个类，我们希望统计有多少次移动构造函数被调用了。
我们只需要一个static的计数器，它会在移动构造的时候自增。
假设在这个类中，唯一一个非静态的数据成员是`std::string`，一种经典的移动构造函数（即，使用std::move）可以被实现如下：
```cpp
class Widget {
public:
    Widget(Widget&& rhs) : s(std::move(rhs.s))  // 触发对std::string对象的移动构造
    { ++moveCtorCalls; }
    // …

private:
    static std::size_t moveCtorCalls;
    std::string s;
};
```
如果要用std::forward来达成同样的效果，代码可能会看起来像：
```cpp
class Widget {
public:
    Widget(Widget&& rhs)                    //不自然，不合理的实现
    : s(std::forward<std::string>(rhs.s))
    { ++moveCtorCalls; }
    // …
}
```
注意，第一，std::move只需要一个函数实参（rhs.s），而std::forward不但需要一个函数实参（rhs.s），还需要一个`模板类型实参std::string`。
其次，我们传递给`std::forward`的类型应当是一个`non-reference`，因为惯例是传递的实参应该是一个`右值`（见Item28）。
同样，这意味着std::move比起std::forward来说需要打更少的字，并且免去了传递一个表示我们正在传递一个右值的类型实参。
同样，它根绝了我们传递错误类型的可能性（例如，`std::string&`可能导致数据成员s被复制而不是被移动构造）。
更重要的是，`std::move`的使用代表着无条件向右值的转换，而！！！使用std::forward只对`绑定了右值的引用`进行到`右值`转换。（std::forward把绑定到右值的右值引用(本身是左值)，转换为右值）
这是两种完全不同的动作。
前者是典型地为了移动操作，而后者只是传递（亦为转发）一个对象到另外一个函数，保留它原有的左值属性或右值属性。
因为这些动作实在是差异太大，所以我们拥有两个不同的函数（以及函数名）来区分这些动作。

请记住：
* std::move执行到右值的无条件的转换，但就自身而言，它不移动任何东西。
* std::forward只有当它的参数被绑定到一个右值时，才将参数转换为右值。
* std::move和std::forward在`运行期`什么也不做。















































# 第7章 并发API
`C++11`的伟大成功之一是将`并发`整合到语言和库中。
熟悉其他线程API（比如pthreads或者Windows threads）的开发者有时可能会对C++提供的斯巴达式（译者注：应该是简陋和严谨的意思）功能集感到惊讶，这是因为 `C++对于并发的大量支持是在对编译器作者约束的层面`。
由此产生的 语言保证 意味着 在C++的历史中，开发者首次通过标准库可以写出跨平台的多线程程序。这为构建表达库奠定了坚实的基础，`标准库并发组件（任务tasks，期望futures，线程threads，互斥mutexes，条件变量condition variables，原子对象atomic objects等）`仅仅是成为并发软件开发者丰富工具集的基础。
在接下来的条款中，记住标准库有两个future的模板：`std::future`和`std::shared_future`。在许多情况下，区别不重要，所以我们经常简单的混于一谈为futures。



# 条款35：优先考虑`基于任务`的编程而非`基于线程`的编程
如果开发者想要异步执行doAsyncWork函数，通常有两种方式。
* 其一
是通过创建`std::thread`执行`doAsyncWork`，这是应用了基于线程（thread-based）的方式：
```cpp
int doAsyncWork();
std::thread t(doAsyncWork);
```

* 其二
是将`doAsyncWork`传递给`std::async`，一种基于任务（task-based）的策略：
```cpp
auto fut = std::async(doAsyncWork); //“fut”表示“future”
```
这种方式中，传递给std::async的函数对象被称为一个任务（task）。

基于任务的方法通常比基于线程的方法更优，原因之一上面的代码已经表明，基于任务的方法代码量更少。
我们假设调用doAsyncWork的代码`对于其提供的返回值是有需求的`。
基于线程的方法对此无能为力，而基于任务的方法就简单了，因为std::async返回的future提供了get函数（从而可以获取返回值）。
**如果`doAsycnWork`发生了异常，get函数就显得更为重要**，因为`get函数`可以`提供抛出异常的访问`，而基于线程的方法，如果doAsyncWork抛出了异常，程序会直接终止（通过调用std::terminate）。


**基于线程与基于任务最根本的区别在于，`基于任务`的`抽象层次更高`。**
基于任务的方式使得开发者从线程管理的细节中解放出来，对此在C++并发软件中总结了“thread”的三种含义：
- `硬件线程（hardware threads）`
    是真实执行计算的线程。现代计算机体系结构为每个CPU核心提供一个或者多个硬件线程。
- `软件线程（software threads）`
    （也被称为系统线程（OS threads、system threads））是操作系统（假设有一个操作系统。有些嵌入式系统没有。）管理的在硬件线程上执行的线程。
    通常可以存在比硬件线程更多数量的软件线程，因为当软件线程被阻塞的时候（比如 I/O、同步锁或者条件变量），操作系统可以调度其他未阻塞的软件线程执行提供`吞吐量`。
- `std::thread`
    是C++执行过程的对象，并作为软件线程的句柄（handle），用户层可以对该线程进行各种操作。
    有些std::thread对象代表“空”句柄，即没有对应软件线程，因为它们处在默认构造状态（即没有函数要执行）；
    有些被移动走（移动到的std::thread就作为这个软件线程的句柄）；
    有些被join（它们要运行的函数已经运行完）；
    有些被detach（它们和对应的软件线程之间的连接关系被打断）。

`软件线程`是`有限的`资源。
如果开发者试图创建大于系统支持的线程数量，会抛出`std::system_error`异常。
即使你编写了不抛出异常的代码，这仍然会发生，比如下面的代码，即使 doAsyncWork是 noexcept，
```cpp
int doAsyncWork() noexcept;         //noexcept见条款14
// 这段代码仍然会抛出异常：
std::thread t(doAsyncWork);         //如果没有更多线程可用，则抛出异常
```
设计良好的软件必须能有效地处理这种可能性，但是怎样做？
- 一种方法是在`当前线程`执行doAsyncWork，但是这可能会导致负载不均，而且如果当前线程是GUI线程，可能会导致响应时间过长的问题。
- 另一种方法是等待某些当前运行的软件线程结束之后再创建新的std::thread，但是仍然有可能当前运行的线程在`等待`doAsyncWork的动作（例如产生一个结果或者报告一个条件变量）。

即使没有超出软件线程的限额，仍然可能会遇到`资源超额`（oversubscription）的麻烦。
这是一种`当前准备运行的（即未阻塞的）软件线程` 大于 `硬件线程的数量`的情况。
情况发生时，`线程调度器`（操作系统的典型部分）会将软件线程`时间切片`，分配到硬件上。
当一个软件线程的时间片执行结束，会让给另一个软件线程，此时发生`上下文切换`。
`软件线程的上下文切换`会增加系统的`软件线程``管理开销`，**当软件线程安排到与上次时间片运行时不同的硬件线程上，这个开销会更高**。
这种情况下，
* （1）CPU缓存对这个软件线程很冷淡（即几乎没有什么数据，也没有有用的操作指南）；
* （2）“新”软件线程的缓存数据会“污染”“旧”线程的数据，旧线程之前运行在这个核心上，而且还有可能再次在这里运行。
避免资源超额很困难，因为`软件线程 之于 硬件线程的 最佳比例`取决于`软件线程的执行频率`，那是动态改变的，比如一个程序从IO密集型变成计算密集型，执行频率是会改变的。
而且比例还依赖`上下文切换的开销`以及`软件线程对于 CPU缓存 的使用效率`。
此外，`硬件线程的数量`和`CPU缓存的细节`（比如缓存多大，相应速度多少）取决于`机器的体系结构`，即使经过调校，在某一种机器平台避免了资源超额（而仍然保持硬件的繁忙状态），换一个其他类型的机器这个调校并不能提供较好效果的保证。
如果你把这些问题推给另一个人做，你就会变得很轻松，而使用`std::async`就做了这件事：
```cpp
auto fut = std::async(doAsyncWork); //线程管理责任交给了标准库的开发者

std::future<int> fut = std::async(std::launch::async, some_function);   // 只有这样设置，才必然在新线程中执行

// std::launch::deferred 参数则是同步执行
```
这种调用方式将线程管理的职责转交给C++标准库的开发者。
举个例子，这种调用方式会`减少抛出资源超额异常的可能性`，因为`这个调用可能不会开启一个新的线程`。
你会想：“怎么可能？如果我要求比系统可以提供的更多的软件线程，创建`std::thread`和调用`std::async`为什么会有区别？
”确实有区别，因为以这种形式调用（即使用`默认启动策略`——见Item36）时，`std::async`不保证会创建新的软件线程。
然而，他们允许通过`调度器`来将特定函数（本例中为doAsyncWork）运行在等待此函数结果的线程上（即在对fut调用get或者wait的线程上），合理的调度器在系统资源超额或者线程耗尽时就会利用这个自由度。

如果考虑自己实现“在等待结果的线程上运行输出结果的函数”，之前提到了可能引出负载不均衡的问题，这问题不那么容易解决，因为 应该是`std::async`和`运行时的调度程序`来解决这个问题而不是你。
遇到负载不均衡问题时，对机器内发生的事情，运行时调度程序比你有更全面的了解，因为它管理的是所有执行过程，而不仅仅个别开发者运行的代码。

有了std::async，GUI线程中响应变慢仍然是个问题，因为调度器并不知道你的哪个线程有高响应要求。
这种情况下，你会想通过向std::async传递`std::launch::async`启动策略来保证想运行函数在不同的线程上执行（见Item36）。
`最前沿的线程调度器`使用`系统级线程池（thread pool）`来避免资源超额的问题，并且通过`工作窃取算法（work-stealing algorithm）`来提升了跨硬件核心的负载均衡。
C++标准实际上并不要求使用线程池或者工作窃取，实际上C++11并发规范的某些技术层面使得实现这些技术的难度可能比想象中更有挑战。
不过，库开发者在标准库实现中采用了这些技术，也有理由期待这个领域会有更多进展。
如果你当前的并发编程采用基于任务的方式，在这些技术发展中你会持续获得回报。
相反如果你直接使用std::thread编程，处理线程耗尽、资源超额、负载均衡问题的责任就压在了你身上，更不用说你对这些问题的解决方法与同机器上其他程序采用的解决方案配合得好不好了。

对比基于线程的编程方式，基于任务的设计为开发者`避免了手动线程管理的痛苦`，并且自然提供了一种获取异步执行程序的结果（即返回值或者异常）的方式。

当然，仍然存在一些场景`直接使用std::thread会更有优势`：
- 你需要访问 非常基础的 线程API。
- C++并发API通常是通过操作系统提供的系统级API（`pthreads`或者`Windows threads`）来实现的，系统级API通常会提供更加灵活的操作方式（举个例子，C++没有线程优先级和亲和性的概念）。
`为了提供对底层系统级线程API的访问`，`std::thread`对象提供了`native_handle`的成员函数，而std::future（即std::async返回的东西）没有这种能力。
你需要且能够优化应用的线程使用。
举个例子，你要开发一款已知执行概况的服务器软件，部署在有固定硬件特性的机器上，作为唯一的关键进程。
你需要实现C++并发API之外的线程技术，比如，C++实现中未支持的平台的线程池。
这些都是在应用开发中并不常见的例子，大多数情况，开发者应该优先采用基于任务的编程方式。

请记住：
- std::thread API不能直接访问异步执行的结果，如果执行函数有异常抛出，代码会终止执行。
- 基于线程的编程方式需要手动的线程耗尽、资源超额、负责均衡、平台适配性管理。
- 通过带有默认启动策略的std::async进行基于任务的编程方式会解决大部分问题。





























