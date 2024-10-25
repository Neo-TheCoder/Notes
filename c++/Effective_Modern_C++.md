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
更糟糕的是，如果你想使用在一个模板内使用`typedef`声明一个链表对象，而这个对象又使用了`模板形参`，你就不得不在typedef前面加上typename：
```cpp
template<typename T>
class Widget {                              //Widget<T>含有一个MyAllocLIst<T>对象作为数据成员
private:
    typename MyAllocList<T>::type list;     // 必须加typename是因为：这种::语法结构带来了歧义：因为 type 也有可能是 静态成员变量 或 静态成员函数
    // …
};
```
这里`MyAllocList<T>::type`使用了一个类型，这个类型依赖于模板参数T。

在C++中，当使用`模板类中的嵌套类型`时，编译器无法确定`MyAllocList<T>::type`是一个类型还是一个静态成员变量或函数，因为`MyAllocList<T>`是一个依赖于`模板参数T`的作用域。
！！！编译器在`实例化模板时`才能确定`MyAllocList<T>::type`的具体类型！！！

因此`MyAllocList<T>::type`是一个`依赖类型（dependent type）`，在C++很多讨人喜欢的规则中的一个提到必须要在`依赖类型名`前加上`typename`。
如果使用别名声明定义一个MyAllocList，就不需要使用`typename`（同时省略麻烦的“::type”后缀）：
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
！！！当编译器处理Widget模板时遇到`MyAllocList<T>`（使用模板别名声明的版本），***它们知道`MyAllocList<T>`是一个类型名**，因为`MyAllocList`是一个别名模板：它一定是一个类型名。因此`MyAllocList<T>`就是一个`非依赖类型（non-dependent type）`，就不需要也不允许使用typename修饰符。
当编译器在Widget的模板中看到`MyAllocList<T>::type`（使用typedef的版本），它不能确定那是一个类型的名称。
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



# 条款12：使用override声明重写函数
在C++面向对象的世界里，涉及的概念有类，继承，虚函数。
这个世界最基本的概念是`派生类的虚函数 重写 基类同名函数`。令人遗憾的是虚函数重写可能一不小心就错了。
似乎这部分语言的设计理念是不仅仅要遵守墨菲定律，还应该尊重它。
虽然“重写（overriding）”听起来像“重载（overloading）”，然而两者完全不相关，所以让我澄清一下，正是虚函数重写机制的存在，才使我们可以通过基类的接口调用派生类的成员函数：
```cpp
class Base {
public:
    virtual void doWork();          //基类虚函数
    …
};

class Derived: public Base {
public:
    virtual void doWork();          //重写Base::doWork
    …                               //（这里“virtual”是可以省略的）
}; 

std::unique_ptr<Base> upb =         //创建基类指针指向派生类对象
    std::make_unique<Derived>();    //关于std::make_unique
…                                   //请参见Item21


upb->doWork();                      //通过基类指针调用doWork，
                                    //实际上是派生类的doWork
                                    //函数被调用
```
要想重写一个函数，必须满足下列要求：
* 基类函数必须是virtual
* 基类和派生类函数名必须完全一样（除非是析构函数)
* 基类和派生类函数形参类型必须完全一样
* 基类和派生类函数常量性constness必须完全一样
* 基类和派生类函数的返回值和异常说明（exception specifications）必须兼容
除了这些C++98就存在的约束外，C++11又添加了一个：
* 函数的`引用限定符（reference qualifiers）`必须完全一样。
成员函数的`引用限定符`是C++11很少抛头露脸的特性，所以如果你从没听过它无需惊讶。它可以`限定成员函数只能用于左值或者右值`。
成员函数不需要virtual也能使用它们：
```cpp
class Widget {
public:
    …
    void doWork() &;    //只有*this为左值的时候才能被调用
    void doWork() &&;   //只有*this为右值的时候才能被调用
}; 

…
Widget makeWidget();    //工厂函数（返回右值）
Widget w;               //普通对象（左值）
…
w.doWork();             //调用被左值引用限定修饰的Widget::doWork版本
                        //（即Widget::doWork &）
makeWidget().doWork();  //调用被右值引用限定修饰的Widget::doWork版本
                        //（即Widget::doWork &&）
```
后面我还会提到引用限定符修饰成员函数，但是现在，只需要记住如果基类的虚函数有引用限定符，派生类的重写就必须具有相同的引用限定符。
如果没有，那么新声明的函数还是属于派生类，但是不会重写父类的任何函数。
这么多的重写需求意味着哪怕一个小小的错误也会造成巨大的不同。
代码中包含重写错误通常是有效的，但它的意图不是你想要的。
因此你不能指望当你犯错时编译器能通知你。比如，下面的代码是完全合法的，咋一看，还很有道理，但是它没有任何虚函数重写————没有一个派生类函数联系到基类函数。你能识别每种情况的错误吗，换句话说，为什么派生类函数没有重写同名基类函数？
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
需要一点帮助吗？
mf1在Base基类声明为const，但是Derived派生类没有这个常量限定符
mf2在Base基类声明为接受一个int参数，但是在Derived派生类声明为接受unsigned int参数
mf3在Base基类声明为左值引用限定，但是在Derived派生类声明为右值引用限定
mf4在Base基类没有声明为virtual虚函数

你可能会想，“哎呀，实际操作的时候，这些warnings都能被编译器探测到，所以我不需要担心。”你说的可能对，也可能不对。就我目前检查的两款编译器来说，这些代码编译时没有任何warnings，即使我开启了输出所有warnings。（其他编译器可能会为这些问题的部分输出warnings，但不是全部。）
（也就是说，编译器不一定认为这是一种错误）
由于正确声明派生类的重写函数很重要，但很容易出错，C++11提供一个方法让你可以显式地指定一个派生类函数是基类版本的重写：将它声明为`override`。
还是上面那个例子，我们可以这样做：
```cpp
class Derived: public Base {
public:
    virtual void mf1() override;
    virtual void mf2(unsigned int x) override;
    virtual void mf3() && override;
    virtual void mf4() const override;
};
```
代码不能编译，当然了，因为这样写的时候，编译器会抱怨所有与重写有关的问题。
这也是你想要的，以及为什么要在所有重写函数后面加上override。
使用override的代码编译时看起来就像这样（假设我们的目的是Derived派生类中的所有函数重写Base基类的相应虚函数）:
```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    virtual void mf4() const;
};

class Derived: public Base {
public:
    virtual void mf1() const override;
    virtual void mf2(int x) override;
    virtual void mf3() & override;
    void mf4() const override;          //可以添加virtual，但不是必要
}; 
```
注意在这个例子中mf4有别于之前，它在Base中的声明有virtual修饰，所以能正常工作。
大多数和重写有关的错误都是在`派生类`引发的，但也可能是基类的不正确导致。

比起让编译器（译注：通过warnings）告诉你想重写的而实际没有重写，不如给你的派生类重写函数全都加上override。
如果你考虑修改修改基类虚函数的函数签名，override还可以帮你评估后果。
如果派生类全都用上override，你可以只改变基类函数签名，重编译系统，再看看你造成了多大的问题（即，多少派生类不能通过编译），然后决定是否值得如此麻烦更改函数签名。
没有override，你只能寄希望于完善的单元测试，因为，正如我们所见，派生类虚函数本想重写基类，但是没有，编译器也没有探测并发出诊断信息。
C++既有很多关键字，C++11引入了两个上下文关键字（contextual keywords），`override`和`final`（**向`虚函数`添加`final`可以`防止派生类重写`**。final也能用于类，这时这个类不能用作基类）。
这两个关键字的特点是它们是保留的，它们只是位于特定上下文才被视为关键字。对于override，它只在成员函数声明结尾处才被视为关键字。这意味着如果你以前写的代码里面已经用过override这个名字，那么换到C++11标准你也无需修改代码：
```cpp
class Warning {         //C++98潜在的传统类代码
public:
    …
    void override();    //C++98和C++11都合法（且含义相同）
    …
};
```
关于override想说的就这么多，但对于`成员函数引用限定（reference qualifiers）`还有一些内容。我之前承诺我会在后面提供更多的关于它们的资料，现在就是"后面"了。
如果我们想写一个函数只接受左值实参，我们声明一个non-const左值引用形参：
```cpp
void doSomething(Widget& w);    //只接受左值Widget对象
```
如果我们想写一个函数只接受右值实参，我们声明一个右值引用形参：
```cpp
void doSomething(Widget&& w);   //只接受右值Widget对象
```
成员函数的引用限定可以很容易的区分一个成员函数被哪个对象（即`*this`）调用。它和在成员函数声明尾部添加一个`const`很相似，暗示了`调用这个成员函数的对象（即*this）是const的`。
对成员函数添加引用限定不常见，但是可以见。举个例子，假设我们的Widget类有一个std::vector数据成员，我们提供一个访问函数让客户端可以直接访问它：
```cpp
class Widget {
public:
    using DataType = std::vector<double>;   //“using”的信息参见Item9
    …
    DataType& data() { return values; }
    …
private:
    DataType values;
};
```
这是最具封装性的设计，只给外界保留一线光。但先把这个放一边，思考一下下面的客户端代码：
```cpp
Widget w;
…
auto vals1 = w.data();  //拷贝w.values到vals1
```
`Widget::data`函数的返回值是一个左值引用（准确的说是`std::vector<double>&`）, 因为左值引用是左值，所以vals1是从左值初始化的。
因此vals1由w.values拷贝构造而得，就像注释说的那样。
现在假设我们有一个创建Widgets的工厂函数，
```cpp
Widget makeWidget();
```
我们想用makeWidget返回的Widget里的std::vector初始化一个变量：
```cpp
auto vals2 = makeWidget().data();   //拷贝Widget里面的值到vals2
```
再说一次，`Widgets::data`返回的是左值引用，还有，左值引用是左值。
所以，我们的对象（vals2）得从Widget里的values拷贝构造。
这一次，Widget是makeWidget返回的临时对象（即`右值`），所以将其中的`std::vector`进行拷贝纯属浪费。最好是`移动`，但是因为data返回左值引用，C++的规则要求编译器不得不生成一个拷贝。（这其中有一些优化空间，被称作“as if rule”，但是你依赖编译器使用这个优化规则就有点傻。）（译注：`“as if rule”简单来说就是在不影响程序的“外在表现”情况下做一些改变`）
我们需要的是指明当data被右值Widget对象调用的时候结果也应该是一个右值。
现在就可以使用引用限定，为左值Widget和右值Widget写一个data的重载函数来达成这一目的：
```cpp
class Widget {
public:
    using DataType = std::vector<double>;
    …
    DataType& data() &              //对于左值Widgets,
    { return values; }              //返回左值
    
    DataType data() &&              //对于右值Widgets,
    { return std::move(values); }   //返回右值
    …

private:
    DataType values;
};
```
注意data重载的返回类型是不同的，左值引用重载版本返回一个左值引用（即一个左值），右值引用重载返回一个临时对象（即一个右值）。
这意味着现在客户端的行为和我们的期望相符了：
```cpp
auto vals1 = w.data();              //调用左值重载版本的Widget::data，
                                    //拷贝构造vals1
auto vals2 = makeWidget().data();   //调用右值重载版本的Widget::data, 
                                    //移动构造vals2
```
这真的很棒，但别被这结尾的暖光照耀分心以致忘记了该条款的中心。这个条款的中心是只要你在派生类声明想要重写基类虚函数的函数，就加上override。

请记住：
* 为重写函数加上`override`
* 成员函数引用限定让我们可以区别对待左值对象和右值对象(即`*this`)



# 条款15：尽可能的使用constexpr
如果要给C++11颁一个“最令人困惑新词”奖，`constexpr`十有八九会折桂。
当用于对象上面，它本质上就是`const的加强形式`，但是当它用于`函数`上，意思就大不相同了。有必要消除困惑，因为你绝对会用它的，特别是当你发现constexpr “正合吾意”的时候。
从概念上来说，constexpr表明一个值不仅仅是`常量`，还是`编译期可知的`。
这个表述并不全面，因为当constexpr被用于函数的时候，事情就有一些细微差别了。
为了避免我毁了结局带来的surprise，我现在只想说，**你不能假设`constexpr函数`的结果是`const`，也`不能保证`它们的（译注：返回）值是在`编译期可知`的**。
最有意思的是，这些是特性。关于constexpr函数返回的结果不需要是const，也不需要编译期可知这一点是良好的行为！
不过我们还是先从constexpr对象开始说起。
这些对象，实际上，和const一样，它们是`编译期可知`的。（技术上来讲，它们的值在`翻译期（translation）决议`，所谓翻译不仅仅包含是`编译（compilation）`也包含`链接（linking）`，除非你准备写C++的编译器和链接器，否则这些对你不会造成影响，所以你编程时无需担心，把这些constexpr对象值看做编译期决议也无妨的。）
编译期可知的值“享有特权”，它们可能被存放到`只读存储空间`中。！！！
对于那些`嵌入式系统`的开发者，这个特性是相当重要的。
更广泛的应用是：`“其值编译期可知”的 常量整数`会出现在需要“`整型常量表达式（integral constant expression）的 上下文`中，这类上下文包括`数组大小`，`整数模板参数`（包括`std::array对象的长度`），`枚举名的值`，`对齐修饰符`（译注：`alignas(val)`），等等。
如果你想在这些上下文中使用变量，你一定会希望将它们声明为`constexpr`，因为编译器会确保它们是`编译期可知`的：
```cpp
int sz;                             //non-constexpr变量
// …
constexpr auto arraySize1 = sz;     //错误！sz的值在编译期不可知

std::array<int, sz> data1;          //错误！一样的问题
constexpr auto arraySize2 = 10;     //没问题，10是编译期可知常量

std::array<int, arraySize2> data2;  //没问题, arraySize2是constexpr
```
注意`const不提供constexpr所能保证之事`，因为const对象不需要在编译期初始化它的值。
```cpp
int sz;                            //和之前一样
// …
const auto arraySize = sz;         //没问题，arraySize是sz的const复制   constexpr必须保证表达式的值编译期可知，比const更强烈
std::array<int, arraySize> data;   //错误，arraySize值在编译期不可知
```
简而言之，`所有constexpr对象都是const，但不是所有const对象都是constexpr`。
如果你想编译器保证一个变量有一个值，这个值可以放到那些需要编译期常量（compile-time constants）的上下文的地方，你需要的工具是constexpr而不是const。
涉及到`constexpr函数`时，constexpr对象的使用情况就更有趣了。
如果实参是`编译期常量`，这些函数将产出`编译期常量`；如果实参是`运行时才能知道的值`，它们就将产出`运行时值`。
这听起来就像你不知道它们要做什么一样，那么想是错误的，请这么看：
`constexpr函数`可以用于：`需求编译期常量的上下文`。
如果你传给constexpr函数的实参在编译期可知，那么结果将在编译期计算。
如果实参的值在编译期不知道，你的代码就会被拒绝。
当一个constexpr函数被一个或者多个编译期不可知值调用时，它就像`普通函数`一样，运行时计算它的结果。这意味着`你不需要两个函数，一个用于编译期计算，一个用于运行时计算。constexpr全做了`。
假设我们需要一个数据结构来存储一个实验的结果，而这个实验可能以各种方式进行。
实验期间风扇转速，温度等等都可能导致亮度值改变，亮度值可以是高，低，或者无。
如果有n个实验相关的环境条件，它们每一个都有三个状态，最终可以得到的组合有3^n个。
储存所有实验结果的所有组合需要足够存放3^n个值的数据结构。
假设每个结果都是int并且n是编译期已知的（或者可以被计算出的），一个std::array是一个合理的选择。
我们需要一个方法`在编译期 计算3^n`。C++标准库提供了`std::pow`，它的数学功能正是我们所需要的，但是，对我们来说，这里还有两个问题。
第一，std::pow是为浮点类型设计的，我们需要整型结果。
第二，std::pow不是constexpr（即，不保证使用编译期可知值调用而得到编译期可知的结果），所以我们不能用它作为std::array的大小。
幸运的是，我们可以应需写个pow。我将展示怎么快速完成它，不过现在让我们先看看它应该怎么被声明和使用：
```cpp
constexpr                                   //pow是 绝不抛异常的
int pow(int base, int exp) noexcept         //constexpr函数
{
 …                                          //实现在下面
}
constexpr auto numConds = 5;                //（上面例子中）条件的个数
std::array<int, pow(3, numConds)> results;  //结果有3^numConds个元素
```
回忆下pow前面的constexpr`不表明pow返回一个const值`，它只说了 `如果base和exp是编译期常量，pow的值可以被当成编译期常量使用`。
如果base和/或exp不是编译期常量，pow结果将会在运行时计算。这意味着pow不止可以用于像`std::array`的大小这种需要编译期常量的地方，它也可以用于运行时环境：
```cpp
auto base = readFromDB("base");     //运行时获取这些值
auto exp = readFromDB("exponent"); 
auto baseToExp = pow(base, exp);    //运行时调用pow函数
```
`因为constexpr函数必须能在编译期值调用的时候返回编译期结果，就必须对它的实现施加一些限制`。这些限制在C++11和C++14标准间有所出入。
C++11中，constexpr函数的代码不超过一行语句：一个return。听起来很受限，但实际上有两个技巧可以扩展constexpr函数的表达能力。
第一，使用三元运算符“?:”来代替if-else语句，第二，使用递归代替循环。因此pow可以像这样实现：
```cpp
constexpr int pow(int base, int exp) noexcept
{
    return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
```
这样没问题，但是很难想象除了使用函数式语言的程序员外会觉得这样硬核的编程方式更好。在C++14中，constexpr函数的限制变得非常宽松了，所以下面的函数实现成为了可能：
```cpp
constexpr int pow(int base, int exp) noexcept   //C++14
{
    auto result = 1;
    for (int i = 0; i < exp; ++i)
        result *= base;
    return result;
}
```
constexpr函数限制为只能获取和返回字面值类型，这基本上意味着那些有了值的类型能在编译期决定。
在C++11中，除了void外的所有内置类型，以及一些用户定义类型都可以是字面值类型，因为构造函数和其他成员函数可能是constexpr：
```cpp
class Point {
public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept
    : x(xVal), y(yVal)
    {}

    constexpr double xValue() const noexcept { return x; } 
    constexpr double yValue() const noexcept { return y; }

    void setX(double newX) noexcept { x = newX; }
    void setY(double newY) noexcept { y = newY; }

private:
    double x, y;
};
```
Point的构造函数可被声明为`constexpr`，因为 `如果传入的参数在编译期可知，Point的数据成员也能在编译器可知`。
因此这样初始化的Point就能为constexpr：
```cpp
constexpr Point p1(9.4, 27.7);  //没问题，constexpr构造函数
                                //会在编译期“运行”
constexpr Point p2(28.8, 5.3);  //也没问题
```

类似的，xValue和yValue的`getter（取值器）函数`也能是constexpr，因为如果对一个编译期已知的Point对象（如一个constexpr Point对象）调用getter，数据成员x和y的值也能在编译期知道。
这使得我们可以写一个constexpr函数，里面调用Point的getter并初始化constexpr的对象：
```cpp
constexpr
Point midpoint(const Point& p1, const Point& p2) noexcept
{
    return { (p1.xValue() + p2.xValue()) / 2,   //调用constexpr
             (p1.yValue() + p2.yValue()) / 2 }; //成员函数
}
constexpr auto mid = midpoint(p1, p2);      //使用constexpr函数的结果
                                            //初始化constexpr对象
```
这太令人激动了。它意味着mid对象通过调用构造函数，getter和非成员函数来进行初始化过程就能在`只读内存`中被创建出来！
它也意味着你可以在`模板实参`或者`需要枚举名的值的表达式`里面使用像`mid.xValue() * 10`的表达式！（因为`Point::xValue返回double，mid.xValue() * 10`也是个double。
**浮点数类型不可被用于实例化模板或者说明枚举名的值，但是它们可以被用来作为产生整数值的大表达式的一部分。**
比如，`static_cast<int>(mid.xValue() * 10)`可以被用来实例化模板或者说明枚举名的值。）它也意味着 **以前相对严格的编译期完成的工作和运行时完成的工作的界限变得模糊，一些传统上在运行时的计算过程能并入编译时**。
越多这样的代码并入，你的程序就越快。（然而，编译会花费更长时间）
在C++11中，有两个限制使得Point的成员函数setX和setY不能声明为constexpr。
第一，它们修改它们操作的对象的状态， 并且**在C++11中，`constexpr成员函数`是隐式的`const`**。
第二，它们有void返回类型，void类型不是C++11中的字面值类型。
这两个限制在C++14中放开了，所以C++14中Point的setter（赋值器）也能声明为constexpr：
```cpp
class Point {
public:
    // …
    constexpr void setX(double newX) noexcept { x = newX; } //C++14
    constexpr void setY(double newY) noexcept { y = newY; } //C++14
    // …
};
```
现在也能写这样的函数：
```cpp
//返回p相对于原点的镜像
constexpr Point reflection(const Point& p) noexcept
{
    Point result;                   //创建non-const Point
    result.setX(-p.xValue());       //设定它的x和y值
    result.setY(-p.yValue());
    return result;                  //返回它的副本
}
```
客户端代码可以这样写：
```cpp
constexpr Point p1(9.4, 27.7);          //和之前一样
constexpr Point p2(28.8, 5.3);
constexpr auto mid = midpoint(p1, p2);

constexpr auto reflectedMid =         //reflectedMid的值
    reflection(mid);                  //(-19.1, -16.5)在编译期可知
```
本条款的建议是尽可能的使用`constexpr`，现在我希望大家已经明白缘由：constexpr对象和constexpr函数可以使用的范围比non-constexpr对象和函数大得多。
使用constexpr关键字可以最大化你的对象和函数可以使用的场景。
还有个重要的需要注意的是 `constexpr是对象和函数接口的一部分`。**加上constexpr相当于宣称“我能被用在C++要求常量表达式的地方”**。
如果你声明一个对象或者函数是constexpr，客户端程序员就可能会在那些场景中使用它。
如果你后面认为使用constexpr是一个错误并想移除它，你可能造成大量客户端代码不能编译。（为了debug或者性能优化而添加I/O到一个函数中这样简单的动作可能就导致这样的问题，因为I/O语句一般不被允许出现在constexpr函数里）“尽可能”的使用constexpr表示你需要长期坚持对某个对象或者函数施加这种限制。
请记住：
* constexpr对象是const，它被在编译期可知的值初始化
* 当传递编译期可知的值时，constexpr函数可以产出编译期可知的结果
* constexpr对象和函数可以使用的范围比non-constexpr对象和函数要大
* constexpr是对象和函数接口的一部分



# 条款16：让const成员函数线程安全
如果我们在数学领域中工作，我们就会发现用一个类表示多项式是很方便的。
在这个类中，使用一个函数来计算多项式的根是很有用的，也就是多项式的值为零的时候（译者注：通常也被叫做零点，即使得多项式值为零的那些取值）。
这样的一个函数它不会更改多项式。所以，它自然被声明为`const函数`。
```cpp
class Polynomial {
public:
    using RootsType =           //数据结构保存多项式为零的值
          std::vector<double>;  //（“using” 的信息查看条款9）
    // …
    RootsType roots() const;
    // …
};
```
计算多项式的根是很复杂的，因此如果不需要的话，我们就不做。
如果必须做，我们肯定不想再做第二次。
所以，如果必须计算它们，就`缓存多项式的根`，然后实现roots来返回缓存的值。下面是最基本的实现：
```cpp
class Polynomial {
public:
    using RootsType = std::vector<double>;
    
    RootsType roots() const
    {
        if (!rootsAreValid) {               //如果缓存不可用
            // …                            //计算根
                                            //用rootVals存储它们
            rootsAreValid = true;
        }
        
        return rootVals;
    }
    
private:
    mutable bool rootsAreValid{ false };    //初始化器（initializer）的更多信息请查看条款7
    mutable RootsType rootVals{};
};
```
从概念上讲，roots并不改变它所操作的Polynomial对象。
但是作为缓存的一部分，它也许会改变rootVals和rootsAreValid的值。
这就是`mutable`的经典使用样例，这也是为什么它是数据成员声明的一部分。
假设现在有两个线程同时调用Polynomial对象的roots方法:
```cpp
Polynomial p;
// …

/*------ Thread 1 ------*/      /*-------- Thread 2 --------*/
auto rootsOfp = p.roots();      auto valsGivingZero = p.roots();
```
这些用户代码是非常合理的。
roots是const成员函数，那就表示着它是一个`读操作`。
**在没有同步的情况下，让`多个线程`执行`读操作`是安全的**。它最起码应该做到这点。
在本例中却没有做到线程安全。
因为在roots中，这些线程中的一个或两个可能尝试修改成员变量rootsAreValid和rootVals。
这就意味着在没有同步的情况下，这些代码会有不同的线程读写相同的内存，这就是数据竞争（data race）的定义。这段代码的行为是未定义的。
问题就是roots被`声明为const`，但`不是线程安全的`。
const声明在C++11中与在C++98中一样正确（检索多项式的根并不会更改多项式的值），因此需要纠正的是线程安全的缺乏。

解决这个问题最普遍简单的方法就是——使用`mutex（互斥量）`：
```cpp
class Polynomial {
public:
    using RootsType = std::vector<double>;
    
    RootsType roots() const
    {
        std::lock_guard<std::mutex> g(m);       //锁定互斥量
        
        if (!rootsAreValid) {                   //如果缓存无效
            …                                   //计算/存储根值
            rootsAreValid = true;
        }
        
        return rootsVals;
    }                                           //解锁互斥量
    
private:
    mutable std::mutex m;
    mutable bool rootsAreValid { false };
    mutable RootsType rootsVals {};
};
```
`std::mutex m`被声明为`mutable`，因为锁定和解锁它的都是non-const成员函数。在roots（const成员函数）中，m却被视为const对象。
值得注意的是，因为std::mutex是一种只可移动类型（move-only type，一种可以移动但不能复制的类型），所以将m添加进Polynomial中的副作用是使Polynomial失去了被复制的能力。
不过，它仍然可以移动。 （译者注：`实际上 std::mutex 既不可移动，也不可复制。因而包含他们的类也同时是不可移动和不可复制的。`）
在某些情况下，`互斥量的副作用`显会得过大。
例如，如果你所做的只是计算成员函数被调用了多少次，使用`std::atomic 修饰的计数器`（保证其他线程视它的操作为不可分割的整体，参见item40）通常会是一个开销更小的方法。
（然而它是否轻量取决于你使用的硬件和标准库中互斥量的实现。）以下是如何使用`std::atomic`来统计调用次数。
```cpp
class Point {                                   //2D点
public:
    …
    double distanceFromOrigin() const noexcept  //noexcept的使用
    {                                           //参考条款14
        ++callCount;                            //atomic的递增
        
        return std::sqrt((x * x) + (y * y));
    }

private:
    mutable std::atomic<unsigned> callCount{ 0 };
    double x, y;
};
```
与std::mutex一样，std::atomic是只可移动类型，所以在Point中存在callCount就意味着Point也是只可移动的。（译者注：与 std::mutex 类似的，实际上 std::atomic 既不可移动，也不可复制。因而包含他们的类也同时是不可移动和不可复制的。）
因为**对`std::atomic变量`的操作通常比`互斥量`的获取和释放的`消耗更小`**，所以你可能会过度倾向与依赖std::atomic。
例如，在一个类中，缓存一个开销昂贵的int，你就会尝试使用一对std::atomic变量而不是互斥量。
```cpp
class Widget {
public:
    // …
    int magicValue() const
    {
        if (cacheValid)
            return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2;              //第一步
            cacheValid = true;                      //第二步
            return cachedValid;
        }
    }
    
private:
    mutable std::atomic<bool> cacheValid{ false };
    mutable std::atomic<int> cachedValue;
};
```
这是可行的，但难以避免有时出现`重复计算`的情况。考虑：
一个线程调用Widget::magicValue，将cacheValid视为false，执行这两个昂贵的计算，并将它们的和分配给cachedValue。
此时，第二个线程调用Widget::magicValue，也将cacheValid视为false，因此执行刚才完成的第一个线程相同的计算。（这里的“第二个线程”实际上可能是其他几个线程。）

这种行为与使用缓存的目的背道而驰。
将cachedValue和CacheValid的赋值顺序交换可以解决这个问题，但结果会更糟：
```cpp
class Widget {
public:
    // …
    int magicValue() const
    {
        if (cacheValid)
            return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cacheValid = true;                      //第一步
            return cachedValue = val1 + val2;       //第二步
        }
    }
    // …
}
```
假设cacheValid是false，那么：
一个线程调用Widget::magicValue，刚执行完将cacheValid设置true的语句。
在这时，第二个线程调用Widget::magicValue，检查cacheValid。**看到它是true，就返回cacheValue，即使第一个线程还没有给它赋值。** 因此返回的值是不正确的。
这里有一个坑。
**对于需要同步的是`单个的 变量 或者 内存位置`，使用`std::atomic`就足够了**。
不过，**一旦你需要对`两个以上的 变量或内存位置 作为一个单元`来操作的话，就应该使用`互斥量`**。对于Widget::magicValue是这样的。
```cpp
class Widget {
public:
    // …
    int magicValue() const
    {
        std::lock_guard<std::mutex> guard(m);   //锁定m
        
        if (cacheValid)
            return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2;
            cacheValid = true;
            return cachedValue;
        }
    }                                           //解锁m
    …

private:
    mutable std::mutex m;
    mutable int cachedValue;                    //不再用atomic
    mutable bool cacheValid{ false };           //不再用atomic
};
```
现在，这个条款是基于，`多个线程 可以同时在一个对象上 执行一个const成员函数 这个假设的`。
如果你不是在这种情况下编写一个const成员函数————你可以保证在一个对象上永远不会有多个线程执行该成员函数————该函数的线程安全是无关紧要的。
比如，**为独占单线程使用而设计的类的成员函数是否线程安全并不重要**。
在这种情况下，你可以避免因使用互斥量和std::atomics所消耗的资源，以及包含它们的类~~只能使用移动语义~~（译者注：既不能移动也不能复制）带来的副作用。
然而，这种线程无关的情况越来越少见，而且很可能会越来越少。
可以肯定的是，`const成员函数`应支持`并发执行`，这就是为什么你应该确保const成员函数是线程安全的。

请记住：
* 确保const成员函数线程安全，除非你确定它们永远不会在并发上下文（concurrent context）中使用。
* 使用std::atomic变量可能比互斥量提供更好的性能，但是它只适合操作单个变量或内存位置。



# 条款17：理解特殊成员函数的生成
在C++术语中，`特殊成员函数`是指`C++自己生成的函数`。
C++98有四个：`默认构造函数`，`析构函数`，`拷贝构造函数`，`拷贝赋值运算符`。
当然在这里有些细则要注意。
这些函数`仅在需要的时候才生成`，比如某个代码使用它们但是它们没有在类中明确声明。
`默认构造函数` 仅在 `类完全没有构造函数的时候`才生成。（防止编译器为某个类生成构造函数，但是你希望那个构造函数有参数）生成的特殊成员函数是`隐式public`且`inline`，它们是非虚的，除非相关函数是在派生类中的析构函数，派生类继承了有虚析构函数的基类。在这种情况下，编译器为派生类生成的析构函数是虚的。
但是你早就知道这些了。好吧好吧，都说古老的历史：美索不达米亚，商朝，FORTRAN，C++98。
但是时代改变了，C++生成特殊成员的规则也改变了。要留意这些新规则，知道什么时候编译器会悄悄地向你的类中添加成员函数，因为没有什么比这件事对C++高效编程更重要。
C++11特殊成员函数俱乐部迎来了两位新会员：`移动构造函数`和`移动赋值运算符`。
它们的签名是：
```cpp
class Widget {
public:
    // …
    Widget(Widget&& rhs);               //移动构造函数
    Widget& operator=(Widget&& rhs);    //移动赋值运算符
    // …
};
```
掌控它们生成和行为的规则类似于拷贝系列。
`移动操作`仅在需要的时候生成，如果生成了，就会对类的non-static数据成员执行逐成员的移动。
那意味着**`移动构造函数` 根据rhs参数里面对应的成员`移动构造`出新non-static部分，移动赋值运算符根据参数里面对应的non-static成员移动赋值**。
移动构造函数也移动构造基类部分（如果有的话），移动赋值运算符也是移动赋值基类部分。
现在，当我对一个数据成员或者基类使用移动构造或者移动赋值时，没有任何保证移动一定会真的发生。
逐成员移动，实际上，更像是`逐成员移动请求`，因为**对`不可移动类型`（即对移动操作没有特殊支持的类型，比如大部分C++98传统类）使用“移动”操作实际上执行的是拷贝操作**。
逐成员移动的核心是对对象使用`std::move`，然后`函数决议时`会选择执行移动还是拷贝操作。Item23包括了这个操作的细节。本条款中，简单记住如果支持移动就会逐成员移动类成员和基类成员，如果不支持移动就执行拷贝操作就好了。
像拷贝操作情况一样，如果你自己声明了移动操作，编译器就不会生成。然而它们生成的精确条件与拷贝操作的条件有点不同。
**两个拷贝操作是独立的**：
    声明一个不会限制编译器生成另一个。所以**如果你声明一个`拷贝构造函数`，但是没有声明拷贝赋值运算符，如果写的代码用到了拷贝赋值，编译器会帮助你`生成拷贝赋值运算符`**。
同样的，如果你声明拷贝赋值运算符但是没有拷贝构造函数，代码用到拷贝构造函数时编译器就会生成它。上述规则在C++98和C++11中都成立。
**两个移动操作不是相互独立的**。
    如果你声明了其中一个，编译器就不再生成另一个。
    如果你给类声明了，比如，一个移动构造函数，就表明对于移动操作应怎样实现，与编译器应生成的默认逐成员移动有些区别。
    如果逐成员移动构造有些问题，那么逐成员移动赋值同样也可能有问题。所以声明移动构造函数阻止移动赋值运算符的生成，声明移动赋值运算符同样阻止编译器生成移动构造函数。
再进一步，**如果一个类显式声明了拷贝操作，编译器就不会生成移动操作**。
    这种限制的解释是如果声明拷贝操作（构造或者赋值）就暗示着平常拷贝对象的方法（逐成员拷贝）不适用于该类，编译器会明白如果逐成员拷贝对拷贝操作来说不合适，逐成员移动也可能对移动操作来说不合适。
这是另一个方向。
**声明移动操作（构造或赋值）使得编译器禁用拷贝操作**。
    （编译器通过给拷贝操作加上`delete`来保证，参见Item11。）
    （译注：禁用的是自动生成的拷贝操作，对于用户声明的拷贝操作不受影响）毕竟，如果逐成员移动对该类来说不合适，也没有理由指望逐成员拷贝操作是合适的。
    听起来会破坏C++98的某些代码，因为C++11中拷贝操作可用的条件比C++98更受限，但事实并非如此。C++98的代码没有移动操作，因为C++98中没有移动对象这种概念。只有一种方法能让老代码使用用户声明的移动操作，那就是使用C++11标准然后添加这些操作，使用了移动语义的类必须接受C++11特殊成员函数生成规则的限制。
也许你早已听过`_Rule of Three_`规则。
    这个规则告诉**我们如果你声明了`拷贝构造函数，拷贝赋值运算符，或者析构函数`三者之一，你应该也声明其余两个**。
    它来源于长期的观察，即**用户接管`拷贝操作`的需求几乎都是因为该类会`做其他资源的管理`**，这也几乎意味着
    （1）无论哪种资源管理如果在一个拷贝操作内完成，也应该在另一个拷贝操作内完成
    （2）类的析构函数也需要参与资源的管理（通常是释放）。
    通常要管理的资源是内存，这也是为什么标准库里面那些管理内存的类（如会动态内存管理的STL容器）都声明了“the big three”：拷贝构造，拷贝赋值和析构。
`Rule of Three`带来的后果就是：只要出现`用户定义的析构函数`就意味着`简单的逐成员拷贝操作不适用于该类`。
    那意味着如果一个类声明了析构，拷贝操作可能不应该自动生成，因为它们做的事情可能是错误的。
    在C++98提出的时候，上述推理没有得倒足够的重视，所以C++98用户声明析构函数不会左右编译器生成拷贝操作的意愿。
    C++11中情况仍然如此，但仅仅是因为限制拷贝操作生成的条件会破坏老代码。
`Rule of Three`规则背后的解释依然有效，再加上对声明拷贝操作阻止移动操作隐式生成的观察，使得C++11不会为那些有用户定义的析构函数的类生成移动操作。
所以仅当下面条件成立时才会生成移动操作（当需要时）：
* 类中没有拷贝操作
* 类中没有移动操作
* 类中没有用户定义的析构

有时，类似的规则也会扩展至拷贝操作上面，C++11抛弃了已声明拷贝操作或析构函数的类的拷贝操作的自动生成。
这意味着：**如果你的某个声明了`析构`或者`拷贝`的类依赖`自动生成的拷贝操作`，你应该考虑升级这些类，消除依赖**。
假设编译器生成的函数行为是正确的（即逐成员拷贝类non-static数据是你期望的行为），你的工作很简单，C++11的`= default`就可以表达你想做的：
```cpp
class Widget {
    public:
    … 
    ~Widget();                              //用户声明的析构函数
    …                                       //默认拷贝构造函数的行为还可以
    Widget(const Widget&) = default;

    Widget&                                 //默认拷贝赋值运算符的行为还可以
        operator=(const Widget&) = default;
    … 
};
```
这种方法通常在`多态基类`中很有用，即通过操作的是哪个派生类对象来定义接口。
多态基类通常有一个`虚析构函数`，因为如果它们非虚，一些操作（比如通过一个基类指针或者引用对派生类对象使用delete或者typeid）会产生未定义或错误结果。
除非类继承了一个已经是`virtual`的`析构函数`，否则要想析构函数为虚函数的唯一方法就是加上virtual关键字。
通常，默认实现是对的，`= default`是一个不错的方式表达默认实现。
然而`用户声明的析构函数`会`抑制 编译器生成 移动操作`，所以如果该类需要具有移动性，就为移动操作加上`= default`。
声明移动会抑制拷贝生成，所以如果拷贝性也需要支持，再为拷贝操作加上`= default`：
```cpp
class Base {
public:
    virtual ~Base() = default;              //使析构函数virtual
    
    Base(Base&&) = default;                 //支持移动
    Base& operator=(Base&&) = default;
    
    Base(const Base&) = default;            //支持拷贝
    Base& operator=(const Base&) = default;
    … 
};
```

实际上，就算编译器乐于为你的类生成拷贝和移动操作，生成的函数也如你所愿，你也应该手动声明它们然后加上`= default`。
这看起来比较多余，但是它让你的意图更明确，也能帮助你避免一些微妙的bug。
比如，你有一个类来表示字符串表，即一种支持使用整数ID快速查找字符串值的数据结构：
```cpp
class StringTable {
public:
    StringTable() {}
    …                   //插入、删除、查找等函数，但是没有拷贝/移动/析构功能
private:
    std::map<int, std::string> values;
};
```
假设这个类没有声明拷贝操作，没有移动操作，也没有析构，如果它们被用到编译器会自动生成。
没错，很方便。
后来需要在对象构造和析构中打日志，增加这种功能很简单：
```cpp
class StringTable {
public:
    StringTable()
    { makeLogEntry("Creating StringTable object"); }    //增加的

    ~StringTable()                                      //也是增加的
    { makeLogEntry("Destroying StringTable object"); }
    …                                               //其他函数同之前一样
private:
    std::map<int, std::string> values;              //同之前一样
};
```
看起来合情合理，但是`声明析构`有潜在的副作用：它`阻止了 移动操作的生成`。
然而，拷贝操作的生成是不受影响的。因此代码能通过编译，运行，也能通过功能（译注：即打日志的功能）测试。
功能测试也包括移动功能，因为**即使该类`不支持移动操作`，对该类的移动请求也能通过编译和运行（`调用拷贝`，只要不`delete`移动操作的话）**。
这个请求正如之前提到的，会转而由拷贝操作完成。它意味着对StringTable对象的移动实际上是对对象的拷贝，即拷贝里面的`std::map<int, std::string>`对象。
拷贝`std::map<int, std::string>`对象很可能比移动慢几个数量级。简单的加个析构就引入了极大的性能问题！对拷贝和移动操作显式加个`= default`，问题将不再出现。
受够了我喋喋不休的讲述C++11拷贝移动规则了吧，你可能想知道什么时候我才会把注意力转入到剩下两个特殊成员函数，默认构造函数和析构函数。
现在就是时候了，但是只有一句话，因为它们几乎没有改变：它们在C++98中是什么样，在C++11中就是什么样。

`C++11`对于`特殊成员函数`处理的规则如下：
**默认构造函数**：
    和C++98规则相同。仅当类不存在用户声明的构造函数时才自动生成。
**析构函数**：
    基本上和C++98相同；稍微不同的是现在`析构`默认`noexcept`（参见Item14）。
    和C++98一样，仅当基类析构为虚函数时该类析构才为虚函数（也就是说，当基类析构不为virtual时，该类的析构就不是virtual的，没有virtual行为）。
**拷贝构造函数**：
    和C++98运行时行为一样：逐成员拷贝non-static数据。
    仅当类没有`用户定义的拷贝构造`时才生成。
    如果类声明了`移动操作`它就是`delete`的。
    当用户声明了拷贝赋值或者析构，该函数自动生成已被废弃。
**拷贝赋值运算符**：
    和C++98运行时行为一样：逐成员拷贝赋值non-static数据。
    仅当类没有用户定义的拷贝赋值时才生成。
    如果类声明了移动操作它就是delete的。
    当用户声明了拷贝构造或者析构，该函数自动生成已被废弃。
**移动构造函数和移动赋值运算符**：
    都对非static数据执行逐成员移动。
    仅当类没有用户定义的拷贝操作，移动操作或析构时才自动生成。
注意没有“成员函数模版 阻止 编译器生成特殊成员函数”的规则。
这意味着如果Widget是这样：
```cpp
class Widget {
    …
    template<typename T>                //从任何东西构造Widget
    Widget(const T& rhs);

    template<typename T>                //从任何东西赋值给Widget
    Widget& operator=(const T& rhs);
    …
};
```
编译器仍会生成移动和拷贝操作（假设正常生成它们的条件满足），即使可以模板实例化产出拷贝构造和拷贝赋值运算符的函数签名。
（当T为Widget时。）很可能你会觉得这是一个不值得承认的边缘情况，但是我提到它是有道理的，Item26将会详细讨论它可能带来的后果。

请记住：
* 特殊成员函数是编译器可能自动生成的函数：默认构造函数，析构函数，拷贝操作，移动操作。
* 移动操作仅当类没有显式声明移动操作，拷贝操作，析构函数时才自动生成。
* 拷贝构造函数仅当类没有显式声明拷贝构造函数时才自动生成，并且如果用户声明了移动操作，拷贝构造就是delete。
* 拷贝赋值运算符仅当类没有显式声明拷贝赋值运算符时才自动生成，并且如果用户声明了移动操作，拷贝赋值运算符就是delete。当用户声明了析构函数，拷贝操作的自动生成已被废弃。
* 成员函数模板不抑制特殊成员函数的生成。



# 第4章 智能指针
诗人和歌曲作家喜欢爱。有时候喜欢计数。很少情况下两者兼有。受伊丽莎白·巴雷特·勃朗宁（Elizabeth Barrett Browning）对爱和数的不同看法的启发（“我怎么爱你？让我数一数。”）和保罗·西蒙（Paul Simon）（“离开你的爱人必须有50种方法。”），我们可以试着枚举一些为什么原始指针很难被爱的原因：
* 它的声明不能指示所指到底是单个对象还是数组。
* 它的声明没有告诉你用完后是否应该销毁它，即`指针是否拥有所指之物`。
如果你决定你应该销毁指针所指对象，没人告诉你该用delete还是其他析构机制（比如将指针传给专门的销毁函数）。
如果你发现该用delete。原因1说了可能不知道该用单个对象形式（“`delete`”）还是数组形式（“`delete[]`”）。如果用错了结果是未定义的。
假设你确定了指针所指，知道销毁机制，也很难确定你在所有执行路径上都执行了恰为一次销毁操作（包括异常产生后的路径）。
少一条路径就会产生资源泄漏，销毁多次还会导致未定义行为。
一般来说没有办法告诉你指针是否变成了`悬空指针`（dangling pointers），即内存中不再存在指针所指之物。在对象销毁后指针仍指向它们就会产生悬空指针。
原始指针是强大的工具，当然，另一方面几十年的经验证明，只要注意力稍有疏忽，这个强大的工具就会攻击它的主人。

`智能指针（smart pointers）`是解决这些问题的一种办法。
智能指针包裹原始指针，它们的行为看起来像被包裹的原始指针，但避免了原始指针的很多陷阱。你应该更倾向于智能指针而不是原始指针。几乎原始指针能做的所有事情智能指针都能做，而且出错的机会更少。
在C++11中存在四种智能指针：`std::auto_ptr，std::unique_ptr，std::shared_ptr， std::weak_ptr`。
都是被设计用来帮助管理动态对象的生命周期，在适当的时间通过适当的方式来销毁对象，以避免出现资源泄露或者异常行为。
std::auto_ptr是来自C++98的已废弃遗留物，它是一次标准化的尝试，后来变成了C++11的std::unique_ptr。
要正确的模拟原生指针需要移动语义，但是C++98没有这个东西。
取而代之，std::auto_ptr拉拢拷贝操作来达到自己的移动意图。这导致了令人奇怪的代码（拷贝一个std::auto_ptr会将它本身设置为null！）和令人沮丧的使用限制（比如不能将std::auto_ptr放入容器）。
`std::unique_ptr`能做std::auto_ptr可以做的所有事情以及更多。它能高效完成任务，而且不会扭曲自己的原本含义而变成拷贝对象。在所有方面它都比std::auto_ptr好。现在std::auto_ptr唯一合法的使用场景就是代码使用C++98编译器编译。除非你有上述限制，否则你就该把std::auto_ptr替换为std::unique_ptr而且绝不回头。
各种智能指针的API有极大的不同。唯一功能性相似的可能就是`默认构造函数`。
因为有很多关于这些API的详细手册，所以我将只关注那些API概览没有提及的内容，比如值得注意的使用场景，运行时性能分析等，掌握这些信息可以更高效的使用智能指针。



# 条款18：对于独占资源使用std::unique_ptr
当你需要一个智能指针时，`std::unique_ptr通常是最合适的`。
可以合理假设，默认情况下，std::unique_ptr大小等同于原始指针，而且对于大多数操作（包括取消引用），他们执行的指令完全相同。
这意味着你甚至可以在内存和时间都比较紧张的情况下使用它。如果原始指针够小够快，那么std::unique_ptr一样可以。
`std::unique_ptr`体现了`专有所有权（exclusive ownership）`语义。
一个non-null std::unique_ptr始终拥有其指向的内容。
移动一个std::unique_ptr将`所有权`从`源指针`转移到`目的指针`。（`源指针被设为null`。）
拷贝一个std::unique_ptr是不允许的，因为如果你能拷贝一个std::unique_ptr，你会得到指向相同内容的两个std::unique_ptr，每个都认为自己拥有（并且应当最后销毁）资源，销毁时就会出现重复销毁。
因此，std::unique_ptr是一种`只可移动类型（move-only type）`。当析构时，一个non-null std::unique_ptr`销毁它指向的资源`。默认情况下，资源析构通过对std::unique_ptr里原始指针`调用delete`来实现。

`std::unique_ptr`的常见用法是：作为`继承层次结构中`对象的`工厂函数` `返回类型`。
假设我们有一个投资类型（比如股票、债券、房地产等）的继承结构，使用基类Investment。
```cpp
class Investment { … };

class Stock: public Investment { … };
class Bond: public Investment { … };
class RealEstate: public Investment { … };
```

这种`继承关系的工厂函数`在堆上分配一个对象然后返回指针，`调用方`在不需要的时候`有责任销毁对象`。
这使用场景完美匹配`std::unique_ptr`，因为`调用者`对工厂返回的资源负责（即对该资源的专有所有权），并且`std::unique_ptr`在自己被销毁时会自动销毁指向的内容。
Investment继承关系的工厂函数可以这样声明：
```cpp
template<typename... Ts>            //返回指向对象的std::unique_ptr，
std::unique_ptr<Investment>         //对象使用给定实参创建
makeInvestment(Ts&&... params);
```

调用者应该在单独的作用域中使用返回的`std::unique_ptr`智能指针：
```cpp
{
    // …
    auto pInvestment = makeInvestment( arguments );    //pInvestment是 std::unique_ptr<Investment> 类型
    // …
}                                       //销毁 *pInvestment
```
但是也可以在`所有权转移的场景中`使用它，比如将工厂返回的`std::unique_ptr`移入`容器`中，然后将 `容器元素` 移入一个`对象`的数据成员中，然后对象过后被销毁。
发生这种情况时，这个对象的`std::unique_ptr`数据成员也被销毁，并且智能指针数据成员的析构将导致从工厂返回的资源被销毁。
如果`所有权链` 由于 异常 或者 其他非典型控制流 出现中断（比如提前从函数`return`或者循环中的`break`），则拥有托管资源的`std::unique_ptr`将保证`指向内容`的`析构函数`被调用，销毁对应资源。（这个规则也有些例外。大多数情况发生于不正常的程序终止。如果一个`异常`传播到线程的基本函数（比如`程序初始线程的main函数`）外，或者违反`noexcept`说明（见Item14），`局部变量`可能`不会被销毁`；如果`std::abort`或者退出函数（如`std::_Exit`，`std::exit`，或`std::quick_exit`）被调用，局部变量一定没被销毁。）

默认情况下，`销毁`将通过`delete`进行，但是在`构造`过程中，`std::unique_ptr`对象可以被设置为使用（对资源的）`自定义删除器`：
    当资源需要销毁时 可调用的任意函数（或者函数对象，包括lambda表达式）。
如果通过`makeInvestment`创建的对象不应仅仅被delete，而应该先做一些别的操作：比如先写一条日志，makeInvestment可以以如下方式实现。（代码后有说明，别担心有些东西的动机不那么明显。）
```cpp
// 自定义删除器
auto delInvmt = [](Investment* pInvestment)
                {
                    makeLogEntry(pInvestment);  // log
                    delete pInvestment; 
                };

template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>     //更改后的返回类型，需要通过delInvmt得到使用auto来推导的lambda表达式的类型
makeInvestment(Ts&&... params)
{
    std::unique_ptr<Investment, decltype(delInvmt)> //应返回的指针
        pInv(nullptr, delInvmt);
    if (/*一个Stock对象应被创建*/)
    {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( /*一个Bond对象应被创建*/ )   
    {     
        pInv.reset(new Bond(std::forward<Ts>(params)...));   
    }   
    else if ( /*一个RealEstate对象应被创建*/ )   
    {     
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));   
    }   
    return pInv;
}
```
稍后，我将解释其工作原理，但首先请考虑如果你是调用者，情况如何。
假设你存储makeInvestment调用结果到auto变量中，那么你将在愉快中忽略在删除过程中需要特殊处理的事实。
当然，你确实幸福，因为使用了unique_ptr意味着你不需要关心什么时候资源应被释放，不需要考虑在资源释放时的路径，以及确保只释放一次，std::unique_ptr自动解决了这些问题。从使用者角度，makeInvestment接口很棒。
这个实现确实相当棒，如果你理解了：
`delInvmt`是从makeInvestment返回的对象的`自定义的删除器`。
所有的自定义的删除行为`接受要销毁对象的原始指针，然后执行所有必要行为实现销毁操作`。
在上面情况中，操作包括调用`makeLogEntry`然后应用`delete`。
使用`lambda`创建delInvmt是方便的，而且，正如稍后看到的，比编写常规的函数更有效。
当使用自定义删除器时，删除器类型必须作为`第二个类型实参`传给`std::unique_ptr`。
在上面情况中，就是delInvmt的类型，这就是为什么makeInvestment返回类型是`std::unique_ptr<Investment, decltype(delInvmt)>`。（对于decltype，更多信息查看Item3）
makeInvestment的基本策略是创建一个空的std::unique_ptr，然后指向一个合适类型的对象，然后返回。为了将自定义删除器delInvmt与pInv关联，我们把delInvmt作为pInv构造函数的第二个实参。
尝试将原始指针（比如new创建）赋值给std::unique_ptr通不过编译，因为是一种从原始指针到智能指针的隐式转换。（没法使用裸指针来赋值）
这种隐式转换会出问题，所以C++11的智能指针禁止这个行为。这就是通过`reset`来让pInv接管通过new创建的对象的所有权的原因。
使用new时，我们使用`std::forward`把传给makeInvestment的`实参`完美转发出去（查看Item25）。这使调用者提供的所有信息可用于正在创建的对象的构造函数。
自定义删除器的一个形参，类型是`Investment*`，不管在makeInvestment内部创建的对象的真实类型（如Stock，Bond，或RealEstate）是什么，它最终在lambda表达式中，作为`Investment*`对象被删除。
这意味着我们通过 `基类指针` `删除派生类实例`，为此，基类Investment必须有`虚析构函数`：
```cpp
class Investment {
public:
    // …
    virtual ~Investment();          //关键设计部分！
    // …
};
```

在`C++14`中，`函数返回类型推导`的存在（参阅Item3），意味着makeInvestment可以以更简单，更封装的方式实现：
```cpp
template<typename... Ts>
auto makeInvestment(Ts&&... params)                 //C++14
{
    auto delInvmt = [](Investment* pInvestment)
                    {
                        makeLogEntry(pInvestment);  //现在在makeInvestment函数内部
                        delete pInvestment; 
                    };

    std::unique_ptr<Investment, decltype(delInvmt)> //同之前一样
        pInv(nullptr, delInvmt);
    if ( … )                                        //同之前一样
    {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( … )                                   //同之前一样
    {     
        pInv.reset(new Bond(std::forward<Ts>(params)...));   
    }   
    else if ( … )                                   //同之前一样
    {     
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));   
    }   
    return pInv;                                    //同之前一样
}
```
我之前说过，当使用默认删除器时（如delete），你可以合理假设std::unique_ptr对象和原始指针大小相同。
当自定义删除器时，情况可能不再如此。`函数指针形式的删除器`，通常会使`std::unique_ptr`的`大小` `从一个字（word）增加到两个`。
对于函数对象形式的删除器来说，变化的大小取决于`函数对象中存储的状态多少`(使用捕获列表的话，还需要存储一些其他的对象)，无状态函数（stateless function）对象（比如不捕获变量的lambda表达式）对大小没有影响，这意味当自定义删除器可以实现为函数或者lambda时，尽量使用lambda：
```cpp
auto delInvmt1 = [](Investment* pInvestment)        //无状态lambda的自定义删除器
                 {
                     makeLogEntry(pInvestment);
                     delete pInvestment; 
                 };

template<typename... Ts>                            //返回类型大小是
std::unique_ptr<Investment, decltype(delInvmt1)>    //Investment*的大小
makeInvestment(Ts&&... args);

void delInvmt2(Investment* pInvestment)             //函数形式的
{                                                   //自定义删除器
    makeLogEntry(pInvestment);
    delete pInvestment;
}
template<typename... Ts>                            //返回类型大小是
std::unique_ptr<Investment, void (*)(Investment*)>  //Investment*的指针
makeInvestment(Ts&&... params);                     //加至少一个函数指针的大小
```
具有很多状态的自定义删除器会产生大尺寸std::unique_ptr对象。如果你发现自定义删除器使得你的std::unique_ptr变得过大，你需要审视修改你的设计。
工厂函数不是std::unique_ptr的唯一常见用法。
作为实现`Pimpl Idiom（译注：pointer to implementation，一种隐藏实际实现而减弱编译依赖性的设计思想`，《Effective C++》条款31对此有过叙述）的一种机制，它更为流行。
代码并不复杂，但是在某些情况下并不直观，所以这安排在Item22的专门主题中。
`std::unique_ptr`有两种形式，一种用于`单个对象（std::unique_ptr<T>）`，一种用于`数组（std::unique_ptr<T[]>）`。
结果就是，指向哪种形式没有歧义。std::unique_ptr的API设计会自动匹配你的用法，比如`operator[]`就是数组对象，解引用操作符（`operator*`和`operator->`）就是单个对象专有。
你应该对`数组的std::unique_ptr`的存在兴趣泛泛，因为std::array，std::vector，std::string这些更好用的数据容器应该取代原始数组。
`std::unique_ptr<T[]>`有用的唯一情况是：你使用`类似C的API` 返回 一个指向堆数组的 原始指针，而你想接管这个数组的所有权。

std::unique_ptr是C++11中表示专有所有权的方法，但是其最吸引人的功能之一是它可以轻松高效的`转换为std::shared_ptr`：
```cpp
std::shared_ptr<Investment> sp =            //将std::unique_ptr
    makeInvestment(arguments);              //转为std::shared_ptr
```
这就是std::unique_ptr非常适合用作工厂函数返回类型的原因的关键部分。
工厂函数无法知道调用者是否要对它们返回的对象使用专有所有权语义，或者共享所有权（即std::shared_ptr）是否更合适。
通过返回std::unique_ptr，工厂为调用者提供了最有效的智能指针，但它们并不妨碍调用者用其更灵活的兄弟替换它。（有关std::shared_ptr的信息，请转到Item19。)

请记住：
* std::unique_ptr是轻量级、快速的、只可移动（move-only）的管理专有所有权语义资源的智能指针
* 默认情况，资源销毁通过delete实现，但是支持自定义删除器。有状态的删除器和函数指针会增加std::unique_ptr对象的大小
* 将std::unique_ptr转化为std::shared_ptr非常简单



# 条款19：对于共享资源使用std::shared_ptr
使用带垃圾回收的语言的程序员指着C++程序员笑看他们如何防止资源泄露。
“真是原始啊！”他们嘲笑着说：“你们没有从1960年的Lisp那里得到启发吗，机器应该自己管理资源的生命周期而不应该依赖人类。”C++程序员翻白眼：“你们得到的所谓启示就是只有内存算资源，而且资源回收的时间点是不确定的？我们更喜欢通用，可预料的销毁，谢谢你。”但我们的虚张声势可能底气不足。因为垃圾回收真的很方便，而且手动管理生命周期真的就像是使用石头小刀和兽皮制作RAM电路。
为什么我们不能同时有两个完美的世界：一个自动工作的世界（像是垃圾回收），一个销毁可预测的世界（像是析构）？
C++11中的`std::shared_ptr`将两者组合了起来。
一个通过std::shared_ptr访问的对象其生命周期由指向它的有共享所有权（shared ownership）的指针们来管理。
没有特定的std::shared_ptr拥有该对象。相反，所有指向它的std::shared_ptr都能相互合作确保在它不再使用的那个点进行析构。当最后一个指向某对象的std::shared_ptr不再指向那（比如因为std::shared_ptr被销毁或者指向另一个不同的对象），std::shared_ptr会销毁它所指向的对象。就垃圾回收来说，客户端不需要关心指向对象的生命周期，而对象的析构是确定性的。
std::shared_ptr通过`引用计数（reference count）`来确保它是否是最后一个指向某种资源的指针，引用计数关联资源并跟踪有多少std::shared_ptr指向该资源。std::shared_ptr构造函数递增引用计数值（注意是通常————原因参见下面），析构函数递减值，拷贝赋值运算符做前面这两个工作。（如果sp1和sp2是std::shared_ptr并且指向不同对象，赋值“sp1 = sp2;”会使sp1指向sp2指向的对象。直接效果就是sp1引用计数减一，sp2引用计数加一。）**如果std::shared_ptr在计数值递减后发现引用计数值为零，没有其他std::shared_ptr指向该资源，它就会销毁资源。**

引用计数暗示着`性能问题`：
`std::shared_ptr`大小是原始指针的两倍，因为它内部包含`一个指向资源的原始指针`，还包含`一个指向资源的引用计数值的原始指针`。（这种实现法并不是标准要求的，但是我（指原书作者Scott Meyers）熟悉的所有标准库都这样实现。）
`引用计数`的内存必须`动态分配`。
概念上，引用计数与所指对象关联起来，但是实际上被指向的对象不知道这件事情（译注：不知道有一个关联到自己的计数值）。因此它们没有办法存放一个引用计数值。（一个好消息是任何对象————甚至是内置类型的————都可以由std::shared_ptr管理。）Item21会解释使用`std::make_shared`创建std::shared_ptr可以`避免引用计数的动态分配`，但是还存在一些std::make_shared不能使用的场景，这时候引用计数就会动态分配。
`递增递减引用计数`必须是`原子性`的，因为多个reader、writer可能在不同的线程。比如，指向某种资源的std::shared_ptr可能在一个线程执行析构（于是递减指向的对象的引用计数），在另一个不同的线程，std::shared_ptr指向相同的对象，但是执行的却是拷贝操作（因此递增了同一个引用计数）。原子操作通常比非原子操作要慢，所以即使引用计数通常只有一个word大小，你也应该假定`读写它们是存在开销的`。
我写道std::shared_ptr构造函数只是“通常”递增指向对象的引用计数会不会让你有点好奇？创建一个指向对象的std::shared_ptr就产生了又一个指向那个对象的std::shared_ptr，为什么我没说总是增加引用计数值？
原因是`移动构造函数`的存在。从另一个std::shared_ptr`移动构造`新std::shared_ptr会将原来的std::shared_ptr设置为null，那意味着老的std::shared_ptr不再指向资源，同时新的std::shared_ptr指向资源。这样的结果就是不需要修改引用计数值。因此移动std::shared_ptr会比拷贝它要快：拷贝要求递增引用计数值，移动不需要。移动赋值运算符同理，所以移动构造比拷贝构造快，移动赋值运算符也比拷贝赋值运算符快。
类似`std::unique_ptr`（参见Item18），std::shared_ptr使用delete作为资源的默认销毁机制，但是它也支持自定义的删除器。这种支持有别于std::unique_ptr。
！！！对于`std::unique_ptr`来说，`删除器类型`是`智能指针类型的一部分`。对于`std::shared_ptr`则不是：
为什么这样设计：
    因为`std::unique_ptr`是独占所有权的，因此其和其删除器的类型联系紧密，删除器的类型是`std::unique_ptr`的类型的一部分。`std::unique_ptr`和`std::shared_ptr`使用场景不同。
这种设计是怎么实现的：
    通过模板类的模板参数来实现：
```cpp
  /// 20.7.1.2 unique_ptr for single objects.
  template <typename _Tp, typename _Dp = default_delete<_Tp> >
    class unique_ptr    // 对unique_ptr，deletor就是模板类型参数的一部分
    {
    // ...
    };

  template<typename _Tp>
    class shared_ptr : public __shared_ptr<_Tp>
    {
        // ...
    };
```
而每一个`shared_ptr`在构造的时候，可以独立构造自己的deletor

```cpp
auto loggingDel = [](Widget *pw)        // 自定义删除器
                  {                     //（和条款18一样）
                      makeLogEntry(pw);
                      delete pw;
                  };

std::unique_ptr<Widget, decltype(loggingDel)> upw(new Widget, loggingDel);      //删除器类型是指针类型的一部分
std::shared_ptr<Widget> spw(new Widget, loggingDel);        //删除器类型不是指针类型的一部分
```
`std::shared_ptr`的设计更为灵活。
考虑有两个`std::shared_ptr<Widget>`，每个自带不同的删除器（比如通过lambda表达式自定义删除器）：
```cpp
auto customDeleter1 = [](Widget *pw) { … };     //自定义删除器，
auto customDeleter2 = [](Widget *pw) { … };     //每种类型不同
std::shared_ptr<Widget> pw1(new Widget, customDeleter1);
std::shared_ptr<Widget> pw2(new Widget, customDeleter2);
```
因为`pw1`和`pw2`有相同的类型，所以它们都可以放到存放那个类型的对象的容器中：
```cpp
std::vector<std::shared_ptr<Widget>> vpw{ pw1, pw2 };
```
它们也能相互赋值，也可以传入一个形参为`std::shared_ptr<Widget>`的函数。
但是自定义删除器类型不同的`std::unique_ptr`就不行，因为std::unique_ptr把删除器视作类型的一部分。

另一个不同于`std::unique_ptr`的地方是，指定自定义删除器不会改变std::shared_ptr对象的大小。
不管删除器是什么，一个std::shared_ptr对象都是两个指针大小。
这是个好消息，但是它应该让你隐隐约约不安。自定义删除器可以是函数对象，函数对象可以包含任意多的数据。它意味着函数对象是任意大的。
std::shared_ptr怎么能引用一个任意大的删除器而不使用更多的内存？
它不能。它必须使用更多的内存。然而，`那部分内存不是std::shared_ptr对象的一部分`。
那部分在堆上面，或者std::shared_ptr创建者利用std::shared_ptr对自定义分配器的支持能力，那部分内存随便在哪都行。我前面提到了std::shared_ptr对象包含了所指对象的引用计数的指针。没错，但是有点误导人。因为引用计数是另一个更大的数据结构的一部分，那个数据结构通常叫做`控制块（control block）`。
每个`std::shared_ptr`管理的对象都有个相应的控制块。
`控制块`除了包含`引用计数值`外还有一个`自定义删除器的拷贝`，当然前提是存在自定义删除器。
如果用户还指定了`自定义分配器`，控制块也会包含一个`分配器的拷贝`。控制块可能还包含一些额外的数据，正如Item21提到的，一个`次级引用计数weak count`，但是目前我们先忽略它。

当指向对象的std::shared_ptr一创建，对象的控制块就建立了。
至少我们期望是如此。通常，对于一个创建指向对象的`std::shared_ptr`的函数来说不可能知道是否有其他std::shared_ptr早已指向那个对象，所以控制块的创建会遵循下面几条规则：
* `std::make_shared`（参见Item21）总是创建一个`控制块`。
**它创建一个要指向的新对象，所以可以肯定`std::make_shared`调用时对象不存在其他控制块。**
* 当从`独占指针`（即std::unique_ptr或者std::auto_ptr）上构造出std::shared_ptr时会`创建控制块`。**独占指针没有使用控制块**，所以指针指向的对象没有关联控制块。（**作为构造的一部分，`std::shared_ptr`侵占独占指针所指向的对象的独占权，所以独占指针被设置为null**）
* 当从`原始指针`上构造出`std::shared_ptr`时会创建控制块。
    如果你想从一个早已存在控制块的对象上创建`std::shared_ptr`，你将假定传递一个`std::shared_ptr`或者`std::weak_ptr`（参见Item20）作为构造函数实参，而`不是原始指针`。
    ！！！对于一个对象而言，只能绑定一个引用控制块，因为引用计数为0时就销毁了
    用`std::shared_ptr`或者`std::weak_ptr`作为构造函数实参创建std::shared_ptr不会创建新控制块，因为它可以依赖传递来的智能指针指向控制块。
这些规则造成的后果就是 从原始指针上构造 超过一个std::shared_ptr 就会让你走上未定义行为的快车道，因为指向的对象有多个控制块关联。
多个控制块意味着多个引用计数值，多个引用计数值意味着对象将会被销毁多次（每个引用计数一次）。
那意味着像下面的代码是有问题的，很有问题，问题很大：
```cpp
auto pw = new Widget;                           //pw是原始指针
// …
std::shared_ptr<Widget> spw1(pw, loggingDel);   //为*pw创建控制块
// …
std::shared_ptr<Widget> spw2(pw, loggingDel);   //为*pw创建第二个控制块
```
`创建原始指针pw`指向动态分配的对象很糟糕，因为它完全背离了这章的建议：倾向于使用智能指针而不是原始指针。（如果你忘记了该建议的动机，请翻到本章开头）。撇开那个不说，创建pw那一行代码虽然让人厌恶，但是至少不会造成未定义程序行为。

现在，传给spw1的构造函数一个原始指针，它会为指向的对象创建一个`控制块`（因此有个引用计数值）。这种情况下，指向的对象是`*pw`（即pw指向的对象）。就其本身而言没什么问题，但是将同样的原始指针传递给spw2的构造函数会再次为`*pw`创建一个`控制块`（所以也有个引用计数值）。
因此`*pw`有两个引用计数值，每一个最后都会变成零，然后最终导致*pw销毁两次。第二个销毁会产生未定义行为。

std::shared_ptr给我们上了两堂课。
* 第一，避免传给`std::shared_ptr`构造函数`原始指针`。
通常替代方案是使用`std::make_shared`（参见Item21），不过上面例子中，我们使用了自定义删除器，用std::make_shared就没办法做到。
* 第二，如果你必须传给`std::shared_ptr`构造函数原始指针，直接传new出来的结果，不要传指针变量。
如果上面代码第一部分这样重写：
```cpp
std::shared_ptr<Widget> spw1(new Widget,    //直接使用new的结果 这样就不存在指针的具名对象，安全多了
                             loggingDel);
```
会少了很多从原始指针上构造第二个std::shared_ptr的诱惑。
相应的，创建spw2也会很自然的用spw1作为初始化参数（即用`std::shared_ptr`拷贝构造函数），那就没什么问题了：
```cpp
std::shared_ptr<Widget> spw2(spw1);         //spw2使用spw1一样的控制块
```
一个尤其令人意外的地方是：使用`this指针`作为`std::shared_ptr`构造函数实参的时候可能导致`创建多个控制块`。
假设我们的程序使用`std::shared_ptr`管理Widget对象，我们有一个数据结构用于跟踪已经处理过的Widget对象：
```cpp
std::vector<std::shared_ptr<Widget>> processedWidgets;
```
继续，假设Widget有一个用于处理的成员函数：
```cpp
class Widget {
public:
    // …
    void process();
    // …
};
```
对于Widget::process看起来合理的代码如下：
```cpp
void Widget::process()
{
    …                                       //处理Widget
    processedWidgets.emplace_back(this);    //然后将它加到已处理过的Widget的列表中，这是错的！
}
```
注释已经说了这是错的——或者至少大部分是错的。
（错误的部分是`传递this`，而不是使用了emplace_back。如果你不熟悉emplace_back，参见Item42）。上面的代码可以通过编译，但是向std::shared_ptr的容器传递一个`原始指针（this）`，std::shared_ptr会由此为指向的Widget（*this）创建一个`控制块`。
！！！那看起来没什么问题，直到你意识到如果成员函数外面早已存在指向那个Widget对象的指针，它是未定义行为的Game, Set, and Match（译注：一部关于网球的电影，但是译者没看过。句子本意“压倒性胜利；比赛结束”）。

`std::shared_ptr` API已有处理这种情况的设施。
它的名字可能是C++标准库中最奇怪的一个：`std::enable_shared_from_this`，是一个模板类。
如果你想创建一个用`std::shared_ptr`管理的类，`这个类能够用this指针安全地创建一个std::shared_ptr`，std::enable_shared_from_this就可作为基类的模板类。
在我们的例子中，Widget将会继承自std::enable_shared_from_this：
```cpp
class Widget: public std::enable_shared_from_this<Widget> {
public:
    // …
    void process();
    // …
};
```
正如我所说，`std::enable_shared_from_this`是一个基类模板。
它的模板参数总是某个继承自它的类，所以Widget继承自`std::enable_shared_from_this<Widget>`。
如果某类型继承自一个由该类型（译注：作为模板类型参数）进行模板化得到的基类这个东西让你心脏有点遭不住，别去想它就好了。代码完全合法，而且它背后的设计模式也是没问题的，并且这种设计模式还有个标准名字，尽管该名字和std::enable_shared_from_this一样怪异。
这个标准名字就是`奇异递归模板模式（The Curiously Recurring Template Pattern（CRTP））`。如果你想学更多关于它的内容，请搜索引擎一展身手，现在我们要回到std::enable_shared_from_this上。
`std::enable_shared_from_this`定义了一个成员函数`shared_from_this`，成员函数会创建指向当前对象的std::shared_ptr却不创建多余控制块。
无论在哪当你想在成员函数中使用std::shared_ptr指向this所指对象时都请使用它。
这里有个Widget::process的安全实现：
```cpp
void Widget::process()
{
    //和之前一样，处理Widget
    // …
    //把指向当前对象的std::shared_ptr加入processedWidgets
    processedWidgets.emplace_back(shared_from_this());
}
```
从内部来说，`shared_from_this` 先`查找当前对象控制块`，然后创建一个新的`std::shared_ptr`关联这个控制块。
设计的依据是当前对象已经存在一个关联的控制块。
要想符合设计依据的情况，必须已经存在一个指向当前对象的`std::shared_ptr`（比如调用shared_from_this的成员函数外面已经存在一个std::shared_ptr）。
！！！如果没有std::shared_ptr指向当前对象（即当前对象没有关联控制块），行为是未定义的，shared_from_this通常抛出一个异常。（也就是说，`shared_from_this()`不负责创建控制块）

要想防止客户端在存在一个指向对象的std::shared_ptr前先调用含有shared_from_this的成员函数，继承自std::enable_shared_from_this的类通常将它们的构造函数声明为private，并且让客户端通过返回std::shared_ptr的工厂函数创建对象。
以Widget为例，代码可以是这样：
```cpp
class Widget: public std::enable_shared_from_this<Widget> {
public:
    //完美转发参数给private构造函数的工厂函数
    template<typename... Ts>
    static std::shared_ptr<Widget> create(Ts&&... params);
    // …
    void process();     //和前面一样
    // …
private:
    Widget();           //构造函数
};
```
现在，你可能隐约记得我们讨论控制块的动机是想了解有关std::shared_ptr的成本。既然我们已经知道了怎么避免创建过多控制块，就让我们回到原来的主题。
控制块通常只占几个word大小，自定义删除器和分配器可能会让它变大一点。
通常控制块的实现比你想的更复杂一些。它使用继承，甚至里面还有一个`虚函数`（！！！用来确保指向的对象被正确销毁）。
这意味着使用std::shared_ptr还会招致控制块使用虚函数带来的成本。

了解了动态分配控制块，任意大小的删除器和分配器，虚函数机制，原子性的引用计数修改，你对于std::shared_ptr的热情可能有点消退。
可以理解，对每个资源管理问题来说都没有最佳的解决方案。但就它提供的功能来说，std::shared_ptr的开销是非常合理的。在通常情况下，使用默认删除器和默认分配器，使用std::make_shared创建std::shared_ptr，产生的控制块只需三个word大小。
它的分配基本上是无开销的。（`开销 被并入了 指向的对象的分配成本里`。细节参见Item21）。对std::shared_ptr解引用的开销不会比原始指针高。执行需要原子引用计数修改的操作需要承担一两个原子操作开销，这些操作通常都会一一映射到机器指令上，所以即使对比非原子指令来说，原子指令开销较大，但是它们仍然只是`单个指令上的`。对于每个被std::shared_ptr指向的对象来说，控制块中的虚函数机制产生的开销通常只需要承受一次，即对象销毁的时候。

作为这些轻微开销的交换，你得到了动态分配的资源的生命周期自动管理的好处。大多数时候，比起手动管理，使用std::shared_ptr管理共享性资源都是非常合适的。如果你还在犹豫是否能承受std::shared_ptr带来的开销，那就再想想你是否需要共享所有权。如果独占资源可行或者可能可行，用std::unique_ptr是一个更好的选择。它的性能表现更接近于原始指针，并且从std::unique_ptr升级到std::shared_ptr也很容易，因为std::shared_ptr可以从std::unique_ptr上创建。

反之不行。当你的资源由std::shared_ptr管理，现在又想修改资源生命周期管理方式是没有办法的。
即使引用计数为一，你也不能重新修改资源所有权，改用std::unique_ptr管理它。`资源和指向它的std::shared_ptr的签订的所有权协议是“除非死亡否则永不分开”。不能分离，不能废除，没有特许。`
`std::shared_ptr不能处理的另一个东西是数组`。和std::unique_ptr不同的是，std::shared_ptr的API设计之初就是针对单个对象的，没有办法`std::shared_ptr<T[]>`。（译者注: 自 C++17 起 std::shared_ptr 可以用于管理动态分配的数组，使用 std::shared_ptr<T[]>）一次又一次，“聪明”的程序员踌躇于是否该使用`std::shared_ptr<T>`指向数组，然后传入自定义删除器来删除数组（即delete []）。
这可以通过编译，但是是一个糟糕的主意。一方面，std::shared_ptr没有提供`operator[]`，所以数组索引操作需要借助怪异的指针算术。另一方面，`std::shared_ptr`支持转换为指向基类的指针，这对于单个对象来说有效，但是当用于数组类型时相当于在类型系统上开洞。（出于这个原因，`std::unique_ptr<T[]>` API禁止这种转换。）更重要的是，C++11已经提供了很多内置数组的候选方案（比如std::array，std::vector，std::string）。声明一个指向傻瓜数组的智能指针（译注：也是”聪明的指针“之意）几乎总是表示着糟糕的设计。

请记住：
* std::shared_ptr为有共享所有权的任意资源提供一种自动垃圾回收的便捷方式。
* (较之于std::unique_ptr，std::shared_ptr对象通常大两倍，控制块会产生开销，需要原子性的引用计数修改操作。
* 默认资源销毁是通过delete，但是也支持自定义删除器。删除器的类型是什么对于std::shared_ptr的类型没有影响。
* 避免从原始指针变量上创建std::shared_ptr。



# 条款20：当std::shared_ptr可能悬空时使用std::weak_ptr
自相矛盾的是，如果有一个像`std::shared_ptr`（见Item19）的、但是不参与`资源所有权共享的`指针是很方便的。
换句话说，是一个类似`std::shared_ptr`但`不影响对象引用计数的`指针。
这种类型的智能指针必须要解决一个std::s`hared_ptr不存在的问题：可能指向已经销毁的对象。
**一个真正的智能指针应该跟踪所指对象，`在悬空时知晓`，悬空（dangle）就是指针指向的对象不再存在**。这就是对`std::weak_ptr`最精确的描述。
你可能想知道什么时候该用`std::weak_ptr`。
你可能想知道关于std::weak_ptr API的更多。它什么都好除了不太智能：std::weak_ptr不能解引用，也不能测试是否为空值。因为std::weak_ptr不是一个独立的智能指针。它是`std::shared_ptr的增强`。
这种关系在它创建之时就建立了：std::weak_ptr通常从std::shared_ptr上创建。
当从std::shared_ptr上创建std::weak_ptr时两者指向相同的对象，但是std::weak_ptr不会影响所指对象的引用计数：
```cpp
auto spw =                      //spw创建之后，指向的Widget的引用计数（ref count，RC）为1。
    std::make_shared<Widget>();
                                //std::make_shared的信息参见条款21
// …
std::weak_ptr<Widget> wpw(spw); //wpw指向与spw所指相同的Widget。RC仍为1
// …
spw = nullptr;                  //RC变为0，Widget被销毁。   ！！！触发了引用计数-1的操作
                                //wpw现在悬空
```
`悬空的std::weak_ptr`被称作已经`expired（过期）`。你可以用它直接做测试：
```cpp
if (wpw.expired()) …            //如果wpw没有指向对象…
```
但是通常你期望的是：检查`std::weak_ptr`是否已经过期，如果没有过期，则`访问其指向的对象`。
这做起来可不是想着那么简单。因为缺少解引用操作，没有办法写这样的代码。即使有，将`检查`和`解引用`分开会引入`竞态条件`：
    **在调用`expired`和`解引用`操作之间，另一个线程可能对指向这对象的`std::shared_ptr`重新赋值或者析构，并由此造成对象已析构。这种情况下，你的解引用将会产生未定义行为**。
你需要的是一个`原子操作`检查`std::weak_ptr`是否已经过期，如果没有过期就访问所指对象。
这可以通过**从`std::weak_ptr`创建`std::shared_ptr`来实现**，具体有两种形式可以从`std::weak_ptr上`创建`std::shared_ptr`，具体用哪种取决于std::weak_ptr过期时你希望std::shared_ptr表现出什么行为。
* 一种形式是`std::weak_ptr::lock`，它返回一个`std::shared_ptr`
    如果std::weak_ptr过期，这个std::shared_ptr为空：
```cpp
std::shared_ptr<Widget> spw1 = wpw.lock();  //如果wpw过期，spw1就为空
 											
auto spw2 = wpw.lock();                     //同上，但是使用auto
```

* 另一种形式是以`std::weak_ptr`为`实参`构造`std::shared_ptr`。这种情况中，如果std::weak_ptr过期，会抛出一个异常：
```cpp
std::shared_ptr<Widget> spw3(wpw);          //如果wpw过期，抛出std::bad_weak_ptr异常
```
但是你可能还想知道为什么`std::weak_ptr`就有用了。
考虑一个`工厂函数`：它基于一个`唯一ID` 从 只读对象上 产出智能指针。
根据Item18的描述，工厂函数会返回一个该对象类型的`std::unique_ptr`：
```cpp
std::unique_ptr<const Widget> loadWidget(WidgetID id);
```
如果调用`loadWidget`是一个昂贵的操作（比如它`操作文件`或者`数据库I/O`）并且重复使用ID很常见，一个合理的优化是：
    再写一个函数    `除了完成loadWidget做的事情之外 再缓存它的结果`。
    当每个请求获取的Widget阻塞了缓存也会导致本身性能问题，所以另一个合理的优化可以是    当Widget不再使用的时候销毁它的缓存。

对于`可缓存的工厂函数`（！！！是一种经典设计），返回`std::unique_ptr`不是好的选择。
调用者应该接收缓存对象的智能指针，调用者也应该确定这些对象的生命周期，但是缓存本身也需要一个指针指向它所缓存的对象。
缓存对象的指针`需要知道它是否已经悬空`，因为当工厂客户端使用完工厂产生的对象后，对象将被销毁，关联的缓存条目会悬空。
所以缓存应该使用`std::weak_ptr`，这可以知道是否已经悬空。
这意味着工厂函数返回值类型应该是`std::shared_ptr`，因为只有当对象的生命周期由`std::shared_ptr`管理时，std::weak_ptr才能检测到悬空。
下面是一个临时凑合的loadWidget的缓存版本的实现：
```cpp
std::shared_ptr<const Widget> fastLoadWidget(WidgetID id)
{
    static std::unordered_map<WidgetID, std::weak_ptr<const Widget>> cache;
                                        //  译者注：这里std::weak_ptr<const Widget>是高亮
    auto objPtr = cache[id].lock();     //  objPtr是去缓存对象的std::shared_ptr（或当对象不在缓存中时为null）

    if (!objPtr) {                      //如果不在缓存中    加载它  缓存它
        objPtr = loadWidget(id);
        cache[id] = objPtr;
    }
    return objPtr;
}
```
这个实现使用了C++11的hash表容器`std::unordered_map`，但是需要的 WidgetID 哈希 和 相等性比较函数 在这里没有展示（PS: 因为 std::unordered_map 需要这些函数来对键进行哈希和比较操作，以便正确地管理缓存中的对象）。
`fastLoadWidget`的实现忽略了以下事实：**缓存可能会累积`过期的std::weak_ptr`，这些指针对应了不再使用的Widget（也已经被销毁了）**。
其实可以改进实现方式，但是花时间在这个问题上不会让我们对std::weak_ptr有更深入的理解，

让我们考虑第二个用例：`观察者设计模式（Observer design pattern）`。
    此模式的主要组件是`subjects（状态可能会更改的对象）`和`observers（状态发生更改时要通知的对象）`。
    在大多数实现中，每个`subject`都包含一个数据成员，该成员持有`指向其observers的 指针`。
    这使subjects很容易发布状态更改通知。
    `subjects`对控制`observers`的生命周期（即它们什么时候被销毁）没有兴趣，但是subjects对确保另一件事具有极大的兴趣，那事就是：一个observer被销毁时，不再尝试访问它。
    **一个合理的设计是：每个`subject`持有一个`std::weak_ptrs容器`指向`observers`，因此可以在`使用前检查 是否已经悬空`。**

作为最后一个使用`std::weak_ptr`的例子，考虑一个持有三个对象A、B、C的数据结构，A和C共享B的所有权，因此持有`std::shared_ptr`：

有三种选择：
* `原始指针`。
    使用这种方法，如果A被销毁，但是C继续指向B，B就会有一个指向A的悬空指针。
    而且B不知道指针已经悬空，所以B可能会继续访问，就会导致未定义行为。

* `std::shared_ptr`。
    这种设计，A和B都互相持有对方的std::shared_ptr，导致的std::shared_ptr`环状结构`（A指向B，B指向A）阻止A和B的销毁。
    甚至A和B无法从其他数据结构访问了（比如，C不再指向B），每个的引用计数都还是1。
    如果发生了这种情况，A和B都被泄漏：程序无法访问它们，但是资源并没有被回收。

* `std::weak_ptr`。
这避免了上述两个问题。如果A被销毁，B指向它的指针悬空，但是B可以检测到这件事。
尤其是，尽管A和B互相指向对方，B的指针不会影响A的引用计数，因此在没有std::shared_ptr指向A时不会导致A无法被销毁。
使用`std::weak_ptr`显然是这些选择中最好的。
但是，需要注意使用std::weak_ptr打破std::shared_ptr循环并不常见。在`严格分层的`数据结构比如树中，子节点只被父节点持有。
    当父节点被销毁时，子节点就被销毁。从父到子的链接关系可以使用`std::unique_ptr`很好的表征。从子到父的反向连接可以使用`原始指针安全实现`，因为子节点的生命周期肯定短于父节点。
    因此没有子节点解引用一个悬垂的父节点指针这样的风险。

当然，不是所有的使用指针的数据结构都是严格分层的，所以当发生这种情况时，比如上面所述`缓存`和`观察者列表`的实现之类的，知道std::weak_ptr随时待命也是不错的。
从效率角度来看，std::weak_ptr与std::shared_ptr基本相同。
两者的大小是相同的，使用`相同的控制块`（参见Item19），`构造、析构、赋值操作涉及引用计数的原子操作`。
这可能让你感到惊讶，因为本条款开篇就提到std::weak_ptr不影响引用计数。
我写的是std::weak_ptr不参与对象的共享所有权，因此不影响指向对象的引用计数。实际上在控制块中还是有第二个引用计数，std::weak_ptr操作的是`第二个引用计数`。想了解细节的话，继续看Item21吧。

请记住：
* 用std::weak_ptr替代可能会悬空的std::shared_ptr。
* std::weak_ptr的潜在使用场景包括：缓存、观察者列表、打破std::shared_ptr环状结构。



# 条款21：优先考虑使用`std::make_unique`和`std::make_shared`，而非`直接使用new`
让我们先对std::make_unique和std::make_shared做个铺垫。
std::make_shared是C++11标准的一部分，但很可惜的是，`std::make_unique`不是。它从`C++14`开始加入标准库。
如果你在使用C++11，不用担心，一个基础版本的std::make_unique是很容易自己写出的，如下：
```cpp
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
    return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```
正如你看到的，make_unique只是将它的参数`完美转发`到所要创建的对象的`构造函数`，从new产生的`原始指针`里面构造出`std::unique_ptr`，并返回这个std::unique_ptr
。这种形式的函数`不支持数组和自定义析构`（见Item18），但它给出了一个示范：只需一点努力就能写出你想要的make_unique函数。
    （要想实现一个特性完备的make_unique，就去找提供这个的标准化文件吧，然后拷贝那个实现。你想要的这个文件是N3656，是Stephan T. Lavavej写于2013-04-18的文档。）
    需要记住的是，不要把它放到std命名空间中，因为你可能并不希望看到升级C++14标准库的时候你放进std命名空间的内容和编译器供应商提供的std命名空间的内容发生冲突。

`std::make_unique`和`std::make_shared`是三个`make函数`中的两个：**接收`任意的多参数集合`，完美转发到构造函数去动态分配一个对象，然后返回这个指向这个对象的指针**。
第三个make函数是`std::allocate_shared`。它行为和std::make_shared一样，只不过第一个参数是用来`动态分配内存的allocator对象`。

即使通过用和不用make函数来创建智能指针的一个小小比较，也揭示了为何使用make函数更好的第一个原因。例如：
```cpp
auto upw1(std::make_unique<Widget>());      //使用make函数
std::unique_ptr<Widget> upw2(new Widget);   //不使用make函数
auto spw1(std::make_shared<Widget>());      //使用make函数
std::shared_ptr<Widget> spw2(new Widget);   //不使用make函数
```
我高亮了关键区别：使用new的版本重复了类型，但是make函数的版本没有。
（译者注：这里高亮的是Widget，用new的声明语句需要`写2遍Widget`，make函数只需要写一次。）重复写类型和软件工程里面一个关键原则相冲突：`应该避免重复代码`。
源代码中的重复增加了编译的时间，会导致`目标代码冗余`（很现实），并且通常会让代码库使用更加困难。
它经常演变成不一致的代码，而代码库中的不一致常常导致bug。
此外，打两次字比一次更费力，而且没人不喜欢少打字吧？

第二个使用make函数的原因和`异常安全`有关。
假设我们有个函数按照某种优先级处理Widget：
```cpp
void processWidget(std::shared_ptr<Widget> spw, int priority);
```
值传递`std::shared_ptr`可能看起来很可疑，但是Item41解释了，如果`processWidget`总是复制`std::shared_ptr`（例如，通过将其存储在`已处理的Widget的一个数据结构中`），那么这可能是一个合理的设计选择。

现在假设我们有一个函数来计算相关的优先级，
```cpp
int computePriority();
```
并且我们在调用`processWidget`时使用了`new`而不是`std::make_shared`：
```cpp
processWidget(std::shared_ptr<Widget>(new Widget),  //潜在的资源泄漏！
              computePriority());
```
如注释所说，这段代码可能在`new`一个`Widget`时发生泄漏。
    为何？调用的代码和被调用的函数都用std::shared_ptrs，且std::shared_ptrs就是设计出来防止泄漏的。
    它们会在最后一个`std::shared_ptr`销毁时自动释放所指向的内存。如果每个人在每个地方都用std::shared_ptrs，这段代码怎么会泄漏呢？

答案和`编译器将源码转换为目标代码`有关（！！！可能因为编译优化而打乱了顺序）。**在`运行时`，一个函数的`实参`必须先被`计算`，这个函数再被调用**，所以在调用processWidget之前，必须执行以下操作，processWidget才开始执行：
表达式“`new Widget`”必须计算，例如，一个Widget对象必须在`堆`上被创建
负责管理new出来指针的`std::shared_ptr<Widget>`构造函数必须被执行
`computePriority`必须运行
**`编译器`不需要按照`执行顺序`生成代码。**
“new Widget”必须在`std::shared_ptr的构造函数`被调用前执行，因为`new`出来的结果作为构造函数的实参，但`computePriority`可能在这之前，之后，或者之间执行。
也就是说，编译器可能按照这个执行顺序生成代码：
* 1. 执行“`new Widget`”
* 2. 执行`computePriority`
* 3. 运行`std::shared_ptr构造函数`
如果按照这样生成代码，并且在运行时`computePriority`产生了`异常`，那么第一步动态分配的Widget就会泄漏。
因为它永远都不会被第三步的`std::shared_ptr`所管理了。
使用`std::make_shared`可以防止这种问题。
调用代码看起来像是这样：
```cpp
processWidget(std::make_shared<Widget>(),   //没有潜在的资源泄漏
              computePriority());
```
在运行时，`std::make_shared`和`computePriority`其中一个会先被调用。
如果是`std::make_shared`先被调用，在`computePriority`调用前，动态分配Widget的`原始指针`会安全的保存在作为返回值的`std::shared_ptr`中。
如果`computePriority`产生一个异常，那么`std::shared_ptr``析构函数`将确保管理的Widget被销毁。
（
PS: 控制权从一处转移到另处，这有两个重要的含义:
* 沿着调用链的函数可能会提早退出。
* 一旦程序开始执行异常处理代码，则`沿着调用链创建的对象`将被销。
    因为跟在throw后面的语句将不再被执行，所以throw语句的用法有点类似于return语句：它通常作为条件语句的一部分或者作为某个函数的最后(或者唯一)一条语句。
）

如果首先调用`computePriority`并产生一个异常，那么`std::make_shared`将不会被调用，因此也就不需要担心动态分配`Widget`（会泄漏）。
如果我们将std::shared_ptr，std::make_shared替换成`std::unique_ptr`，std::make_unique，同样的道理也适用。
因此，在编写异常安全代码时，使用std::make_unique而不是new与使用std::make_shared（而不是new）同样重要。
`std::make_shared`的一个特性（与直接使用new相比）是效率提升。
使用std::make_shared允许编译器生成更小，更快的代码，并使用更简洁的数据结构。
考虑以下对new的直接使用：
```cpp
std::shared_ptr<Widget> spw(new Widget);
```
显然，这段代码需要进行内存分配，但它实际上执行了两次。
Item19解释了每个std::shared_ptr指向一个`控制块`，其中包含被指向对象的引用计数，还有其他东西。
这个控制块的内存在std::shared_ptr构造函数中分配。因此，直接使用new需要为`Widget`进行一次内存分配，为`控制块`再进行一次内存分配。
如果使用`std::make_shared`代替：
```cpp
auto spw = std::make_shared<Widget>();
```
一次分配足矣。
这是因为`std::make_shared`分配一块内存，同时容纳了`Widget对象`和`控制块`。
这种优化减少了程序的静态大小，因为代码只包含一个内存分配调用，并且它提高了可执行代码的速度，因为`内存只分配一次`（因为放在一个函数体内了，可以实现：一次性地申请所需的内存）。
此外，使用`std::make_shared`避免了对控制块中的某些簿记信息的需要，潜在地减少了程序的总内存占用。
对于`std::make_shared`的效率分析同样适用于`std::allocate_shared`，因此std::make_shared的性能优势也扩展到了该函数。
更倾向于使用make函数而不是直接使用new的争论非常激烈。
尽管它们在软件工程、异常安全和效率方面具有优势，但本条款的建议是，更倾向于使用make函数，而不是完全依赖于它们。这是因为有些情况下它们不能或不应该被使用。

例如，**`make`函数都不允许指定`自定义删除器`**（见Item18和19），但是`std::unique_ptr`和`std::shared_ptr`的构造函数可以接收一个删除器参数。有个Widget的自定义删除器：
```cpp
auto widgetDeleter = [](Widget* pw) { … };
```
创建一个使用它的智能指针只能直接使用`new`：
```cpp
std::unique_ptr<Widget, decltype(widgetDeleter)>
    upw(new Widget, widgetDeleter);

std::shared_ptr<Widget> spw(new Widget, widgetDeleter);
```
对于make函数，没有办法做同样的事情。

`make`函数第二个限制来自于其实现中的语法细节。
Item7解释了，当构造函数重载，有使用`std::initializer_list`作为参数的重载形式和不用其作为参数的的重载形式，
用`花括号创建`的对象 更倾向于使用`std::initializer_list`作为形参的重载形式，而用`小括号`创建对象将调用不用`std::initializer_list`作为参数的的重载形式。
`make`函数会将它们的参数`完美转发`给对象构造函数，但是它们是使用小括号还是花括号？
对某些类型，问题的答案会很不相同。
例如，在这些调用中，
```cpp
auto upv = std::make_unique<std::vector<int>>(10, 20);
auto spv = std::make_shared<std::vector<int>>(10, 20);
```
生成的智能指针指向带有10个元素的std::vector，每个元素值为20，还是指向带有两个元素的std::vector，其中一个元素值10，另一个为20？或者结果是不确定的？
好消息是这并非不确定：两种调用都创建了10个元素，每个值为20的std::vector。
这意味着在`make`函数中，`完美转发`使用`小括号`，而不是花括号。
坏消息是**如果你想用`花括号初始化`指向的对象，你必须直接使用`new`**。
使用`make`函数会需要能够完美转发花括号初始化的能力，但是，正如`Item30`所说，花括号初始化无法完美转发。
但是，Item30介绍了一个变通的方法：使用`auto类型推导`从`花括号初始化`创建`std::initializer_list`对象（见Item2），然后将auto创建的对象传递给make函数。
```cpp
//创建std::initializer_list
auto initList = { 10, 20 };
//使用std::initializer_list为形参的构造函数创建std::vector
auto spv = std::make_shared<std::vector<int>>(initList);
```
对于`std::unique_ptr`，只有这两种情景（自定义删除器和花括号初始化）使用make函数有点问题。

对于`std::shared_ptr`和它的`make`函数，还有2个问题。都属于边缘情况，但是一些开发者常碰到，你也可能是其中之一。
`一些类`重载了`operator new`和`operator delete`。
这些函数的存在意味着`对这些类型的对象的 全局内存分配和释放 是 不合常规的`。
设计这种`定制操作`往往只会精确的分配、释放对象大小的内存。
例如，Widget类的`operator new`和`operator delete`只会处理`sizeof(Widget)`大小的内存块的分配和释放。
这种系列行为不太适用于`std::shared_ptr`对自定义分配（通过std::allocate_shared）和释放（通过自定义删除器）的支持，因为`std::allocate_shared`需要的内存总大小不等于动态分配的对象大小，还需要再加上`控制块大小`（`std::allocate_shared`的实际实现往往不会调用特定的`operator new`）。
因此，使用`make`函数去创建重载了`operator new`和`operator delete`类的对象是个典型的糟糕想法。
与直接使用`new`相比，`std::make_shared`在`大小`和`速度`上的优势源于`std::shared_ptr`的控制块与指向的对象放在同一块内存中（通过组合出新的一个对象来实现）。
当对象的引用计数降为0，对象被销毁（即析构函数被调用）。
但是，因为控制块和对象被放在同一块分配的内存块中，直到`控制块的内存`也被销毁，对象占用的内存才被释放。
正如我说，`控制块`除了`引用计数`，还包含`簿记信息`。
`引用计数`追踪有多少`std::shared_ptrs`指向控制块，但控制块还有`第二个计数`，记录多少个`std::weak_ptrs`指向`控制块`。
第二个引用计数就是`weak count`。
（实际上，`weak count`的值不总是等于`指向控制块的 std::weak_ptr的 数目`，因为库的实现者找到一些方法在weak count中添加附加信息，促进更好的代码产生。为了本条款的目的，我们会忽略这一点，假定weak count的值等于指向控制块的std::weak_ptr的数目。）
当一个`std::weak_ptr`检测它是否过期时（见Item19），它会检测`指向的控制块中的 引用计数`（而不是weak count）。
如果引用计数是0（即对象没有std::shared_ptr再指向它，已经被销毁了），std::weak_ptr就已经过期。
否则就没过期。
只要`std::weak_ptrs`引用一个控制块（即`weak count`大于零），该控制块必须继续存在（要不然`std::weak_ptr`就无从访问了）。
只要`控制块`存在，包含它的`内存`就必须保持分配。
通过`std::shared_ptr`的`make`函数分配的内存，直到最后一个`std::shared_ptr`和最后一个指向它的`std::weak_ptr`已被销毁，才会释放。
如果对象类型非常大，而且`销毁最后一个std::shared_ptr`和`销毁最后一个std::weak_ptr`之间的时间很长，那么在`销毁对象`和`释放它所占用的内存`之间可能会出现延迟。
（是`std::make_shared`特有的情况！！！而`std::shared_ptr`中，控制块内存和对象内存是分开的）
```cpp
class ReallyBigType { … };

auto pBigObj =                          //通过std::make_shared  创建一个大对象
    std::make_shared<ReallyBigType>();

…           //创建std::shared_ptrs和std::weak_ptrs指向这个对象，使用它们

…           //最后一个std::shared_ptr在这销毁，但std::weak_ptrs还在

…           //在这个阶段，原来分配给大对象的内存还分配着

…           //最后一个std::weak_ptr在这里销毁；控制块和对象的内存被释放
```
直接只用new，一旦最后一个`std::shared_ptr`被销毁，ReallyBigType对象的内存就会被释放：
```cpp
class ReallyBigType { … };              //和之前一样

std::shared_ptr<ReallyBigType> pBigObj(new ReallyBigType);  //通过new创建大对象

…           //像之前一样，创建std::shared_ptrs和std::weak_ptrs指向这个对象，使用它们
            
…           //最后一个std::shared_ptr在这销毁,但std::weak_ptrs还在；对象的内存被释放

…           //在这阶段，只有控制块的内存仍然保持分配

…           //最后一个std::weak_ptr在这里销毁；
            //控制块内存被释放
```
如果你发现自己处于不可能或不合适使用`std::make_shared`的情况下，你将想要保证自己不受我们之前看到的异常安全问题的影响。
最好的方法是**确保在直接使用`new`时，在一个不做其他事情的语句中，立即将`结果`传递到`智能指针构造函数`**。
这可以`防止 编译器生成的代码 在使用new和调用管理new出来对象的智能指针的构造函数之间 发生异常`。
例如，考虑我们前面讨论过的processWidget函数，对其非异常安全调用的一个小修改。
这一次，我们将指定一个自定义删除器:
```cpp
void processWidget(std::shared_ptr<Widget> spw,     //和之前一样
                   int priority);
void cusDel(Widget *ptr);                           //自定义删除器

// 这是非异常安全调用:
processWidget( 									    //和之前一样，
                std::shared_ptr<Widget>(new Widget, cusDel),    //潜在的内存泄漏！
                computePriority() );
```
回想一下：
如果`computePriority`在“`new Widget`”之后，而在`std::shared_ptr`构造函数之前调用，并且如果`computePriority`产生一个异常，那么动态分配的Widget将会泄漏（因为析构行为和shared_ptr的生命周期绑定了）。
这里使用`自定义删除`排除了对`std::make_shared`的使用，因此避免出现问题的方法是**将`Widget的分配`和`std::shared_ptr的构造`放入它们自己的语句中，然后使用得到的`std::shared_ptr`调用processWidget**。
这是该技术的本质，不过，正如我们稍后将看到的，我们可以对其进行调整以提高其性能：
```cpp
std::shared_ptr<Widget> spw(new Widget, cusDel);    // 确保实际对象和shared_ptr的构造都完成了
processWidget(spw, computePriority());  // 正确，但是没优化，见下
```
这是可行的，因为`std::shared_ptr`获取了传递给它的构造函数的原始指针的所有权，即使构造函数产生了一个异常。
此例中，如果spw的构造函数抛出异常（比如无法为控制块动态分配内存），仍然能够保证`cusDel`会在`“new Widget”`产生的指针上调用。

一个小小的性能问题是，在非异常安全调用中，我们将一个`右值`传递给`processWidget`：
```cpp
processWidget(
    std::shared_ptr<Widget>(new Widget, cusDel),    //实参是一个右值
    computePriority()
);
```
但是在异常安全调用中，我们传递了`左值`：
```cpp
processWidget(spw, computePriority());              //实参是左值
```
因为`processWidget`的`std::shared_ptr`形参是`传值`，从`右值`构造只需要移动，而`传递左值`构造需要拷贝。
对`std::shared_ptr`而言，这种区别是有意义的，**因为`拷贝`std::shared_ptr需要对引用计数`原子递增`，`移动`则不需要对引用计数有操作**。
为了使异常安全代码达到非异常安全代码的性能水平，我们需要用`std::move`将`spw`转换为`右值`（见Item23）：
```cpp
processWidget(std::move(spw), computePriority());   //高效且异常安全
```
这很有趣，也值得了解，但通常是无关紧要的，因为您很少有理由不使用make函数。除非你有令人信服的理由这样做，否则你应该使用make函数。

请记住：
* 和直接使用new相比，make函数消除了代码重复，提高了异常安全性。
    对于`std::make_shared`和`std::allocate_shared`，生成的代码更小更快。
* 不适合使用make函数的情况包括需要指定自定义删除器和希望用花括号初始化。
* 对于std::shared_ptrs，其他**不建议使用`make`函数的情况**包括：(1)有自定义内存管理的类；(2)特别关注内存的系统，非常大的对象，以及std::weak_ptrs比对应的std::shared_ptrs活得更久。







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



# 条款22：当使用Pimpl惯用法，请在实现文件中定义特殊成员函数
如果你曾经与`过多的编译次数`斗争过，你会对`Pimpl（pointer to implementation）惯用法`很熟悉。
凭借这样一种技巧，你可以将类数据成员替换成一个指向包含具体实现的类（或结构体）的指针，并将放在主类（primary class）的数据成员们移动到实现类（implementation class）去，而这些数据成员的访问将`通过指针间接访问`。
举个例子，假如有一个类Widget看起来如下：
```cpp
class Widget() {                    //定义在头文件“widget.h”
public:
    Widget();
    // …
private:
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;              //Gadget是用户自定义的类型
};
```
因为类Widget的数据成员包含有类型std::string，std::vector和Gadget，定义有这些类型的头文件在类Widget编译的时候，必须被包含进来，这意味着类Widget的使用者必须要`#include <string>`，`<vector>`以及`gadget.h`。
这些头文件将会增加类Widget使用者的编译时间，并且让这些使用者依赖于这些头文件。
如果一个头文件的内容变了，类Widget使用者也必须要重新编译。`标准库文件<string>和<vector>`不是很常变，但是`gadget.h`可能会经常修订。
在C++98中使用Pimpl惯用法，可以把Widget的数据成员替换成一个原始指针，指向一个 `已经被声明过 却还未被定义的` 结构体，如下:

`接口类`
```cpp
class Widget                        //仍然在“widget.h”中
{
public:
    Widget();
    ~Widget();                      //析构函数在后面会分析
    // …

private:
    struct Impl;                    //使用前向声明  声明一个 实现结构体
    Impl *pImpl;                    //以及指向它的指针
};
```

因为类Widget不再提到类型std::string，std::vector以及Gadget，Widget的使用者不再需要为了这些类型而引入头文件。
`这可以加速编译，并且意味着，如果这些头文件中有所变动，Widget的使用者不会受到影响。`
一个已经被声明，却还未被实现的类型，被称为`不完整类型（incomplete type）`。
Widget::Impl就是这种类型。 你能对一个不完整类型做的事很少，但是声明一个指向它的指针是可以的。Pimpl惯用法利用了这一点。
* Pimpl惯用法的第一步，是声明一个数据成员，它是个指针，指向一个不完整类型。 
* 第二步是动态分配和回收一个对象，该对象包含那些以前在原来的类中的数据成员。
内存分配和回收的代码都写在实现文件里，比如，对于类Widget而言，写在Widget.cpp里:
```cpp
#include "widget.h"             //以下代码均在实现文件“widget.cpp”里
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl {           //含有之前在Widget中的数据成员的Widget::Impl类型的定义
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};

Widget::Widget()                //为此Widget对象分配数据成员
: pImpl(new Impl)   // ！！！
{}

Widget::~Widget()               //销毁数据成员
{ delete pImpl; }   // ！！！
```
在这里我把#include命令写出来是为了明确一点，对于std::string，std::vector和Gadget的头文件的整体依赖依然存在。
然而，这些依赖从头文件widget.h（它被所有Widget类的使用者包含，并且对他们可见）移动到了`widget.cpp`（该文件只被Widget类的实现者包含，并只对他可见）。
我高亮了其中动态分配和回收Impl对象的部分（译者注：markdown高亮不了，实际高亮的是new Impl和delete pImpl;两个语句）。
这就是为什么我们需要Widget的析构函数————我们需要Widget被销毁时回收该对象。
但是，我展示给你们看的是一段C++98的代码，散发着一股已经过去了几千年的腐朽气息（没必要用裸指针）。
它使用了原始指针，原始的new和原始的delete，一切都让它如此的...原始。
这一章建立在“智能指针比原始指针更好”的主题上，并且，如果我们想要的只是在类Widget的构造函数动态分配Widget::impl对象，在Widget对象销毁时一并销毁它，`std::unique_ptr`（见Item18）是最合适的工具。
在头文件中用std::unique_ptr替代原始指针，就有了头文件中如下代码:
```cpp
class Widget {                      //在“widget.h”中
public:
    Widget();
    // …

private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;    //使用智能指针而不是原始指针
};
```

实现文件也可以改成如下：
```cpp
#include "widget.h"                 //在“widget.cpp”中
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl {               //跟之前一样
    std::string name;
    std::vector<double> data;
    Gadget g1,g2,g3;
};

Widget::Widget()                    //根据条款21，通过std::make_unique
: pImpl(std::make_unique<Impl>())   //来创建std::unique_ptr
{}
```
你会注意到，Widget的析构函数不存在了。这是因为我们没有代码加在里面了。 
std::unique_ptr在自身析构时，会自动销毁它所指向的对象，所以我们自己无需手动销毁任何东西。这就是智能指针的众多优点之一：它使我们从手动资源释放中解放出来。

以上的代码能编译，但是，最普通的Widget用法却会导致编译出错：
```cpp
#include "widget.h"

Widget w;                           //错误！
```
你所看到的错误信息根据编译器不同会有所不同，但是其文本一般会提到一些有关于“把sizeof或delete应用到不完整类型上”的信息。对于不完整类型，使用以上操作是禁止的。
在`Pimpl惯用法`中使用`std::unique_ptr`会抛出错误，有点惊悚，因为：
* 第一  std::unique_ptr宣称它支持不完整类型，
* 第二  Pimpl惯用法是std::unique_ptr的最常见的使用情况之一。
幸运的是，让这段代码能正常运行很简单。
只需要对上面出现的问题的原因有一个基础的认识就可以了。
在对象w被析构时（例如离开了作用域），问题出现了。
在这个时候，它的`析构函数`被调用。我们在类的定义里使用了std::unique_ptr，所以我们没有声明一个析构函数，因为我们并没有任何代码需要写在里面。根据编译器自动生成的特殊成员函数的规则（见 Item17），编译器会自动为我们生成一个析构函数。 在这个析构函数里，编译器会插入一些代码来调用类Widget的数据成员pImpl的析构函数。 pImpl是一个`std::unique_ptr<Widget::Impl>`，也就是说，一个`使用默认删除器的std::unique_ptr`。
默认删除器是一个函数，它使用delete来销毁内置于std::unique_ptr的原始指针。然而，在使用delete之前，通常会使 默认删除器使用C++11的特性 `static_assert`来确保 原始指针指向的类型不是一个不完整类型。 当编译器为Widget w的析构生成代码时，它会遇到`static_assert`检查并且失败，这通常是错误信息的来源。
（也就是编译器在生成默认析构函数的时候，生成其代码时，对指针指向的类型是否是不完整的类型）
这些错误信息只在对象w销毁的地方出现，因为类Widget的析构函数，正如其他的编译器生成的特殊成员函数一样，是暗含`inline`属性的。
错误信息自身往往指向对象w被创建的那行，因为这行代码明确地构造了这个对象，导致了后面潜在的析构。
**为了解决这个问题，你只需要确保在编译器生成销毁`std::unique_ptr<Widget::Impl>`的代码之前， `Widget::Impl`已经是一个`完整类型（complete type）`。**
当编译器“看到”它的定义的时候，该类型就成为完整类型了。
但是 Widget::Impl的定义在widget.cpp里。成功编译的关键，就是在widget.cpp文件内，让编译器在“看到” Widget的析构函数实现之前（也即编译器插入的，用来销毁std::unique_ptr这个数据成员的代码的，那个位置），`先定义Widget::Impl`。

做出这样的调整很容易。只需要先在widget.h里，只声明类Widget的析构函数，但不要在这里定义它：
```cpp
class Widget {                  //跟之前一样，在“widget.h”中
public:
    Widget();
    ~Widget();                  //只有声明语句
    …

private:                        //跟之前一样
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};
```
在widget.cpp文件中，在结构体Widget::Impl被定义之后，再定义析构函数：
```cpp
#include "widget.h"                 //跟之前一样，在“widget.cpp”中
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl {               //跟之前一样，定义Widget::Impl
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
}

Widget::Widget()                    //跟之前一样
: pImpl(std::make_unique<Impl>())
{}

Widget::~Widget()                   //析构函数的定义（译者注：这里高亮）！！！使得编译器到这里才生成代码。此时Impl类已经有明确的定义
{}
```
这样就可以了，并且这样增加的代码也最少，你声明Widget析构函数只是为了在 Widget 的实现文件中（译者注：指widget.cpp）写出它的定义，但是如果你想强调编译器自动生成的析构函数会做和你一样正确的事情，你可以直接使用“= default”定义析构函数体
```cpp
Widget::~Widget() = default;        //同上述代码效果一致
```
使用了Pimpl惯用法的类自然适合支持`移动操作`，因为编译器自动生成的移动操作正合我们所意：对其中的std::unique_ptr进行移动。
正如Item17所解释的那样，`声明 一个类Widget的 析构函数 会阻止 编译器生成移动操作`（因为可能涉及资源的管理，默认生成的析构函数不够用了），所以如果你想要支持移动操作，你必须自己声明相关的函数。
考虑到编译器自动生成的版本会正常运行，你可能会很想按如下方式实现它们：
```cpp
class Widget {                                  //仍然在“widget.h”中
public:
    Widget();
    ~Widget();

    Widget(Widget&& rhs) = default;             //  思路正确，
    Widget& operator=(Widget&& rhs) = default;  //  但代码错误
    …

private:                                        //跟之前一样
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};
```
这样的做法会导致同样的错误，和之前的声明一个不带析构函数的类的错误一样，并且是因为同样的原因。
编译器生成的移动赋值操作符，在重新赋值之前，需要先销毁指针pImpl指向的对象。
然而在Widget的头文件里，pImpl指针指向的是一个不完整类型。
移动构造函数的情况有所不同。 移动构造函数的问题是：`编译器自动生成的代码里，包含有 抛出异常的 事件，在这个事件里会生成销毁pImpl的代码`。
然而，销毁pImpl需要Impl是一个完整类型。

因为这个问题同上面一致，所以解决方案也一样————把移动操作的定义移动到实现文件里：
```cpp
class Widget {                          //仍然在“widget.h”中
public:
    Widget();
    ~Widget();

    Widget(Widget&& rhs);               //只有声明
    Widget& operator=(Widget&& rhs);
    // …

private:                                //跟之前一样
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

#include <string>                   //跟之前一样，仍然在“widget.cpp”中
// …
    
struct Widget::Impl { … };          //跟之前一样

Widget::Widget()                    //跟之前一样
: pImpl(std::make_unique<Impl>())
{}

Widget::~Widget() = default;        //跟之前一样

Widget::Widget(Widget&& rhs) = default;             //这里定义
Widget& Widget::operator=(Widget&& rhs) = default;
```

Pimpl惯用法是`用来 减少类的实现 和 类使用者之间的编译依赖 的一种方法`，但是，从概念而言，使用这种惯用法并不改变这个类的表现。
原来的类Widget包含有std::string，std::vector和Gadget数据成员，并且，假设类型Gadget，如同std::string和std::vector一样，允许复制操作，所以类Widget支持复制操作也很合理。
我们必须要自己来写这些函数，因为
    第一，对包含有只可移动（move-only）类型，如`std::unique_ptr`的类，编译器不会生成复制操作；
    第二，即使编译器帮我们生成了，生成的复制操作也只会复制std::unique_ptr（也即`浅拷贝`（shallow copy）），而实际上我们需要复制指针所指向的对象（也即深拷贝（deep copy））。

使用我们已经熟悉的方法，我们在头文件里声明函数，而在实现文件里去实现他们:
```cpp
class Widget {                          //仍然在“widget.h”中
public:
    // …

    Widget(const Widget& rhs);          //只有声明
    Widget& operator=(const Widget& rhs);

private:                                //跟之前一样
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};


#include <string>                   //跟之前一样，仍然在“widget.cpp”中
// …
    
struct Widget::Impl { … };          //跟之前一样

Widget::~Widget() = default;		//其他函数，跟之前一样

Widget::Widget(const Widget& rhs)   //  拷贝构造函数
: pImpl(std::make_unique<Impl>(*rhs.pImpl)) // 触发Impl的拷贝构造函数，构造了新对象，是深拷贝
{}

Widget& Widget::operator=(const Widget& rhs)    //拷贝operator=
{
    *pImpl = *rhs.pImpl;
    return *this;
}
```
两个函数的实现都比较中规中矩。在每个情况中，我们都只从源对象（rhs）中，复制了结构体Impl的内容到目标对象中（*this）。
我们利用了`编译器 会为我们自动生成结构体Impl的复制操作函数 的机制`，而不是逐一复制结构体Impl的成员，自动生成的复制操作能自动复制每一个成员。
因此我们通过调用编译器生成的Widget::Impl的复制操作函数来实现了类Widget的复制操作。
在复制构造函数中，注意，我们仍然遵从了Item21的建议，使用`std::make_unique`而非直接使用new。
为了实现Pimpl惯用法，std::unique_ptr是我们使用的智能指针，因为位于对象内部的pImpl指针（例如，在类Widget内部），对所指向的对应实现的对象的享有`独占所有权`。
然而，有趣的是，如果我们使用std::shared_ptr而不是std::unique_ptr来做pImpl指针，我们会发现本条款的建议不再适用。
我们不需要在类Widget里声明析构函数，没有了用户定义析构函数，编译器将会愉快地生成移动操作，并且将会如我们所期望般工作。widget.h里的代码如下，
```cpp
class Widget {                      //在“widget.h”中
public:
    Widget();
    // …                            //没有析构函数和移动操作的声明

private:
    struct Impl;
    std::shared_ptr<Impl> pImpl;    //用std::shared_ptr
};                                  //而不是std::unique_ptr
```
这是#include了widget.h的客户代码，
```cpp
Widget w1;
auto w2(std::move(w1));     //移动构造w2
w1 = std::move(w2);         //移动赋值w1
```
这些都能编译，并且工作地如我们所望：w1将会被默认构造，它的值会被移动进w2，随后值将会被移动回w1，然后两者都会被销毁（因此导致指向的Widget::Impl对象一并也被销毁）。

`std::unique_ptr`和`std::shared_ptr`在pImpl指针上的表现上的区别的深层原因在于，他们`支持自定义删除器的方式不同`。
* 对std::unique_ptr而言，删除器的类型是这个智能指针的一部分，这让编译器有可能生成更小的运行时数据结构和更快的运行代码。
这种更高效率的后果之一就是`std::unique_ptr`指向的类型，在编译器的`生成特殊成员函数（如析构函数，移动操作）`被调用时，必须已经是一个`完整类型`。
* 而对std::shared_ptr而言，删除器的类型不是该智能指针的一部分，这让它会生成更大的运行时数据结构和稍微慢点的代码，但是当编译器生成的特殊成员函数被使用的时候，指向的对象不必是一个完整类型。（译者注：知道std::unique_ptr和std::shared_ptr的实现，这一段才比较容易理解。）

对于Pimpl惯用法而言，在std::unique_ptr和std::shared_ptr的特性之间，没有一个比较好的折中。
因为对于像Widget的类以及像Widget::Impl的类之间的关系而言，他们是独享占有权关系，这让std::unique_ptr使用起来很合适。
然而，有必要知道，在其他情况中，当共享所有权存在时，std::shared_ptr是很适用的选择的时候，就没有std::unique_ptr所必需的声明——定义（function-definition）这样的麻烦事了。

请记住：
* Pimpl惯用法通过减少在类实现和类使用者之间的编译依赖来减少编译时间。
* 对于std::unique_ptr类型的pImpl指针，需要在头文件的类里声明特殊的成员函数，但是在实现文件里面来实现他们。即使是编译器自动生成的代码可以工作，也要这么做。
* 以上的建议只适用于std::unique_ptr，不适用于std::shared_ptr。



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




# 条款24：区分通用引用与右值引用
据说，真相使人自由，然而在特定的环境下，一个精心挑选的谎言也同样使人解放。这一条款就是这样一个谎言。因为我们在和软件打交道，然而，让我们避开“谎言（lie）”这个词，不妨说，本条款包含了一种“`抽象`（abstraction）”。
为了声明一个指向某个类型T的右值引用，你写下了T&&。由此，一个合理的假设是，当你看到一个“T&&”出现在源码中，你看到的是一个右值引用。
唉，事情并不如此简单:
```cpp
void f(Widget&& param);             //右值引用
Widget&& var1 = Widget();           //右值引用
auto&& var2 = var1;                 //不是右值引用

template<typename T>
void f(std::vector<T>&& param);     //右值引用

template<typename T>
void f(T&& param);                  //不是右值引用
```
事实上，“T&&”有两种不同的意思。
* 第一种，当然是右值引用。这种引用表现得正如你所期待的那样：它们只绑定到右值上，并且它们主要的存在原因就是为了识别可以移动操作的对象。
* “T&&”的另一种意思是，它既可以是右值引用，也可以是左值引用。
这种引用在源码里看起来像右值引用（即“T&&”），但是它们可以表现得像是左值引用（即“T&”）。它们的二重性使它们既可以绑定到右值上（就像右值引用），也可以绑定到左值上（就像左值引用）。
此外，它们还可以绑定到const或者non-const的对象上，也可以绑定到volatile或者non-volatile的对象上，甚至可以绑定到既const又volatile的对象上。
它们可以绑定到几乎任何东西。
这种空前灵活的引用值得拥有自己的名字。我把它叫做`通用引用（universal references）`。（Item25解释了std::forward几乎总是可以应用到通用引用上，并且在这本书即将出版之际，一些C++社区的成员已经开始将这种通用引用称之为转发引用（forwarding references））。

在两种情况下会出现通用引用。
最常见的一种是`函数模板形参`，正如在之前的示例代码中所出现的例子：
```cpp
template<typename T>
void f(T&& param);                  //param是一个通用引用
```

第二种情况是`auto声明符`，它是从以上示例中拿出的：
```cpp
auto&& var2 = var1;                 //var2是一个通用引用
```
这两种情况的共同之处就是都存在`类型推导（type deduction）`。
在模板f的内部，`param`的类型需要被推导，而在变量var2的声明中，`var2`的类型也需要被推导。
同以下的例子相比较（同样来自于上面的示例代码），下面的例子不带有类型推导。
如果你看见“T&&”不带有类型推导，那么你看到的就是一个右值引用：
```cpp
void f(Widget&& param);         //没有类型推导（因为已经明确了），
                                //param是一个右值引用
Widget&& var1 = Widget();       //没有类型推导，
                                //var1是一个右值引用
```
因为通用引用是引用，所以它们必须被初始化。
**一个`通用引用`的`初始值`决定了它是代表了右值引用还是左值引用。**
如果初始值是一个右值，那么通用引用就会是对应的右值引用，如果初始值是一个左值，那么通用引用就会是一个左值引用。
对那些是函数形参的通用引用来说，初始值在调用函数的时候被提供：
```cpp
template<typename T>
void f(T&& param);              //param是一个通用引用

Widget w;
f(w);                           //传递给函数f一个左值；param的类型
                                //将会是Widget&，也即左值引用

f(std::move(w));                //传递给f一个右值；param的类型会是
                                //Widget&&，即右值引用  （std::move将表达式强制转为右值类型）
```
对一个通用引用而言，类型推导是必要的，但是它还不够。
引用声明的形式必须正确，并且该形式是被限制的。它必须恰好为“`T&&`”。再看看之前我们已经看过的代码示例：
```cpp
template <typename T>
void f(std::vector<T>&& param);     //param是一个右值引用
```
当函数f被调用的时候，类型T会被推导（除非调用者显式地指定它，这种边缘情况我们不考虑）。
但是param的类型声明并不是T&&，而是一个`std::vector<T>&&`。这排除了param是一个通用引用的可能性。
param因此是一个右值引用————当你向函数f传递一个左值时，你的编译器将会乐于帮你确认这一点:
```cpp
std::vector<int> v;
f(v);                           //错误！不能将左值绑定到右值引用
```
即使一个简单的`const`修饰符的出现，也足以使一个引用失去成为通用引用的资格:
```cpp
template <typename T>
void f(const T&& param);        //param是一个右值引用
```

如果你在一个模板里面看见了一个函数形参类型为“T&&”，你也许觉得你可以假定它是一个通用引用。错！这是由于 `在模板内部并不保证一定会发生类型推导`。
考虑如下push_back成员函数，来自`std::vector`：
```cpp
template<class T, class Allocator = allocator<T>>   //来自C++标准
class vector
{
public:
    void push_back(T&& x);
    // …
}
```
push_back函数的形参当然有一个通用引用的正确形式，然而，在这里并没有发生类型推导。
**因为`push_back`在有一个特定的vector实例之前不可能存在，而实例化vector时的类型已经决定了push_back的声明。**（`push_back`不是一个模板函数）
也就是说，
```cpp
std::vector<Widget> v;
```
将会导致std::vector模板被实例化为以下代码：
```cpp
class vector<Widget, allocator<Widget>> {
public:
    void push_back(Widget&& x);             //右值引用
    // …
};
```
现在你可以清楚地看到，函数push_back不包含任何类型推导。
`push_back`对于`vector<T>`而言（有两个函数————它被重载了）总是声明了一个类型为`rvalue-reference-to-T`的形参。

作为对比，std::vector内的概念上相似的成员函数`emplace_back`，却确实包含`类型推导`（因为它是模板函数）:
```cpp
template<class T, class Allocator = allocator<T>>   //依旧来自C++标准
class vector {
public:
    template <class... Args>
    void emplace_back(Args&&... args);
    // …
};
```
这儿，类型参数（type parameter）Args是独立于vector的类型参数T的，所以Args会在每次emplace_back被调用的时候被推导。（好吧，Args实际上是一个`parameter pack`，而不是一个类型参数，但是为了方便讨论，我们可以把它当作是一个类型参数。）

虽然函数emplace_back的类型参数被命名为Args，但是它仍然是一个通用引用，这补充了我之前所说的，通用引用的格式必须是“T&&”。
你使用的名字T并不是必要的。
举个例子，如下模板接受一个通用引用，因为形式（“type&&”）是正确的，并且param的类型将会被推导（重复一次，不考虑边缘情况，即当调用者明确给定类型的时候）。
```cpp
template<typename MyTemplateType>           //param是通用引用
void someFunc(MyTemplateType&& param);
```
我之前提到，类型为auto的变量可以是通用引用。
更准确地说，类型声明为`auto&&`的变量是通用引用，因为会发生类型推导，并且它们具有正确形式(T&&)。
auto类型的通用引用不如函数模板形参中的通用引用常见，但是它们在C++11中常常突然出现。
而它们在C++14中出现得更多，因为C++14的lambda表达式可以声明`auto&&`类型的形参。
举个例子，如果你想写一个C++14标准的lambda表达式，来`记录任意函数调用的时间开销`，你可以这样写：
```cpp
auto timeFuncInvocation =
    [](auto&& func, auto&&... params)           //C++14
    {
        start timer;
        std::forward<decltype(func)>(func)(     //对params调用func
            std::forward<delctype(params)>(params)...
        );
        stop timer and record elapsed time;
    };
```
如果你对lambda里的代码“`std::forward<decltype(blah blah blah)>`”反应是“这是什么鬼...?!”，只能说你可能还没有读Item33。
别担心。在本条款，重要的事是lambda表达式中声明的`auto&&`类型的形参。func是一个通用引用，可以被绑定到任何可调用对象，无论左值还是右值。
args是0个或者多个通用引用（即它是个通用引用parameter pack），它可以`绑定到 任意数目、任意类型的 对象上`。
多亏了auto类型的通用引用，函数timeFuncInvocation可以对近乎任意（pretty much any）函数进行计时。(如果你想知道任意（any）和近乎任意（pretty much any）的区别，往后翻到Item30)。
牢记整个本条款————通用引用的基础————是一个谎言，噢不，是一个“抽象”。
其底层真相被称为`引用折叠（reference collapsing）`，Item28的专题将致力于讨论它。
但是这个真相并不降低该抽象的有用程度。区分右值引用和通用引用将会帮助你更准确地阅读代码（“究竟我眼前的这个T&&是只绑定到右值还是可以绑定任意对象呢？”），并且，当你在和你的合作者交流时，它会帮助你避免歧义（“在这里我在用一个通用引用，而非右值引用”）。
它也可以帮助你弄懂Item25和26，它们依赖于右值引用和通用引用的区别。所以，拥抱这份抽象，陶醉于它吧。就像牛顿的力学定律（本质上不正确），比起爱因斯坦的广义相对论（这是真相）而言，往往更简单，更易用。所以通用引用的概念，相较于穷究引用折叠的细节而言，是更合意之选。

请记住：
* 如果一个函数模板形参的类型为`T&&`，并且T`需要被推导`得知，或者如果一个对象被声明为`auto&&`，这个形参或者对象就是一个通用引用。
* 如果类型声明的形式不是标准的`type&&`，或者如果类型推导没有发生，那么`type&&`代表一个右值引用。
* 通用引用，如果它被右值初始化，就会对应地成为右值引用；如果它被左值初始化，就会成为左值引用。



# 条款30：熟悉完美转发失败的情况
C++11最显眼的功能之一就是`完美转发功能`。
完美转发，太完美了！哎，开始使用，你就发现“完美”，理想与现实还是有差距。
C++11的完美转发是非常好用，但是只有当你愿意忽略一些误差情况（译者注：就是完美转发失败的情况），这个条款就是使你熟悉这些情形。
在我们开始误差探索之前，有必要回顾一下“完美转发”的含义。
**“转发”**仅表示**将一个函数的形参传递——就是转发——给另一个函数**。
对于第二个函数（被传递的那个）目标是收到与第一个函数（执行传递的那个）完全相同的对象。
这规则`排除了按值传递的形参`，因为它们是原始调用者传入内容的`拷贝`。
我们希望被转发的函数能够使用最开始传进来的那些对象。
指针形参也被排除在外，因为我们不想强迫调用者传入指针。关于通常目的的转发，我们将处理`引用形参`。
`完美转发（perfect forwarding）`意味着我们不仅转发对象，我们还转发显著的特征：它们的类型，是`左值`还是`右值`，是`const`还是`volatile`。
结合到我们会处理`引用形参`，这意味着我们将使用`通用引用`（参见Item24），因为通用引用形参被传入实参时才确定---==是左值还是右值。
假定我们有一些函数f，然后想编写一个转发给它的函数（事实上是一个函数模板）。
我们需要的核心看起来像是这样：
```cpp
template<typename T>
void fwd(T&& param)             //接受任何实参
{
    f(std::forward<T>(param));  //转发给f
}
```
从本质上说，`转发函数`是`通用的`。
例如`fwd`模板，接受任何类型的实参，并转发得到的任何东西。
这种通用性的逻辑扩展是，转发函数不仅是模板，而且是可变模板，因此可以接受任何数量的实参。
`fwd`的可变形式如下：
```cpp
template<typename... Ts>
void fwd(Ts&&... params)            //接受任何实参
{
    f(std::forward<Ts>(params)...); //转发给f
}
```
这种形式你会在标准化容器置入函数（`emplace` functions）中（参见Item42）和智能指针的工厂函数`std::make_unique`和`std::make_shared`中（参见Item21）看到，当然还有其他一些地方。
给定我们的`目标函数f`和`转发函数fwd`，**如果f使用某特定实参会执行某个操作，但是fwd使用相同的实参会执行不同的操作，完美转发就会失败**
```cpp
f( expression );        //调用f执行某个操作
fwd( expression );		//但调用fwd执行另一个操作，则fwd不能完美转发expression给f
```
导致这种失败的实参种类有很多。
知道它们是什么以及如何解决它们很重要，因此让我们来看看无法做到完美转发的实参类型。

## 花括号初始化器
假定f这样声明：
```cpp
void f(const std::vector<int>& v);
```
在这个例子中，用花括号初始化调用f通过编译，
```cpp
f({ 1, 2, 3 });         //可以，“{1, 2, 3}”隐式转换为std::vector<int>
```
但是传递相同的列表初始化给fwd不能编译
```cpp
fwd({ 1, 2, 3 });       //错误！不能编译
```
这是因为这是完美转发失效的一种情况。
所有这种错误有相同的原因。
在对f的直接调用（例如`f({ 1, 2, 3 })`），编译器看看调用地传入的实参，看看f声明的形参类型。
它们把调用地的实参和声明的实参进行比较，看看是否匹配，并且必要时执行`隐式转换操作`使得调用成功。
在上面的例子中，从`{ 1, 2, 3 }`生成了临时`std::vector<int>`对象，因此f的形参v会绑定到`std::vector<int>`对象上。
当通过调用`函数模板``fwd`间接调用f时，编译器不再把调用地传入给fwd的`实参`和f的声明中`形参`类型进行比较。
（因为函数模板在实例化代码时，有推导行为，而非模板函数是明确的，只是进行重载决议，在必要时进行隐式转换）
而是`推导`传入给fwd的实参类型，然后比较`推导后的实参类型`和`f的形参声明类型`。

当下面情况任何一个发生时，完美转发就会失败：
* 编译器不能推导出fwd的一个或者多个形参类型。
    这种情况下代码无法编译。
* 编译器推导“错”了fwd的一个或者多个形参类型。 
    在这里，“错误”可能意味着fwd的实例将无法使用推导出的类型进行编译，但是也可能意味着使用fwd的推导类型调用f，与用传给fwd的实参直接调用f表现出不一致的行为。

这种不同行为的原因可能是因为f是个重载函数的名字，并且由于是“不正确的”类型推导，在fwd内部调用的f重载和直接调用的f重载不一样。
在上面的`fwd({ 1, 2, 3 })`例子中，问题在于，将`花括号初始化`传递给未声明为`std::initializer_list`的函数模板形参，被判定为————就像标准说的————“非推导上下文”。
简单来讲，这意味着编译器不准在对fwd的调用中推导表达式`{ 1, 2, 3 }`的类型，因为fwd的形参没有声明为`std::initializer_list`。
对于fwd形参的推导类型被阻止，编译器只能拒绝该调用（对于函数模板的类型推导，编译器就是不支持传代表了`std::initialization_list`的花括号表达式，而`auto`直接假定花括号表示std::initialization_list）。
有趣的是，Item2说明了使用花括号初始化的auto的变量的类型推导是成功的。
这种变量被视为`std::initializer_list`对象，在转发函数应推导出类型为`std::initializer_list`的情况，这提供了一种简单的解决方法————使用auto声明一个局部变量，然后将局部变量传进转发函数：
```cpp
auto il = { 1, 2, 3 };  //il的类型被推导为std::initializer_list<int>
fwd(il);                //可以，完美转发il给f   类型明确是std::initializer_list了，而不是花括号这种还需要推导的表达式
```

## 0或者NULL作为空指针
Item8说明当你试图传递`0`或者`NULL`作为`空指针`给模板时，类型推导会出错，会把传来的实参推导为一个`整型类型（典型情况为int）`而不是`指针类型`。
结果就是不管是0还是NULL都不能作为空指针被完美转发。
解决方法非常简单，传一个`nullptr`而不是0或者NULL。具体的细节，参考Item8。

## 仅有声明的整型`static const`数据成员
通常，无需在类中定义整型static const数据成员；声明就可以了。
这是因为编译器会对此类成员实行`常量传播（const propagation）！！！`，因此`消除了保留内存的需要`。
比如，考虑下面的代码：
```cpp
class Widget {
public:
    static const std::size_t MinVals = 28;  //MinVal的声明
    …
};
…                                           //没有MinVals定义

std::vector<int> widgetData;
widgetData.reserve(Widget::MinVals);        //使用MinVals
```
这里，我们使用`Widget::MinVals（或者简单点MinVals）`来确定widgetData的初始容量，即使MinVals缺少定义。
编译器通过将值28放入所有提到MinVals的位置来补充缺少的定义（就像它们被要求的那样）。
没有为MinVals的值留存储空间是没有问题的。
如果要使用MinVals的地址（例如，有人创建了指向MinVals的指针），则MinVals需要存储（这样指针才有可指向的东西），尽管上面的代码仍然可以编译，但是链接时就会报错，直到为MinVals提供定义。
按照这个思路，想象下f（fwd要转发实参给它的那个函数）这样声明：
```cpp
void f(std::size_t val);
```
使用`MinVals`调用f是可以的，因为编译器直接将值28代替MinVals：
```cpp
f(Widget::MinVals);         //可以，视为“f(28)”
```
不过如果我们尝试通过fwd调用f，事情不会进展那么顺利：
```cpp
fwd(Widget::MinVals);       //错误！不应该链接
```
代码可以编译，但是不应该链接。
如果这让你想到使用MinVals地址会发生的事，确实，底层的问题是一样的。
尽管代码中没有使用MinVals的地址，但是fwd的形参是`通用引用`，而引用，在编译器生成的代码中，通常被视作`指针`。
**在程序的二进制底层代码中（以及硬件中）`指针`和`引用`是`一样的`。**
在这个水平上，引用只是可以自动解引用的指针。
在这种情况下，通过引用传递MinVals实际上与通过指针传递MinVals是一样的，因此，**必须有`内存`使得指针可以指向（光靠编译器生成代码时，简单地替换一些常量是不够的）**。
**通过引用传递的整型`static const`数据成员，通常需要`定义它们`**，这个要求可能会造成在不使用完美转发的代码成功的地方，使用等效的完美转发失败。（译者注：这里意思应该是没有定义，完美转发就会失败）
可能你也注意到了在上述讨论中我使用了一些模棱两可的词：
代码“不应该”链接。引用“通常”被看做指针。传递整型static const数据成员“通常”要求定义。看起来就像有些事情我没有告诉你......
确实，根据标准，通过引用传递MinVals要求有定义。
**但不是所有的实现都强制要求这一点。所以，取决于你的编译器和链接器，你可能发现你可以在未定义的情况使用完美转发，恭喜你，但是这不是那样做的理由**。
为了具有可移植性，只要给整型`static const`提供一个定义，比如这样：
```cpp
const std::size_t Widget::MinVals;  //在Widget的.cpp文件
```
注意**`定义中不要重复初始化`（这个例子中就是赋值28）。**
但是不要忽略这个细节。如果你忘了，并且在两个地方都提供了初始化，编译器就会报错，提醒你只能初始化一次。

## 重载函数的名称和模板名称
假定我们的函数f（我们想通过fwd完美转发实参给的那个函数）可以通过向其传递执行某些功能的函数来自定义其行为。
假设这个函数接受和返回值都是int，f声明就像这样：
```cpp
void f(int (*pf)(int));             //pf = “process function”
```
值得注意的是，也可以使用更简单的`非指针语法声明`。
这种声明就像这样，含义与上面是一样的：
```cpp
void f(int pf(int));                //与上面定义相同的f
```
无论哪种写法都可，现在假设我们有了一个重载函数，processVal：
```cpp
int processVal(int value);
int processVal(int value, int priority);
```
我们可以传递processVal给f，
```cpp
f(processVal);                      //可以
```
但是我们会发现一些吃惊的事情。
f要求一个函数指针作为实参，但是processVal不是一个函数指针或者一个函数，它是同名的两个不同函数。
但是，**编译器可以知道它需要哪个：匹配上f的形参类型的那个**。因此选择了仅带有一个int的processVal地址传递给f。
**工作的基本机制是`f的声明`让编译器识别出哪个是需要的processVal（f本身明确地带声明了，和传进来的函数一匹配就知道该重载哪个了！）**。
但是，`fwd`是一个`函数模板`，没有它可接受的类型的信息，使得编译器不可能决定出哪个函数应被传递：
```cpp
fwd(processVal);                    //错误！那个processVal？
```
单用`processVal`是没有类型信息的，所以就`不能类型推导`，完美转发失败。
如果我们试图使用`函数模板`而不是（或者也加上）`重载函数`的名字，同样的问题也会发生。
**一个`函数模板`不代表单独一个函数，它表示一个`函数族`**：
```cpp
template<typename T>
T workOnVal(T param)                //处理值的模板
{ … }

fwd(workOnVal);                     //错误！哪个workOnVal实例？根本不明确
```
**要让像`fwd的完美转发函数`接受一个`重载函数名`或者`模板名`，方法是`指定 要转发的那个重载 或者 实例`**。
比如，你可以创造与f相同形参类型的`函数指针`，通过processVal或者workOnVal实例化这个函数指针（这可以引导选择正确版本的processVal或者产生正确的workOnVal实例），然后传递指针给fwd：
```cpp
using ProcessFuncType =                         //写个类型定义；见条款9
    int (*)(int);

ProcessFuncType processValPtr = processVal;     //指定所需的processVal签名

fwd(processValPtr);                             //可以
fwd(static_cast<ProcessFuncType>(workOnVal));   //也可以
```
当然，这要求你知道fwd转发的函数指针的类型。
没有理由去假定完美转发函数会记录着这些东西。
毕竟，完美转发被设计为接受任何内容，所以如果没有文档告诉你要传递什么，你又从何而知这些东西呢？

## 位域
完美转发最后一种失败的情况是函数实参使用`位域`这种类型。
为了更直观的解释，IPv4的头部有如下模型：（这假定的是位域是按从最低有效位（least significant bit，lsb）到最高有效位（most significant bit，msb）布局的。
C++不保证这一点，但是编译器经常提供一种机制，允许程序员控制位域布局。）
```cpp
struct IPv4Header {
    std::uint32_t version:4,
                  IHL:4,
                  DSCP:6,
                  ECN:2,
                  totalLength:16;
    // …
};
```
如果声明我们的函数f（转发函数fwd的目标）为接收一个`std::size_t`的形参，则使用IPv4Header对象的totalLength字段进行调用没有问题：
```cpp
void f(std::size_t sz);         //要调用的函数

IPv4Header h;
…
f(h.totalLength);               //可以
```
如果通过fwd转发h.totalLength给f呢，那就是一个不同的情况了：
```cpp
fwd(h.totalLength);             //错误！
```
问题在于`fwd`的形参是引用，而`h.totalLength`是non-const位域。
听起来并不是那么糟糕，但是C++标准非常清楚地谴责了这种组合：**`non-const引用`不应该绑定到`位域`**。
禁止的理由很充分。**位域可能包含了机器字的任意部分（比如32位int的3-5位），但是这些东西无法直接寻址**。
我之前提到了在硬件层面引用和指针是一样的，所以没有办法创建一个指向任意bit的指针（C++规定你可以指向的最小单位是char），同样没有办法绑定引用到任意bit上。
**一旦意识到接收位域实参的函数都将接收位域的`副本`，就可以轻松解决位域不能完美转发的问题**。
毕竟，没有函数可以绑定引用到位域，也没有函数可以接受指向位域的指针，因为不存在这种指针。
位域可以传给的形参种类只有按值传递的形参，有趣的是，还有reference-to-const。
在传值形参的情况中，被调用的函数接受了一个位域的副本；
在传`reference-to-const`形参的情况中，标准要求这个引用实际上绑定到存放位域值的副本对象，这个对象是某种整型（比如int）。
`reference-to-const`不直接绑定到位域，而是绑定位域值拷贝到的一个普通对象。
传递位域给完美转发的关键就是利用传给的函数接受的是一个副本的事实。
你可以自己创建副本然后利用副本调用完美转发。在IPv4Header的例子中，可以如下写法：
//拷贝位域值；参看条款6了解关于初始化形式的信息
```cpp
auto length = static_cast<std::uint16_t>(h.totalLength);    // 必须拷贝一把，变成别的类型，而不是位域类型，因为位域类型不允许被引用绑定
fwd(length);                    //转发这个副本
```
总结
在大多数情况下，完美转发工作的很好。你基本不用考虑其他问题。
但是当其不工作时——当看起来合理的代码无法编译，或者更糟的是，虽能编译但无法按照预期运行时————了解完美转发的缺陷就很重要了。同样重要的是如何解决它们。在大多数情况下，都很简单。

请记住：
* 当模板类型推导失败或者推导出错误类型，完美转发会失败。
* 导致完美转发失败的实参种类有花括号初始化，作为空指针的0或者NULL，仅有声明的整型static const数据成员，模板和重载函数的名字，位域。





# 第6章 lambda表达式
lambda表达式是C++编程中的游戏规则改变者。
这有点令人惊讶，因为它没有给语言带来新的表达能力。lambda可以做的所有事情都可以通过其他方式完成。但是lambda是创建函数对象相当便捷的一种方法，对于日常的C++开发影响是巨大的。
没有lambda时，STL中的“`_if`”算法（比如，`std::find_if`，`std::remove_if`，`std::count_if`等）通常需要繁琐的谓词，但是当有lambda可用时，这些算法使用起来就变得相当方便。用比较函数（比如，`std::sort`，`std::nth_element`，`std::lower_bound`等）来自定义算法也是同样方便的。
在STL外，lambda可以快速创建std::unique_ptr和std::shared_ptr的自定义删除器（见Item18和19），并且使线程API中条件变量的谓词指定变得同样简单（参见Item39）。除了标准库，lambda有利于即时的回调函数，接口适配函数和特定上下文中的一次性函数。lambda确实使C++成为更令人愉快的编程语言。
与lambda相关的词汇可能会令人疑惑，这里做一下简单的回顾：
lambda表达式（lambda expression）就是一个表达式。下面是部分源代码。在
```cpp
std::find_if(container.begin(), container.end(),
             [](int val){ return 0 < val && val < 10; });   //译者注：本行高亮
```
中，代码的高亮部分就是lambda。
`闭包（enclosure）`
    是lambda创建的运行时 `对象`。创建的对象，学名叫闭包。
    依赖捕获模式，闭包`持有被捕获数据的 副本 或者 引用`。在上面的std::find_if调用中，闭包是作为第三个实参在运行时传递给std::find_if的对象。

`闭包类（closure class）`
    是从中实例化闭包的类。
    每个lambda都会使编译器生成唯一的`闭包类`。lambda中的语句 成为 其闭包类的 成员函数中的 可执行指令。

lambda通常被用来创建闭包，该闭包仅用作函数的实参。上面对std::find_if的调用就是这种情况。然而，闭包通常可以拷贝，所以可能有多个闭包对应于一个lambda。比如下面的代码：
```cpp
{
    int x;                                  //x是局部对象
    // …

    auto c1 =                               //c1是lambda产生的闭包的副本
        [x](int y) { return x * y > 55; };

    auto c2 = c1;                           //c2是c1的拷贝

    auto c3 = c2;                           //c3是c2的拷贝
    // …
}
```
c1，c2，c3都是lambda产生的闭包的副本。

非正式的讲，模糊lambda，闭包和闭包类之间的界限是可以接受的。
但是，在随后的Item中，区分什么存在于编译期（lambdas 和闭包类），什么存在于运行时（闭包）以及它们之间的相互关系是重要的。


条款31：避免使用默认捕获模式
C++11中有两种默认的捕获模式：`按引用捕获`和`按值捕获`。
但默认按引用捕获模式可能会带来`悬空引用`的问题，而默认按值捕获模式可能会诱骗你让你以为能解决悬空引用的问题（实际上并没有），还会让你以为你的闭包是独立的（事实上也不是独立的）。
这就是本条款的一个总结。如果你偏向技术，渴望了解更多内容，就让我们从按引用捕获的危害谈起吧。
`按引用捕获`会导致闭包中包含了 `对某个局部变量 或者 形参的引用`，变量或形参只在定义lambda的作用域中可用。
如果 该lambda创建的闭包 生命周期 超过了 局部变量或者形参的 生命周期，那么闭包中的引用将会变成悬空引用。
举个例子，假如我们有元素是过滤函数（filtering function）的一个容器，该函数接受一个int，并返回一个bool，该bool的结果表示传入的值是否满足过滤条件：
```cpp
using FilterContainer =                     //“using”参见条款9，
    std::vector<std::function<bool(int)>>;  //std::function参见条款2

FilterContainer filters;                    //过滤函数
```
我们可以添加一个过滤器，用来过滤掉5的倍数：
```cpp
filters.emplace_back(                       //emplace_back的信息见条款42
    [](int value) { return value % 5 == 0; }
);
```
然而我们可能需要的是能够在运行期计算除数（divisor），即不能将5硬编码到lambda中。因此添加的过滤器逻辑将会是如下这样：
```cpp
void addDivisorFilter()
{
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();

    auto divisor = computeDivisor(calc1, calc2);

    filters.emplace_back(                               //危险！对divisor的引用
        [&](int value) { return value % divisor == 0; } //将会悬空！
    );
}
```
这个代码实现是一个定时炸弹。lambda`对局部变量divisor进行了引用`，但该变量的生命周期会在addDivisorFilter返回时结束，刚好就是在语句filters.emplace_back返回之后。因此添加到filters的函数添加完，该函数就死亡了。
使用这个过滤器（译者注：就是那个添加进filters的函数）会导致未定义行为，这是由它被创建那一刻起就决定了的。
现在，同样的问题也会出现在divisor的显式按引用捕获。
```cpp
filters.emplace_back(
    [&divisor](int value) 			    // 危险！对divisor的引用将会悬空！
    { return value % divisor == 0; }
);
```
但通过显式的捕获，能更容易看到lambda的可行性依赖于变量divisor的生命周期。
另外，写下“divisor”这个名字 能够 `提醒` 我们要注意确保divisor的生命周期至少跟lambda闭包一样长。比起“`[&]`”传达的意思，显式捕获能让人更容易想起“确保没有悬空变量”。

如果你知道一个闭包将会被马上使用（例如被传入到一个STL算法中）并且不会被拷贝，那么在它的lambda被创建的环境中，将不会有闭包的引用比父函数的局部变量和形参活得长的风险。
在这种情况下，你可能会争论说，没有悬空引用的危险，就不需要避免使用默认的引用捕获模式。
例如，我们的过滤lambda只会用做`C++11中std::all_of`的一个实参，返回满足条件的所有元素：
```cpp
template<typename C>
void workWithContainer(const C& container)
{
    auto calc1 = computeSomeValue1();               //同上
    auto calc2 = computeSomeValue2();               //同上
    auto divisor = computeDivisor(calc1, calc2);    //同上

    using ContElemT = typename C::value_type;       //容器内元素的类型
    using std::begin;                               //为了泛型，见条款13
    using std::end;

    if (std::all_of(                                //如果容器内所有值都为
            begin(container), end(container),       //除数的倍数
            [&](const ContElemT& value)
            { return value % divisor == 0; })
        ) {
        …                                           //它们...
    } else {
        …                                           //至少有一个不是的话...
    }
}
```
的确如此，这是安全的做法，但这种安全是不确定的。
如果发现lambda在其它上下文中很有用（例如 作为一个函数被添加在filters容器中），然后拷贝粘贴到一个 divisor变量已经死亡，但闭包生命周期还没结束的 上下文中，你又回到了悬空的使用上了。
同时，在该捕获语句中，也没有特别提醒了你注意分析divisor的生命周期。

从长期来看，`显式列出lambda依赖的局部变量和形参，是更加符合软件工程规范的做法`。

额外提一下，C++14支持了在lambda中使用auto来声明变量，上面的代码在C++14中可以进一步简化，ContElemT的别名可以去掉，if条件可以修改为：
```cpp
if (std::all_of(begin(container), end(container),
               [&](const auto& value)               // C++14
               { return value % divisor == 0; }))			
```
一个解决问题的方法是，divisor默认`按值捕获`进去，也就是说可以按照以下方式来添加lambda到filters：
```cpp
filters.emplace_back( 							    //现在divisor不会悬空了
    [=](int value) { return value % divisor == 0; }
);
```
这足以满足本实例的要求，但在通常情况下，按值捕获并不能完全解决悬空引用的问题。
这里的问题是：如果你`按值捕获`的是`一个指针`，你将该指针拷贝到lambda对应的闭包里，但这样并不能避免lambda外delete这个指针的行为，从而导致你的副本指针变成悬空指针。
也许你要抗议说：“这不可能发生。看过了第4章，我对智能指针的使用非常热衷。只有那些失败的C++98的程序员才会用裸指针和delete语句。”这也许是正确的，但却是不相关的，因为事实上你的确会使用裸指针，也的确存在被你delete的可能性。
只不过在现代的C++编程风格中，不容易在源代码中显露出来而已。
假设在一个Widget类，可以实现向过滤器的容器添加条目：
```cpp
class Widget {
public:
    …                       //构造函数等
    void addFilter() const; //向filters添加条目
private:
    int divisor;            //在Widget的过滤器使用
};
```
这是Widget::addFilter的定义：
```cpp
void Widget::addFilter() const
{
    filters.emplace_back(
        [=](int value) { return value % divisor == 0; }
    );
}	
```
这个做法看起来是安全的代码。lambda依赖于divisor，但默认的按值捕获确保divisor被拷贝进了lambda对应的所有闭包中，对吗？
    错误，完全错误。

**捕获 只能应用于 `lambda被创建时` 所在作用域里的 `non-static局部变量（包括形参）`**。
在`Widget::addFilter`的视线里，divisor并不是一个局部变量，而是Widget类的一个成员变量。它不能被捕获。
而如果默认捕获模式被删除，代码就不能编译了：
```cpp
void Widget::addFilter() const
{
    filters.emplace_back(                               //错误！
        [](int value) { return value % divisor == 0; }  //divisor不可用
    ); 
} 
```
另外，如果尝试去显式地捕获divisor变量（或者按引用或者按值——这不重要），也一样会编译失败，因为divisor不是一个局部变量或者`形参`。
```cpp
void Widget::addFilter() const
{
    filters.emplace_back(
        [divisor](int value)                //错误！没有名为divisor局部变量可捕获
        { return value % divisor == 0; }
    );
}
```
所以如果 默认按值捕获 不能捕获divisor，而不用默认按值捕获代码就不能编译，这是怎么一回事呢？

解释就是 这里`隐式使用了一个原始指针：this`。
每一个`non-static成员函数`都有一个this指针，每次你使用一个类内的数据成员时都会使用到这个指针。
例如，在任何Widget成员函数中，`编译器会在内部将divisor替换成this->divisor`。
在默认按值捕获的Widget::addFilter版本中，
```cpp
void Widget::addFilter() const
{
    filters.emplace_back(
        [=](int value) { return value % divisor == 0; }
    );
}
```
真正被捕获的是`Widget的this指针`，而不是divisor。
编译器会将上面的代码看成以下的写法：
```cpp
void Widget::addFilter() const
{
    auto currentObjectPtr = this;

    filters.emplace_back(
        [currentObjectPtr](int value)
        { return value % currentObjectPtr->divisor == 0; }
    );
}
```
明白了这个就相当于明白了lambda闭包的生命周期与Widget对象的关系，闭包内含有Widget的this指针的拷贝。
特别是考虑以下的代码，参考第4章的内容，只使用智能指针：
```cpp
using FilterContainer = 					//跟之前一样
    std::vector<std::function<bool(int)>>;

FilterContainer filters;                    //跟之前一样

void doSomeWork()
{
    auto pw =                               //创建Widget；std::make_unique
        std::make_unique<Widget>();         //见条款21

    pw->addFilter();                        //添加使用Widget::divisor的过滤器

    // …
}                                           //销毁Widget；filters现在持有悬空指针！
```
当调用doSomeWork时，就会创建一个过滤器，其生命周期依赖于由std::make_unique产生的Widget对象，即一个含有 `指向Widget的` 指针————Widget的this指针————的过滤器。
这个过滤器被添加到filters中，但当doSomeWork结束时，Widget会由管理它的std::unique_ptr来销毁（见Item18）。从这时起，filter会含有一个存着悬空指针的条目。

这个特定的问题可以通过给你想捕获的数据成员做一个局部副本，然后捕获这个副本去解决：
```cpp
void Widget::addFilter() const
{
    auto divisorCopy = divisor;                 //拷贝数据成员 这样就不隐含地拷贝this指针了

    filters.emplace_back(
        [divisorCopy](int value)                //捕获副本
        { return value % divisorCopy == 0; }	//使用副本
    );
}
```
事实上如果采用这种方法，默认的按值捕获也是可行的。
```cpp
void Widget::addFilter() const
{
    auto divisorCopy = divisor;                 //拷贝数据成员

    filters.emplace_back(
        [=](int value)                          //捕获副本
        { return value % divisorCopy == 0; }	//使用副本
    );
}
```
但为什么要冒险呢？当一开始你认为你捕获的是divisor的时候，默认捕获模式就是造成可能意外地捕获this的元凶。

在C++14中，一个更好的捕获成员变量的方式时使用通用的lambda捕获：
```cpp
void Widget::addFilter() const
{
    filters.emplace_back(                   //C++14：
        [divisor = divisor](int value)      //拷贝divisor到闭包
        { return value % divisor == 0; }	//使用这个副本
    );
}
```
这种通用的lambda捕获并没有默认的捕获模式，因此在C++14中，本条款的建议————避免使用默认捕获模式————仍然是成立的。

使用默认的按值捕获还有另外的一个缺点，它们预示了相关的闭包是独立的并且不受外部数据变化的影响。
一般来说，这是不对的。
lambda可能会依赖局部变量和形参（它们可能被捕获），还有静态存储生命周期（static storage duration）的对象。
这些对象定义在全局空间或者命名空间，或者在类、函数、文件中声明为static。
这些对象也能在lambda里使用，但它们不能被捕获。
但默认按值捕获可能会因此误导你，让你以为捕获了这些变量。
参考下面版本的addDivisorFilter函数：
```cpp
void addDivisorFilter()
{
    static auto calc1 = computeSomeValue1();    //现在是static
    static auto calc2 = computeSomeValue2();    //现在是static
    static auto divisor =                       //现在是static！！！
    computeDivisor(calc1, calc2);

    filters.emplace_back(
        [=](int value)                          //什么也没捕获到！
        { return value % divisor == 0; }        //引用上面的static，是不行的！！！
    );

    ++divisor;                                  //调整divisor
}
```
随意地看了这份代码的读者可能看到“`[=]`”，就会认为“好的，lambda拷贝了所有使用的对象，因此这是独立的”。但其实不独立。这个lambda没有使用任何的non-static局部变量，所以它没有捕获任何东西。
然而lambda的代码引用了static变量divisor，在每次调用addDivisorFilter的结尾，divisor都会递增，通过这个函数添加到filters的所有lambda都展示新的行为（分别对应新的divisor值）。
这个lambda是通过引用捕获divisor，这和默认的按值捕获表示的含义有着直接的矛盾。如果你一开始就避免使用默认的按值捕获模式，你就能解除代码的风险。

请记住：
* 默认的按引用捕获可能会导致悬空引用。
* 默认的按值捕获对于悬空指针很敏感（尤其是this指针），并且它会误导人产生lambda是独立的想法。





















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



# 条款37：使std::thread在所有路径最后都不可结合
每个`std::thread`对象处于两个状态之一：`可结合的（joinable）` 或者 `不可结合的（unjoinable）`。
`可结合状态`的`std::thread`对应于 `正在运行` 或者 `可能要运行的` 异步执行线程。
比如，对应于一个 `阻塞的（blocked）` 或者 `等待调度的` 线程的std::thread是可结合的，对应于`运行结束的`线程的`std::thread`也可以认为是可结合的。

`不可结合`的`std::thread`正如所期待：一个不是可结合状态的std::thread。
不可结合的std::thread对象包括：
* `默认构造`的std::threads。
    这种std::thread没有函数执行，因此没有对应到底层执行线程上。
* `已经被移动走`的std::thread对象。
    移动的结果就是一个std::thread原来对应的执行线程现在对应于另一个std::thread。
* `已经被join`的std::thread 。
    `在join之后`，std::thread不再对应于已经运行完了的执行线程。
* `已经被detach`的std::thread 。
    detach断开了std::thread对象与执行线程之间的连接。
（译者注：std::thread可以视作`状态保存的对象`，保存的状态可能也包括可调用对象，有没有具体的线程承载 就是 有没有连接）

std::thread的可结合性如此重要的原因之一就是 `当可结合的线程的析构函数被调用，程序执行会终止`。
比如，假定有一个函数doWork，使用一个过滤函数filter，一个最大值maxVal作为形参。
doWork检查是否满足计算所需的条件，然后 使用在0到maxVal之间的通过过滤器的所有值 进行计算。
如果进行过滤非常耗时，并且确定doWork条件是否满足也很耗时，则将两件事并发计算是很合理的。
我们希望为此采用`基于任务的`设计（参见Item35），但是假设我们希望 设置做过滤的线程的优先级。
Item35阐释了那需要线程的原生句柄，只能通过std::thread的API来完成；基于任务的API（比如future）做不到。
所以最终采用基于线程而不是基于任务。
我们可能写出以下代码：
代码如下：
```cpp
constexpr auto tenMillion = 10000000;           //constexpr见条款15

bool doWork(std::function<bool(int)> filter,    //返回计算是否执行；
            int maxVal = tenMillion)            //std::function见条款2
{
    std::vector<int> goodVals;                  //满足filter的值

    std::thread t([&filter, maxVal, &goodVals]  //填充goodVals
                  {
                      for (auto i = 0; i <= maxVal; ++i)
                          { if (filter(i)) goodVals.push_back(i); }
                  });

    auto nh = t.native_handle();                //使用t的原生句柄
    …                                           //来设置 t的优先级

    if (conditionsAreSatisfied()) {
        t.join();                               //等t完成
        performComputation(goodVals);
        return true;                            //执行了计算
    }
    return false;                               //未执行计算
}
```
在解释这份代码为什么有问题之前，我先把tenMillion的初始化值弄得更可读一些，这利用了C++14的能力，使用单引号作为数字分隔符：
```cpp
constexpr auto tenMillion = 10'000'000;         //C++14
```
还要指出，在开始运行之后 设置t的优先级就像把马放出去之后再关上马厩门一样（译者注：太晚了）。
更好的设计是 `在挂起状态时 开始t（这样可以在执行任何计算前调整优先级）`，但是我不想你为考虑那些代码而分心。
如果你对代码中忽略的部分感兴趣，可以转到Item39，那个Item告诉你如何以开始那些挂起状态的线程。

返回doWork。
如果`conditionsAreSatisfied()`返回true，没什么问题，但是如果返回false或者抛出异常，在doWork结束、调用t的析构函数时，std::thread对象t会是可结合的。这造成程序执行中止。

你可能会想，为什么std::thread析构的行为是这样的，那是因为另外两种显而易见的方式更糟：
* 隐式join 。
    这种情况下，`std::thread的析构函数`将等待其底层的异步执行线程完成。
    这听起来是合理的，但是可能会导致难以追踪的异常表现。比如，如果conditonAreStatisfied()已经返回了false，doWork继续等待过滤器应用于所有值就很违反直觉。

* 隐式detach 。
    这种情况下，std::thread析构函数会分离std::thread与其底层的线程。底层线程继续运行。听起来比join的方式好，但是可能导致更严重的调试问题。
    比如，在doWork中，goodVals是通过引用捕获的局部变量。它也被lambda修改（通过调用push_back）。
    假定，lambda异步执行时，conditionsAreSatisfied()返回false。这时，doWork返回，同时局部变量（包括goodVals）被销毁。栈被弹出，并在doWork的调用点继续执行线程。

调用点之后的语句有时会进行其他函数调用，并且至少一个这样的调用可能会占用曾经被doWork使用的栈位置（原先存在局部变量的栈位被使用，导致出错）。我们调用那么一个函数f。当f运行时，doWork启动的lambda仍在继续异步运行。该lambda可能在栈内存上调用push_back，该内存曾属于goodVals，但是现在是f的栈内存的某个位置。这意味着对f来说，内存被自动修改了！想象一下调试的时候“乐趣”吧。

标准委员会认为，销毁可结合的线程如此可怕以至于实际上禁止了它（`规定销毁可结合的线程 导致 程序终止`）。
这使你有责任确保使用std::thread对象时，在所有的路径上超出定义所在的作用域时都是不可结合的。
但是覆盖每条路径可能很复杂，可能包括自然执行通过作用域，或者通过return，continue，break，goto或异常跳出作用域，有太多可能的路径。
每当你想在执行跳至块之外的每条路径执行某种操作，最通用的方式就是将该操作放入局部对象的析构函数中。这些对象称为`RAII对象（RAII objects）`，从RAII类中实例化。（RAII全称为 “Resource Acquisition Is Initialization”（资源获得即初始化），尽管技术关键点在析构上而不是实例化上）。
RAII类在标准库中很常见。比如`STL容器（每个容器析构函数都销毁容器中的内容物并释放内存）`，`标准智能指针（Item18-20解释了，std::uniqu_ptr的析构函数调用他指向的对象的删除器，std::shared_ptr和std::weak_ptr的析构函数递减引用计数）`，`std::fstream对象（它们的析构函数关闭对应的文件）`等。
但是标准库没有std::thread的RAII类，可能是因为标准委员会拒绝将join和detach作为默认选项，不知道应该怎么样完成RAII。
幸运的是，完成自行实现的类并不难。
比如，下面的类实现允许调用者指定ThreadRAII对象（一个std::thread的RAII对象）析构时，调用join或者detach：
```cpp
class ThreadRAII {
public:
    enum class DtorAction { join, detach };     //enum class的信息见条款10
    
    ThreadRAII(std::thread&& t, DtorAction a)   //析构函数中对t实行a动作
    : action(a), t(std::move(t)) {}

    ~ThreadRAII()
    {                                           //可结合性测试见下
        if (t.joinable()) {
            if (action == DtorAction::join) {
                t.join();
            } else {
                t.detach();
            }
        }
    }

    std::thread& get() { return t; }            //见下

private:
    DtorAction action;
    std::thread t;
};
```
我希望这段代码是不言自明的，但是下面几点说明可能会有所帮助：

构造器只接受std::thread右值，因为我们想要把传来的std::thread对象`移动`进ThreadRAII。（std::thread不可以复制。）
构造器的形参顺序设计的符合调用者直觉（首先传递std::thread，然后选择析构执行的动作，这比反过来更合理），但是` 成员初始化列表 设计的匹配 成员声明的顺序 `。
将std::thread对象放在声明最后。
在这个类中，这个顺序没什么特别之处，但是通常，！！！可能`一个数据成员的初始化依赖于另一个，因为std::thread对象可能会在初始化结束后就立即执行函数了，所以在最后声明是一个好习惯`。
这样就能保证 **一旦构造结束，在前面的所有数据成员都初始化完毕，可以供std::thread数据成员绑定的异步运行的线程安全使用**。

ThreadRAII提供了get函数访问内部的std::thread对象。这类似于标准智能指针提供的get函数，可以`提供访问原始指针的入口`。
    提供get函数避免了ThreadRAII复制完整std::thread接口的需要，也意味着ThreadRAII可以在需要std::thread对象的上下文环境中使用。
在ThreadRAII析构函数调用std::thread对象t的成员函数之前，`检查t是否可结合`。这是必须的，因为在不可结合的std::thread上调用join或detach会导致未定义行为。
客户端可能会构造一个std::thread，然后用它构造一个ThreadRAII，使用get获取t，然后移动t，或者调用join或detach，每一个操作都使得t变为不可结合的。

如果你担心下面这段代码
```cpp
if (t.joinable()) { // 也许判断完了，其他线程又改变了其状态
    if (action == DtorAction::join) {
        t.join();
    } else {
        t.detach();
    }
}
```
存在竞争，因为在`t.joinable()`的执行和调用join或detach的中间，可能有其他线程改变了t为不可结合，你的直觉值得表扬，但是这个担心不必要。
只有调用成员函数才能使std::thread对象从可结合变为不可结合状态，比如join，detach或者移动操作。在ThreadRAII对象析构函数调用时，应当没有其他线程在那个对象上调用`成员函数`。
如果同时进行调用，那肯定是有竞争的，但是不在析构函数中，是`在客户端代码中试图同时在一个对象上调用两个成员函数（析构函数和其他函数）`。
通常，仅当所有都为const成员函数时，在一个对象同时调用多个成员函数才是安全的。
在doWork的例子上使用ThreadRAII的代码如下：
```cpp
bool doWork(std::function<bool(int)> filter,        //同之前一样
            int maxVal = tenMillion)
{
    std::vector<int> goodVals;                      //同之前一样

    ThreadRAII t(                                   //使用RAII对象
        std::thread([&filter, maxVal, &goodVals]
                    {
                        for (auto i = 0; i <= maxVal; ++i)
                            { if (filter(i)) goodVals.push_back(i); }
                    }),
                    ThreadRAII::DtorAction::join    //RAII动作
    );

    auto nh = t.get().native_handle();
    // …

    if (conditionsAreSatisfied()) {
        t.get().join();
        performComputation(goodVals);
        return true;
    }

    return false;
}
```
这种情况下，我们选择 在ThreadRAII的析构函数 对异步执行的线程进行join，因为在先前分析中，detach可能导致噩梦般的调试过程。
我们之前也分析了join可能会导致表现异常（坦率说，也可能调试困难），但是在未定义行为（detach导致），程序终止（使用原生std::thread导致），或者表现异常之间选择一个后果，可能表现异常是最好的那个。

哎，Item39表明了使用ThreadRAII来保证在`std::thread`的析构时执行join有时不仅可能`导致程序表现异常，还可能导致程序挂起`。
“适当”的解决方案是 此类程序应该和异步执行的lambda通信，告诉它不需要执行了，可以直接返回，但是C++11中不支持可中断线程（interruptible threads）。
可以自行实现，但是这不是本书讨论的主题。（关于这一点，Anthony Williams的《C++ Concurrency in Action》（Manning Publications，2012）的section 9.2中有详细讨论。）（译者注：此书中文版已出版，名为《C++并发编程实战》，且本文翻译时（2020）已有第二版出版。）
Item17说明因为ThreadRAII声明了一个析构函数，因此不会有编译器生成移动操作，但是没有理由ThreadRAII对象不能移动。
如果要求编译器生成这些函数，函数的功能也正确，所以显式声明来告诉编译器自动生成也是合适的：
```cpp
class ThreadRAII {
public:
    enum class DtorAction { join, detach };         //跟之前一样

    ThreadRAII(std::thread&& t, DtorAction a)       //跟之前一样
    : action(a), t(std::move(t)) {}

    ~ThreadRAII()
    {
        // …                                        //跟之前一样
    }

    ThreadRAII(ThreadRAII&&) = default;             //支持移动
    ThreadRAII& operator=(ThreadRAII&&) = default;

    std::thread& get() { return t; }                //跟之前一样

private: // as before
    DtorAction action;
    std::thread t;
};
```
请记住：
* 在所有路径上保证thread最终是不可结合的。
* 析构时join会导致难以调试的表现异常问题。
* 析构时detach会导致难以调试的未定义行为。
* 声明类数据成员时，最后声明std::thread对象。



# 条款39：对于一次性事件通信考虑使用void的futures
有时，`一个任务 通知 另一个异步执行的任务 发生了 特定的事件`很有用，因为第二个任务要等到这个事件发生之后才能继续执行。
事件也许是一个数据结构已经初始化，也许是计算阶段已经完成，或者检测到重要的传感器值。这种情况下，线程间通信的最佳方案是什么？
一个明显的方案就是使用`条件变量（condition variable，简称condvar）`。
如果我们将检测条件的任务称为`检测任务（detecting task）`，对条件作出反应的任务称为`反应任务（reacting task）`，
策略很简单：反应任务等待一个条件变量，检测任务在事件发生时改变条件变量。
代码如下：
```cpp
std::condition_variable cv;         //事件的条件变量
std::mutex m;                       //配合cv使用的mutex
```
`检测任务`中的代码不能再简单了：
```cpp
…                                   //检测事件
cv.notify_one();                    //通知反应任务
```
如果有多个反应任务需要被通知，使用`notify_all`代替`notify_one`，但是这里，我们假定只有一个反应任务需要通知。
反应任务的代码稍微复杂一点，因为在对条件变量调用wait之前，必须通过`std::unique_lock`对象锁住互斥锁。
（`在等待条件变量前 锁住互斥锁`对线程库来说是经典操作。
通过`std::unique_lock`锁住互斥锁的需要仅仅是C++11 API要求的一部分。）
`反应任务`概念上的代码如下：
```cpp
…                                       //反应的准备工作
{                                       //开启关键部分

    std::unique_lock<std::mutex> lk(m); //锁住互斥锁

    cv.wait(lk);                        //等待通知，但是这是错的！
    
    …                                   //对事件进行反应（m已经上锁）
}                                       //关闭关键部分；通过lk的析构函数解锁m
…                                       //继续反应动作（m现在未上锁）
```
这份代码的第一个问题就是有时被称为`代码异味（code smell）`的一种情况：即使代码正常工作，但是有些事情也不是很正确。
在这个情况中，这种问题源自于使用`互斥锁`。
互斥锁被用于保护共享数据的访问，但是可能检测任务和反应任务可能不会同时访问共享数据，比如说，检测任务会初始化一个全局数据结构，然后给反应任务用，如果检测任务在初始化之后不会再访问这个数据结构，而在检测任务表明数据结构准备完了之前反应任务不会访问这个数据结构，这两个任务在程序逻辑下互不干扰，也就没有必要使用互斥锁。但是条件变量的方法必须使用互斥锁，这就留下了令人不适的设计。

即使你忽略掉这个问题，还有两个问题需要注意：
* 如果 在反应任务wait之前检测任务通知了条件变量，反应任务会挂起。
    为了能使条件变量唤醒另一个任务，`任务必须等待在条件变量上`。如果在反应任务wait之前检测任务就通知了条件变量，反应任务就会丢失这次通知，永远不被唤醒。
* `wait语句虚假唤醒`。
线程API的存在一个事实（在许多语言中————不只是C++），等待一个条件变量的代码即使在条件变量没有被通知时，也可能被唤醒，这种唤醒被称为虚假唤醒（spurious wakeups）。
正确的代码 通过 `确认要等待的条件确实已经发生` 来处理这种情况，并`将这个操作作为唤醒后的第一个操作`。
C++条件变量的API使得这种问题很容易解决，因为允许把一个`测试要等待的条件的lambda（或者其他函数对象）`传给wait。
因此，可以将反应任务wait调用这样写：
```cpp
cv.wait(lk, 
        []{ return whether the evet has occurred; });
```
要利用这个能力 需要 反应任务 能够确定其等待的条件是否为真。
但是我们考虑的场景中，它正在等待的条件是 检测线程负责识别的事件的发生情况。
反应线程可能无法确定等待的事件是否已发生。这就是为什么它在等待一个条件变量！

在很多情况下，使用条件变量进行任务通信非常合适，但是也有不那么合适的情况。
对于很多开发者来说，他们的下一个诀窍是`共享的布尔型flag`。flag被初始化为false。当检测线程识别到发生的事件，将flag置位：
```cpp
std::atomic<bool> flag(false);          //共享的flag；std::atomic见条款40
…                                       //检测某个事件
flag = true;                            //告诉反应线程
```
就其本身而言，反应线程轮询该flag。
当发现flag被置位，它就知道等待的事件已经发生了：
```cpp
…                                       //准备作出反应
while (!flag);                          //等待事件
…                                       //对事件作出反应
```
这种方法不存在基于条件变量的设计的缺点。
不需要互斥锁，在反应任务开始轮询之前检测任务就对flag置位也不会出现问题，并且不会出现虚假唤醒。好，好，好。
不好的一点是反应任务中轮询的`开销`。
在任务等待flag被置位的时间里，任务基本被阻塞了，但是一直在运行。
这样，反应线程占用了可能能给另一个任务使用的硬件线程，每次启动或者完成它的时间片都增加了上下文切换的开销，并且保持核心一直在运行状态，否则的话本来可以停下来省电。
一个真正阻塞的任务不会发生上面的任何情况。这也是基于条件变量的优点，因为wait调用中的任务真的阻塞住了。
**将`条件变量`和`flag`的设计组合起来很常用**。
一个flag表示是否发生了感兴趣的事件，但是通过互斥锁同步了对该flag的访问。
因为互斥锁阻止并发访问该flag，所以如Item40所述，不需要将flag设置为`std::atomic`。
一个简单的bool类型就可以，检测任务代码如下：
```cpp
std::condition_variable cv;             //跟之前一样
std::mutex m;
bool flag(false);                       //不是std::atomic
…                                       //检测某个事件
{
    std::lock_guard<std::mutex> g(m);   //通过g的构造函数锁住m
    flag = true;                        //通知反应任务（第1部分）
}                                       //通过g的析构函数解锁m
cv.notify_one();                        //通知反应任务（第2部分）
```
反应任务代码如下:
```cpp
…                                       //准备作出反应
{                                       //跟之前一样
    std::unique_lock<std::mutex> lk(m); //跟之前一样
    cv.wait(lk, [] { return flag; });   //使用lambda来避免虚假唤醒
    …                                   //对事件作出反应（m被锁定）
}
…                                       //继续反应动作（m现在解锁）
```
这份代码解决了我们一直讨论的问题。
无论在检测线程对条件变量发出通知之前，反应线程是否调用了wait都可以工作，即使出现了虚假唤醒也可以工作，而且不需要轮询。
（如果条件先满足了，反应线程也不会无脑wait，不会错过；如果因为某种原因被唤醒而条件尚未满足，也会再一次判断条件是否满足，避免虚假唤醒）
（
PS: 为什么会虚假唤醒？
1. 由于线程调度不可预测，可能会有多个线程被同时唤醒，即便在只调用一次`pthread_cond_signal`的情况下。

2. 有些`硬件平台/操作系统`就是会在没有明确唤醒信号的情况下恢复线程。
    pthread 的条件变量等待 `pthread_cond_wait` 是使用 `阻塞的系统调用`实现的（比如 Linux 上的 `futex`），这些阻塞的系统调用在`进程被信号中断（如键盘输入、硬件中断、或者其他进程发来的软件信号）`后，通常会中止阻塞、直接返回 EINTR 错误。
    同样是阻塞系统调用，你从 read 拿到 EINTR 错误后可以直接决定重试，因为这通常不影响它本身的语义。
    而条件变量等待则不能，因为本线程拿到 EINTR 错误和重新调用 futex 等待之间，可能别的线程已经通过 pthread_cond_signal 或者 pthread_cond_broadcast发过通知了。 所以，虚假唤醒的一个可能性是条件变量的等待被信号中断。

    或者说 因为`操作系统的通知不可信`，自己再校验一次，如果是虚假唤醒就再wait一次。
3. 多个线程同时等待一个条件变量，信号的传递可能导致不止一个线程被唤醒。

4. 可能由于线程调度的原因，被条件变量唤醒的线程在本线程内真正执行「加锁并返回」前，另一个线程插了进来，完整地进行了一套「拿锁、改条件、还锁」的操作。使得条件又不满足了

）
但是仍然有些古怪(`实际上就是说 检测线程的通知不可靠，还得让反应线程再做检查`)，因为检测任务通过奇怪的方式与反应线程通信。
（译者注：下面的话挺绕的，可以参考原文）
检测任务通过通知条件变量告诉反应线程，等待的事件可能发生了，但是反应线程必须通过检查flag来确保事件发生了。
检测线程置位flag来告诉反应线程事件确实发生了，但是检测线程仍然还要先需要通知条件变量，以唤醒反应线程来检查flag。这种方案是可以工作的，但是`不太优雅`。

一个替代方案是：让`反应任务`通过在检测任务设置的`future`上`wait`来`避免使用条件变量，互斥锁和flag`。
这可能听起来也是个古怪的方案。毕竟，Item38中说明了future代表了从被调用方到（通常是异步的）调用方的通信信道的接收端，这里的检测任务和反应任务没有调用-被调用的关系。
然而，Item38中也说说明了发送端是个`std::promise`，接收端是个`future`的通信信道不是只能用在调用-被调用场景。
这样的通信信道可以用在任何你需要从程序一个地方传递信息到另一个地方的场景。这里，我们用来在检测任务和反应任务之间传递信息，传递的信息就是感兴趣的事件已经发生。
方案很简单。`检测任务`有一个`std::promise对象（即通信信道的写入端）`，`反应任务`有对应的`future`。
当检测任务看到事件已经发生，设置std::promise对象（即写入到通信信道）。同时，wait会阻塞住反应任务直到std::promise被设置。
现在，std::promise和futures（即std::future和std::shared_future）都是需要类型参数的模板。
形参表明通过通信信道被传递的信息的类型。
在这里，没有数据被传递，只需要让反应任务知道它的future已经被设置了。
我们在std::promise和future模板中需要的东西是：`表明 通信信道中 没有数据被传递的一个类型`。
这个类型就是`void`。
`检测任务`使用`std::promise<void>`，`反应任务`使用`std::future<void>`或者`std::shared_future<void>`。
当感兴趣的事件发生时，`检测任务`设置`std::promise<void>`，`反应任务`在`future`上`wait`。
尽管`反应任务 不从 检测任务 那里接收任何数据`，通信信道也可以让反应任务知道，检测任务什么时候已经通过对`std::promise<void>`调用`set_value`“写入”了void数据。
所以，有
```cpp
std::promise<void> p;                   //通信信道的promise
```
检测任务代码很简洁：
```cpp
…                                       //检测某个事件
p.set_value();                          //通知反应任务
```
反应任务代码也同样简单：
```cpp
…                                       //准备作出反应
p.get_future().wait();                  //等待对应于p的那个future
…                                       //对事件作出反应
```
像使用flag的方法一样，此设计`不需要互斥锁`，无论在反应线程调用wait之前检测线程是否设置了std::promise都可以工作，并且**不受`虚假唤醒`的影响（只有条件变量才容易受到此影响）**。
与基于条件变量的方法一样，反应任务在调用wait之后是真被阻塞住的，不会一直占用系统资源。是不是很完美？
当然不是，基于future的方法没有了上述问题，但是有其他新的问题。
比如，Item38中说明，`std::promise`和`future`之间有个`共享状态`，并且共享状态是`动态分配`的。因此你应该假定`此设计会产生 基于堆的 分配和释放开销`。
也许更重要的是，**`std::promise`只能`设置一次`**。
`std::promise`和`future`之间的通信是`一次性的`：不能重复使用。
这是与基于条件变量或者基于flag的设计的明显差异，条件变量和flag都可以通信多次。（条件变量可以被重复通知，flag也可以重复清除和设置。）
一次通信可能没有你想象中那么大的限制。
假定你想创建一个挂起的系统线程。就是，你想避免多次线程创建的那种经常开销，以便想要使用这个线程执行程序时，避免潜在的线程创建工作。或者你想创建一个挂起的线程，以便 在线程运行前 对其进行设置 这样的设置包括`优先级`或者`核心亲和性（core affinity）`。
C++并发API没有提供这种设置能力，但是std::thread提供了`native_handle成员函数`，它的结果就是提供给你对平台原始线程API的访问（通常是`POSIX`或者`Windows`的线程）。这些低层次的API使你可以对线程设置优先级和亲和性。
假设你仅仅想要`对某线程挂起一次（在创建后，运行线程函数前）`，使用void的future就是一个可行方案。
这是这个技术的关键点：
```cpp
std::promise<void> p;
void react();                           //反应任务的函数
void detect()                           //检测任务的函数
{
    std::thread t([]                    //创建线程
                  {
                      p.get_future().wait();    //挂起t直到future被置位
                      react();
                  });
    …                                   //这里，t在 调用react前 挂起

    p.set_value();                      //解除挂起t（因此调用react）

    …                                   //做其他工作
    
    t.join();                           //使t不可结合（见条款37）
}
```
因为 所有离开detect的路径中t都要是不可结合的，所以使用类似于Item37中ThreadRAII的RAII类很明智。
代码如下：
```cpp
void detect()
{
    ThreadRAII tr(                      //使用RAII对象
        std::thread([]
                    {
                        p.get_future().wait();
                        react();
                    }),
        ThreadRAII::DtorAction::join    //有危险！（见下）
    );
    …                                   //tr中的线程在这里被挂起    ？？？
    p.set_value();                      //解除挂起tr中的线程
    …
}
```
这样看起来安全多了。
问题在于第一个“…”区域中（注释了“tr中的线程在这里被挂起”的那句），**如果`异常`发生，p上的`set_value`永远不会调用，这意味着lambda中的`wait`永远不会返回**。
那意味着在lambda中运行的线程不会结束，这是个问题，因为RAII对象tr在析构函数中被设置为在（tr中创建的）那个线程上实行join。
换句话说，如果在第一个“…”区域中发生了异常，函数挂起，因为`tr的析构函数永远无法完成`。
？？？有很多方案解决这个问题，但是我把这个经验留给读者。（一个开始研究这个问题的好地方是我的博客The View From Aristeia中，2013年12月24日的文章“ThreadRAII + Thread Suspension = Trouble?”）

这里，我只想展示如何扩展原始代码（即不使用RAII类）使其挂起然后取消挂起不仅一个反应任务，而是`多个任务`。
简单概括，关键就是在react的代码中使用`std::shared_future`代替`std::future`。
一旦你知道`std::future`的`share成员函数`将`共享状态所有权`转移到share产生的`std::shared_future`中，代码自然就写出来了。
唯一需要注意的是，每个反应线程都需要自己的`std::shared_future`副本，该副本引用共享状态，因此通过share获得的shared_future要被在反应线程中运行的`lambda按值捕获`：
```cpp
std::promise<void> p;                   //跟之前一样
void detect()                           //现在针对多个反映线程
{
    auto sf = p.get_future().share();   //sf的类型是std::shared_future<void>
    std::vector<std::thread> vt;        //反应线程容器
    for (int i = 0; i < threadsToRun; ++i) {
        vt.emplace_back(
                        [sf] {
                                sf.wait();    //在sf的局部副本上wait；
                                react();
                            }
                        );  //emplace_back见条款42
    }
    …                                   //如果这个“…”抛出异常，detect挂起！
    p.set_value();                      //所有线程解除挂起
    …
    for (auto& t : vt) {                //使所有线程不可结合；
        t.join();                       //“auto&”见条款2
    }
}
```
使用future的设计可以实现这个功能值得注意，这也是你应该考虑将其应用于一次通信的原因。

请记住：
* 对于简单的事件通信，基于条件变量的设计需要一个`多余的互斥锁`，对检测和反应任务的相对进度有约束，并且需要`反应任务`来`验证事件是否已发生`。
* 基于`flag`的设计避免的上一条的问题，但是是基于`轮询`，而不是阻塞。
* `条件变量`和`flag`可以组合使用，但是产生的通信机制很不自然。
* 使用std::promise和future的方案避开了这些问题，但是这个方法使用了堆内存存储共享状态，同时有只能使用一次通信的限制。






































