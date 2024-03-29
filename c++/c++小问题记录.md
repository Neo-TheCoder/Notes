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
 假如某一时刻条件成立，所有的线程都被唤醒了，然后去竞争锁，因为**同一时刻只会有一个线程能拿到锁**，其他的线程都会阻塞到获取锁的动作无法往下执行。
 等到成功争抢到锁的线程消费完条件，释放了锁，后面的线程继续运行，拿到锁时这个条件很可能已经不满足了，这个时候线程应该继续在这个条件上阻塞下去，而不应该继续执行（！！！也就是要再检查一次），如果继续执行了，就说发生了虚假唤醒。

## 直观的解决办法是：
使用`while循环`重新检查等待条件。当线程在等待条件变量时，需要通过while循环来重新检查是否达到等待条件，条件不满足则再次wait。这样可以防止线程在没有满足条件的情况下被误唤醒。
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















# C++函数调用的本质
函数调用背后是栈帧（***stack frame***）的建立（与操作系统平台以及编译器的实现有关）

**前言**
**EBP：基址指针寄存器(extended base pointer)**，其内存放着一个指针（帧指针），该指针永远指向系统栈最上面一个栈帧的底部。所以ebp指向的是栈的**栈底**的数据。
**ESP：栈指针寄存器(extended stack pointer)**，其内存放着一个指针（栈指针），该指针永远指向系统栈最上面一个栈帧的栈顶。所以esp指向的是栈的**栈顶**的数据。



ESP就是一直指向栈顶的指针,而EBP只是存取**某时刻的栈顶**指针,以方便对栈的操作,如获取函数参数、局部变量等。


以函数为例：
```cpp
 int foo(int arg1, int arg2, int arg3); // foo函数用到3个4字节的局部变量
```
当`main`调用到`foo`的时候，实际是这样的：
`ESP寄存器`被`foo函数`用来指示**栈顶**的位置，`EBP寄存器`存储**基准指针**

从 main 传递到 foo 的参数以及 foo 本身的局部变量都可以以基准指针 EBP 为参考，加上偏移量找到。
由于被调用者允许使用 `EAX、ECX 和 EDX 寄存器`，所以如果调用者希望保存这些寄存器的值，就必须在调用子函数之前显式地把它们保存在栈中。

如果除了上面提到的几个寄存器，被调用者还想使用别的寄存器，比如 `EBX、ESI 和 EDI`，那么，被调用者就必须在栈中保存这些被额外使用的寄存器，并在调用返回前恢复它们。
也就是说，如果被调用者只使用约定的 `EAX、ECX 和 EDX 寄存器`，它们由调用者负责保存并恢复，但**如果被调用者还额外使用了别的寄存器，则必须由它们自己保存并回复这些寄存器的值**。

传递给 foo 的参数被压到栈中，最后一个参数先进栈，所以第一个参数是位于栈顶的。

foo 中声明的**局部变量**以及函数执行过程中需要用到的一些**临时变量**也都存在**栈**中。 
**小于等于 4 个字节**的返回值会被保存到 `EAX`。
如果**大于 4 字节，小于 8 字节**，那么 `EDX` 也会被用来保存返回值。
如果返回值占用的空间还要大，那么调用者会向被调用者传递一个额外的参数，这个额外的参数指向将要保存返回值的地址。
用 C/C++ 语言来描述这个过程就是函数调用：
`x = foo(a, b, c);`
由于 x 的内容占用的空间过大被转化为：
`foo(&x, a, b, c);`
注意，这仅仅在返回值占用**大于 8 个字节**时才发生。
有的编译器不用 EDX 保存返回值，所以当返回值大于 4 个字节时，就用这种转换。 
当然，并不是所有函数调用都直接赋值给一个变量，还可能是直接参与到某个表达式的计算中，例如：
`m = foo(a, b, c) + foo(d, e, f);`
或者作为另外一个函数的参数， 例如：
`bar(foo(a, b, c), 3);`
这些情况下，foo 的返回值会被保存在一个临时变量中参加后续的运算。


## 函数调用前，调用者的动作
在函数调用前，`main`正在用 `ESP` 和 `EBP`寄存器指示它自己的栈帧。
首先，`main` 把 `EAX、ECX 和 EDX` 压栈。这是一个可选的步骤，如果这三个寄存器即将被用到，但当前存储的内容需要保存下来以备将来恢复（恢复现场），则执行此步骤。

接着，main 把传递给 foo 的参数一一进栈，最后的参数最先进栈。例如，假设我们的函数调用是：
`a = foo(12, 15, 18);`
```
push dword 18
push dword 15
push dword 12
```
然后main调用call指令，来调用子函数`foo`
`call foo`

当 `call` 指令执行的时候，`EIP 指令指针寄存器`的内容被压入栈中。因为 **EIP 寄存器是指向 main 中的下一条指令**，所以现在返回地址就在栈顶了（整个函数调用完成后，栈顶继最终回到起始位置）。
在 `call` 指令执行完之后，下一个执行周期将从名为 `foo` 的标记处开始。

当函数 foo，也就是被调用者取得程序的控制权，它必须做 3 件事：
1. **建立它自己的栈帧，为局部变量分配空间**（如果需要，保存 EBX、ESI 和 EDI 等寄存器的值）。
EBP（extended base pointer）寄存器现在**正指向 main 的栈帧中的某个位置**，这个值必须被保留，因此，EBP 进栈保存当前值；
然后 ESP 的内容赋值给了 EBP，这使得函数的参数可以通过对 EBP 附加一个**偏移量**得到，而栈寄存器 ESP 便可以空出来做其他事情。
如此一来，几乎所有的 C/C++ 函数调用都从如下两个指令开始：
```
push ebp
mov ebp,esp
```
（栈顶在上，栈底在下，栈底是高地址）
            +-------------------------+
ESP=EBP --> main使用的EBP（栈底）的内容
            +-------------------------+
            返回地址
            +-------------------------+
            实际参数#1 = 12 [EBP + 8]
            +-------------------------+
            实际参数#2 = 15 [EBP + 12]
            +-------------------------+
            实际参数#3 = 18 [EBP + 16]
            +-------------------------+
            调用者保存的寄存器（EAX、ECX、EDX）现场（如有需要）
            +-------------------------+


下一步，foo 必须为它的**局部变量**分配空间，同时，也必须为它可能用到的一些**临时变量**分配空间。

例如，foo 中的一些 C/C++ 语句可能包括复杂的表达式，其子表达式的**中间值**就必须得有地方存放，因为它们可能被下一个复杂表达式所复用。
为说明方便，我们假设：
foo 中有两个 int 类型（每个占 4 字节）的局部变量，另外还需要额外的 12 字节作为临时存储空间。简单地把栈指针减去 20 便是为这 20 个字节分配了空间：
`sub esp, 20`

            +-------------------------+
ESP     --> 被调用者保存的寄存器（EBX、ESI、EDI）现场（如有需要）
            临时空间    [EBP -20]
            +-------------------------+
            局部变量#2  [EBP - 8]
            +-------------------------+
            局部变量#1  [EBP - 4]
            +-------------------------+
EBP     --> main使用的EBP（栈底）的内容
            +-------------------------+
            返回地址
            +-------------------------+
            实际参数#1 = 12 [EBP + 8]
            +-------------------------+
            实际参数#2 = 15 [EBP + 12]
            +-------------------------+
            实际参数#3 = 18 [EBP + 16]
            +-------------------------+
            调用者保存的寄存器（EAX、ECX、EDX）现场（如有需要）
            +-------------------------+

foo 的函数体现在就可以执行了：
这其中也许有进栈、出栈的动作，**栈指针 ESP 也会上下移动，但 EBP 是保持不变的**。
这意味着我们可以一直用 [EBP+8] 找到第一个参数（实参位置不变），而不管在函数中有多少进出栈的动作。 函数 foo 的执行也许还会调用别的函数，甚至递归地调用 foo 本身。
然而，只要 EBP 寄存器在这些子调用返回时被恢复，就可以继续用 EBP 加上偏移量的方式访问实际参数、局部变量和临时存储空间。


## 被调用者返回前的动作
在把程序控制权返还给调用者前，被调用者 foo 必须先把**返回值保存在 EAX 寄存器中**。
当**返回值占用多于 4 个或 8 个字节**时，接收返回值的变量地址会作为一个额外的**指针参数**被传到函数中，而函数本身就不需要返回值了。
这种情况下，被调用者直接通过内存拷贝把返回值直接拷贝到接收地址，从而省去了一次通过栈的中转拷贝。
其次，foo 必须恢复 EBX、ESI 和 EDI 寄存器的值。如果这些寄存器被修改，正如我们前面所说，我们会在 foo 执行开始时把它们的原始值压入栈中（！！！用栈帧的一块内存，来拷贝这些寄存器的值）。
如果 ESP 寄存器指向正确位置，寄存器的原始值就可以出栈并恢复。
可见，在 foo 函数的执行过程中正确地跟踪 ESP 是多么的重要——也就是说，**进栈和出栈操作的次数必须保持平衡**。
这两步之后，我们不再需要 foo 的局部变量和临时存储空间了，我们可以通过下面的指令销毁当前栈帧：
```
mov esp, ebp    # ESP存储栈顶位置，被赋值为EBP的内容，栈帧的栈顶就是由ESP界定的
pop ebp

// 如果是i386指令集，可使用以下等价的指令
leave
ret
```

## 调用者在返回后的动作
在程序控制权返回到调用者（也就是例子中的`main函数`）后，传递给 foo 的参数通常已经不需要了。
我们可以把这 3 个参数一起弹出栈，这可以通过把栈指针加 12（3 个 4 字节）实现：
```
add esp, 12
```
如果在函数调用前，保存过 EAX、ECX 和 EDX 寄存器的值，调用者 main 函数现在可以把它们弹出以恢复它们当时的值。这个动作之后，栈顶就回到了开始整个函数调用过程前的位置





# 关于lambda表达式






# 关于std::forward的使用
参考链接：
***https://zhuanlan.zhihu.com/p/99524127***

以SamplePtr为例
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
最初的代码里使用 static_cast 仅仅是为了显示的说明这个表达式的类型是个rvalue reference (Widget&&)。rvalue reference 类型的具名变量和参数是 lvalues。(你可以对他们取地址。)

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
```
var1 的类型是 Widget&&，但是它的 reference-ness 在推导 var2 类型的时候被忽略了;var1 这时候就被当做 Widget。

因为它是个lvalue，所以初始化一个universal reference(var2)的时候，var1 的类型就被推导成Widget&。
在 var2 的定义当中将 auto 替换成Widget& 会生成下面的非法代码:
```cpp
Widget& && var2 = var1;          // note reference-to-reference
```
而在引用折叠之后，就变成了:
```cpp
Widget& var2 = var1;             // var2 is of type Widget&
```



#### 引用折叠之场景三
在形成和使用`typedef`时
```cpp
template<typename T>
class Widget {
    typedef T& LvalueRefType;
    ...
};
int main() {
    Widget<int&> w; // 根据引用折叠规则，这样子调用Widget，T会被推导为int&，那么在typedef中，LvalueRefType就是T的引用，即左值引用的左值引用，折叠为左值引用
}
```


#### `decltype`
decltype对一个具名的、非引用类型的变量，会推导为类型T (i.e., 一个非引用类型)。
在相同条件下，模板和auto却会推导出T&。
decltype 进行类型推导只依赖于 decltype 的表达式; 用来对变量进行初始化的表达式的类型(如果有的话)会被忽略。
```cpp
Widget w1, w2;
auto&& v1 = w1;   // v1是左值引用（因为w1是左值，是具名对象）
decltype(w1)&& v2 = w2;    // decltype表达式推导结果为Widget，因此w2很明确地是右值引用，没法绑定到左值
```

```cpp
template <typename T>
foo(T &&)
```
**T类型的推导和形参的类型有关**
如果传递过去的参数是左值，T 的推导结果是左值引用，那`T&&`的结果仍然是左值引用--即 T& && 坍缩成了T&。
如果传递过去的参数是右值，T 的推导结果是参数的类型本身。那 T&& 的结果自然就是一个右值引用。


```cpp
#include<iostream>

class shape{
public:
    shape() = default;
};

void foo(const shape&)
{
	puts("foo(const shape&)");
}
void foo(shape&&)
{
	puts("foo(shape&&)");
}
void bar(const shape& s)
{
	puts("bar(const shape&)");
	foo(s);
}
void bar(shape&& s)
{
	puts("bar(shape&&)");
	foo(s);
}
int main()
{
	bar(shape());
}
```

结果：
bar(shape&&)
foo\(const shape&）

bar中传入的是右值，调用bar的&&重载函数，但是对于void bar(shape&& s)来说，s本身是一个lvalue，所以在foo(s)后，仍旧调用的是&重载函数。


```cpp
void foo(const shape&)
{
	puts("foo(const shape&)");
}
void foo(shape&&)
{
	puts("foo(shape&&)");
}
template <typename T>
void bar(T&& s)
{
	foo(std::forward<T>(s));  /// 完美转发的意义是：如果s绑定在了右值上，就重新转为右值，如果不完美转发的话，s是右值引用，但是本身是左值，就调用左值的函数去了
}
int main() {
    circle temp;
    bar(temp);
    bar(circle());
}
```


### `std::move()`与`std::forward()`源码剖析
```cpp
// 移除引用
template<typename _Tp>
struct remove_reference
{ typedef _Tp   type; };
// 目的是去除引T中的用部分，只获取类型部分

// 特化版本
template<typename _Tp>
struct remove_reference<_Tp&>
{ typedef _Tp   type; };    // 原理是特化一个左值引用版本，如果是左值引用类型，就会匹配到这里，然后通过typedef使得type取得纯粹的类型

template<typename _Tp>
struct remove_reference<_Tp&&>
{ typedef _Tp   type; };
```


```cpp
// std::forward 转发左值
template<typename _Tp>
constexpr _Tp&&
forward(typename std::remove_reference<_Tp>::type& __t) noexcept
{ return static_cast<_Tp&&>(__t); }
// 先取得纯粹类型type，然后定义一个type类型的左值引用类型的左值变量__t，再对这个变量强转为右值引用类型，！！！发生引用折叠，如果_Tp是左值引用类型，则折叠为_Tp&，最后返回一个左值引用类型，如果_Tp被推导为右值引用类型，则返回右值引用类型
```

```cpp
// std::forward 转发右值
template<typename _Tp>
constexpr _Tp&&
forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
{
  static_assert(!std::is_lvalue_reference<_Tp>::value, "template argument"
        " substituting _Tp is an lvalue reference type");
  return static_cast<_Tp&&>(__t);
}
/*
    当不是左值类型的时候，才继续往下执行
*/
```

```cpp
// std::move
template<typename _Tp>  constexpr typename std::remove_reference<_Tp>::type&&  move(_Tp&& __t) noexcept  
{ 
    return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); 
}
```
传递的是左值，推导`_Tp`为左值引用，仍旧`static_cast`转换为右值引用。
传递的是右值，推导`_Tp`为右值引用，仍旧`static_cast`转换为右值引用。


## 总结
在《Effective Modern C++》中建议：对于**右值引用**使用`std::move`，对于**万能引用**使用`std::forward`。
`std::move()`与`std::forward()`都仅仅做了类型转换而已。**真正的移动操作**是在**移动构造函数**或者**移动赋值操作符**中发生的。
`std::move()`可以应用于左值(普通的变量int这些使用move与不使用move效果一样)，但这么做要谨慎。因为一旦**移动了左值**，就**表示当前的值不再需要了**，如果后续使用了该值，产生的行为是未定义。



### 不要返回本地变量的引用

这会导致C++编程错误，是在函数里返回一个本地对象的引用。由于在函数结束时本地对象即被销毁，返回一个指向本地对象的引用属于未定义行为。

在C++11之前，返回一个本地对象意味着这个对象会被拷贝，除非编译器发现可以做**返回值优化（named return value optimization，或 NRVO），能把对象直接构造到调用者的栈上**（重点是少了一次拷贝）。从 C++11 开始，返回值优化仍可以发生，但在没有返回值优化的情况下，编译器将试图把本地对象移动出去，而不是拷贝出去。这一行为不需要程序员手工用 std::move 进行干预——使用std::move 对于移动行为没有帮助，反而会影响返回值优化。


### 总结
1. 在类型声明当中， “&&” 要不就是一个 rvalue reference ，要不就是一个 universal reference ———— 一种可以解析为lvalue reference或者rvalue reference的引用。对于某个被推导的类型T，universal references 总是以 T&& 的形式出现。

2. 引用折叠是 会让 **universal references** (其实就是一个处于引用折叠背景下的rvalue references ) 有时解析为 lvalue references 有时解析为 rvalue references 的**根本机制**。
引用折叠只会在一些特定的可能会产生"引用的引用"场景下生效。 
这些场景包括：模板类型推导，auto 类型推导， typedef 的形成和使用，以及decltype 表达式。

3. `std::move`与`std::forward`本质都是`static_cast`转换，对于右值引用使用`std::move`，对于万能引用使用`std::forward`。
`std::move`解决的问题是对于一个本身是左值的右值引用变量需要绑定到一个右值上，所以需要使用一个能够传递右值的工具，而std::move就干了这个事。
`而std::forward`解决的问题是一个绑定到`universal reference`上的对象可能具有 lvalueness 或者 rvalueness，正是因为有这种二义性，所以催生了std::forward: 如果一个本身是 **左值** 的 万能引用如果绑定在了一个 **右值** 上面，就把它重新转换为右值。
函数的名字 (“forward”) 的意思就是L我们希望在传递参数的时候，可以保存参数原来的lvalueness 或 rvalueness，即是说把参数转发给另一个函数。

4. 移动语义使得在 C++ 里返回大对象（如容器）的函数和运算符成为现实，因 而可以提高代码的简洁性和可读性，提高程序员的生产率。




## 把智能指针std::move会怎么样？
当使用 `std::move` 来移动智能指针时，实际上是将指针的所有权从一个对象转移到另一个对象。智能指针通常用于管理动态分配的资源，例如堆上的对象，通过使用引用计数或其他机制来确保资源的正确释放。

使用 `std::move` 移动智能指针的效果是将源指针的所有权转移给目标指针，同时将源指针置为空指针。这样做的目的是避免在转移所有权后，源指针继续管理资源，从而导致资源的重复释放或访问错误。

以下是一个示例代码，展示了如何使用 `std::move` 移动智能指针：

```cpp
std::unique_ptr<int> sourcePtr = std::make_unique<int>(42);
std::unique_ptr<int> destPtr = std::move(sourcePtr);

// 现在 sourcePtr 不再拥有资源，而 destPtr 拥有资源
```

在上述代码中，通过 `std::move` 将 `sourcePtr` 的所有权转移到 `destPtr`，这意味着 `sourcePtr` 不再拥有资源，而 `destPtr` 现在拥有资源。移动后，`sourcePtr` 被置为空指针。

需要注意的是，移动智能指针并不会移动资源本身，而是移动资源的所有权。这意味着，移动后的智能指针和移动前的智能指针指向同一个资源，但是只有移动后的智能指针可以管理和释放该资源。








# 关于通信的接收方缓存大小的问题
`GetNewSamples`（ConvertToIdl）如果执行得太慢，就会有数据没有来得及放入缓存中，也就是被丢弃了，这是必然的，因为缓存不可能无限制增大（而且也必然是一开始就确定的一个大小），造成数据的堆积。


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


# `unique_ptr`作为函数参数时，应该以`值`还是`右值引用`类型传递？
情况无非以下几种：
```cpp
1. By value
  callee(unique_ptr<Widget> smart_w)

2. By non-const l-value reference
  callee(unique_ptr<Widget> &smart_w)

3. By const l-value reference
  callee(const unique_ptr<Widget> &smart_w)

4. By r-value reference
  callee(unique_ptr<Widget> &&smart_w)

// 裸指针调用
callee(Widget *w)
callee(Widget &w)
callee(const Widget *w)
callee(const Widget &w)
// 虽然引用和指针本质上一样，但是使用时有区别：w是否可以是mullptr，引用可没法绑定到空指针上
// const类型的指针，意味着w这个裸指针不能修改其内部数据，只允许调用Widget的只读方法
```


## By value: callee(unique_ptr<Widget> smart_w)
caller（来调用callee），必然要产生一个copy，传给callee，让它使用。
即从对象上看，caller有一个对象，callee也有一个对象，是**两个对象**，但是callee对象是从caller复制而来。
caller和callee分别管理自己对象的生命周期lifetime。
callee管理起来比较简单，因为函数参数总是在其堆栈上，所以是自动管理的，函数退出即销毁。
如果Copy by value for smart pointer，也是两个对象，即两个smart pointer对象。
但smart pointer会很特别：
smart pointer里面有一个**内部指针**，指向一个要用的对象（一般在heap上）。即caller和callee，虽然都有一个自己的smart pointer对象（重点在于，毕竟还是个指针，不是本体），但它们可能通过内部指针，指向某一个共有的真正要用到的对象。
对于`unique pointer`，这个内部指针所指向的对象，就是实际的Widget对象。
对于`shared pointer`，这个内部指针所指向的对象，并不是实际的对象(Widget indirectly)，它包括两样东西：一个共享计数（或者准确说：两个共享计数，但常规理解，只考虑其中的唯一strong counter）和一个真正的Widget对象指针。


## `unique_ptr`由于其特殊性，不可以copy
```cpp
void callee(unique_ptr<Widget> smart_w)
{
  // ... use smart_w
}

void caller()
{
  unique_ptr<Widget> smart_w = std::make_unique<Widget>();

  callee(smart_w);    // 编译报错，unique pointer不能直接copy

  // 必须调用std::move
  callee(std::move(smart_w));

}
```
`std::move`表示这个对象的资源（对于智能指针，资源就是`Widget对象`）理应被转移走了，

```cpp
void caller()
{
  callee(std::move(smart_w));

  smart_w->method();    // 这个很可能导致Segment Fault，是绝对错误的，因为smart_w已经无效了(Widget不见了)，指针对象被转移到callee中去了，生命周期也改变了，其内部指针所指向的Widget，转给了callee。caller的smart_w，其内部指针不再指向Widget了。
}
```
注意：caller的smart_w，其内部指针不再指向Widget了。
我们也称之为sink。即Widget这个资源，从caller()，下降到callee()里;或者说，caller里的smart_w，已经转移Widget的ownership到callee里的smart_w。（之所以移动了指针对象，酒把指针指向的对象的所有权都转移了，是因为移动构造函数）

`callee(const unique_ptr<Widget> smart_w)`是一个特别变种，本质和`unique_ptr<Widget> smart_w`差别不大，只是callee()申明smart_w不会变（但仍可以调用里面Widget的write动作）。类似`foo(int *p)`和`foo(int * const p)`的道理（注意：不是foo(const int *p)）。


## By non-const l-value reference: callee(unique_ptr<Widget> &smart_w)

### l-value reference的意义
和raw pointer指针的意义是一样的: caller和callee都指向同一对象。
从代码角度，我们为什么要用reference，几个理由：
1. 避免上面的copy by value，因为copy可能是个很大的动作（cost is big），即callee()里的第二个对象（即参数）的建造constrution是很消耗资源的（CPU时间或大的内存分配）。
2. 我们应该在callee()里修改这个对象，然后callee返回后，caller()可以看到这个改过的效果。
当然，在callee()里，程序员有权可以不修改这个对象，如果是绝对不修改，我们不应该用non const，我们需要加上const（可以参考下面的By const l-value reference）。不管如何，这里既然没有加上const，也就是说：我们在callee()有意向要修改，至少有一行修改的代码，即使它可能出现在if等conditional语句下。

#### 用在unique pointer这个对象上，又是何意义
修改smart_w又是什么含义？正常来说，应该是**换掉里面的Widget**（因为unique ptr内容就一个指针，指向Widget）。比如：smart_w不再指向当前的Widget对象，而是另外一个Widget对象（或者清空为nullptr也可以，anyway，我们必须修改之）。

但是实际生产应用中，上面的逻辑几乎看不到。通过上面的分析，我们发现实践中，这样做的可能性不大。我们为什么要换掉一个unique pointer里指向的Widget呢？
所以，结论：By non-const l-value reference理论上可以用，但几乎看不到这样的实际案例。如果我是code viewer，如果有程序员这样用了l-value reference for unique ptr，第一时间，我会怀疑他/她用错了，然后仔细检查callee()的逻辑代码。


## By const l-value reference: callee(const unique_ptr<Widget> &smart_w)
unique pointer禁止copy，所以应该不用考虑copy的开销，那为什么不用原始指针呢？
```cpp
void caller()
{
  unique_ptr<Widget> smart_w = std::make_unique<Widget>();
  callee(smart_w);
}

void callee(const unique_ptr<Widget> &smart_w)
{ // callee不会修改smart_w对象
  // use smart_w
}

// 这样调用
void caller()
{
  unique_ptr<Widget> smart_w = std::make_unique<Widget>();
  callee(smart_w.get());
}

void callee(Widget *w)
{
  // use w
}

```
你可能会说，unique pointer安全呀，我们用smart pointer就是为了保证资源不会泄漏。而raw pointer做不到资源安全呀。
**如果callee的raw pointer保证来自caller的smart pointer，它也保证资源是安全的**。
```cpp
void caller()
{
  unique_ptr<Widget> smart_w = std::make_unique<Widget>();

  callee1(smart_w.get());

  callee2(smart_w.get());

  // 可以继续用smart_w，也可以随便抛出异常
}   // guarantee: 当caller退出后，smart_w的析构函数会销毁Widget资源，不管发生任何异常 --> 而不造成资源泄露：因为抛出异常而没能释放资源

void callee1(Widget *w)
{
  // use w，同时，可以随便抛出异常
}

void callee2(Widget *w)
{
  // use w，同时，可以随便抛出异常
}
```
而从代码规范的角度来看，callee里`delete`的话，确实会有问题，但这种代码不应该通过code review，拿到裸指针，没有理由delete对象，delete对象的责任在caller。

```
Don’t pass a smart pointer as a function parameter unless you want to use or manipulate the smart pointer itself, such as to share or transfer ownership.  Prefer passing objects by value, *, or &, not by smart pointer.
```



## By r-value reference: callee(unique_ptr<Widget> &&smart_w)
对于callee，如果参数是右值引用，我们应该在callee里拿走这个对象的资源（即Widget），同时，我们应该将参数的资源，设置为空（一般是nullptr）。
当然，callee()里面可以不做这些动作（C++程序员的上帝自由），但在caller()角度，这非常让人疑惑，callee你到底是拿，还是不拿？或者说，为了调用callee()，caller必须用std::move()，callee你就算不拿，caller也不敢继续用了。

### 对于unique pointer右值引用和上面的By value的对比
`unique pointer`是个non-copiable object，请参考上面的By value说明，unique pointer不能直接copy，必须`std::move()`下才可以，即**unique pointer的copy内部就是右值引用**。
所以，用copy by value和右值引用，本质上，没有什么区分，用哪个都可以。但理论上，对于unique pointer，右值引用还是和copy by value有稍微不同。
针对unique pointer，特定情况下，**右值引用对于编译器而言，效率要略高于copy by value**，但带来的麻烦是，代码中，我们必须不停使用`perfect forwarding`。
总之可以认为：**对于unique pointer，右值引用和Copy by value几乎等效**。



# `std::ref`和`std::reference_wrapper`
`std::reference_wrapper`用于包装一个对象的引用，常用于STL容器中存储引用类型的对象（STL容器要求元素必须是可复制的）
`std::ref`是一个函数模板，接收一个对象，返回一个`std::reference_wrapper`对象。相比`std::reference_wrapper`，`std::ref`可以直接将一个对象作为参数传递给函数，而不需要事先创建一个`std::reference_wrapper`对象。

`std::bind`可以将函数调用的参数延迟绑定，以便在以后的调用中使用。

```cpp
#include <iostream>
#include <functional>
#include <vector>

void foo(int& x) {
    x++;
}

int main() {
    int x = 1;
    std::reference_wrapper<int> r1(x);  // r1是x的引用
    std::cout << r1 << std::endl; // 输出1  
    r1.get() = 2;  // 通过get()取得绑定的引用，而不能直接赋值
    std::cout << x << std::endl; // 输出2

    std::vector<std::reference_wrapper<int>> v;
    v.push_back(x); // int类型，会创建临时对象，绑定到x上
    v.push_back(r1);  // reference_wrapper<int>类型
    for (auto& i : v) {
        i.get()++;  // 两个引用对象，实质都是绑定x，也就是加两次
    }
    std::cout << x << std::endl; // 输出4

    std::function<void()> f = std::bind(foo, std::ref(x));
    f();  // std::bind创建函数对象，无出入参，将函数foo和f绑定在一起，当f被调用时，调用函数foo，并且将std::ref(x)作为参数传递给它
    std::cout << x << std::endl; // 输出5

    return 0;
}
```
注意，以上代码中的`std::bind(foo, x)`的调用，和`foo(x)`是不一样的。换言之，传入`std::ref(x)`是有必要的，要不然会做隐式的拷贝。



# 类模板的派生
```cpp
// 基类模板
template <typename T>
class BaseTemplate {
public:
    BaseTemplate(T value) : data(value) {}
    void print() {
        std::cout << "BaseTemplate: " << data << std::endl;
    }
protected:
    T data;
};

// 派生类模板
template <typename T>
class DerivedTemplate : public BaseTemplate<T> {
public:
    DerivedTemplate(T value) : BaseTemplate<T>(value) {}
    void printDerived() {
        std::cout << "DerivedTemplate: " << this->data << std::endl; // 可以访问基类的成员变量
    }
};

int main() {
    DerivedTemplate<int> derived(42);
    derived.print(); // 调用基类的成员函数
    derived.printDerived(); // 调用派生类的成员函数
    return 0;
}
```



# CRTP(Curiously Recurring Template Pattern)
在C++中，派生类可以继承基类的模板，其中模板参数是派生类本身(奇异递归模板模式)
1. 静态多态性：通过CRTP，可以在编译时实现静态多态性。派生类可以通过继承基类模板来获得基类的行为，并在编译时进行类型检查。
2. 提供编译时代码生成：通过CRTP，可以在派生类中生成特定于派生类的代码。基类模板可以使用派生类的类型信息来生成特定的代码，从而提高代码的效率和灵活性。
3. 实现静态接口：通过CRTP，可以在派生类中实现静态接口。基类模板可以定义一组接口函数，并在派生类中实现这些函数。这样可以在编译时进行接口检查，确保派生类正确地实现了基类的接口。

```cpp
// 举例
#include <iostream>
using namespace std;

template <typename Child>
struct Base
{
	void interface()
	{
		static_cast<Child*>(this)->implementation();  // 转为派生类指针，调用派生类方法
	} // 使用static_cast是因为只有继承了基类的类型才能调用interface()，且是向下转型，是安全的
};

struct Derived : Base<Derived>
{
	void implementation()
	{
		cerr << "Derived implementation\n";
	}
};

int main()
{
	Derived d;  // 调用派生类对象的基类提供的方法
	d.interface();  // Prints "Derived implementation"

	return 0;
}
```


```cpp
template<typename Child>
class Animal
{
public:
	void Run()
	{
		static_cast<Child*>(this)->Run();
	}
};

class Dog :public Animal<Dog>
{
public:
	void Run()
	{
		cout << "Dog Run" << endl;
	}
};

class Cat :public Animal<Cat>
{
public:
	void Run()
	{
		cout << "Cat Run" << endl;
	}
};

template<typename T>
void Action(Animal<T> &animal)
{
	animal.Run();
}

int main()
{
	Dog dog;
	Action(dog);  // 该函数具有多态性，对于不同的对象，调用不同的方法

	Cat cat;
	Action(cat);
	return 0;
}
```

# 类的引用成员的初始化 必须在初始化列表
在C++中，类的引用类型成员必须在初始化列表中进行初始化，这是因为引用类型成员在创建时必须引用一个已经存在的对象。
**引用类型成员不能被重新赋值，一旦初始化后，它将一直引用同一个对象**。
初始化列表提供了一种在对象构造时初始化成员的机制，它**在构造函数的函数体执行之前执行**。
通过在初始化列表中初始化引用类型成员，可以**确保在构造函数的函数体中使用这些成员时，它们已经被正确地初始化**。
另外，引用类型成员的初始化不能延迟到构造函数的函数体中，因为**引用类型成员没有默认构造函数**。
如果不在初始化列表中初始化引用类型成员，编译器将会报错。
总结来说，引用类型成员必须在初始化列表中初始化，以确保它们引用一个已经存在的对象，并且避免编译错误。




# 关于锁
## `std::lock_guard`
在构造时自动锁定互斥量，在析构时自动释放互斥量

## `std::unique_lock`
提供了更多的灵活性，可以手动锁定和释放互斥量，并且可以在不同时间点上锁定和释放互斥量。
还提供了额外的功能，如延迟锁定、所有权转移等。



# 三五法则
C++中的"三五法则"（Rule of Three/Five）是指：
## 1. Rule of Three（三法则）：
   - 如果一个类需要手动实现`析构函数`、`拷贝构造函数`或`拷贝赋值运算符`中的一个，通常需要手动实现另外两个，以确保资源的正确释放和对象的正确拷贝。
   - 这是因为如果一个类需要手动管理资源（如动态分配内存），则需要确保在`拷贝`、`赋值`或`销毁对象`时资源能够正确地被管理。

## 2. Rule of Five（五法则）：
   - 随着`移动语义`的引入，Rule of Five指的是如果一个类需要手动实现`析构函数`、`拷贝构造函数`、`拷贝赋值运算符`、`移动构造函数`或`移动赋值运算符`中的一个，通常需要手动实现另外四个，以确保资源的正确管理和对象的高效移动。
   - 移动语义可以提高对象的性能，避免不必要的资源拷贝，因此在实现类时需要考虑移动语义的支持。

## 3. Rule of Zero（零规则）：
   - 在C++11之后，引入了智能指针和RAII等现代C++特性，使得在很多情况下可以避免手动管理资源。
   - Rule of Zero指的是尽量使用`现代C++特性（如智能指针、RAII）`来管理资源，避免手动实现析构函数、拷贝构造函数、拷贝赋值运算符、移动构造函数和移动赋值运算符，从而避免出现资源管理的错误和提高代码的可维护性。

总之，"三五零规则"是在C++中设计和实现类时的一些最佳实践，可以帮助开发者正确地管理资源、避免内存泄漏和提高代码的性能和可维护性。



# 深拷贝（deep copy）和浅拷贝（shallow copy）
## 1. 浅拷贝：
浅拷贝是指将一个对象的值复制到另一个对象，包括对象中的基本数据类型和指针。当进行浅拷贝时，只是简单地复制指针的值，而不是复制指针指向的内容。这意味着如果原始对象中有指针指向堆内存中的数据，浅拷贝后的对象和原始对象将共享同一块内存，当其中一个对象释放内存时，另一个对象可能会出现指针悬空的情况。

## 2. 深拷贝：
深拷贝是指在拷贝对象时，不仅复制对象本身的值，还会**复制对象指向的动态内存**。这样，即使原始对象和拷贝对象指向的是不同的内存地址，它们之间是相互独立的，互不影响。深拷贝通常需要**自定义拷贝构造函数和赋值运算符重载函数**来确保指针指向的内存被正确复制。

总的来说，浅拷贝只是简单地复制对象的值，而**深拷贝**则会复制对象指向的动态内存，**确保拷贝后的对象是完全独立的**。在使用中需要根据具体情况选择合适的拷贝方式，以避免出现内存管理问题。


# `std::weak_ptr`
`std::weak_ptr` 是 C++11 标准库中的智能指针类型，用于解决 `std::shared_ptr` 的循环引用问题。下面详细说明 `std::weak_ptr` 的作用：

1. **避免循环引用**：
   - 当多个对象相互持有 `std::shared_ptr` 智能指针时，可能会形成循环引用，导致对象无法被正确释放，从而产生内存泄漏。
   - `std::weak_ptr` 允许创建一个指向 `std::shared_ptr` 所管理对象的弱引用，而不会增加引用计数。这样即使存在循环引用，对象也能被正确释放。

2. **安全地访问对象**：
   - 通过 `std::weak_ptr` 可以安全地访问 `std::shared_ptr` 所管理的对象，因为 `std::weak_ptr` 不会增加引用计数，也不会影响对象的生命周期。
   - 在使用 `std::weak_ptr` 访问对象时，需要先将 `std::weak_ptr` 转换为 `std::shared_ptr`，如果对象已被释放，则返回空指针。

3. **用于观察者模式**：
   - `std::weak_ptr` 在观察者模式中非常有用，观察者持有被观察者对象的 `std::weak_ptr`，可以在需要时通过 `std::weak_ptr` 获取 `std::shared_ptr`，并安全地访问被观察者对象。

4. **用于缓存**：
   - 在缓存中，可以使用 `std::weak_ptr` 来存储对象的弱引用，当对象不再被需要时，可以被正确释放，而不会影响缓存中的 `std::weak_ptr`。


## 循环引用
循环引用（Circular Reference）是指在对象之间相互引用，形成一个闭环，导致对象之间的引用计数无法归零，从而造成内存泄漏的情况。以下是循环引用的详细说明：

1. **形成原因**：
   - 循环引用通常发生在使用智能指针（如`std::shared_ptr`）管理对象时，当多个对象相互持有对方的智能指针时，就会形成循环引用。
   - 当对象之间存在循环引用时，它们的引用计数永远无法归零，即使没有任何外部引用，这些对象也无法被正确释放，从而导致内存泄漏。

2. **影响**：
   - 循环引用会导致内存泄漏，占用系统资源并可能导致程序性能下降。
   - 内存泄漏可能会导致程序崩溃或异常终止，特别是在长时间运行的系统中，循环引用可能会导致系统稳定性问题。

3. **解决方法**：
   - 使用弱引用（`std::weak_ptr`）：将循环引用中的某些引用改为弱引用，避免增加引用计数，从而打破循环引用。
   - 手动解除引用：在不再需要对象之间相互引用时，手动断开引用，使得引用计数能够正确归零。
   - 设计良好的对象生命周期管理：合理设计对象之间的关系，避免形成循环引用。

总的来说，循环引用是指对象之间相互引用形成闭环，导致引用计数无法归零，从而造成内存泄漏。为避免循环引用带来的问题，需要注意对象之间的引用关系，合理使用智能指针，并采取相应的解决方法来避免内存泄漏。








