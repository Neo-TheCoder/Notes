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








# 关于接收方缓存大小的问题
GetNewSamples如果执行得太快，就会有数据没有来得及放入缓存中，也就是被丢弃了。





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
循环的起始位置是应用程序缓存中已经存在的样本数量，因为这些样本不需要从反应堆缓存中获取。
循环的终止条件是达到请求的样本数量或者反应堆缓存中的样本已经全部移动到应用程序缓存中。循环的每一次迭代都将反应堆缓存中的第一个样本移动到应用程序缓存的末尾，并从反应堆缓存中删除该样本。




