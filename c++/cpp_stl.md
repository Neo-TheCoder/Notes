# 前言
STL（Standard Template Library）实现百花齐放，本书采用的是SGI（Silicon Graphics Computer Systems, Inc.）版本，它被纳入GNU C++标准库
***具体版本为：cygnus C++ 2.91.57 for Windows版本***

## 学习重点：
1. 容器
vector、list、deque、set、map的底层数据结构和实现原理，如何管理内存、数据存储方式、迭代器设计

2. 迭代器
STL的核心之一，提供了统一的方式来遍历容器中的元素，了解迭代器的种类以及如何使用

3. 算法
STL提供了大量算法：排序、查找、复制、删除等，学习其实现原理以及如何与迭代器、容器交互

4. 仿函数
即函数对象，实现了operator()，有助于了解很多算法的灵活性和通用性

5. 分配器
管理内存分配和释放

6. 空间配置器
底层的内存分配和释放

7. 内存管理和性能优化
学习SGI STL在内存管理和性能优化方面的思路


# 第一章 STL概论与版本简介

## 1.1 STL概论
追求数据结构和算法的复用性
STL提供了一种高层次的、具有泛型思维的“软件组件分类学”，STL可视作一个抽象概念库（可赋值、默认构造、输出功能迭代器、单项迭代器，二元函数、关联式容器）


### 1.1.1 STL的历史
于C++创造的前后

### 1.1.2 STL与C++标准程序库
98年，STL进入了C++标准程序库的一大脉系（stream、string都用template重新实现）

## 1.2 STL六大组件 功能与运用
1. 容器
vector、list、deque、set、map，本质上都是类模板

2. 算法
sort、search、copy、erase，本质上是函数模板

3. 迭代器
容器和算法之间的胶合剂，又称“泛型指针”，重载了operator*、operator->、operator++等
所有STL容器都附带有自己专属的迭代器

4. 仿函数
行为类似于函数，可以用作算法的某种策略
本质上是一种重载了operator()的类或者类模板，一般函数指针可视为狭义的仿函数

5. 配接器
用于修饰容器、仿函数、迭代器接口
例如STL提供的queue和stack虽然看作容器，但其实只能算是容器配接器，因为底层是借助deque实现的

6. 配置器
负责空间配置与管理，本质上是实现了动态空间配置、空间管理、空间释放的类模板

C++编译器一定支持有一份STL，在各个C++头文件里
C++标准规定所有标准头文件不再有扩展名（但是有的STL版本同时存在有无扩展名的两种版本）

### STL六大组件的交互关系
Container通过Allocator取得数据储存空间
Algorithm通过Iterator存取Container内容
Functor协助Algorithm完成不同的策略变化
Adapter可以修饰或套接Functor

## 1.3 GNU源码开放精神
STL原始版本在惠普，允许拷贝、修改、传播，但必须把其声明放在开发者的文件内
**GNU**是一个开放改革计划（GNU Is Not Unix）
其中早期最著名的软件有Emacs（文本编辑器）和GCC
（C/C++编译器），后期著名的是Linux操作系统
GNU以GPL（Genral Public License，广泛开放授权）来保护其成员

Cygnus公司之于GCC，就像Red Hat之于Linux，而GCC最终控制权仍在GCC指导委员会

### 1.9.2 临时对象的产生与运用
STL常将其用于仿函数和算法的搭配上

### 1.9.3 静态常量整数成员在class内部直接初始化
只初始化一次

### 1.9.5 左闭右开区间表示法
任何一个STL算法对应的迭代器标示的区间/操作范围，都是一个左闭右开区间：[first, last)
实际范围是first ~ last - 1。



# 第二章 空间配置器(allocator)
之所以称之为空间配置器，而不是内存配置器，是因为空间不只是`内存`，也可能是`磁盘`或者`别的存储介质`（可以实现allocator直接向硬盘存取空间）
SGI STL提供的配置器则是内存配置器

## 2.1 空间配置器的标准接口
```cpp
// Allocator adaptor to turn an SGI-style allocator (e.g. alloc, malloc_alloc)
// into a standard-conforming allocator.   Note that this adaptor does
// *not* assume that all objects of the underlying alloc class are
// identical, nor does it assume that all of the underlying alloc's
// member functions are static member functions.  Note, also, that 
// __allocator<_Tp, alloc> is essentially the same thing as allocator<_Tp>.

template <class _Tp, class _Alloc>
struct __allocator {
  _Alloc __underlying_alloc;

  typedef size_t    size_type;
  typedef ptrdiff_t difference_type;
  typedef _Tp*       pointer;
  typedef const _Tp* const_pointer;
  typedef _Tp&       reference;
  typedef const _Tp& const_reference;
  typedef _Tp        value_type;

  template <class _Tp1> struct rebind {
    typedef __allocator<_Tp1, _Alloc> other;
  };

  __allocator() __STL_NOTHROW {}
  __allocator(const __allocator& __a) __STL_NOTHROW
    : __underlying_alloc(__a.__underlying_alloc) {}
  template <class _Tp1> 
  __allocator(const __allocator<_Tp1, _Alloc>& __a) __STL_NOTHROW
    : __underlying_alloc(__a.__underlying_alloc) {}
  ~__allocator() __STL_NOTHROW {}

  pointer address(reference __x) const { return &__x; }
  const_pointer address(const_reference __x) const { return &__x; }

  // __n is permitted to be 0.
  _Tp* allocate(size_type __n, const void* = 0) {   // 配置空间，存储n个_Tp对象，第二个入参
    return __n != 0 
        ? static_cast<_Tp*>(__underlying_alloc.allocate(__n * sizeof(_Tp))) 
        : 0;
  }

  // __p is not permitted to be a null pointer.
  void deallocate(pointer __p, size_type __n)
    { __underlying_alloc.deallocate(__p, __n * sizeof(_Tp)); }

  size_type max_size() const __STL_NOTHROW 
    { return size_t(-1) / sizeof(_Tp); }

  void construct(pointer __p, const _Tp& __val) { new(__p) _Tp(__val); }
  void destroy(pointer __p) { __p->~_Tp(); }
};

```

## 2.2 具备次配置力(sub-allocation)的SGI空间配置器
SGI STL的配置器和标准规范不同的，名称是`alloc`（不接收类型参数），而非`allocator`。
```cpp
vector<int, std::allocator<int>> iv;    // 标准写法 VC中的

vector<int, std::alloc> iv; // GCC写法
// 但我们一般用缺省的空间配置器，不太需要自行指定配置器的名字

// SGI STL的每个容器都已经指定缺省的空间配置器为alloc
// 例如：
template<class T, class Alloc = alloc>
class vector{...};
```

### 2.2.1 SGI标准的空间配置器 `std::allocator`
它虽然符合部分标准，但是效率不好，只是对`operator new`和`operator delete`的简单封装

### 2.2.2 SGI特殊的空间配置器 `std::alloc`
该空间配置器效率高，推荐使用
#### new操作实际上分为两步：
1. 配置内存
2. 构造对象

delete操作：
1. 析构对象
2. 释放内存

std::alloc实现中：
`allocate()`用于`内存申请操作`
`deallocate()`用于`内存释放`
`construct()`用于`对象构造`
`destroy()`用于`对象析构`

`#include<memory>`中含有以下文件：
```cpp
#include<stl_alloc.h> 
// 负责内存空间的配置与释放 定义了一、二级配置器，二者协作
allocate()、deallocate()

#include<stl_construct.h>
// 负责对象内容的构造与析构 隶属于STL标准规范
construct()、destroy()

#include<stl_uninitialized.h>
// 定义了一些全局函数，用于fill或者copy大块内存数据 隶属于STL标准规范
// 虽然不属于配置器的范畴，和对象初值有关，考虑了性能
un_initialized_copy()
un_initialized_fill()
un_initialized_fill_n()
// 它们的底层实现，在最差情况下会调用construct()、最佳情况下调用C标准函数memmove()直接进行内存数据的移动
```

### 2.2.3 构造和析构基本工具 `construct()`和`destroy()`
**在已配置内存的基础上**，用于对象的构造和析构
```cpp
#include<stl_construct.h>

template<class T1>
inline void construct(T1* p){
    new ((void*)p) T1();  // 其实就是placement new，调用了T1的构造函数（无参数）
}

template<class T1, class T2>
inline void construct(T1* p, const T2& value){
    new (p) T1(value);  // 其实就是placement new，调用了T1的构造函数（有参数）
}

template<class T>
inline void destroy(T* pointer){
    pointer->~T();  // 调用析构
}

// 第二版本：析构 指定范围 的对象
// 接收两个迭代器，该函数会找出元素的数值型别（value type），以便得到最佳措施
template <class _ForwardIterator>
inline void _Destroy(_ForwardIterator __first, _ForwardIterator __last) {
  __destroy(__first, __last, __VALUE_TYPE(__first));    // __destroy第三个参数接收一个指针，__VALUE_TYPE获取值类型
}

// 其有不同的特化版本，是根据每个对象析构函数来判断的，先利用value_type()获得迭代器指定对象的型别，再利用_type_traits<T>判断其析构函数是否是trivial的（即析构函数没有特殊操作，类对象内部不包含需要特殊处理的资源）
// 如果是，则什么都不做，如果不是，则循环范围内的所有对象，调用第一个版本的destroy()

// 判断元素的数值型别(value type)是否有trivial destructor
template<class ForwardIterator, class T>
inline void __destroy(ForwardIterator first, ForwardIterator last, T*) {
       typedef typename __type_traits<T>::has_trivial_destructor  trivial_destructor;
       __destroy_aux(first, last, trivial_destructor());    // 通过traits技术得到value type，临时调用构造，返回true / false，以在编译期判断
}

// 如果元素的数值型别(value type)有non-trivial destructor, 则派送(dispatch)到这里
template<class ForwardIterator>
inline void __destroy_aux(ForwardIterator first, ForwardIterator last,  __false_type) {
       for (; first < last; ++first)
              destroy(&*first);
}

// 如果元素的数值型别(value type)有trivial destructor, 则派送(dispatch)到这里
template<class ForwardIterator>
inline void __destroy_aux(ForwardIterator first, ForwardIterator last,  __true_type) {
}
// 比如是char* [start, end)，就不需要调用析构，因为没构造对象，那块内存直接让别人拿去用就行了

// destroy() 第二版针对迭代器为char*和wchar_t*的特化版
// 原生指针区间不需要析构, 因为没有对象, 类似地，还有int*, long*, float*, double*，这里省略
inline void destroy(char*, char*) { }
inline void destroy(wchar_t*, wchar_t*) { }
// ...
```

### 2.2.4 空间的配置与释放 std::alloc
`<std_alloc.h>`负责：`对象构造前的空间配置` 和 `对象析构后的空间释放`
SGI对此的设计哲学：
1. 向`system heap`要求空间
2. 考虑`多线程状态`
3. 考虑`内存不足`时的应变措施
4. 考虑`内存碎片`问题

对于内存碎片，SGI设计了`双层级配置器`
1. 第一级配置器
    直接使用`malloc()`，`free()`

2. 第二级配置器
    当配置区块超过`128bytes`，认为`足够大`，使用第一级配置器
    当小于`128bytes`，认为`过于小`，为了降低负担，使用复杂的`memory pool`整理方式 --> 减少内存碎片，提高利用率
相比第一级就是多了个对要分配内存大小的判断
(第二级配置器的开关取决于`__USE_MALLOC`)

```cpp
// 取得SGI规定的配置器类型
#ifdef __USE_MALLOC
// ...
typedef __malloc_alloc_template<0> malloc_alloc;
typedef malloc_alloc alloc; // 使得alloc表示第一级配置器

# else
// ...
typedef __default_alloc_template<__NODE_ALLOCATOR_THREADS, 0> alloc; // 第二级配置器
# endif
```
alloc不接受任何`template型别参数`,所以 如此包装才能使得alloc符合STL规格
```cpp
template<class T, class Alloc>
class simple_alloc
//  简单的 传递调用 ，使得配置器的配置单位由"bytes"转为元素类型的大小
{    
    static T *allocate(size t n)
    {
        return 0 == n?0 : (T*) Alloc::allocate(n * sizeof(T));
    }
    static T *allocate(void)
    {
        return (T*) Alloc::allocate(sizeof(T));
    }
    static void deallocate(T *p， size_t n)
    {
        if (0 != n)
            Alloc::deallocate(p，n * sizeof(T));
    }
    static void deallocate(T *p)
    {
        Alloc::deallocate(p，sizeof(T)); 
    }
};
```

```cpp
template<class T, class Alloc = alloc> // 默认使用alloc
class Vector{
protected:
    // 专属的空间配置器
    typedef simple_alloc<T, Alloc> data_allocator;
    data_allocator::allocate(n);    // 配置n个元素，然后设定初值

    void deallocate(){
        if(...)
            data_allocator::deallocate(start, end_of_storage - start);
    }
}
```

```cpp
template<class T, class Alloc = alloc, size_t BufSiz = 0>
class deque{
protected:
    typedef simple_alloc<T, Alloc> data_allocator;
    typedef simple_alloc<T*, Alloc> map_allocator;
    data_allocator::allocate(n);    // 配置n个元素
    map_allocator::allocate(n);    // 配置n个元素
    // 配置完成后，设置初值
}
```


# 一、二级配置器对比
## SGI STL第一级配置器
```cpp
template<int inst>
class __malloc_alloc_template{...};
```
1. `allocate()`直接使用`malloc()`
   `deallocate()`直接使用`free()`
2. 模拟C++的`set_new_handler()`以处理内存不足的状况（在内存分配失败时调用）

## SGI STL第二级配置器
```cpp
template<bool threads, int inst>
class __default_alloc_template{...};
```

1. 维护`16个自由链表`(free lists)，负责16个小型区块的次配置能力
内存池以`malloc()`配置而得
如果内存不足，则转调用第一级配置器（有对内存不足的处理程序）
（小内存容易产生碎片）

2. 如果需求区块大于`128bytes`，则转而调用第一级配置器

### 2.2.5 第一级配置器`__malloc_alloc_template`剖析
```cpp
#ifndef __THROW_BAD_ALLOC
#  if defined(__STL_NO_BAD_ALLOC) || !defined(__STL_USE_EXCEPTIONS)
#    include <stdio.h>
#    include <stdlib.h>
#    define __THROW_BAD_ALLOC fprintf(stderr, "out of memory\n"); exit(1)
#  else /* Standard conforming out-of-memory handling */
#    include <new>
#    define __THROW_BAD_ALLOC throw std::bad_alloc()
#  endif
#endif
// 编译器会根据自身情况，选择alloc失败时相应的处理异常（有的编译器没法抛出异常）

// Malloc-based allocator.  Typically slower than default alloc below.
// Typically thread-safe and more storage efficient.
#ifdef __STL_STATIC_TEMPLATE_MEMBER_BUG
# ifdef __DECLARE_GLOBALS_HERE
    void (* __malloc_alloc_oom_handler)() = 0;  // 即nullptr
    // g++ 2.7.2 does not handle static template data members.
# else
    extern void (* __malloc_alloc_oom_handler)();
# endif
#endif
// 不同编译器对模板中的静态成员函数处理不同

template <int __inst>       // 注意：这是一种非类型模板，模板参数不是常规的类，而是个int类型
class __malloc_alloc_template {

private:

  static void* _S_oom_malloc(size_t);   // 内存不足时的处理函数
  static void* _S_oom_realloc(void*, size_t);

#ifndef __STL_STATIC_TEMPLATE_MEMBER_BUG
  static void (* __malloc_alloc_oom_handler)();
#endif

public:

  static void* allocate(size_t __n)
  {
    void* __result = malloc(__n);   // 直接调用malloc
    if (0 == __result) __result = _S_oom_malloc(__n);   // 分配失败，则直接调用malloc对应的处理函数
    return __result;
  }

  static void deallocate(void* __p, size_t /* __n */)
  {
    free(__p);  // 直接使用free
  }

  static void* reallocate(void* __p, size_t /* old_sz */, size_t __new_sz)
  {
    void* __result = realloc(__p, __new_sz);    // 直接使用realloc
    if (0 == __result) __result = _S_oom_realloc(__p, __new_sz);
    return __result;
  }

// 入参是一个函数指针，出参还是一个函数指针
  static void (* __set_malloc_handler(void  (*__f)() )) () // 模拟new失败时的回调函数，由于不是使用operator new来配置内存的，所以算是模拟
  {
    void (* __old)() = __malloc_alloc_oom_handler;  // 创建函数指针，保存当前的 new失败时 回调
    __malloc_alloc_oom_handler = __f;
    return(__old);  // 把旧的函数作为返回值返回出去，用户可能使用
  }

};  // class __malloc_alloc_template



// malloc_alloc out-of-memory handling
#ifndef __STL_STATIC_TEMPLATE_MEMBER_BUG
template <int __inst>
void (* __malloc_alloc_template<__inst>::__malloc_alloc_oom_handler)() = 0;
#endif

template <int __inst>
void*
__malloc_alloc_template<__inst>::_S_oom_malloc(size_t __n)  // malloc失败时的处理函数
{
    void (* __my_malloc_handler)();
    void* __result;

    for (;;) {  // 反复尝试：“释放、配置” 的过程，死循环
        __my_malloc_handler = __malloc_alloc_oom_handler;
        if (0 == __my_malloc_handler) { __THROW_BAD_ALLOC; }
        (*__my_malloc_handler)();
        __result = malloc(__n);
        if (__result) return(__result);
    }
}

template <int __inst>
void* __malloc_alloc_template<__inst>::_S_oom_realloc(void* __p, size_t __n)
{
    void (* __my_malloc_handler)();
    void* __result;

    for (;;) {
        __my_malloc_handler = __malloc_alloc_oom_handler;
        if (0 == __my_malloc_handler) { __THROW_BAD_ALLOC; }
        (*__my_malloc_handler)();
        __result = realloc(__p, __n);
        if (__result) return(__result);
    }
}


typedef __malloc_alloc_template<0> malloc_alloc;    // 直接将inst指定为0
```
所谓 C++ new handler机制是，你可以要求系统在内存配置需求无法被满足时，调用一个你所指定的函数。
换句话说，一旦`operator new`无法完成任务在丢出`std::bad_alloc`异常状态之前，会先调用由客端指定的处理例程。该处理函数即`new-handler`。


### 2.2.6 第二级配置器
针对内存碎片问题、配置问题，对于每一块内存，还需要相应的内存管理它，那么碎片越多，管理内存越多
1. 当区块小于128字节
    使用内存池管理（以链表形式维护多个小区块：`配置`和`回收`）

    区块大小是不等的：8, 16, 24, 32, 40, 48 .. 128字节
```cpp
union obj{
    union obj* free_list_link;
    char client_data[1];
};
```
由于`union`的特性，`free_list_link`和`client_data`共享同一块内存空间（即obj实际占据的内存，其大小由成员中最大的数据类型决定，其内容由最后赋值的成员决定），这样就节省了维护链表所必须造成的内存浪费。

2. 大于128字节
    转交第一级配置器

`__default_alloc_template`
```cpp
// Default node allocator.
// With a reasonable compiler, this should be roughly as fast as the
// original STL class-specific allocators, but with less fragmentation.
// Default_alloc_template parameters are experimental and MAY
// DISAPPEAR in the future.  Clients should just use alloc for now.
//
// Important implementation properties:
// 1. If the client request an object of size > _MAX_BYTES, the resulting
//    object will be obtained directly from malloc.
// 2. In all other cases, we allocate an object of size exactly
//    _S_round_up(requested_size).  Thus the client has enough size
//    information that we can return the object to the proper free list
//    without permanently losing part of the object.
//

// The first template parameter specifies whether more than one thread
// may use this allocator.  It is safe to allocate an object from
// one instance of a default_alloc and deallocate it with another
// one.  This effectively transfers its ownership to the second one.
// This may have undesirable effects on reference locality.
// The second parameter is unreferenced and serves only to allow the
// creation of multiple default_alloc instances.
// Node that containers built on different allocator instances have
// different types, limiting the utility of this approach.

#if defined(__SUNPRO_CC) || defined(__GNUC__)
// breaks if we make these template class members:
  enum {_ALIGN = 8};    // 每个内存块的上调边界？？？
  enum {_MAX_BYTES = 128};  // 内存块大小的上限
  enum {_NFREELISTS = 16}; // _MAX_BYTES/_ALIGN free-lists的个数
#endif
// 没有常见的类型参数，第二个类型参数没用上
template <bool threads, int inst>
class __default_alloc_template {

private:
  // Really we should use static const int x = N
  // instead of enum { x = N }, but few compilers accept the former.
#if ! (defined(__SUNPRO_CC) || defined(__GNUC__))
    enum {_ALIGN = 8};
    enum {_MAX_BYTES = 128};
    enum {_NFREELISTS = 16}; // _MAX_BYTES/_ALIGN
# endif
  static size_t
  _S_round_up(size_t __bytes)   // 将输入字节数上调至8的倍数
    { return (((__bytes) + (size_t) _ALIGN-1) & ~((size_t) _ALIGN - 1)); }

__PRIVATE:
  union _Obj {
        union _Obj* _M_free_list_link;
        char _M_client_data[1];    /* The client sees this.        */
  };
private:
# if defined(__SUNPRO_CC) || defined(__GNUC__) || defined(__HP_aCC)
    static _Obj* __STL_VOLATILE _S_free_list[]; 
        // Specifying a size results in duplicate def for 4.1
# else
    static _Obj* __STL_VOLATILE _S_free_list[_NFREELISTS];  // ！！！声明了一个static指针数组，存储的是16个指针，指针的类型是_Obj类型，__STL_VOLATILE即volatile，表示避免编译器对该变量的访问（编译器会将变量的值存储在寄存器中，而不是每次都从内存中读取）做优化，防止多线程访问时产生问题
# endif
  static  size_t _S_freelist_index(size_t __bytes) {
        return (((__bytes) + (size_t)_ALIGN-1)/(size_t)_ALIGN - 1);
  } // 根据区块大小，决定使用第几号free-list

  // Returns an object of size __n, and optionally adds to size __n free list.
  static void* _S_refill(size_t __n);   // 分配一块指定大小：n字节的内存空间，并在需要时将其添加到空闲列表中
  // Allocates a chunk for nobjs of size size.  nobjs may be reduced
  // if it is inconvenient to allocate the requested number.
  static char* _S_chunk_alloc(size_t __size, int& __nobjs); // 配置一大块空间，容纳nobjs个大小为size的区块

  // Chunk allocation state.
  static char* _S_start_free;   // 内存池的起始位置，只在_S_chunk_alloc时变化
  static char* _S_end_free; // 内存池的结束位置，只在_S_chunk_alloc时变化
  static size_t _S_heap_size;

# ifdef __STL_THREADS
    static _STL_mutex_lock _S_node_allocator_lock;
# endif

    // It would be nice to use _STL_auto_lock here.  But we
    // don't need the NULL check.  And we do need a test whether
    // threads have actually been started.
    class _Lock;
    friend class _Lock;
    class _Lock {
        public:
            _Lock() { __NODE_ALLOCATOR_LOCK; }
            ~_Lock() { __NODE_ALLOCATOR_UNLOCK; }
    };

public:

  /* __n must be > 0      */
  static void* allocate(size_t __n)
  {
    void* __ret = 0;

    if (__n > (size_t) _MAX_BYTES) {
      __ret = malloc_alloc::allocate(__n);
    }
    else {
      _Obj* __STL_VOLATILE* __my_free_list
          = _S_free_list + _S_freelist_index(__n);
      // Acquire the lock here with a constructor call.
      // This ensures that it is released in exit or during stack
      // unwinding.
#     ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;
#     endif
      _Obj* __RESTRICT __result = *__my_free_list;
      if (__result == 0)
        __ret = _S_refill(_S_round_up(__n));
      else {
        *__my_free_list = __result -> _M_free_list_link;
        __ret = __result;
      }
    }

    return __ret;
  };

  /* __p may not be 0 */
  static void deallocate(void* __p, size_t __n)
  {
    if (__n > (size_t) _MAX_BYTES)
      malloc_alloc::deallocate(__p, __n);
    else {
      _Obj* __STL_VOLATILE*  __my_free_list
          = _S_free_list + _S_freelist_index(__n);
      _Obj* __q = (_Obj*)__p;

      // acquire lock
#       ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;
#       endif /* _NOTHREADS */
      __q -> _M_free_list_link = *__my_free_list;
      *__my_free_list = __q;
      // lock is released here
    }
  }

  static void* reallocate(void* __p, size_t __old_sz, size_t __new_sz);

} ;
```

`__default_alloc_template`的静态成员变量的定义、初值设定
```cpp
template <bool __threads, int __inst>
char* __default_alloc_template<__threads, __inst>::_S_start_free = 0;

template <bool __threads, int __inst>
char* __default_alloc_template<__threads, __inst>::_S_end_free = 0;

template <bool __threads, int __inst>
size_t __default_alloc_template<__threads, __inst>::_S_heap_size = 0;

template <bool __threads, int __inst>
typename __default_alloc_template<__threads, __inst>::_Obj* __STL_VOLATILE
__default_alloc_template<__threads, __inst> ::_S_free_list[
# if defined(__SUNPRO_CC) || defined(__GNUC__) || defined(__HP_aCC)
    _NFREELISTS
# else
    __default_alloc_template<__threads, __inst>::_NFREELISTS
# endif
] = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, }; // 16个元素的指针数组初始化为全0
// The 16 zeros are necessary to make version 4.1 of the SunPro
// compiler happy.  Otherwise it appears to allocate too little
// space for the array.

#endif /* ! __USE_MALLOC */
// 在SunPro编译器版本4.1中，如果不将数组的所有元素初始化为0，编译器可能会分配不足的空间给数组
```

### 2.2.7 空间配置函数`allocate()`
#### `allocate()`
```cpp
  /* __n must be > 0      */
  static void* allocate(size_t __n)
  {
    void* __ret = 0;

    if (__n > (size_t) _MAX_BYTES) {    // 若大于128则调用第一级配置器，即malloc
      __ret = malloc_alloc::allocate(__n);
    }
    else {  // 若小于128字节，则检查free list：声明了一个指针变量，通过_S_freelist_index()函数得到了偏移量（free list有十六个空闲内存块，内存块大小是按8递增的）
      _Obj* __STL_VOLATILE* __my_free_list
          = _S_free_list + _S_freelist_index(__n);  // 此处是二级指针，该指针指向一个地址，这个地址，指向一个地址，它存储着free list的某一个节点
      // Acquire the lock here with a constructor call.
      // This ensures that it is released in exit or during stack
      // unwinding.
#     ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;  // 需要加锁保护：对内存链表的修改操作
#     endif
      _Obj* __RESTRICT __result = *__my_free_list;  // 得到指针数组的某个元素，它是一个指针
      if (__result == 0)  // 没找到可用的free list，重新填充free list
        __ret = _S_refill(_S_round_up(__n));
      else {  // 调整free list
        *__my_free_list = __result -> _M_free_list_link;  // 空闲内存块链表少了一个元素。总之，这个空闲内存块链表只会保存空闲的内存块，一开始其实是空的，在return之前，访问union的_M_free_list_link操作是有效的
        __ret = __result;
      }
    }

    return __ret;
  };
```

`deallocate()`
```cpp
  /* __p may not be 0 */
  static void deallocate(void* __p, size_t __n) // 指向待释放的内存块的指针，以及释放的大小
  {
    if (__n > (size_t) _MAX_BYTES)  // 大于128字节
      malloc_alloc::deallocate(__p, __n);
    else {  // 小于128字节
      _Obj* __STL_VOLATILE*  __my_free_list
          = _S_free_list + _S_freelist_index(__n);  // 得到free list对应的指针数组的元素的位置
      _Obj* __q = (_Obj*)__p; // 得到指向当前待释放位置的、_Obj类型的指针

      // acquire lock
#       ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;
#       endif /* _NOTHREADS */
      __q -> _M_free_list_link = *__my_free_list; // 只是往一大片待回收的内存区域的前几个字节写入了一个地址，没做回收操作的话，依然是分配在堆上
      *__my_free_list = __q;  // 更新头部节点（即头插法）
      // lock is released here
    }
  }
```
`deallocate()`回收了内存区块。


### 2.2.9 重新填充free lists
当调用`allocate()`时发现free list没有可用区块时，调用`refill()`，为free list重新填充空间，新的空间取自内存池（通过`chunk_alloc()`完成），默认是取得20个新区快。
`refill()`
```cpp
/* Returns an object of size __n, and optionally adds to size __n free list.*/
/* We assume that __n is properly aligned.                                */
/* We hold the allocation lock.                                         */
template <bool __threads, int __inst>
void*
__default_alloc_template<__threads, __inst>::_S_refill(size_t __n)
{
    int __nobjs = 20;
    char* __chunk = _S_chunk_alloc(__n, __nobjs); // 尝试取得nobjs个区块，作为free list的新节点，_S_chunk_alloc的第二个入参是传引用，也就是会修改
    _Obj* __STL_VOLATILE* __my_free_list;
    _Obj* __result;
    _Obj* __current_obj;
    _Obj* __next_obj;
    int __i;

    if (1 == __nobjs) return(__chunk);  // 如果_S_chunk_alloc调用只得到1个区块
    __my_free_list = _S_free_list + _S_freelist_index(__n);

    /* Build free list in chunk */
      __result = (_Obj*)__chunk;
      *__my_free_list = __next_obj = (_Obj*)(__chunk + __n);  // 让掌管96字节的free list节点，得到相应的地址
      for (__i = 1; ; __i++) {
        __current_obj = __next_obj;
        __next_obj = (_Obj*)((char*)__next_obj + __n);  // 下一个96字节的位置
        if (__nobjs - 1 == __i) {
            __current_obj -> _M_free_list_link = 0;
            break;
        } else {
            __current_obj -> _M_free_list_link = __next_obj;
        }
      } // 让掌管96字节的free list节点，把20个96字节的内存块串联起来
    return(__result);
}
```

### 2.2.10 内存池
`chunk_alloc()`负责：从内存池中取空间给free list
```cpp
/* We allocate memory in large chunks in order to avoid fragmenting     */
/* the malloc heap too much.                                            */
/* We assume that size is properly aligned.                             */
/* We hold the allocation lock.                                         */
template <bool __threads, int __inst>
char*
__default_alloc_template<__threads, __inst>::_S_chunk_alloc(size_t __size, int& __nobjs)    // 注意：__nobjs是引用类型！！！
{
    char* __result;
    size_t __total_bytes = __size * __nobjs;  // 需要几个__nobjs字节的内存块
    size_t __bytes_left = _S_end_free - _S_start_free;  // 内存池剩余空间

    if (__bytes_left >= __total_bytes) {  // 内存池剩余空间满足需求
        __result = _S_start_free;
        _S_start_free += __total_bytes;
        return(__result);
    } else if (__bytes_left >= __size) {  // 内存池剩余空间无法完全满足需求
        __nobjs = (int)(__bytes_left/__size); // 尽力满足
        __total_bytes = __size * __nobjs;
        __result = _S_start_free;
        _S_start_free += __total_bytes;
        return(__result);   // __nobjs重新设定为当前足够的数量，而不是预定的20个
    } else {  // 内存池剩余空间连一个区块都不足
        size_t __bytes_to_get = 
	  2 * __total_bytes + _S_round_up(_S_heap_size >> 4); // 调用malloc向操作系统申请空间，要预申请更多的，避免频繁地申请
        // Try to make use of the left-over piece.
        if (__bytes_left > 0) { // 充分利用内存池剩余空间
            _Obj* __STL_VOLATILE* __my_free_list =
                        _S_free_list + _S_freelist_index(__bytes_left); // 将剩余空间分配给适当的free list，即便可能是更小区快，也就是利用碎片。注意：如果执行了下面的malloc，allocator无法再通过一对指针来管理线性空间，因为原来的指针指向的位置失效了

            ((_Obj*)_S_start_free) -> _M_free_list_link = *__my_free_list;  // 头插法
            *__my_free_list = (_Obj*)_S_start_free; // 头节点重置
        }
        _S_start_free = (char*)malloc(__bytes_to_get);  // 内存池几乎耗尽，重新调用malloc，补充内存池
        if (0 == _S_start_free) { // heap空间一点都不剩了
            size_t __i;
            _Obj* __STL_VOLATILE* __my_free_list;
	    _Obj* __p;
            // Try to make do with what we have.  That can't
            // hurt.  We do not try smaller requests, since that tends
            // to result in disaster on multi-process machines.
            for (__i = __size;                // 遍历更大的内存块，先把更大的内存块拿过来救急：先交给内存池管理，理论上刚好满足一个区块
                 __i <= (size_t) _MAX_BYTES;  // _MAX_BYTES是最大的内存块，当前为128字节
                 __i += (size_t) _ALIGN) {  // _ALIGN是8字节
                __my_free_list = _S_free_list + _S_freelist_index(__i); // 遍历每一个free list，利用那些空闲区块
                __p = *__my_free_list;
                if (0 != __p) {
                    *__my_free_list = __p -> _M_free_list_link;
                    _S_start_free = (char*)__p;   // _S_start_free - _S_end_free 是一块连续的空闲内存：执行 if-else 中的第二个分支
                    _S_end_free = _S_start_free + __i;
                    return(_S_chunk_alloc(__size, __nobjs));  // 修正实际能提供的__nobjs数量，返回值是地址，递归调用得到地址
                    // Any leftover piece will eventually make it to the
                    // right free list.
                }
            }
	    _S_end_free = 0;	// In case of exception.  真的一点都没了
            _S_start_free = (char*)malloc_alloc::allocate(__bytes_to_get);  // 调用第一级配置器，启动out-of-memory机制：先调用用户回调（假设这个回调能够回收一些内存），然后抛出std::bad_alloc异常
            // This should either throw an
            // exception or remedy the situation.  Thus we assume it
            // succeeded.
        }
        _S_heap_size += __bytes_to_get;   // malloc成功，更新边界信息
        _S_end_free = _S_start_free + __bytes_to_get;
        return(_S_chunk_alloc(__size, __nobjs));
    }
}
```
SGI的容器通常默认使用第二级配置器：
```cpp
template<class T, class Alloc = alloc>
class vector{
  // ...
}
```


## 2.3 内存基本处理工具
STL定义了`五个全局函数`，用于未初始化空间上，有利于容器的实现
`uninitialized_copy()`, `uninitialized_fill()`, `uninitialized_fill_n()`对应STL算法中的`copy()`, `fill()`, `fill_n()`

### 2.3.1 `uninitialized_copy`
```cpp
    template<class InputIterator, class ForwardIterator>
    ForwardIterator
    uninitialized_copy(InputIterator first, InputIterator last, ForwardIterator result);
```
使我们能够 将 **内存的配置** 与 **对象的构造行为** 分离
如果 输出目的地(即: `[ result, result + ( last - first ) )` ) 指向的是未初始化区域
它会调用`拷贝构造函数`为（ `[result, last)`范围内的 ）每一个对象产生一份copy对象，放入输出范围中（拷贝一片区域的元素到另一片区域）

容器的全区间构造函数通常以两个步骤完成：
1. 配置内存区块，其大小足以包含范围内的所有元素
2. 使用`uninitialized_copy()`，在该内存区块上`构造对象`

C++标准要求：`uninitialized_copy()`要么构造出所有必要元素，要么（当有任何一个copy constructor失败时）不构造任何东西


`uninitialized_copy`底层实现
```cpp
// Valid if copy construction is equivalent to assignment, and if the
//  destructor is trivial.
template <class _InputIter, class _ForwardIter>
inline _ForwardIter 
__uninitialized_copy_aux(_InputIter __first, _InputIter __last,
                         _ForwardIter __result,
                         __true_type)
{
  return copy(__first, __last, __result);   // 对于POD类型，这样处理最高效
}

template <class _InputIter, class _ForwardIter>
_ForwardIter 
__uninitialized_copy_aux(_InputIter __first, _InputIter __last,
                         _ForwardIter __result,
                         __false_type)
{
  _ForwardIter __cur = __result;
  __STL_TRY {
    for ( ; __first != __last; ++__first, ++__cur)
      _Construct(&*__cur, *__first);
    return __cur;
  }
  __STL_UNWIND(_Destroy(__result, __cur));  // 对于非POD类型，这样处理最稳妥
}
```

针对`const char*`和`const wchar_t*`的特化：（wchar_t即宽字节的char，常为2/4位）
`memmove`直接移动内存内容，它可以处理源内存区域和目标内存区域重叠的情况，因此在处理重叠内存区域时比`memcpy`更安全
```cpp
inline char* uninitialized_copy(const char* __first, const char* __last,
                                char* __result) {
  memmove(__result, __first, __last - __first); // 把first到last的内容，copy到__result位置
  return __result + (__last - __first);
}

inline wchar_t* 
uninitialized_copy(const wchar_t* __first, const wchar_t* __last,
                   wchar_t* __result)
{
  memmove(__result, __first, sizeof(wchar_t) * (__last - __first));
  return __result + (__last - __first);
}
```

### 2.3.2 `uninitialized_fill`
```cpp
    template<class ForwardIterator, class T>
    uninitialized_fill(ForwardIterator first, ForwardIterator last, const T& x);
    // 使用第三个入参，来初始化 first - last 的元素
```
一片区域的元素用一个元素来初始化

C++标准要求：`uninitialized_fill()`要么构造出所有必要元素，要么（当有任何一个copy constructor失败时）不构造任何东西（把已经构造好的析构掉）

`uninitialized_fill`底层实现：
```cpp
// Valid if copy construction is equivalent to assignment, and if the
// destructor is trivial.
template <class _ForwardIter, class _Tp>
inline void
__uninitialized_fill_aux(_ForwardIter __first, _ForwardIter __last, 
                         const _Tp& __x, __true_type)
{
  fill(__first, __last, __x);   // 调用STL的fill
}

template <class _ForwardIter, class _Tp>
void
__uninitialized_fill_aux(_ForwardIter __first, _ForwardIter __last, 
                         const _Tp& __x, __false_type)
{
  _ForwardIter __cur = __first;
  __STL_TRY {
    for ( ; __cur != __last; ++__cur)
      _Construct(&*__cur, __x);
  }
  __STL_UNWIND(_Destroy(__first, __cur));
}
```


### 2.3.3 `uninitialized_fill_n`
```cpp
    template<class ForwardIterator, class Size, class T>
    ForwardIterator
    uninitialized_fill_n(ForwardIterator first, Size n, const T& x);
    // 使用第三个入参，来初始化first开始的n个元素
```
一片区域的n个元素用一个元素来初始化

C++标准要求：`uninitialized_fill_n()`要么构造出所有必要元素，要么（当有任何一个copy constructor失败时）不构造任何东西（把已经构造好的析构掉）


`uninitialized_fill_n()`源码：
```cpp
template <class _ForwardIter, class _Size, class _Tp>
inline _ForwardIter 
uninitialized_fill_n(_ForwardIter __first, _Size __n, const _Tp& __x)
{
  return __uninitialized_fill_n(__first, __n, __x, __VALUE_TYPE(__first));  // 通过__VALUE_TYPE得到类型后传入临时变量
}   // 用__x来初始化，从__first开始的__n个元素
```

`__uninitialized_fill_n()`
```cpp
template <class _ForwardIter, class _Size, class _Tp, class _Tp1>
inline _ForwardIter 
__uninitialized_fill_n(_ForwardIter __first, _Size __n, const _Tp& __x, _Tp1*)  // 不需要使用入参传来的临时变量，只需要其类型
{
  typedef typename __type_traits<_Tp1>::is_POD_type _Is_POD;    // 使用traits技术，判断类型是否所需
  return __uninitialized_fill_n_aux(__first, __n, __x, _Is_POD());
}
```

`__uninitialized_fill_n_aux()`对于`POD类型`
`POD`类型：`标量类型`、`C stuct`，必然有：`trival ctor/ dtor / copy / assignment`函数
```cpp
// Valid if copy construction is equivalent to assignment, and if the
//  destructor is trivial.
template <class _ForwardIter, class _Size, class _Tp>
inline _ForwardIter
__uninitialized_fill_n_aux(_ForwardIter __first, _Size __n,
                           const _Tp& __x, __true_type)
{
  return fill_n(__first, __n, __x); // 6.4.2
}
```

`__uninitialized_fill_n_aux()`对于`非POD类型`
```cpp
template <class _ForwardIter, class _Size, class _Tp>
_ForwardIter
__uninitialized_fill_n_aux(_ForwardIter __first, _Size __n,
                           const _Tp& __x, __false_type)
{
  _ForwardIter __cur = __first;
  __STL_TRY {
    for ( ; __n > 0; --__n, ++__cur)
      _Construct(&*__cur, __x); // 一个个地调用拷贝构造
    return __cur;
  }
  __STL_UNWIND(_Destroy(__first, __cur));
}
```


### 总结
`uninitialized_copy(first, last, result)`：
是将指定范围内的元素拷贝到未初始化的内存区域，并在目标内存中构造对象

`uninitialized_fill(first, last, value)`：
是将相同的值填充到范围内的每个元素中

`uninitialized_fill_n(first, n, value)`：
是用于指定填充元素个数的函数，适用于某个范围的连续一段元素需要填充相同的值的情况


# 第三章 迭代器概念与traits编程技法
`迭代器模式`是一种设计模式：提供一种方法，使之能够**依序巡访**某个聚合物(容器)所含的各个元素，而又**无需暴露该聚合物的内部表述方式**。

## 3.1 迭代器设计思维————STL关键所在
STL的中心思想：**将数据容器与算法分开**，彼此独立设计，最后再以胶着剂撮合。
`find()`的实现，可以用在不同的容器上
```cpp
template <class _InputIter, class _Tp>
inline _InputIter find(_InputIter __first, _InputIter __last,
                       const _Tp& __val,
                       input_iterator_tag)
{
  while (__first != __last && !(*__first == __val))
    ++__first;
  return __first;
}
```

## 3.2 迭代器是一种智能指针
迭代器**行为类型指针**：提供两大行为：
1. 解引用（重载`operator*()`）
2. 成员访问（重载`operator->`）

假设现在有链表类，要设计其专属迭代器，那么必须对链表类的实现细节很了解，由于迭代器类必然和链表类的节点类有关联，那么节点类是必然暴露的，那么迭代器的开发工作应该是链表类的设计者一并开发，因此每一种STL容器都有专属迭代器。

## 3.3 迭代器相应型别（`associated types`）
**利用函数模板的参数推导机制**
```cpp
template<class T>
void func(I iter){
    func_impl(iter, *iter);
}

template<class I, class T>
void func_impl(I iter, T t){    // 一旦被调用，编译器会自动进行模板参数推导，得出型别T
    T tmp;  // 取得类型
    // ...
}
```

常用的迭代器相应型别有五种

## 3.4 Traits编程技法
迭代器所指对象的型别，又称`value_type`

`关联类型`是一种概念，`value type`是具体的`类型`（是一种`特性`），`value type`可以用来表述`关联类型`的具体类型。

```cpp
// 为迭代器MyIter内嵌类型value_type
template<class T>
struct MyIter // 陷阱：如果迭代器是原生指针，根本就没这样一个class type
{
  typedef T value_type; // 内嵌类型声明(nested type)！！！
  T* ptr;
  MyIter(T* p = 0) : ptr(p) { }
  T& operator*() const { return *ptr; }
  // ...
};

// 将“template的参数类型推导”机制，针对value type，专门写成一个function template
template<class I>
typename I::value_type func(I ite) // typename I::value_type是func的返回值类型
{ return *ite; }

// 客户端
// ...
MyIter<int> ite(new int(8)); // 定义迭代器ite, 指向int 8
cout << func(ite);           // 输出：8

// 如果传给func的参数（迭代器），是一个原生指针，这种方式就失效了
```
为什么func的返回值必须加上`typename`关键字？
因为T是模板参数，编译器生成代码之前，编译器不知道T是什么，那么编译器根本不知道`I::value_type`的真正语义：类型 / 成员函数 / 数据成员。

声明内嵌类型有一个陷阱：不是所有`迭代器`都是`class type`，比如`原生指针（native pointer）`就不是（其实原生指针也是迭代器）。
如果不是class type，就无法为它定义内嵌类型，但**STL又必须接受原生指针作为迭代器**，那要怎么办？
答案是可以针对这种特定情况，使用`template partial specialization`（模板偏特化）做特殊化处理。

### `template partial specialization`
即：对模板参数进行部分特化（将泛化版本的某些template参数明确指定）
```cpp
// 通用版 class template
template<typename T>
class C{...} // 这个泛化版本允许（接受）T为任何类型

template<typename T>
class C<T*>{...}  // 偏特化版class template，仅适用于“T为原生指针”的情况
                  // “T为原生指针”是“T为任何类型”的一个更进一步的条件限制

```
偏特化的`class C<T*>`仍然是一个模板，不过针对class C的模板参数T做了**进一步限制**，即T必须是指针类型。


#### `类型萃取`：萃取迭代器的几大特性
##### 萃取`value_type`
```cpp
// 萃取内嵌类型value type
template<class I>
struct iterator_traits { // traits意为“特性”，用于萃取出模板参数I的关联的原生类型value type
  typedef typename I::value_type value_type;
};
```
所谓traits，意义是，如果`模板参数I`定义有自己的`value type`，那么通过traits的作用，萃取出来的value_type就是`I::value_type`。

```cpp
template<class T>
tyename I::value_type func(I ite) // typename I::value_type是func返回值类型
{ return *ite; }




// 改写 ==>

// 萃取内嵌类型value type
template<class I>
struct iterator_traits { // traits意为“特性”，用于萃取出模板参数I的关联的原生类型value type
  typedef typename I::value_type value_type;    // 取到内嵌类型
};

// 原生指针不是class，没有内嵌类型，通过偏特化版本定义value type
template<class T>
struct iterator_traits<T*> // 针对原生指针设计的偏特化版iterator_traits
{
  typedef T value_type;
};

template<class T>
typename iterator_traits<I>::value_type // 这一整行是函数返回值类型
func(I ite)
{ return *ite; }
```
多了一层间接性（利用`iterator_traits<>`来做萃取）
这样，虽然原生指针int*不是一种class type，但也可以通过traits技法萃取出value type（iterator_traits偏特化版为其定义的，关联类型）。如此，就解决了先前的问题。


##### 对于指向常量的指针
注意，通过`iterator_traits<const int*>::value_type`获得的是`const int`，也就是const性质也一并取出了，如果只要int怎么办？
还是需要特化：
```cpp
// 通过针对pointer-to-const设计的偏特化版本，为萃取关联类型去掉const
template<class T>
struct iterator_traits<const T*>
{
  typedef T value_type; // 注意这里value_type不再是const类型, 通过该偏特化版已经去掉了const属性
};
```

##### 迭代器特性
traits扮演“特性萃取机”的角色，`针对迭代器的traits`称为`iterator_traits`，用于萃取各个迭代器特性。
而迭代器特性，是指`迭代器的关联类型`（`associated types`）。为了让traits能正常工作，每个迭代器必须遵守约定，以内嵌类型定义（nested typedef）的方式，定义出关联的类型。

当然，迭代器常用的关联类型不止value type，还有另外4种：difference type，pointer，reference，iterator category。如果希望开发的容器能与STL融合，就必须为你的容器的迭代器定义这5种类型。
如此，“特性萃取机”traits就能忠实地将各种特性萃取出来：

```cpp
template<class I>
struct iterator_traits
{
  typedef typename I::iterator_category iterator_category;
  typedef typename I::value_type        value_type;
  typedef typename I::difference_type   difference_type;
  typedef typename I::pointer           pointer;
  typedef typename I::reference         reference;
};
```
`iterator_traits`必须针对传入类型为`pointer`、`pointer-to-const`者，设计偏特化版。

### 3.4.1 迭代器相应型别之一：value_type

### 3.4.2 迭代器相应型别之二：difference_type
`difference_type`用来表示`两个迭代器之间的距离`（两个迭代器之间`相差的元素个数`），因此也可以用来表示`容器的最大容量`。
因为对于连续空间的容器而言，头尾之间的距离就是最大容量。
如果一个泛型算法提供计数功能，如STL `count()`，其返回值必须使用迭代器的difference type。

例如，`std::count()`算法对迭代器区间，对值为value的元素进行计数：
```cpp
template<class I, class T>
typename iteartor_traits<I>::difference_type // 这一整行是函数返回类型
count(I first, I last, const T& value)
{
  typename iteartor_traits<I>::difference_type n = 0; // 迭代器之间的距离
  for (; first != last; ++first)
    ++n;
  return n;
}
```

`原生指针的difference type`
同value type，iterator_traits无法为原生指针内嵌`difference type`，需要设计特化版本。具体地，以C++内建`ptrdiff_t`（<stddef.h>）作为原生指针的difference type。

```cpp
// 通用版，从类型I萃取出difference type
template<class I>
struct iterator_traits
{
  ...
  typedef typename I::difference_type difference_type;
};

// 针对原生指针而设计的偏特化版
template<class I>
struct iterator_traits<T*>
{
  ...
  typedef ptrdiff_t difference_type;
};

// 针对原生pointer-to-const而设计的偏特化版
template<class I>
struct iterator_traits<const T*>
{
  ...
  typedef ptrdiff_t difference_type;
};
```

使用`difference_type`：
```cpp
typename iterator_traits<I>::difference_type
```

### 3.4.3 迭代器关联类型 reference type
从“迭代器所指之物的内容是否允许改变”的角度看，迭代器分为两种：
1. 不允许改变“所指对象的内容”，称为`constant iterators`（常量迭代器），例如`const int* pic`；
2. 允许改变“所指对象的内容”，称为`mutable iterators`（可变迭代器），例如`int* pi`；

当对一个`mutable iterators`进行解引用（dereference）时，获得不应该是一个`右值（rvalue）`，而应当是一个`左值（lvalue）`。
因为右值不允许赋值操作（assignment），左值才允许。

而对一个`constant iterator`进行解引用操作时，获得的是一个右值。

C++中，函数如果要传回左值，都是以`by reference`方式进行，所以当p是个`mutable iterators`时，如果其`value type`是`T`，那么p的类型不应该是`T`，而应该是`T&`。
如果p是个`constant iterators`，其`value type`是`T`，那么p的类型不应该是const T，而应该是`const T&`。
这里讨论的*p的类型，也就是reference type。

### 3.4.4 迭代器关联类型 pointer type
pointer与reference在C++关系非常密切。
```cpp
Iterm& operator*() const { return *ptr; }  // 返回值类型是reference type，需要返回一个左值
Iterm* operator->() const { return ptr; } // 返回值类型是pointer type
```

```cpp
// 通用版traits
template<class I>
struct iterator_traits
{
  ...
  typedef typename I::pointer pointer;          // native pointer 无法内嵌pointer, 因此需要偏特化版
  typedef typename I::reference reference;      // native pointer 无法内嵌reference
};

// 针对原生指针的偏特化版traits
template<class I>
struct iterator_traits<T*>
{
  ...
  typedef T* pointer;
  typedef T& reference;
};

// 针对pointer-to-const的偏特化版traits，去掉了const常量性
template<class I>
struct iterator_traits<const T*>
{
  ...
  typedef T* pointer;
  typedef T& reference;
};
```

PS：C++语法决定了：
在C++中，`iter->value`等价于`iter.operator->()->value`，而不是`iter.operator->().value`，这是因为箭头运算符(`->`)的优先级高于成员访问运算符(`.`)。
当我们写`iter->value`时，实际上会被解释为`iter.operator->()->value`，首先调用重载的箭头运算符函数返回指针，然后再通过该指针访问成员变量`value`。
如果写成`iter.operator->().value`，这会被解释为先调用重载的箭头运算符函数返回一个临时对象，然后再通过该临时对象访问成员变量`value`，这并不符合箭头运算符的语义。


### 3.4.5 迭代器关联类型 iterator_category
根据移动特性和施行操作，迭代器分为5类：
1. `Input Iterator`：
    这种迭代器所指对象，不允许外界改变，**只读**（read only）。

2. `Output Iterator`：
    **只写**（write only）。

3. `Forward Iterator`：
    允许“写入型”算法（如`replace()`），在此种迭代器所形成的的区间上进行读写操作。

4. `Bindirectional Iterator`：
    可双向移动。某些算法需要逆向走访某个迭代器区间（录入逆向拷贝某范围内的元素）。

5. `Random Access Iterator`：
    前4种迭代器都只供应一部分指针算术能力（前3支持operator++，第4中加上operator--）。
    第5中则涵盖所有指针算术能力，包括`p+n, p-n，p[n]，p1-p2，p1 < p2`。

**设计算法时，如果可能，应尽量针对某种迭代器提供一个明确定义，并针对更强化的某种迭代器提供另一种定义，重复利用迭代器特性。这样，才能在不同情况下，提供最大效率。**

#### `advance()`为例
`advance()`是一个许多算法内部常用的函数，**功能是内部将p累进n次（前进距离n）**。
```cpp
// 针对InputIterator的advance()版本
// 要求 n > 0
template<class InputIterator, class Distance>
void advance_II(InputIterator& i, Distance n)
{
  // 单向，逐一前进
  while (n--) ++i; // or for(; n > 0; --n; ++i);
}

// 针对BidirectionalIterator的advance()版本
// n没有要求，可大于等于0，也可以小于0
template<class BidirectionalIterator, class Distance>
void advance_BI(BidirectionalIterator& i, Distance n)
{
  // 双向，逐一前进
  if (n >= 0)
    while (n--) ++i; // or for (; n > 0; --n, ++i);
  else
    while (n++) --i; // or for (; n < 0; ++n, --i);
}

// 针对RandomAccessIterator的advance()版本
// n没有要求
template<class RandomAccessIterator, class Distance>
void advance_RAI(RandomAccessIterator& i, Distance n)
{
  // 双向，跳跃前进
  i += n;
}
```
现在，当程序调用`advance()`时，应选择哪个版本的函数呢？
如果选择`advance_II()`：对`Random Access Iterator`效率极低，原本O(1)操作成了O(N)；
如果选择`advance_RAI()`，则无法接收`Input Iterator`（`Input Iterator`不支持跳跃前进）。
我们可以将三者合一，对外提供统一接口。其设计思想是根据迭代器i的类型，来选择最适当的`advance()`版本：
并且，不能在运行时判断，那样就慢了，要在编译期判断：
利用函数重载，以及类型筛选：


如果traits有能力萃取出迭代器的类型，便可以利用这个“迭代器类型”的关联类型，作为advance()的第三个参数，
**让编译器根据函数重载机制自动选择对应版本`advance()`**。
这个关联类型一定是一种`class type`，不能是数值、号码类的东西，因为编译器需要依赖它进行`函数重载决议（overloaded resolution）`，
而函数重载是根据`参数类型`来决定的。

需要额外定义5个`tag type`
```cpp
// 5个作为标记用的类型（tag types）
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};
```
这5个class**只用作标记**，不需要任何成员，因此**不会有任何内存代价**。

统一`__advance()`接口：
```cpp
// 如果迭代器类型是input_iterator_tag，就会dispatch至此
template<class InputIterator, class Distance>
inline void __advance(InputIterator& i, Distance n, input_iterator_tag)
{
    // 单向, 逐一前进
    while (n--) ++i;
}

// 如果迭代器类型是forward_iterator_tag，就会dispatch至此
/***
    这是一个单纯的传递调用函数(trivial forwarding function), 实际stl中不需要该函数
    如果编译器没有匹配到与forward_iterator_tag严格匹配的版本时，就会从继承关系来匹配input_iterator_tag版的__advance()
***/

template <class ForwardIterator, class Distance>
inline void __advance(ForwardIterator& i, Distance n, forward_iterator_tag)
{
    // 单纯地进行传递调用(forwarding)
    __advance(i, n, input_iterator_tag());
}

// 如果迭代器类型是bidirectional_iterator_tag，就会dispatch至此
template<class BidirectionalIterator, class Distance>
inline void __advance(BidirectionalIterator& i, Distance n, bidirectional_iterator_tag)
{
    // 双向, 逐一前进
    if (n >= 0)
        while (n--) ++i;
    else
        while (n++) --i;
}

template<class RandomAccessIterator, class Distance>
inline void __advance(RandomAccessIterator& i, Distance n, random_access_iterator_tag)
{
    // 双向, 跳跃前进
    i += n;
}
```
注意：每个`__advanec()`最后一个参数都**只声明类型，而未指定参数名称**，因为纯粹只是用来**激活重载机制**，函数中不使用该参数。
如果要加上参数名也可以，但不会用到。

还需要对外提供一个`上层控制接口`，调用上述各个重载的`__advance()`。
接口只需要两个参数，当它准备将工作转给上述的`__advance()`时，才自行加上第三参数：迭代器类型。
因此，这个上层接口函数必须有能力从它所获得的迭代器中推导出其类型，这个工作可以交给`traits机制`：
**上层调用**
注意，模板类型参数`InputIterator`只是个名字，不是特指
问题：为什么`advance()`的template参数名称，是最低阶的`InputIterator`？而不是最强化的那个？
`advance()`能接受各种类型的迭代器，但其型别参数命名为`InputIterator`。
这是STL算法的一个命名规则：以算法锁能接受的最低阶迭代器类型，来为其迭代器型别参数命名

```cpp
// 接口函数, 利用iterator_traits萃取迭代器的类型特性iterator_category
template<class InputIterator, class Distance>
inline void advance(InputIterator& i, Distance n)
{
    __advance(i, n, iterator_traits<InputIteartor>::iterator_category());   // 利用类型萃取技术 + 临时构造相应对象触发函数重载
}
```

`iterator_category`
```cpp
// iterator_category() 返回一个临时对象，类型是参数I的迭代器类型（iterator_category）
template<class I>
inline typename iterator_traits<I>::iterator_category // 一整行是含返回型别
iterator_category(const I&)
{
    typedef typename iterator_traits<I>::iterator_category category;
    return category(); // 返回临时对象
}
```

`iteartor_traits`类的 定义 / 特化
```cpp
// 通用版traits, 可用于萃取I的iterator_category
template<class I>
struct iteartor_traits
{
    ...
    typedef typename I::iterator_category iterator_category; // 为traits添加萃取特性iterator_category
};

template <class _Iterator>
struct iterator_traits {
  typedef typename _Iterator::iterator_category iterator_category;
  typedef typename _Iterator::value_type        value_type;
  typedef typename _Iterator::difference_type   difference_type;
  typedef typename _Iterator::pointer           pointer;
  typedef typename _Iterator::reference         reference;
};

// 针对原生指针的特化
template <class _Tp>
struct iterator_traits<_Tp*> {
  typedef random_access_iterator_tag iterator_category; // 特化
  typedef _Tp                         value_type;
  typedef ptrdiff_t                   difference_type;
  typedef _Tp*                        pointer;
  typedef _Tp&                        reference;
};

// 针对原生const指针的特化
template <class _Tp>
struct iterator_traits<const _Tp*> {
  typedef random_access_iterator_tag iterator_category; // 特化
  typedef _Tp                         value_type;
  typedef ptrdiff_t                   difference_type;
  typedef const _Tp*                  pointer;
  typedef const _Tp&                  reference;
};
```

问题：原生指针是一种`Random Access Iterator`. why?
任何迭代器，其类型永远应该落在**该迭代器所隶属之各种类型中，最强化的那个**。
例如，int*，既是Random Access Iterator，又是Bindirectional Iterator，同时也是Forward Iterator，而且是Input Iterator，那么其类型应该是最强化的`random_access_iterator_tag`。

##### 消除“单纯传递调用的函数”
`用class来定义迭代器的各种分类标签`的意义：
1. 不仅可以促成重载机制运作，使得编译器能正确执行重载决议，
2. 还可以通过继承，我们不必再写“单纯只做传递调用的函数”，如前面`__advance()`的`ForwardIterator`版（直接复用`input_iterator_tag`那版函数即可）。
为什么？
！！！因为编译器会优先匹配`实参与形参完全匹配`的函数版本，然后是从`继承关系`来匹配。这也是为什么5个迭代器类型中，存在继承关系。
即：重载函数如果入参是基类，而传进去的是派生类对象，依然可以匹配，毕竟派生类继承自基类。


#### 以`distance()`为例
`distance()`是一个常用迭代器操作函数，用来计算两个迭代器之间的距离。针对不同迭代器类型，有不同的计算方式，带来不同的效率。

`distance()`
```cpp
// 2个__distance()重载函数

// 如果迭代器类型是input_iterator_tag，就dispatch至此
template<class InputIterator>
inline typename iterator_traits<InputIterator>::difference_type
       __distance(InputIterator first, InputIterator last,
              input_iterator_tag) {
       typename iterator_traits<InputIterator>::difference_type n = 0;
       // 逐一累计距离
       while (first != last) {
              ++first; ++n;
       }
       return n;
}

// 如果迭代器类型是random_access_iterator_tag，就dispatch至此
template<class RandomAccessIterator>
inline typename iterator_traits<RandomAccessIterator>::difference_type
       __distance(RandomAccessIterator first, RandomAccessIterator last,
              random_access_iterator_tag)
{
       // 直接计算差距
       return last - first;
}

// 对用户接口, 可以自动推导出类型, 然后编译器根据推导出的iterator_category特性,
// 自动选择调用__distance()重载函数版本
/* 上层函数, 从所得的迭代器中推导出类型 */
template<class InputIterator>
inline typename iterator_traits<InputIterator>::difference_type
       distance(InputIterator first, InputIterator last)
{
       typedef typename iterator_traits<InputIterator>::iterator_category  category; // 萃取迭代器类型iterator_category
       return __distance(first, last, category()); // 根据迭代器类型，选择不同的__distance()重载版本
}
```

## 3.5 `std::iterator`的保证
为了符合规范，任何迭代器必须遵守一定的约定，提供5个内嵌关联类型，以便于traits萃取；否则，无法兼容STL。但如果每写一个迭代器，都要提供这5个内嵌关联类型，谁能保证每次都不漏掉或者出错？有没有一种更简便方式？
SGI STL在`<stl_iterator.h>`中提供一个公共iterator class，每个新设计的迭代器只需要继承它，就可以保证符合STL所需要规范：

```cpp
template <class _Category, class _Tp, class _Distance = ptrdiff_t,
          class _Pointer = _Tp*, class _Reference = _Tp&>
struct iterator {
  typedef _Category  iterator_category;
  typedef _Tp        value_type;
  typedef _Distance  difference_type;
  typedef _Pointer   pointer;
  typedef _Reference reference;
};
```
terator class不含任何成员，纯粹只是类型定义。因此继承自它并不会有任何额外负担（无运行负担、无内存负担）。
而且后三个模板参数由于有默认值，新迭代器可以无需提供实参。

例如，前面自定义`ListIter`，改用继承自iterator方式，可以这些编写
```cpp
template<class Item>
struct ListIter : public std::iterator<std::forward_iterator_tag, Item>
{
  // ...
};
```
这样的好处是很明显的，可以极大地简化自定义迭代器类ListIter的设计，使其专注于自己的事情，而且不容易出错

## 总结
1. 设计适当的关联类型（associated types），是迭代器的责任；
设计适当的迭代器，是容器的责任。
因为只有容器本身，才知道设计出怎样的迭代器来遍历自己，并执行迭代器的各种行为（前进，后退，取值，取用成员，...）。
至于算法，完全可以独立于容器和迭代器之外，只要设计以迭代器为对外接口即可。

2. traits编程技法大量应用于STL实现中，利用“内嵌类型”的编程技巧和编译器template参数推导功能，增强了C++未能提供的关于类型认证方面的能力。


## 3.7 __type_traits
`iterator_traits`是STL针对迭代器加以规范，用来萃取迭代器特性的机制。
SGI STL把这种技法扩大到迭代器以外的地方，于是有了`__type_traits`。（双下划线表示是内部所用，不在STL标准规范内）
`__type_traits`负责萃取型别（`type`）的特性。

SGI中，`__type_traits`位于`<type_traits.h>`，它提供了一种机制，用于萃取不同类型属性：在编译期就能完成`函数派生决定（function dispatch）`。

这对于编写template很有帮助，比如，当我们准备对一个 “元素类型未知” 的数组执行copy操作时，如果能事先知道元素类型是否有一个`trivial copy constructor`（平凡的拷贝构造函数），就能帮助我们决定是否使用更高效的内存直接处理操作，如：`memcpy()`或`memmove()`。

`__type_traits`的应用：
```cpp
__type_traits<T>::has_trivial_default_constructor
__type_traits<T>::has_trivial_copy_constructor
__type_traits<T>::has_trivial_assignment_operator
__type_traits<T>::has_trivial_destructor
__type_traits<T>::is_POD_type // POD: Plain Old Data
```
上面的式子，返回的是“真”或“假”，便于我们决定采用上面策略。
但结果不是bool值，而是有着真/假属性的“对象”。
因为我们希望编译器可以在编译期，就利用响应结果来进行参数推导，而编译器只有面对class object形式的参数，才会做参数类型推导。
因此，上面式子应该传回这样的东西：

仅用作标记的类，用于表示`true / false`
```cpp
// 用作真、假的标记类
struct __true_type {};
struct __false_type {};
```
这两个空白的class没有任何成员，仅作为标记类，不会带来任何负担。

```cpp
template<class type>
struct __type_traits
{
  typedef __true_type this_dummy_member_must_be_first;
  /*
    不要移除这个成员：
    它通知“有能力自动将__type_traits特化的编译器：
    我们现在看到的这个__type_traits template是特殊的。
    这是为了确保万一编译器也使用一个名为__type_traits而其实与此处定义无任何关联的template时，所有事情仍将顺利运作
  */

  /*
    以下条件应该遵守，因为编译器有可能自动为各类型专属的__type_traits特化版本（有些编译器是这样的）：
    - 你可以重新安排以下的成员次序
    - 你可以移除以下任何成员
    - 绝对不可以将以下成员重命名而没有改变编译器中对应名称
    - 新加入的成员会被视为一般成员，除非你在编译期中加上适当支持
  */
  typedef __false_type has_trivial_default_constructor;
  typedef __false_type has_trivail_copy_constructor;
  typedef __false_type has_trivail_assignment_constructor;
  typedef __false_type has_trivail_destructor;
  typedef __false_type is_POD_type; // POD: Plain Old Data
};
```
先将各个内嵌的型别设置为`__false_type`，然后再针对每一个`标量类型（scalar types）`设计适当的`__type_traits`特化版本。

`__type_traits`可以接受任何类型的参数，5个typedefs将经由以下渠道获得实值：

`一般具现（general instantiation）`，内含对所有类型都必定有效的保守值。
比如，上面各个`has_trivial_xxx`类型都被定义为`__false_type`，就是对所有类型都有效的保守值。
经过声明的特化版本，例如`<type_traits.h>`中对所有C++标量类型（scalar types）（指`bool/char/int/float/double等基本类型`）都提供了对应`特化`声明。
某些编译器自动为 所有类型 提供适当的特化版本。

### __type_traits针对C++标量类型的特化版
```cpp
/* 以下针对C++基本型别char, signed char, unsignedchar, short, unsigned short,
int, unsigned int, long, unsigned long, float, double, long double 提供特化版本.
注意, 每个成员的值都是__true_type, 表示这些型别都可采用最快速方式(如memcpy)来进行拷贝(copy)或赋值(assign) */

#   define __STL_TEMPLATE_NULL template<>

__STL_TEMPLATE_NULL struct __type_traits<bool> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

#endif /* __STL_NO_BOOL */

__STL_TEMPLATE_NULL struct __type_traits<char> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<signed char> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<unsigned char> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<short> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<unsigned short> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<int> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<unsigned int> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<long> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<unsigned long> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};
```

### __type_traits在STL中的应用
#### `uniitialized_fill_n()全局函数`
```cpp
// __type_traits应用: uninitialized_fill_n()
template<class ForwardIterator, class Size, class T>
inline ForwardIterator uninitialized_fill_n(ForwardIterator first, Size n, const  T& x) {
       return __uninitialized_fill_n(first, n, x, value_type(first));
}
```

`uninitialized_fill_n()`函数以x为初始化元素，自迭代器first开始`构造n个元素`。
为求取最大效率，首先用`value_type()`萃取出迭代器first的value type，再用`__type_traits`判断该型别是否为POD类型：

```cpp
// __type_traits萃取T的is_POD_type特性，判断是否为POD类型
// 利用__type_traits判断该型别是否为POD类型
template<class ForwardIterator, class Size, class T, class T1>
inline ForwardIterator __uninitialized_fiil_n(ForwardIterator first, Size n, const  T& x, T1*) {
       typedef typename __type_traits<T1>::is_POD_type is_POD;
       return __uninitialized_fiil_n_aux(first, n, x, is_POD);
}
```
根据is_POD的结果（通过__type_traits根据value_type是否为POD类型，返回标记类型的对象）
```cpp
// 如果不是POD类型, 就派送(dispatch)到这里
template<class ForwardIterator, class Size, class T>
ForwardIterator __uninitialized_fill_n_aux(ForwardIterator first, Size n, const T&  x, __false_type) {
  ...
}

// 如果是POD类型, 就会派送(dispatch)到这里.
template<class ForwardIterator, class Size, class T>
inline ForwardIterator ___uninitialized_fill_n_aux_(ForwardIterator first, Size n,  const T& x, __true_type) {
  ...
}

// 以下定义于<stl_algobase.h>中的fill_n()
template<class OutputIterator, class Size, class T>
OutputIterator fill_n(OutputIterator first, Size n, const T& value) {
  ...
}
```

#### 负责对象析构的`destroy()`全局函数
```cpp
// destroy()第一个版本, 接受一个指针
template<class T>
inline void destroy(T* pointer) {
  ...
}

// destroy()第二个版本, 接受两个迭代器. 次函数设法找出元素的数值类型,
// 进而利用__type_traits<>求取最适当措施
template<class ForwardIterator>
inline void destroy(ForwardIterator first, ForwardIterator last) {
  __destroy(first, last, value_type(first));
}

// __type_traits 萃取T的has_trivial_destructor特性, 判断是否为平凡析构函数
// 判断元素的数值类型(value type)是否有trivial destructor
template<class ForwardIterator, class T>
inline void __destroy(ForwardIterator first, ForwardIterator last, T*) {
  typedef typename __type_traits<T>::has_trivial_destructor  trivial_destructor;
  __destroy_aux(first, last, trivial_destructor());
}

// 如果元素的数值类型(value type)有non-trivial destructor, 则派送(dispatch)到这里
template<class ForwardIterator>
inline void __destroy_aux(ForwardIterator first, ForwardIterator last,  __false_type) {
  ...
}

// 如果元素的数值类型(value type)有trivial destructor, 则派送(dispatch)到这里
template<class ForwardIterator>
inline void __destroy_aux(ForwardIterator first, ForwardIterator last,  __true_type) {
}
```

#### `copy()`全局函数
```cpp
// 利用__type_traits萃取T的has_trivial_copy_constructor特性，判断是否为平凡copy构造函数
// 拷贝一个数组, 其元素为任意型别, 视情况采用最有效率的拷贝手段
template<class T> inline void copy(T* source, T* destination, int n) {
       copy(source, destination, n, typename  __type_traits<T>::has_trivial_copy_constructor());
}

// 拷贝一个数组, 其元素型别拥有non-trivial copy constructors, 则dispatch到这里
template<class T> void copy(T* source, T* destination, int n, __false_type) {
       // ...
}

// 拷贝一个数组, 其元素型别拥有trivial copy constructors, 则dispatch到这里
// 可借助memcpy()完成工作
template<class T> void copy(T* source, T* destination, int n, __true_type) {
       // ...
}
```

### 自定义`__type_traits`特化版
通常，不应该直接使用__type_traits，STL内部使用。正如SGI <type_traits.h>内部有这样的声明：
```cpp
/* NOTE: This is an internal header file, included by other STL headers.
 * You should not attempt to use it directly.
 */
```
编译器通常会自动为class生成这些特性，这些特性取决于自定义class是否包含trivial default constructor，trivial copy constructor，trivial assignment operator，或者trivial destructor。
但有些古老的编译器，可能并不能为class生成这些特性，导致即使是POD类型，但萃取出来的特性依然是__false_type。

此时，需要自行设计一个`__type_traits特化版本`，明确告诉编译器该class具有哪些特性：
```cpp
// 假设自定义class名为Shape
// 针对Shape设计的__type_traits特化版本
template<>  // template<>用于明确告诉编译器这是一个模板特化的声明
struct __type_traits<Shape> {
    typedef __true_type  has_trivial_default_constructor; // 告诉编译器Shape拥有trivial default constructor
    typedef __false_type has_trivial_copy_constructor;
    typedef __false_type has_trivial_assignment_constructor;
    typedef __false_type has_trivial_destructor;
    typedef __false_type is_POD_type;
};
```
### 如何判断一个class什么时候有自己的`non-trivial default constructor，non-trivial copy constructor，non-trivial assignment operator，non-trivial destructor`？
一个简单判断原则：如果`class`内含`指针成员`，并且进行了`动态内存配置`，那么该class就需要实现出自己的`non-trivial-xxx`。







# 第四章 序列式容器
STL容器将运用最广的一些数据结构进行了实现。
常用的数据结构不外乎：
array，list，tree，stack，queue，hash table，set，map
根据`数据在容器中的排列`特性，这些数据结构分为`序列式`、`关联式`


# 4.1 容器的概观与分类
### 4.1.1 序列式容器
其中的元素可序，未必有序
如`array`
序列式容器中的元素是按照它们`被插入容器的顺序`进行存储的，因此元素在容器中是有序的。
换句话说，元素的顺序是可以保证的，即元素的插入顺序和访问顺序是一致的。
然而，序列式容器中的元素并不一定是有序的，这是因为`容器本身并不会对元素进行排序`。


## 4.2 vector
### 4.2.1 vector概述
类似`array`，但是是`动态空间`，**内部机制会自行扩充元素**
vector实现的关键在于：**对大小的控制，以及重新配置时的数据移动效率**。
假如旧空间满载，再如何为新元素扩充空间，是一个重要问题，徐，需要提高“配置新空间 -- 数据移动 -- 释放旧空间”的操作效率



### 4.2.2 vector定义摘要
`vector`
```cpp
template <class T, class Alloc = alloc>
class Vector{
public:
    // 定义嵌套型别
    typedef T   value_type;
    typedef value_type* pointer;
    typedef value_type* iterator;
    typedef value_type& reference;
    typedef size_t size_type;
    typedef ptrdiff_t difference_type;  // 保存两个指针做减法的操作结果 同类指针作差，得到的是相隔的距离

protected:
    typedef simple_alloc<value_type, Alloc>data_allocator;  // 是SGI STL的空间配置器
    iterator start; // 表示目前使用空间的头
    iterator finish; // 表示目前使用空间的尾
    iterator end_of_storage; // 表示目前可用空间的尾

    void insert_aux(iterator position, const T& x); // 是在当前空间不够的情况下调用的
    void deallocate(){
        if(start)   // 回收分配的内存
        data_allocator::deallocate(start, end_of_storage - start);
    }

    void fill_initialize(size_type n, const T& value){
        start = allocate_and_fill(n, value);    // 分配内存并赋值
        finish = start + n;
        end_of_storage = finish;    // 此时空间并未扩展，可用空间等于使用空间
    }

public:
    iterator begin() {return start;}
    iterator end() {return finish;}
    size_type size() const { return size_type(end() - begin()); }
    size_type capacity() const {
        return size_type(end_of_storage - begin());
    }
    bool empty() const {return begin() == end();}
    reference operator[](size_type n){return *(begin() + n);}

    // 各种参数的构造函数
    vector() : start(0), finish(0), end_of_storage(0) {}    // 这样构造的话 没分配内存
    vector(size_type n, const T& value) {fill_initialize(n, value);}
    vector(int n, const T& value){fill_initialize(n, value)};
    vector(long n, const T& value){fill_initialize(n, value)};
    explicit vector(size_type n){fill_initialize(n, T());}

    // 析构函数
    ~vector(){
        destroy(start, finish); // 全局函数
        deallocate();   // vector的成员函数，回收分配的内存
    }
    reference front() {return *begin();}    // 返回第一个元素的引用
    reference back() {return *(end() -1);}
    void push_back(const T& x){
        if(finish != end_of_storage){
            construct(finish, x);   // 全局函数
            ++finish;   // 使用指针后移
        }
        else
            insert_aux(end(), x);
    }

    void pop_back(){
        --finish;
        destroy(finish);    // 销毁尾端元素
    }

    iterator erase(iterator position){
        if(position + 1 != end())   // end()指向的目前已使用空间的尾部，它是不存数据的
            copy(position + 1, finish, position);
            --finish;
            destroy(finish);
            return position;
    }

    void resize(size_type new_size, const T& x){
        if(new_size < size())
            erase(begin() + new_size, end());
        else
            insert(end(), new_size - size(), x);
    }
    void resize(size_type new_size){return resize(new_size, T());}
    void clear(){erase(begin(), end());}

protected:
    // 配置空间并填满内容
    iterator allocate_and_fill(size_type n, const T& x){
        iterator result = data_allocator::allocate(n);  // 分配内存
        uninitialized_fill_n(result, n, x); // 利用x，来初始化从result位置开始的n个元素
        return result;
    }
};

```
### 4.2.3 vector的迭代器
vector维护的是一个连续线性空间，不论元素类型是什么，普通指针都可以作为vector的迭代器而满足所有必要条件，因为vector需要的操作为：`operator*，operator->，operatro++， operator--， operator+， operator-， operator+=， operator-=`。
而普通指针天生就具备这些操作。vector支持随机存取（`operator[]`），而普通指针也有这样的能力。因此，vector提供的迭代器是`Random Access Iterators`。

```cpp
vector<int>::iterator ivite;    // ivite的型别就是int*
vector<Shape>::iterator svite;  // svite的型别就是Shape*
```

### 4.2.4 vector数据结构
vector是线性连续空间，以两个迭代器`start`和`finish`分别指向配置得来的`连续空间中目前已被使用的范围`，
迭代器`end_of_storage`指向`整块连续空间（含备用空间）的尾端`

**为了降低空间配置时的速度成本，vector`实际配置的大小`可能`比客户端需求量更大一些`，以备将来可能的扩充**
--> 即capacity的概念：一个vector的容量永远 `>=` 大小
**三个关键迭代器**：start、finish、end_of_storage

**vector在增加新元素时，如果超过当时的容量，则容量会扩充两倍，如果两倍仍然不足，则扩充至足够大的容量**
容量的扩张必须经历：`重新配置、元素移动、释放原空间`（必须要重新申请空间，移动元素，释放原空间）


### 4.2.5 vector的构造与内存管理
#### `_Vector_base`类
vector空间配置的细节都封装在`_Vector_base`内部
```cpp
template <class _Tp, class _Alloc> 
class _Vector_base {
public:
  typedef _Alloc allocator_type;
  allocator_type get_allocator() const { return allocator_type(); }

  _Vector_base(const _Alloc&)
    : _M_start(0), _M_finish(0), _M_end_of_storage(0) {}
  _Vector_base(size_t __n, const _Alloc&)
    : _M_start(0), _M_finish(0), _M_end_of_storage(0) 
  {
    _M_start = _M_allocate(__n);  // 配置n字节空间
    _M_finish = _M_start;
    _M_end_of_storage = _M_start + __n;
  }

  ~_Vector_base() { _M_deallocate(_M_start, _M_end_of_storage - _M_start); }

protected:
  _Tp* _M_start;
  _Tp* _M_finish;
  _Tp* _M_end_of_storage;

  typedef simple_alloc<_Tp, _Alloc> _M_data_allocator;
  _Tp* _M_allocate(size_t __n)
    { return _M_data_allocator::allocate(__n); }
  void _M_deallocate(_Tp* __p, size_t __n) 
    { _M_data_allocator::deallocate(__p, __n); }
};
```
注意：`simple_alloc`做了转换处理：把元素个数转换为以byte为单位的内存（因为对于第一级/第二级配置器而言，内存都是以byte为单位）
转换如下：
```cpp
template<class T, class Alloc>
class simple_alloc
//  简单的 传递调用 ，使得配置器的配置单位由"bytes"转为元素类型的大小
{    
    static T *allocate(size t n)
    {
        return 0 == n?0 : (T*) Alloc::allocate(n * sizeof(T));
    }
    static T *allocate(void)
    {
        return (T*) Alloc::allocate(sizeof(T));
    }
    static void deallocate(T *p， size_t n)
    {
        if (0 != n)
            Alloc::deallocate(p，n * sizeof(T));
    }
    static void deallocate(T *p)
    {
        Alloc::deallocate(p，sizeof(T)); 
    }
};
```

#### `vector`的构造函数
vector的默认分配子是alloc（二级空间配置器），使用父类`_Vector_base`来屏蔽（空间配置）细节，方便以`元素大小`为配置单位。
```cpp
// _Base是别名
template <class _Tp, class _Alloc = __STL_DEFAULT_ALLOCATOR(_Tp) >
class vector : protected _Vector_base<_Tp, _Alloc> 
{
private:
  typedef _Vector_base<_Tp, _Alloc> _Base;

public:
  explicit vector(const allocator_type& __a = allocator_type()) // 构造空vector, 对应客户端vector<int> ivec
    : _Base(__a) {}

  vector(size_type __n, const _Tp& __value,
         const allocator_type& __a = allocator_type())  // 构造大小为n个元素, 所有值为value的vector, 对应客户端vector<int> ivec3(20, 5)
    : _Base(__n, __a)
    { _M_finish = uninitialized_fill_n(_M_start, __n, __value); }

  explicit vector(size_type __n)  //  构造大小为n个元素的vector, 对应客户端vector<int> ivec2(10)
    : _Base(__n, allocator_type())
    { _M_finish = uninitialized_fill_n(_M_start, __n, _Tp()); }

  vector(const vector<_Tp, _Alloc>& __x)  // 拷贝构造的vector, 对应客户端vector<int> ivec4(ivec3)
    : _Base(__x.size(), __x.get_allocator())
    { _M_finish = uninitialized_copy(__x.begin(), __x.end(), _M_start); }

#ifdef __STL_MEMBER_TEMPLATES
  // Check whether it's an integral type.  If so, it's not an iterator.
  template <class _InputIterator>   // _InputIterator对整数做了特化！
  vector(_InputIterator __first, _InputIterator __last,
         const allocator_type& __a = allocator_type()) : _Base(__a) {   // 用迭代区间[first, last)构造vector
    typedef typename _Is_integer<_InputIterator>::_Integral _Integral;
    _M_initialize_aux(__first, __last, _Integral());  // 如果是整数，则不把输入参数作为迭代器处理
  }
}

// PS：_M_initialize_aux()的处理
  template <class _Integer>
  void _M_initialize_aux(_Integer __n, _Integer __value, __true_type) { // 输入参数为整数类型，例如：vector<int> ivec6(ivec3.begin(), ivec3.end());
    _M_start = _M_allocate(__n);
    _M_end_of_storage = _M_start + __n; 
    _M_finish = uninitialized_fill_n(_M_start, __n, __value);
  }

  template <class _InputIterator>
  void _M_initialize_aux(_InputIterator __first, _InputIterator __last,
                         __false_type) {
    _M_range_initialize(__first, __last, __ITERATOR_CATEGORY(__first));
  }

```


#### `vector`的析构函数
```cpp
// 释放迭代器区间[first, last), 如果迭代器指向基本类型, 什么也不做; 如果迭代器指向class 类型, 就先逐个析构
~vector() { destroy(_M_start, _M_finish); }

// 注意vector析构之后, 会接着析构基类. 因此即使vector什么也没做, 基类会回收线性内存空间
// 析构基类会先回收内存段(start, end_of_storage - start)
// 空间配置那一章提到过, 如果是二级配置器, 内存 > 128byte, 会直接返还给OS; 如果内存 <= 128byte, 会加入到某个合适的free list, 留作备用！！！
~_Vector_base() { _M_deallocate(_M_start, _M_end_of_storage - _M_start); }
```

#### 指定数组初始大小`reserve()`, `resize()`
`reserve()`，预留指定大小的线性空间
```cpp
 // 让vector容量 >= n 个元素
  void reserve(size_type __n) {
    if (capacity() < __n) { // 只有当前容量 < n时, 才需要重新配置空间
      const size_type __old_size = size(); // 已经装了元素个数
      iterator __tmp = _M_allocate_and_copy(__n, _M_start, _M_finish); // 1. 配置n个元素新空间, 并将[start, finish)元素拷贝到新空间
      destroy(_M_start, _M_finish); // 2. 调用全局destroy()销毁[start, finish)上的元素. 对于基本类型, 什么也不做; 对于class类型, 析构对象
      _M_deallocate(_M_start, _M_end_of_storage - _M_start); // 3. 调用基类的deallocate() 释放线性空间
        // 重新设置start, finish, end_of_storage 用于管理线性空间
      _M_start = __tmp;
      _M_finish = __tmp + __old_size;
      _M_end_of_storage = _M_start + __n;
    }
  }

    // 配置n个元素新空间, 并将[start, finish)元素拷贝到新空间
    // 遵循 "commit or rollback"规则
  iterator _M_allocate_and_copy(size_type __n, const_iterator __first,
                                               const_iterator __last)
  {
    iterator __result = _M_allocate(__n);
    __STL_TRY {
      uninitialized_copy(__first, __last, __result);  // 把first到last的元素拷贝到__result
      return __result;
    }
    __STL_UNWIND(_M_deallocate(__result, __n)); // commit or rollback精髓: 发生异常时, 释放线性空间
  }
```

`resize()`：`resize()`用于指定vector的新size()，`改变`线性空间的`finsih`指针，但`不改变``start`和`end_of_storage`指针。
```cpp
    // 如果size()超过new_size, 就擦除多余的; 如果不超过, 就在末尾插入指定元素x. 最终目标是让size()等于new_size
  void resize(size_type __new_size, const _Tp& __x) {
    if (__new_size < size()) // 新size < 现有size()时, 说明原来的size较大, 需要擦除一部分
      erase(begin() + __new_size, end()); // 擦除多余空间元素 [begin() + new_size, end())
    else
      insert(end(), __new_size - size(), __x);  // 用__x来补充若干元素
  }
  void resize(size_type __new_size) { resize(__new_size, _Tp()); }  // 其实是调用T的默认构造函数

    // 擦除指定位置position的元素, 后面的(position~末尾)元素整体向前移动
  iterator erase(iterator __position) {
    if (__position + 1 != end()) // 要删除的元素不是末尾元素
      copy(__position + 1, _M_finish, __position); // 将擦除位置后的区间[position+1, finish)元素, 拷贝到position起始处
    --_M_finish;  // 因为只擦除一个元素, finish向前移动1
    destroy(_M_finish); // finish指向的就是要删除的那个元素, 析构之, 但不释放空间(尚未归还给配置器)
    return __position; // 返回销毁元素的位置
  }

    // 擦除迭代区间[first, last), 后面的元素整体向前移动（因为要确保随机访问，需要连续存储）
  iterator erase(iterator __first, iterator __last) {
    iterator __i = copy(__last, _M_finish, __first); // 将擦除区间后的区间[last, finish)元素, 拷贝到first起始处
    destroy(__i, _M_finish); // 析构[i, finish)对象, 但并没有释放空间
    _M_finish = _M_finish - (__last - __first); // 先前移动finish指针
    return __first; // 返回销毁后的起始位置
  }
```





#### 关于`capacity`的变化
如果是一个个地`push_back`元素，那么capacity的变化就是1，2，4，8...
并且不会变小，即便是`clear()`了，也不会变小

擦除元素时，会导致`finish`指针移动，改变`size()`大小，但并不会导致`end_of_storage`指针变化，也就是不会导致容量变化。
如果想要`改变容量`以节省空间，需要调用`vector::shrink_to_fit()`，`减少容量`以适应size()大小。不过`shrink_to_fit()`是`C++11`加入的内容，老版的SGI STL并没有该功能。
简而言之，**老版SGI STL无法缩减vector容量**。

内部实现：
vector缺省使用`alloc`作为空间配置器，并据此另外定义了一个`data_allocator`，用于更方便地以元素大小为配置单位。
vector提供许多构造函数，可以指定空间大小及初值：
```cpp
vector(size_type n, const T& value){
  fill_initialize(n, value);  // 其中调用allocate_and_fill()，进行空间配置，调用uninitialized_fill_n()进行初始化对象
}
```

`push_back`插入元素会先检查是否有备用空间，没有则扩充（重新配置2倍的内存、移动数据、释放原空间）


注意：**对vector的任何操作，一旦涉及到空间的重新配置，指向原vector的所有迭代器就失效了**！！！

### 4.2.6 vector的元素操作：`pop_back`、`erase`、`clear`、`insert`
```cpp
void pop_back(){
    --finish;   // 尾端标记前移，表示放弃尾端元素
    destroy(finish);
}

// 清楚[first, last)的元素
iterator erase(iterator first, iterator last){
    iterator i = copy(last, finish, first);
    destroy(i, finish);
    finish = finish - (last - first);
    return first;
}

// 清除某个位置上的元素
iterator erase(iterator position){
    if(position +1 != end())
        copy(position + 1, finish, position);
        --finish;
        destroy(finish);
        return position;
}

void clear(){
    erase(begin(), end());
}

// insert
template <class t, class Alloc>
void vector<T, Alloc>::insert(iterator position, size_type n, const T& x){
if(n!= 0){
    if(size_type(end_of_storage - finish) >= n){
    // 备用空间大于等于“新增元素个数”
    T x_copy = x;   // 拷贝
    // 计算插入点后的现有元素个数
    const size_type elems_after = finish - position;
    iterator old_finish = finish;
    if(elems_after > n){
    // 当前position与finish之间的数量 > 新增元素个数（采取的是：因为这种情况n比较小，所以先让position后面腾出相应位置）
    uninitialized_copy(finish - n, finish, finish); // 拷贝finish-n到finish的元素到finish位置
    finish += n;    // 后移n位
    copy_backward(position, old_finish - n, old_finish);    // 拷贝position到old_finish-n的位置的元素到old_finish
    fill(position, position + n, x_copy);   // 从插入点开始填入新值
    }
    else{
    // 当前position与finish之间的数量 < 新增元素个数（这种情况特别在于：先把新旧finish之间赋值（利用备用空间））
    uninitialized_fill_n(finish, n - elems_after, x_copy);  // 对指定范围赋值
    finish += n - elems_after;
    uninitialized_copy(position, old_finish, finish);
    finish += elems_after;
    fill(position, old_finish, x_copy);
    }
    }
    else{
    // 备用空间不足，显然需要配置额外的内存
    // 新长度是旧长度的两倍
    // 复制和赋值
    


    }
#ifdef 异常
    catch

#endif





}




}





```















