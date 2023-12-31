# 前言
传统编译器被分离成多个部分：type checking、simplification、code generation。
simplification用于转换内部的程序表现，即代码优化
涉及三种转换
1. 与特定编译器息息相关的转换

2. 语言语意转换
    构造/析构的扩展、拷贝/赋值、成员逐一初始化、对深拷贝的支持、类型转换操作、临时对象、构造和析构的调用

3. 程序代码和对象模型的转换
    虚函数，虚基类，继承、new和delete、class对象数组、局部静态类实例、非常量表达式的全局对象的静态初始化

PS：**浅拷贝**，对于类成员变量为指针的情况，直接进行指针的赋值
    **深拷贝**，对于以上情况，会调用指针的类型的构造函数，拷贝一份内存数据

编译器合成默认构造函数是用浅拷贝的
但是以下情况会生成深拷贝的构造函数：
1. 当class内带一个member object，而其class有一个copy assignment operator时
2. 当一个class的base class有一个copy assignment operator时
3. 当一个class声明了任何virtual functions(我们一定不能够拷贝右端class object的vptr地址，因为它可能是一个derived class object)
4. 当class继承自一个virtual base class(不论此 base class 有没有 copy operator)时


在结构化语言中，函数调用背后有着**堆栈建立、参数排列、返回地址、堆栈清除等幕后机制**，还算能理解。
但对于面向对象语言，由于编译器帮我们做了太多操作，让人困惑。比如：构造函数、析构、虚函数、继承、多态，有时编译器还会合成一些额外的函数，有时编译器又会扩张我们的函数内容，放进更多的操作，有时往对象加入填充字节，影响`sizeof`的结果。


## 什么是C++对象模型
1. 语言中直接支持面向对象程序设计的部分
2. 对于各种支持的底层实现机制

**“不变量”概念**：
例如：C++ class的完整virtual函数在编译期就固定了，运行时无法动态增加虚函数 --> 使得虚拟调用有快速的dispatch结果，但牺牲了运行期的弹性

了解底层实现模型，才能写出高效的C++代码
本书还旨在去除一些对于C++的错误认识。
以静态初始化为例：
一个类在main函数外面被初始化，这就是通过**静态初始化**做到的，它的实现依赖于开发环境对此的支持属于哪一个层级。



# 第一章 关于对象
C++相比C提供了面向对象的思维方式，相比C语言完全可以用**对象封装**，甚至可以利用**模板**来实现类型参数。
## 那么C++加上封装的代价是什么？
以`class Point3d`为例，相比C语言版本`struct Point3d`的实现，并没有增加成本，而且成员函数并不会在对象中产生函数实体。
**每一个非inline成员函数只产生一个函数实体**，而拥有0个或者1个定义的inline函数则会在其每一个使用者（模块）身上产生一个函数实体）（也就是生成多份函数代码）。
因此，`class Point3d`即便做了封装，也没有在空间上、或者执行效率有不良影响

实际上，C++在**布局以及存取时间上主要的额外负担**是由**virtual**引起的：
1. 虚函数（用于实现有效率的执行期绑定）
2. 虚基类（用于实现多次在继承体系中出现的基类，**有一个单一而被共享的实体**）
此外，还有**多重继承下的额外负担**，发生在：一个派生类和其第二或后继的基类的转换之间（为了应对命名冲突和二义性）
一般来说，并无理由说C++程序一定比C庞大或者迟缓


## 1.1 C++对象模式
C++中，数据成员有两种情况：**static、非static**
三种成员函数的情况：**static、非static、virtual**

```cpp
class Point{
public:
    Point(float xval);
    virtual ~Point();
    float x() const;
    static int PointCount();

protected:
    virtual ostream& print(ostream &os) const;
    float _x;
    static int _point_count;
};
```
以下是可能的对象模型：
### 简单对象模型
图表示Point对象**本体全都是存储地址**（本体全是指针），每一个地址指向了某一成员变量、或者是成员函数。好处是避免了因为成员变量有不同的类型，因而需要不同的存储空间。
访问成员是通过指针数组的索引来实现的。

这个模型实际根本不会用，但是思想被应用到C++的指向成员的指针观念中。

### 表格驱动对象模型
把所有与members相关的信息抽出来，分别放在：
1. data member table
2. member function table
class object本身持有的是**指向这2个表格的指针**

这个模型实际也不会用，但是思想被应用到C++的虚函数中。

### c++对象模型
C++之父的C++对象模型是简单对象模型的派生：
**把非静态数据成员放在每一个class object内，静态数据成员置于之外，所有函数（静态与非静态）也是放在class object之外。**

而虚函数的实现：
1. 每个**class**产生一个**虚函数表（vtbl, virtual table）**：存放指向虚函数的指针
2. 每一个**class object**持有一个**指向相关的虚函数表的指针（vptr, virtual pointer）**
    vptr的set、reset都是由一个class的constructor、destructor、copy assignment自动完成

每一个class所关联的type_info object（即类型信息类，用于支持RTTI的、存储类型信息的对象）也是靠虚函数表来指示，通常由虚函数表的第一个位置来指示。

该模型这样设计的好处是：考虑**占据空间和存取数据的效率**
缺点是：类定义如有改动，需要重新编译

#### 加上继承
**虚继承**：比如钻石继承的情况下，基类只会有一个实体

如何存储继承信息？搞一个基类表怎么样，类对象本身只持有一个基类表指针？
缺点是随着继承深度增加，“间接性”也增加，即：孙子类取得爷爷类的成员要两次间接存取。如果让派生类对应的基类表记录所有层级的基类，访问就快了，但是付出了空间上的代价。

当一个类继承了一个或多个虚基类时，编译器会在**派生类的对象中分配一个虚基类指针**，用于指向虚基类子对象在对象中的位置。
同时，编译器还会为每个虚基类生成一个**虚基类表**，用于存储**虚基类子对象在派生类对象（派生类对象的首地址存放是虚基类表指针）中的偏移量**。
当派生类对象被创建时，虚基类子对象会被初始化，并且虚基类指针会被设置为指向虚基类子对象在派生类对象中的位置。
当派生类对象被销毁时，虚基类子对象也会被销毁。
在访问虚基类成员时，编译器会使用虚基类指针和虚基类表来计算虚基类成员在派生类对象中的地址（因为要确保虚继承的Base只有一份，所以要用多份指针来指），从而实现对虚基类成员的访问



### 对象模型如何影响程序
不同的对象模型，会导致：
1. 现有的程序代码必须修改
2. 必须加入新的程序代码
因为要考虑C++代码转变为机器码（这个转换是和模型相关的）
举例：
```cpp
X foobar(){
    X xx;
    X *px = new X;

    // foo()是一个虚函数
    xx.foo();
    px->foo();

    delete px;
    return xx;
}

// 编译器生成代码时会转换为以下的逻辑：
void foobar(X &_result){
    // 构造_result，用于取代原来的局部变量 xx
    _result.X::X(); // 构造函数

    // X *px = new X;的展开
    px = _new(sizeof(X));   // 分配一块内存
    if( px!= 0)
        px->X::X();     // 构造对象

    // xx.foo()的展开 调用对象的方法，显然不是虚函数那样的动态绑定
    // 以_result取代xx
    foo(&_result);  // 调用普通成员函数

    // 使用虚函数机制，扩展px->foo()
    (*px->vtbl[2])(px)  // 调用虚函数表对应的虚函数，是foo
    // 当一个对象调用其虚函数时，实际上是通过虚指针找到该对象所属类的虚函数表，然后根据虚函数在虚函数表中的位置，根据对象的类型调用相应的虚函数。

    // 扩展delete px;
    if( px!= 0 ){
        (*px->vtbl[1])(px); // destructor   虚析构，为了基类指针指向派生类的时候，能够调用派生类的析构函数
        _delete(px);
    }

    // 不需使用named return statement
    // 不需要摧毁局部变量
    return;
}   // PS：虚函数表：1、指向type_info的指针 2、指向析构函数 3、指向foo

```

## 1.2 关键词所带来的差异
```cpp
// 无法确定是（函数指针声明）声明还是函数调用（把返回值类型转为int）（直到看到1024这个整数）
int (*pf)(1024);

int(*pq)(); // 声明而非调用
```
当语言无法区分语句是一个声明还是一个表达式，那么就需要一个超越语言范围的规则。

### 策略性正确的struct
C语言中可以巧妙地将指针放在结构体尾部，以实现struct对象拥有可变大小的数组
```cpp
// struct
struct mumble{
    char pc[1]; // 放在尾端，实现数据大小可变
};

struct mumble *pmumb1 = (struct mumble*)malloc(sizeof(struct mumble) + strlen(string) + 1);
strcpy(&memble.pc, string);

/*
    如果采用class来声明
    该class指定不同的访问权限区域，可能由其他类派生而来，有虚函数
    那么该类可能不能类似struct地转化
*/

class stumble{
public:
    // ...
protected:
    // ...
private:
    // ...
    char pc[1]; // 只有protected数据成员放在private数据成员前面，此处的数组才有效，如果存在虚函数，这招就更无效了
}
// 
```

C++中，处于同一个访问区域的数据，必定要保证以其声明次序出现在内存布局中，然而放置在多个访问区域中的各笔数据，排序次序是不定的。

如果想让一个复杂的C++ class的某部分数据，使它拥有C声明的那种样子，这部分最好抽取出来成为一个独立的struct声明，操作如下：
```cpp
struct C_point{...};
class Point: public C_point{...};
// 利用派生

// 于是C和C++两种用法都可获得支持：
extern void draw_line(Point, Point);
extern "C" void draw_rect(C_point, C_point);
draw_line(Point(0, 0), Point(0, 0));
draw_rect(Point(0, 0), Point(0, 0));
```
但是这种继承的做法已经不再推荐，因为某些编译器在虚函数机制中对class的继承布局做了一些改变。


在C++中，`extern "C"`是一个关键字组合，它用于告诉编译器使用C语言的命名和调用约定来编译某段代码块或函数，并避免C++的名称修饰（name mangling）。
**名称修饰**：C++编译器为了支持函数重载和命名空间等特性，会对函数和全局变量等进行名称修饰，即在函数或变量名前添加一些符号或编码，以体现它们的类型和特征。这种名称修饰导致C++编译生成的符号与C语言不兼容，无法直接调用。
因此，当需要在C++代码中调用C语言编写的代码时（比如使用C语言编写的库），就需要使用`extern "C"`来告诉编译器不对特定的代码块进行名称修饰，以便在C++代码中正确调用C语言代码。
因而，**组合，而非继承，才是把C、C++组合在一起的唯一可行方法**。
```cpp
// 组合而非继承
struct C_point(...);
class Point{
public:
    operator C_point(){
        return _c_point;
    }
/*
    conversion函数，如果将Point对象赋值给C_point对象，会调用该函数 例如：
    Point a;
    C_point t = a;
*/
private:
    C_point _c_point;
};
```
C struct中的合理用途，是当你要传递“一个复杂的class object的全部或部分”到某个C函数中去，**struct声明可以将数据封装起来**，并保证拥有与C兼容的空间布局（条件是组合而非继承，如果是继承而非组合，编译器会决定是否应该有额外的data members被安插到base struct subobject中）。


## 1.3 对象的差异
c++程序设计模型直接支持三种**程序设计典范**
### 程序模型
是指面向过程

### 抽象数据类型模型 ADT
抽象指的是：和一组public接口一起提供，而运算的定义的是隐藏的
比如string的·`operator==`方法或者赋值，就隐藏了细节

### 面向对象模型 OO
比如先设计一个抽象基类，然后相关的派生类由此派生而来。

混用不同的程序设计模型会有问题：
比如：
```cpp
// 想要以基类实例实现多态时：

Base b;
class Derived1: public Base{...};
Derived1 d1;
d1 = b;
b.check_in();   // 调用的还是基类的方法

// 多态应该通过基类的指针或者引用来实现
Base& b2 = d1;
b2.check_in();  // 调用的是派生类的方法
```
面向对象的多态性质需要通过（基类的）指针或者引用
在面向对象中，被指向的对象的真实类型在每一个特定执行点之前，是无法解析的（由于继承体系的存在，其类型有多种可能）
而ADT范式中，编译期就已经确定了类型

在C++中，多态只存在于一个个的public class中，非public的派生行为以及void*指针可以称作多态，但是没有被语言明确地支持，也就是必须由程序员转型来实现。
C++以下列方法支持多态：
1. 经由一组隐含的转化操作：派生类指针转为指向public基类的指针
2. 经由虚函数机制
3. 经由dynamic_cast和typeid运算符
(dynamic_cast会做类型安全检查，只有基类指针指向派生类才能转为派生类指针)

多态的主要用途是经由一个**共同的接口**来影响类型的封装，这个接口通常定义在抽象基类中，然后在运行期判断到底是哪一个实体被调用。
--> 好处是：
1. 当类型增、改、减时，不用相应地修改代码
2. 可以利用虚函数，不用针对每种类型写出相应的代码，一份代码即可，只要放一份基类指针，根据传进来的对象自然会调用不同的函数

**需要多少内存才能够表现一个class object？**
1. 非static数据成员的总和大小
2. 加上任何由于alignment的需求而填补上去的空间（可能存在于members之间、也可能存在于集合体边界）
**alignment是指将数值调整到某数的倍数，比如在32位计算机，通过alignment为（4Byte）32位，以使得bus的运输量达到最高效率**
3. 以及为了支持virtual而在内部产生的任何额外负担

一个指针（或者引用，引用底层也是通过指针来实现），不管指向的是哪一种数据类型（无论有多么复杂），指针本身所占据的内存大小必然是固定的（存储地址）。


### 指针的类型
那么，一个指向复杂class的指针究竟和一个指向整数的指针、或者一个指向Array<String>的指针有何不同
**本质上都只是地址的值而已**
区别在于：**根据不同类型的指针寻址出来的object类型不同，即：“指针类型”教导编译器如何解释某个特定地址中的内存内容及其大小**

简单的class object的布局和指针布局是可以推测的
void*指针就只能含有一个地址，而无法通过它操作指向的object
所以，**转型实际上是一种编译器指令，大部分情况下不改变一个指针所含的真正地址，只影响“被指出的内存的大小以及内容”的解释方式**

### 加上多态...
派生类的内存布局相比基类只是增加一些特有成员的位置

```cpp
Derived d;
Base *pb = &d;   // 指向派生类对象的基类指针 涵盖的范围只是派生类的基类成分的部分
Derived* pd = &d;   // 指向派生类的派生类指针 涵盖整个派生类的部分
```
因此，显然指向派生了对象的基类指针无法操作派生类独有的函数，除非是虚函数
或者显式转为派生类指针
或者运行时操作：
```cpp
if(Derived* pd = dynamic_cast<Base*>(pb))
    pd->fun_derived();
```
如果通过基类指针调用虚函数：
指针类型将在编译期决定两点：
1. 固定的可用接口，基类指针只能调用基类的public接口
2. 接口的访问级别
基类指针所指的对象类型决定虚函数调用的实体，这个类型信息的封装存在于：**对象的虚函数表指针和虚函数表中**
```cpp
Derived d;
Base b = d; // 这样的转换导致切割

b.fun_virtual();    // 无法实现多态
```
为什么b的vptr不指向d的虚函数表？
编译器在**初始化**和**赋值**操作之间做了仲裁，编译器必须确保**如果某个object含有一个或一个以上的vptrs(虚继承、或者多重继承)，那些vptrs的内容不会被基类对象初始化或改变**
重点是：某一对象的虚函数表的内容是不会被基类对象初始化或者修改的，因为编译器要确保虚函数表指针指向正确的虚函数表

指针的类型如同契约，限制了内存的“资源需求量”，如果基类对象被赋给派生类，派生类对象就会切割出其中的基类成分，无法实现多态。

而ADT的程序风格，或称为object-based（OB），例如String class。一个OB的设计相比对等的OO设计，速度更快而且空间更紧凑：速度快是因为所有的函数引发操作都在编译期解析完成，没有虚函数的机制，空间紧凑也是因为没有虚函数的额外负担，但是这种设计思想缺乏弹性。



# 第二章 构造函数语意学
常有人抱怨C++编译器背着程序员做了太多事情，比如Conversion运算符














