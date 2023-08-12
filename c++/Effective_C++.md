***
前言：
学习一门语言，不只是学点语法，高效的使用也是很重要的。对于C++而言更是如此。
***
对后者：C++的有效使用有两大类的建议：
1、一般性的设计策略（两种做法，选哪种）
2、带有具体细节的特定语言特性

# P1 习惯C++

# 01 语言联邦
C++由四个次语言组成：
1. C
2. Object-Oriented C++
3. Template
4. STL

# 02 以const、enum、inline取代#define
因为是预处理阶段做处理，编译器没做处理，**无法正确地出现编译错误**
用const double t = 1.23;就可以看到是t产生的编译错误
使用常量还可以使代码量较小，预处理器只是盲目地替换出现1.23的位置
对于class的专属常量，为了限制作用域在class内部，并防止产生多个实体，最好使用static
如果需要在编译期就使用一个class常量值，最好改用枚举enum,枚举不能用于取地址，不会为它分配额外的存储空间
对于形似函数的宏，最好改用inline的模板函数

# 03 尽可能使用const
```cpp
const* 
// 底层引用
*const
// 顶层引用：即指针本身是常量，值不能改

```
const_iterator可令函数返回常量值，减少错误
const标记成员函数的目的是明确该成员函数可作用于const对象
const和no const成员函数可重载，传入不同参数时调用不同的版本
如果需要这样做，但又不想要代码重复，可以在no const成员调用const成员函数来解决代码重复的问题
```cpp
const_cast<char&>(static_cast<const TextBlock&>(*this)[position]);
// 先确保调用的是const版本，再做去const处理

```


# 04 确定对象被使用前就已经被初始化
对于内置类型要进行手动初始化构造函数最好使用初始化列表（注意顺序），而不要在构造函数中赋值操作来初始化
--> 因为**成员初值列表可以在对象创建时直接初始化成员变量**，而不需要先调用默认构造函数再进行赋值操作，以提高性能
如果使用赋值操作初始化，首先调用默认构造函数创建对象，再调用赋值操作符进行赋值，多进行了对象的创建和销毁，增加了不必要的开销
并且成员初值列表还可以确保成员变量在对象创建时就被正确初始化，避免忘记初始化而可能导致的未定义行为
成员初值列表还可以用于初始化const成员变量和引用类型成员变量，而赋值无法做到这点

对于static对象，在跨编译单元之间的初始化次序说不确定的，C++只保证在本文件内使用之前一定被初始化
```cpp
class FileSystem{...};
FileSystem& tfs()
{
    static FileSystem fs;
    return fs;
}
// 以local static对象替换non-local static对象
```


# P2 构造/析构/赋值
# 05 了解C++默认生成的函数
如果不定义，C++会生成默认的构造函数，析构函数，拷贝赋值运算符，拷贝构造函数
但是如下情况不会生成默认拷贝赋值运算符
1. 类中含有**引用类型**的成员变量
2. 含有**const**的成员变量
3. 基类中的拷贝赋值运算符是私有成员函数

# 不想要编译器自动生成成员函数，则应该明确拒绝
可以声明为private并不予实现

# 07 为多态基类声明virtual析构函数
为了把动态绑定的派生类的析构函数一起调用，否则会内存泄漏

# 08 别让异常逃离析构函数
析构函数不要抛出异常，如果实在要抛出异常，最好使用std::abort，放在catch，把这个行为压下去
由于析构是用于清理资源和执行对象销毁前的操作的，如果析构中抛出异常，会导致对象的销毁被中断，资源无法正确释放，导致内存泄漏或其他不可预测的错误
其次，如果异常逃离析构函数，那么异常可能传播到调用析构函数的地方，可能导致更严重的问题，比如，如果析构函数是由一个容器类的析构函数调用的，那么异常将会传播到容器类的析构函数中，进而导致容器类的析构函数无法正常执行，可能导致容器内的其他对象无法正确销毁。
如果异常逃离析构函数，那么程序的异常处理机制将无法正常工作。通常，当异常发生时，程序会尝试捕获并处理异常，以保证程序的正常执行。但是，如果异常逃离析构函数，那么程序将无法捕获并处理该异常，可能导致程序崩溃或产生未定义的行为。
为了保证程序的稳定性和可靠性，不建议让异常逃离析构函数。可以在析构函数中使用try-catch块来捕获并处理异常，以确保对象的销毁过程能够正常进行，并正确释放资源。

# 09 不在构造和析构中调用vitual函数
构造一个派生类对象时，先调用基类构造函数构造基类部分，此时派生类那部分的成员变量还没有初始化，如果此时调用派生类版本的成员函数，可能会用到并未初始化的派生类部分的成员变量，导致未定义行为，因此，对于在构造函数中调用虚函数，编译器不会产生多态效果，而是调用基类版本。

析构函数：先调用派生类的析构，如果基类中调用派生类版本的virtual函数，显然导致未定义行为。

# 10 令operator=返回一个reference to *this
重载运算符时，尽量保留运算符原本的特性
为了实现连锁赋值（右结合律）
```cpp
(x = y) = z = 15;
// z = 15, x = y, x = z
// x = y返回对x的引用
```

# 11 在operator=中处理自我赋值
编译器生成的默认复制拷贝运算符函数是浅拷贝，对于指针是地址赋值
深拷贝则是：先释放原先的指针指向的内存，再开辟新内存，让指针指向新内存，内容是原指针指向的内容
```cpp
/* 对于以上代码，自我赋值会产生问题
例如：
SampleClass obj;
...
SampleClass& s = obj;
...
s = obj;
*/
// 问题在于先释放了内存,那就没法操作它了


//解决方法之一:
float* tmp = new float(*s.p);
// 用临时指针申请空间并填充内容
delete p;
p = tmp;

// 但是代码重复,有更好的办法:
SampleClass tmp(s);
swap(*this, tmp);
return *this;
// 把负担都交给了拷贝构造函数,因为如果拷贝构造函数无法执行,下面的swap就不会执行

// 当然也可以把入参由传引用改为传值,省略代码量
```

# 12 复制对象时勿忘其每一个成分
copying函数:即copy构造函数,copy assignment操作符
1. 拷贝函数要确保复制对象内"对象内的所有成员变量"及"所有base class 成分".
自定义构造函数时,如果有继承关系,要注意Base部分也要一同复制

2. 不要尝试以某个copying函数实现另一个copying函数.应该将共同机能放进第二个函数中,并由两个copying函数共同调用.
(因为可能导致无限地调用拷贝构造,最后导致栈溢出)

--> 正确做法:建立一个新的成员函数给copying函数调用,这种函数往往是private并且命名为init,从而安全地消除重复代码

# P3 资源管理
# 13 以对象管理资源
在构造函数中获取资源,在析构函数中释放资源,避免资源泄漏
考虑一个函数中：调用工厂函数得到返回的工厂对象，做完一系列操作后释放，但是在这个一系列操作中可能提前return（或许是由于软件开发过程中其他人的修改），因此要从根本上处理：把资源放进对象内，确保当控制流离开函数作用域，对象的析构函数自动释放资源，
实际的做法可以考虑用智能指针进行封装：智能指针对象析构时，会自动释放资源（当引用计数为0）
这就是所谓的**RAII思想：资源取得时机便是初始化时机**
                    **管理对象运用析构函数确保资源被释放**
需要注意，如果资源释放动作可能导致抛出异常，那么就可能无法无法正确释放资源，但是条款8规定：析构函数不要抛出异常，如果实在要抛出异常，最好使用std::abort，放在catch，把这个行为压下去
常用引用计数型智能指针
别用裸指针，容易忘记释放

# 14 在资源管理类中小心copying行为
由于有些资源不是定义在堆上的，没法用智能指针，需要自己实现资源管理类，这需要注意以下细节：
```cpp
// 比如锁定互斥量的Lock对象
class Lock{
private:
    Mutex* mutexPtr;
public:
    explicit Lock(Mutex* pm):mutexPtr(pm)
    {lock(mutexPtr);}   
    ~Lock(){unlock(mutexPtr);}  // 符合RAII
};
```

RAII对象被复制，会有问题：
对此，要么
## **禁止复制**：
将拷贝操作声明为private或者delete

## 要么**对底层资源进行引用计数**：
```cpp
// 比如锁定互斥量的Lock对象
class Lock{
private:
    std::shared_ptr<Mutex>mutexPtr;
public:
    explicit Lock(Mutex* pm):mutexPtr(pm, unlock)   // 将unlock函数作为删除器
    {
        lock(mutexPtr.get());   // get返回裸指针
    }
};
```
由于类的析构函数会自动调用其非静态成员变量的析构函数，此处就是mutexPtr的析构函数，其析构函数会在互斥量的引用计数为0时**自动调用shared_ptr的删除器（unlock函数）**

### 复制底部资源
有时复制资源管理类进行的是深拷贝。
某些标准字符串类型是由指向heap内存的指针构成（用于存放字符串的字符），这种字符串对象内含一个指针指向一块堆内存，当这种字符串对象被复制，不论指针或其所指内存都会被深拷贝。

### 转移底部资源的拥有权
有时需要确保永远只有一个RAII对象指向一个未加工资源，那么就涉及到移动。


# 15 在资源管理类中提供堆原始资源的访问
资源管理类可以防止资源泄漏，但是有时候得绕过资源管理对象直接访问原始资源。

智能指针提供了这样的接口
一方面，重载了->，*以获取内部资源
```cpp
class Font{
    ...
    public:
        FontHandle get() const {return f;}  // 显式转换函数

    private:
        FontHandle f;   // 原始资源
}
    显然如果用户API中大量需要FontHandle对象时，都得调用get()
    可以隐式转换：
    operator FontHandle() const
    {return f;}
    使得调用看起来比较自然：
    上述的隐式函数转换是一种函数语法格式，成员函数的一种，用于将本类型的函数转换成为其他类型，其中的operator相当于一个标志符，说明该是一个隐式函数转换。

    Font f(getFont());
    changeFontSize(f, newFontSize);
    // 但是显然有问题：
    Font f1(getFont());
    FontHandle f2 = f1; // 本意是拷贝Font对象，却反而将f1隐式转换为FontHandle才复制
    由于由Font对象f1管理的FontHandle对象也被f2取得，这是危险的，比如f1销毁了，f2就成了虚吊的。尽管此处是拷贝了一份。
```
提供get()方法获得裸指针并非是设计灾难，因为用户往往需要访问原始资源。

# 16 成对使用new和delete时要采取相同形式
new运算符：
1. 分配了内存
2. 针对此内存调用构造函数
delete恰恰相反

delete指针（指向的对象）的时候，一定要注意指向的是单一对象还是对象数组。因为二者内存布局不一样。显然数组所占用的内存还需要记录数组大小（以便delete时知道腰调用多少次析构函数）
--> 因此程序员需要显示地让delete知道内存中是否存在数组大小记录。
```cpp
std::string* stringPtr2 = new std::string[100];
delete [] stringPtr2;

typedef std::string AddressLines[4];
std::string* pal = new AddressLines;
// 等号右侧返回的是一个string*，就像new string[4]

// 对应：
delete [] pal;
最好不要对数组做typedef，容易忽略这个问题
```

# 17 以独立语句将newed对象置于智能指针
```cpp
// 考虑以下情形：
void processWidget(std::shared_ptr<Widget>pw, int priority);

processWidget(new Widget, priority());
// 由于Widget对象的构造函数声明为explicit，无法隐式转换，因此通过不了编译
// 必须这样写：
processWidget(std::shared_ptr<Widget>(new Widget), priority());
// 但是这种调用可能泄露资源
```
编译器在生成processWidget调用码之前，必须核算即将被传递的各个实参，其中第一个实参：
1. 执行new Widget
2. 调用shared_ptr构造函数
（3. 还有调用priority）
编译器生成的代码完成这三件事的顺序是有多种可能的，当然new Widget必然在shared_ptr之前，但是priority的调用的位置是不确定的
如果是1, 3，2的顺序执行，其中priority的调用异常，就会导致构造失败，new Widget返回的指针丢失，也就是说，在**资源创建**和**资源被转为资源管理类对象**之间可能发生异常干扰
解决办法是，把语句分开：
1. std::shared_ptr<Widget>pw(new Widgnet);
    用智能指针创建对象的语句单独一句
2. processWidget(pw, priority());
    这样调用，不会造成泄漏
因为编译器生成代码时指令乱序是针对语句内的，对于跨语句它不会这么做


# 18 让接口容易被正确使用，而不是误用
本条开始是针对软件设计与声明，以保证正确性、高效性、封装性、维护性、延展性以及协议的一致性。
function接口、class接口、template接口，每一种接口都是客户与你的代码互动的手段。
必须考虑客户可能做出什么样的错误，以日期类为例：
```cpp
Data(int month, int day, int year);
// 可能会以错误的次序传递参数
Data d(2, 30, 1995);

// 可以通过导入新类型来避免
Data d(Day(30), Month(3), Year(1995));  // 在其中做合法性检验
```
预防客户错误的另一个办法：限制类型内的操作：
比如：加上const
一个一般性准则：尽量让你的types的行为与内置types保持一致
避免无端与内置类型不兼容 --> 为了提供行为一致的接口
比如STL容器的接口就十分一致，比如每个容器都有名为size()的成员
比如之前提到的createInvestment函数，返回智能指针而不是裸指针可以避免两个用户错误的机会：忘记在最终把指针delete，或者重复释放一块资源。

shared_ptr有一个好的性质是：会自动调用它的“每个指针专属的删除器” --> 避免跨DLL错误（对象在动态链接库中被new创建，却在另一个DLL内被delte的运行时错误）shared_ptr中缺省的删除器是来自创建时的哪个DLL的delete
```cpp
如果Stock派生自Investment
std::shared_ptr<Investment>creatInvestment(){
    return std::shared_ptr<Investment>(new Stock);
    // 返回的shared_ptr可能被传递给任何其他DLL，这个指向Stock的shared_ptr会追踪记录“当Stock的引用计数为0时该调用的DLL的delete”
}

```

# 19 设计class犹如设计type
class类似于type的概念，设计者需要考虑：函数与操作符的重载、控制内存的分配和归还、定义对象的初始化和终结
如何设计高效的class？需要搞清楚以下问题：
1. 如何创建和销毁（构造函数和析构函数、内存分配和释放函数）
2. 对象的初始化和赋值有什么样的差别（考虑构造和赋值操作符的行为）
3. 新type的对象如果以值传递，意味着什么？（拷贝构造函数的行为）
4. 什么是新type的合法值？由于类的成员变量必然只在一定条件下才是有效的（因此显然要在构造、赋值、set函数里检查错误）
5. 新type如果继承自某个类，那么显然它必须受到父类的束缚。还有某类被继承，那么其析构需要为virtual
6. 新type需要什么样的转换？比如如果希望T1隐式转换为T2，需要在T1类内写一个类型转换函数（operator T2），或者T2内部写一个**可被单一实参调用**的构造函数
如果构造函数为explicit，就得显示地写出负责转换的函数
7. 什么样的操作符和函数对此type是合理的？
8. 什么样的标准函数应该驳回？（不想使用编译器生成的函数就必须禁掉）
9. 谁该取用新type的成员？--> 确定成员是public/protected/private，以及friend
10. 什么是新type的未声明接口（它对效率、异常、安全性以及资源运用（eg.多任务锁定、动态内存）提供何种保证
11. 新type有多么一般化（有时你实际上应该定义一个template）
12. 真的需要一个新的type吗？有时继承更能达到目的


# 20 宁以pass by reference to const替换pass by value
```cpp
// 考虑这种情况
class Person{
    public:
        Person();
    private:
        std::string name;
        std::string address;
};

class Student:public Person{
    public:
        Student();
    private:
        std::string schoolName;
        std::string schoolAddress;
};
```
如果传递Student对象时是传值，就要经历很多次的拷贝，而且创建的临时变量在函数返回时就销毁了，没意义。而且其中的成员变量std::string也会被拷贝。成本太大。
但是如果是**pass by reference to const**，就不需要任何的构造和析构，因为没有创建任何新对象。const意味着传入的对象不会被修改。
传递引用还可以避免**对象切割问题**：当一个派生类对象以传值方式传递并被视为基类对象，基类的构造函数会被调用，创建一个临时的基类对象，而派生类对象独有的部分被抛弃了。
```cpp
class Window{
public:
    virtual void display() const;
};

class WindowWithScrollBars: public Window{
public:
    virtual void display() const;
};

// 函数调用是这样的：
WindowWithScrollBars wwsb;
printNameAndDisplay(wwsb);
void printNameAndDisplay(Window w){
    std::cout << w.name();
    w.display();
}
// 不管接收的参数是什么类型，必然都只调用Window对象的display

void printNameAndDisplay(const Window& w){
    std::cout << w.name();
    w.display();
}
// 因为传递引用的实质是传递对象的指针，那个对象既可以是派生类又可以是基类。
```
如果对象是如同int在内的内置类型，传值效率更高。
但是即便是小型的type，也未必拷贝构造函数代价就小，因为类本身可能只含有一个指针，但拷贝构造的时候要把指向的对象一并拷贝。
而且某些编译器对于“内置类型”和“用户自定义类型”的操作是不一样的（即便两者底层的数据成员是一样的），比如某些编译器拒绝把只由一个double组成的对象放进cache，而对普通的double却会这么做。这种情况下，传递指针显然更好，因为指针必然会被放入cache。
***PS：Effective C++中提到的情况是指，某些编译器可能不会将仅由一个double类型的对象放入高速缓存中。这是因为高速缓存存储数据的方式是以缓存行（Cache Line）的形式进行的，每个缓存行有固定的大小，通常是64字节或者更大。如果一个double对象的大小正好等于一个缓存行的大小，那么存储该对象可能会占用两个缓存行，造成存储冗余或浪费。为了最大化利用高速缓存，一些编译器可能会尝试将多个对象放入一个缓存行中，以提高访问效率。而将只由一个double组成的对象放入缓存行可能会导致浪费，因为它占用了整个缓存行的空间而没有被充分利用。***
需要注意的是，不同编译器、不同平台以及不同编译器的版本可能会有不同的行为。这个具体问题的解决方法和影响因素可能会因系统和编译器的不同而有所差异。因此，在编程中，需要根据具体情况来决定如何优化代码以提高性能。
而且用户自定义类型的大小容易变化，因为内部实现可能改变，不同标准库实现版本的string大小可能有数倍差异。
可以合理假设传值不昂贵的只有：**内置类型和STL的迭代器和函数对象**

# 21 必须返回对象时，别想返回其引用
因为要注意：引用所指的对象可能并不存在
考虑一个Rational有理数类，有一个函数operator*计算两个有理数的乘积（返回该类的const对象），如果返回一个Rational对象，势必有构造和析构的开销，这能否避免？
如果是返回reference，那么必然是某个对象的别名，
```cpp
Rational a(1, 2);
Rational b(3, 5);
Rational c = a * b;
```
如果非得要返回引用，会有什么问题？
**注意不要返回局部变量（定义在栈上）的引用，局部变量出函数的作用域就被销毁**
如果非得要返回指针，会有什么问题？
**得在函数体内通过new把对象定义在堆上，但是这函数付出了构造的代价。因为分配得到的内存将以构造函数来完成初始化，但是谁来对构造出来的对象调用delete**
```cpp
Rational w, x, y, z;
w  = x * y * z;
// 等价于：
w = operator*(operator*(x, y), z);
// 其中调用了两次new，但是没法调用两次delete --> 导致内存资源占用却无法释放，即资源泄漏
```
还是有避免构造函数的办法...
```cpp
const Rational& operator* (const Rational& lhs, const Rational& rhs)
{
    static Rational result;
    result = ...;
    return result;
}
// 这段代码显然在多线程安全性方面有问题，但是还有更为严重的问题
bool operator== ...
{
    ...
    if((a*b) == (c*d)){ // 都是Rational对象，这个if条件必然被判断为true
    // 等价于：
    if(operator==(operator*(a,b), operator*(c,d)))  // 两次调用返回的都是static对象，即便两次调用都会分别改变静态对象的值，但是返回的是引用，在比较的时候两者是相同的
    }
}
```
实际上得认了：**一个必须返回新对象的函数，就让它返回一个新对象**
注意：不要返回局部变量的指针或者引用，也不要返回堆分配对象的引用

# 22 将成员变量声明为private
**如果成员变量不是public，客户唯一能访问该对象的办法就是通过成员函数
使用函数可以让程序员对成员变量的处理有更精确的控制 --> 比如实现“不准访问”、“只读访问”...，有些成员变量未必既要getter又要setter
这也有利于封装性：用户只知道调用这个接口而不知道内部实现
考虑一个速度数据采集类，其中有一个返回平均速度的函数，记录至今以来所有速度的平均值，其实现有两种：
1. 直接返回平均值成员变量   --> 需要另外创建多个相关的成员变量  但该函数本身很高效
2. 每次调用时重新计算，调取每一个速度值 --> 本身消耗大
视具体情况而定

将成员变量隐藏在函数接口后扩展性是极好的，而且还提供了class的约束条件
成员变量的内容改变时所破坏的代码数量越多，封装性越差
需要考虑一坨类的代码可能经常改变，因此要考虑一个成员变量改变后多少代码跟着改变
从封装角度看，只有两种访问权限，private（提供封装）和其他

# 23 宁以non-memer、non-friend替换member函数
考虑一个class用于表示网页浏览器，提供如下函数：清除元素高速缓存区、清除访问过的URLs的历史记录、移除系统中的所有cookies
许多用户可能想一次性执行所有操作，因此我们也提供一个封装了以上所有操作的成员函数：clearEverything()
当然它也可以是非成员函数：
```cpp
class clearBroswer(WebBroswer& wb){
    wb.clearCache();
    wb.clearxxx();
    ```
}
```
需要注意的是：**member函数带来的封装性不如non-member函数**
因为non-member函数可对WebBrowser相关内容有较大的包裹弹性（延展性），尽管导致了较低的编译相依度。
封装的意义在于：使我们能够改变事物而只影响有限客户。（越少的人可见，我们就有越大的弹性去改变它）

考虑对象内的数据，越少代码可以访问它，越多的数据可被封装，越多的函数可访问它，数据的封装性越低。
由于成员变量尽可能地被设计为private，能够访问成员变量的函数只有member函数和friend函数。
non-member non-friend函数好在并不增加“能够访问class内private成分”。
回到上述例子，c++中常见的做法是：clearBroswer函数与类的定义处于同一个namespace（可以跨越多个源码文件而class不行）。
考虑更为普遍的情况：比如一个书签类，我们可以将与其相关的便利函数都声明于一个头文件。
c++标准程序库也是如此组织的，便利函数分布于多个头文件但是隶属于同一个namespace --> 用户可以轻松扩展这一组便利函数，即添加更多的non-member或non-friend函数
（class相比non-member缺点是对于用户而言不能扩展，即便是派生类，也无法访问基类中的private成员，更何况并非所有的类都被设计为基类）

# 24 若所有参数都需要类型转换，采用non-member比较好
一般只有对于数值类型，才让这种class支持隐式转换
比如有理数类，那么有理数相乘的函数是否设计在内类比较好呢？
如果这样实现的话：
```cpp
class Rational{
    public:
    ...
    const Rational operator*(const Rational& rhs) const;
}

// 用起来就是这样：
Rational oneEigth(1, 8);
Rational oneHald(1, 2);
Rational result = oneHalf * oneEight;

// 如果和int混合运算，就有问题：
result = oneHalf * 2;
result = 2 * oneHalf;   // 错误！毕竟2不是Rational类型的对象，无法调用operator*方法
前一个调用之所以可行是因为发生了隐式类型转换：
const Rational temp(2);
result = oneHalf * temp;
当然前提是构造函数是non-explicit的，编译器才会做隐式类型转换
而且之所以可以隐式类型转换是由于参数位于参数列表内，才能进行隐式类型转换。
```
这样就可以看出non-member函数的好处：
```cpp
const Rational operator*(const Rational& lhs, const Rational& rhs);
// 而且 该函数不需要是Rational的友元函数
```
可以避免friend函数就要避免

# 25 考虑写出一个不抛出异常的swap函数
swap函数，可用于异常安全性编程，以及处理自我赋值
```cpp
// 典型实现
namespace std{
    tempplate<typename T>
    void swap(T& a, T& b){
        T temp(a);
        a = b;
        b = temp;
    }
}
```
只要T类型支持copying（通过拷贝构造函数和copy assignment操作符完成）
这样的实现效率太差，不够刺激（因为产生了不必要的拷贝，对于某些类型而言）
```cpp
// 比如这个类型
class WidgetImpl{
public:
    ...
private:
    int a, b, c;
    std::vector<double>v;   // 数据复制的时间很长

};

class Widget{
public:
    Widget(const Widget& rhs);
    Widget& operator=(const Widget& rhs){
        ...
        *pImpl = *(rhs.pImpl);
        ...
    }
    ...
private:
    WidgetImpl* pImpl;
};
// 对于Widget类型的swap，显然只要置换pImpl指针，但是原先的swap算法会无脑产生3个Widget的实例
```
想要让它不无脑，可以针对Widget类型特化
```cpp
namespace std{
    template<>  // 全特化
    void swap<Widget>(Widget& a, Widget& b){    // 说明针对什么全特化
        swap(a.pImpl, b.pImpl);
    }
}
// 我们可以为std模板增加特化版本，使得针对我们自定义的类型
// 但无法通过编译，因为pImpl是友元类型，虽然可以声明为friend，但是不太好

// 我们可以在Widget内部声明一个public成员函数：swap
class Widget{
public:
    ...
    void swap(Widget& other){
        using std::swap;
        swap(pImpl, other.pImpl);
    }
};
// 再加上：
namespace std{
    template<>  // 全特化
    void swap<Widget>(Widget& a, Widget& b){    // 说明针对什么全特化
    a.swap(b);
    }
}
// 这种方法不仅能通过编译，而且和STL有一致性，所有STL容器也提供有public swap成员函数和std::swap特化版本

// 如果Widget和WidgetImpl都是类模板呢？
// 情况不同了
namespace std{
    template<typename T>
    void swap<Widget<T>>(Widget<T>& a, Widget<T>& b)
    {a.swap(b);}
}
因为C++只允许对类模板偏特化，而不是对函数模板偏特化

可以这样做：
namespace std{
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b)
    {a.swap(b);}
}   // 重载
// 但是这种方式被std禁止,std是特殊的命名空间，客户客户全特化std的模板，但是不可以添加新的模板/函数/类

// 还有办法：声明non-member函数，让它调用member swap，但是不将non-member swap特化或者重载
namespace WidgetStuff{
    ...
    template<typename T>
    class Widget{...};  // 内含swap成员函数
    ...
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b){
        a.swap(b);
    }
}   // 这个swap是这个类专属的
// C++的查找规则会找到WidgetStuff内的专属版本
// 但是还是要为class特化std::swap

// swap有多个版本：一般化版本、特化版本、某个命名空间的T专属版本
// 我们需要调用最佳swap版本（调用T专属版本，如果没有就调用一般的swap）
template<typename T>
void doSth(T& obj1, T& obj2)
{
    using std::swap;
    ...
    swap(obj1, obj2);
}


C++的名称查找法则确保找到global作用域或者T所在命名空间内的T的专属swap，如果没有专属的就用std::swap


```

现在我们对上述内容做一个总结：
如果缺省的swap可以对你的class或者class template提供可接受的效率，那不需要做任何事情。
如果swap缺省实现版本效率不足（那几乎总是意味着你的class或template使用了某种pimpl手法），试着做以下事情：
**提供一个public swap成员函数，让它高效地置换你的类型的两个对象值；在你的class或者template所在的命名空间内提供一个non-member swap，并令它调用上述swap成员函数。**
如果你正在编写一个class（而非class template），为你的class特化std::swap，并令它调用你的swap成员函数。
如果你调用swap，请确定包含一个using声明式，以便让std::swap在你的函数内曝光可见，然后不加任何namespace修饰符，赤裸裸地调用swap。

请记住当std::swap对你的类型效率不高时，提供一个swap成员函数，并确定这个函数不抛出异常。
如果你提供一个member swap，也该提供一个non-member swap用来调用前者。对于classes（而非templates），也请特化std::swap。
调用swap时应针对std::swap使用using声明式，然后调用swap并且不带任何“命名空间资格修饰”（比如std）。
为"用户定义类型"进行std templates全特化是好的，但千万不要尝试在std内增加某些对std而言全新的东西。

# 26 尽可能延后变量定义式出现的时间
需要注意,只要一个变量其类型带有构造&析构,就需要考虑其构造和析构的成本,即便这个变量最终未被使用,还是耗费了这些成本.
定义了变量但并未使用,这种情况并不罕见,比如由于抛出异常就有可能更改原先的控制流,**这时可以把局部变量定义在抛出异常之后**.
而且要注意:构造时传入参数比构造后用参数赋值效率更高.
也就是说,**尽可能延后**指的是:**不仅应该延后变量的定义,直到非得使用的前一刻,甚至应该尝试延后定义直到能够给它初值实参**.
而避免无意义的(默认)构造/析构.
对于for循环(n次循环)而言,变量定义在循环体内部还是外部比较好呢?
1. 定义在内部:  n次构造 + n次析构
2. 定义在外部:  1次构造 + 1次析构 + n次赋值
这显然要看情况:如果classes的一个赋值成本低于一组构造 + 析构成本,2效率更高,反之则1效率高
定义在循环外还会导致对象的作用域覆盖整个循环,有时会对程序的可理解性和易维护性造成冲突

# 27 尽量少做类型转换动作
C++规则的设计目标之一就是保证"类型错误"不会发生.而类型转换破坏了类型系统.
```cpp
// 旧式转型

// C语言风格的转型
(T) expression; // 将expression转型为T

// 函数风格
T(expression);  // 将expression转型为T

// 新式转型
const_cast<T>(expression)
dynamic_cast<T>(expression)
reinterpret_cast<T>(expression)
static_cast<T>(expression)
```
## const_cast
移除对象的const特性

## dynamic_cast
安全向下转型,用于决定某一对象是否属于继承体系中的某个类型,唯一无法由旧式语法实现的动作,唯一可能耗费重大运行成本的转型动作

## reinterpret_cast
执行低级转型,结果可能取决于编译器,因而不可移植,例如将int指针类型转为int(很少见,只有针对原始内存写出一个调试用的分配器时会用到)

## static_cast
强制隐式类型转换,例如将非const转为const,int转为double
还有上述类型的反向转换
将void*指针转为typed指针,基类指针转派生类指针
无法将const转为非const

新式转型比较容易阅读、而且更具针对性
唯一使用旧式转型的时机：调用一个explicit构造函数将一个对象传递给一个函数时：
```cpp
class Widget{
public:
    explicit Widget(int size);
    ...
};
void doSomeWork(const Widget& w);

doSomeWork(Widget(15)); // 显式地创建了对象

doSomeWork(static_cast<Widget>(15));    // 转型的方式创建
// 可以用新式转型，但是旧式更为显眼
```
注意：类型转换会使得编译器生成相关的机器码，而不只是让编译器把某种类型视为另一种类型
当把一个基类指针指向派生类对象时：
```cpp
Base* pb = &derived;
// 隐式地将派生类指针转为基类指针（转换的实现是对派生类指针所指的地址增加一个offset）
```
也就是说，在C++中，一个对象可能拥有多个地址（用基类指针指向时的地址、用派生类指针指向时的地址），因而无法简单地假设对象在C++中的布局
对象布局和编译器有关，是会变化的

关于转型，我们容易写出看似正确的代码：
在派生类中定义一个虚函数onResize（基类也有它的虚函数定义）
想法是：在派生类中调用一下基类的函数，让其生效
```cpp
class Derived: public Base{
public:
    virtual void onResize(){
        static_cast<Base>(*this).onResize();    // 试图将派生类转为基类，调用该方法，是错误的
    }
}
```
错在何处？
这种做法调用的对象并非是纯粹的基类对象，而是转型动作建立的一个“*this对象的base class部分”的临时副本的onResize
（类的成员函数都有隐藏的this指针，会影响成员函数操作的数据）
这种做法是在当前对象的base class成分的副本上调用基类的方法，这样会导致当前派生类对象实际没被改动，改动的是副本（**会创建临时基类对象**）。
解决办法：
```cpp
class Derived: public Base{
public:
    virtual void onResize(){
        Base::onResize();   // 显示地调用
    }
}
```

dynamic_cast
dynamic_cast的实现速度相当慢，这是因为：比如四层深度的单继承体系，对某个对象执行dynamic_cast，可能会调用多次strcmp，用于比较class名称，因此要在注重效率的代码中对其保持怀疑态度。
用于：想要对派生类对象执行派生类的函数，但是手头只有基类指针或者引用指向它。
1. 使用容器，并在其中存储直接指向派生类对象的（智能）指针
```cpp
typedef std::vector<std::shared_ptr<Derived>> DRV;
DRV derivedPtrs;
...
for(DRV::iterator iter = derivedPtrs.begin(); iter != derivedPtrs.end(); ++iter){
    (*iter)->fun(); // 不要用dynamic_cast把基类指针转为派生类指针，而是直接使用派生类指针
}
// 当然，如果基类有多种继承形式，那么就需要多种容器，并且要具备类型安全性
```

2. 基类提供虚函数，利用多态

绝对要避免连串的dynamic_cast
```cpp
if(Derived1* p1 = dynamic_cast<Derived1*>(iter->get()))
    ...
else if(Derived2* p2 = dynamic_cast<Derived2*>(iter->get()))
    ... 
else if(Derived3* p3 = dynamic_cast<Derived3*>(iter->get()))
    ...
```
因为需要考虑到：这些类的继承体系一旦变化，这边的代码都可能需要一并修改
这完全可以用虚函数的方式避免

如果非得要转型，要把它封装起来隐藏在某个函数的背后，不要让客户直接操作那些转型的代码

# 28 避免返回handles指向对象内部成分
考虑以下场景：
矩形类中有一个点类（辅助类）
```cpp
struct RectData{
    Point ulhc;
    Point lrhc;
};

class Rectangle{
    ...
private:
    std::shared_ptr<RectData> pData;
}
```
由于用户需要计算矩形的范围，也就是需要得到点对象，如果我们对外提供传引用的接口：
```cpp
class Rectangle{
    ...
    Point& upperLeft() const { return pData->ulhc; };
};
```
这样的设计是有问题的：因为：传引用会导致用户可能更改点数据，用户只要得知就行了。
更何况这样的函数实际是把Rectangle类的private数据传出去了。
教训：
1. 成员变量的封装程度取决于返回其引用的函数（即便声明为private，还是可能被public函数传出去）
2. 如果const成员函数传出一个引用，该引用所指的数据与对象自身有关，而它又存储于对象本身的位置之外，那么会导致该函数的调用者可能修改那笔数据。

引用、指针、迭代器都是所谓的handles（号码牌，用于取得某个对象）
返回这么一个handle的风险就是降低了对象的封装性 --> 即便调用的是const成员函数，对象的状态仍然会改变
要留心protected或者private的函数不要返回成员变量的handles，也就是说，绝对不要让成员函数返回一个指针，它指向访问级别较低的成员函数，这变相地提高了这个访问级别较低的成员函数的访问级别

因此，上述的返回点对象的接口可以在返回值增加const标识符，从而实现了**有限度的封装**
但是也有隐患：可能导致悬空的handles（因为引用所指的对象可能并不存在）
```cpp
const Rectangle getRectangle(const GUIObject& obj);

// 如果这样调用：
GUIObject* g;   // 指向某一GUIObject对象
...
const Point* pUpperLeft = &(getRectangle(*g).upperLeft());
// getRectangle返回了一个新的临时对象，是匿名的，这行语句结束就销毁了，所以导致了悬空
```
也并非成员函数就不能返回handle了，比如operator[]，允许用户得到vector的某个元素，那些数据随着容器的销毁而销毁。



# 29 为“异常安全”而努力是值得的
假设一个class用于表示GUI菜单，它用于多线程，因此有一个互斥量是它的成员变量，用于并发控制。
如果代码是这样写的：
```cpp
class PrettyMenu{
public:
    ...
    void changeBackground(std::istream& imgSrc);
    ...
private:
    Mutex mutex;
    Image* bgImage;
    int imageChanges;   // 执行changeBackground的次数
};

// 如果这样实现：（要能看出缺陷在何处）
void PrettyMenu::changeBackground(std::istream& imgSrc){
    lock(&mutex);
    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
    unlock(&mutex);
}
```

**异常安全有两个条件：如果一个函数是异常安全的，在异常抛出时，它可以做到：1. 不泄露任何资源 2. 不允许数据破坏**
上面代码错误在于：
1. 互斥量资源泄漏了：new Image可能会抛出异常，那么unlock就永远不会执行，释放不了互斥量了
2. 同样，new Image若抛出异常，bgImage仍然是一个指向被删除对象的指针，而imageChanges计数仍然累加，与它的意义不符合（这行代码的位置不能随便放）

解决办法是：**以对象管理资源**
让对象的生命周期去控制互斥量的上锁与解锁
```cpp
void PrettyMenu::changeBackground(std::istream& imgSrc){
    Lock ml(&mutex);    // 这样写的好处还有比较简洁，出错概率小
    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
}
```

## 异常安全函数的保证
### 基本承诺
如果异常抛出，程序的任何事物都要保持有效状态，不破坏任何对象或数据结构，比如：changeBackground函数中一旦抛出异常，PrettyMenu对象可以继续拥有原背景图像，或者是拥有缺省的背景图像

### 强烈保证
如果异常安全的函数调用失败，程序会回到调用该函数之前的状态
这样的话，可以确保程序状态只有两种可能，要不然就无法预测了

### nothrow保证
承诺不抛出异常，对内置类型做的操作都是不抛出异常的
如果异常明细为空（C++11之前用的），也不代表就必然不抛出异常，这取决于函数实现

实际上很难完全没有调用任何一个可能抛出异常的函数。比如STL容器如果内存空间不足，就会抛出bad_alloc
对于大部分函数，只能做到基本保证和强烈保证

异常安全码必须提供上述三种保证之一。

使得changeBackground具有强烈保证的代码：
将裸指针改为智能指针，防止资源泄漏

```cpp
void PrettyMenu::changeBackground(std::istream& imgSrc){
    Lock ml(&mutex);    // 这样写的好处还有比较简洁，出错概率小
    bgImage.reset(new Image(imgSrc));   // reset函数保证：只有在参数正确生成才会被调用，内部有delete原来的指针的操作
    ++imageChanges;
}
```
美中不足之处是：如果Image构造函数抛出异常，istream对象的读取记号已经被移走，导致状态的改变。
将其完善成真正的强烈保证的办法有很多，但是有个很一般化的设计策略：**copy and swap**。
即：为打算修改的对象做一份副本，然后在副本上做修改，若有任何修改则抛出异常，在确保所有改变成功后，才将本体和副本置换。
实际的实现通常是**pimpl idiom**：将所有“隶属对象的数据”从原对象放进另一个对象内，然后赋予原对象一个指针，指向副本。
```cpp
// 为了强烈保证而再度优化
struct PMImpl{
    std::shared_ptr<Image>bgImage;
    int imageChanges;
};  // 封装它的shared_ptr已经在class里设计为private

class PrettyMenu{
    ...
private:
    Mutex mutex;
    std::shared_ptr<PMImpl> pImpl;
};

void PrettyMenu::changeBackground(std::istream& imgSrc){
    using std::swap;    // 为了高效地swap
    Lock m1(&mutex);
    std::shared_ptr<PMImpl>pNew(new PMImpl(*pImpl));    // 通过本体产生副本对象
    pNew->bgImage.reset(new Image(imgSrc)); // 修改的是副本
    ++pNew->imageChanges;
    swap(pImpl, pNew);
}
```
copy and swap这招有利于保证对象的状态“全有或全无”，但是它一般不能保证整个函数具有强烈的异常安全性。
```cpp
void someFunc(){
    ... // 拷贝本地数据
    f1();
    f2();
    ... // 置换
}
```
如果f1或f2的异常安全性比“强烈保证”低，那么就难以实现强烈异常安全。要不然就得记录下来调用f1前的整个程序的状态。
即便f1、f2是“强烈异常安全”，f1结束后，程序状态仍有可能改变，f2如果抛出异常的话，就没能保证状态的一致性。
关键在于**连带影响**，若函数只操作局部性状态，例如someFunc只影响其调用者的状态，就容易提供强烈保证，如果对“非局部性数据”有影响，那就困难了。

而且copy and swap在时间和空间上消耗较大，强烈保证因此可能是不切实际的。那就提供基本保证好了，尽力而为就是通情达理的。

一个软件系统要么是不具备异常安全性，要么就是具备的。


# 30 透彻了解inling的里里外外
内联函数：看似是函数，相比宏又可以在编译期做出一些检查，而且又因为函数体展开而不用付出函数调用的开销。
编译优化机制通常用于浓缩那些不含函数调用的代码，声明inline后编译器就可以对它执行语境相关最优化，编译器一般不会对非inline函数做这种优化。
## 代价是什么呢？
编译器会把每一处函数调用都用函数本体替换，导致目标代码的增加，进而可能导致额外的换页行为，降低cache命中率，导致低效率。
如果inline函数的本体很小，展开的那些代码比函数调用产生的代码更小，那么inline很好。
inline只是一个申请，不是强制的。而且类内部的成员函数都是默认inline的，友元函数如果被定义于类内部，那么也会被声明为inline。
inline未必就好，比如以下代码：
```cpp
template<typename T>
inline const T& std::max(const T& a, const T& b)
{ return a < b ? b : a; }
```
inline函数常常被放在头文件，因为大部分编译环境在编译时做inlining时，为了将“函数调用”替换为“被调用函数的本体”，编译器当然得知道函数的内部实现。
某些编译环境可以在链接时完成inlining，有的甚至能在运行时完成，当然对于c++而言，inlining是编译期行为。
而templates通常也放在头文件，其实例化与inline无关，如果你写的模板函数没有必要让所有实例化版本都是inline的，就要避免inline（因为需要成本）

编译器会拒绝复杂的函数inline（带有循环或者递归），如果有虚函数（毕竟是运行时才确定调用哪个函数）的调用也无法inline。

**总结：一个声明inline的函数，是否真的被内联化，取决于编译环境和编译器。（不做内联会提出警告）**
有时编译器会inline某个函数，但还是会生成函数体，比如：
```cpp
inline void f(){...}
void (*pf)() = f;   // 使得函数指针pf指向f
...
f();    // 会被内联
pf();   // 由于是通过函数指针调用的函数体，不会被内联
```
编译器有时生成构造和析构的非内联副本，以便通过指针指向那些函数
实际上构造函数和析构不宜inline：
原因是：构造和析构内也有许多逻辑（比如包含了每一个成员的构造），可能会导致很大的开销。

更何况，由于被inline的函数体被展开了，那么如果它是函数库里的函数，客户用到此函数之处就都得重新编译，不是inline的话，只要重新链接就行了，动态链接的话更是神不知鬼不觉。
还有，调试器无法处理inline，没法打断点。

因此，比较好的操作是：只有必须要inline的函数，以及十分简单的函数才要inline。
经验之谈：一个程序80%的执行时间花费在20%的代码上。

# 31 将文件间的编译依存关系降至最低







