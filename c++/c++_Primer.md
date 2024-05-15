C++ Primer：


# 第一章
类定义了行为：
类定义了创建一个对象时会发生什么事情


# 第二章

变量和基本类型

引用
左值引用必须被初始化
定义引用时，程序把引用和它的初始值绑定在一起，而不是把初始值拷贝给引用。
无法令引用重新绑定到另外一个对象。
int &b = a;
若再执行b = c，那只是把c的值赋给了a，a和b之间的绑定关系没变 --> 所以必须赋初值
若是const引用，则无法更改被引用的对象的值。

指针
而指针无需赋初值。
空指针 nullptr
赋值永远改变的是等号左边的对象

void* 可以存放任意对象的地址
可用于作为函数的输入或输出，或者赋给另外一个void指针

二级指针
指向指针的引用
int *& r = p

const 限定符
const int t= 111;
任何试图修改的行为都将引发错误
编译器实际上是直接替换该变量

const int &r1 = ci;

注意虽然引用的类型必须与其所引用的对象的类型一致，但是例外情况：1、再初始化常量引用时允许用任意表达式作为初始值，只要其结果能够转化成支持的引用类型即可。允许为一个常量引用绑定非常量的对象、字面值，甚至是一般表达式。

int i=42;
const int &r1=i;    // 正确
const int &r2=42;   //正确
const int &r4 = r1*2;   //错误
r4本身是普通的非常量引用

当一个常量引用被绑定到另外一种类型时到底发生了什么？
double dval = 3.14;
const int &ri = dval;

ri引用了一个ini类型的数，
编译器会优化：
const int tmp = dval;   // 由于双精度浮点数生成的一个临时的整形常量tmp
const int &ri = tmp;    // 让r1绑定这个临时量


常量引用不能修改值

指向常量的指针·不能·用于改变其所指对象的值
常量对象对应的指针必须也是const

常量指针必须初始化

const double pi = 3.14;
const double *const pip = &pi;
（从右往左看）

顶层const表示指针本身是个常量
int i = 0;
int *const p1 = &i; // p1的值无法改变

底层const表示指针所指的对象是一个常量
底层const：
当执行对象的拷贝操作时，拷入和拷出的对象必须具有相同的底层const资格，或者两个对象的数据类型可以转换。

const int &r = ci;
用于声明引用的const都是底层const

当执行对象的拷贝操作时，常量是顶层const还是底层const区别明显，顶层const不受什么影响。

int i = 0;
int *const p1 = &i; // 顶层const，不能改变p1的值
const int ci = 42;  // 不能改变ci的值，顶层const
const int *p2 = &ci;    // 底层const，允许改变p2的值
const int *const p3 = p2;   // 左边的const是底层

i = ci; // 正确 ci是顶层const
p2 = p3;    // 正确，p2,p3指向的对象类型相同

底层const的限制不能忽视：
拷贝操作时，拷入和拷出的对象必须具有相同的底层const资格
或者数据类型可以转换
非常量可以转换成常量

int *p = p3;    // p3包含底层const的定义，而p没有
p2 = p3;    // 正确
p2 = &i;    // int 可以转为const int*
int &r = ci;    // 错误，普通的int&不能绑定到int常量上
const int &r2 = i;  // 正确

常量表达式
constexpr
值不会改变并且在编译过程就能得到计算结果的表达式

const int t = 20;   // t是常量表达式

字面值类型：
算术类型、引用和指针
自定义类、IO库、string类型就不属于
const int *p = nullptr; // 指向整型常量的指针
constexpr int *q = nullptr; // 是指向整数的常量指针

## 2.5 处理类型
### 2.5.1 类型别名
typedef double wages;   // 同义词
typedef wages base, *p; // base是double的同义词 p是double\*的同义词

别名声明：
using SI = Sales_item;

typedef char *pstring;  // 意思是：pstring是指向char的指针
const pstring cstr = 0; // 指向char的常量指针
const pstring *ps; // ps是一个指针，对象是指向char的常量指针

注意，不能替换以理解
const char *cstr = 0; //是错误的
原因是：
const pstring cstr = 0; // 意味着基本数据类型是指针
这样改写的话，数据类型就变成了char，*成了声明符的一部分，就变成了指向常量char的指针




`auto`类型说明符
让编译器分析表达式所属的类型
这是通过**初始值**来推算的，所以auto定义的变量必须有初始值

auto会忽视顶层const，保留底层const。

而类型为auto的引用可以保留顶层常量属性。

&和*都只从属于某个声明符，而非基本数据类型的一部分。


`decltype`类型指示符
希望从表达式的类型推断出要定义的变量的类型，但是不想用该表达式的值初始化变量
作用是：选择并返回操作数的数据类型
编译器分析表达式并得到它的类型，却不实际计算表达式的值
decltype保留顶层const
```cpp
int i = 42, *p = &i, &r = i;
decltype(*p) c;  // 表达式是解引用，decltype得到引用类型
```
！！！解引用指针可以得到指针所指的对象，因此得到的是引用（要注意型别信息也一并推导）

```cpp
int i= 42;
decltype((i)) d;    // 错误，d是int&，必须初始化
// 双层括号，编译器把它当成表达式，是可以作为赋值语句左值的特殊表达式，总之双层括号的结果永远是引用
decltype(i) e;   //正确 int
```

### 2.6.3 头文件
类通常被定义在头文件，头文件通常包含那些只能被定义一次的实体

预处理器的工作：编译前的预处理，做一些替换
头文件保护符依赖于预处理变量
#define把一个名字设定为预处理变量
#ifdef当且仅当变量已定义时为真 执行操作直到#endif
#ifndef ~
用于防止重复包含的发生



# 第三章
头文件不应包含using声明，因为头文件的内容会被拷贝到所有引用它的文件中去，可能引起名字冲突

## 3.2标准库类型string
=实际上是拷贝初始化



## 3.3 vector
是一个类模板
根据模板创建类或者函数的过程称为实例化
vector<T>v4{1,2,3};
而不是()圆括号

vector对象能高效增长
不能使用下标形式添加元素，而应该使用push_back
下标只能访问已存在的

### vector的初始化方式
1. 列表初始化
    vector<int>nums = {1, 2, 3};

2. 使用push_back逐个添加元素
3. 使用（另一个容器的）迭代器范围初始化
4. vector自带的构造函数：vector(int n)，内容是n个0


## 3.4 迭代器介绍
类似于指针类型，提供对对象的间接访问

有迭代器的类型同时拥有返回迭代器的成员
这些类型都有begin和end的成员，begin成员负责返回指向第一个元素
auto b = v.begin()  // 由编译器决定b的类型
end成员返回的迭代器称为尾后迭代器
通常用== !=运算符

for循环种用!=而非<，是因为这种风格在标准库提供的所有容器上都有效

凡是使用了迭代器的循环体，不要向迭代器所属的容器添加元素




## 3.5 数组
int *p[10]; // 含有10个整型指针的数组
下标运算符内必须是常量表达式
字符数组必有一个空字符'\0'

不存在引用组成的数组
int (*p)[10] = &arr;    // 指向一个含有10个整数的数组
默认情况下，类型修饰符从右向左依次绑定
但是对于数组而言，由内向外阅读

使用数组的时候编译器一般会把它转换成指针

string *p2 = nums;  // nums是字符串数组
等价于 p2 = &nums[0]

指针加上一个整数结果还是指针
尽量别用c风格字符串
尽量使用vector和迭代器，避免使用内置数组和指针



# 第四章 表达式
对于结果为左值的表达式，decltype的结果为引用类型。
点运算符和箭头运算符都可以用于访问成员。
ptr->mem等价于(*ptr).mem


## 4.11 类型转换
隐式转换：不需要程序员介入
数组转换成指针
指向任意非常量的指针能转换成void*，指向任意对象的指针能转换成const void*

整型提升：把小整数类型转换成较大的整数类型

### 4.11.3 显式转换
强制类型转换(cast)
#### static_cast
只要不包含底层const，都可以使用
通常用于不在于精度损失的情况

#### dynamic_cast
运行时类型识别

#### const_cast
作用是去掉const性质，只能改变运算对象的底层const
常用于有重载函数的上下文

#### reinterpret_cast
为运算对象的位模式提供较低层次上的重新解释
int *ip;
char *pc = reinterpret_cast<char*>(ip);
pc所指的真实对象是一个int而非字符，如果把pc当成普通的字符指针使用就可能在运行时发生错误。
reinterpret_cast的使用是危险的，本质上依赖于机器。

最好避免强制类型转换。



# 第五章 语句
空语句，语法上需要但逻辑上不需要

switch(ch){
case 'a':
    xxx;
    break;
case 'b':
    xxx;
    break;

}

continue中断当前迭代，继续执行循环


try catch：
try {
    xxx;
}
catch(异常声明){
    处理逻辑;
}
catch(runtime_error err){
    std::cout<<err.what()<<std::endl;
}

编写异常安全的代码非常困难




# 第六章 函数

函数的形参列表：
int f4(int v1,int v2)

局部对象
名字有作用域、对象有生命周期
函数终止时，形参也被自动销毁

局部静态对象
使得局部变量的生命周期贯穿函数调用及之后的时间
static声明

函数只能定义一次，但是能被声明多次
函数声明也被称作函数原型
声明放在头文件，定义放在源文件
头文件被包含到定义函数的源文件中

分离式编译
把程序分割到多个文件，每个文件独立编译

## 6.2 参数传递
传引用 / 传值

使用引用避免拷贝
拷贝对于大类型而言低效，甚至有的类类型不支持拷贝

如果无须改变引用形参的值，最好声明为常量引用


const形参和实参
形参有顶层const时，传给它常量对象或者非常量对象都是可以的
int i = 0;
const int ci = i;
reset(ci);  // 是错的，因为ci是不可修改的

尽量使用常量引用

f(int (&arr)[10])   // arr是具有10个整数的整型数组的引用

### 6.2.5 处理命令行选项
prog -d -o ofile data0
int main(int argc , char *argv[])
argc：表示数组中字符串的数量
argv：指向字符串的指针

argv[0]="prog"
argv[1]="-d"
argv[2]="-o"
...
argv[5]=0   // 最后一个元素必须为0

PS: int *p[4]   // 先形成数组
    int (*p)[4] // 先形成指针
注意：()优先级比[\]高，[]的优先级比*高


6.26 含有可变形参的函数
场景：想要编写代码输出程序产生的错误信息，最好用同一个函数实现该功能
如果所有的实参类型相同，可以传递一个名为initializer_list的标准库类型
如果实参类型不同，可使用可变参数模板
eg initializer_list<int>li;
其中元素永远是常量值，无法改变其值
这个initializer_list可用作函数形参

省略符形参
只能出现在形参列表的最后一个位置
void foo(parm_list,...);
void foo(...);

## 6.3 返回类型和return语句
*** 不要返回局部对象的引用或指针 ***
这是错误的，因为函数终止时，局部变量的引用将指向不再有效的内存区域

引用返回左值
调用一个返回引用的函数得到左值，其他类型得到右值

列表初始化返回值
以花括号形式返回
允许main函数没有return语句直接结束-->编译器隐式地插入return 0


### 6.3.3 返回数组指针
typedef int arr[10];    // arr表示的类型是含有10个整数的数组
using arr = int[10];


声明一个返回数组指针的函数
int *p1[10];    // p1是一个含有10个指针的数组
int (*p2)[10]=&arr; // p2是一个指针，指向数组

int ((*func(int i))[10];
目的是返回数组指针，加括号的话表示先和*符号结合


尾置返回类型
auto func(int i) -> int(*)[10];




## 6.4 函数重载
main函数不能重载
编译器根据实参的类型确定应该调用哪一个函数

Record lookup(const Account&);  // 省略了形参的名字
声明时可以省略，但是定义时一定要有

重载函数时，顶层const并不足以区分

底层const可区分

Record lookup(Account&);
Record lookup(const Account&);

const_cast可用于重载函数 去掉const性质
实现：当实参不是常量时，返回值也不是常量

在不同的作用域（函数体内部也算是作用域）中无法重载函数名



### 6.5.1 默认实参
某些函数的形参，在函数的很多次调用中都被赋予一个相同的值。
缺省值只能在右侧



### 6.5.2 内联函数
避免函数调用的开销

编译过程中直接进行替换
省去了函数调用栈的开销
编译器可以忽略内联说明

constexpr函数：
能用于常量表达式的函数
即常量表达式


### 6.5.3 调试帮助
assert预处理宏
assert(expr);
如果表达式为假，则输出信息并终止程序



## 6.7 函数指针

bool func(const string &,const string &);

指向该函数的函数指针：
bool (*pf)(const string &,const string &);
形参列表体现出pf指向的是函数



当我们把函数名作为一个值使用时，该函数自动地转换成指针。
甚至可以省略取地址符

重载函数的指针
void ff(int*);
void ff(unsigned int);

void (*pf1)(unsigned int) = ff;


函数指针形参
不能定义函数类型的形参，但是形参可以是指向函数的指针



# 第七章 类
类的基本思想是 ***数据抽象*** 和 ***封装***。
数据抽象依赖于接口和实现分离。
类的接口包括用户所能执行的操作;类的实现包括类的数据成员、负责接口实现的函数体及定义类所需的各种私有函数。

封装实现了类的接口和实现的分离，封装后的类隐藏了实现细节。
类的用户只能使用接口而无法访问实现部分。

*this代表的是执行该函数的对象

### 7.1.4 构造函数
只要类的对象被创建，就会执行构造函数。
不能是const的，构造函数在const对象的构造过程中可以向其写入值，因为直到构造函数完成初始化过程，对象才能真正取得常量属性。

默认构造函数是有缺陷的：没有赋给成员初值


*** 初始值列表 ***
Sales_data(const std::string &s,unsigned n,double p):book(s),units_sold(n),revenue(p*n) {}


局部对象会在创建它的块结束时被销毁
使用vector或者string的类能避免分配和释放内存带来的复杂性


把数据成员的访问权限设成private，外层就无法访问，当修改了类的接口，用户代码就无需改变，而且也可以防止用户的原因造成数据被破坏。

定义在类内部的成员函数自动inline
本质上减少了汇编:函数调用的开销(汇编call指令、清理现场:将压入栈的数据都出栈)
像宏一样展开，相比宏替换，编译器对待内联函数会检查参数类型，而宏替换不会，有安全隐患


mutable关键字，用于能修改类的某个数据成员，声明一个可变数据成员（即便是const对象的成员）。

类数据成员的初始值
提供类内初始值必须以'='或者花括号表示

可以根据对象是否是const重载函数


### 7.3.3 类类型
前向声明：先声明类而不定义


## 7.4 类的作用域
在类的作用域外，普通的数据和函数成员只能由对象、引用或者指针使用成员访问运算符来访问。

注意：如果类内部的成员使用了外层作用域中的某个名字，而该名字代表一种类型，则类不能在之后重新定义该名字。

类型名的定义通常在类的开始处，确保所有使用该类型的成员都出现在类名的定义之后。


## 7.5 构造函数再探
const或者引用必须初始化

初始化和赋值差别在于底层的效率问题，前者直接初始化，后者先初始化再赋值

X(int val):j(val),i(j){}    // 实际上先初始化i，再初始化j

写成这样更好：i (val),j(val)

### 7.5.2 委托构造函数
一个委托构造函数使用它所属类的其他构造函数执行它自己的初始化过程


```cpp
// 委托构造举例
class Sales_data{
public:
    // 非委托构造函数使用对应的实参初始化成员
    Sales_data(std::string s, unsigned cnt, double price):
        bookNo(s), units_sold(cnt)... {}

    // 其余构造函数全都委托给另一个构造函数
    Sales_data():Sales_data("", 0, 0){}
    Sales_data(std::string s):Sales_data(s, 0, 0){}

    // 可见，委托构造函数后面的初始化列表是另一个构造函数

};
```
当一个构造函数委托给另一个构造函数时，受委托的构造函数的初始值列表和函数体被依次执行，如果受委托的构造函数体有代码，就先执行这些代码。






### 7.5.4 隐式的类类型转换
只允许一步类型转换
string n = "999";
item.combine(n);

或者item.combine(string("999"));


在要求隐式转换的程序上下文中，可以通过将构造函数声明为explicit阻止隐式构造

explicit构造函数只能用于直接初始化


### 7.5.5 聚合类
表示可以直接访问成员、并且具有特殊的初始化语法形式

constexpr构造函数用于生成constexpr对象


## 7.6 类的静态成员
static 关键字使得类的成员与类关联在一起（而非对象）
类的静态成员实际上存储在任何对象之外，不包含this指针
不是由类的构造函数初始化

eg double r;
r = Account::rate();




# 第八章 IO库





# 第九章 顺序容器
需要注意容器的增删查的性能代价

标准库中定义了
vector变长数组、deque双端队列、list双向链表、forward_list单向链表、array固定长度数组、string字符数组

vector和string是连续存储，所以查找很快（直接根据下标计算地址），而插入和删除很耗时

list和forward_list由于采用链表，添加删除快，但无法随机访问，只能遍历，相比其他容器还存在额外的内存开销

deque双端队列，支持快速的随机访问，在中间位置添加删除代价大，但是两端的添加删除很快

标准库的容器经过优化，比手写的好

***通常使用vector是最好的选择***



## 9.2 容器库概览
所有容器都适用的操作
注意如果传入的类型是没有默认构造函数的话，必须传入元素初始化器

#### 容器操作
1. 类型别名：iterator、value_type、reference
2. 构造函数
3. 赋值 SWAP
4. 得到大小
5. 添加/删除
6. 获取迭代器
7. 反向容器的额外成员



### 9.2.1 迭代器
迭代器范围：
[begin.end)
对二者作比较< = >，可得出迭代器的范围

### 9.2.2 容器类型成员
```c++
list<string>::iterator iter;
// iter是通过list<string>定义的一个迭代器类型
vector<int>::difference_type count;
```
difference_type是什么？
--> 表示两个迭代器之间的距离

### 9.2.3 begin和end成员
```c++
list<string>a = {"a","b"};
auto it1 = a.begin();
// auto等价于list<string>::iterator
```


### 9.2.4 容器定义和初始化
除了array的容器的默认构造函数都会创建一个指定类型的容器，其参数可以接收大小和初值

```c++
C c;
C c1(c2);
C c1 = c2;
C c{a,b,c}; // c初始化为初始化列表中元素的拷贝
C c = {a,b,c};
C c(b,e);  // c初始化为迭代器b和e指定范围中的元素的拷贝

// 除了array
C seq(n);    // 接收大小参数
C seq(n,t);     // 接收n个初始化为值t的元素

```

#### 将一个容器初始化为另一个容器的拷贝
既可以直接拷贝整个容器，又可以拷贝迭代器指定的范围


array具有固定大小
```c++
array<int,42>arr;
array<int,42>::size_type i;
```
size_type是什么？
--> 用于保存任意string对象或者vector对象的长度（被设计得足够大，是unsigned int类型）


### 9.2.5 赋值和swap
swap(c1,c2)

assign操作：
不适用于关联容器和array
seq.assign(b,e)
// 将seq中的元素替换为迭代器b和e表示的范围的元素

#### assign常见用法：
    assign(begin, end)  将一个容器的内容替换为另一个容器的内容
    assign(n, value)    可以重置容器大小，截断或扩展，使得确保容器为指定大小
                        可用于值初始化



注意：swap不对任何元素进行拷贝、删除或者插入操作 因此保证了常数时间完成
元素本身并未交换，只是交换了两个容器的内部数据结构（指向原先容器的迭代器不会失效）
但是对array进行swap会真正交换元素（因此所需时间和元素数目成正比）


### 9.2.7 关系运算符
比较两个容器实际上是对元素的逐对比较

```c++
vector<int>v1 = {1,3,5};
vector<int>v2 = {1,3,9};
v1 < v2   // true 因为v1[2] < v2[2]（而前面都相同）

```

## 9.3 顺序容器操作

### 9.3.1 添加元素
push_back   // 尾部创建 插入的是拷贝 而非对象本身
push_front  
emplace_back    // push或者insert是直接传对象，而emplace是将参数传递给元素类型的构造函数，在容器的内存空间中直接构造元素
因此要和构造函数的参数匹配
emplace_back:
push_back会通过拷贝构造函数创建一个对象，而这个对象必须事先创建，即便你传入临时对象也是如此
   - `emplace_back` 接受的参数是**传递给元素类型的构造函数的参数，而不是已有对象的拷贝**，从而减少参数传递的成本，那么对于复杂类型就不需要复制整个对象。它会**在容器的末尾就地构造一个新的元素**。
   - `emplace_back` 不需要创建临时对象或执行拷贝操作，因此通常比 `push_back` 更高效。

emplace_front
insert  // 返回值为指向第一个新加入元素的迭代器

需要注意操作的性能



### 9.3.2 访问元素
back()  返回尾元素的引用
front() 首元素的引用
[]  下标访问
at(n)   相比下标访问，会进行边界检查

注意到访问操作返回的是引用

下标运算符的入参必须在范围内，这是程序员的责任，编译器并不检查是否合法，虽然这是一种严重的程序设计错误

而at方法会抛出异常


### 9.3.3 删除元素
pop_back()  // 删除尾元素
pop_front() // 删除首元素
erase(p)
clear()


### 9.3.4 特殊的forward_list操作
单向链表删除一个元素时会改变序列中的链接
forward_list中添加或者删除元素的操作是通过改变给定元素之后的元素来完成的，以便访问被添加或删除操作影响的元素（意思是说得到一个元素的前驱不容易，因此我们通常操作后继节点）


### 9.3.5 改变容器大小
resize(n)


### 9.3.6 容器操作可能使迭代器失效
***向容器添加元素后***
如果是vector或string，且存储空间重新分配，则指向容器的迭代器、指针、引用都会失效
此处省略


## 9.4 vector对象是如何增长的
因为支持随机访问，vector的元素是连续存储的
只有当不得不获取新的内存空间时，vector的实现才会分配比新的空间需求更大的内存空间
当然在每次重新分配内存空间时需要移动所有元素

capacity()方法，得到可以保存元素的数目（不重新分配内存空间的前提下）
而size()是已经保存多少
可以通过reserve()方法修改capcity的大小
只要没有操作的需求超出vector的容量，vector就不能重新分配内存空间


## 9.5 额外的string操作
### 9.5.1 构造string的其他方法
```c++
string s(cp,n); // s是cp指向的字符数组(char*)中，钱n歌字符的拷贝
PS：从const char*创建string时，指针指向的数组必须以空字符结尾

string s(s2,pos2); // s是string s2从下标pos2开始的字符的拷贝

string s(s2,pos2,len2); // 从下标pos2开始的len2个字符的拷贝

s.substr(0,5)   // [0，5)
s.substr(6)   // [6, end)

```

### 9.5.2 改变string的其他方法
支持额外的insert和erase

append用于在末尾进行插入

replace即替换，等价于erase和insert

### 9.5.3 string搜索操作
返回的是string::size_type值，实际上是unsigned类型

find()

rfind()


## 9.6 容器适配器
适配指的是：所有容器适配器都支持的操作和类型















# 第十章 泛型算法












# 第十章 泛型算法
标准库的提供了一组算法，大多可以独立于任何特定的容器，这些算法是通用的、泛型的（generic）
例如：查找特定元素、替换、删除、重排




## 10.1 概述
#include<algorithm>
#include<numeric> // 数值泛型算法

这些算法遍历由两个迭代器指定的一个元素范围

```cpp
// 在vector中找一个元素
auto result = find(vec.cbegin(), vec.cend(), val);
// const_iterator
// 若未找到返回vec.cend()

```
算法都是运行于迭代器之上



## 10.2 初识泛型算法


### 10.2.1 只读算法
accumulate
```cpp
    int sum = accumulate(vec.cbegin(), vec.cend(), 0);
    // 和的范围、初值
    // 需要注意元素相加是可行的
```



#### 算法和元素类型
只读算法最好用cbegin()、cend()


#### 操作两个序列的算法
equal 确定两个序列是否保存相同的值 所有对应元素都相等才返回true
```cpp
    equal(roster1.cbegin(), roster1.cend(), roster2.cbegin());

```

### 10.2.2 写容器元素的算法
```cpp
fill(vec.begin(), vec.end(), 0);    // 每个元素重置为0
fill(vec.begin(), vec.begin() + vec.size()/2 ,10);
```

#### 算法不检查写操作
```cpp
fill_n(dest, n ,val);
// 假定dest指向一个元素，从dest开始的序列至少包含n个元素
```
#### 介绍back_inserter
插入迭代器
用于保证算法具有足够元素空间来容纳输出数据
```cpp
vector<int>vec;
auto it = back_inserter(vec);   // 会将元素添加到vec中
*it = 42;


vector<int>vec;
fill_n(back_inserter(vec), 10, 0);
// 创建了一个插入迭代器，用于添加元素
// 等价于10次对vec调用push_back
```

#### 拷贝算法
```cpp
int a1[] = {0,1,2 ... 9};
int a2[sizeof(a1)/sizeof(*a1)];
auto ret = copy(begin(a1), end(a1), a2);    // a1拷贝给a2
// ret指向拷贝到a2的尾元素之后的位置

replace(list.begin(), list.end(), 0 ,42);
// 把所有的0替换为42

replace_copy(list.cbegin(), list.cend(), back_inserter(ivec), 0, 42);
// ivec得到list的拷贝，并且其中的0-->42

```

### 10.2.3 重排容器元素的算法
sort（利用<运算符实现）

场景：一篇文章，若干单词，去除重复
先排序
```cpp
    sort(words.begin(), words.end());
    auto end_unique = unique(words.begin(), words.end());   // unique重排元素，使得每个单词只出现一次（消除相邻重复项），指向最后一个不重复元素之后的位置
    // unique并不真的删除，只是覆盖相邻的重复元素(毕竟算法不对容器进行操作，而是对迭代器操作)
    words.erase(end_unique, words.end());


```



















## 10.3 定制操作
比如可以重载sort的默认行为


### 10.3.1 向算法传递函数
sort()第三个参数，是一个**谓词**（是一个可调用的表达式，返回结果是一个能用作条件的值）
一元谓词即只接受单一参数
二元谓词两个参数

```cpp
bool isShorter(const string &s1, const string &s2);

sort(words.begin(), words.end(), isShorter);    // 由短到长排序
// stable_sort算法是稳定排序，即对于相同长度的单词仍然按原先顺序排列
```

### 10.3.2 lambda表达式
由于一元谓词的特性（传给find_if的函数必须只能接受一个参数，没法传递第二个参数给它），需要lambda表达式
```cpp
auto wc = find_if(words.begin(), words.end(),
// 第三个参数
    [](const string &a)
    {return a.size() >= sz; } );

```


[capture list](parameter list) -> return type {function body}

捕获列表和函数体必须包含

#### 向lambda传递参数
由于lambda不能有默认参数 因此lambda调用的实参数目永远与形参数目相等
空的捕获列表表示该lambda不使用它所在函数中的任何局部变量


#### 使用捕获列表


***只有在捕获列表中捕获了某一局部变量，才能在函数体中使用（但是可以直接使用局部static变量喝全局变量）***

#### for_each算法
```cpp
for_each(wc, words.end(),
        [](const string &s){cout<<s<<"";});
// 对输入序列中的每个元素调用此可调用对象
// 只对lambda所在函数中定义的（非static）变量使用捕获列表（即只用于局部非static变量），可以直接使用定义在当前函数之外的名字
```


### 10.3.3 lambda捕获和返回
当定义一个lambda时，编译器生成一个与lambda对应的薪的（未命名的）类类型
当向一个函数传递一个lambda时，同时定义了一个薪类型和该类型的一个对象：传递的参数就是编译器生成的类类型的未命名对象


#### 值捕获
前提是变量可以拷贝

被捕获的变量的值是在lambda创建时拷贝，而非调用时
也就是说在创建lambda表达式时，只要捕获了变量，就记录了当时的值


#### 引用捕获
auto f2 = [&v1]{return v1;};
必须确保被引用的对象在lamdba执行的时候是存在的


#### 隐式捕获
让编译器根据lambda中的代码推断我们使用哪些变量
[=] [&] 
[&, c] 表示显式捕获c，其他的是引用
第一个位置指定了默认捕获方式，必须是引用或值

当我们需要为一个lambda定义返回类型时，必须使用尾置返回类型
[](int i)->int


### 10.3.4 参数绑定
lambda的应用场景是只在一两个地方使用的简单操作
反之应该定义函数

#### 标准库bind函数
头文件functional
可以将bind函数看作一个通用的函数适配器
***接收一个可调用对象，生成一个新的可调用对象来适应原对象的参数列表***
auto newCallable = bind(callable,arg_list);
实际上就是：
当我们调用newCallable时，newCallable会调用callable，并传递给它atg_list中的参数（其中可能是_n这样的名字，意思是占位符）


```c++
// check_size是一个函数 接收一个string类型的参数
auto check6 = std::bind(check_size, _1, 6);
/* 意思是：check6作为一个可调用对象，接受一个string类型的参数，并用此string和值6来调用check_size函数
只有_1一个占位符表示check6只接受单一参数
占位符出现在arg_list的第一个尾置表示check6的此参数对应check_size的第一个参数
*/
string s = "hello";
bool b1 = check6(s);
// check6(s)调用check_size(s,6)


```
每个占位符名字都要提供一个单独的using声明
可以使用using namespace namespace_name;
表示想要使用所有来自namespace_name的名字
用std::placeholders替代


#### bind的参数
可见bind可以用来修正参数的值
也可以用bind绑定给定可调用对象中的参数 重新安排顺序
```c++
auto g = bind(f, a, b, _2, c, _1);
f本身接受5个参数
新生成的g有两个参数:_2和_1
意思是:
g将自己的参数作为第三个和第五个参数传递给f
其他的参数绑定到a,b,c
调用的时候也就是:
g(_1,_2)
-->
f(a,b,_2,c,_1)
// 也就是说 bind可以把原来的函数的参数固定化
```



#### 用bind重排参数顺序
bind(isShorter, _2, _1)

#### 绑定引用参数
**bind拷贝参数,如果想传递对象而不拷贝(有些对象不能拷贝),就必须使用std::ref函数
bind(print,ref(of),_1,'')



## 10.4 再探迭代器
iterator头文件中还有其他几种迭代器
插入迭代器
流迭代器
反向迭代器
移动迭代器

### 10.4.1 插入迭代器
接受一个容器,生成一个迭代器,实现向给定容器添加元素
back_inserter:创建一个使用push_back的迭代器
front_inserter:创建一个使用push_front的迭代器
inserter:创建一个使用insert的迭代器,接受第二个参数,其必须是一个指向给定容器的迭代器,元素将被插入到给定迭代器表示的元素之前
只有在容器支持push_front的情况下,才可以使用front_inserter
支持push_back才可以用back_inserter

```cpp
it = inserter(c, inter);
// 得到一个迭代器
*it = val;
// 等价于:
it = c.insert(it ,val); // it指向新加入的元素(在原先元素之前)
++it;   // 递增it让它指向原来的元素


// front_inserter生成的迭代器,元素总是插入到容器第一个元素之前
list<int>lst = {1, 2, 3, 4};
list<int>lst2;  // 空的
copy(lst.cbegin(), lst.cend(), front_inserter(lst2));
// 执行后,lst2包含 4 3 2 1

```



### 10.4.2 iostream迭代器
可用于IO类型对象的迭代器
#### istream_iterator
```cpp
istream_iterator<int> int_it(cin);
// 从cin读取int


```



#### ostream_iterator













# 第十一章 关联容器
按照关键字保存和访问（区别于顺序容器）
主要的关联容器：
map和set mult- unordered-

## 11.1 使用关联容器
map：键值对的集合
又称关联数组

set：关键字的简单集合

#### 使用map

#### 使用set










## 11.2 关联容器概述
由于是根据关键字存储，因为不支持顺序容器的位置相关的操作
关联容器的迭代器都是双向的




### 11.2.1 定义关联容器
eg. map
```cpp
map<string,size_t>w;

set<string>exclude = {"e3","ergw"};

map<string,string> authors = {{"11","12"},
                              {"21","22"}};
```




#### 初始化multimap或multiset

普通的map和set不能有重复的关键字，mult-的可以


#### 使用关键字类型的比较函数
比如对于自定义类型的比较函数可以作为函数指针传入multiset




### 11.2.3 pair类型
```cpp
    std::pair<string,size_t>word_count;

// 列表初始化
    
    vector<string> v;
    
    // 某一函数返回pair<string, int>类型

    // 可以返回列表初始化 返回由v中最后一个string及其大小组成的pair
    return {v.back(), v.back().size()};

    // 隐式构造返回值
    return pair<string, int>();

    // make_pair
    return make_pair(v.back(), v.back().size());

```



## 11.3 关联容器操作
```cpp
set<string>::value_type v1; // 得到string类型 等价于key_type
map<string, int>::value_type v3;    // 得到pair<const string, int>
map<string, int>::mapped_type v5;


```



### 11.3.1 关联容器迭代器
map的value_type是pair
无法通过迭代器改变元素，因为map的关键字是const的


















# 第十二章 动态内存
为了更安全地使用动态对象，使用智能指针
当一个对象应该被释放时，指向它的智能指针可以确保自动地释放它

*** 静态内存用来保存局部static对象、类static数据成员以及定义在任何函数之外的变量 ***
*** 栈内存用来保存定义在函数内的非static对象 ***
分配在这两处的对象由编译器自动创建和销毁

堆区域：动态分配
new delete

### 12.1.1 shared_ptr类
shared_ptr<list<int>>p2;

使用make_shared来分配和使用动态内存
shared_ptr<int>p3 = make_shared<int>(42);

拷贝一个shared_ptr，引用计数会递增。

用局部变量来保存shared_ptr，离开作用域就被销毁。
在return一个shared_ptr时，引用计数递增。

需要注意的是销毁程序不再需要的shared_ptr，否则就浪费内存了，有这么一种情况：将shared_ptr存放在一个容器中，需要使用erase删除不再需要的对象。


### 12.1.2 直接管理内存
int *pi = new int;
int *pi = new int(1024);

动态分配的const对象
const int *pci = new const int(1024);

当内存耗尽，new会失效，抛出一个bad_alloc的异常。

int *p2 = new (nothrow) int;

释放动态内存
delete p;
delete的指针必须指向动态分配的内存/空指针，否则这种行为就是未定义的。

谨防内存泄漏：忘记delete

空悬指针：被delete的指针
可以赋nullptr，表明指针不指向任何对象。

shared_ptr<int> p2(new int (42));
必须使用直接初始化
因为不允许内置指针到智能指针间的隐式转换

get方法将指针的访问权限传递给代码，只有确定代码不会delete指针的情况下，才能使用get

智能指针可以确保即使发生异常，也能释放

不要delete get()返回的指针，因为智能指针并不知道被delete了，会出现重复析构。


### 12.1.5 unique_ptr
一个unique_ptr“拥有”它所指向的对象。
某个时刻只能有一个unique1_ptr指向一个给定对象。

unique_ptr<int>p2(new int(42));
不支持拷贝、赋值

转移unique_ptr所有权：
unique_ptr<string>p2(p1.release());

可以拷贝或赋值一个将要被销毁的unique_ptr
举例：从函数返沪一个unique_ptr
因为编译器知道要返回的对象将要被销毁，编译器会执行一种特殊的拷贝（见十三章）


### 12.1.6 weak_ptr
use_count()方法，返回引用计数



### 12.2.2 allocator类
new不够灵活，因为它把内存分配和对象构造组合在一起
delete类似，把对象析构和内存释放组合在了一起

考虑以下场景：
在分配单个对象时，希望内存分配和对象初始化放在一起，因为我肯定知道对象有什么值。

当分配一大块内存时，我们通常计划在这块内存上按需要构造对象，也就是把内存分配和对象构造分离：可以分配大块内存，但只有真正需要时才真正执行对象创建操作（付出开销）





### allocator类
```cpp
#include<memory>
```
用于将内存分配和对象构造分离
它分配的内存是原始的、未构造的

```cpp

allocator<string>alloc;
auto const p = alloc.allocate(n);
// 为n个string分配了内存

// construct接受一个指针和零个或多个额外参数
// 用于在给定位置构造一个元素,额外参数用来初始化构造的对象

// allocator分配的内存是未构造的，按需要在此内存中构造对象
auto p = q; // q指向最后构造的元素之后的位置
alloc.construct(q++);   // *q为空字符串
alloc.construct(q++, 10, 'c');    // *q为cccccccccc
// 使用未构造的内存是未定义行为
// 只能对真正构造了的元素进行destroy


```


#### 拷贝和填充未初始化内存的算法
```cpp
uninitialized_copy(b, e, b2);
// 从迭代器b和e的输入范围中拷贝元素到迭代器b2指定的未构造的原始内存中
```








# 第十三章 拷贝控制
类内可以定义 ***拷贝控制操作***：拷贝构造、拷贝赋值、移动构造、移动赋值、析构
这些操作若未手动定义，编译器会自动定义（但是可能并非我们想要的）


## 13.1 拷贝、赋值与销毁

### 13.1.1 拷贝构造函数
class Foo{
    Foo(const Foo&);    // 拷贝构造函数 入参必须为引用
}

拷贝构造不应该是explicit的，因为是隐式地使用


#### 合成拷贝构造函数
内置类型的成员直接拷贝、类类型的则需要调用拷贝构造函数

#### 拷贝初始化
string dots(10,'.');    // 直接初始化
string s(dots); // 直接初始化 直接函数匹配调用了拷贝构造函数，所以它是直接初始化，而不是拷贝初始化
string s2 = dots;   // 拷贝初始化
string null_book = "9-999-9";   // 拷贝
string nines = string(100,'9'); // 拷贝

***直接初始化***实际上是编译器使用函数匹配选择构造函数
***拷贝初始化***实际上是编译器将有猜测运算对象拷贝到正在创建的对象中（可能还会涉及类型转换）
C++语言标准规定： ***拷贝初始化***应该是先调用对应的构造函数创建一个 ***临时对象*** ，即额外创建对象
拷贝初始化有时会通过移动构造函数完成

拷贝初始化发生在：
对象作为实参传递给非引用类型的形参
返回类型为非引用类型的函数 返回一个对象
{}列表初始化一个数组中的元素、或一个聚合类中的成员

初始化标准库容器或者调用insert或push时，容器对齐元素进行拷贝初始化
emplace时直接初始化


拷贝构造函数的参数必须是引用类型-->因为如果不是引用，调用永远都不会成功，为了调用拷贝构造函数，我们必须拷贝它的实参，为了拷贝实参，又必须调用拷贝构造函数

如果用explicit声明构造函数，拷贝/直接初始化需要有选择








### 13.1.2 拷贝赋值运算符
#### 重载赋值运算符
赋值运算符实质就是一个名为operator=的函数
Foo operator=(const Foo&);

#### 合成拷贝赋值运算符
编译器会为未定义自己的拷贝赋值运算符的类定义一个 ***合成拷贝运算符***

### 13.1.3 析构函数
用于释放资源，销毁对象的非static数据成员
构造函数中：先初始化成员，再执行函数体
而析构函数恰恰相反
销毁内置类型成员什么都不需要做

隐式销毁一个内置指针类型的成员不会delete它所指向的对象

#### 什么时候调用析构函数
1. 变量离开其作用域时被销毁
2. 对象被销毁时，其成员也被销毁
3. 容器被销毁时，元素被销毁
4. 动态分配的对象，指向它的指针应用delete运算符时被销毁
5. 对于临时对象，当创建完它的完整表达式结束时被销毁

也就是执行到'}'，退出局部作用域，进行调用析构函数，以及引用计数减一等操作

#### 合成析构函数
编译器定义的
析构函数体自身并不直接销毁成员
成员是在析构函数体之后 ***隐含的析构阶段***中被销毁的


### 13.1.4 三/五法则
***如果一个类需要一个析构函数（显示地定义出来），那么也需要拷贝构造函数和一个拷贝赋值运算符***

举例：
类HasPtr在构造函数中分配内存，需要在析构函数中delete一个指针数据成员
采用合成的拷贝构造和拷贝赋值运算符是错误的，因为这会导致简单拷贝指针成员，意味着多个HasPtr对象可能指向相同的内存，导致多次销毁（类的指针成员）。

```c++
class HasPtr{
public:
    HasPtr(const std::string &s = std::string()):ps(new std::string(s)),i(0) {}
private:
    std::string *ps;
    int i;

};
// 为了提供默认参数值，创建了一个匿名的、空字符串的std::string对象作为默认值，也就是说如果没有传入实参，就用这个空字符初始化

HasPtr f(HasPtr hp){
    HasPtr ret = hp;    // 拷贝了hp对象
    return ret; // 返回时，ret，hp对象都被销毁
}

HasPtr p("some values");    // 构造p对象
f(p);   // 当f执行完时，p.ps指向的内存被释放（因为是临时变量）
HasPtr q(p);    // p和q都指向无效内存


```

需要拷贝操作的类也需要赋值操作，不一定需要自定义析构函数

***如果一个类需要一个拷贝构造函数，几乎可以肯定它也需要一个拷贝赋值运算符***



### 3.15 使用=default
显式要求编译器生成合成的版本

### 13.1.6 阻止拷贝
有些类必须阻止拷贝或者赋值
iostream类组织了拷贝，以避免多个对象写入或读取相同的IO缓冲
因此需要 = delete
析构函数不能删除
如果删除了析构函数，则不能把对象定义在栈上了，但是可以定义在堆上，但同样无法释放。

如果一个类有数据成员不能默认构造、拷贝、复制或销毁  则对应的成员函数被定义为删除

#### private拷贝控制
private声明（不需定义）可以阻止友元和成员函数进行拷贝
如果试图拷贝则会在编译时标记为错误
成员函数或友元函数中的拷贝操作会导致链接错误


## 13.2 拷贝控制和资源管理
通常管理类外资源的类必须定义拷贝控制成员

### 13.2.1 行为像值的类

#### 类值拷贝赋值运算符
赋值运算符通常组合了析构和构造
因为会销毁左侧运算对象的资源 从右侧运算对象拷贝数据
```c++
HasPtr& HasPtr::operator=(const HasPtr &rhs){
    auto newp = new string(*rhs.ps);
    // 拷贝底层string
    delete ps;  // 释放旧内存
    ps = newp;  // 拷贝数据
    i = rhs.i;
    return *this;
}

// 先将右侧对象拷贝到局部对象，再销毁左侧对象是安全的
```

需要防范 ***自赋值操作***
要保证一个对象赋予它自身，也要正确工作
因为两个指针可能指向同一块内存

### 13.2.2 定义行为像指针的类
要定义拷贝构造函数和拷贝赋值运算符
使用shared_ptr
引用计数的计数器保存在动态内存中，当拷贝或赋值对象时，拷贝指向计数器的指针，目的是让副本和原对象都指向相同的计数器

PS: std::size_t
范围是目标平台下最大可能的数组尺寸


## 13.3 交换操作
目的是减少拷贝


## 13.5 动态内存管理类
vector原理：连续内存 若无空间则重新分配，把旧元素移动到新元素，释放旧空间

#### 移动构造函数和std::move
#include<utility>
std::move

## 13.6 对象移动
移动而非拷贝对象 拷贝是不必要的，或者有些类包含不能被共享的资源（如指针或IO缓冲）


### 13.6.1 右值引用
***rvalue reference***
用于支持移动操作
即必须绑定到右值的引用
只能绑定到一个将要销毁的对象
本质上就是某对象的另一个名字而已
不能将一个右值引用绑定到一个左值上

1. 所引用的对象将要销毁
2. 该对象没有其他用户

由于变量是左值，因此不能将右值引用绑定到一个右值引用类型的变量上
int &&rr1 = 42;
int &&rr2 = rr1;    // 错误

```c++
int &&rr3 = std::move(rr1);
//  将一个左值转换为对象的右值引用类型
// 获得绑定到左值的右值引用
```
move意味着：我们有rr1这个左值，但我们希望像一个右值一样处理它，除了对rr1赋值或者销毁外，不再使用它

可以销毁移后源对象 赋值 但不能使用一个移后源对象的值


### 13.6.2 移动构造函数和移动赋值运算符
移动构造不分配任何新内存 "窃取"资源而非拷贝（因而不抛出异常）

PS：移动操作时，一个对象的资源所有权从源对象转移到目标对象
底层使用指针或引用，通常将源对象的数据成员指针设置为null，以避免在析构源对象时释放已经转移的资源
移动赋值运算符中，先释放目标对象的资源，然后将源对象的资源指针赋给目标对象的资源指针

一旦资源完成移动，源对象必须不再指向被移动的资源

移后源对象必须可析构

若类成员定义了自己的拷贝构造函数且未定义移动构造函数，则移动构造被定义为delete

右值可以作为拷贝构造函数的入参

移动迭代器

不要随意使用移动，因为移后源对象具有不确定的状态，必须确认移后源对象没有其他用户

### 13.6.3 右值引用和成员函数
参数列表后加&，表示引用限定符
&表示this可以指向左值，&&则是右值
Foo anotherMem() const &;


PS：移动语义是把左值转换成右值，转移值的所有权，移后源变量不再拥有原始值的所有权。（但由于C++标准允许程序员对临时对象优化、延迟销毁等，因而可以赋值）减少了拷贝。



# 第十四章 操作重载与类型转换

## 14.8 函数调用运算符
指的是()符号

### 14.8.1 lambda是函数对象
lambda可以通过***引用***或者***值***捕获变量






### 14.8.3 可调用对象与function
C++中的可调用对象：
函数、函数指针、lambda表达式、bind创建的对象、重载了函数调用运算符的类

























# 第十六章 模板与泛型编程
泛型编程：编译时获知类型

### 16.1.1 函数模板
template<typenam T> // template后是模板参数列表
int compare(const T &v1,const T &v2){
    if(v1 < v2) return -1;
    if(v2 < v1) return 1;
    return 0;
}

使用模板时，隐式或者显式地指定模板实参，将其绑定到模板参数上

*** 实例化函数模板 ***
compare(1,0)    // 编译器会推出模板实参为int
编译器生成的版本被称为模板的实例

*** 模板类型参数 ***
template <typename T> T foo(T* p)
{
    T tmp = *p;
    // ...
    return tmp;
}

模板参数列表中，typename和class可以互换使用

*** 非类型模板参数 ***
一个非类型模板参数表示一个值而非一个类型
非类型模板参数被 ***常量表达式***替代
比如用于指定数组大小


***编写类型无关的代码***
将函数参数设定为const的引用，**保证了函数可以用于不能拷贝的类型。

只用<，而不用>，可以降低compare函数对要处理的类型的要求（也就是说，只需重载<）

如果更加追求类型无关和可移植性
template<typename T>int compare(const T &v1,const T &v2)
{
    if(less<T>()(v1,v2))    return -1;
    if(less<T>()(v2,v1))    return 1;
    return 0;
}


***模板程序应该尽量减少对实参类型的要求***


只有实例化出模板的一个特定版本，编译器才会生成代码，使用而非定义模板时才会生成代码

*** 函数模板和类模板成员函数的定义通常放在头文件 ***

大多数编译错误在实例化期间报告:
通常编译器会在三个阶段报告错误:

第一个阶段是**编译模板本身**时. 在这个阶段编译器通常不会发现很多错误, 编译器检查语法错误.
第二个阶段是编译器遇到模板使用时. 在此阶段, 编译器仍然没有很多可以检查的, 对于函数模板调用, 编译器通常会**检查实参数目是否正确, 还能检查参数类型是否匹配, 对于类模板, 编译器可以检查用户是否提供了正确数目的模板实参**.
第三个阶段是模板实例化时, 只有这个阶段才能发现**类型相关**的错误, 依赖于编译器如何管理实例化, 这类错误可能在链接时报告.






### 16.1.2 类模板
使用类模板必须显式地在<>添加信息

对于一个实例化了的类模板，其成员只有在使用时才被实例化。

在类模板自己的作用域中，可以直接使用模板而不提供实参。

类模板外使用类模板名需要显式地指出<T>


### 16.1.3 模板参数
模板声明
模板声明必须包含模板参数
如函数参数，声明中的模板参数的名字不必与定义中相同。

使用类的类型成员
默认情况下，c++假定通过作用域运算符访问的名字不是类型
如果要用模板类型参数的类型成员，就必须显式地告诉编译器该名字是一个类型
这需要typename关键字

template<typename T>
...
    return typename T::value_type();
容器类型中，提供初始值以进行值初始化
T()这样的表达式是用于显式地请求值初始化


#### 默认模板实参
template<typename T,typename F = less<T>>
int compare(const T & v1,const T &v2, F f = F())
{
    if(f(v1,v2))    return -1;
    if(f(v1,v2))    return 1;
    return 0;
}

默认模板实参less<T>，默认函数实参F()

第二个类型参数F，表示可调用对象的类型，函数参数f，会被绑定到一个可调用对象上。

默认模板参数指出：compare将使用标准库的less函数对象类，它是使用与compare一样的类型参数实例化的
默认函数实参指出：f将是类型F的一个默认初始化的对象

当用户将该模板函数实例化时，可以提供自己的比较操作（也可以不提供）

bool i = compare(0,42); // 使用less，i = -1
// 使用默认函数实参，类型less<T>的一个默认初始化对象
// T为int，即less<T>

bool j = compare(item1, item2, compareIsbn);
// 传入了两个对象和一个可调用对象，该对象的返回类型必须可以转换为bool值，且接受实参类型必须与前面两个实参的类型兼容 这时的T是Sales_data类型，F被推断为compareIsbn类型

#### 模板默认实参与类模板
template<class T = int>class Numbers{

}





### 16.1.4 成员模板
即模板成员函数（不能是虚函数）
普通类可以包含模板成员函数
类模板也可以定义模板成员函数，二者的模板参数可以独立
在类外定义时就要同时为二者提供参数列表。
template <typename T>
template <typename It>
    Blob<T>::Blob(It b,It e);

实例化与成员模板
实例化类模板的成员模板，需要同时提供二者的实参







### 16.1.5 控制实例化
由于在多个文件中实例化相同模板的额外开销可能很严重（因为没必要），因此需要 ***显式实例化***。
例如：

```c++
extern template class Blog<string>; // 声明
// extern 意味着编译器不会在本文件生成实例化代码，因为其他位置已有非extern声明（定义）
template int compare(const int&,const int&);    // 定义
```


## 16.2 模板实参推断

### 16.2.1 类型转换与模板类型参数
顶层const传递给函数模板会被忽略

如果函数形参是常量引用，但是传入的是指针（来源于数组），是无法转换的。
将实参传递给带模板类型的函数形参时，能够自动应用的类型转换只有const转换、以及数组/函数到指针的转换。


使用相通模板参数类型的函数形参
compare(const T&,const T&)
long lng;
compare(lng,1024)  // 二者类型必须相同

--> 可以这样
template<typename A,typename B>

#### 正常类型转换用于普通函数实参
template<typename T>ostream & print(ostream &os,const T &obj){
}


### 16.2.2 函数模板显式实参
指定显式模板实参

```c++
template<typename T1,typename T2,typename T3>
T1 sum(T2,T3);
// 无法通过函数实参的类型推断T1的类型
// 必须显式指定
auto val3 = sum<long long>(i,lng);  // int long
```
第一个模板实参与第一个模板参数匹配，第二个实参和第二个参数匹配
只有最右侧参数的显式模板实参可以忽略


### 16.2.3 尾置返回类型与类型转换
template <typename It>
auto fcn(It beg,It end) -> decltype(*beg)
因为尾置返回出现在参数列表之后，所以可以使用函数的参数

标准库的类型转换：type_traits
用于模板元编程
remove_reference可以去除引用，得到本身的类型
template <typename It>
auto fcn2(It beg,It end) -> 
typename remove_reference<decltype(*beg)>::type
{
    return *beg;
}
// type表示一个类型 是类的成员


### 16.2.4 函数指针和实参推断
template <typename T>int compare(const T&,const T&);
int (*pf1)(const int&,const int&) = compare;
// 定义了指向compare的函数指针
// pf1中参数的类型决定T的模板实参的类型


### 16.2.5 模板实参推断和引用
从左值引用函数参数推断类型
当函数参数为T&这种左值引用时，智能传递左值（一个变量或一个返回引用类型的表达式）

如果函数参数的类型是const T&，我们可以传递给它任何类型的实参：一个对象(const/非const)、一个临时对象、一个字面常量值

当一个函数参数是一个右值引用（形如T&&）时，
可以传递给它一个右值

#### 引用折叠和右值引用参数
```c++
template <typename T>void f3(T&&);
```
f3(i)   // 编译器推断T的类型为int&，而不是int
通常不能定义一个引用的引用，但是通过类型别名或者模板类型参数间接定义是可以的
如果我们间接创建一个引用的引用，则形成了引用折叠-->引用会折叠成一个普通的左值引用类型
   

指向模板类型参数的右值引用可以绑定到一个左值
我们可以将任意类型的实参传递给T&&类型的函数参数

实际中，右值引用常用于两种情况：
模板转发其实参  /   模板被重载




### 16.2.6 理解std::move
不能直接将一个右值引用绑定到一个左值，但可以用move获得一个绑定到左值上的右值引用





### 16.2.6理解std::move
#### 它是如何定义的？
```c++
template <typenmae T>
typename remove_reference<T>::type&& move(T&& t)
{
    return static_cast<typename remove_reference<T>::type&&>(t);
}
// 此处的type指的是T原本的类型（本质上和remove_reference的底层实现有关，该模板类有个成员是type）
// 通过引用折叠，此参数可与任何类型的实参匹配（既可以给move传递左值，又可以传递右值）

```
PS：引用折叠
int& &&等价于int&



#### 它是如何工作的？

推断出T的类型（比如string&，而非普通string）

虽然不能隐式将一个左值转换为右值引用，但可以用static_cast显示地完成


### 16.2.7 转发
需要将实参连同类型不变地转发给其他函数，需要保持被转发实参的性质（const、左值、右值）
```c++
// 该模板接受一个可调用对象和另外两个参数
template<typename F,typename T1,typename T2>
void flip1(F f,T1 t1,T2 t2)
{
    f(t2,t1);
}
// 这样的模板参数没法接收引用
```
模板实例化后的调用：
void flip(void(*fcn)(int,int&),int t1,int t2);
t1（不带引用）再传入第一个位置的函数

由于使用引用参数可以保持const属性，可以将参数定义为T1&&，通过引用折叠保持翻转实参的左值/右值属性

```c++
template<typename F,typename T1,typename T2>
void filp2(F f,T1 &&t1,T2 &&t2){
    f(t2,t1);
}

```
flip2(g,i,42);
但是不能传递右值给filp2
因为传给g的是flip2中名为t2的参数，传进g的右值引用参数的实参是一个左值

***std::forward***可以保持给定实参的左值/右值属性
如果实参是右值，则Type为普通类型，forward<Type>返回Type&&
如果实参是左值，则通过引用折叠，返回一个左值引用类型


## 16.3 重载与模板
重载模板
精准匹配
选择更特例化的版本
非模板和模板版本，优先非模板


## 16.4 可变模板参数
接受可变数目参数的模板函数/模板类
可变数目的参数被称为 ***参数包***
（模板参数包/函数参数包）

```c++
template<typename T,typename... Args>
void foo(const T &t,const Args& ...rest);
// foo是一个可变参数函数模板，有一个名为T的类型参数，有一个名为Args的模板参数包
// foo的函数参数列表包含一个const&类型的参数，指向T的类型，还包含一个名为rest的函数参数包，此包表示0/多个函数参数
```
foo(i,str,42,d);    //包中有三个参数
foo("hi");  //空包

可以使用sizeof运算符来得知包中元素的个数

```c++
template<typename T>
ostream &print(ostream &os,const T &t){
    return os << t;
}
// 用于终止递归，并打印最后一个元素

template<typename T,typename... Args>
ostream &print(ostream &os,const T &t,const Args&... rest)
{
    os<< t <<",";
    return print(os,rest...);   // 递归
    // 打印其他实参
}

// 注意到：
return print(os,rest...);
// 递归调用
// 只传递了两个实参，rest中的第一个实参被绑定到T，剩余实参形成下一个print调用的参数包
```
比如执行：
print(cout,i,s,42); // 包中两个参数
print(cout,s,42);
print(cout,42); // 此时调用非可变参数版本的print
原因是非可变参数模板必可变参数模板更加特例化
编译器选择非可变参数版本
（要把非可变参数版本声明在作用域中，否则无限递归）






### 16.4.2 包扩展
对于参数包，要么获取大小，要么就是扩展
扩展时要提供用于每个扩展元素的 ***模式***

const Args&... rest // 扩展Args 生成函数参数列表

return print(os,rest...);
// 扩展rest 为print调用生成实参列表

模式是函数参数包的名字（rest）

return print(os,debug_rep(rest)...);

扩展结果是一个逗号分隔的debug_rep调用列表

16.4.3 转发参数包
组合 ***可变参数模板***和 ***forward机制***


## 16.5 模板特例化
```c++
template<>  // 空尖括号表示为原模版提供实参
int compare(const char* const &p1,const char* const &p2)
{
    return strcmp(p1,p2);
}
// 这是template <typename T>int compare(const T&,const T&);的特例化版本


```
特化一个函数模板时，必须为原模版中的每个模板参数提供实参
本质上是接管了编译器的工作
特例化版本的产物已经是实例，特例化不影响函数匹配

#### 类模板特例化
可以部分特例化类模板（模板参数；列表是原始模板的参数列表的子集或者特例化版本），而不能部分特例化函数模板

可以只特例化成员函数而不是特例化整个模板





## 多重继承和虚继承
### 18.3.1 多重继承
如果没有显式初始化基类，就会隐式调用默认构造函数
基类的构造顺序与派生列表中基类的出现顺序保持一致

派生类可以继承基类的构造函数，但是如果继承了多个相同的构造函数，必须定义一个自己的版本

注意派生类的析构函数用于清除派生类本身分配的资源，成员和基类都是自动销毁的


### 18.3.2 类型转换与多个基类
派生类不会在派生类向若干基类的转换中比较和选择
因此，倘若有一个函数，重载的参数是一个派生类的不同层次的基类，是无法通过把派生类对象传入来调用重载函数的


#### 基于指针类型或引用类型的查找
用基类指针指向派生类对象时，无法调用派生类对象特有的接口，这部分是不可见的。


### 18.3.3 多重继承下的类作用域
当一个类继承了多个基类，它可能继承了同名成员，访问同名成员必须加上前缀限定符。


### 18.3.4 虚继承
派生类可以多次继承同一个类，因为存在间接的继承关系
这种包含了重复的继承会导致不想要的效果
因此需要***虚继承***，这样的话派生类只包含一个共享的虚基类子对象

举例：B和C虚继承了A，D再继承B和C。
实际上在定义D时，才出现对虚派生的需求（B和C如果不是虚继承，D则会保留两份A的成员）。

虚派生只影响从指定了虚基类的派生类中进一步派生出的类，它不会影响派生类本身

#### 使用虚基类
class Bear: virtual public ZooAnimal{}

派生类的成员x比共享虚基类的x优先级更高


### 18.3.5 构造函数与虚继承
虚继承情况下的构造顺序：









# 第十九章
### 19.1.1 重载new和delete
#### new的原理
new表达式调用了一个名为operator new/operator new[]的标准库函数，
1. 分配一块足够的、原始的、未命名的内存空间以储存对象
2. 然后编译器允许构造函数来构造对象
3. 接着对象被分配空间，构造完成，返回指向该对象的指针

delete表达式：
1. 执行指针所指的对象的析构函数
2. 编译器调用operator delete标准库函数来释放内存空间
PS：析构函数不释放空间，只是删除对象，那块内存未被释放

#### malloc和free
cstdlib
malloc：入参为字节数，返回指向分配空间的指针，若返回9则表示失败
free：接受一个void*，释放内存



## 19.2 运行时类型识别
RTTI:
run-time type identification
由两个运算符实现
typeid运算符(返回表达式的类型)
dynamic_cast运算符(用于将基类的指针或引用安全地转换成派生类的指针或引用)

这两个运算符的适用情况:
想要使用基类对象的指针或引用来执行某个派生类的操作并且该操作不是虚函数.
(因为如果是虚函数的话,编译器直接就根据对象的动态类型自动选择正确的函数版本,即多态)
所以无法使用虚函数的话,就得使用RTTI运算符了(必须注意类型转换是否成功执行)


### 19.2.1 dynamic_cast运算符
在运行时把指向基类的指针转为指向派生类的指针


#### 指针类型的dynamic_cast
```cpp
if(Derived *dp = dynamic_cast<Derived*>(bp))
// 因为bp这个基类指针既可能指向基类，又可能指向派生类
// 如果通过以上代码用类型转换来初始化dp，并令其指向bp所指的Derived对象
```
若bp指向Derived对象，则if成立
若dp为0则说明if语句失败，即bp指向的是Base对象

这样在if部分定义dp是同时完成了 ***类型转换***和 ***条件检查***
实际上指针dp（未绑定）在if语句外是不可访问的
保证程序安全




### 19.2.2 typeid运算符
typeid(表达式)
其结果是std::type_info(或其公有派生类型)的常量对象引用
顶层const会被忽略
如果对数组a执行typeid,得到的结果是数组类型而非指针类型

当运算对象不属于类类型或者是一个不包含任何虚函数的类时,
typeid运算符指示的是运算对象的静态类型
如果运算对象是定义了至少一个虚函数的类的左值时,typeid的结果直到运行时才会求得

```cpp
    Derived* dp = new Derived;
    Base* bp = dp;  // 基类和派生类指针都指向派生类
    // 运行时比较两个对象的类型
        if(typeid(*bp) == typeid(*dp))
            // 指向同一类型的对象,所以等号成立

        if(typeid(*bp) == typeid(Derived))
            // bp实际指向Derived 等号成立

```
当typeid作用于指针时,返回的结果时该指针的静态编译时类型
**只有当表达式含有虚函数 编译器才会求值 反之则是静态类型 不需要求值**




### 19.2.3 使用RTTI
使用场景：
想为具有继承关系的类实现相等运算符



### 19.2.4 type_info类
提供==、!=、name()(返回C风格字符串)、before(t2)(判断t1是否位于t2之前)
我们无法定义或者拷贝type_info类型的对象，也不能赋值
**创建type_info对象的唯一途径是使用typeid运算符**



## 19.3 枚举类型
目的是将一租整型常量组织在一起，枚举属于字面值常量类型
C++包含两种：限定作用域 和 不限定作用域
C++11引入了限定作用域的枚举类型
```cpp
enum class open_modes
{
    input,output,append
};  // 限定作用域的枚举类型
om = open_modes::input;

enum color{red, yellow, green}; // 不限定作用域的枚举类型
enum {floatPrec = 6,doublePrec = 10 };
// 未命名的enum，需要在定义时定义其对象

```

#### 枚举成员
```cpp
enum color{red, yellow, green};
// 不限定作用域的枚举类型
enum stoplight{red, yellow, green};
// 不能再定义枚举成员了，因为上面定义过了
enum class peppers{red, yellow, green};
// 枚举成员被隐藏
color eyes = green; // 有效
peppers p = green;  // 错误，peppers的枚举成员不在有效作用域中
// 必须这样访问
peppers p2 = peppers::red;  // 需要增加peppers作用域
color hair = color::red;    // 可以显式


```
默认情况下，枚举值从0开始，依次加1
也可以指定值 如果没有显示提供初始值，则当前枚举成员的值等于之前枚举成员的值加1
枚举成员是const，因此初始值必须是常量表达式

不限定作用域的枚举类型的对象或者枚举成员可自动转换成整型
int i = color::red;
而限定作用域的枚举类型不能隐式转换成整型

#### 指定enum的大小
```cpp
enum intValues: unsigned long long{
    charTyp = 255,

};
```
对于限定作用域enum，不指定类型的话，默认就是int
不限定作用域的enum，不存在默认类型

#### 枚举类型的前置声明
enum的前置声明必须指定其成员的大小


#### 形参匹配与枚举类型

要初始化一个enum对象，必须用enum类型的对象或者其枚举成员，直接用整型值是不行的



## 19.4 类成员指针


### 19.4.1 数据成员指针
成员指针必须包含成员所属的类
```cpp
    class Screen{
        // 省略
        private:
            std::string contents;
    };


    const std::string Screen::*pdata;
// 声明了一个指向Screen对象的const string成员的指针
```
常量对象的数据成员本身也是常量
pdata可以指向任何Screen对象的一个成员，而不管Screen对象是否是常量。（是设定）
但是只能使用pdata读取它所指的成员，而不能向它写入内容
初始化/赋值成员指针，必须指定所指的成员

```cpp
pdata = &Screen::contents;  // 表示pdata指向某个非特定Screen对象的contents成员
```
注意取地址符&的作用对象是Screen类的成员而非内存中的一个该类对象



#### 使用数据成员指针
```cpp
Screen myScreen, *pScreen = &myScreen;
auto s = myScreen.*pdata;   // pdata是指向某个Screen对象的contents成员的指针，解引用取得contents对象s

```

#### 返回数据成员指针的函数
因为数据成员一般是私有的，所以不能直接获得数据成员的指针，除非定义这么一个函数，返回指向该成员的指针

```cpp
static const std::string Screen::*data()   // 返回类型表示返回指向某个Screen对象的const string成员的指针
{
    return &Screen::contents;   // 返回指向contents成员的指针
}

const string Screen::*pdata = Screen::data();
// 绑定到Screen类型的对象上
auto s = myScreen.*pdata;

```


### 19.4.2 成员函数指针
```cpp
auto pmf = &Screen::get_cursor;
// pmf是指针，指向成员函数

char (Screen::*pmf2)(Screen::pos, Screen::pos) const;
// 声明了一个指针，类型是指向Screen的成员函数，参数列表如上
pmf2 = &Screen::get;    // 必须显式调用取地址符

```

#### 使用成员函数指针
```cpp
    Screen myScreen, *pScreen = &myScreen;
    char c1 = (pScreen->*pmf)();    // 调用指向成员函数的pmf指针
    /* pScreen和pmf怎么联系起来？：
    pScreem指向Screen对象
    pmf是指向Screen类的get_cursor成员函数的指针
    */

    char c2 = (myScreen.*pmf2) (0,0);
    // myScreen的成员函数get

    // 需要注意：要用括号把myScreen.*pmf括起来
    // 否则含义等同：myScreen.*(pmf())
    // 调用一个名为pmf的函数，然后其返回值作为指针指向成员运算符(.*)的运算对象
        
```


#### 使用成员指针的类型别名
取别名使成员指针更容易理解
```cpp
using Action = 
char (Screen::*)(Screen::pos, Screen::pos) const;
// Action代表的类型是：指向Screen类的常量成员函数的指针，参数列表在其中，返回值是char类型

// 用法
Action get = &Screen::get;

// 作为形参
Screen& action(Screen&, Action = &Screen::get);

Screen myScreen;
action(myScreen, &Screen::get);
```

#### 成员指针函数表
定义一个类，其中的成员函数都不接受参数，返回对象的引用
这些函数可以作为数组的成员，形成一个函数表


### 19.4.3 将成员函数用作可调用对象
成员指针和普通的函数指针不同，它不是一个可调用对象，不支持函数调用运算符（必须用.*或者->\*运算符将指针绑定到特定对象上）



#### 使用function生成一个可调用对象
```cpp
std::function<bool (const string&)> fcn = &string::empty;
// empty是一个接收string参数并返回bool的函数
```
执行成员函数的对象将被传给隐式的this形参，想要使用function为成员函数生成一个可调用对象时，必须使得隐式的形参变成显式的
本质上，function类将函数调用传换了形式：

```cpp
// fcn是可调用对象的名字
if(fcn(*it))
// it是find_if内部的迭代器，*it是给定范围内的一个对象
--> 变为：
if(((*it).*p) ())
// p是fcn内部的一个指向成员函数的指针

```

使用function对象必须指定该对象表示的函数类型，如果是成员函数，第一个形参必须表示该成员是在哪个对象上执行的
并且指明对象是否是以指针或引用的形式传入的

eg
```cpp
// 想要在string对象的序列上调用find_if
vector<string*>pvec;
std::function<bool(const string*)> fp = &string::empty;
// fp接收一个指向string的指针，然后使用->*调用empty
find_if(pvec.begin(),pvec.end(), fp);
// 重点是function的形参string表明了成员函数是在哪个对象上执行的
```

#### 使用mem_fn生成一个可调用对象
让编译器负责推断成员的类型
mem_fn可以根据成员指针的类型推断可调用对象的类型，无须显式指定
```cpp
find_if(svec.begin(), svec.end(), mem_fn(&string::empty));

```
mem_fn生成的可调用对象可以通过对象调用，也可以通过指针调用
```cpp
auto f = mem_fn(&string::empty);
f(*svec.begin());   // 传入一个string对象，f使用.*调用empty

f(&svec[0]);    // 传入一个string的指针，f使用->*调用empty 显然，因为是指针所以使用->取得成员
// 可以理解为mem_fn生成的可调用对象含有一对重载的函数调用运算符
```

#### 使用bind生成一个可调用对象
```cpp
auto it = find_if(svec.begin(), svec.end(), std::bind(&string::empty, _1));
// 必须将函数中用于表示执行对学校的隐式形参转换成显式的，bind生成可调用对象的第一个实参可以是指针/引用
f(*svec.begin());
f(&svec[0]);

```


## 19.5 嵌套类
常用于定义作为实现部分的类
嵌套类是一个独立的类，与外层类基本没什么关系
自然，外层类的对象和嵌套类的对象是相互独立的（二者在定义时，成员之间是无关的）
**嵌套类的名字在外层类作用域中是可见的，在外层类作用域之外就不可见了**

**重点**：嵌套类被视为外部类的成员（作用域和外部类相同），它们享有与外部类相同的访问权限。

注意**嵌套类定义的类型的访问控制**：
由于嵌套类在其外层类中定义了一个类型成员，和其他成员类似，这个类型的（对外）访问权限由外层类决定
1. 位于外层类public部分的嵌套类实际上定义了一种可随处访问的类型
2. 位于外层类protected部分的嵌套类定义的类型，只能被外层类及其友元和派生类访问
3. private的话，只能被外层类的成员和友元访问
（PS：之所以嵌套在别人的类里，就是为了便于访问这个类）
嵌套类可以访问外部类的所有成员，包括私有成员。嵌套类可以用于实现更好的封装和组织代码的结构。

常见做法是：把嵌套类作为外部类的成员

也可以在外层列之外定义一个嵌套类，比如：
```cpp
class TextQuery::QueryResult{
    public:
        // 嵌套类可以直接使用外层类的成员，不需要对名字做限定（内层可以用外层）
};
```

### 嵌套类的静态成员定义
```cpp
int TextQuery::QueryResult::static_mem = 1024;  // 这样访问
```

### 嵌套类和外层类是相互独立的

## 19.6 union
可以有多个数据成员，但是任意时刻只有一个数据成员可以有值
当给其中某个成员赋值后，其他成员就变成未定义状态了
可见，分配给一个union对象的空间至少要能容纳它的最大数据成员

union不能含有引用类型的成员
但可以包含绝大多数类型（一些自定义类型，只要有构造和析构）
但由于union既不能继承自其他类，也不能作为基类，所以union中不能含有虚函数

#### 定义union
```cpp
union Token{
    char cval;
    int ival;
    double dval;
}
// Token可以保存以上类型中的一种 
```

#### 使用
```cpp
    Token first_token = {'a'};
    Token last_token;
    Token *pt = new Token;  // 指向一个未初始化的Token对象的指针
    // 初始值用于初始化 第一个 成员cval
    last_token.cval = 'z';
    pt->ival = 42;
    // 对一个数据成员赋值会使其他成员变成未定义状态
```

#### 匿名union
```cpp
// 一旦定义了一个匿名union，编译器就会自动为该union创建一个未命名的对象
union{
    char cval;
    int ival;
    double dval;
};
// 对于未命名对象，可以直接访问其成员 只要在定义匿名union的作用域内
cval = 'c';
ival = 42;

```


#### 含有类类型成员的union
对于自定义类型的成员，想要改变union保存的值比较复杂：



# C++相关的补充：

#### std::bind
对于一个可调用对象，可以减少可调用对象传入的参数
```c++
int TestFunc(int a,char c,float f)
auto bindFunc1 = bind(TestFunc,std::placeholders::_1,'A',100.1);
// 然后可以直接调用bindFunc1，如同一个新函数
bindFunc1(10);


auto bindFunc3 = bind(TestFunc, std::placeholders::_2,std::placeholders::_3, std::placeholders::_1);

bindFunc3(100.1, 30, 'C');
/*意思是：
bindFunc3的第二个参数和TestFunc的第一个参数匹配

*/

```


size_t
size_t在32位架构中定义为：typedef   unsigned int size_t；
size_t在64位架构中被定义为：typedef  unsigned long size_t；
size_t是无符号的，并且是平台无关的，表示0-MAXINT的范围；int为是有符号的；


std::packaged_task

std::bind


#### 异步编程 std::future
C++的std::future是一种异步编程的工具，它可以在一个线程中执行一个任务，并在另一个线程中获取该任务的结果。具体来说，std::future可以将一个函数的返回值异步地传递给另一个线程，从而实现并行计算。

***get_future***是std::promise的一个成员函数，它***返回一个std::future对象***，***用于获取promise的值***。promise是一种线程间通信的机制，它可以在一个线程中设置一个值，并在另一个线程中获取该值。get_future函数返回的std::future对象可以在另一个线程中调用get函数来获取promise设置的值。

例如，我们可以在一个线程中使用std::promise设置一个值，然后在另一个线程中使用std::future获取该值：

```c++
#include <iostream>
#include <future>

void setValue(std::promise<int>& p) {
    p.set_value(42);    // promise变量如同中介，提供set_value()方法
}

int main() {
    std::promise<int> p;
    std::future<int> f = p.get_future();    // 得到一个future对象，promise对象提供get_future()方法
    // 绑定之后 promise对象的值的改变都可以通过get()方法得到
    std::thread t(setValue, std::ref(p));
    /* std::ref 用于函数式编程 结合std::bind
    由于bind是直接对参数拷贝，无法传入引用，std::ref因此可用于模板传参
    */
    t.join();

    std::cout << "The value is: " << f.get() << std::endl;

    return 0;
}
```
在上面的例子中，我们创建了一个std::promise对象p，并使用get_future函数获取了一个std::future对象f。然后，我们在另一个线程中调用setValue函数，该函数将42设置为promise的值。最后，我们在主线程中调用f.get()函数获取promise的值，并输出到控制台上。

总之，std::future和std::promise是C++中实现异步编程的重要工具，它们可以帮助我们实现并行计算和线程间通信。


#### 条件变量
考虑生产者消费者场景
很容易想到两个问题：
1、生产者生产好了但是消费者还在沉睡（为了释放CPU资源，而不是一直忙等），资源没有及时使用
2、消费者沉睡结束时生产者还在沉睡，导致不必要的CPU消耗

```c++
// 条件变量有notify_one()、notify_all()、wait()方法
#include <thread>
#include <mutex>
#include <atomic>
#include <iostream>
#include <unistd.h>
#include <condition_variable>
 
std::mutex mtx;
std::condition_variable cond;   // 定义条件变量
 
void func1(int& apple)
{   // 对应生产者
    while (1)
    {
        {
            std::unique_lock<std::mutex> lck(mtx);
            apple += 1; 
            cond.notify_one();//唤醒一个因条件变量不满足而等待的线程
        }
        usleep(1000 * 1000);
    }
}
 
void func2(int& apple)
{   // 对应消费者
    while (1)
    {
        std::unique_lock<std::mutex> lck(mtx);
        if (apple) {
            std::cout << apple << '\n';
            apple -= 1;
            continue;
        }
 
        cond.wait(lck);//等待线程
    }
}
 
int main()
{
    int apple = 0;
    std::thread t1(func1,std::ref(apple));
    std::thread t2(func2, std::ref(apple));
    t2.join();
    t1.join();
    return 0;
}
```

#### 优先队列
特殊的队列，不是先进先出
而是按照优先级，一般用堆来实现
（相比普通队列，加上了内部的排序）
其实就是最大堆或者最小堆


#### shared_ptr坑点

```cpp
int *p = new int;
shared_Ptr<int>ptr1(p);
shared_Ptr<int>ptr2(p);
// 注意，构造ptr1和ptr2都调用了shared_ptr的构造函数，都会重新开辟引用计数的资源（shared_ptr的原理：其对象持有两部分指针，一部分指向资源，一部分指向引用计数）
// 因此析构的时候，都认为自己需要去释放资源，就释放了两次
// 所以应该调用拷贝构造函数，只做引用计数的增减
```

如果一个类要提供一个函数接口，返回一个指向当前对象的shared_ptr智能指针，可以：
继承enable_shared_from_this类，然后通过调用基类继承来的shared_form_this()方法
（目的是不调用额外的shared_ptr的构造函数）


#### std::add_pointer
用于将给定类型**转换为指针类型**。它接受一个类型作为模板参数，并返回该类型的指针类型（在模板编程和类型转换方面常用）。


#### string_view
c++17加入了std::string_view
很多情况下优于std::string（尤其是函数形参，优于const std::string&）
它只是用来指代“字符串”（这个字符串可以指代的东西很多）的，并不拥有所有权，自然也**不可变**。
通常实现只保有两个数据成员：
1. 一个指向字符串的指针
2. 一个表示字符串的的size（通常是size_t类型）。
通常64位系统下大小为16个字节。


##### std::string
```cpp
class string {//简单示例，实际不可能如此
public:
    // all 83 member functions
private:
    char* m_data;
    size_type m_size;
    size_type m_capacity;
    std::array<char, 16> m_sso;
};
对于 64 位系统，每个字符串std::string有 24 个字节的“开销”（size，capacity，data），另外还有 16 个字节用于 SSO 缓冲区。
加起来也就是40。
```

##### 实际使用
```cpp
void func(const std::string&s){
    std::cout << s << '\n';
}

// 在很多传参的情况下，开销很大
std::string s{"abc"};
func("abc");    // 开销很大 实际传入的是字符串字面量（字符数组），先隐式转换为指针，然后再调用std::string的构造函数，构造一个临时string变量
func(s);
// 而const std::string&s这种形参类型是可以被赋纯右值表达式的，顺便延长了临时对象的生命周期

// 另外，使用const std::string&还更容易造成一些bug，比如：
 const std::string& f(const std::string& str) {
    return str;
}
int main() {
    auto& ret = f("abc");  // 根本不能传入临时的字符数组，会导致返回其引用（悬垂引用）
    std::cout << ret << '\n';
}

// 使用std::string_view
void func(std::string_view s){
    std::cout << s << '\n';
}

int main(){
    std::string s{"abc"};
    const char* p = "abc";
    func("abc");
    func(s);
    func(p);
}
```
**std::string有一个到std::string_view的转换函数，其他的都是正常走std::string_view的构造函数。**


#### std::atomic
首先要知道**原子操作**：该操作是连续执行，不可分割的
std::atmoic用于保护一个变量
而常规的互斥量保护的数据范围比较大
如果共享数据只是一个变量的话，使用原子操作std::atmoic效率更高
对std::atomic对象的访问不会造成竞争/冒险
比如std::atomic<int>，实际上是将int拓展成了原子类型，使得int类型的++，--都变成了原子操作，同时拓展了fetch_add、fetch_sub等原子加减方法

#### extern
**声明**用来**告诉编译器变量的名称和类型**，而不分配内存，不赋初值。
定义为了给变量分配内存，可以为变量赋初值。
注：定义要为变量分配内存空间；而声明不需要为变量分配内存空间。

#### 声明一个定义在别处的变量

#### 在C＋＋文件中调用C方式编译的函数
比如在C＋＋中调用C库函数，就需要在C＋＋程序中用 extern “C” 声明要引用的函数。这是给链接器用的，告诉链接器在链接的时候用C函数规范来链接。主要原因是C＋＋和C程序编译完成后在目标代码中命名规则不同。


#### optional
容器类，表示一个可能存在或不存在的值，提高代码的可读性和安全性
```cpp
#include <iostream>
#include <optional>

std::optional<int> divide(int a, int b) {
    if (b != 0) {
        return a / b;
    } else {
        return std::nullopt; // 表示没有值
    }
}

int main() {
    std::optional<int> result = divide(10, 2);
    if (result.has_value()) {
        std::cout << "Result: " << result.value() << std::endl;
    } else {
        std::cout << "Division by zero!" << std::endl;
    }
    
    return 0;
}
```


