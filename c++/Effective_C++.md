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

# 06 不想要编译器自动生成成员函数，则应该明确拒绝
可以声明为private并不予实现

# 07 为多态基类声明virtual析构函数
为了把动态绑定的派生类的析构函数一起调用，否则会内存泄漏

# 08 别让异常逃离析构函数
析构函数不要抛出异常，如果实在要抛出异常，最好使用std::abort，放在catch，把这个行为压下去
由于析构是用于清理资源和执行对象销毁前的操作的，如果析构中抛出异常，会导致对象的销毁被中断，资源无法正确释放，导致内存泄漏或其他不可预测的错误
其次，如果异常逃离析构函数，那么异常可能传播到调用析构函数的地方，可能导致更严重的问题，比如，如果析构函数是由一个容器类的析构函数调用的，那么异常将会传播到容器类的析构函数中，进而导致容器类的析构函数无法正常执行，可能导致容器内的其他对象无法正确销毁。
如果异常逃离析构函数，那么程序的异常处理机制将无法正常工作。通常，当异常发生时，程序会尝试捕获并处理异常，以保证程序的正常执行。但是，如果异常逃离析构函数，那么程序将无法捕获并处理该异常，可能导致程序崩溃或产生未定义的行为。
为了保证程序的稳定性和可靠性，不建议让异常逃离析构函数。可以在析构函数中使用try-catch块来捕获并处理异常，以确保对象的销毁过程能够正常进行，并正确释放资源。

# 09 不在构造和析构中调用virtual函数
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
struct PMImpl {
    std::shared_ptr<Image> bgImage;
    int imageChanges;
};  // 封装它的shared_ptr已经在class里设计为private

class PrettyMenu {
    // ...
private:
    Mutex mutex;
    std::shared_ptr<PMImpl> pImpl;
};

void PrettyMenu::changeBackground(std::istream& imgSrc) {
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
如果只改动c++的某个class的private部分的实现，会导致大量的重新编译和链接。--> 原因是接口和实现分离做的不好。
比如class Person中私有成员是自定义的Date类型，那么在定义class Person的头文件里必然会有calss Date的定义。此处就形成了**依赖关系**，被依赖的部分有改变，那么相应地部分也必须重新编译。这种**连串**（因为实际情况可能是一连串）的编译依存关系是灾难性的。
那如果不用头文件include，而是在相应的namespace里采用类的前置声明呢？
比如在std的namespace里前置声明class string。
有两个问题：
1. string不是class，它本质上是对`basic_string<char>`的`typedef`，因此简单的`class string;`是错误的，正确的前置声明很复杂
2. 前置声明只适用于编译器在编译期间知道对象大小的情况，要不然编译器根本不知道类占据多少空间
如果是Java，A类定义中，如果B类的对象是成员变量，实际底层实现是持有B类型的指针。
因此在C++中我们也可以尝试将对象实现细节隐藏在指针背后：
```cpp
#include<string>
#include<memory>

class PersonImpl;
class Date;
class Address;

class Person {
public:
    Person(const std::string& name, ...Data...);
    std::string name() const;
    std::string birthDate() const;
    // ...
private:
    std::shared_ptr<PersonImpl> pImpl;
};
```
这就是所谓的pimpl idiom（pointer to implementation），实现了接口和实现的分离
它遵循了**编译依存性最小化的本质：让头文件尽可能自我满足，如果做不到，就让它与其他文件内的`声明`（而非定义）相依（即：以声明的依存性替换定义的依存性）**
这个指导思想还产生了一些其他的设计策略：
1. 如果使用`引用`或者`指针`可以完成任务，就不要使用对象，因为仅需类的声明就能定义指针或者引用，而创建对象需要类的实现

2. 尽可能地以class的声明式替换class定义式，因为**声明函数的时候是不用类的定义的，类的声明已经足够**
```cpp
class Date; // 声明
Date today();
void clearAppointments(Date d);
```
但是如果是这个函数的调用者，那就必须得到Date的定义式。
这样做的实际意义在于：一个函数库内可能有数百个函数，每个客户只会调用其中几个，这样把class定义式`延迟`到含有函数调用的客户文件就实现了去除与客户端之间的编译依存性。

3. 为声明式和定义式提供不同的头文件
具体来说，比如前面的例子：在`datefwd.h`文件中，只含有Date的声明而无定义。
c++标准库有类似的设计：`<iosfwd>`含有iostream各组件的声明式，而定义分布在`<sstream>`、`<fstream>`中，可见本条款也适用于`template`
如果template定义放在非头文件，就可以将“只含有声明”的头文件提供给templates

```cpp
#include"Person.h"
#include"PersonImpl.h"  // 需要其定义式，调用其成员函数
// Person和PersonImpl有着完全相同的成员函数，接口完全相同
Person::Person(const std::string& name, ...) : pImpl(new PersonImpl(name, birthday, addr)){}    // 内部调用PersonImpl构造函数

std::string Person::name() const {
    return pImpl->name();
}
```
让Person变成一个`Handle class`不会改变它做的事，只会改变它做事的方法

另一种办法是抽象基类（又称`interface class`）
通常只含有虚析构、一系列纯虚函数、或者也可以含有一定的成员函数的实现
```cpp
class Person {
public:
    virtual ~Person();
    virtual std::string name() const = 0;
    // ...
}
```
接口类的客户需要调用其中的函数得到真正的派生类的实例化，这样的函数通常被称为工厂函数，返回智能指针，指向动态分配的对象。而且通常被声明为static
```cpp
class Person {
public:
    // ...
static std::shared_ptr<Person>create(const std::string& name, const Data& birthday, const Address& addr);
}

// 客户这样使用：
std::shared_ptr<Person> pp(Person::create(name, dataOfBirth, address));
std::cout << pp->name();
```

尽管handle classes和interface classes带来了额外的开销，但是这是为了更大的利益。



# 32 确定你的public继承，是含有is-a的关系的
public继承意味着“is a”的关系
比如Derived公有继承Base，等于告诉编译器每一个类型为Derived的对象同时也是一个类型为Base的对象，反之不成立
**利斯科夫替代原理：任何基类可以出现的地方，子类一定可以出现。**
经典的情况：Student公有继承Person，Student是Person的特殊化
因此，**C++函数如果入参为Person类型的对象，那么也可以接收一个Student类型的对象，反之则不行**
注意前提条件是public继承，private继承的意义完全不同，protected则是意味不明
但有时，程序未必能简单地描述出一些关系：比如我们会说，鸟会飞，但是企鹅是鸟，但是企鹅不会飞，严谨来说，企鹅和会飞的鸟都是鸟的派生类才对。而如果某些软件系统可能不需要区分会飞的鸟和不会飞的鸟，直接让企鹅继承鸟类即可。
当然也可以为企鹅重新定义fly函数（虚函数），让其产生一个运行期错误（这样就意味着企鹅会飞，但是尝试飞是一种错误）
从错误产生的角度来看，企鹅不会飞在编译期判断出来，而企鹅尝试飞是一种错误是直到运行时才检测出来。
好的接口应该要防止无效的代码通过编译

再考虑一下正方形和矩形的关系：
如果Rectangle公有继承Square，而Rectangle提供一个函数makeBigger()，用于将宽度增加10，其中断言高度是否未曾发生变化
Square对象s，在调用makeBigger()前后，破坏了正方形的长宽相等的情况。
**问题根源在于Rectangle和Square的关系并不是简单的公有继承关系**，而public主张基类能调用的方法，也能作用于派生类上

除了is-a的关系，类间关系还有has-a，以及is-implemented-in-terms-of。

# 33 避免遮掩继承而来的名称
```cpp
int x;  // 全局变量
void someFunc(){
    double x;
    std::cin >> x;  // 全局变量和局部变量重名的话，优先选择局部变量，也就是说全局变量被遮掩
}
```
**内层作用域的名字会掩盖外层作用域的名字**
此处就是double类型的x遮掩了int类型x。
接下来考虑继承的情况，当在派生类成员函数内部使用基类中的某物（成员函数、typedef、成员变量）时，此时意味着派生类作用域被嵌套在基类作用域内。
编译器会**优先找局部的作用域**
考虑此例：
```cpp
class Base{
private:
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    virtual void mf2();
    void mf3();
    void mf3(double);
    ...
};

class Derived: public Base{
public:
    virtual void mf1();
    void mf3();
    void mf4();
    ...
};
```
这样写的后果是：派生类无法继承基类的函数：mf1和mf3
```cpp
Derived d;
int x;
...
d.mf1();    // 调用派生类版本mf1
d.mf1(x);   // 错误！被遮掩
d.mf2();    // 没和基类重名
d.mf3(x);    // 错误！被遮掩
```
**可见，在派生体系中，即便函数参数不一样，也不管是不是虚函数，名称遮掩的规则都存在（重点就是函数名字）**。
这种设定的意义在于：
防止在程序库内建立新的派生类时，一并把关系较为疏远的基类里的重载函数一并继承。如果你正在使用public继承而又不继承那些重载函数，就违反了is-a的关系。
对此解决办法是：
```cpp
class Base{
public:
    virtual void mf1() = 0;
};
class Derived: public Base{
public:
    using Base::mf1;    // !使得基类的mf1在派生类的作用域也可见 放在public的位置也是有意义的：基类的public名称在公有继承的派生类内也是public
}
```
如果派生类是private继承基类，并且唯一想继承基类的某个函数，没法用using声明了，因为它会导致继承而来的所有同名的函数在派生类中都可见（**无法具体到某一个函数**）
可以使用转交函数来实现（forwarding function）
```cpp
class Base{
public:
    virtual void mf1() = 0;
    virtual void mf1(int) = 0;
};

class Derived: public Base{
public:
    virtual void mf1(){
        Base::mf1();    // 偷偷地成为inline函数
    }
}
```
转交函数的好处还有：为那些不支持using的老式编译器将继承而得的名称汇入派生类作用域。


# 34 区分接口继承和实现继承
在继承体系中，既有**函数接口（function interfaces）继承（或者说声明），又有函数实现（function implementations）继承**。
```cpp
class Shape{
public:
    virtual void draw() const = 0;  // 纯虚函数
    virtual void error(const std::string& msg); // 虚函数
    int objectID() const;   // 普通函数
    ...
};  // 是抽象类

class Rectangle: public Shape{...};
class Ellipse: public Shape{...};
```
注意：对于public继承，成员函数的接口总是会被继承，毕竟是is-a的关系
以上的基类Shape设计了三种不同类型的函数：
1. 纯虚函数（必须被派生类重新实现），目的就是**为了让派生类只继承函数接口**，此处的函数也很有意义：毕竟不知道是什么图形，怎么知道怎么draw
但是C++编译器允许你为纯虚函数提供定义，但是调用它必须要明确其class名称：
```cpp
Shape* ps = new Rectangle;
ps->Shpe::draw();
```
目的是为了给虚函数提供平常且安全的缺省实现。

2. 而虚函数的话，派生类可以继承其函数接口，但本身它也会提供一份代码，派生类可以覆写它(override)。
**声明非纯的虚函数，是为了让派生类继承该函数的接口和缺省实现**
```cpp
virtual void error(const std::string& msg); // 虚函数
```
其中蕴含的语义是：`每个class都必须支持一个“当遇上错误时可调用”的函数，但是每个class也可以自由地处理错误，不想特殊处理的话就可以退回到缺省的错误处理行为`

但是，允许虚函数同时指定函数声明和函数缺省行为，有可能造成危险。
```cpp
class Airport{...};
class Airplane{
public:
    virtual void fly(const Airport& destination){...// 缺省代码}
    ...
}
}
;
class ModelA: public Airplane{...};
class ModelB: public Airplane{...};
// 经典的面向对象设计
```
ModelA和ModelB同时继承了一份相同性质的函数，因为可以使用缺省的实现，所以可以避免代码重复
但是如果此时有个ModelC类，它的fly()函数是必须重新实现的，如果程序员忘记为其重新实现，那程序运行时就会找到缺省实现，而ModelC无法拒绝这个缺省实现。
解决办法之一是：可以设置Airplane中fly为纯虚函数，另外再设计一个普通函数（不必是虚函数）defaultFly（设置为protected，只要在继承体系中可见即可），在派生类的fly中调用defaultFly就等于是有了默认实现，不调用则没有，直接通不过编译。

以上的fly和defaultFly实际上是以不同的函数分别提供接口和缺省实现，这可能会导致过度雷同的函数名称而引起的class命名空间污染问题，但是接口和缺省实现分开是合理的。
此时就可以利用纯虚函数也可以偷偷实现的特点，在派生类的fly中调用纯虚函数fly的实现，使得函数名字只有一份，并且将接口和缺省实现分离了（不特意设置的话就没有缺省实现了）。

3. 普通函数objectID
普通函数就意味着它并不打算在派生类中有不同的行为，**实际上，一个非虚成员函数所表现的不变性，凌驾于其特异性**：不论派生类多么特异化，它的行为都不可以改变
因而可以总结：声明普通函数是为了让派生类继承函数的接口以及一份强制的实现****
objectID的例子也很有意义：每个Shape对象都有一个用于产生对象识别码的函数，计算方法是死的，派生类不必也不应该改变其行为。

以上三种类型的函数，使得程序员可以精确地指定想要让派生类继承的东西：
1. 只继承接口
2. 继承接口和一份缺省实现
3. 继承接口和一份强制实现

不必担心virtual带来的成本，还是经典的80-20法则，一个程序有80%的时间花费在20%的代码上，那么即便80%的代码用到虚函数，剩余的20%更是举足轻重

# 35 考虑virtual函数以外的其他选择
考虑这么一个场景：
```cpp
class GameCharacter{
public:
    virtual int healthValue() const;    // 返回每个人的健康指数，每个人都是有所不同的，因而是虚函数，而且不是纯虚的，说明有缺省实现
    ...
};
```
有什么问题呢？
先考虑别的实现：
```cpp
// 有人认为：较好的设计是保留healthValue为public成员函数，但让它成为non-virtual，并调用一个private virtual来做实际工作
class GameCharacter{
public:
    int healthValue() const{
        ...
        int retVal = doHealthValue();
        ...
        return retVal;
    }
    ...
private:
    virtual int doHealthValue() const{
        ...
    }
};
```
## 这样的设计被称为：令客户通过public non-virtual成员函数间接调用private virtual函数
**NVI（non-virtual interface）手法**，实际是所谓的***Template Method设计模式***
实际上是把虚函数包装了起来，好处在于：可以在虚函数调用前后，做一些事前工作（锁定互斥量、调用日志记录、验证class约束条件、验证函数先决条件）和事后工作（解锁互斥量...）
其实没有必要让virtual函数一定得是private，某些class继承体系要求派生类在virtual函数的实现内必须调用其基类的兄弟（想要调用一下基类版本的函数），那就得提升函数的可见程度，至少得是protected，有时虚函数甚至一定得是public（基类指针指向派生类对象，而delete的话，如果基类析构不是虚函数，就只会调用基类的析构，而不会调用派生类的析构），那就不能使用NVI手法了


## 借由函数指针实现Strategy模式
NVI手法public virtual函数还是得使用一堆虚函数，不够好。
实际上，人物健康指数的计算与人物类型无关，这样的计算实际是不需要人物的
**Strategy Design Pattern是一种行为型设计模式，它允许在运行时选择不同的算法或行为，从而使得一个类的行为可以根据需要动态地改变，而不需要修改其结构。该模式将不同的算法封装成一系列独立的类，使得它们可以互相替换，而客户端代码则无需关心具体的算法实现。**
```cpp
// 利用函数指针
class GameCharacter;    // 前置声明
// 计算健康指数的缺省算法
int defaultHealthCalc(const GameCharacter& gc);
class GameCharacter{
public:
    typedef int (*HealthCalcFunc)(const GameCharacter&);    // 定义函数指针
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc): healthFunc(hcf){}
    int healthValue() const{
        return healthFunc(*this);
    }
    ...
private:
    HealthCalcFunc healthFunc;  // 函数指针对象
};
```
正因为使用了函数指针，才实现了可以调用不同的函数
同一人物类型之间不同实体可以有不同的健康计算函数，例如：
```cpp
class EvilBadGuy: public GameCharacter{
public:
    explicit EvilBadGuy(HealthCalcFunc hcf = defaultHealthCalc): GameCharacter(hcf){
      ...
    }
};
int loseHealthQuickly(const GameCharacter&);    // 计算函数1
int loseHealthSlowly(const GameCharacter&);     // 计算函数2

EvilBadGuy ebg1(loseHealthQuickly); // 相同类型人物可以调用不同的健康计算函数
EvilBadGuy ebg2(loseHealthSlowly);  // 选择哪个健康计算函数在对象构造时传入
```
而且调用哪个函数是可以在运行期变更的。
这种设计之下，健康计算函数不再是GameCharacter继承体系内的成员函数，因为健康计算函数甚至根本不访问非共有成分。
如果人物的健康需要由非公有成分计算得来，那么这个设计就有问题了。
--> 解决办法是：弱化class的封装，比如class可以声明对应的非成员函数为友元函数，或者为其实现的某一部分提供public访问函数。

## 借助`std::function`完成Strategy模式
玩明白template以及它们对隐式接口的使用的话，函数指针就显得死板了。
```cpp
class GameCharacter;    // 前置声明
int defaultHealthCalc(const GameCharacter& gc);
class GameCharacter{
public:
    typedef std::function<int (const GameCharacter&)> HealthCalcFunc;    // 定义函数指针
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc): healthFunc(hcf){}
    int healthValue() const{
        return healthFunc(*this);
    }
    ...
private:
    HealthCalcFunc healthFunc;  // 函数指针对象
};
// 其中的std::function对象可以保存任何与函数签名兼容的可调用对象（兼容的意思是：入参和出参都可以隐式转换）
```
准确来说，相比函数指针，`std::function`包装的产物可称为是一个指向函数的泛化指针。重点就在于：**入参和出参都可以隐式转换**，显然灵活多了：
因为它甚至可以接收这种可调用对象：`std::bind(&GameLevel::health, currentLevel, _1)`;
当调用这个函数对象时，它会调用currentLevel对象的health成员函数，并将一个参数传递给health函数，这个参数将会被放置在占位符_1的位置。


# 36 绝不重新定义继承而来的non-virtual函数
这会导致基类指针指向派生类对象的情况下，调用这样的函数会发生静态绑定，永远调用基类版本。
之前提到public继承是is-a的关系，那么适用于Base对象的每一件事，也适用于Derived对象，因为每个Derived对象都是一个Base对象
（Base含有一个非虚函数mf）Base的派生类一定会继承mf的接口和实现
逻辑上是有问题的：如果派生类实现了与基类不同的mf函数，那么每一个Derived都是一个Base就不对了
如果D是public继承了B，并实现了与基类不同的mf函数，这个mf函数就无法反映出本身是非虚函数的性质
总之不具备多态性的函数就不要在派生类重新实现

# 37 绝不重新定义继承而来的缺省参数值
本章讨论局限于继承一个带有缺省参数值的virtual函数
**虚函数是动态绑定，缺省参数值是静态绑定**
静态绑定又称前期绑定，动态绑定又称后期绑定
对象的静态类型，就是它在程序中被声明时所采用的类型
```cpp
class Shape{
public:
    enum ShapeColor{Red, Green, Blue};
    virtual void draw(ShapeColor color = Red) const = 0;
    ...
};

class Rectangle: public Shape{
public:
    virtual void draw(ShapeColor color = Green) const;
    ...
};

class Circle: public Shape{
public:
    virtual void draw(ShapeColor color) const;  // 静态绑定，也就是编译期绑定，这个函数无法从base函数继承缺省值
    // 而动态绑定的话，用指针或引用调用此函数，可以不指定参数，因为动态绑定会从base继承缺省参数值
    ...
};

// 考虑以下指针
Shape* ps;  // 静态类型为Shape*
Shape* pc = new Circle;
Shape* pr = new Rectangle;
// 都是基类指针，指向不同的对象
```
**对象的动态类型指的是“目前所指对象的类型”**
动态类型可以表现出一个对象将会有什么行为，以上的pc的动态类型就是Circle*，ps没有动态类型，不过当然可以通过运行时赋值来改变
花样来了：你可能会在调用一个定义于派生类内的虚函数时，却使用基类所指定的缺省参数值
```cpp
pr->draw(); // 动态绑定继承了基类的缺省参数（如果不显式指定），并不是预想的Green！
```
重点在于：draw是个虚函数，而它有个缺省参数值在派生类中被重新定义了
根本原因是：C++追求运行期效率，缺省参数值是静态绑定，便于编译期处理并且效率较高
但是如果你同时提供缺省参数值给基类和派生类，就更糟糕了，代码重新并且有依赖性，改了一处就得改另一处
解决方法是替代设计：NVI（非虚接口）：让基类的public非虚函数调用纯虚函数，在派生类中重新定义private虚函数。
我们可以在非虚函数中指定缺省参数。
```cpp
class Shape{
public:
    enum ShapeColor{Red, Green, Blue};
    void draw(ShapeColor color = Red) const
    {
        doDraw(color);
    }
    ...
private:
    virtual void doDraw(ShapeColor color) const = 0;    // 真正的实现在此
};
class Rectangle: public Shape{
public:
    ...
private:
    virtual void doDraw(ShapeColor color) const;    // 不需要指定缺省参数值
}
```
由于非虚函数绝对不被派生类重写，这个设计就能清晰地使得draw函数的color缺省参数值总是为red

# 38 通过组合关系塑造出has-a或“根据某物实现出”
类图中的组合关系：类A是类B的成员
组合关系意味着：has-a的关系 / 根据某物实现出
软件设计中：
## 对应现实概念的对象（人、汽车）属于应用域部分    -->     has-a的关系
人有一个名字

## 实现域部分：缓冲区、互斥量、查找树      -->     根据某物实现出的关系
例如，想要自己实现set，底层调用list<T>，如果直接继承，那就错了：毕竟list可重复，而set不可重复
此处的关系就是典型的：Set对象可由一个list对象实现出来（即：set的接口封装了对list的操作）


# 39 明智而审慎地使用private继承
private继承：编译器不会隐式地将派生类转为基类，并且派生类中继承的所有成员都是private
意味着：“根据某物实现出”（**意义等同于类的组合关系，但是：尽可能使用组合，必要时才用private继承**。必要的意思是：当protected成员以及virtual函数牵扯进来，或者是内存空间有极端的要求的时候）

private继承在软件设计层面没有意义，只在软件实现层面有意义：private意味着只有实现部分被继承，接口部分被略去，Derived对象根据Base对象实现得来

```cpp
// 考虑以下场景：
// 我们有一个Widget类，需要按照一定频率调用成员函数，有一个现成的Timer类，我们完全可以继承它，但是不能是public，因为显然不能让用户调用到Timer的接口，我们不能让用户轻易地错误使用接口
// 考虑private继承
class Timer{
public:
    explicit Timer(int tickFrequency);
    virtual void onTick() const;
};

class Widget: private Timer{
private:
    virtual void onTick() const;
};
// 但是完全可以使用组合的方式来替代，实现同样的效果
class Widget{
private:
    class WidgetTimer: public Timer{    // PS: 嵌套类 完全可以公有继承，因为是私有对象
    public:
        virtual void onTick() const;
        ...
    };
    WidgetTimer timer;
};
```
## 这样的设计好处何在？
1. 原先的private继承无法阻止派生类（如果有）重新定义onTick函数。
而Widget的派生类无法调用WidgetTimer，也就是不会被继承体系给继承。
```cpp
// 或者也有简单的办法 但是成书时想必没有这个关键字
virtual void fun() final;
```

2. 有利于将Widget的编译依存性降至最低
如果是私有继承，当Widget编译时必须要有Timer的定义，也就是include进来，如果WidgetTimer只作为成员指针，Widget就只需要简单的WidgetTimer的声明（而不需要#include任何与Timer有关的代码），从而实现解耦

特殊情况使用private继承：class不带任何数据（non-static变量、没有虚函数（每个对象会有额外的虚表指针）、没有虚基类）
但是C++规定任何对立对象大小都必须非0，因为C++会默默地安插一个char到空对象内，而且再加上内存对齐，会加上一些padding，导致大小甚至大于char
但是这个约束不适用于派生类的基类成分，因为它们不独立（即派生类的基类成分不做这种内存对齐）
```cpp
class Empty{};

class A: private Empty{
private:
    int x;
}

sizeof(A) == sizeof(int);   // 必然的
```
这就是所谓的空白基类最优化（EBO），而且只有单继承才行
而实际工程中，“empty”class往往含有：typedef、enum、static、non-virtual函数，EBO的广泛实践减少了派生类的大小
但是毕竟大多数类都不是empty class，所以EBO很少成为private继承的正当理由

# 40 明智而审慎地使用多重继承
多重继承显然会因为从多个基类继承相同的函数或者typedef名字,容易造成歧义
编译器不对同名函数做可用性检查:C++解析重载函数调用的规则:
C++首先确认这个函数对此调用是最佳匹配,找到最佳匹配才检验可用性,如果Derived继承了Base1和Base2的同名函数,一个是public,一个是private,那么这两个函数的匹配程度是相同的,没有所谓的最佳匹配,都轮不到可用性的检查.
```cpp
// 必须显式指定调用的是哪个基类的函数
mp.BorrowableItem::checkout();
```
经典的钻石继承:
File被InputFile、OutputFile继承,然后双双被IOFile继承
思考:如果File基类中有个成员变量fileName,那么IOFile中内该有多少这个名称的数据呢?直觉上应该只有一份,这就需要InputFile和OutputFile都是virtual继承自File.
那么看起来,public继承应该总是virtual的比较好,但是为了实现virtual继承,编译器得在幕后操作:导致占据内存比非virtual更大,并且访问成员变量的速度更慢.
virtual继承成本:virtual基类初始化责任由继承体系中的最低层开始，凡是派生自virtual基类的类都必须知道如何初始化虚拟基类.
当你向继承体系中添加一个新的派生类时，这个新类必须负责初始化其继承的虚拟基类，无论这些虚拟基类是直接还是间接继承的。

结论就是:
1. 非必要不使用virtual基类
2. 如果使用virtual基类,尽可能避免在其中放置数据

例子:
```cpp
class IPerson{ 
    ...
    virtual std::string name() const = 0;
    ...
 };

// IPerson的客户使用工厂函数获得指向一个Person对象的指针
// 工厂函数:
std::shared_ptr<IPerson> makePerson(DatabaseID personIdentifier);

// 调用:
std::shared_Ptr<IPerson> pp(makePerson(id));
// 显然工厂函数中,创建了派生自抽象基类的具象class(称之为CPerson)
// 假设PersonInfo类提供了CPerson所需要的实质东西
class PersonInfo{
public:
    ...
    virtual const char* theName() const;
    virtual const char* theBirthDate() const;
    ...
private:
    virtual const char* valueDelimOpen() const; // 虚函数,用于重写各种头限定符版本的函数
};
// theName()需要调用valueDelimOpen()、然后产生name值、valueDelimClose()
// theName()的结果不仅取决于PersonInfo、还取决于PersonInfo的派生类、
```
CPerson和PersonInfo的关系是:PersonInfo提供若干函数帮助CPerson实现,即:根据某物实现出的关系
此处因为要重新定义virtual函数,所以private继承是必要的,单纯的组合关系是无法应付的.
CPerson必须继承IPerson接口以重写,因此就有了多重继承
```cpp
class IPerson{
public:
    virtual ~IPerson();
    virtual std::string name() const = 0;
    ...
};

class DatabaseID{...};

class PersonInfo{
public:
    explicit PersonInfo(DatabaseID pid);
    virtual const char* theBirthDate() const;
    virtual const char* valueDelimOpen() const;
    ...
};

class CPerson: public IPerson, private PersonInfo{
public:
    explicit CPerson(DatabaseID pid): PersonInfo(pid);
    virtual std::string name() const
    {
        return PersonInfo::theName();
    }
private:
    const char* valueDelimOpen() const {return "";}
};
/*
CPerson继承IPerson是因为：
public继承IPerson这一接口类

CPerson继承PersonInfo是因为：
要重写继承而来的虚函数（PersonInfo属于是协助实现的class，重写的接口都是私有的）
*/
```

# 41 了解隐式接口和编译期多态
PS：c++ template产生的动机：建立**类型安全**的容器（vector、list、map），除了容器，template还可以实现泛型编程（例如STL算法：for_each、find、merge）
而由于template图灵完备，还可以实现模板元编程（即：在编译期在编译器执行，在编译完成时停止执行）

显式接口和运行时多态是很常见的：
```cpp
class Widget{
public:
    Widget();
    virtual ~Widget();  // 意味着运行时多态
    virtual void normalize();
    ...
};

void doProcessing(Widget& w){   // 入参类型明确指定为Widget&类型，那么w就必然支持Widget的接口，是显式接口，因为它的实现在源码中明确可见
    ...
    Widget temp(w);
}

// 泛型编程和面向对象有所不同
template<typename T>
void doProcessing(T& w){
    ...
    T temp(w);
}
// 现在w不是明确的类型了，现在w必须支持的接口缩小了，只要最终能够支持doProcessing内部的操作就行了（这些接口被称之为隐式接口）
```
注意，凡是涉及到w的任何函数调用吗，operator->等，都有可能造成template实例化，这种实例化发生在编译期
**编译期多态**：用不同的template参数实例化函数模板，导致调用不同的函数
显式接口由函数的签名式（函数名称、参数类型、返回类型）构成，一个类的public接口可以包括：构造、析构、基本函数的参数类型 / 返回类型、常量性，编译器产生的copy构造、typdefs
**隐式接口**：由**有效表达式**组成
```cpp
template<typename T>
void doProcessing(T& w){
    if(w.size() > 10 && w != sth){  // T的隐式接口看似有这些约束：提供size()函数并返回一个整数值，支持operator!=函数
        ...
    }
    ...
}
```
但是由于**操作符重载**的存在，实际上隐式接口不必满足以上两约束：
T必须支持size()成员函数，但是这个函数可能是由基类继承而来，T类型本身未必要有该函数，并且该函数不必返回一个整数类型，只要能返回一个类型为X的对象，能够支持operator>即可，而且，实际上，operator>只要能接收Y类型，只要能够隐式类型转换X为Y
**隐式接口仅仅是由一组有效表达式构成，表达式再怎么复杂，但他们要求的约束条件很直接**
```cpp
if(w.size() > 10 && w != sth)
    ...
// 实际上if内部的表达式无论涉及什么实际类型，最终都必须与bool兼容，这是doProcessing模板函数对T的隐式接口的要求之一
```

# 42 了解typename的双重意义
在声明template类型参数时，class和typename的意义完全相同
而typename可以暗示参数并非必须是class类型，内建类型也行
而二者也有不等价的时候，有时必须得用typename：
```cpp
template<typename C>
void print2nd(const C& container){
    if(container.size() >= 2){
        C::const_iterator iter(container.begin());
        ++iter; // 该局部变量的类型是C::const_iterator，实际的类型是取决于模板参数的，专业术语叫做嵌套从属类型名称
        int value = *iter;  // 明确的int类型，所谓的非从属名称
        std::cout << value;
    }
}
```
**嵌套：即又是一个类**
**从属：template中出现的名称依赖于某个template参数**
嵌套从属类型名称可能导致解析困难
```cpp
template<typename C>
void print2nd(const C& container){
    C::const_iterator* x;   // 此处x的类型可能有歧义：C如果有个static成员变量叫做const_iterator，或者x是全局变量，那就变成相乘操作了
    // 在确定C的类型前，C::const_iterator都是意味不明的
    ...
}
```
C++有个规则可以解析这个歧义状态：如果解析器在template中遇到一个嵌套从属名称，会**假设这名称不是个类型，除非明确地告诉它是**
即：缺省情况下嵌套从属名称不是类型
```cpp
template<typename C>
void print2nd(const C& container){
    ...
    C::const_iterator iter(container.begin());  // 直接认为C::const_iterator不是类型，那么也就无法通过编译
    ...
}

// 想要通过编译需要明确地指定它是一个类型
    typename C::const_iterator iter(container.begin());

typename<typename C>
void f(const C& container, typename C::iterator iter);
// 第一个入参前面是不能加typename的，因为C并非依附任何template参数
// 第二个入参必须要加typename，它就是所谓的嵌套从属类型名称
```
typename不能出现在基类的list中，也不能在成员初值列表中修饰基类
```cpp
template<typename T>
class Derived: public Base<T>::Nested{  // 基类list，不允许加typename
public:
    explicit Derived(int x): Base<T>::Nested(x){    // 初始化列表
        typename Base<T>::Nested temp;
        ... // 可以加上typename修饰
    }
    ...
}
```

```cpp
template<typename IterT>
void workWithIterator(IterT iter){
    typename std::iterator_traits<IterT>::value_type temp(*iter);   // 创建了一个局部变量
    // 如果IterT是vector<int>::iterator，temp就是int类型
    // std::iterator_traits<IterT>::value_type是个嵌套从属类型名称，因为value_type嵌套于std::iterator_traits<IterT>之内，并且依赖于IterT这个模板参数
    ...
    // 这样的写法太长，经常用typedef：
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    value_type temp(*iter);
    ...
}
```

# 43 学习处理模板化基类内的名称
***面对涉及到基类成员的无效代码，编译器的诊断时间可能在解析派生类模板的定义时，也可能在类模板实例化后，但是C++的策略是尽早诊断***
程序：在编译期判断将信息传递到哪一家公司
```cpp
class MsgInfo{...};
class CompanyA{
public:
    ...
    void sendCleartext(const std::string& msg);
    void sendEncrypted(const std::string& msg);
    ...
};

template<typename Company>
class MsgSender{
public:
    ...
    void sendClear(const MsgInfo& info){
        ...
        Comapny c;
        c.sendCleartext(msg);
    }
    sendEncrypted也类似;
};

// 如果想要在每次送出信息时记录某些log
template<typename Company>
class LoggingMsgSender: public MsgSender<Company>{
public:
    ...
    void sendClearMsg(const MsgInfo& info){
        传送前的信息写入log;
        sendClear(info);    // 注意，这行代码会导致无法通过编译：编译器认为该函数不存在
        传送后的信息写入log;
    }
    ...
};
```
重点来了：LoggingMsgSender是模板类，它根本不知道继承的是什么类（实际上是未经实例化的MsgSender<Company>）
```cpp
// 假设CompanyZ只有sendEncrypted，因而不能作为MsgSender的类型参数传入
class CompanyZ{
public:
    ...
    void sendEncrypted(const std::string& msg);
    ...
};

// 可以特化出一个MsgSender
template<>  // 模板全特化
class MsgSender<CompanyZ>{
public:
    ...
    void sendSecret(const MsgInfo& info){
        ...
    }
};

// 此时，考虑LoggingMsgSender，类型参数不能是CompanyZ，因为没有提供sendClear函数
```
**C++规定：在template C++编程时，会拒绝模板化基类中寻找继承而来的名称（函数）**，因为：基类模板可能特化，特化出的版本很可能不提供和一般性模板相同的接口
为了实现目的，我们必须让c++进入模板基类观察接口
1. 在调用基类函数之前加上this->
```cpp
    this->sendClear(info);  // 可以通过编译
```

2. using声明式
```cpp
using MsgSender<Company>::sendClear;    // 向编译器强调：sendClear是位于基类内部的，编译器本来是不会进入模板基类查找东西的，但是强调一下就会去找了
```

3. 明确指出被调用的函数位于基类
```cpp
MsgSender<Company>::sendClear(info);
// 该方法有缺陷：如果调用的是虚函数，这样的explicit qualification会关闭virtual绑定行为
``` 
**三种方法本质上都是对编译器承诺：模板基类的任何特化版本都支持其一般版本所提供的接口**
```cpp
// 
LoggingMsgSender<CompanyZ> zMsgSender;  // 注意类型参数是CompanyZ
MsgInfo msgData;
...
zMsgSender.sendClearMsg(msgData);   // 无法通过编译，因为编译器知道基类是特化版本CompanyZ，并且不提供sendClear函数
```

# 44 将与参数无关的代码抽离templates
templates显然节省了代码量
类模板的成员函数只有被使用时才会实例化
但有时使用模板可能导致代码膨胀，即便源码看似整齐，目标码可能还是臃肿的
我们需要做的是：**共性与变性分析**
在编写若干函数时，我们会自然而然地把函数之间共同的部分放在单独的函数里面，类也是类似的道理
模板也是类似的，需要注意的是：模板的代码之间，重复是隐晦的
```cpp
template<typename T, std::size_t n>
class SquareMatrix{ // 元素类型为T的n*n矩阵
public:
    ...
    void invert();  // 求逆
};  // PS：n其实是非类型参数

// 如果是这样调用：
SquareMatrix<double, 5>sm1;
sm1.invert();

SquareMatrix<double, 10>sm1;
sm2.invert();
// 除了常量5和10，两个函数的其他部分完全相同

// 如果这样修改：
template<typename T>
class SquareMatrixBase{
protected:
    ...
    void invert(std::size_t matrixSize);
};

template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T>{
private:
    using SquareMatrixBase<T>::invert;
public:
    ...
    void invert(){
        this->invert(n); // 调用基类版本的invert
    }
};
// 这样调用造成的额外成本是0，因为是inline函数
// 使用this->是因为C++的设定：模板化基类内的函数名称会被派生类直接忽视
// private继承是因为这里的基类只是为了帮助派生类的实现，并非is-a的关系
```
但是当前设计存在问题：基类的invert函数不知道操作的数据在哪里（即未能拿到矩阵的内存地址而是只知道一个矩阵大小）
因而可能要传入一个指针给invert函数，也就是增加一个入参，但是这不好，会出现多次告诉基类相同的信息的情况，那如果让基类自己持有那么一个指向矩阵的指针呢？？
```cpp
template<typename T>
class SquareMatrixBase{
protected:
    SquareMatrixBase(std::size_t n, T* pMem): size(n), pData(pMem){}
    void setDataPtr(T* ptr){pData = ptr;}

private:
    std::size_t size;
    T* pData;
};

// 派生类
template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T>{
public:
    SquareMatrix():SquareMatrixBase<T>(n, data){}   // 初始化列表以传值

private:
    T data(n*n);    // 内存会分配在栈上，但是可能很大，放在堆上会更好
};

// 动态分配版本，如下：
template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T>{
public:
    SquareMatrix():SquareMatrixBase<T>(n, 0), pData(new T[n*n])
    {this->setDataPtr(pData.get())};    // 返回一个裸指针

private:
    boost::scope_array<T> pData;    // 等价于指向数组的智能指针
};
```
但是代价是什么呢？
对于入参为空的invert函数（尺寸在模板实例化时传入），**编译器可能会生成比把尺寸作为参数传入的invert更好的代码**，因为尺寸是个编译期常量（入参的方式则是运行时才传入），借助常量的广传达到最优化，最终可以折成指令成为直接操作数
另外，不同大小的矩阵只有一个版本的invert，可以减少可执行文件大小，降低**工作集**（即一个进程所使用的一组内存页）大小，强化指令cache内的引用集中化。
到底哪种更好，都尝试下并观察平台的行为以及面对代表性数据时的行为。
另一个效率关心的是对象大小，前面的例子里：每一个SquareMatrix对象都有一个指针指向基类内的数据，导致每个派生类多了一个指针的体积。如果让基类持有一个指针指向矩阵，则会失去封装性，并且导致资源管理的混乱。
此处重点辩论的是：非类型模板参数带来的膨胀，类型参数页会带来膨胀。
例如很多平台上，int和long底层是一样的，那么vector<int>和vector<long>就是重复的，有的链接器会合并完全相同的函数实现代码。大多平台上，指针都具有相同的底层实现。



# 45 运用成员函数模板接受所有兼容类型
比如智能指针，就是“行为像指针”的对象，相比内置指针，提供++运算符，就很方便。
内置指针支持隐式转换，这是极好的，派生类指针可以隐式地转换为基类指针、指向非const对象的指针可以转为指向const对象的指针
```cpp
class Top{...};
class Middle: public Top{...};
class Bottom: public Middle{...};
Top* pt1 = new Middle;  // Middle*类型的指针转为Top*
Top* pt2 = new Bottom;  // Bottom*转为Top*  转向基类指针肯定没问题
const Top* pct2 = pt1;  // Top*转换为const Top*

// 如果是智能指针
SmartPtr<Top> pt1 = SmartPtr<Middle>(new Middle);   // 智能指针也是要以内置指针来初始化的
SmartPtr<Top> pt2 = SmartPtr<Bottom>(new Bottom);
SmartPtr<const Top> pt2 = pt1;

// 此处SmartPtr<Top>和Smart<Middle>并没有继承关系，是两个完全不同的类
```

## Templates和泛型编程
***如果想让以上两个SmartPtr之间能够像内置指针那样转型，就得在模板加逻辑***
准确来说，需要添加成员函数模板
```cpp
template<typename T>
class SmartPtr{
public:
    template<typename U>
    SmartPtr(const SmartPtr<U>& other);
// 对于任何的类型T和U，可以根据SmartPtr<U>生成SmartPtr<T>
// 这样的构造函数可以称为：泛化copy构造函数
// 它不需要声明为explicit，因为原始指针类型之间的转换是隐式的，无需写清楚转型动作
};
```
由于基类指针转派生类指针是不对的，也没有类似于int*转为double*的隐式转换行为，因此要对这一成员函数进行拣选。
```cpp
// 如何约束这种转换行为
template<typename T>
class SmartPtr{
public:
    template<typename U>
    SmartPtr(const SmartPtr<U>& other): heldPtr(other.get()) {...}
    T* get() const {return heldPtr;}
// 使用初始化列表。其中做了隐式转型，这就是我们想要的，可以隐式转型才能通过编译
// 即：这个构造函数现在只有实参能够兼容才能通过编译

    private:
        T* heldPtr;
};

// share_ptr支持多种智能指针的构造
template<class T>
class share_ptr{
public:
    template<class Y>
        explicit shared_ptr(Y* p);
    template<class Y>
        explicit shared_ptr(shared_ptr<Y> const& r);
    template<class Y>
        explicit shared_ptr(weak_ptr<Y> const& r);
    template<class Y>
        shared_ptr& operator=(shared_ptr<Y> const& r);
// 只有这个不是explicit，允许将某个shared_ptr类型隐式转换成另一个shared_ptr
// 但不允许将内置指针或者其他类型的智能指针隐式转为shared_ptr，必须显式
};

成员函数模板不改变以前的设定：编译器生成四个成员函数（包括copy构造、copy赋值）（如果没有声明的话），而且如果T和Y类型相同，就等价于正常的copy构造函数了
```


# 46 需要类型转换时请为模板定义非成员函数
如24条所说，比如有理数相乘的函数就得定义为非成员，否则像2*ration1这种就无法实现
```cpp
template<typename T>
class Rational{
public:
    Rational(const T& numerator = 0, const T& denominator = 1); // 传引用，因为传引用足以解决问题
    const T numerator() const;  // 传值，是因为创建一个新的对象是必要的
    const T denominator() const;    // 尽量声明const，可以减少错误
    ...
};

// 非成员函数
template<typename T>
const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs)
{...}

// 这部分代码为什么不能通过编译？
Rational<int> oneHalf(1, 2);
Rational<int> result = oneHalf * 2; // 右半部分是错误的
// 实际上调用的是：
Rational<int> result = operator*(oneHalf, 2);   // 编译到此处时，编译器会去找某个“名为operator*并接受两个Rational<T>参数”的函数
/*  而为了推导T，编译器会去查看传入的实参类型：Rational<int>, int
    根据第一实参，可以推导出T是int
    但是根据第二实参，就难了，编译器不会使用Rational<int>的隐式构造函数把2转换为Rational<int>，隐式类型转换函数不是模板实参推导过程中该做的
*/
```
现在问题的关键就是template实参推导
有种办法：**template class内的friend声明式可以指涉某个特定函数**
比如：class Rational<T>可以声明operator*是它的一个friend函数
只有函数模板依赖模板实参推导，编译器总是能在class Rational<T>实例化时得知T
```cpp
template<typename T>
class Rational{
public:
    friend const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs);
};
// 实现:
template<typename T>
const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs)
{...}

// 为什么声明为友元就能通过编译了呢?
```
**因为友元声明在类里面,当模板类得到参数实例化时,友元函数也一并实例化了,它就不是模板函数,而是一个函数了**
但是这段代码虽然可以通过编译,但是无法通过链接

类模板中,可以写Rational<T>,而不是Rational(二者都可以)
之所以这样能通过编译,是因为编译器找到了函数声明,但是这个函数只声明在Rational内部,没定义出来,而我们想让类外部的template operator提供定义是做不到的
上面的代码看似类外部也定义了模板函数,但是没能够在类得到类型参数的同时实例化
那么声明的同时完成定义,可以解决问题
此处对friend的使用并不是为了访问类的非public部分

由于定义在类内部的函数都偷偷成为inline函数,包括上面的operator*
如果想要把inline声明带来的冲击最小化,可以让operator*只调用类外部的辅助模板函数(照样可以把类型参数传进去)



# 47 请使用traits classes表现类型信息
STL主要由表现容器,迭代器和算法的template构成
```cpp
// 有这么一个工具函数
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d);     // 将迭代器向前移动d单位
// 也就是做 iter += d; 的操作
```

**STL迭代器分类:**
1. input迭代器 
只能向前移动    只可以读取指向的对象,只能读一次
类似于指向输入文件的阅读指针
举例:istream_iterator

2. output迭代器

以上两种都只适合一次性操作算法

3. forward迭代器
可以做到以上两种迭代器能做的每一件事,并且不是一次性的
是**单向的**

4. Bidirectional迭代器
**双向的**
list,set,multiset,map的迭代器

5. 功能最多的是random access迭代器
底层实现采用了内置指针
可以执行迭代器算术
vector, deque, string

对于以上分类,C++提供专属的tag来确认:
例如:
```cpp
    struct random_access_iterator_tag: public bidirectional_iterator_tag{};
    // is-a的继承关系

// advance函数实现如果采用random access就只耗费常量时间
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d){
    if(iter is a random access iterator){   //! 需要判断类型,需要在编译期取得类型信息
        iter += d;
    }
    else{
        if(d >= 0){
            while(d--)
                ++iter;
        }
        else{
            if(d >= 0){
                while(d--)
                    ++iter; // 非random access iterator只能反复调用++
            }
            else{
                while(d++)
                    --iter;
            }
        }
    }
}
```
Traits是一种技术,是一种C++程序员遵守的协议
其要求是:**内置类型和用户自定义类型必须表现的一样好**
比如:上述advance如果接收一个指针,如const char*,也可以运行
那么类型信息就没法嵌套在类型内部了
解决办法是:
把traits放在一个template以及其特化版本中,标准库就有若干个这样的templates
针对迭代器的被命名为iterator_traits,虽然定义成struct,但是往往被称为traits class
在struct iterator_traits<IterT>一定声明着某个typedef名为iterator_category,用于确认IterT的迭代器分类

每一个用户自定义的迭代器类型必须嵌套一个typedef,名为iterator_category,用于确认tag struct
```cpp
template<...>
class deque{
public:
    class iterator{
        public:
            typedef random_access_iterator_tag iterator_category;
            ...
    };
};

// 响应定义在iterator内部的typedef
template<typename IterT>
struct iterator_traits{
    typedef typename IterT::iterator_category iterator_category;
    ...
}

// 为了支持内置指针,专门偏特化
template<typename IterT>
struct iterator_traits<IterT*>  // 针对内置指针
{
    typedef random_access_iterator_tag iterator_category;
}
```
实现一个traits class的流程:
1. 确认可取得的类型信息的集合
2. 确定该类型信息的名字
3. 提供一个template和一组特化,内含类型信息

然后如何在编译期做类型判断呢?
C++提供了重载的办法,因为重载时,编译器是根据实参类型,选择最合适的重载函数

```cpp
template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, std::random_access_iterator_tag){
    ...
}

template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, std::bidirectional_iterator_tag){
    ...
}

template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, std::input_iterator_tag){
    ...
    // d若小于0,则直接抛出异常
}

// 由于forward_iterator_tag继承自input_iterator_tag,因此第三个重载也能处理forward迭代器
// 这是iterator_tag struct继承关系带来的好处

// 在advance函数中调用上述的doAdvance函数
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d){
    doAdvance(iter, d, typename std::iterator_traits<IterT>::iterator_category());
}
```
其他迭代器相关信息也需要提供,比如value_type, char_traits(保存字符类型的相关信息), numeric_limits(保存数值类型的相关信息)


# 48 认识template元编程
***Template metaprogramming，TMP***
指的是编写执行于编译期的代码，模板元编程的程序执行完毕，产生输出：template实例化出来的C++源码
TMP的好处：
1. 有些事情只有TMP才能完成
2. 使得有些错误在编译期就能侦测到
把一些工作放在编译期：减小可执行文件大小、运行时间较少、减少内存占用。但是编译时间变长。
TMP是Turing-complete的

47提到的traits就是TMP
由于typeid只能在运行时判断，因此编译时是无视if判断中的typeid的（编译器会确保所有源码都有效）

TMP主要是“函数式语言”，没有循环，只有递归（递归模板实例化）
```cpp
#include<iostream>

template<unsigned n>
struct Factorial{
    enum { value = n * Factorial<n-1>::value };
    // 匿名enum，不会引入额外的类型名，在同一作用域（比如Factorial内部就可以使用）
};

template<>
struct Factorial<0>{
    enum{value = 1};
};

int main(){
    std::cout << Factorial<3>::value << std::endl;
}
// 每个struct都有个value，如果是循环的逻辑，每次循环都会value，但是现在这种递归模板实例化就不会
```
## TMP到底有什么用？
1. 确保度量单位正确（在编译期就做判断）
2. 减少临时对象  -->    优化矩阵运算（例如：m1*m2*m3的过程中，每一次operator*都产生一个临时对象），这又被称为expression templates（表达式模板）
3. 有利于设计模式的实现（35：Strategy模式、Observer模式、Visitor模式）

策略（Policy）基础设计（**Policy-based Design**，可以生成客户定制代码，也可以用来避免生成对某些特殊类型不合适的代码）是C++编程中的一种设计模式，它允许你在类或组件中引入可配置的策略，以便在不修改核心类的情况下改变其行为。这种设计模式通过将可变的部分（策略）与不变的部分（核心功能）分离，提高了代码的灵活性和可维护性。
策略是一组实现特定功能的类或对象，它们定义了某个方面的行为。核心类或组件包含一个或多个策略对象，并在运行时选择或配置使用哪个策略。这使得在不修改核心类的情况下，可以轻松地更改或扩展系统的行为。
一个经典的例子是C++的STL容器，如**std::vector**，它使用了**分配策略**来管理内存分配和释放。你**可以在编译时或运行时选择不同的分配策略**，而std::vector的核心功能保持不变。
策略基础设计的优点包括：
灵活性：可以根据需要轻松地更改组件的行为，而无需修改核心代码。
可维护性：策略模式将可变的部分封装在策略类中，使得代码更易于理解和维护。
可复用性：策略类可以在不同的上下文中重复使用，从而提高代码的可重用性。
可测试性：由于策略可以独立测试，因此可以更容易地编写单元测试。
总之，策略基础设计是一种C++编程中的设计模式，它通过将可变行为与核心功能分离，提高了代码的灵活性和可维护性。它在许多C++库和框架中都有广泛的应用。
Policy-based Design是generative programming的一个基础。

TMP编程由于是新兴的，不易调试。



# 49 了解new-handler的行为
## 前言
C++允许手工管理内存，程序员以此可以获得系统最佳时空间效率。
了解**内存的分配和归还**显然是很重要的。
当operator new无法满足客户的内存需求时，调用new-handler
在多线程环境下，进行内存管理是困难的。堆区资源需要适当的同步控制
operator new和operator delete只适合分配单一对象，对于数组采用operator new[]
STL容器使用的heap内存是由容器所拥有的allocator对象管理，而不是被new和delete直接管理

当operator new无法满足某一内存分配需求时，就会抛出异常，但在真正抛出异常之前，它会**先调用一个客户指定的错误处理函数**，即所谓的new-handler。用户如何指定new-handler呢？
```cpp
namespace std{
    typedef void (*new_handler)();  // 声明函数指针
    new_handler set_new_handler(new_handler p) throw(); // 表示不抛出异常
}

// 这样调用：
std::set_new_handler(outOfMem);
int* p = new int[1000000];
```
当operator new无法满足内存申请时，会不断调用new handler直至找到足够内存
一个设计良好的new-handler必须做到：
1. 让更多内存可被使用
一开始就分配一块大内存，执行new handler时归还

2. 安装另一个new-handler
调用set_new_handler换个别的handler

3. 置空new-handler
使得operator new在内存分配不成功时抛出异常

4. 抛出bad_alloc及其派生的异常
operator new函数不会捕捉到该异常

5. 不返回
abort或exit退出

如何实现针对不同的类，new失败时调用不同的new-handler？
```cpp
class Widget{
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();    // set新的，返回旧的
    static void* operator new(std::size_t size) throw(std::bad_alloc);
private:
    static std::new_handler currentHandler;
};
// static成员必须在class外部定义，毕竟静态成员不属于对象，在给对象分配内存时，不必给静态成员分配内存
std::new_handler Widget::currentHandler = 0;

// operator new的实现：
// 需要借助一个资源管理类
NewHandlerHolder，构造时把std::new_handler传入，赋给private成员
析构时，调用std::set_new_handler，把private成员当前的new_handler
把拷贝构造声明为private变量

void* Widget::operator new(std::size_t size) throw(std::bad_alloc){
    NewHandlerHolder h(std::set_new_handler(currentHandler))；
    return ::operator new(size);
}

// 这样调用：
void outOfMem();
Widget::set_new_handler(outOfMem);

// 实际上可以把以上操作都放在模板类里面
class Widget: public NewHandlerSupport<Widget>{
...}

// 这种模板类被称为CRTP：怪异的循环模板模式
```
使用noexecept的new来创建Widget，虽然new本身不会抛出异常，但是构造函数内部还有可能调用抛出异常的new。


# 50 了解new和delete的合理替换时机
什么情况才要替换？
1. 用来检测运用上的错误
常见错误：delete操作时失败导致内存泄漏、delete第二次会导致未定义行为
如果让new中持有地址，delete将其地址置空，就能检测以上行为

有时编程错误会导致数据分配在内存之外的区域、或者之前的区域
如果自定义的new分配内存时就分配超额的内存，额外空间用于存放特定的信息，delete时检查它，若有改变则说明发生了内存越界。

2. 为了强化性能（增加分配和归还的速度）
new和delete是复杂的，毕竟有内存碎片，要考虑持续的分配和归还。因而默认版本的实现是比较通用的，对特定情况表现未必好。

3. 为了收集使用上的统计数据
new在分配动态内存时，分配的机制可能需要记录（分配区块的顺序、大小等）。

```cpp
// 自定义new示例
static const int signature = 0xDEADBEEF;
typedef unsigned char Byte;
void* operator new(std::size_t size) throw(std::bad_alloc){
    using namespace std;
    size_t realSize = size + 2 * sizeof(int);   // 额外放两个int
    void* pMem = malloc(realSize);
    if(!pMem)   throw bad_alloc();

    // 将signature写入内存的最前和最后
    (*static_cast<int*>(pMem)) = signature;
    *(reinterpret_cast<int*>(static_cast<Byte*>(pMem) + realSize - sizeof(int)) ) = signature;  // 按照字节数取到位置

    return static_cast<Byte*>(pMem) + sizeof(int);  // 返回第一个signature之后的位置
}

```
以上代码有缺陷：**“对齐”**
许多计算机体系结构要求特定的类型必须放在特定的内存地址，比如：指针的地址必须是4的倍数（4-byte aligned）
double的地址必须是8倍数，如果不满足的话，可能会导致运行时异常，或者是效率比较低
**C++要求所有的operator new返回的指针都有适当的对齐（取决于数据类型）**
而malloc就是这样工作的，所以让operator new返回一个来自malloc的指针是安全的
“内存对齐”这种技术细节（可移植性、线程安全性...），正可以区分出优秀的内存管理器，不是能跑就完事了

而实际情况下，很多编译器在内存管理函数中可以切换debug和log状态，因而一般不用手写内存管理
boost中的内存管理对“分配大量小内存对象”实现得比较好

4. 还可以降低缺省内存管理器带来的空间额外开销
通用的内存管理器常常比定制的使用更多的内存

5. 为了弥补缺省分配器中的非最佳齐位

6. 为了将相关对象成簇集中
尽可能地把数据都放在一个内存页，减少缺页错误的频率

7. 实现非传统的功能
比如针对SHM实现一版


# 51 编写new和delete时需固守常规
## operator new
实现一致性operator new必须返回正确的值
内存不足时必须调用new handling函数
还要考虑零内存需求
避免掩盖正常的new（准确来说这条是接口要求）

返回值无法是：
可以申请内存，则返回指向那块内存的指针
不行的话，则抛出一个bad_alloc异常

但是实际上 operator new并不只是分配一次内存，每次分配失败后会调用new-handling函数（假设它能执行某些操作使得内存释放），只有当指向new-handling函数的指针是null，operator new才会抛出异常

奇怪的是：**C++规定：即使客户要求0bytes，operator new必须返回一个合法指针**
```cpp
void* operator new(std::size_t size) throw(std::bad_alloc)
{
    using namespace std;
    if(size == 0){
        size = 1;   // 视作申请1byte
    }
    while(true){    // 要不分配成功、要不new-handling完成了自己的义务：让更多内存可用、安装另一个new-handler、卸载new-handler、抛出bad_alloc异常，或者直接承认失败return
        尝试分配size bytes;
        if(分配成功)
            return 一个指向某块内存的指针;
        // 分配失败 找出目前的new-handling函数
        new_handler globalHandler = set_new_handler(0); // 返回旧的，只能set(0)这样取得旧的
        set_new_handler(globalHandler);

        if(globalHandler)
            (*globalHandler)();
        else
            throw std::std::bad_alloc();

    }
}

```
还需要考虑：operator new成员函数可能被派生类继承，然而实际上定制内存管理器的目的是为某特定class的对象分配内存提供最优化，而不是其派生类。
即：针对类X设计的operator new，其行为很典型地只为大小为sizeof(X)的对象设计
如果operator new被继承到派生类中，它可能会被用于分配派生类对象
```cpp
Derived* p = new Derived;   // 调用Base::operator new（假设派生类没声明operator new）

// 对于这种情况，可以在基类new函数里增加判断
if(size != sizeof(Base))
    return ::operator new(size);
// 而size == 0的情况被转移到底层的operator new中判断了
// C++设定，任何独立对象都必须有非零大小
```

### 对于array
#### operator new[]的要点：
要注意基类的operator new[]可能被继承，因此每个对象占据的空间是不确定的

#### operator delete：
保证：删除null指针永远安全
检查是否是空指针，如果是，直接返回
在基类版本的delete函数中，判断如果入参size和sizeof(Base)不等，则令标准delete处理操作
如果派生类对象将被删除，而基类中未定义virtual析构函数（那么就不会调用正确的析构函数），那么传入operator delete的size_t则不正确。


# 52 写了placement new也要写placement delete
new一个对象，会调用`operator new`和`类的构造函数`。如果new成功，而构造函数抛出异常，那么**new操作分配的内存必须取消**，要不然就内存泄漏了。

如果new带参数，delete也要对应，
常规的new，以`size_t`类型为入参
如果`operator new`有额外的入参重载（即所谓的placement new，指将构造的对象分配在指定的内存空间上），

```cpp
#include<new>
void operator new(std::size_t, void* pMemory) throw();  // 特殊入参：指向构造出来的对象的内存，可以指定对象构造在某块内存
// 谈及placement new，就是指这个版本

// 考虑以下情况
Widget* pw = new (std::cerr) Widget;    // cerr为其ostream实参
// 想要让系统在new成功，而构造失败时，把内存回收，只能通过调用相同参数（类型、个数都相同）的delete

// 如果这样调用
delete pw;  // 调用的是普通版本的new，而非placement版本，因为其只有在调用到placement new的构造函数发生异常时才调用
// 显然，对一个指针调用delete，是不会调用placement版本delete
```

还需要注意：由于成员函数的名字会掩盖**外围作用域**中的同名函数
比如Base类中定义了`operator new(std::size_t, std::ostream& logStream)`，派生类调用new的时候，就只能调用这个基类的版本，而不能调用普通版本了。
同样地，`派生类中的new`还会掩盖继承而得的new。
如果还想要调用普通的new：可以在类内部再封装。



# 53 不要轻忽编译器的警告
虽然警告的要紧程度低于错误，但也要注意，比如以下情况：
B类中定义了一个const虚函数，在D中想要重写该虚函数，但是**忘记声明为const**
编译器会警告：D中的f()隐藏了虚函数：B::f()
（内层作用域优先于外层作用域）
意思是：声明于B中的f，不仅没在D中重新声明，而是整个遮掩了（这个是重点）
如果忽视了这条编译器早已发现的警告，就可能要调试半天
警告和编译器是相关联的


# 54 熟悉TR1在内的标准程序库
TR1：Technical Report1

C++98标准程序库：
1. STL
2. Iostream
3. wchar_t、wstring以支持unicode
4. 数值处理：complex、valarray
5. 异常阶层体系
6. C89标准程序库

TR1：
1. 智能指针
2. function
3. bind
4. hash table
5. 正则表达式
6. 变量组
7. array
8. mem_fn
9. reference_wrapper
10. 随机数
11. 数学函数
12. C99兼容扩充
13. type traits
14. tr1::result_of


# 55 让自己熟悉boost
C++开发者集结的社群，自由下载
http://boost.org

boost相比其他组织，有两大优势：
1. 和C++标准委员会之间关系密切，boost就是C++新标准的预备役
2. 公开的同僚review


