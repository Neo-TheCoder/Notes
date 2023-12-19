# 空指针调用成员函数
## 正常调用
### 非静态函数
（指针类型为该类）如果没有使用this指针，是可以正常调用的
**为什么可以呢？**
这样调用的时候，有一个隐含的参数是this指针，而类的普通成员函数的地址在程序开始时已经是一个固定的地址了，只有访问到类的非静态成员才会使用this指针
（在编译期生成代码时发生了静态绑定，编译器根据指针的类型，找到了对应的成员函数地址并进行绑定）

### 静态函数
自然是可以

## 程序崩溃
### 这个成员函数调用类的其他成员

### 调用类的虚函数



# main函数的三个参数
```cpp
int main(int argc, char*argv[], char*envp[])
{ 
	return 0;
}
```
## 三个参数的含义
1. 记录（传入给程序的）参数个数
2. 字符指针数组，指向每个参数
3. 字符指针数组，指向环境变量里的内容：例如："SHELL=bin/bash"


# 自定义new和delete函数需要注意什么？
内存泄漏



# c++程序堆内存跟栈内存的扩展方向是什么样的？
C++程序中的内存分为堆内存和栈内存。堆内存用于动态分配内存，而栈内存用于存储局部变量和函数调用的上下文信息。
在C++程序中，栈内存的扩展方向是**向下**，也就是从高地址到低地址。当函数被调用时，栈会分配一块内存用于存储局部变量和函数调用的上下文信息，随着函数的返回，栈会释放这些内存。
而堆内存的扩展方向是**向上**，也就是从低地址到高地址。堆内存用于动态分配内存，通过使用new和delete或者malloc和free等操作符来进行内存的分配和释放。
内存的扩展方向可能会受到操作系统和编译器的影响，因此具体的实现可能会有所不同。但一般情况下，栈内存的扩展方向是向下，堆内存的扩展方向是向上。


# std::shared_ptr的第二个模板参数
是删除器，是一个**可调用对象**：用于在shared_ptr的引用计数变为0时释放所管理的资源
如果不指定删除器，则默认使用delete操作符来释放资源。
```cpp
std::shared_ptr<int> p(new int[10], [](int* p){delete [] p;});      //lambda
std::shared_ptr<int> p1(new int[10], std::default_delete<int[]>()); //指定默认删除器
```


# 虚假唤醒
***本不应该被唤醒线程被唤醒了，导致程序执行结果错误。***
## 可能的原因
1. 多个线程在等待同一个条件变量时，其中一个线程被误唤醒，导致其他线程也被唤醒。
2. 条件变量在等待之前没有正确的设置条件，导致其他线程没有满足条件就被唤醒。
3. 在条件变量的wait函数返回之前，其他线程已经修改了满足条件的变量，导致当前线程在检查条件时发现条件已经被满足了。
4. 唤醒信号丢失：当一个线程在等待条件变量时，另一个线程发出唤醒信号，但该信号可能会被操作系统丢失。
5. 操作系统信号处理：当线程被操作系统中断处理程序唤醒时，可能会发生虚假唤醒。

 多线程编程中，常见情况：让多个线程等待在一个条件上，等到这个条件成立的时候我们再去唤醒这些线程，让它们接着往下执行代码的场景。
 假如某一时刻条件成立，所有的线程都被唤醒了，然后去竞争锁，因为**同一时刻只会有一个线程能拿到锁**，其他的线程都会阻塞到获取锁的动作无法往下执行，等到成功争抢到锁的线程消费完条件，释放了锁，后面的线程继续运行，拿到锁时这个条件很可能已经不满足了，这个时候线程应该继续在这个条件上阻塞下去，而不应该继续执行，如果继续执行了，就说发生了虚假唤醒。

## 直观的解决办法是：
使用while循环重新检查等待条件。当线程在等待条件变量时，需要通过while循环来重新检查是否达到等待条件，条件不满足则再次wait。这样可以防止线程在没有满足条件的情况下被误唤醒。
while循环的精髓：**当线程被唤醒后，再次检查条件是否满足，如果不满足，则继续等待。**
```cpp
std::unique_lock<std::mutex> lock(mtx);
if(condition == false)
	cv.wait(lock);

	func();	// 执行唤醒后应该执行的逻辑

// 考虑两个线程都是这段代码（主线程中执行condition变为true的逻辑）
// 主线程
    // 唤醒等待的线程
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
        cv.notify_one();
    }
// 当条件满足时，两线程本都是因为wait而阻塞，然后线程1、线程2都被唤醒（执行下面的代码逻辑），但实际上应该只有一个线程被唤醒并往下执行
```



# 关于std::function
思考一下为什么可以包裹函数和lambda表达式？包裹的对象储存在哪里？

## 简介
`std::function`是一个可变参类模板，是一个通用的函数包装器（Polymorphic function wrapper）。
`std::function`的实例可以存储、复制和调用任何可复制构造的**可调用目标**：包括普通函数、成员函数、类对象（重载了`operator()`的类的对象）、Lambda表达式等。是对C++现有的可调用实体的一种**类型安全**的包裹（相比而言，函数指针这种可调用实体，是类型不安全的）。
`std::function`中存储的可调用对象被称之为`std::function`的目标。若`std::function`中不含目标，调用不含目标的std::function会抛出`std::bad_function_call`异常。


## 使用
```cpp


```


## 源码分析
```cpp
template<typename _Res, typename... _ArgTypes>
class function<_Res(_ArgTypes...)>
  : public _Maybe_unary_or_binary_function<_Res, _ArgTypes...>
  , private _Function_base
{
private:
    using _Invoker_type = _Res (*)(const _Any_data&, _ArgTypes&&...);
    _Invoker_type _M_invoker;
// ... ...
// 还有很多成员函数
};

// 重载了operator()
template<typename _Res, typename... _ArgTypes>
_Res function<_Res(_ArgTypes...)>::operator()(_ArgTypes... __args) const
{
  if (_M_empty())
      __throw_bad_function_call();
  return _M_invoker(_M_functor, std::forward<_ArgTypes>(__args)...);    // 调用存储的可调用对象
  // _M_functor的类型为_Any_data
}

```
模板参数进行了偏特化，`_Res`为可调用目标的返回值类型，可调用目标的入参为`_ArgTypes`









# 关于lambda表达式





# 关于std::forward的使用，以SamplePtr为例
```cpp
// 专用于sample数据的自定义智能指针
template <typename T, typename... Args>
SamplePtr<T> make_sample_ptr(Args&&... args)
{
    return SamplePtr<T>(new T(std::forward<Args>(args)...));
}
```
此处使用了`std::forward`对入参进行完美转发（以保持参数的原始值类别和引用类型），
`make_sample_ptr`函数的入参传递方式为右值引用，


## 关于左值和右值
简单理解：出现在等号左边的值是左值
出现在等号右边的值是右值
**左值可以用`&`取地址，而右值不行。**
临时变量、字面值（1.23、100）也属于右值。
```cpp
Test t = T(); // T()就是临时变量，是右值，t属于左值

const string &s = "asd";  // "asd"是字符串字面量，是不可被更改的左值，不具名，但是可以取地址，因而是左值
```
**纯右值**：
1. 返回非引用类型的表达式
`x+1、x++`
2. 除了字符串字面量之外的字面量
`42、true`

**将亡值**：
1. 隐式或显式调用函数的结果，该函数的返回类型是：对所返回对象类型的右值引用
2. 对对象类型右值引用的转换
```cpp
static_cast<int&&>(7);
std::move(7);
```

3. 类成员访问表达式，指定非引用类型的非静态数据成员，其中对象表达式是xvalue


## 关于右值引用
**&&不一定是代表右值引用**

### 左值引用和右值引用
**左值引用**：可以绑定到左值，某些情况下可以绑定到右值：
  只有**常左值引用**可以绑定到右值，非常左值引用不能绑定到右值。

**右值引用**：
  只能绑定到右值

**万能引用（转发引用）**：
用&&声明，可能是左值引用，可能是右值引用

### 万能引用出现场合
如果一个变量或者参数被声明为`T&&`，**其中`T`是被推导的类型**，那这个变量或者参数就是一个**universal reference**。
万能引用必须形如`T&&`

实际情况中，几乎所有的万能引用都是**函数模板的参数**。因为auto声明的变量的类型推导规则本质上和模板是一样的，使用`auto`也可能得到一个万能引用。
PS：使用`typedef`和`decltype`时，也可能出现万能引用。

和所有的引用一样，你必须对universal references进行初始化，**正是其initializer决定了它到底代表的是lvalue reference 还是 rvalue reference**:
如果用来初始化universal reference的表达式是一个**左值**，那么universal reference就变成**lvalue reference**。
如果用来初始化universal reference的表达式是一个**右值**，那么universal reference就变成**rvalue reference**。


## 生命周期延长
一个变量的生命周期在超出作用域时结束。
临时对象生命周期的规则是：一个临时对象 会在包含这个临时对象的完整表达式估值完成后、按生成顺序的逆序被销毁，除非有生命周期延长发生。


### 临时对象生命周期的延长
如果一个**prvalue**（对xvalue无效）被绑定到一个引用上，它的生命周期则会延长到跟这个引用变量一样长。
如果纯右值在绑定到引用之前，就变成了将亡值，则生命周期不会延长。可以理解为，将亡值不会延长生命周期。

#### 应用
可以把没有（基类）虚析构的子类对象绑定到基类的引用变量上
（绑定到派生类的引用和指向派生类的指针的区别在于：引用是别名，本身是不占据内存（先不考虑底层实现），因此不需要像指针那样`delete`，而是起到了延长生命周期的作用，到该析构的时候，自然是派生类析构了）


## 区分万能引用
```cpp
Widget&& var1 = someWidget;
auto&& var2 = var1; // 等价于：Widget& var2 = var1;

```
注意：`var1`虽然是右值引用，但是左值。
而`var2`由于类型声明为`auto&&`，所以是一个万能引用，并且因为被一个左值var1初始化了，所以`var2`变成了一个左值引用。

```cpp
std::vector<int> v;
auto&& val = v[0];
/*
  v[0]，等价于std::vector<int>::operator[]，该函数返回vectot元素的左值引用（左值引用必然是左值），那么万能引用val也就被初始化为左值引用
*/
```

```cpp
template<typename T>
void f(T&& param);  // “&&” might mean rvalue reference

// 调用：
f(10);  // 10是右值
// param被字面值10（右值）初始化，则param变成右值引用，即int&&类型

int x = 10;
f(x); // param被左值初始化，param变成左值引用，即int&类型
```
在发生**类型推导**的时候，`&&`才代表 **universal reference**。
如果没有**类型推导**，就没有universal reference。这种时候，类型声明当中的`&&`总是代表着rvalue reference。

```cpp
template<typename T>
void f(T&& param);               // deduced parameter type ⇒ type deduction;
                                 // && ≡ universal reference
 
template<typename T>
class Widget {
    ...
    Widget(Widget&& rhs);        // fully specified parameter type ⇒ no type deduction;
    ...                          // && ≡ rvalue reference
};
 
template<typename T1>
class Gadget {
    ...
    template<typename T2>
    Gadget(T2&& rhs); // deduced parameter type ⇒ type deduction;
    ... // && ≡ universal reference
};
 
void f(Widget&& param); // fully specified parameter type ⇒ no type deduction;
// && ≡ rvalue reference
```
简单理解：如果你看到T&& (其中T是模板参数)，那这里就有类型推导，那T&&就是universal reference。如果你看到 “&&” 跟在一个具体的类型名后面 (e.g., Widget&&)，那它就是个rvalue reference。

```cpp
template<typename T>
void f(std::vector<T>&& param); // “&&” means rvalue reference
```
！！！由于此处的参数不是`T&&`的形式，而是`std::vector<T>&&`，这就导致param只是一个普通的右值引用，而不是万能引用。
```cpp
template<typename T>
void f(const T&& param);  // “&&” means rvalue reference
// 也不是万能引用
```

```cpp
template <class T, class Allocator = allocator<T> >
class vector {
public:
    ...
    void push_back(T&& x);       // fully specified parameter type ⇒ no type deduction;
    ...                          // && ≡ rvalue reference
};
```
为什么此处没有发生类型推导呢？
这就要追溯到std::vector<T>::push_back的声明：
```cpp
template <class T>
void vector<T>::push_back(T&& x);
// 可见，和之前说的T&&的经典场景有所不同，因为vector<T>的T一旦确定，push_back的参数类型就完全确定了
```

```cpp
template <class T, class Allocator = allocator<T> >
class vector {
public:
    ...
    template <class... Args>
    void emplace_back(Args&&... args); // deduced parameter types ⇒ type deduction;
    ...                                // && ≡ universal references
};
```
`emplace_back`和`push_back`的情况不同：
**它的每一个参数的类型都需要推导**。
函数的模板参数Args和类的模板参数T无关。
例如：std::vector<Widget>，还是没法知道`emplace_back`的入参类型

```cpp
template<class... Args>
void std::vector<Widget>::emplace_back(Args&&... args); // 仍然需要推导
```


### 表达式的左右值性质与类型无关
**！！！**
**值类别（value category）**：用于区分左值、右值
**值类型（value type）**：区分`reference type`，表明一个变量是实际数值，还是引用另外一个数值。
在C++里，所有的原生类型、枚举、结构、联合、类都代表值类型，只有引用（&）和指针（*）才是引用类型。

一个表达式的lvalueness (左值性)或者 rvalueness (右值性)和它的类型无关。

来看下 int：可以有lvalue的int (e.g., 声明为int的变量)，还有rvalue的int (e.g., 字面值10)。用户定义类型Widget等等也是一样的。

一个Widget对象可以是lvalue(e.g., a Widget 变量) 或者是rvalue (e.g., 创建Widget的工程函数的返回值)。

表达式的类型不会告诉你它到底是个lvalue还是rvalue。因为表达式的 lvalueness 或 rvalueness 独立于它的类型，我们就可以有一个 lvalue，但它的类型确是 rvalue reference，也可以有一个 rvalue reference 类型的 rvalue :
```cpp
Widget makeWidget();
// factory function for Widget
 
Widget&& var1 = makeWidget();
// var1 is an lvalue, but
// its type is rvalue reference (to Widget)
//  var1的类别是左值，但它的类型是右值引用

Widget var2 = static_cast<Widget&&>(var1);
// the cast expression yields an rvalue, but
// its type is rvalue reference  (to Widget)
// 右侧表达式是右值，但它的类型是右值引用
```
var1类别是左值，但它的类型是右值引用。static_cast<Widget&&>(var1)表达式是个右值，但它的类型是右值引用。

把 lvalues (例如 var1) 转换成 rvalues 比较常规的方式是对它们调用std::move，所以 var2 可以像这样定义:
```cpp
Widget var2 = std::move(var1);             // equivalent to above
```
我最初的代码里使用 static_cast 仅仅是为了显示的说明这个表达式的类型是个rvalue reference (Widget&&)。rvalue reference 类型的具名变量和参数是 lvalues。(你可以对他们取地址。)

```cpp
template<typename T>
class Widget {
    ...
    Widget(Widget&& rhs);        // rhs’s type is rvalue reference,
    ...                          // but rhs itself is an lvalue
};
 
template<typename T1>
class Gadget {
    ...
    template <typename T2>
    Gadget(T2&& rhs);            // rhs is a universal reference whose type will
    ...                          // eventually become an rvalue reference or
};                               // an lvalue reference, but rhs itself is an lvalue
```

在 Widget 的构造函数当中, rhs 是一个rvalue reference，前面提到，**右值引用只能被绑定到右值上**，所以我们知道它被绑定到了一个rvalue上面(i.e., 因此我们需要传递了一个rvalue给它)。
但是 rhs 本身是一个 lvalue，所以，当我们想要用到这个被绑定在 rhs 上的rvalue 的 rvalueness 的时候，我们就需要把 rhs 转换回一个rvalue。之所以我们想要这么做，是因为我们想将它作为一个移动操作的source，这就是为什么我们用`std::move`将它转换回一个 rvalue。

类似地，Gadget 构造函数当中的rhs 是一个 universal reference，所以它可能绑定到一个 lvalue 或者 rvalue 上，但是**无论它被绑定到什么东西上，rhs 本身还是一个 lvalue**。
如果它被绑定到一个 rvalue 并且我们想利用这个rvalue 的 rvalueness， 我们就要重新将 rhs 转换回一个rvalue。如果它被绑定到一个lvalue上，当然我们就不想把它当做 rvalue。
**一个绑定到universal reference上的对象可能具有 lvalueness 或者 rvalueness，正是因为有这种二义性，所以催生了`std::forward`**: 如果一个本身是 lvalue 的 universal reference 如果绑定在了一个 rvalue 上面，就把它重新转换为rvalue。
函数的名字 (“forward”) 的意思就是，我们希望在传递参数的时候，可以保存参数原来的lvalueness 或 rvalueness，即是说把参数**转发**给另一个函数。


### 引用折叠和完美转发
为了说明一个万能引用是如何在经过类型推导和引用折叠后，变成一个左值引用的
实际上，universal reference 其实只是一个身处于引用折叠背景下的rvalue reference。

#### 引用折叠本质细节
c++禁止引用的引用。
在对一个universal reference的模板参数进行类型推导时候，同一个类型的 lvalues 和 rvalues 被推导为稍微有些不同的类型。
具体来说，类型T的lvalues被推导为T&(i.e., lvalue reference to T)。
而类型T的 rvalues 被推导为 T。
(注意，虽然 lvalue 会被推导为lvalue reference，但 rvalues 却不会被推导为 rvalue references!) 
我们来看下分别用rvalue和lvalue来调用一个接受universal reference的模板函数时会发生什么:
```cpp
template<typename T>
void f(T&& param);
 
int x;
 
f(10);  // invoke f on rvalue
// T被推导为int

f(x); // invoke f on lvalue
// T被推导为int&，那么代码就变成：
void f(int& && param);

// 编译器实际上处理为：
void f(int& param);
```
为了避免编译器对此处的代码报错，C++提供了**引用折叠**的设定。

**引用折叠可能的情况**
1. lvalue reference to lvalue reference
2. lvalue reference to rvalue reference
3. rvalue reference to lvalue reference
4. rvalue reference to rvalue reference

**引用折叠两条规则**
1. 一个 rvalue reference to an rvalue reference 会变成 (“折叠为”) 一个 rvalue reference.
2. 所有其他种类的"引用的引用" (i.e., 组合当中含有lvalue reference) 都会折叠为 lvalue reference.

```cpp
// ！！！当一个变量本身的类型是引用类型，会忽略类型中所带的引用
int x;
 
int&& r1 = 10;  // r1’s type is int&&
 
int& r2 = x;  // r2’s type is int&

```
调用`f`时，`r1`和`r2`的类型都被认为是`int`，为什么呢？
```cpp
template<typename T>
void f(T &&param) {
    static_assert(std::is_lvalue_reference<T>::value, "T& is lvalue reference");
    cout << "T& is lvalue reference" << endl;
}

int main()
{
    int x;
    int &&r1 = 10;
    int &r2 = x;
    f(r1);
    f(r2);
}
```
会发现，r1和r2的类型都被推导为`int&`。
因为都是左值，所以对万能引用参数进行类型推导时，得到的类型都是`int&`。

引用折叠只发生在“像是**模板实例化**这样的场景当中”。 
声明auto变量是另一个这样的场景。
**推导一个universal reference的 auto 变量的类型**，在本质上和**推导universal reference的函数模板参数是一样的**，所以类型T的lvalue被推导为T&，类型T 的rvalue被推导为T。
```cpp
Widget&& var1 = someWidget;      // var1 is of type Widget&& (no use of auto here)
 
auto&& var2 = var1;              // var2 is of type Widget& (see below)
var1 的类型是 Widget&&，但是它的 reference-ness 在推导 var2 类型的时候被忽略了;var1 这时候就被当做 Widget。
```
因为它是个lvalue，所以初始化一个universal reference(var2)的时候，var1 的类型就被推导成Widget&。
在 var2 的定义当中将 auto 替换成Widget& 会生成下面的非法代码:
```cpp
Widget& && var2 = var1;          // note reference-to-reference
```
而在引用折叠之后，就变成了:
```cpp
Widget& var2 = var1;             // var2 is of type Widget&
```



## 把智能指针std::move会怎么样？







# 关于通信的接收方缓存大小的问题
GetNewSamples（ConvertToIdl）如果执行得太慢，就会有数据没有来得及放入缓存中，也就是被丢弃了，这是必然的，因为缓存不可能无限制增大（而且也必然是一开始就确定的一个大小），造成数据的堆积。


## VECTOR代码中
**size()返回实际作为payload的元素个数，capacity()返回实际申请的内存空间大小**

(错误的调用)
reactor_cache_是一个`StaticList<std::unique_ptr<IpcSampleCacheEntry<SampleType>>>`类型的变量
### 调用`GetSamples`：
将sample指针从reactor cache `move`到app cache，并提供对cache的访问
该函数返回对`SampleCacheContainer`的引用，它应该被用于获取和移除cache中处理过的samples
`GetSamples`的使用者不允许在修改返回的引用的同时调用`GetSamples`。

调用该函数后，`SampleCacheContainer`可能有更少的、相同的、更多的元素
更少：相比调用`GetSamples`时请求的可用元素更少了
相等：至少还有请求时那么多的元素
更多：在上一次`GetSamples`中，部分samples被处理了，并且当前调用请求更少的samples（比起上一次调用时未被处理的samples）

#### 关于实现细节：

（正确的调用）
reactor_cache_是一个`StaticList<std::unique_ptr<SomeIpSampleCacheEntry>`类型的变量

先对`reactor_cache_.size()`和`app_cache.size()`求和，计算`total_cache_size`，
然后是一个for循环：
  `InvisibleSampleCache`中，有一个`std::size_t`类型的成员`capacity_`，记录invisible cache存储的events的最大数量。
  在for遍历的过程中，不断增加`drop_index`（初始化为`capacity`）
  PS：对于超出buffer capacity的samples直接丢弃
  直到：drop_index == total_cache_size
  过程中，app_cache_不断`pop_front()`，所以其实是app_cache_有total_cache_size个元素出队列了（即便pop的次数超出了实际的长度，也没关系，对此会什么都不做）
  （app_cache_总是有至少samples的充分的数量）

接着，计算`cleaned_app_cache_size`（初始化为`app_cache_.size()`）。
`available_samples_count`由`reactor_cache_.size()`和`cleaned_app_cache_size`求和得来。
`samples_to_return`取自`requested_sample_count`（**入参**）和`available_samples_count`较小值。

然后又是一个for循环：
  遍历`samples_to_return - cleaned_app_cache_size`次，
  过程中，
  ```cpp
  app_cache_.push_back(std::move(reactor_cache_.size()));
  reactor_cache_.pop_front();
  将samples移动到application cache
  （application cache可能仍然储存上一次GetSamples调用的samples，只需从reactor_cache中移动差值即可达到请求的样本数）
  ```
循环的起始位置是应用程序缓存中已经存在的样本数量，因为这些样本不需要从reactor缓存中获取。
循环的终止条件是达到请求的样本数量或者reactor缓存中的样本已经全部移动到应用程序缓存中。循环的每一次迭代都将reactor缓存中的第一个样本移动到应用程序缓存的末尾，并从reactor缓存中删除该样本。

