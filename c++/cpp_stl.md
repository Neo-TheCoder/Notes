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

2. 大于128字节
    转交第一级配置器


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
  enum {_ALIGN = 8};
  enum {_MAX_BYTES = 128};
  enum {_NFREELISTS = 16}; // _MAX_BYTES/_ALIGN
#endif

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
  _S_round_up(size_t __bytes)   // 按照8的整数倍分配
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
    static _Obj* __STL_VOLATILE _S_free_list[_NFREELISTS]; 
# endif
  static  size_t _S_freelist_index(size_t __bytes) {
        return (((__bytes) + (size_t)_ALIGN-1)/(size_t)_ALIGN - 1);
  }

  // Returns an object of size __n, and optionally adds to size __n free list.
  static void* _S_refill(size_t __n);
  // Allocates a chunk for nobjs of size size.  nobjs may be reduced
  // if it is inconvenient to allocate the requested number.
  static char* _S_chunk_alloc(size_t __size, int& __nobjs);

  // Chunk allocation state.
  static char* _S_start_free;
  static char* _S_end_free;
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
`uninitialzied_copy()`, `uninitialized_fill()`, `uninitialized_fill_n()`对应STL算法中的`copy()`, `fill()`, `fill_n()`

### 2.3.1 `uninitialzied_copy`
```cpp
    template<class InputIterator, class ForwardIterator>
    ForwardIterator
    uninitialized_copy(InputIterator first, InputIterator last, ForwardIterator result);
```
使我们能够 将 **内存的配置** 与 **对象的构造行为** 分离
如果 输出目的地(即: `[ result, result + ( last - first ) )` ) 指向的是未初始化区域
它会调用`拷贝构造函数`为（ `[result, last)`范围内的 ）每一个对象产生一份copy对象，放入输出范围中

容器的全区间构造函数通常以两个步骤完成：
1. 配置内存区块，足以包含范围内的所有元素
2. 使用`uninitialized_copy()`，在该内存区快上构造对象

C++标准要求：`uninitialized_copy()`要么构造出所有必要元素，要么（当有任何一个copy constructor失败时）不构造任何东西***

### 2.3.2 uninitialized_fill
```cpp
    template<class ForwardIterator, class T>
    uninitialized_fill(ForwardIterator first, ForwardIterator last, const T& x);
    // 第三个入参是要填充的值
```

### 2.3.3 uninitialized_fill_n
```cpp
    template<class ForwardIterator, class Size, class T>
    ForwardIterator
    uninitialized_fill_n(ForwardIterator first, Size n, const T& x);
    // 第三个入参是要填充的值
```
uninitialized_copy(first, last, result)：
是将指定范围内的元素拷贝到未初始化的内存区域，并在目标内存中构造对象

uninitialized_fill(first, last, value)：
是将相同的值填充到范围内的每个元素中

uninitialized_fill_n(first, n, value)：
是用于指定填充元素个数的函数，适用于某个范围的连续一段元素需要填充相同的值的情况










# 第四章 序列式容器

## 4.2 vector
### 4.2.1 vector概述
类似array，但是是动态空间，**内部机制会自行扩充元素**
vector实现的关键在于**对大小的控制，以及重新配置时的数据移动效率**，假如旧空间满载，再如何为新元素扩充空间，是一个重要问题。

### 4.2.2 vector定义摘要
```cpp
template <class T, class Alloc = alloc>
class Vector{
public:
    // 定义一堆类型别名
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
        destroy(start, finish); // 全局函数，是allocator的函数？？？
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
        if(position + 1 != end())   // end()所指向的目前已使用空间的尾部是不存数据的
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
        uninitialized_fill_n(result, n, x); // ？？？函数体在哪里 利用分配的内存，初始化对象
        return result;
    }
};

```

### 4.2.3 vector的迭代器
**维护连续线性空间**，所以普通指针可以作为vector的迭代器而满足所有必要条件：比如operator*, operator->, operator++, operator--等操作，普通指针天生就具备
由于vector有类似数组的特性，支持**Random Access Iterators**

```cpp
vector<int>::iterator ivite;    // int*
vector<Shape>::iterator svite;  // Shape*
```

### 4.2.4 vector数据结构
vector是线性连续空间，以两个迭代器start和finish分别指向配置得来的连续空间中目前已被使用的范围，
迭代器end_of_storage指向整块连续空间（含备用空间）的尾端

**为了降低空间配置时的速度成本，vector实际配置的大小可能比客户端需求量更大一些，以备将来可能的扩充**
--> 即capacity的概念：一个vector的容量永远 >= 大小
**三个关键迭代器**：start、finish、end_of_storage

**vector在增加新元素时，如果超过当时的容量，则容量会扩充两倍，如果两倍仍然不足，则扩充至足够大的容量**
容量的扩张必须经历：重新配置、元素移动、释放原空间

### 4.2.5 vector的构造与内存管理

关于capacity的变化
如果是一个个地push_back元素，那么capacity的变化就是1，2，4，8...
并且不会变小，即便是clear()了，也不会变小


内部实现：
vector缺省使用alloc作为空间配置器，并据此另外定义了一个data_allocator，用于更方便地以元素大小为配置单位。

push_back插入元素会先检查是否有备用空间，没有则扩充（重新配置2倍的内存、移动数据、释放原空间）
注意：***对vector的任何操作，一旦涉及到空间的重新配置，指向原vector的所有迭代器就失效了**

### 4.2.6 vector的元素操作：pop_back、erase、clear、insert
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















