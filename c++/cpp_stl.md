
# 第二章 空间配置器(allocator)
### 2.2.3 构造和析构基本工具
construct()和destroy()

## 2.3 内存基本处理工具
STL定义了**五个全局函数，用于未初始化空间上**，有利于容器的实现

### 2.3.1 uninitialzied_copy
```cpp
    template<class InputIterator, class ForwardIterator>
    ForwardIterator
    uninitialized_copy(InputIterator first, InputIterator last, ForwardIterator result);
```
使我们能够将**内存的配置与对象的构造行为分离**
它会调用拷贝构造函数为入参指示范围的每一个对象产生一份copy对象，放入范围中
C++标准要求：***uninitialized_copy()要么构造出所有必要元素，要么（当有任何一个copy constructor失败时）不构造任何东西***

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















