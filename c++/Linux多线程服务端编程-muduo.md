
# 第一章 线程安全的对象生命期管理


## 1.1 当析构函数遇到多线程
与其他面向对象语言不同，C++ 要求程序员自己管理对象的生命期，这在多线程环境下显得尤为困难。
当一个对象能被多个线程同时看到时，那么对象的销毁时机就会变得模糊不清，可能出现多种竟态条件(race condition):
* 在即将析构一个对象时，从何而知此刻是否有别的线程正在执行该对象的成员函数？
* 如何保证在执行成员函数期间，对象不会在另一个线程被析构？
* 在调用某个对象的成员函数之前，如何得知这个对象还活着?它的析构函数会不会碰巧执行到一半？

使用`std::shared_ptr`可以一劳永逸解决这些问题。

### 1.1.1 线程安全的定义
`一个线程安全的class`满足以下三条件：
1. 多线程同时访问时，其表现出正确的行为
2. 无论操作系统如何调度这些线程，无论这些线程的执行顺序如何交织
3. 调用端代码无须额外的同步或其他协调操作
显然`std::vector`、`std::string`不是线程安全的


## 1.2 对象的创建容易
对象构造要做到线程安全，唯一的要求是`在构造期间不要泄露 this 指针`，即
* 不要在构造函数中注册任何回调（会把this指针暴露出去，其他线程可能拿到this指针，立刻执行，而本对象尚未构造完毕）;
* 也不要在构造函数中把`this`传给跨线程的对象:
* 即便在构造函数的最后一行也不行。

之所以这样规定，是因为在构造函数执行期间对象还没有完成初始化，如果 this被泄露（escape）给了其他对象（其自身创建的子对象除外），
那么别的线程有可能访问这个半成品对象，这会造成难以预料的后果。

也就是说，对于观察者模式（需要把自己的this指针传递出去），要用两段式构造（`构造函数()` + `Init()`）：构造函数 + 注册回调
既然允许二段式构造，那么构造函数不必主动抛异常（抛出异常的话，要注意内存等资源的释放），调用方靠`initialize()`的返回值来判断对象是否构造成功，这能简化错误处理。
即使构造函数的最后一行也不要泄露this，因为Foo有可能是个基类，基类先于派生类构造，执行完`Foo::Foo()`的最后一行代码还会继续执行派生类的构造函数，这时派生类对象还处于构造中，仍然不安全。
相对来说，对象的构造做到线程安全还是比较容易的，毕竟曝光少，回头率为零。
而析构的线程安全就不那么简单，这也是本章关注的焦点。


## 1.3 销毁困难
对象析构，这在单线程里不构成问题，最多需要注意避免空悬指针和野指针。而在多线程程序中，存在了太多的竞态条件。对一般成员函数而言，做到线程安全的办法是让它们顺次执行，而不要并发执行（关键是不要同时读写共享状态，也就是让每个成员函数的临界区不重叠。这是显而易见的，不过有一个隐含条件或许不是每个人都能立刻想到:成员函数用来保护临界区的互斥器本身必须是有效的。**而析构函数破坏了这一假设，它会把mutex成员变量销毁掉**！
- 空指针：指向已销毁的对象或已回收的地址
- 野指针：未初始化的指针


## 1.4 线程安全的Observer很难
**一个动态创建的对象是否还存活，是不能通过指针看出来的**：
* 如果销毁，就无法访问，无法访问就无从得知状态，
* 如果原址又创建了新的对象，则无从判断原对象。

面向对象中，对象间关系主要有三种（后两种关系容易出现内存问题）：
1. 组合
    对象x的生命周期由其唯一的拥有者owner控制，因而不会有多线程下的问题，外层对象析构的时候自然析构了x

2. 聚合
    类似大雁和雁群的关系，即整体与部分的关系
    如果b是动态建的并在整个程序结束前有可能被释放，那么就会出现谈到的竞态条件。

3. 关联
    很宽泛的关系
    对象a使用到了对象b，调用了后者的成员函数，例如：a持有b的指针，而b的生命周期不由a单独控制


那么似乎一个简单的解决办法是：只创建不销毁。
程序使用一个对象池来暂存用过的对象，下次申请新对象时，如果对象池里有存货，就重复利用现有的对象，否则就新建一个。对象用完了，不是直接释放掉，而是放回池子里。
这个办法当然有其自身的很多缺点，但至少能避免访问失效对象的情况发生。
这种山寨办法的问题有:
- 对象池的线程安全，如何安全地、完整地把对象放回池子里，防止出现“部分放回”的竞态？(线程A认为对象x已经放回了，线B认为对象x还活着。)
- 全局共享数据引发的lock contention（即由于获取锁而导致的并发度低的问题），这个集中化的对象池会不会把多线程并发的操作串行化？
- 如果共享对象的类型不止一种，那么是重复实现对象池还是使用类模板？
- 会不会造成内存泄漏与分片？因为对象池占用的内存只增不减，而且多个对象池不能共享内存(想想为何)。
回到正题上来，**如果对象x注册了任何非静态成员函数回调，那么必然在某处持有了指向x的指针，这就暴露在了race condition之下。**


## 1.5 裸指针为什么不行
指向对象的原始指针(raw pointer)是坏的，尤其当暴露给别的线程时。
observable应当保存的不是原始的 observer*，而是别的什么东西，能分辨 observer 对象是否存活。
类似地，如果 observer 要在析构函数里解注册(这虽然不能解决前面提到的race condition，但是在析构函数里打扫战场还是应该的)
那么 subject_的类型也不能是原始的 observable*。

### 空悬指针
有两个指针 p1和p2，指向堆上的同一个对象 object，p1和p2位于不同的线程中。
假设线程A通过p1指针将对象销了(尽管把p1置为了NULL)，那p2就成了空指针。
这是一种典型的C/C++ 内存错误。

要想安全地销毁对象，最好在别人(线程)都看不到的情况下，偷偷地做。
(这正是垃圾回收的原理：**所有人都用不到的东西一定是垃圾。**)

#### “解决办法”
引入一层间接性：让p1和p2指向的对象（称之为proxy）永久有效（p1和p2是二级指针）。
何时让proxy释放呢？

#### 优化
##### `引用计数`
为了安全地释放proxy，我们可以引入引用计数(reference counting)，再把p1和p2都从指针变成对象sp1和sp2。
proxy 现在有两个成员：指针、计数器


#### 一个万能的解决方案
引入另外一层间接性(another layer ofindirection)，用对象来管理共享资源(如果把object 看作资源的话)亦即 handle/body 惯用技法(idiom)。当然，编写线程安全、高效的引用计数 handle 的难度非凡作为一名谦卑的序员7，用现成的库就行。万幸，C++的TR1标准库里提供了一对“神兵利器”可助我们完美解决这个头疼的问题。

`shared_ptr<T>`是一个类模板(class template)，它只有一个类型参数，使用起来很方便。
引用计数是自动化资源管理的常用手法，当引用计数降为0时，对象（资源）即被销毁。
`weak_ptr<T>`也是一个引用计数型智能指针，但是它不增加对象的引用次数，即弱(weak)引用。
* `shared_ptr`控制对象的生命期。
shared_ptr是`强引用`(想象成用铁丝绑住堆上的对象)：只要有一个指向对象的 shared_ptr 存在，该x对象就不会析构。当指向对象x的最后一个shared_ptr析构或reset()的时候，保证会被销毁。

* `weak_ptr`不控制对象的生命期，但是它知道对象是否还活着(想象成用棉线轻轻拴住堆上的对象)。
如果对象还活着，那么它可以提升(promote)为有效的shared_ptr；
如果对象已经死了，提升会失败，返回一个空的shared_ptr。`提升/lock()`行为是线程安全的。

PS：`lock()`函数是 std::weak_ptr 的成员函数，它的作用是尝试获取所监视对象的一个 std::shared_ptr。
如果对象仍然存在，`lock()`函数会返回一个指向该对象的 std::shared_ptr；
如果对象已经被销毁，`lock()`函数会返回一个空的 std::shared_ptr。

`shared_ptr/weak_ptr`的`计数`在主流平台上是原子操作，没有用锁，性能不俗。
`shared_ptr/weak_ptr`的线程安全级别与std::string和STL容器样，后面还会讲。


***孟岩在《垃圾收集机制批判》中一针见血地点出能指针的优势:“C++利用智能指针达成的效采是:一旦某对象不再被引用，系统刻不容缓，立刻回收内存。这通常发生在关键任务完成后的清理 (clean up) 时期，不会影响关键任务的实时性同时，内存里所有的对象都是有用的，绝对没有垃圾空占内存。”***


## 1.7 系统避免各种指针错误
C++里可能出现的内存问题大致有这么几个方面
1. 缓冲区溢出(buffer overrun)。
2. 空悬指针/野指针。
3. 重复释放(double delete)。
4. 内存泄漏(memory leak)。
5. 不配对的 new[]/delete。
6. 内存碎片(memory fragmentation)。
正确使用智能指针能很轻易地解决前面5个问题，解决第6个问题需要别的思路。

1. 缓冲区溢出:用`std::vector<char>`/`std::string`或自已编写`Buffer class`来管理缓冲区，自动记住用缓冲区的长度，并通过成员函数而不是裸指针来修改缓冲区。
2. 空悬指针/野指针:用`shared_ptr/weak_ptr`，这正是本章的主题。
3. 重复释放:用scoped_ptr，只在对象析构的时候释放一次。
4. 内存泄漏:用scoped_ptr，对象析构的时候自动释放内存。
5. 不配对的new[]/delete: 把`new[]`统统替换为`std::vector/scoped_array`。

我认为，在现代的 C++ 序中一般不会出现 delete语句，资源(包括复杂对象本身)都是通过对象(智能指针或容器来管理的)，不需要程序员还为此操心。
在这几种错误里边，内存泄漏相对危害性较小，因为它只是借了东西不归还，程序功能在一段时间内还算正常。
其他如缓冲区溢出或重复释放等致命错误可能会造成安全性(security和data safety)方面的严重后果。
需要注意一点:`scoped_ptr/shared_ptr/weak_ptr`都是`值语意`（它强调对象拷贝的是其值，而非其身份或引用），要么是栈上对象，或是其他对象的直接数据成员，或是标准库容器里的元素。
几乎不会有下面这种用法:
```cpp
shared_ptr<Foo>* Foo = new shared_ptr<Foo>(new Foo); // WRONG semantic
```

还要注意，如果这几种智能指针是对象x的数据成员，而它的模板参数T是个incomplete类型，那么x的析构函数不能是默认的或内联的，必须在cpp文件里边显式定义，否则会有编译错或运行错。
PS：不完全类型（incomplete type）是指编译器在某个特定位置只看到了类型的声明，但没有看到完整的定义。
编译器在处理析构函数时需要知道如何正确地析构智能指针所管理的对象。而在这种情况下，编译器无法确定类型的大小和内部结构，因此无法正确地生成相关代码。

## 1.8 应用到Observer上
```cpp
class Observer : public boost::enable_shared_from_this<Observer>
{
 public:
  virtual ~Observer();
  virtual void update() = 0;

  void observe(Observable* s);

 protected:
  Observable* subject_;
};

class Observable
{
 public:
  void register_(boost::weak_ptr<Observer> x);
  // void unregister(boost::weak_ptr<Observer> x);

  void notifyObservers()
  {
    muduo::MutexLockGuard lock(mutex_);
    Iterator it = observers_.begin();
    while (it != observers_.end())
    {
      boost::shared_ptr<Observer> obj(it->lock());  // 提升为shared_ptr，只要对象至少被一个shared_ptr指着，就可以提升成功
      if (obj)
      {
        obj->update();  // 当前接口调用时，对象不可能被析构，因为obj尚未离开作用域，不可能被销毁
        ++it;
      }
      else  // 
      { // 观察者对象已销毁，则移除
        printf("notifyObservers() erase\n");
        it = observers_.erase(it);  // erase指向被删除的元素之后的位置
      }
    }
  }

 private:
  mutable muduo::MutexLock mutex_;
  std::vector<boost::weak_ptr<Observer> > observers_;   // 必须是weak_ptr，如果是shared_ptr会产生循环引用，对象无法正确销毁
  typedef std::vector<boost::weak_ptr<Observer> >::iterator Iterator;
};
```
PS：
`std::weak_ptr`没有重载`operator->`

PS：
`shared_ptr`的析构函数的伪代码：
循环引用是因为实际对象的析构函数被接管了，
只有（在shared_ptr的析构函数中判断：）当引用计数为0时才调用，而**shared_ptr对象作为类的成员对象，要在类的析构函数里调用**，
如果A持有了指向B对象的shared_ptr，而B同样，则shared_ptr变量的生命周期互相被对方的对象所管理了，A和B都定义在堆上，出现死局
```cpp
template <typename T>
std::shared_ptr<T>::~shared_ptr() {
    if (ptr_ != nullptr) { // ptr_ 是指向所管理对象的指针
        // 减少引用计数
        decrement_reference_count();

        // 如果引用计数变为零
        if (reference_count_ == 0) {
            // 调用删除器删除对象
            deleter_(ptr_);
        }
    }
}
```

### 缺陷
* 侵入性
强制要求observer必须以`shared_ptr`来管理。

* 不是完全线程安全
observer的析构函数会调用`subject_->unregister(this);`
万一subject_已经不复存在了呢？为了解决它，又要求observable本身是用shared_ptr管理的，并且subject_多半是个`weak_ptr<observable>`。

* 锁争用(lock contention)
即 observable 的三个成员函数都用了互斥器来同步（对临界资源的访问，需要加锁以保证结果的一致性），
这会造成`register_()`和`unregister()`等待`notifybservers()`，而后者的执行时间是无上限的，因为它同步回调了用户提供的`update`函数（如果极其缓慢）。
我们希望`register_()`和`unregister()`的执行时间不会超过某个固定的上限，以免殃及无辜群众。

* 死锁
万一`update()`虚函数中调用了`(un)register`呢？
如果`mutex_`是不可重入的，那么会死锁；
如果`mutex_`是可重入的，程序会面临迭代器失效 (core dump是最好的结果)，因为`vector<observers_>`在遍历期间被意外地修改了（`register`会操作`vector<observers_>`）。
这个问题乍看起来似乎没有解决办法，除非在文档里做要求。
（一种办法是:用可重入的mutex_，把容器换为`std::list`，并把`++it`往前挪一行。
我个人倾向于使用不可重入的mutex，例如 Pthreads 默认提供的那个，因为**“要求mutex可重人”本身往往意味着设计上出了问题**(2.1.1)。
Java的intrinsiclock是可重入的，因为要允许synchronized方法相互调用（派生类调用基类的同名synchronized方法），我觉得这也是无奈之举）。


## 1.9 再论shared_ptr的线程安全
虽然我们借shared_ptr 来实现线程安全的对象释放，但是shared_ptr 本身不是100% 线程安全的。
它的引用计数本身是安全且无锁的，但对象的读写则不是，**因为shared_ptr有两个数据成员，读写操作不能原子化**。

根据文档，shared_ptr的线程安全级别和内建类型、标准库容器、std::string 一样，
即：一个shared_ptr对象实体可被多个线程`同时读取`。
两个shared_ptr对象实体可以被两个线程同时写入，`析构`算写操作。
如果要从多个线程读写同一个shared_ptr对象，那么需要加锁。
请注意，以上是shared_ptr对象本身的线程安全级别，不是它管理的对象的线程安全级别。

以下代码的临界区很小，只有简单的赋值操作，读操作很快
```cpp
// globalPtr能被多个线程看到，那么它的读写需要加锁。注意我们不必用读写锁
// 而只用最简单的互斥锁，这是为了性能考虑。
// 因为临界区非常小，用互斥锁也不会阻塞并发读。
// 为了拷贝 globalPtr，需要在读取它的时候加锁，即:
void read()
{
  std::shared_ptr<Foo> localPtr;
  {
    MutexLockGuard lock(mutex);
    localPtr = globalPtr; // read globalPtr
  }
  // use localPtr since here，读写 localPtr 也无须加锁
  doit(localPtr);   // 假设不对shared_ptr指向的对象做操作
}

// 写入的时候也要加锁:
void write()
{
  shared_ptr<Foo> newPtr(new Foo); // 注意，对象的创建在临界区之外，对象的创建比较耗时，这样可以缩小临界区
  {
    MutexLockGuard lock(mutex);
    globalPtr = newPtr; // write to globalPtr
  }
  //use newPtr since here，读写 newPtr 无须加锁
  doit(newPtr);
}
```
注意到上面的`read()`和`write()`在临界区之外都没有再访问globalPtr，
而是用了一个指向同一Foo对象的栈上shared_ptr，local copy。
下面会谈到，只要有这样的local copy存在，shared_ptr作为函数参数传递时不必复制，用`reference to const`作为参数类型即可。

另外注意到上面的`new Foo`是在临界区之外执行的，这种写法通常比在临界区内写`globalPtr.reset(new Foo)`要好，因为缩短了临界区长度。
如果要销毁对象，我们固然可以在临界区内执行`globalPtr.reset()`，但是这样往往会让对象析构（有一定的性能开销）发生在临界区以内，增加了临界区的长度。
一种改进办法是像上面一样定义一个`localPtr`，用它在临界区内与`globalPtr`交换(`swap()`)，
这样能保证把对象的销毁推迟到临界区之外。

练习：在`write()`函数中，`globalPtr = newPtr;`
这一句有可能会在临界区内销毁原来globalptr指向的Foo对象（`operator=`可能导致引用计数减一为0），设法将销毁行为移出临界区。


## 1.10 shared_ptr技术与陷阱
### 意外延长对象的生命期
`shared_ptr`是强引用(铁丝绑的)，只要有一个指向x对象的`shared_ptr`存在，该对象就不会析构。
而`shared_ptr`又是允许拷贝构造和赋值的(否则引用计数就无意义了)，如果不小心遗留了一个拷贝，那么对象就永世长存了。
例如前面提到如果把 p.16中observers_的类型改为`vector<shared_ptr<observer>>`，那么除非手动调用`unregister()`（从而调用`pop()`），否则observer对象永远不会析构（因为即使shared_ptr对象析构了，其指向的对象也未必析构）。
即便它的析构函数会调用`unregister()`，但是不`unregister()`就不会调用observer的析构函数，这变成了鸡与蛋的问题。这也是Java内存泄漏的常见原因。

另外一个出错的可能是`boost::bind`，因为`boost::bind`会把实参拷贝一份，如果参数是个`shared_ptr`，那么对象的生命期就不会短于`boost::function`对象:
```cpp
class Foo
{
  void doit();
};
shared_ptr<Foo> pFoo(new Foo);
boost::function<void()> func = boost::bind(&Foo::doit, pFoo); // long life foo
```

这里`func对象`持有了`shared_ptr<Foo>`的一份拷贝，有可能会在不经意间延长倒数第二行创建的`Foo`对象的生命期。

### 函数参数
因为要修改引用计数(而且拷贝的时候通常要加锁)，`shared_ptr`的拷贝开销比拷贝原始指针要高，但是需要拷贝的时候并不多。
多数情况下它可以以`const reference 方式传递`，一个线程只需要在最外层函数有一个实体对象，之后都可以用const reference来使用这个 shared_ptr。例如有几个函数都要用到 Foo 对象:
```cpp
void save(const shared_ptr<Foo>& pFoo); // pass by const reference
void validateAccount(const Foo& foo);
bool validate(const shared_ptr<Foo>& pFoo) // pass by const reference
{
  validateAccount(*pFoo);
  // ...
}
// 那么在通常情况下，我们可以传常引用(pass by const reference):

void onMessage(const string& msg)
{
  // 只要在最外层持有一个实体，安全不成问题
  shared_ptr<Foo> pFoo(new Foo(msg)):
  if (validate(pFoo)){
  // 没有拷贝 pFoo
    save(pFoo);
  }
}
```
遵照这个规则，基本上不会遇到反复拷贝 shared_ptr 导致的性能问题。
另外由于 pFoo 是栈上对象，不可能被别的线程看到，那么读取始终是线程安全的。

### 析构动作在创建时被捕获
这是一个非常有用的特性，这意味着:
* 虚析构不再是必需的。
  ```cpp
    std::shared_ptr<Base> ptr = std::make_shared<Derived>();
  ```
  为什么能够调用派生类的析构函数：
  可能是取得了派生类的类型，同时释放了this指针，具体需要看`std::shared_ptr`代码

* `shared_ptr<void>`可以持有任何对象，而且能安全地释放。
* shared_ptr对象可以安全地跨越模块边界，比如从DLL里返回，而不会造成从模块A分配的内存在模块B里被释放这种错误。
* `二进制兼容性`：
  即便 Foo 对象的大小变了，那么旧的客户代码仍然可以使用新的动态库，而无须重新编译。
  前提是 Foo 的头文件中不出现访问对象的成员的inline函数，并且Foo对象的由动态库中的Factory构造，返回其shared_ptr。
* 析构动作可以定制。

PS：如果客户代码里有`new Bar`，那么肯定不安全，因为 new 的字节数不够装下新 Bar。
相反，如果 library 通过 factory 返回 Bar* （并通过 factory 来销毁对象）或者直接返回 shared_ptr<Bar>，客户端不需要用到 sizeof(Bar)，那么可能是安全的。同样的道理，直接定义 Bar bar; 对象（无论是函数局部对象还是作为其他 class 的成员）也有二进制兼容问题。**重点就是客户不直接调用Bar**

最后这个特性的实现比较巧妙，因为`shared_ptr<T>`只有一个模板参数，而`析构行为`可以是函数指针、仿函数(functor)或者其他什么东西。
这是泛型编程和面向对象编程的一次完美结合。有兴趣的读者可以参考Scott Meyers 的文章。这个技术在后面的对象池中还会用到。


### 析构所在的线程
对象的析构是同步的，当最后一个指x的shared_ptr离开其作用域的时候，x会同时在同一个线程析构。这个线程不一定是对象诞生的线程。
这个特性是把双刃剑：如果对象的析构比较耗时，那么可能会拖慢`关键线程`的速度(如果最后一个shared_ptr引发的析构发生在关键线程);
同时，我们可以用一个`单独的线程`来专门做析构，通过一个`BlockingQueue<shared_ptr<void>>`把对象的析构都转移到那个专用线程，从而解放关键线程。


### 现成的RAII handle
我认为`RAII(资源取即初始化)`是C++语言区别于其他所有编程语言的最重要的特性，一个不懂 RAII的 C++ 程序员不是一个合格的C++程序员。
初学C++的教条是`new 和 delete要配对，new了之后要记着 delete`，
如果使用 RAII[CCS,条款13]，要改成`每一个明确的资源配置动作(例如 new)都应该在单一语句中执行，并在该语句中立刻将配置获得的资源交给 handle 对象(如shared_ptr)，程序中一般不出现 delete`。
shared_ptr 是管理共享资源的利器，需要注意避免循环引用，通常的做法是 owner 持有指向child 的 shared_ptr，child 持有指向owner的weak_ptr。


## 1.11 对象池
假设有`Stock`类，代表一只股票的价格。
每一只股票有一个唯一的字符串标识，比如Google 的 key 是`NASDAQ:GOOG`，IBM是`NYSE:IBM`。
Stock 对象是个主动对象，它能不断获取新价格。为了节省系统资源，同一个程序里边每一只出现的股票只有一个 Stock 对象，如果多处用到同一只股票，那么Stock 对象应该被共享。
如果某一只股票没有再在任何地方用到，其对应的 Stock 对象应该析构，以释放资源，这隐含了`“引用计数”`。
为了达到上述要求，我们可以设计一个`对象池 StockFactory`。
它的接口很简单，根据 key 返回 Stock 对象。
我们已经知道，在多线程程序中，既然对象可能被销毁，那么返回shared_ptr 是合理的。自然地，我们写出如下代码(可惜是错的)
```cpp
// version 1: questionable code
class StockFactory : boost::noncopyable
public:
  shared_ptr<Stock> get(const string& key);

private:
  mutable MutexLock mutex_;
  std::map<string，shared_ptr<Stock>> stocks_;  // 就目前接口而言，只要map存在，Stock对象就一直存在
```

`get()`的逻辑很简单，如果在stocks_里找到了 key，就返回 stocks_[key];
否则新建一个 Stock，并存入 stocks_[key]。

细心的读者或许已经发现这里有一个问题，stock 对象永远不会被销毁，因为map 里存的是 shared_ptr，始终有“铁丝”绑着。那么或许应该仿照前面observable那样存一个weak_ptr?
比如
```cpp
// // version 2:数据成员修改为 std::map<string，weak_ptr<Stock> >stocks_;
shared_ptr<Stock> StockFactory::get(const string& key){
  shared_ptr<Stock> pStock;
  MutexLockGuard lock(mutex_);
  weak_ptr<Stock>& wkStock = stocks_[key]; // 如果 key 不存在，会默认构造一个（这是std::map的特性！）
  pStock = wkStock.lock(); //尝试把“棉线”提升为“铁丝”
  if (!pStock){
    pStock.reset(new Stock(key));   // 析构原先对象，确保map里的都是最新的Stock对象
    wkStock = pStock; // 这里更新了 stocks_[key]，注意 wkStock 是个引用
  }
  return pStock;
}
```


这么做固然 Stock 对象是销毁了，但是程序却出现了`轻微的内存泄漏`，为什么?
因为`stocks_`的大小只增不减，`stocks_.size()`是曾经存活过的 Stock 对象的总数，即便活的 stock 对象数目降为0。（因为`std::weak_ptr`始终在占据内存空间）
或许有人认为这不算泄漏，因为内存并不是彻底遗失不能访问了，而是被某个标准库容器占用了。我认为这也算内存泄漏，毕竟是“战场”没有打扫干净。

其实，考虑到世界上的股票数目是有限的，这个内存不会一直泄漏下去，大不了把每只股票的对象都创建一遍，估计泄漏的内存也只有几兆字节。
如果这是一个`其他类型的对象池`，对象的 key 的集合不是封闭的，内存就会一直泄漏下去。
解决的办法是，**利用`shared_ptr`的定制析构功能**。
`shared_ptr`的构造函数可以有一个额外的模板类型参数，传人一个函数指针或仿函数 d，在析构对象时执行d(ptr)，其中 ptr是 shared_ptr 保存的对象指针。
`shared_ptr`这么设计并不是多余的，因为反正要在创建对象时捕获释放动作，始终需要一个 bridge。

```cpp
template<class Y，class D>
shared_ptr::shared_ptr(Y* p，D d);

template<class Y，class D>
void shared_ptr::reset(Y* p，D d);
//注意 Y的类型可能与 T不同，这是合法的，只要 Y* 能隐式转换为 T*。
```
那么我们可以利用这一点，在析构 Stock 对象的同时清理 stocks_。

```cpp
// version 3
class StockFactory : boost::noncopyable
{
  //在 get()中，将 pStock.reset(new Stock(key));改为:
  pStock.reset(new Stock(key)，boost::bind(&StockFactory::deleteStock，this，_1));
  private:
  void deleteStock(Stock* stock)
  {
  if (stock) {
    MutexLockGuard lock(mutex_);
    stocks_.erase(stock->key());  // ！！！清除掉map里的元素
  }
  delete stock; // sorry，I lied
  // assuming StockFactory lives longer than all Stock's ...
  }
}
```

这里我们向`pStock.reset()`传递了第二个参数，一个boost::function，让它在析构Stock* p时调用本 StockFactory 对象的`deleteStock`成员函数。
警惕的读者可能已经发现问题，那就是我们把一个原始的`StockFactory this`指针保存在了`boost::function`里，这会有线程安全问题。
如果这个`Stock-Factory`先于`Stock`对象析构，那么会`core dump`。
正如observer在析构函数里去调用`Observable::unregister()`，而那时Observable对象可能已经不存在了。
通过`弱回调`技术可以解决


### 1.11.1 `enable_shared_from_this`
用于解决`在类成员函数中`获取`shared_ptr`的问题，使用时，要继承`enable_shared_from_this`这一基类

`StockFactory::get()`把原始指针 this 保存到了boost::function中
如果 StockFactory 的生命期比 Stock 短，那么Stock析构时去调StockFactory::deleteStock 就会 core dump。
似乎我们应该祭出惯用的 shared_ptr 大法来解决对象生命期问题，但是`StockFactory::get()`本身是个成员函数，如何获得一个`指向当前对象`的`shared_ptr<stockFactory>`对象呢？
有办法，用`enable_shared_from_this`。
这是一个`以其派生类为模板类型实参 的 基类 模板`，继承它，`this`指针就能变身为`shared_ptr`。
```cpp
class StockFactory : public boost::enable_shared_from_this<StockFactory>, boost::noncopyable
{/*...*/};
```
为了使用`shared_from_this()`，StockFactory不能是stack object，必须是heap obiect，且由 shared_ptr 管理其生命期，即`shared_ptr<StockFactory> stockFactory(new StockFactory);`
万事俱备，可以让`this`摇身一变，化为`shared_ptr<stockFactory>`了

```cpp
// version 4
shared_ptr<Stock> StockFactory::get(const string& key)
{
// change
// pStock.reset(new Stock(key)， boost::bind(&StockFactory::deleteStock， this，_1));
// to
  pStock.reset(new Stock(key), boost::bind(&StockFactory::deleteStock, shared_from_this(), _1));
}
```

这样一来，boost::function里保存了一份`shared_ptr<StockFactory>`，可以保证调用`StockFactory::deleteStock`的时候那个`StockFactory`对象还活着。注意一点，`shared_from_this()`不能在构造函数里调用，因为在构造`StockFactory`的时候，它还没有被交给shared_ptr 接管。
最后一个问题，stockFactory 的生命期似乎被`意外延长`了。



### 1.11.2 弱回调
把`shared_ptr`绑(`boost::bind`)到`boost:function`里，那么回调的时候`StockFactory对象`始终存在，是安全的。
这同时也延长了对象的生命期，使之不短于绑得的 boost:function 对象。

**有时候我们需要`“如果对象还活着，就调用它的成员函数，否则忽略之”`的语意**
就像`observable::notifyobservers()`那样，我称之为“弱回调”。
这也是可以实现的，利用`weak_ptr`，我们可以把 weak_ptr 绑到 boost::function 里，这样对象的生命期就不会被延长。
然后在回调的时候先尝试提升为 shared_ptr，
如果提升成功，说明接受回调的对象还健在，那么就执行回调;
如果提升失败，就不必劳神了。

使用这一技术的完整 StockFactory 代码如下:
```cpp
class StockFactory : public boost::enable_shared_from_this<StockFactory>, boost::noncopyable
{
public:
  shared_ptr<Stock> get(const string& key)
  {
    shared_ptr<Stock> pStock;
    MutexLockGuard lock(mutex_);
    weak_ptr<Stock>& wkStock = stocks_[key]; // 注意 wkStock 是引用
    pStock = wkStock.lock();
    if (!pStock){
    pStock.reset(new Stock(key), boost::bind(&StockFactory::weakDeleteCallback, boost::weak_ptr<StockFactory>(shared_from_this()), _1));
    wkStock = pStock;
    }
  return pStock;
  }

// 上面必须强制把shared_from_this()转型为 weak_ptr，才不会延长生命期
// 因为 boost::bind 拷贝的是实参的类型，不是形参的类型，从而拷贝weak_ptr类型

 private:
  static void weakDeleteCallback(const boost::weak_ptr<StockFactory>& wkFactory,
                                 Stock* stock)
  {
    printf("weakDeleteStock[%p]\n", stock);
    boost::shared_ptr<StockFactory> factory(wkFactory.lock());
    if (factory)
    {
      factory->removeStock(stock);
    }
    else
    {
      printf("factory died.\n");
    }
    delete stock;  // sorry, I lied
  }
};
```


当然，通常 Factory对象是个singleton，在程序正常运行期间不会销毁，这里只是为了展示弱回调技术 15，这个技术在事件通知中非常有用。
本节的stockFactory 只有针对单个Stock 对象的操作，如果程序需要遍历整个stocks_，稍不注意就会造成死锁或数据损坏(S2.1)请参考S2.8 的解决办法。



## 1.12 替代方案
除了使用`shared_ptr/weak_ptr`，要想在C++里做到线程安全的对象回调与析构，可能的办法有以下一些：
1. 用一个全局的 facade 来代理 Foo类型对象访问，所有的 Foo 对象回调和析构都通过这个facade来做，也就是把指针替换为 objId/handle，每次要调用对象的成员函数的时候先check-out，用完之后再check-in（释放所有权）。这样理论能避免racecondition，但是代价很大：
因为要想把这个 facade 做成线程安全的，那么必然要用`互斥锁`。
这样一来，从两个线程访问两个不同的 Foo 对象也会用到同一个锁，让本来能够并行执行的函数变成了串行执行，没能发挥多核的优势。
当然可以像Java的 ConcurrentHashMap 那样用多个buckets，每个bucket分别加锁以降低contention。

2. 1.4提到的“只创建不销毁”手法，实属无奈之举。

3. 自己编写引用计数的智能指针。
本质上是重新发明轮子，把shared_ptr 实现一遍。
正确实现线程安全的引用计数智能指针不是一件容易的事情，而高效的实现就更加困难。
既然shared_ptr 已经提供了完整的解决方案，那么似乎没有理由抗拒它。
将来在C++11里有unique_ptr，能避免引用计数的开销，或许能在某些场合替换shared_ptr。


## 1.13 心得与小结
学习多线程程序设计远远不是看看教程了解API怎么用那么简单，这最多“主要是为了读懂别人的代码，如果自己要写这类代码，必须专门花时间严肃、认真、系统地学习，严禁半桶水上阵”(孟岩)。
一般的多线程教程上都会提到要让`加锁的区域足够小`，这没错，问题是如何找出这样的区域并加锁，本章 1.9 举的安全读写shared_ptr可算是一个例子。

据我所知，目前 C++ 没有特别好的多线程领域专著，但C语言有，Java 语言也有。《Java Concurrency in Practice》CP]是我读过的写得最好的书，内容足够新可读性和可操作性俱佳。
C++ 程序员反过来要向Java 学习，多少有些讽刺。
除了编程书，操作系统教材也是必读的，至少要完整地学习一本经典教材的相关章节，可从《操作系统设计与实现》《现代操作系统》《操作系统概念》任选一本，了解各种同步原语、临界区、竞态条件、死锁、典型的IPC 问题等等，防止闭门造车。
分析可能出现的race condition不仅是多线编程的基本功，也是设计分布式系统的基本功，需要反复历练，形成一定的思考范式，并积累一些经验教训，才能少犯错误。这是一个快速发展的领域，要不断吸收新知识，才不会落伍。单 CPU 时代的多线程编程经验到了多CPU时代不一定有效，因为多CPU能做到真正的并行执行，**每个CPU看到的事件发生顺序不一定完全相同**。
正如狭义相对论所说的每个观察者都有自己的时钟，在不违反因果律的前提下，可能发生十分违反直觉的事情。
尽管本章通篇在讲如何安全地使用(包括析构)跨线程的对象，但我建议`尽量减少使用跨线程的对象`，我赞同水木网友 ilovecpp 说的:“用流水线，生产者消费者，任务队列这些有规律的机制，最低限度地共享数据。这是我所知最好的多线程编程的建议了。”
不用跨线程的对象，自然不会遇到本章描述的各种险态。如果迫不得已要用，希望本章内容能对你有帮助。


## 1.14 Observer之谬

本章1.8把`shared_ptr/weak_ptr`应用到Observer模式中，部分解决了其线程安全问题。
我用 Observer 举例，因为这是一个广为人知的设计模式，但是它有本质的问题。
Observer 模式的本质问题在于其面向对象的设计。
换句话说，我认为正是面向对象(OO)本身造成了 Observer 的缺点。
`observer 是基类`，这带来了非常强的耦合，强度仅次于友元(friend)。
这种耦合不仅限制了：成员函数的名字、参数、返回值，
还限制了成员函数所属的类型（必须是observer 的派生类）。

PS：
* `目标对象`和`观察者对象`之间的`直接依赖关系`：
  在观察者模式中，`目标对象`需要维护一个`观察者列表`，并`在状态变化时通知观察者`。
  这导致目标对象和观察者对象之间存在直接的依赖关系，目标对象需要知道观察者对象的存在并与之进行交互。
  这种直接的依赖关系增加了两者之间的耦合程度。

* 观察者接口的实现：观察者模式要求观察者对象实现一个共同的接口，以便`目标对象 可以 调用观察者的特定方法进行通知`。
  这意味着`观察者对象需要：按照接口定义实现相应的方法`，这种实现细节的要求增加了观察者对象的耦合度。

* 目标对象的通知方式：在观察者模式中，`目标对象通常通过直接调用观察者对象的方法来进行通知`。
这种直接的调用方式使得目标对象需要了解观察者对象的具体实现细节，进一步增加了两者之间的耦合程度。

`Observer class`是基类，这意味着如果 Foo 想要观察两个类型的事件(比如时钟和温度，分别需要实现：时钟观察者和温度观察者)，需要使用多继承。
这还不是最糟糕的，如果要重复观察同一类型的事件(比如1秒一次的心跳和30一次的自检)，就要用到一些伎俩来 work around，因为不能从一个Base class 继承两次。

在C++里为了替换Observer，可以用`Signal/Slots`，我指的不是QT那种靠语言扩展的实现，而是完全靠标准库实现的 thread safe、race condition free、thread contention free的Signal/Slots，
并且不强制要求 shared_ptr 来管理对象，也就是说完全解决了1.8列出的Observer 遗留问题。这会用到2.8介绍的`“借shared_ptr实现copy-on-write”`技术。
在C++11中，借助可变模板参数（variadic template），实现最简单(trivial)的一对多回调可谓不费吹灰之力，代码如下。
？？？多思考下
```cpp
template <typename RET, typename... ARGS>
class SignalTrivial<RET(ARGS...)>
{
 public:
  typedef std::function<void (ARGS...)> Functor;

  void connect(Functor&& func)
  {
    functors_.push_back(std::forward<Functor>(func));
  }

  void call(ARGS&&... args)
  {
    // gcc 4.6 supports
    //for (const Functor& f: functors_)
    typename std::vector<Functor>::iterator it = functors_.begin();
    for (; it != functors_.end(); ++it)
    {
      (*it)(args...);
    }
  }

 private:
  std::vector<Functor> functors_;
};
```



# 第二章 线程同步精要
并发编程有两种基本模型，一种是`message passing`，另一种是`shared memory`。
在分布式系统中，运行在多台机器上的多个进程的并行编程只有一种实用模型`message passing`。
在单机上，我们也可以照搬message passing 作为多个进程的并发模型。
这样整个分布式系统的架构的一致性很强，扩容(scale out)起来也较容易。
在多线程编程中，message passing 更容易保证程序的正确性，有的语言只提供这一种模型。
不过在用C/C++ 编写多线程程序时，我们仍然需要了解底层的 shared memory模型下的同步原语，以备不时之需。
本章不是多线程教程，而是个人经验总结，分享一些C++多线程编程的经验。
本章多次用《Real-World Concurrency》一文的观点，这篇文章的地址是 http://queue.acm.org/detailcfm?id=1454462，后文简称[RWC]。

**线程同步的四项原则**，按重要性排列:
1. 首要原则是尽量最低限度地共享对象，`减少需要同步的场合`。
一个对象 能不暴露给别的线程就不要暴露：
  如果要暴露，优先考虑`immutable对象`（即只读对象）
  实在不行才暴露可修改的对象，并用`同步措施`来充分保护它。

2. 其次是使用`高级的并发编程构件`
  如：`TaskQueue`、`Producer-Consumer Queue`、`CountDownLatch`（？？？）等等。

3. 最后不得已必须使用`底层同步原语`(primitives)时，只使用： `非递归的互斥器`和`条件变量`， 慎用`读写锁`， `不要用信号量`。

4. 除了使用`atomic整数之外`，`不自己编写lock-free代码`，也不要用`“内核级同步原语”`（不是由C++提供）。
  不凭空猜测“哪种做法性能会更好”，比如`spin lock` vs. `mutex`。

前面两条很容易理解，这里着重讲一下第 3 条:底层同步原语的使用。


## 2.1 互斥器
`互斥器(mutex)`恐怕是使用得最多的同步原语。
粗略地说，它保护了临界区，**任何一个时刻最多只能有一个线程在此mutex划出的临界区内活动**。
单独使用mutex时，我们主要为了`保护共享数据`。
我个人的原则是:
* 用`RAII手法`封装mutex的`创建`、`销毁`、`加锁`、`解锁`这四个操作。
用RAII封装这几个操作是通行的做法，这几乎是 C++ 的标准实践，后面我会给出具体的代码示例，相信大家都已经写过或用过类似的代码了
。Java 里的synchronized语句和C#的using语句也有类似的效果，即保证锁的生效期间等于一个作用域(scope)，不会因异常而忘记解锁。

* 只用`非递归的mutex`(即不可重入的mutex)。

* 不手工调用`lock()`和`unlock()`函数，一切交给栈上的Guard对象的`构造和析构函数`负责。
`Guard 对象的生命期`正好等于临界区(分析对象在什么时候析构，是C++程序员的基本功)。
这样我们保证始终在`同一个函数`、`同一个scope`里对某个mutex加锁和解锁。
避免在foo()里加锁，然后跑到bar()里解锁;
也避免在不同的语句分支中分别加锁、解锁。
这种做法被称为`Scoped Locking`。
在每次构造Guard 对象的时候，思考一路上(`调用栈上`)`已经持有的锁`，防止因加锁顺序不同而导致`死锁`(deadlock)。
由于Guard对象是栈上对象，看函数调用栈就能分析用锁的情况，非常便利。


次要原则有:
* 不使用`跨进程`的mutex，进程间通信只用`TCP sockets`。
* `加锁、解锁在同一个线程`，线程a不能去unlock线程b已经锁住的mutex(RAII自动保证)。
* 别忘了解锁(RAII自动保证)。
* 不重复解锁(RAII自动保证)。
* 必要的时候可以考虑用`PTHREAD_MUTEX_ERRORCHECK`（是互斥量的属性之一，自动检测死锁）来排错。
mutex恐怕是最简单的同步原语，按照上面的几条原则，几乎不可能用错。
我自已从来没有违背过这些原则，编码时出现问题都很快能定位并修复。



### 2.1.1 只使用非递归的mutex
谈谈我坚持使用非递归的互斥器的个人想法mutex分为递归(recursive)和非递归(non-recursive)两种，
这是POSIX的叫法，另外的名字是可重入(reentrant)与非可重入。
这两种mutex作为线程间(inter-thread)的同步工具时没有区别，它们的唯一区别在于：**同一个线程可以重复对recursive mutex加锁，但是不能重复对non-recursive mutex加锁**。

首选非递归mutex，绝对不是为了性能，而是为了**体现设计意图**。
non-recursive和recursive 的性能差别其实不大，因为少用一个计数器，前者略快一点点而已。
在同一个线程里多次对non-recursive mutex加锁会立刻导致死锁，我认为这是它的优点，能帮助我们思考代码对锁的期求，并且及早(在编码阶段)发现问题。
毫无疑问recursive mutex使用起来要方便一些，因为不用考虑一个线程会自己把自已给锁死了，我猜这也是Java和Windows 默认提供recursive mutex的原因。


正因为它方便，recursive mutex可能会隐藏代码里的一些问题。
典型情况是你以为拿到一个锁就能修改对象了，没想到外层代码已经拿到了锁，正在修改(或读取)同一个对象呢。
来看一个具体的例子(recipes/thread/test/NonRecursiveMutex test.cc):
```cpp
  MutexLock mutex;
  std::vector<Foo> foos;

  void post(const Foo& f)
  {
    MutexLockGuard lock(mutex);
    foos.push_back(f);
  }

  void traverse()
  {
    MutexLockGuard lock(mutex);
    for (std::vector<Foo>::const_iterator it = foos.begin(); it != foos.end(); ++it)
    {
      it->doit();
    }
  }
```

post()加锁，然后修改 foos 对象;
traverse() 加锁，然后遍历 foos 向量。这些都是正确的。


将来有一天，`Foo::doit()`间接调用了`post()`（重复获取锁），那么会很有戏剧性的结果:
1. mutex 是非递归的，于是死锁了。
2. mutex是递归的，由于push_back()可能(但不总是)导致`vector选代器失效`（因为被修改了），程序偶尔会crash。

这时候就能体现non-recursive 的优越性：**把程序的逻辑错误暴露出来**。
死锁比较容易 debug，把各个线程的调用栈打出来，只要每个函数不是特别长，很容易看出来是怎么死的，见2.1.2的例子9。
或者可以用`PTHREAD_MUTEX_ERRORCHECK`一下子就能找到错误(前提是 MutexLock 带 debug 选项)。
程序反正要死，不如死得有意义点，留个“全尸”，让验尸(post-mortem)更容易些。

如果确实需要在遍历的时候修改vector，有两种做法：
1. 一是把修改推后，记住循环中试图添加或删除哪些元素，**等循环结束了再依记录修改 foos**;
2. 二是用`copy-on-write`，见2.8的例子。

如果一个函数既可能在已加锁的情况下调用，又可能在未加锁的情况下调用，那么就拆成两个函数:
1. 跟原来的函数同名，函数加锁，转而调用第2个函数。
2. 给函数名加上后缀 withLockHold，不加锁，把原来的函数体搬过来。
（总之就是两份实现）

就像这样:
```cpp
void post(const Foo& f){
  MutexLockGuard lock(mutex);
  postwithLockHold(f); // 不用担心函数调用的开销，编译器会自动内联的
}

//引入这个函数是为了体现代码作者的意图，尽管 push_back 通常可以手动内联
void postWithLockHold(const Foo& f)
{
  foos.push_back(f);
}
```

这有可能出现两个问题(感谢水网友 ovecpp 提出):
1. 误用了加锁版本，死锁了。
2. 误用了不加锁版本，数据损坏了。


对于1，仿造2.1.2的办法能比较容易地排错。
对于2，如果Pthreads提供`isLockedByThisThread()`就好办，可以写成:
```cpp
void postWithLockHold(const Foo& f)
{
  assert(mutex.isLockedByThisThread()); // muduo::MutexLock 提供了这个成员函数
  // ...
}
```

另外，withLockHold这个显眼的后缀也让程序中的误用容易暴露出来。

我还没有遇到过需要使用recursive mutex的情况，我想将来遇到了都可以借助wrapper改用non-recursive mutex，代码只会更清晰

Pthreads的权威专家，《Programming with POSIX Threads》的作者 David Butenhof 也排斥使用recursive mutex。他说:
***First, implementation of efficient and reliable threaded code revolves around one simple and basic principle: follow your design. That implies, of course, that you have a design, and that you understand it. A correct and well understood design does not require recursive mutexes.***
回到正题。本文这里只谈了 mutex本身的正确使用，在C++里多线编程还会遇到其他一些 race condition，请参看第1章

性能注脚:Linux的Pthreads mutex 采用`futex(2)`？？？实现，不必每次加锁、解锁都陷入系统调用，效率不错。
Windows 的CRITICAL_SECTION 也是类似的，不过它可以嵌人一小段spin lock。在多CPU系统上，如果不能立刻拿到锁，它会先 spin 小段时间，如果还不能拿到锁，才挂起当前线程。

### 2.1.2 死锁
前面说过，如果坚持只使用`Scoped Locking`（在同一个函数、同一个作用域，加解锁），那么在出现死锁的时候很容易定位。
考虑下面这个线程自己与自已死锁的例子(recipes/thread/test/SelfDeadLock.cc)。

```cpp


class Request
{
public:
void process() // __attribute_- ((noinline))
{
    muduo::MutexLockGuard lock(mutex_);
    // ...
    print(); // 原本没有这行，某人为了调试程序不小心添加了。
}

void print() const // -_attribute_- ((noinline))
{
  muduo::MutexLockGuard lock(mutex_);   // 重复获取锁，导致死锁
  // ...
}
private:
  mutable muduo::MutexLock mutex_;
};

int main()
{
  Request req;
  req.process0;
}
```

`死锁示例`
有一个Inventory(清单)class，记录当前的 Request 对象。
容易看出，下面这个Inventory class 的 add()和 remove()成员函数都是线程安全的，
它使用了mutex来保护共享数据requests_

```cpp
class Inventory
{
public:
  void add(Request* req)
  {
    muduo::MutexLockGuard lock(mutex_);
    requests_.insert(req);
  }

  void remove(Request* req) // -_attribute_- ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    requests_.erase(req);
  }

  void printAll() const;
private:
  mutable muduo::MutexLock mutex_;
  std::set<Request*> requests_;
};

Inventory g_inventory; //为了简单起见，这里使用了全局对象。
```
Request class 与 Inventory class 的交互逻辑很简单：
* 在处理 (process)请求的时候，往g_inventory中添加自己。
* 在析构的时候，从g_inventory 中移除自己。
目前看来，整个程序还是线程安全的。

```cpp
class Request
{
public:
  void process() // __attribute_- ((noinline))
  {
  muduo::MutexLockGuard lock(mutex_);
  g_inventory.add(this);
  // ...
  }

  ~Request() __attribute_- ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    sleep(1);//为了容易复现死锁，这里用了延时
    g_inventory.remove(this);
  }

void print() const _attribute_- ((noinline))
{
  muduo::MutexLockGuard lock(mutex_);
  // ...
}

private:
  mutable muduo::MutexLock mutex_;
};

```


Inventory class还有一个功能是打印全部已知的 Request 对象。
`Inventory::printAll()`里的逻辑单独看是没问题的，但是它有可能引发死锁。
```cpp
void Inventory::printAll() const
{
  muduo::MutexLockGuard lock(mutex_);
  sleep(1); //为了容易复现死锁，这里用了延时
  for (std::set<Request*>::const_iterator it = requests_.begin(); it != requests_.end(); ++it)
  {
    (*it)->print(); // 需要得到request对象的锁
  }
  printf("Inventory::printAll() unlocked\n");
}
```
下面这个程序运行起来发生了死锁:

```cpp
void threadFunc()
{
  Request* req = new Request;
  req->process(); // 获取Request对象的锁
  delete req;   // 获取Inventory对象的锁
}

int main()
{
  muduo::Thread thread(threadFunc);
  thread.start();
  //为了让另一个线程等在前面第 14 行的 sleep()上
  usleep(500 *1000):
  g_inventory.printAll();   // 获取Inventory对象的锁，内部还要获取Request对象的锁
  thread.join();
}
```

注意到两个操作获取锁的顺序相反，导致循环等待
一个线程正在析构Request()，另一个线程正在调用Print()

解决死锁的办法很简单：
要么把print()移出printAll()的临界区，这可以用2.8介绍的办法;
要么把remove()移出~Request()的临界区，比如交换p.37中C13和C15两行代码的位置。
```cpp

  ~Request() __attribute_- ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    sleep(1);//为了容易复现死锁，这里用了延时
    g_inventory.remove(this);
  }

    g_inventory.remove(this);

  muduo::MutexLockGuard lock(mutex_);
```

当然这没有解决对象析构的race condition，留给读者当做练习吧。
思考:Inventory::printAll- Request::print 有没有可能与 Request::process Inventory::add 发生死锁?
* Inventory::printAll- Request::print
  获取Inventory对象的锁、获取Request对象的锁
* Request::process Inventory::add
  获取Request对象的锁，获取Inventory对象的锁

死锁会让程序行为失常，其他一些锁使用不当则会影响性能，
例如潘爱民老师写的《Lock Convoys Explained》15 详细解释了一种性能衰退的现象。
除此之外，编写高性能多线程程序至少还要知道 false sharing 和CPU cache 效应，可看这儿篇入门文章



## 2.2 条件变量
`互斥器(mutex)`是加锁原语，用来`排他性地访问共享数据`，它不是等待原语。
在使用mutex的时候，我们一般都会期望：**锁不要阻塞，总是能立刻拿到锁。然后尽快访问数据，用完之后尽快解锁，这样才能不影响并发性和性能**。
如果需要等待某个条件成立，我们应该使用条件变量(condition variable)。
条件变量顾名思义是`一个或多个线程等待某个布尔表达式为真，即等待别的线程“唤醒”它`。
条件变量的学名叫`管程(monitor)`。
Java object 内置的 wait()、notify()、notifyAl1()是条件变量。
条件变量只有一种正确使用的方式，几乎不可能用错。
### 对于`wait端`：
1. 必须与`mutex`一起使用，该布尔表达式的读写需受此mutex保护
2. 在`mutex`已上锁的时候才能调用`wait()`。
3. 把`判断布尔条件`和`wait`放到`while`循环中


写成代码是:
```cpp
muduo::MutexLock mutex;
muduo::Condition cond(mutex);
std::deque<int> queue;
int dequeue()
{
  MutexLockGuard lock(mutex); // 上锁
  while (queue.empty()) // 必须用循环 必须在判断之后再wait()
  {
    cond.wait();  // 这一步会原子地 unlock mutex 并进入等待，不会与 enqueue 死锁
    // wait()执行完毕时会自动重新加锁
  }
  assert(!queue.empty());
  int top = queue.front();
  queue.pop_front();
  return top;
}
```

上面的代码中必须用 while 循环来等待条件变量，而不能用if 语句，原因是`虚假唤醒`。
这也是面试多线程编程的常见考点。

### 对于`signal/broadcast端`
1. 不一定要在mutex已上锁的情况下调用signal(理论上)。
2. 在signal之前一般要修改布尔表达式。
3. 修改布尔表达式通常要用mutex保护(至少用作full memory barrier)。
4. 注意区分signal与broadcast:
  `broadcast`通常用于表明`状态变化`
    通知到所有等待该条件变量的线程：条件已经发生变化，让它们重新检查条件决定是否继续执行
  `signal`通常用于表示`资源可用`
    表示资源可用，通知至少一个线程

  (broadcast should generally be used to indicate state change rather than resource availability。) 
写成代码是：
```cpp
void enqueue(int x)
{
  {
    MutexLockGuard lock(mutex);
    queue.push_back(x);
  }
  cond.notify_one(); //可以移出临界区之外，因为对临界资源的修改已经完成，可以把锁释放了
}
```
上面的`dequeue()/enqueue()`实际上实现了一个简单的容量无限的(unbounded)`BlockingQueue`

思考:
`enqueue()`中每次添加元素都会调用`condition::notify()`，如果改成：只在`queue.size()`从0变1的时候才调用 Condition::notify()，会出现什么后果？
--> 入队线程1进行入队，通知了所有出队线程，入队线程2也进行入队，但不进行通知，某一个出队线程拿到锁，进行出队
假如入队速度很快，就一直堆积了，可能只有一个出队线程一直工作（毕竟notify_one只执行了一次），导致其他的出队线程都饥饿了
假如出队速度很快，出队线程都有机会被唤醒

**条件变量是非常底层的同步原语，很少直接使用**，一般都是用它来实现高层的同步措施，如`BlockingQueue<T>`或`CountDownLatch`。

`倒计时(CountDownLatch)`是一种常用且易用的同步手段。
它主要有两种用途:
1. 主线程发起多个子线程，等这些子线程各自都完成一定的任务之后，主线程才继续执行。
通常用于**主线程等待多个子线程完成初始化**。

2. 主线程发起多个子线程，子线程都等待主线程，**主线程完成其他一些任务之后，通知所有子线程开始执行**。
通常用于**多个子线程等待主线程发出“起跑”命令**。

当然我们可以直接用条件变量来实现以上两种同步。
不过如果用CountDownLatch的话，程序的逻辑更清晰。
CountDownLatch的接口很简单：
```cpp
class CountDownLatch : boost:noncopyable
{
public:
  explicit CountDownLatch(int count);   // 倒数几次
  void wait();                          // 等待计数值变为0
  void countDown();                     // 计数减一

private:
  mutable MutexLock mutex_;
  Condition condition_;
  int count_;
};

// CountDownLatch的实现同样简单，几乎就是条件变量的教科书式应用
//构造函数见第48页
void CountDownLatch::wait()
{
  MutexLockGuard lock(mutex_);
  while (count_ > 0)
    condition_.wait();
}

void CountDownLatch::countDown()
{
  MutexLockGuard lock(mutex_);
  --count_;
  if (count_ == 0)
    condition_.notifyAll();
}
```

注意到`CountDownLatch::countDown()`使用的是`Condition::notifyAll()`，而前面p.41的enqueue()使用的是`Condition::notify_one()`，
这都是有意为之。请读者思考，如果交换两种用法会出现什么情况?
--> broadcast是状态变化，需要通知所有的signal端，wait端的状态发生了变化
    signal是资源可用，每次唤醒一个线程即可


`互斥器`和`条件变量`构成了多线程编程的全部必备同步原语，用它们即可完成任何多线程同步任务，二者不能相互替代。
我认为应该精通这两个同步原语的用法，先学会编写正确的、安全的多线程程序，再在必要的时候考虑用其他“高技术”手段提高性能，如果确实能提高性能的话。
千万不要连 mutex 都还没学会、用好，一上来就考虑lock-free设计。


## 2.3 不要用读写锁和信号量
`读写锁(Readers-Writer lock，简写为rwlock)`是个看上去很美的抽象，它明确区分了read和write两种行为。
初学者常干的一件事情是，一见到某个共享数据结构频繁读而很少写，就把mutex替换为rwlock。
甚至首选rwlock来保护共享状态，这不见得是正确的。
从正确性方面来说，**一种典型的易犯错误是在持有 read lock 的时候修改了共享数据**。
这通常发生在程序的维护阶段，为了新增功能，程序员不小心在原来readlock 保护的函数中调用了会修改状态的函数。
这种错误的后果跟无保护并发读写共享数据是一样的。
**从性能方面来说，读写锁不见得比普通mutex更高效**。
无论如何reader lock加锁的开销不会比mutex lock小，因为它要`更新当前reader的数目`。
如果临界区很小，锁竞争不激烈，那么mutex 往往会更快。见1.9 的例子。
reader lock可能允许提升(upgrade)为writer lock，也可能不允许提升。
考虑2.1.1的 post()和traverse()示例，如果用读写锁来保护 foos 对象，那么post()应该持有写锁，而traverse()应该持有读锁。
```cpp
  MutexLock mutex;
  std::vector<Foo> foos;

  void post(const Foo& f)
  {
    MutexLockGuard lock(mutex);
    foos.push_back(f);
  }

  void traverse()
  {
    MutexLockGuard lock(mutex);
    for (std::vector<Foo>::const_iterator it = foos.begin(); it != foos.end(); ++it)
    {
      it->doit();
    }
  }

  // 假设traverse()的doit调用到post()
```
如果允许把读锁提升为写锁，后果跟使用recursive mutex一样，会造成迭代器失效（递归锁会导致迭代器失效），程序崩溃。（在使用读写锁时，可以先获取读锁，然后在需要写操作时，尝试升级为写锁。）
如果不允许提升，后果跟使用non-recursive mutex一样，会造成死锁（重复获取锁）。
我宁愿程序死锁，留个“全尸”好查验。


通常**reader lock 是可重入的**，**writer lock 是不可重入的**。
但是为了防止writer饥饿，writer lock通常会阻塞后来的reader lock，因此reader lock在重入的时候可能死锁。？？？
--> 补充一下rwlock死锁的问题，线程1获取了读锁，在临界区执行代码；这时，线程2获取写锁，在该锁上等待线程1完成读操作，同事线程2阻塞了后续的读操作；线程1仍在进行剩余读操作，但是它通过函数调用等间接方式，再次获取那个读锁，此时，线程1阻塞，因为线程2已经上了写锁；同时，线程2也在等待线程1释放读锁，才能进行写操作。因此发生了死锁，原因就在于，读锁是可重入的。





另外，在追求低延迟读取的场合也不适用读写锁，见 p.55 muduo线程库有意不提供读写锁的封装，因为我还没有在工作中遇到过用rwlock 替换普通mutex 会显著提高性能的例子。相反，我们一般建议首选mutex。
遇到并发读写，如果条件合适，我通常会用2.8的办法，而不用读写锁，同时避免reader 被 writer 阻塞。
如果确实对并发读写有极高的性能要求，可以考虑`read-copy-update信号量(Semaphore)`:
我没有遇到过需要使用信号量的情况，无从谈及个人经验。
我认为信号量不是必备的同步原语，因为条件变量配合互斥器可以完全替代其功能，而且更不易用错。
除了[RWC]指出的“semaphore has no notion ofownership”之外，信号量的另一个问题在于它有自己的计数值，而通常我们自己的数据结构也有长度值，这就造成了**同样的信息存了两份**，**需要时刻保持一致**，这增加了程序员的负担和出错的可能。如果要控制并发度，可以考虑用`muduo::ThreadPool`。

说一句不知天高地厚的话，如果程序里需要解决如`“哲学家就餐”`之类的复杂IPC问题，我认为应该首先检讨这个设计：
为什么线程之间会有如此复杂的资源争抢(一个线程要同时抢到两个资源，一个资源可以被两个线程争夺)？
如果在工作中遇到，我会把“想吃饭”这个事情专门交给一个为各位哲学家分派餐具的线程来做，然后每个哲学家等在一个简单的condition variable 上，到时间了有人通知他去吃饭。
**从哲学上说，教科书上的解决方案是平权，每个哲学家有自己的线程，自己去拿筷子;我宁愿用集权的方式，用一个线程专门管餐具的分配，让其他哲学家线程拿个号等在食堂门口好了**。
这样不损失多少效率，却让程序简单很多。
虽然Windows的WaitForMultipleobjects 让这个问题trivial化，但在Linux下正确模拟WaitForMultipleObjects 不是普通程序员该干的。
Pthreads还提供了barrier这个同步原语，我认为不如CountDownLatch实用



## 2.4 封装`MutexLock`、`MutexLockGuard`、`Condition`
这几个类都不允许拷贝构造/赋值

其实就是封装`pthread系列接口`

`MutexLock`
```cpp
class CAPABILITY("mutex") MutexLock : noncopyable
{
 public:
  MutexLock()
    : holder_(0)
  {
    MCHECK(pthread_mutex_init(&mutex_, NULL));
  }

  ~MutexLock()
  {
    assert(holder_ == 0);
    MCHECK(pthread_mutex_destroy(&mutex_));
  }

  // must be called when locked, i.e. for assertion
  bool isLockedByThisThread() const   // ！！！可用于assert
  {
    return holder_ == CurrentThread::tid();
  }

  void assertLocked() const ASSERT_CAPABILITY(this)
  {
    assert(isLockedByThisThread());
  }

  // internal usage

  void lock() ACQUIRE()
  {
    MCHECK(pthread_mutex_lock(&mutex_));
    assignHolder();
  }

  void unlock() RELEASE()
  {
    unassignHolder();
    MCHECK(pthread_mutex_unlock(&mutex_));
  }

  pthread_mutex_t* getPthreadMutex() /* non-const */
  { // 用户代码不会调用此处，仅供Condition调用
    return &mutex_;
  }

 private:
  friend class Condition;

  class UnassignGuard : noncopyable   // 辅助类，用于在构造时解除MutexLock对象的持有者，析构时重新分配持有者，确保在作用域内正确管理MutexLock对象的持有者
  {
   public:
    explicit UnassignGuard(MutexLock& owner)
      : owner_(owner)
    {
      owner_.unassignHolder();
    }

    ~UnassignGuard()
    {
      owner_.assignHolder();
    }

   private:
    MutexLock& owner_;
  };

  void unassignHolder()
  {
    holder_ = 0;
  }

  void assignHolder()
  {
    holder_ = CurrentThread::tid();
  }

  pthread_mutex_t mutex_;
  pid_t holder_;
};
```


`MutexLockGuard`，直接封装`MutexLock`，无法中途加解锁，生命周期和锁的有效区域绑定
```cpp
class MutexLockGuard : boost::noncopyable
{
 public:
  explicit MutexLockGuard(MutexLock& mutex) : mutex_(mutex)
  {
    mutex_.lock();
  }

  ~MutexLockGuard()
  {
    mutex_.unlock();
  }

 private:

  MutexLock& mutex_;
};

#define MutexLockGuard(x) static_assert(false "missing mutex guard var name")
```

注意上面代码的最后一行定义了一个宏，这个宏的作用是防止程序里出现如下错误:
```cpp
void doit()
{
  MutexLockGuard(mutex);// 遗漏变量名，产生一个临时对象又马上销毁了
  // 结果没有锁住临界区。
  // 正确写法是：
  MutexLockGuard lock(mutex);
  //临界区...
}
```
我见过有人把MutexLockGuard 写成template，
我没有这么做是因为**它的模板类型参数只有 MutexLock 一种可能，没有必要随意增加灵活性**，
于是我手工把模板具现化(instantiate)了。
此外一种更激进的写法是，把lock/unlock 放到 private 区（不让外部加解锁），然后把MutexLockGuard设为 MutexLock 的friend。
我认为在注释里告知程序员即可，另外check-in（释放所有权）之前的code review 也很容易发现误用的情况 (grep getPthreadMutex)。


这段代码没有达到工业强度:
* mutex创建为`PTHREAD_MUTEX_DEFAULT`类型，而不是我们预想的`PTHREAD_MUTEX_NORMAL`类型(实际上这二者很可能是等同的)，
严格的做法是用mutexattr来显式指定mutex的类型。
* 没有检查返回值。
这里不能用 `assert()`检查(pthread_mutex_lock等操作的)返回值，因为`assert()`在release build 里是空语句。
我们检查返回值的意义在于：防止ENOMEM之类的资源不足情况，这一般只可能在负载很重的产品程序中出现。
一旦出现这种错误，程序必须立刻清理现场并主动退出，否则会莫名其妙地崩溃，给事后调查造成困难。
这里我们需要 non-debug 的assert，或许 google-glog 的 CHECK()宏是个不错的思路。
PS: 此处摘抄的代码进行了返回值的检查
```cpp
#define MCHECK(ret) ({ __typeof__ (ret) errnum = (ret);         \
                       assert(errnum == 0); (void) errnum;})

#endif // CHECK_PTHREAD_RETURN_VALUE
```

以上两点改进留作练习。

muduo库的一个特点是只提供最常用、最基本的功能，特别有意避免提供多种功能近似的选择。
muduo不是“杂货铺”，不会不分青红皂白地把各种有用的、没用的功能全铺开摆出来。muduo 删繁就简，举重若轻;减少选择余地，生活更简单。

MutexLock 没有提供`trylock()`函数，它可能用于观察多线程竞争同一个锁的情况。
Condition class 的实现有点意思。
`Pthreads condition variable`允许在`wait()`的时候指定mutex，但是我想不出有什么理由一个condition variable会和不同的mutex 配合使用。
Java的intrinsic condition和Condition class 都不支持这么做，因此我觉得可以放弃这一灵活性，老老实实地一对一好了。

相反，boost::thread的condition_variable 是在 wait 的时候指定mutex，请参观其同步原语的庞杂设计:
* Concept有四种 Lockable、TimedLockable、SharedLockable、UpgradeLockable。
* Lock 有六种: lock_guard、unique_lock、shared_lock、upgrade_lock、up grade_to_unique_lock、scoped_try_lock。
* Mutex 有七种: mutex、try_mutex、timed_mutex、recursive_mutex、recursive_try_mutex、recursive_timed_mutex、shared_mutex。
恕我愚钝，见到 boost::thread 这样如 Rube Goldberg Machine一样让人眼花缭乱的库，我只得三揖绕道而行。
很不幸 C++11的线程库也采纳了这套方案。这些class名字也很无厘头，为什么不老老实实用 readers_writer_lock 这样的通俗名字呢?非得增加精神负担，自己发明新名字。我不愿为这样的灵活性付出代价，宁愿自
已做个简简单单的一看就明白的 class 来用，这种简单的几行代码的“轮子”造造也无妨。提供灵活性固然是本事，然而在不需要灵活性的地方把代码写死，更需要大智慧。
下面这个muduo::Condition class 简单地封装了 Pthreads condition variable，用起来也容易，见本节前面的例子。这里我用notify/notifyA11作为函数名，因为signal有别的含义，C++里的signal/slot、C里的signalhandler 等等。就别overload这个术语了。

`Condition`，直接和`MutexLock`绑定
```cpp
class Condition : noncopyable
{
 public:
  explicit Condition(MutexLock& mutex)
    : mutex_(mutex)
  {
    MCHECK(pthread_cond_init(&pcond_, NULL));
  }

  ~Condition()
  {
    MCHECK(pthread_cond_destroy(&pcond_));
  }

  void wait()
  {
    MutexLock::UnassignGuard ug(mutex_);
    MCHECK(pthread_cond_wait(&pcond_, mutex_.getPthreadMutex())); // 使用到mutex，释放锁并且阻塞等待
  }

  // returns true if time out, false otherwise.
  bool waitForSeconds(double seconds);

  void notify()
  {
    MCHECK(pthread_cond_signal(&pcond_)); // 唤醒一个阻塞等待当前条件变量的线程
  }

  void notifyAll()
  {
    MCHECK(pthread_cond_broadcast(&pcond_)); // 唤醒所有阻塞等待当前条件变量的线程
  }

 private:
  MutexLock& mutex_;
  pthread_cond_t pcond_;
};
```

如果一个 class 要包含 MutexLock 和 Condition，请注意它们的声明顺序和初始化顺序，**mutex_应先于condition_构造，并作为后者的构造参数**:
```cpp
class CountDownLatch
{
  public:
  CountDownLatch(int count)
  : mutex_(),
  // 初始化顺序要与成员声明保持一致
  condition_(mutex_),
  count_(count)
  {}

  private:
  mutable MutexLock mutex_; // 顺序很重要，先 mutex后 condition
  Condition condition_;
  int count_;
};
```

请允许我再次强调，虽然本章花了大量篇幅介绍如何正确使用 mutex 和 condition variable，但并不代表我鼓励到处使用它们。
这两者都是非常底层的同步原语，主要用来实现更高级的并发编程工具。**一个多线程程序里如果大量使用 mutex 和condition variable来同步，基本跟用铅笔刀锯大树(孟岩语)没啥区别**。

在序里使用 Pthreads 库有一个额外的好处:分析工具认得它们，懂得其语意线程分析工具如IntelThread Checker 和 Valgrind-Helgrind34等能识别 Pthreads 调用，
并依据happens-before 关系，分析序有无data race。



## 2.5 线程安全的Singleton实现

研究Singleton的线程安全实现的历史会发现很多有意思的事情，人们一度认为`double checked locking(缩写为DCL)`是王道，兼顾了效率与正确性。
---
### PS：`double checked locking`
`传统的单例模式`
```cpp
class Singleton
{
private:
    Singleton(){}
public:
    static Singleton* instance()
    {
        if(_instance == 0)
        {
            _instance = new Singleton();
        }
 
        return _instance;
    }
private:
    static Singleton* _instance;
 
public:
    int atestvalue;
};
 
Singleton* Singleton::_instance = 0;
```
这样的代码可能存在问题：
1. 线程A执行到`if(_instance == 0)`，尚未创建对象，就挂起了
2. 线程B进来，发现尚未创建对象，主动创建对象，但是如果此处线程A也继续执行的话，它也会创建对象

解决办法之一是，每次函数调用都请求锁：
```cpp
Singleton* Singleton::instance() {
    Lock lock; // acquire lock (params omitted for simplicity)
    if (_instance == 0) {
        _instance = new Singleton;
    }
    return _instance;
} // release lock (via Lock destructor)
```
但是这样显然不必，只要第一次使用上锁就行了
可以用`DCLP`解决
`Double-Checked Locking Pattern`，双重检测
```cpp
Singleton* Singleton::instance() {
    if (_instance == 0) { // 1st test
        Lock lock;
        if (_instance == 0) { // 2nd test
            _instance = new Singleton;  // 1. 分配内存空间 2. 构造对象 3. 指针赋值
        }
    }
    return _instance;
}
```

注意！
其中：`_instance = new Singleton;`需要分为以下三步：
1. 分配内存空间
2. 构造对象
3. 指针赋值
**编译器并不是严格按照上面的顺序来执行的。可以交换2和3**（因为只要分配完内存空间，就可以进行指针赋值了）
考虑以下情况：
1. 线程A执行了， 1，3然后挂起，也就是说，尚未构造对象，就返回了一个空悬指针
2. 线程B检查发现指针不为空，直接`return _instance`

解决办法之一是`volatile`关键字
`volatile`告诉编译器不要对变量的读取和写入进行优化

有“神牛”指出由于`乱序执行`的影响，DCL是靠不住的。
Java开发者还算幸运，可以借助内部静态类的装载来实现。
C++ 就比较惨，要么次次锁，要么`eager initialize`(**指的是在程序运行时立即初始化对象，而不是延迟到对象首次被使用时再初始化**)，
或者动用`memory barrier`这样的“大杀器”。

接下来Java5修订了内存模型，并给 volatile 赋予了acquire/release 语义，这下DCL (with volatile)又是安全的了。
然而C++的内存模型还在修订中41，C++的volatile目前还不能(将来也难说)保证DCL的正确性(只在VisualC++2005及以上版本有效)。
其实没那么麻烦，在实践中用`pthread_once`就行:
```cpp
template<typename T>
class Singleton : boost::noncopyable
{
public:
  static T& instance()
  {
    pthread_once(&ponce_，&Singleton::init);  // 关键调用，确保只调用一次
    return *value_;
  }
private:
  Singleton();
  ~Singleton();
static void init()
{
  value_ = new T();
}

private:
  static pthread_once_t ponce_;
  static T* value_;
};

template<typename T>
pthread_once_t Singleton<T>::ponce_ = PTHREAD_ONCE_INIT;

template<typename T>
T* Singleton<T>::value_ = NULL;
```

上面这个Singleton没有任何花哨的技巧，它用`pthread_once_t`来保证`lazy-initialization`(首次调用才初始化)的线程安全。
线程安全性由Pthreads库保证，如果系统的Pthreads 库有bug，那就认命吧，多线程程序反正也不可能正确执行了
使用方法也很简单:
```cpp
Foo& foo = Singleton<Foo>::instance();
```
这个 Singleton 没有考虑对象的销毁。
在长时间运行的服务器程序里，这不是一个问题，反正进程也不打算正常退出(9.2.2)。
在短期运行的程序中，**程序退出的时候自然就释放所有资源了**(前提是程序里不使用不能由操作系统自动关闭的资源，比如跨进程的 mutex)。
在实际的`muduo::Singleton class`中，通过`atexit(3)`（C标准库函数，当程序通过exit()退出时，调用注册的清理函数，释放资源，关闭文件）提供了销毁功能，聊胜于无罢了。
另外，这个Singleton 只能调用默认构造函数，如果用户想要指定T的构造方式我们可以用`模板特化(template specialization)`*(特化出模板的一个版本来特殊处理)技术来提供一个定制点，这需要引入另一层间接(another level ofindirection)。


## 2.6 sleep(3)不是同步原语
我认为`sleep()/usleep()/nanosleep()`只能出现在测试代码中，比如`写单元测试`的时候;
或者用于`有意延长临界区，加速复现死锁`的情况，就像 S2.1.2示范的那样。
**sleep 不具备memory barrier语义，它不能保证内存的可见性**，见Page 84的例子
**生产代码中线程的等待可分为两种**:
* 一种是`等待资源可用`
(要么等在`select/ poll/epoll_wait`上，要么等在`条件变量`上);

* 一种是等着进入`临界区`(等在mutex上)以便读写共享数据。
后一种等待通常极短，否则程序性能和伸缩性就会有问题。

在程序的正常执行中，如果需要等待一段已知的时间，应该往event loop 里注册一个`timer`，然后在 timer 的回调函数里接着干活，因为**线程是个珍贵的共享资源，不能轻易浪费(阻塞也是浪费)**。
如果等待某个事件发生，那么应该采用条件变量或IO事件回调，不能用 sleep 来轮询。
不要使用下面这种业余做法:
```cpp
while (true) {
if (!dataAvailable)
  sleep(some_time);
else
  consumeData();
}
```

如果多线程的安全性和效率要靠代码主动调用 sleep 来保证，这显然是设计出了问题。
**等待某个事件发生，正确的做法是用 select()等价物或Condition，抑或(更理想地)高层同步工具**;
**在用户态做轮询(polling)是低效的**。


## 2.7 归纳与总结
前面几节内容归纳如下:
* 线程同步的四项原则，尽量用`高层同步设施(线程池、队列、倒计时)`;
* 使用`普通互斥器`和`条件变量`完成剩余的同步任务，采用`RAII惯用手法(idiom)`和`Scoped Locking`。

用好这几样东西，基本上就能应付多线程服务端开发的各种场合。
或许有人会觉得性能没有发挥到极致。我认为，应该先把程序写正确(并尽量保持清晰和简单)，然后再考虑性能优化，如果确实还有必要优化的话。
这在多线程下仍然成立。让一个正确的程序变快，远比“让一个快的程序变正确”容易得多。

在现代的多核计算背景下，多线程是不可避免的。
尽管在一定程度上可以通过framework来屏蔽，让你感觉像是在写单线程程序，比如Java Servlet。了解under the hood 发生了什么对于编写这种程序也会有帮助。
多线程编程是一项重要的个人技能，不能因为它难就本能地排斥，现在的软件开发比起 10年、20年前已经难了不知道多少倍。掌握多线程编程，才能更理智地选择用还是不用多线程，因为你能预估多线程实现的难度与收益，在一开始做出正确的选择。
要知道把一个单线程程序改成多线程的，往往比从头实现一个多线程的程序更困难。
要明白多线程编程中哪些是能做的，哪里是一般程序员应该避开的雷区。

掌握同步原语和它们的适用场合是多线程编程的基本功。
以我的经验，熟练使用文中提到的同步原语，就能比较容易地编写线程安全的程序。本文没有考虑 signal对多线程编程的影响(S4.10)，Unix的 signal在多线程下的行为比较复杂，一般要靠底层的网络库(如Reactor)加以屏蔽，避免干扰上层应用程序的开发

通篇来看，“效率”并不是我的主要考虑点，我提倡正确加锁而不是自己编写lock-free 算法(使用原子整数除外)，更不要想当然地自己发明同步设施。
在没有实测数据支持的情况下，妄谈哪种做法效率更高是靠不住的，不能听信传言或凭感觉“优化”。
很多人误认为用锁会让程序变慢，**其实真正影响性能的不是锁，而是锁争用(lock contention)**。
在程序的复杂度和性能之前取得平衡，并考虑未来两三年扩容的可能(无论是CPU 变快、核数变多，还是机器数量增加、网络升级)。
我认为在分布式系统中，多机伸缩性(scale out)比单机的性能优化更值得投入精力。
本章内容记录了我目前对多线程编程的理解，用文中介绍的手法，我能化繁为简，编写容易验证其正确性的多线程程序，解决自己面临的全部多线程编程任务。
如果本章的观点与你的经验不合，比如你使用了我没有推荐使用的技术或手法(共享内存、信号量等等)，只要你理由充分，但行无妨。



## 2.8 借`shared_ptr`实现`copy-on-write`
本节解决2.1的几个未决问题
* 2.1.1 `post()、traverse()死锁`
* 2.1.2 `把Request::print()移出Inventory::printAll() 临界区`
* 2.1.2 `解决Request 对象析构的race condition`
然后再示范用普通 mutex 替换读写锁。
解决办法都基于同一个思路，那就是用shared_ptr来管理共享数据。
原理如下:
* shared_ptr是引用计数型智能指针，如果当前只有一个观察者，那么引用计数的值为1。
* 对于 write 端，如果发现引用计数为1，说明就自己在使用，这时可以安全地修改共享对象，不必担心有人正在读它。
* 对于 read 端，在读之前把引用计数加1，读完之后减1，这样保证在读的期间其引用计数大于 1，可以阻止并发写。
* 比较难的是，对于 write 端，如果发现用计数大于1，该如何处理？


先来看一个简单的例子，解决2.1.1中的post()和traverse()死锁。
PS: `前情提要`
```cpp
MutexLock mutex;
std::vector<Foo> foos;

void post(const Foo& f)   // 写操作
{
  MutexLockGuard lock(mutex);
  foos.push_back(f);
}

void traverse() // 读操作
{
  MutexLockGuard lock(mutex);
  for ( std::vector<Foo>::const_iterator it = foos.begin(); it != foos.end(); ++it )
  {
    it->doit();   // 如果在doit()中调用post，对于非递归锁会导致死锁以及迭代器失效
  }
}
```
数据结构改成:
```cpp
typedef std::vector<Foo> FooList;
typedef boost::shared_ptr<FooList> FooListPtr;
MutexLock mutex;
FooListPtr g_foos;
```

### 在read端
用一个栈上局部`FooListPtr`变量当做`“观察者”`，它使得g_foos的引用计数增加。
临界区内只读了一次共享变量g_foos(这里多线程并发读写 shared_ptr，因此必须用mutex 保护)，
比原来的写法大为缩短。
而且多个线程同时调用traverse()也不会相互阻塞。
`遍历操作没有加锁！所以其中的doit()调用post()不会发生死锁！？？？但是这操作不会导致迭代器失效吗？--> 如果要在doit里做写操作，就假设它不会导致迭代器失效，可能不一定是push_back，而是修改某个元素`
```cpp
void traverse()
{
  FooListPtr foos;    // 指向vector<Foo>的指针
  {
    MutexLockGuard lock(mutex);
    foos = g_foos;    // 拷贝shared_ptr，实现：只要有人读取了，引用计数就加1
    assert(!g_foos.unique());   // 判断是否是资源的唯一拥有者, assert中返回false则终止，因为拷贝了share_ptr，所以肯定不是唯一拥有者
  }

  // assert(!foos.unique()); 这个断言不成立
  for ( std::vector<Foo>::const_iterator it = foos->begin(); it != foos->end(); ++it )
  {
    it->doit();
  }
}
```

### write端的`post()`
按照前面的描述，如果`g_foos.unique()`为 true，
我们可以放心地在原地 (in-place)修改 FooList。
如果g_foos.unique()为false，说明这时别的线程正在读取FooList，我们**不能原地修改，而是复制一份，在副本上修改**。这样就避免了死锁。
```cpp
void post(const Foo& f)
{
  printf("post \n");
  { // ---临界区开始---
    MutexLockGuard lock(mutex);
    if (!g_foos.unique()) // 说明别的线程正在读，我们不能原地修改，而是在副本上修改，避免死锁
    {
      g_foos.reset(new FooList(*g_foos));   // 最为核心！！！这样就不影响原来对象的读取，因为至少还有两个shared_ptr实例，不会调用析构函数，等到读操作结束，shared_ptr析构。那么读时的对象也被析构了
      printf("copy the whole list \n");  //练习:将这话移出临界区
    }
    assert(g_foos.unique());  // 毕竟reset了，新的指针必然是不会增加引用计数的
    g_foos->push_back(f); // 在副本上进行修改
  } // ---临界区结束---
}
```
当读操作`traverse`做完了，那么其`shared_ptr`指向的对象也就析构了，内存释放了


以下是几种错误写法：
(在`traverse()`使用上述实现的前提下：)
```cpp
//错误一:直接修改 g_foos 所指的 FooList，     因为对应的读操作没加锁，所以会导致读写不一致
void post(const Foo& f)
{
  MutexLockGuard lock(mutex);
  g_foos->push_back(f);
}

// 错误二:试图缩小临界区，把 copying 移出临界区       多写者则会出错，线程1，2同时在副本上push_back了，那么二者就不同步了
void post(const Foo& f)
{
  FooListPtr newFoos(new FooList(*g_foos));
  newFoos->push_back(f);  // traverse()里的doit，可能调用post
  {
    MutexLockGuard lock(mutex);
    g_foos = newFoos;// 或者 g_foos.swap(newFoos);
  }   // traverse()的for循环doit()操作没加锁，可能会有数据出错
}

// 错误三:把临界区拆成两个小的，把 copying 放到临界区之外     和错误二同样的问题
void post(const Foo& f)
{
  FooListPtr oldFoos;
  {
    MutexLockGuard lock(mutex);
    oldFoos = g_foos;
  }
  FooListPtr newFoos(new FooList(*oldFoos));
  newFoos->push_back(f);
  {
    MutexLockGuard lock(mutex);
    g_foos = newFoos;//或者 g_foos.swap(newFoos);
  }   // 不能直接操作g_foos指向的内存，因为正在traverse()的线程也可能感知到，导致数据出错
}
```


希望读者先吃透上面举的这个例子，再来看如何用相同的思路解决剩下的问题。

### 解决2.1.2
**前情提要**
Inventory记录当前的Request对象，内部维护一个`std::set<Request*> requests_;`对象，对外提供add()，remove()接口，他们都是线程安全的
而Request类对外提供process()接口，把自己添加到add()接口中，而在Request的析构中，把自己从Inventory的set中删除。
printAll()接口可能引起死锁
```cpp

class Inventory
{
public:
  void add(Request* req)
  {
    muduo::MutexLockGuard lock(mutex_);
    requests_.insert(req);
  }

  void remove(Request* req) // -_attribute_- ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    requests_.erase(req);
  }

  void printAll() const;
private:
  mutable muduo::MutexLock mutex_;
  std::set<Request*> requests_;
};

Inventory g_inventory; //为了简单起见，这里使用了全局对象。

void Inventory::printAll() const
{
  muduo::MutexLockGuard lock(mutex_);
  sleep(1); //为了容易复现死锁，这里用了延时
  for (std::set<Request*>::const_iterator it = requests_.begin(); it != requests_.end(); ++it)
  {
    (*it)->print(); // 需要得到request对象的锁
  }
  printf("Inventory::printAll() unlocked\n");
}
```
注意：print()操作是Request中的接口，是要加锁的，因为要考虑是多线程环境，另外有个线程可能在运行这样的函数：
```cpp
void threadFunc()
{
  Request* req = new Request;
  req->process(); // 获取Request对象的锁
  delete req;   // 获取Inventory对象的锁
}
```
这样出现了死锁的经典场景之一：上锁顺序不一致导致死锁


把Request::print()移出Inventory::printAll()临界区有两个做法。
其实很简单，**把requests_复制一份， 在临界区之外遍历这个副本**。
```cpp
void Inventory: : printAll() const
{
  std: :set<Request*> requests;
  {
      muduo::MutexLockGuard lock(mutex_);
      requests = requests_;
  }   // 拷贝需要加锁
  //遍历局部变量requests,调用Request::print()
  // ...
}
```
这么做有一个明显的缺点，它复制了整个std::set中的每个元素，开销可能会比较大。
如果遍历期间没有其他人修改requests_,那么我们可以减小开销，这就引出了第二种做法（当没有线程修改requests_时，做修改不需要拷贝）。

**第二种做法的要点是用`shared_ptr`管理`std::set`，`在遍历的时候先增加引用计数，阻止并发修改`**。
当然`Inventory::add()`和`Inventory::remove()`也要相应修改，采用本节前面post()和traverse()的方案。
完整的代码见recipes/thread/test/RequestInventory_test.cc
`Inventory`持有`boost::shared_ptr<std::set<Request*>>`

`Inventory::add()`和`Inventory::remove()`以及`printAll()`
```cpp
  void add(Request* req)
  {
    muduo::MutexLockGuard lock(mutex_);
    {
      if (!requests_.unique())
      {
        requests_.reset(new RequestList(*requests_));
        printf("Inventory::add() copy the whole list\n");
      }
      assert(requests_.unique());
      requests_->insert(req);
    }
  }

  void remove(Request* req) // __attribute__ ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    {
      if (!requests_.unique())
      {
        requests_.reset(new RequestList(*requests_));
        printf("Inventory::remove() copy the whole list\n");
      }
      assert(requests_.unique());
      requests_->erase(req);
    }
  }

void Inventory::printAll() const
{
  RequestListPtr requests = getData();  // 返回指向set<Request*>的指针，拷贝了一个std::shared_ptr，通过引用计数
  sleep(1);
  for (std::set<Request*>::const_iterator it = requests->begin(); it != requests->end(); ++it)
  {
    (*it)->print();   // 不用加锁，因为
  }
}
```

注意`目前的方案仍然没有解决Request对象析构的race condition`，这点还是留作练习吧。
一种可能的答案见recipes/thread/test/Requestnventory_test2.c。


**在`Request`定义中，增加标志位**
```cpp
class Request : public boost::enable_shared_from_this<Request>
{
 public:
  Request() : x_(0)
  {
  }

  ~Request()
  {
    x_ = -1;
  }

  void cancel() __attribute__ ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    x_ = 1;
    sleep(1);
    printf("cancel()\n");
    g_inventory.remove(shared_from_this());
  }

  void process() // __attribute__ ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    g_inventory.add(shared_from_this());
    // ...
  }

  void print() const __attribute__ ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    // ...
    printf("print Request %p x=%d\n", this, x_);
  }

 private:
  mutable muduo::MutexLock mutex_;
  int x_;
};
```


外层调用：
```cpp
void threadFunc()
{
  RequestPtr req(new Request);
  req->process();
  req->cancel();
}
```





# 第三章 多线程服务器的适用场景与常用编程模型
“`进程` (process)”是操作里最重要的两个概念之一(另一个是`文件`)，粗略地讲，一个进程是“内存中正在运行的程序”。
本书的进程指的是Linux操作系统通过`fork()`调用产生的

每个进程有自已独立的地址空间(address space)，**“在同一个进程”还是“不在同一个进程”是系统功能划分的重要决策点**。
《Erlang 程序设计》[ERL]把“进程比喻为“人”，我觉得十分精当，为我们提供了一个思考的框架每个人有自己的记忆(memory)，人与人通过谈话(消息传递)来交流，
谈话既可以是面谈(同一台服务器)，也可以在电话里谈(不同的服务器，有网络通信)。
面谈和电话谈的区别在于，面谈可以立即知道对方是否死了 (crash, SIGCHLD)，而电话谈只能通过周期性的心跳来判断对方是否还活着。
有了这些比喻，设计分布式系统时可以采取“角色扮演”，团队里的几个人各自扮演一个进程，人的角色由进程的代码决定(管登录的、管消息分发的、管买卖的等等)。
每个人有自己的记忆，但不知道别人的记忆，要想知道别人的看法，只能通过交谈(暂不考虑共享内存这种 IPC)。然后就可以思考:
* 容错 万一有人突然死了
* 扩容 新人中途加进来
* 负载均衡 把甲的活儿挪给乙做
* 退休 甲要修复bug，先别派新任务，等他做完手上的事情就把他重启

“线程”这个概念大概是在1993 年以后才慢慢流行起来的，距今不到20年，比不得有40年光辉历史的Unix操作系统。
线程的出现给Unix添了不少乱，
很多C库函数(`strtok()`、`ctime()`)不是线程安全的，需要重新定义(4.2);
signal 的语意也大为复杂化。
据我所知，最早支持多线程编程的(民用)操作系统是 Slaris 2.2和WindowsNT3.1，它们均发布于1993年。
随后在1995 年，POSIX threads标准确立。
**线程的特点是共享地址空间，从而可以高效地共享数据**。
一台机器上的多个进程能高效地共享代码段(操作系统可以映射为同样的物理内存)，但不能共享数据。
如果多个进程大量共享内存，等于是把多进程程序当成多线程来写，掩耳盗铃。
“多线程”的价值，我认为是为了更好地发挥多核处理器(multi-cores)的效能。
在单核时代，多线程没有多大价值。Alan Cox说过:“A computer is a state machine. Threads are for people who can't program state machines”
 (计算机是一台状态机线程是给那些不能编写状态机程序的人准备的。)
 如果只有一块 CPU、一个执行单元，那么确实如 Alan Cox 所说，按状态机的思路去写程序是最高效的，这正好也是下一节展示的编程模型。


## 3.2 单线程服务器的常用编程模型
高性能的网络程序中，最常用的是`非阻塞式IO + IO复用`模型，即`Reactor模型`
相反，`Boost.Asio`和`Windows I/O Completion Ports`实现了`Proactor 模式`，应用面似乎要窄一些。
PS: `Proactor`模式：异步IO模型

此外，ACE 也实现了 Proactor 模式。
在`non-blocking IO + IO multiplexing`这种模型中，
**程序的基本结构是一个事件循环(event loop)，以事件驱动(event-driven)和事件回调的方式实现业务逻辑**
```cpp
// 代码仅为示意，没有完整考虑各种情况
while (!done)
{
  int timeout_ms = max(1000，getNextTimedCallback());
  int retval = ::poll(fds, nfds, timeout_ms);
  if (retval < 0) {
    // 处理错误，回调用户的 error handler
  }
  else {
    // 处理到期的 timers，回调用户的 timer handler
    if (retval > 0) {
    // 处理IO事件，回调用户的IO event handler
    }
}
}
```

这里`select(2)/poll(2)`有伸缩性方面的不足，Linux下可替换为`epoll(4)`，
其他操作系统也有对应的高性能替代品。

## Reactor模型的优点
Reactor模型的优点很明显，编程不难，效率也不错。
不仅可以用于**读写socket连接的建立**(`connect(2)` / `accept(2)`)
甚至DNS解析都可以用非阻塞方式进行，以提高并发度和吞吐量 (throughput)，
对于IO密集的应用是个不错的选择。

## Reactor模型的缺点
基于事件驱动的编程模型也有其本质的缺点，它要求`事件回调函数必须是非阻塞的`。
对于涉及网络IO的请求响应式协议，它容易割裂业务逻辑，使其散布于多个回调函数之中，相对不容易理解和维护。现代的语言有一些应对方法(例如协程).











## 3.3 多线程服务器的常用编程模型
这方面我能找到的文献不多，大概有这么几种：
1. 每个请求创建一个线程，使用阻塞式IO操作。
  在Java 1.4引入NIO之前，这是Java网络编程的推荐做法。可惜伸缩性不佳。

2. 使用线程池，同样使用阻塞式IO操作。
  与第1种相比，这是提高性能的措施。

3. 使用non-blocking IO + IO multiplexing。
  即Java NIO 的方式。

4. Leader/Follower 等高级模式。
在默认情况下，我会使用第3种，即`non-blocking IO` + `one loop per thread`模式来编写多线程C++网络服务程序。

### 3.3.1 one loop per thread
此种模型下，程序里的`每个IO线程`有一个`event loop`(或者叫`Reactor`)，
用于处理读写和定时事件(无论周期性的还是单次的)，代码框架跟3.2一样。
libev的作者说: 
***One loop per thread is usually a good model. Doing this is almost never wrong, sometimes a better-performance model exists, but it is always a good start.***

这种方式的**好处**是:
* 线程数目基本固定，可以在程序启动的时候设置，不会频繁创建与销毁
* 可以很方便地在线程间调配负载。
* IO事件发生的线程是固定的，同一个 TCP 连接不必考虑事件并发。

Eventloop代表了线程的主循环，需要让哪个线程干活，就把timer或IO channel(如TCP连接)注册到哪个线程的 loop 里即可。

**什么时候单启线程**
对`实时性`有要求的 connection 可以`单独用一个线程`;
`数据量大`的connection可以`独占一个线程`，
并把`数据处理任务`分摊到另几个计算线程中(用`线程池`);
其他要的辅助性connections可以共享一个线程。

对于专业的服务端程序，一般会采用`non-blocking IO + IO multiplexing`，
每个connection/acceptor都会注册到某个event loop上，程序里有多个event loop，
每个线程至多有一个event loop，多线程程序对event loop 提出了更高的要求，那就是“线程安全”。
要允许一个线程往别的线程的 loop 里塞东西，这个loop 必须得是线安全的。
如何实现一个优质的多线程 Reactor？可参考第8章。




### 3.3.2 线程池
不过，对于`没有IO`，而光有`计算任务`的线程，使用event loop 有点浪费，
我会用一种补充方案，即用`blocking queue`实现的任务队列(TaskQueue):
```cpp
typedef boost::function<void()> Functor;
BlockingQueue<Functor> taskQueue; // 线程安全的阻塞队列

void workerThread() {
  while(running) // running 变量是个全局标志
  {
    Functor task = taskQueue.take();  // 空则阻塞
    task(); //在产品代码中需要考虑异常处理
  }
}
```


用这种方式实现线程池特别容易，以下是启动容量(并发数)为N的线程池：
```cpp
  int N = num_of_computing_threads;
  for (int i = 0; i < N; ++ i)
  {
    create_thread(&workerThread); //伪代码:启动线程
  }

  // 使用起来也很简单:
  Foo foo;// Foo 有 calc() 成员函数
  boost::function<void()> task = boost::bind(&Foo::calc，&foo);
  taskQueue.post(task);
```

除了任务队列，还可以用`BlockingQueue<T>`实现数据的生产者消费者队列，
即T是`数据类型`而非`函数对象`，queue的消费者(s)从中拿到数据进行处理。

`BlockingQueue<T>`是多线程编程的利器，它的实现可参照Java util.concurrent里的(ArraylLinked)BlockingQueue。
这份Java代码可读性很高，代码的基本结构和教科书一致(1个mutex，2个condition variables)，健壮性要高得多。
如果不想自己实现，用现成的库更好。
**muduo 里有一个基本的实现，包括无界的 BlockingQueue 和 有界的 BoundedBlockingQueue 两个class**。
有兴趣的读者还可以试试Intel Threading Building Blocks里的concurrent_queue<T>，性能估计会更好。



### 3.3.3 推荐模式
总结起来，我推荐的C++多线程服务端编程模式为:  `one(event) loop per thread + thread pool`。
* event loop(也叫IO loop)用作IOmultiplexing，配合non-blocking IO和定时器。
* thread pool用来做计算，具体可以是`任务队列`或`生产者消费者队列`。
以这种方式写服务器程序，需要一个优质的基于 Reactor 模式的网络库来支撑muduo正是这样的网络库。

程序里具体用**几个loop、线程池的大小等参数需要根据应用来设定**，基本的原则是“`阻抗匹配`”，
使得 CPU 和IO 都能高效地运作，具体的例子见p.80.
此外，程序里或许还有个别执行特殊任务的线程，比如 logging，
这对应用程序来说基本是不可见的，但是在分配资源(CPU 和IO)的时候要算进去，以免高估了系统的容量。




## 3.4 进程间通信只用TCP
Linux下进程间通信(IPC)的方式数不胜数，
光[UNPv2] 列出的就有:
`匿名管道(pipe)、具名管道(FIFO)、POSIX 消息队列、共享内存、信号(signals)`等等，更不必说`Sockets`了。

同步原语(synchronization primitives)也很多:
`如互斥器(mutex)、条件变量(condition variable)、读写锁(reader-writer lock)、文件锁record locking)、信号量(semaphore)`等等。

如何选择呢？
根据我的个人经验，贵精不贵多，认真挑选三四样东西就能完全满足我的工作需要，而且每样我都能用得很熟，不容易犯错。

进程间通信我首选`Sockets` (主要指 TCP，我没有用过 UDP，也不考虑Unix domain 协议)，
其**最大的好处**在于：
可以跨主机，具有伸缩性。
反正都是多进程了，如果一台机器的处理能力不够，很自然地就能用多台机器来处理。
把进程分散到同一局域网的多台机器上，程序改改 host:port 配置就能继续用。
相反，前面列出的其他IPC都不能跨机器，这就限制了延展性。

在编程上，TCPsockets 和 pipe 都是操作`文件描述符`，用来收发字节流，都可以`read/write/fcntl/select/poll`等。
不同的是，TCP是双向的，Linux的pipe是单向的，进程间双向通信还得开两个文件描述符，不方便，
而且**进程要有父子关系才能用 pipe，这些都限制了 pipe 的使用**。
在收发字节流这一通信模型下，没有比 Sockets/TCP 更自然的IPC了。
当然，pipe 也有一个经典应用场景:
  那就是`写Reactor/event loop`时用来`异步唤醒 select (或等价的 poll/epoll_wait)`调用

TCP port由一个进程独占，且**操作系统会自动回收**
(listening port和已建立连接的TCP socket 都是文件描述符，在进程结束时操作系统会关闭所有文件描述符)
这说明，即使程序意外退出，也不会给系统留下垃圾，程序重启之后能比较容易地恢复，而不需要重启操作系统(用跨进程的 mutex 就有这个风险)。
还有一个好处，既然 port 是独占的，那么可以防止程序重复启动，后面那个进程抢不到 port，自然就没法初始化了，避免造成意料之外的结果。

两个进程通过 TCP通信，如果一个崩溃了，操作系统会关闭连接，另一个进程几乎立刻就能感知，可以快速 failover。
当然应用层的心跳也是必不可少的 (9.3)。

并且TCP**可记录、可重现**。可抓包分析性能。TCP还能够**跨语言**。

另外，如果网络库带“连接重试”功能的话，我们可以不要求系统里的进程以特定的顺序启动，任何一个进程都能单独重启。
换句话说，**TCP 连接是可再生的**，连接的任何一方都可以退出再启动，重建连接之后就能继续工作，这对开发牢靠的分布式系统意义重大。

使用TCP这种字节流(bytestream)方式通信，会有`marshal/unmarshal`的开销，这要求我们选用合适的消息格式，准确地说是 wire format，目前我推荐Google Protocol Buffers。
见S9.6关于分布式系统消息格式的讨论。
有人或许会说，具体问题具体分析，如果两个进程在同一台机器，就用共享内存，否则就用TCP，比如MS SQL Server 就同时支持这两种通信方式。
试问，是否值得为那么一点性能提升而让代码的复杂度大大增加呢？
何况TCP的local吞吐量一点都不低，见6.5.1 的测试结果。
* `TCP` 是`字节流`协议，只能`顺序`读取，有`写缓冲`;
* `共享内存`是`消息`协议，a进程填好一块内存让b进程来读，基本是“停等(stop wait)”方式。
**要把这两种方式揉到一个程序里，需要建一个抽象层，封装两种IPC**。
这会带来不透明性，并且增加测试的复杂度。
而且万一通信的某一方崩溃，状态 reconcile 也会比 sockets 麻烦。
 (数据刚写到一半，怎么办?)为我所不取。再说了，你舍得让几万块买来的SOLServer 和其他应用程序分享机器资源吗?
生产环境下的数据库服务器往往是独立的高配置服务器，一般不会同时运行其他占资源的程序。

TCP 本身是个数据流协议，除了直接使用它来通信外，还可以在此之上构建`RPC/HTTP/SOAP`之类的上层通信协议，这超过了本章的范围。
另外，除了点对点的通信之外，应用级的广播协议也是非常有用的，可以方便地构建可观可控的分布式系统，见7.11。



## 3.5 多线程服务器的使用场合
“服务器开发”包罗万象，本书所指的“服务器开发”的含义请见本章开头。
用一句话形容是：**跑在多核机器上的、Linux 用户态的、没有用户界面的、长期运行的、网络应用程序，通常是分布式系统的组成部件**。
开发服务端程序的一个基本任务是`处理并发连接`。
现在`服务端网络编程处理并发连接`主要有两种方式:
* 当“线程”很廉价时，一台机器上可以创建远高于 CPU 数目的“线程”。
这时`一个线程只处理一个TCP 连接(甚至半个)`，通常使用阻塞IO(至少看起来如此)。
例如，Python gevent、Go goroutine、Erlang actor。
**这里的“线程”由语言的runtime自行调度，与操作系统线程不是一回事**。

* 当线程很宝贵时，一台机器上只能创建与CPU数目相当的线程。
这时`一个线程要处理多个TCP连接上的IO`，通常使用非阻塞IO和ultiplexing。
例如libevent、muduo、Netty。
**这是原生线程，能被操作系统的任务调度器看见**。

在处理并发连接的同时，也要充分发挥硬件资源的作用，不能让 CPU 资源闲置

以上列出的库不是每个都能做到这一点。既然本书讨论的是 C++ 编程，那么只考虑后一种方式，
这是在 Linux下使用native语言编写用户态高性能网络序的最成熟的模式。
本节主要讨论的是：**这些“线程”应该属于一个进程(以下模式2)还是分属多个进程(模式3)**。
与前文相同，本节的“进程”指的是`fork(2)`系统调用的产物。
“线程”指的是`pthread_create()`的产物，因此是宝贵的那种原生线程。
而且我指的 Pthreads 是`NPTL(Native POSIX Thread Library)`的，每个线程由`clone(2)`产生，对应一个内核的`task_struct`。
首先，一个由多台机器组成的分布式系统必然是多进程的(字面意义上)，因为进程不能跨 OS 边界。
在这个前提下，我们把目光集中到一台机器，一台拥有至少4个核的普通服务器。

如果要在一台多核机器上提供一种服务或执行一个任务，可用的模式有:
(这里的“模式”不是 pattern，而是 model，不巧它们的中译文是一样的。)
1. 运行一个单线程的进程;
2. 运行一个多线程的进程;
3. 运行多个单线程的进程;
4. 运行多个多线程的进程。

这些模式之间的比较已经是老生常谈，简单地总结如下：
* 模式1是不可伸缩的(scalable)，不能发挥多核机器的计算能力。

* 模式3是目前公认的主流模式。它有以下两种子模式:
  - 3a  简单地把模式1中的进程运行多份 (身份对等的多个单线程的进程)
  - 3b  主进程 + worker 进程，如果必须绑定到一个TCP port，比如httpd + fastcgi

* 模式2是被很多人所鄙视的，认为多线程序难写，而且与模式3相比并没有什么优势。

* 模式4 更是千夫所指，它不但没有结合2和3的优点，反而汇聚了二者的缺点
本文主要想讨论的是模式2和模式 3 的优劣，即**什么时候一个服务器程序应该是多线程的？**
从功能上讲，没有什么是多线程能做到而单线程做不到的，反之亦然，都是状态机嘛(我很高兴看到反例)。
从性能上讲，无论是 IO bound 还是CPU bound 的服务，多线程都没有什么优势。

PS: 
**IO密集型任务 && CPU密集型任务：**
* `IO密集型任务`：
特点：IO密集型任务的特点是其主要时间花费在`等待IO操作`（如磁盘读写、网络通信）的完成上，而不是在CPU计算上。
表现：这种任务的CPU利用率相对较低，因为大部分时间都在等待数据的输入输出，而非进行复杂的计算。
例子：例如，**Web服务器处理请求、文件处理、数据库操作等**通常是IO密集型任务，因为它们大部分时间在等待网络IO或磁盘IO的完成。

* `CPU密集型任务`：
特点：CPU密集型任务是指主要时间花费在进行`CPU计算`的任务。这些任务需要大量的CPU资源来完成复杂的计算或者算法运算。
表现：CPU密集型任务通常会占用较多的CPU时间，因为它们涉及到大量的数学运算、数据处理或者算法执行。
例子：例如，**科学计算、图像处理、加密解密算法、大数据分析等**都是CPU密集型任务，因为它们需要大量的计算来处理数据或执行复杂的算法。


PS: 
**内核态**
一个程序在内核态执行的情况包括以下几种：
- `系统调用`：
  当用户程序需要访问受保护的资源（如文件、网络、设备等）或者执行特权指令时，需要通过系统调用进入内核态。
  例如，打开文件、读写数据、创建进程、分配内存等操作都需要通过系统调用进入内核态执行相应的操作。
- `中断处理`：
  当硬件设备发生中断（如定时器中断、IO设备完成操作的中断等），CPU会暂停当前任务的执行，转入中断处理程序执行，这时候CPU处于内核态。
- `异常处理`：
  当发生异常情况（如除零错误、内存访问错误等），CPU会转入异常处理程序执行，这也是在内核态执行。
- `特权指令执行`：
  特定的CPU指令（如修改页表、切换特权级别等）只能在内核态执行，因为这些指令涉及到对系统资源的管理和保护。
总的来说，程序在需要执行特权操作、访问受保护资源、处理中断或异常时，会进入内核态执行。在内核态执行时，程序可以访问系统的底层资源和硬件，执行特权指令，进行系统管理和保护操作。


**为什么要区分用户态和内核态？**
在CPU的所有指令中，有一些指令是非常危险的，如果错用，将导致整个系统崩溃。
比如：清内存、设置时钟等。如果所有的程序都能使用这些指令，那么你的系统一天死机N回就不足为奇了。
所以，CPU将指令分为`特权指令`和`非特权指令`，对于那些危险的指令，只允许操作系统及其相关模块使用，普通的应用程序只能使用那些不会造成灾难的指令。

**「用户态和内核态切换的开销大」，但是它的开销大在那里呢？**
简单点来说有下面几点：
- 保留用户态现场（上下文、寄存器、用户栈等）
- 复制用户态参数，用户栈切到内核栈，进入内核态
- 额外的检查（因为内核代码对用户不信任）
- 执行内核态代码
- 复制内核态代码执行结果，回到用户态
- 恢复用户态现场（上下文、寄存器、用户栈等）


***Paul E. McKenney 在《Is Parallel Programming Hard, And, If So, What Can You Do About It?》第 3.5 节指出，“As a rough rule of thumb, use the simplest tool that willget thejob done.”***
比方说，使用速率为50MB/s的数据压缩库、在`进程`创建销毁的开销是 `800 us`、`线程`创建销毁的开销是`50s`的前提下，考虑如何执行压缩任务:
* 如果要偶尔压缩`1GB`的文本文件，预计运行时间是`20s`，那么起一个进程去做是合理的，因为进程启动和销毁的开销远远小于实际任务的耗时。
* 如果要经常压缩`500kB`的文本数据，预计运行时间是`10ms`，那么每次都起进程似乎有点浪费了，可以每次单独起一个线程去做。
* 如果要频繁压缩`10kB`的文本数据，预计运行时间是`200s`，那么每次起线程似乎也很浪费，不如直接在当前线程搞定。
也可以用一个线程池，每次把压缩任务交给线程池，避免阻塞当前线程(特别要避免阻塞IO线程)。
由此可见，多线程并不是万灵丹(silver bullet)，它有适用的场合。
那么究竟什么时候该用多线程?在回答这个问题之前，我先谈谈必须用单线程的场合。


### 3.5.1必须用单线程的场合
据我所知，有两种场合必须使用单线程:
1. 程序可能会`fork(2)`;
2. 限制程序的`CPU占用率`。

### 只有单线程程序能 fork(2) ###
`根据后面 4.9 的分析，一个设计为可能调用fork(2)的程序必须是单线程的`，
比如后面S3.5.3 中提到的“看门狗进程”。
多线程序不是不能调用 fork(2)，而是这么做会遇到很多麻烦，我想不出做的理由。

一个程序fork(2)之后一般有两种行为:
1. 立刻执行`exec()`，变身为另一个程序。
例如shell 和inetd;
又比如 lighttpd fork()出子进程，然后运行 fastcgi。
或者集群中运行在计算节点上的负责启动job的守护进程(即我所谓的“看门狗进程”)。

2. 不调用`exec()`，继续运行当前程序。
要么通过共享的文件描述符与父进程通信协同完成任务; 要么接过父进程传来的文件描述符，独立完成工作，例如20世纪80年代的Web 服务器CSA httpd。
这些行为中，我认为只有“看门狗进程”必须坚持单线程，其他的均可替换为多线程程序(从功能上讲)。



### 单线程程序能限制程序的CPU 占用率 ###
这个很容易理解，比如在一个8核的服务器上，一个单线程程序即便发生 busy-wait (无论是因为bug,还是因为overload)占满1个core，其CPU使用率也只有 12.5%
在这种最坏的情况下，系统还是有87.5%的计算资源可供其他服务进程使用。

因此对于一些辅助性的程序，如果它必须和主要服务进程运行在同一台机器的话(比如它要`监控其他服务进程的状态`)，那么`做成单线程的能避免过分抢夺系统的计算资源`。
比方说如果要把生产服务器上的日志文件压缩后备份到 NFS 上，那么应该使用普通单线程压缩工具(gzip/bzip2)。
它们对系统造成的影响较小，在8核服务器上最多占满1个core。
如果有人为了“提高速度”，开启了多线程压缩或者同时起多个进程来压缩多个日志文件，有可能造成的结果是：非关键任务耗尽了 CPU 资源，正常客户的请求响应变慢。这是我们不愿意看到的。

### 3.5.2 单线程程序的优缺点
从编程的角度，单线程程序的优势无须赘言: 简单。
程序的结构一般如3.2所言，是一个基于`IO multiplexing 的event loop`。
或者如云风所言1，直接用阻塞IO。
event loop的典型代码框架见3.2。

Event loop 有一个明显的缺点，它是非抢占的(non-preemptive)。
假设事件a的优先级高于事件 b，处理事件a需要 1ms; 处理事件b需要 10ms。
如果事件b稍早于a发生，那么当事件 a到来时，程序已经离开了`pol1(2)`调用，并开始处理事件b。
事件a 要等上 10ms 才有机会被处理，总的响应时间为 11s。
这等于发生了优先级反转。这个缺点可以用多线程来克服，这也是多线程的主要优势。

#### 多线程程序有性能优势吗
前面我说，无论是IO bound还是CPU bound 的服务，多线都没有什么绝对意义上的性能优势。
这句话是说，如果用很少的 CPU负载就能让IO 跑满，或者用很少的IO流量就能让 CPU 跑满，那么多线程没啥用处。

举例来说:
* 对于静态 Web 服务器，或者 FTP 服务器，CPU的负载较轻，主要瓶颈在磁盘IO和网络IO方面。
这时候往往一个单线程的程序(模式1)就能撑满IO。
用多线程并不能提高吞吐量，因为IO硬件容量已经饱和了。
同理，这时增加CPU数目也不能提高吞吐量。？？？

CPU 跑满的情况比较少见，这里我只好虚构一个例子。
假设有一个服务，它的输入是n个整数，问能否从中选出m个整数，使其和为0(这里n<100, m>0)。
这是著名的subset sum 问题，是NP-Complete 的。对于这样一个“服务”，哪怕很小的n值也会让 CPU算死。
比如n=30，一次的输入不过200字节(32-bit 整数)，CPU的运算时间却能长达几分钟。
对于这种应用，模式3a 是最适合的（即多个单线程的进程），能发挥多核的优势，程序也简单。
也就是说，无论任何一方早早地先到达瓶颈，多线程程序都没啥优势
说到这里，可能已经有读者不耐烦了:你讲了这么多，都在说单线程的好处，那么多线程究竟有什么用?

### 3.5.3 适用多线程程序的场景
我认为多线程的适用场景是: `提高响应速度，让IO和“计算”相互重叠，降低latency`。

**虽然多线程不能提高绝对性能，但能提高平均响应性能**
一个程序要做成`多线程`的，大致要满足:
* 有`多个CPU`可用。
  单核机器上多线程没有性能优势(但或许能简化并发业务逻辑的实现)。
* 线程间有`共享数据`，即内存中的全局状态。
  如果没有共享数据，用模型 3b 就行(即`主进程 + worker进程`)。
  虽然我们应该把线程间的共享数据降到最低，但不代表没有。
* 共享的数据是`可以修改的`，而不是静态的常量表。
  如果数据不能修改，那么可以在进程间用shared memory，模式3就能胜任
* 提供非均质的服务。
  即，`事件的响应`有`优先级`差异，我们可以用专门的线程来处理优先级高的事件 --> 防止优先级反转。
* latency和throughput 同样重要，不是逻辑简单的IO bound或CPU bound程序。
  换言之，程序要有相当的计算量。
* 利用`异步`操作。
  比如logging。
    无论往磁盘写log file，还是往 log server 发送消息都不应该阻塞critical path，写log显然必须多线程。
* 能`scale up`。
  一个好的多线程程序应该能享受增加CPU数目带来的好处，目前主流是8核，很快就会用到16核的机器了。
* 具有`可预测的性能`。
  随着负载增加，性能缓慢下降，超过某个临界点之后会急速下降。线程数目一般不随负载变化。
* 多线程能有效地`划分责任与功能`，让每个线程的逻辑比较简单，`任务单一`，便于编码。
  而不是把所有逻辑都塞到一个event loop 里，不同类别的事件之间相互影响。


---
#### PS: 到底什么是`event loop？`
Java Script就是一种典型，JS是单线程的。
Wikipedia这样定义：
"Event Loop是一个程序结构，用于等待和发送消息和事件。
（a programming construct that waits for and dispatches events or messages in a program.）"
简单说，就是在程序中设置两个线程：
- 一个负责程序本身的运行，称为"`主线程`"；
- 另一个负责主线程与其他进程（主要是各种I/O操作）的通信，被称为"Event Loop线程"（可以译为"`消息线程`"）。

每当遇到`I/O`的时候，主线程就让`Event Loop线程`去`通知相应的I/O程序`，然后接着往后运行，所以不存在红色的等待时间。
等到I/O程序完成操作，Event Loop线程再把结果返回主线程。
主线程就调用事先设定的回调函数，完成整个任务。
可以看到，由于多出了橙色的空闲时间，所以主线程得以运行更多的任务，这就提高了效率。
这种运行方式称为"异步模式"（asynchronous I/O）或"非堵塞模式"（non-blocking mode）。
这正是JavaScript语言的运行方式。
单线程模型虽然对JavaScript构成了很大的限制，但也因此使它具备了其他语言不具备的优势。
如果部署得好，JavaScript程序是不会出现堵塞的，这就是为什么node.js平台可以用很少的资源，应付大流量访问的原因。









***UNIX网络编程***
##### IO
IO (Input/Output，输入/输出)
即`数据的读取（接收）或写入（发送）操作`，
通常用户进程中的一个完整IO分为两阶段：
1. `用户进程空间` <--> `内核空间`
2. `内核空间` <--> `设备空间（磁盘、网络等）`。

IO有`内存IO`、`网络IO`和`磁盘IO`三种，通常我们说的IO指的是后两者。

LINUX中进程无法直接操作I/O设备，其必须通过`系统调用`请求`kernel`来协助完成`I/O`动作；内核会为`每个I/O设备`维护一个`缓冲区`。

对于一个输入操作来说，进程IO系统调用后，内核会`先看缓冲区`中有没有相应的缓存数据：
- 没有的话再到设备中读取，因为设备IO一般速度较慢，需要等待；
- 内核缓冲区有数据则直接复制到进程空间。

所以，对于一个`网络IO`通常包括两个不同阶段：
- 等待网络数据到达网卡 --> 读取到内核缓冲区，数据准备好；
- 从内核缓冲区复制数据到进程空间。

**五大IO模型**
5种IO模型分别是
* 阻塞IO模型
* 非阻塞IO模型
* IO复用模型
* 信号驱动的IO模型
* 异步IO模型
前4种为`同步IO`操作，只有异步IO模型是`异步IO`操作。


###### 阻塞IO模型
进程发起IO系统调用后，进程被阻塞，转到内核空间处理，整个IO处理完毕后返回进程。操作成功则进程获取到数据。

1. 典型应用：阻塞socket、Java BIO；
2. 特点：
- 进程阻塞挂起不消耗CPU资源，及时响应每个操作；
- 实现难度低、开发应用较容易；
- 适用并发量小的网络应用开发；
- 不适用并发量大的应用：因为一个请求IO会阻塞进程，所以，得为每请求分配一个处理进程（线程）以及时响应，系统开销大。

###### 非阻塞IO模型
进程发起IO系统调用后，**如果内核缓冲区没有数据，需要到IO设备中读取，进程返回一个错误而不会被阻塞**；
进程发起IO系统调用后，如果内核缓冲区有数据，内核就会把数据返回进程。

对于上面的阻塞IO模型来说，内核数据没准备好需要进程阻塞的时候，就返回一个错误，以使得进程不被阻塞。

1. 典型应用：socket是非阻塞的方式（设置为NONBLOCK）
2. 特点：
进程`轮询`（重复）调用，消耗CPU的资源；
实现难度低、开发应用相对阻塞IO模式较难；
适用并发量较小、且不需要及时响应的网络应用开发；


###### IO复用模型
多个的进程的IO可以注册到一个`复用器（select）`上，然后用一个进程调用该select， `select`会`监听所有注册进来的IO`；

如果select没有监听的IO在内核缓冲区都没有可读数据，select调用进程会被`阻塞`；而当任一IO在内核缓冲区中有可数据时，select调用就会返回；

而后 select调用进程 可以自己或通知另外的进程（注册进程）来再次发起读取IO，读取内核中准备好的数据。

可以看到，多个进程注册IO后，只有另一个select调用进程被阻塞。
1. 典型应用：`select、poll、epoll`三种方案，nginx都可以选择使用这三个方案;Java NIO;

2. 特点：
专一进程解决多个进程IO的阻塞问题，性能好；Reactor模式;
实现、开发应用难度较大；
适用`高并发`服务应用开发：`一个进程（线程）响应多个请求`；

3. `select、poll、epoll`
Linux中IO复用的实现方式主要有select、poll和epoll：
Select：
  注册IO、阻塞扫描，监听的IO最大连接数不能多于FD_SIZE；
Poll：
  原理和Select相似，没有数量限制，但IO数量大扫描线性性能下降；
Epoll ：
  事件驱动不阻塞，mmap实现内核与用户空间的消息传递，数量很大，Linux2.6后内核支持；


###### 信号驱动IO模型
当进程发起一个IO操作，会向内核注册一个`信号处理函数`，然后进程返回不阻塞；
`当内核数据就绪时`会`发送一个信号`给进程，进程便在`信号处理函数`中调用IO读取数据。

特点：回调机制，实现、开发应用难度大；


###### 异步IO模型
当进程发起一个IO操作，进程`返回（不阻塞）`，但也不能返回结果；
`内核把整个IO处理完后，会通知进程结果`。
如果IO操作成功则进程直接获取到数据。

1. 典型应用：JAVA7 AIO、高性能服务器应用
2. 特点：
不阻塞，数据一步到位；`Proactor模式`；
需要操作系统的底层支持，LINUX 2.5 版本内核首现，2.6 版本产品的内核标准特性；
实现、开发应用难度大；
非常适合高性能高并发应用；


##### `阻塞IO调用`和`非阻塞IO调用`、`阻塞IO模型`和`非阻塞IO模型`
注意这里的阻塞IO调用和非阻塞IO调用不是指阻塞IO模型和非阻塞IO模型：
`阻塞IO调用`：
在用户进程（线程）中调用执行的时候，进程会等待该IO操作，而使得其他操作无法执行。
`非阻塞IO调用`：
在用户进程中调用执行的时候，无论成功与否，该IO操作会`立即返回`，之后进程可以进行其他操作（当然如果是读取到数据，一般就接着进行数据处理）。
这个直接理解就好，进程（线程）IO调用会不会阻塞进程自己。
所以这里两个概念是相对调用进程本身状态来讲的。

`阻塞IO模型`是`一个阻塞IO调用`，
而`非阻塞IO模型`是`多个非阻塞IO调用` + `一个阻塞IO调用`(因应用层代码必然使用轮询)，
因为多个IO检查会立即返回错误，不会阻塞进程。



##### `同步IO`和`异步IO`
`同步IO`：
  导致请求进程阻塞，直到I/O操作完成。
`异步IO`：
  不导致请求进程阻塞。
上面两个定义是《UNIX网络编程 卷1：套接字联网API》给出的。这不是很好理解。
我们来扩展一下，先说说同步和异步，同步和异步关注的是双方的消息通信机制：

同步：双方的动作是经过双方协调的，步调一致的。
异步：双方并不需要协调，都可以随意进行各自的操作。
这里我们的双方是指，`用户进程`和`IO设备`；
明确同步和异步之后，我们在上面网络输入操作例子的基础上，进行扩展定义：
`同步IO`：
  用户进程发出IO调用，去获取IO设备数据，双方的数据要经过`内核缓冲区同步`，完全准备好后，再复制返回到用户进程。
  而复制返回到用户进程会导致请求进程阻塞，直到I/O操作完成。
`异步IO`：
  用户进程发出IO调用，去获取IO设备数据，并不需要同步，`内核直接复制到进程`，整个过程`不导致请求进程阻塞`。


所以，阻塞IO模型、非阻塞IO模型、IO复用模型、信号驱动的IO模型者为同步IO模型，
只有异步IO模型是异步IO(因为，在异步IO模型中，当进程发起一个IO操作，进程`返回（不阻塞）`，但也不能返回结果；`内核把整个IO处理完后，会通知进程结果`。也就是说，异步IO模型借助了异步IO这种特殊的机制)。


"`Event Loop`是一个程序结构，用于等待和发送消息和事件。
（a programming construct that waits for and dispatches events or messages in a program.）"

简单说，就是在程序中设置`两个线程`：
一个负责程序本身的运行，称为`"主线程"`；
另一个负责主线程与其他进程（主要是各种I/O操作）的通信，
被称为"Event Loop线程"（可以译为`"消息线程"`）。
因为总有线程要因为I/O操作要被阻塞，那就让专门的消息线程被阻塞

每当遇到I/O的时候，主线程就让Event Loop线程去通知相应的I/O程序，然后接着往后运行，所以不存在红色的等待时间。
等到I/O程序完成操作，Event Loop线程再把结果返回主线程。
主线程就`调用事先设定的回调函数`，完成整个任务。
（I/O操作完成时，触发某个回调，整个一次任务就算结束了）

---


这些条件比较抽象，这里举两个具体的(虽然是虚构的)例子
假设要管理一个Linux 服务器机群，这个机群里有**8 个计算节点，1个控制节点**。
机器的配置都是一样的，双路四核 CPU，千兆网互联。
现在需要编写一个简单的机群管理软件(参考LLNL的SLURM)，这个软件由3个程序组成:
1. 运行在控制节点上的master，这个程序监视并控制整个机群的状态。
2. 运行在每个计算节点上的slave，负责启动和终止job，并监控本机的资源。
3. 供最终用户使用的client命令行工具，用于提交job。

根据前面的分析
* `slave`是个“看门狗进程”，它会启动别的job进程，因此必须是个`单线程`程序。
另外它不应该占用太多的CPU资源，这也适合单线程模型。
* `master`应该是个模式2的多线程程序:
  它独占一台8核的机器，如果用模型1（单线程的一个进程），等于浪费了87.5%的CPU资源
整个机群的状态应该能完全放在内存中，这些状态是共享且可变的。
如果用模式3，那么**进程之间的状态同步会成大问题**（进程间通信的开销显然比线程间通信更大）。
而如果大量使用共享内存，则等于是掩耳盗铃，是披着多进程外衣的多线程程序。
因为一个进程一旦在临界区内阻塞或crash，其他进程会全部死锁。

**master的主要性能指标**不是throughput，而是`latency`，即尽快地响应各种事件。它几乎不会出现把IO或CPU跑满的情况。

master监控的事件有`优先级`区别，一个程序正常运行结束和异常崩溃的处理优先级不同， 计算节点的磁盘满了 和 机箱温度过高 这两种报警条件的优先级也不同。如果用单线程，则可能会出现优先级反转。

PS：`优先级反转`
假设事件a的优先级高于事件 b，处理事件a需要 1ms; 
处理事件b需要 10ms。
如果事件b稍早于a发生，那么当事件 a到来时，程序已经离开了`pol1(2)`调用，并开始处理事件b。
事件a 要等上 10ms 才有机会被处理，总的响应时间为 11s。
这等于发生了优先级反转。这个缺点可以用多线程来克服，这也是多线程的主要优势。

假设master 和每个slave 之间用一个TCP连接，那么master采用2个或4个IO线程来处理8个TCPconnections 能有效地降低延迟。
master要异步地往本地硬盘写log，这要求logging library 有自己的IO线程，master有可能要读写数据库，那么数据库连接这个第三方 library可能有自己的线程，并回调master的代码。

master要服务于多个clients，用多线程也能降低客户响应时间。
也就是说它可以再用2个IO线程专门处理和clients 的通信。

master还可以提供一个monitor接口，用来广播推送(pushing)机群的状态，这样用户不用主动轮询(polling)。
这个功能如果用单独的线程来做，会比较容易实现，不会搞乱其他主要功能

**master一共开了10个线程:**
* 4个用于和slaves通信的IO线程
* 1个logging 线程。
* 1个数据库IO线程
* 2个和clients 通信的IO线程。
* 1个主线程，用于做些背景工作，比如iob 调度
* 1个pushing线程，用于主动广播机群的状态。
虽然线程数目略多于 core 数目，但是这些线程很多时候都是空闲的，可以依赖OS的进程调度来保证可控的延迟

综上所述，master 用多线程方式编写是自然且高效的。

再举一个 TCP 聊天服务器的例子，这里的“聊天”不完全指人与人聊天，也可能是机器与机器“聊天”。
这种服务的特点是`并发`连接之间有`数据交换`，从一个连接收到的数据要转发给其他多个连接。
因此我们不能按模式3 的做法，把多个连接分到多个进程中分别处理(这会带来复杂的进程间通信)而只能用模式1或者模式2
如果纯粹只有数据交换，那么我想模式1也能工作得很好，因为现在的CPU 足够快，单线程应付几百个连接不在话下。



#### `线程`的`分类`
一个多线程服务程序中的线程大致可分为3类：
1. `IO线程`
  这类线程的主循环是`IO multiplexing`，阻塞地等在`select/pol1/epoll_wait`系统调用上。
这类线程也处理定时事件。
当然它的功能不止IO，有些简单计算也可以放入其中，比如消息的编码或解码。

2.`计算线程`
  这类线程的主循环是`blocking queue`，阻塞地等在condition variable上。这类线程一般位于thread pool中。
  即thread pool的worker线程
  这种线程通常不涉及IO，一般要避免任何阻塞操作。

3.`第三方库所用的线程`
  比如`logging`，又比如database connection。

服务器程序一般不会频繁地启动和终止线程。
甚至，在我写过的程序里，create thread只在程序启动的时候调用，在服务运行期间是不调用的。



## 3.6 答疑
### Linux能同时启动多少个线程?
对于`32-bit`Linux，一个进程的地址空间是4GiB(即2^X, X是CPU位数)，其中用户态能访问3GiB 左右
而一个线程的默认栈(stack)大小是`10MB`，心算可知，一个进程大约最多能同时启动300个线程(3G / 10M)。
如果不改线程的调用栈大小的话，300 左右是上限，因为程序的其他部分(数据段、代码段、堆、动态库等等)同样要占用内存(地址空间)。
对于64-bit系统，线程数目可大大增加，具体数字我没有测试过，因为我在实际项目中一台机器上最多只用到过几十个用户线程，其中大部分还是空闲的。


## 多线程能提高`并发度`吗
如果指的是“并发连接数”，则不能。
由问题1可知，假如单纯采用`thread per connection`的模型，那么并发连接数最多300，这远远低于基于事件的单线程程序所能轻松达到的并发连接数(几千乃至上万，甚至几万)。
所谓“基于事件”指的是用`IO multiplexing event loop`的编程模型，又称`Reactor`模式，在前文中已有介绍。
那么采用前文中推荐的`one loop per thread`呢?至少不逊于单线程程序。
实际上单个event loop处理1万个并发长连接并不罕见，一个multi-loop 的多线程程序应该能轻松支持`5万`并发链接。
小结:thread per connection 不适合高并发场合，其scalability 不佳。
one loop per thread 的并发度足够大，且与 CPU数目成正比。


## 多线程能提高`吞吐量`吗
对于`计算密集型服务`，不能
假设有一个耗时的计算服务，用单线程算需要 0.8s。
在一台8核的机器上，我们可以启动8个线程一起对外服务(如果内存够用，启动8个进程也一样 )。
这样完成单个计算仍然要 0.8s，但是由于这些进程的计算可以同时进行，理想情况下吞吐量可以从单线程的1.25qps (query per second，即平均每秒计算1.25次)上升到10qps。
(实际情况可能要打个八折————如果不是打对折的话。)

假如改用并行算法，用8个核一起算，理论上如果完全并行，加速比高达 8，那么计算时间是 0.1s，吞吐量还是 10qps，但是首次请求的`响应时间`却降低了很多。
实际上根据Amdahls law，即便算法的并行度高达95%，8核的加速比也只有6，计算时间为 0.133s，这样会造成吞吐量下降为 7.5qps。
不过以此为代价，换得响应时间的提升，在有些应用场合也是值得的。

再举一个例子，如果要在一台8核机器上压缩100个1GB的文本文件，每个core的处理能力为200MB/s。
那么`“每次起8个进程，每个进程压缩1个文件”`与`“依次压缩每个文件，每个文件用8个线程并行压缩”`这两种方式的总耗时相当，因为CPU 都是满载的。
但是第2种方式能较快地拿到第一个压缩完的文件，也就是首次响应的延时更小。

如果用thread per request 的模型，每个客户请求用一个线程去处理，那么当并发请求数大于某个临界值T时，吞吐量反而会下降，因为线程多了以后`上下文切换的开销`也随之增加。
thread per request 是最简单的使用线程的方式，编程最容易，简单地把多线程程序当成一堆串行程序，用同步的方式顺序编程，

为了在并发请求数很高时也能保持稳定的吞吐量，我们可以用线程池，线程池的大小应该满足`“阻抗匹配原则”`，见问题 7。

线程池也不是万能的，
如果响应一次请求需要做比较多的计算 (比如计算的时间占整个response time的1/5)那么用线程池是合理的，能简化编程。如果在一次
请求响应中，主要时间是在等待IO，那么为了进一步提高吞吐量，往往要用其他编程模型，比如 Proactor？？？，见问题8。


## 多线程能降低`响应时间`吗？
如果设计合理，充分利用多核资源的话，可以。
在`突发(burst)`请求时效果尤为明显。
### 例1:多线程处理输入
以memcached服务端为例。memcached 一次请求响应 大概可以分为3步:
1. 读取并解析客户端输人;
2. 操作hashtable;
3. 返回客户端。

- 在`单线程模式`下，这3步是串行执行的。
- 在启用`多线程模式`时，它会启用`多个输入线程`(默认是4个)
并在建立连接时按`round-robin(完全平均地轮流分配)`法把新连接分派给其中一个输入线程，这正好是我说的one loop per thread 模型。
这样一来，第1步的操作就能多线程并行，在多核机器上提高多用户的响应速度。
第2步用了全局锁，还是单线程的，这可算是一个值得继续改进的地方。
比如，有两个用户同时发出了请求，这两个用户的连接正好分配在两个IO线程上，
**那么两个请求的第1步操作可以在两个线上并行执行，然后汇总到第2步串行执行，这样总的响应时间比完全串行执行要短一些(`在“读取并解析”所占的比重较大的时候，效果更为明显`，也就是说，要看具体的应用场景)**。

### 例2:多线程分担负载
假设我们要做一个求解数独的服务，这个服务程序在9981端口接受请求，
输入为一行81个数字(待填数字用0表示)，输出为填好之后的81个数字(1~9)，如果无解，输出“NO\r\n”

由于输入格式很简单，用单个线做IO 就行了。
先假设每次求解的计算用时为10ms，用前面的方法计算，单线程程序能达到的吞吐量上限为 100qps;
在8核机器上，如果用线程池来做计算，能达到的吞吐量上限为 800qps。
下面我们看看多线程如何降低响应时间：
假设1个用户在极短的时间内发出了10个请求，如果用单线程“来一个处理一个”的模型，这些 reqs 会排在队列里依次处理(这个队列是操作系统的`TCP缓冲区`，不是程序里自己的任务队列)。
在不考虑网络延迟的情况下，第1个请求的响应时间是10ms;
第2个请求要等第1个算完了才能获得CPU资源，它等了10ms，算了10ms，响应时间是20ms;依此类推，第10个请求的响应时间为100s;这10个请求的平均响应时间为55ms。
* 如果Sudoku服务在每个请求到达时开始计时，会发现每个请求都是10ms 响应时间;
* 而从用户的观点来看，10 个请求的平均响应时间为55ms

下面改用多线程:1个IO线程，8个计算线程(线程池)。
二者之间用`BlockingQueue`沟通。
同样是 10个并发请求，第1个请求被分配到计算线程1，第2个请求被分配到计算线程 2，依此类推，直到第 8 个请求被第8个计算线承担第9和第10号请求会等在 BlockingQueue 里，直到有计算线程回到空闲状态其才能被处理(请注意，这里的分配实际上由操作系统来做，操作系统会从处于 waiting 状态的线程里挑一个，不一定是round-robin 的。)
这样一来，前8个请求的响应时间差不多都是10ms，后2个请求属于第二批，其响应时间大约会是 20ms，总的平均响应时间是12ms。比单线程快多了

由于每道 Sudoku题目的难度不一，对于简单的题目，可能1ms 就能算出来，复杂的题目最多用 10ms。
那么线程池方案的优势就更明显，**线程池能有效地降低“简单任务被复杂任务压住”的出现概率**。
以上举的都是`计算密集`的例子，即`线程在响应一次请求时不会等待IO`。
下面谈谈更复杂的情况。

## 多线程程序如何让IO和“计算”相互重叠，降低latency?
基本思路是，把`IO操作(通常是写操作)`通过`BlockingQueue`交给别的线程去做，自己不必等待。

### 例1:`日志(logging)`
在多线程服务器程序中，日志(logging)至关重要，本例仅考虑写log file 的情况，不考虑log server。
在一次请求响应中，可能要写多条日志消息，而如果用同步的方式写文件(`fprintf或fwrite`)，多半会降低性能，因为:
* 文件操作一般比较慢，服务线程会等在IO上，让CPU闲置，增加响应时间。
* 就算有buffer，还是不灵。
  多个线程一起写，为了不至于把 buffer 写错乱，往往要加锁。
  这会让服务线程(为了获取锁而)互相等待，降低并发度。
  (同时用多个 log 文件不是办法，除非你有多个磁盘，且保证 log files 分散在不同的磁盘上，否则还是要受到磁盘 IO 瓶颈的制约。)

解决办法是：`单独用一个 logging 线程，负责写磁盘文件，通过一个或多个BlockingQueue 对外提供接口`。
别的线要写日志的时候，先把消息(字符串)准备好，然后往 queue 里一塞就行，基本不用等待。
这样`服务线程的计算`就和`logging线程的磁盘IO`相互重叠（即二者没有依赖关系），降低了服务线程的响应时间。

尽管 logging 很重要，但它不是程序的主要逻辑，因此对程序的结构影响越小越好，
最好能简单到如同一条 printf 语句，且不用担心其他性能开销。而一个好的多线程异步 logging 库能帮我们做到这一点，见第5章。(Apache 的 lg4cxx 和 log4i都支持AsyncAppender 这种异步logging方式。)

### 例2:`memcached`客户端 
假设我们用memcached来保存用户最后发帖的时间
那么：每次响应用户发帖的请求时，程序里要去设置一下memcached 里的值。这一步如果用同步IO，会增加延迟
对于“设置一个值”这样的`只写操作`，我们其实不用等memcached 返回操作结果，这里也不用在乎set 操作失败，那么可以借助多线程来降低响应延迟。
比方说我们可以写一个多线程版的memcached 的客户端
对于set操作，调用方只要把 key 和 value 准备好，调用一下`asyncSet()`函数，把数据往`BlockingQueue`(还是在强调阻塞队列，即队列为空时，出队操作阻塞)上一放就能立即返回，延迟很小。
剩下的事就留给memcached 客户端的线程去操心，而服务线程不受阻碍。

其实所有的网络写操作都可以这么异步地做，不过这也有一个缺点，那就是每次`asyncWrite()`都要在线程间传递数据。
其实如果`TCP缓冲区`是空的，我们就可以在本线程写完，不用劳烦专门的IO线程。
Netty 就使用了这个办法来进一步降低延迟

以上都仅讨论了“打一枪就跑”的情况，如果是一问一答，比如从 memcached取一个值，那么“重叠IO”并不能降低响应时间，因为你无论如何要等 memcached的回复。
这时我们可以用别的方式来提高并发度，见问题 8。 (`虽然不能降低响应时间，但也不要浪费线程在空等上`。)

以上的例子也说明，`BlockingQueue`是构建多线程程序的利器。另见S12.8.3。


## 为什么第三方库往往要用自己的线程?
event loop模型没有标准实现。
如果自己写代码，尽可以按所用`Reactor`的推荐方式来编程。
但是第三方库不一定能很好地适应并融人这个event loop framework
有时需要用线程来做一些`串并转换`：
比方说检测串口上的数据到达，可以用文件描述符的可读事件，因此可以方便地融入 event loop。
但是检测串口上的某些控制信号(例如`DCD`)只能用轮询(`ioctl(fd，TIOCMGET，&flags)`)或阻塞等待 (`ioctl(fd,TIOCMIWAIT，TIOCM_CAR)`);
要想融event loop，需要单独起一个线程来查询串口信号翻转，再转换为文件描述符的读写事件(可以通过`ppe(2)`)。
对于Java，这个问题还好办一些，因为 thread pool 在 Java 里有标准实现叫ExecutorService。
如果第三方库支持线程池，那么它可以和主程序共享一个ExecutorService，而不是自己创建一堆线程。(比如在初始化时传入主程序的 obi。)
对于C++，情况麻烦得多，Reactor 和 thread pool都没有标准库。



## 什么是线程池大小的`阻抗匹配原则`？
如果池中线程在执行任务时，`密集计算所占的时间比重`为P ( `0<P<1` )，
而系统一共有`C个CPU`，为了让这 C个CPU 跑满而又不过载，线程池大小的经验公式：`T = C/P`。
T是个hint，考虑到P值的估计不是很准确，T的最佳值可以上下浮动50%.
这个经验公式的原理很简单：
T个线程，每个线程占用P的CPU时间，如果刚好占满C个CPU，那么必有T x P = C。

下面验证一下边界条件的正确性：
- 假设 C = 8，P = 1.0，线程池的任务完全是密集计算，那么T = 8。
只要 8个活动线程就能让8个CPU 饱和，再多也没用，因为CPU资源已经耗光了。
- 假设 C = 8，P = 0.5，线程池的任务有一半是计算，有一半等在IO 上，那么T = 16。
考虑操作系统能灵活、合理地调度`sleeping / writing / running`线程，那么大概16个“50%繁忙的线程”能让8个CPU忙个不停。
**启动更多的线并不能提高吞吐量，反而因为增加上下文切换的开销而降低性能**
如果 P < 0.2，这个公式就不适用了，T可以取一个固定值，比如5 x C。
另外公式里的C不一定是CPU总数，可以是`“分配给这项任务的CPU数目”`，比如在8核机器上分出4个核来做一项任务，那么C=4。



## 除了你推荐的`Reactor + thread poll`，还有别的`non-trivial多线程编程模型`吗?
有，`Proactor`模型(？？？)。
如果 一次请求响应中 要和 `别的进程` 打多次交道，那么Proactor模型往往能做到更高的并发度。
当然，代价是代码变得支离破碎，难以理解。

这里举`HTTP proxy`为例，一次 HTTP proxy的请求如果没有命中本地 cache 那么它多半会:
1. 解析域名(不要小看这一步，对于一个陌生的域名，解析可能要花几秒的时间):
2. 建立连接;
3. 发送HTTP请求;
4. 等待对方回应;
5. 把结果返回给客户

这5步中跟2个server发生了3次round-trip，每次都可能花几百毫秒
1. 向 `DNS` 问域名，等待回复;
2. 向对方的`HTTP服务器`发起连接，等待TCP三路握手完成;
3. 向对方发送`HTTP request`，等待对方 response。
而实际上HTTP proxy 本身的运算量不大，如果用线程池，池中线程的数目会很庞大，不利于操作系统的管理调度。

这时我们有两个解决思路:
1. 把`“域名已解析”`、`“连接已建立”`、`“对方已完成响应”`做成event，继续按照`Reactor`的方式来编程。
  这样一来，**每次客户请求就不能用一个函数从头到尾执行完成，而要分成多个阶段，并且要管理好请求的状态(`“目前到了第几步?”`)**
2. 用回调函数，让系统来把任务串起来。
比如收到用户请求，如果没有命中本地缓存，那么需要执行:
  * 立刻发起 异步的DNS解析`startDNSResolve()`，告诉系统在解析完之后调用`DNSResolved()`函数;
  * 在`DNSResolved()`中，发起`TCP连接请求`，告诉系统在连接建立之后调用`connectionEstablished()`;
  * 在`connectionEstablished()`中发送`HTTP request`，告诉系统在收到响应 之后调用`httpResponsed()`;
  * 最后，在`httpResponsed()`里把结果返回给客户。
（系统调用结合连续的回调）
NET大量采用的BeginInvoke/EndInvoke操作也是这个编程模式。
当然，对于不熟悉这种编程方式的人，代码会显得很难看。
有关 Proactor 模式的例子可参看`Boost.Asio`的文档，这里不再多说。

Proactor模式依赖`操作系统或库`来高效地调度这些子任务，每个子任务都不会阻塞，因此`能用比较少的线程达到很高的IO并发度`。
Proactor`能提高吞吐，但不能降低延迟`，所以我没有深人研究。
另外，在没有语言直接支持的情况下，Proactor 模式让代码非常破碎，在C++ 中使用 Proactor 是很痛苦的。
因此最好在“线程”很廉价的语言中使用这种方式，这时runtime往往会屏蔽细节，程序用`单线程阻塞IO`的方式来`处理TCP连接`。


## 模式2和模式3a该如何取舍?
3.5中提到：
- 模式2是`一个多线程的进程`
- 模式3a是`多个相同的单线程进程`
我认为，在其他条件相同的情况下，可以根据`工作集(work set)的大小`来取舍。
`工作集`是指：`服务程序响应一次请求所访问的内存大小`。
如果`工作集较大`，那么就用`多线程`，避免`CPU cache 换入换出`，影响性能;
否则，就用`单线程多进程`，享受单线程编程的便利。举例来说：
**PS: 关于cache：**
由于：`CPU通用寄存器的速度`与`主存`之间存在这太大的差异。
程序访问的`空间局部性`、`时间局部性`。
**在较短时间间隔内，程序产生的地址往往集中在一个很小范围内**。
比如：一个很大的循环程序段，在一段时间内一直在局部区域执行指令，故`循环内指令`的`时间局部性`好；
一段`连续地址的数据元素`，按顺序遍历故程序`空间局部性`好。

如何将`缓存`和`内存`关联：
CPU在获取指令时，在主存中把一段数据，都搬到Cache中，也就把下一个执行的指令一起搬到Cache中了。
CPU访问`cache`或`主存`时，以`字`为单位；
`Cache和主存` `交换信息`时，以`块`为单位，一次读入一块或多块内容；每`块`由`若干个字`组成；
`Cache`的`每行`都设置有`标记`，`CPU`访问程序或数据时，先访问`标记`。


如果程序有一个较大的本地cache（业务逻辑所需的自用的一块内存），用于缓存一些基础参考数据(in-memory look-up table)，几乎每次请求都会访问 cache，那么多线程更适合一些，因为：
可以避免每个进程都自己保留一份 cache，增加内存使用。
memcached(是一个高性能，分布式内存对象缓存系统)这个内存消耗大户，用多线程服务端就比在同一台机器上运行多个memcached instance要好。 
(但是如果你在16GiB内存的机器上运行32-bit memcached，那么此时多instance 是必需的。)
求解Sudoku用不了多大内存。如果单线程编程更方便的话，可以用单线程多进程来做。再在前面加一个单线程的 load balancer（负载均衡器），仿 lighttpd + fastcgi的成例。

线程不能减少工作量，即不能减少 CPU 时间。如果解决一个问题需要执行一亿条指令(这个数字不大，不要被吓到)，那么用多线程只会让这个数字增加。
但是通过合理调配这一亿条指令在多个核上的执行情况，我们能让工期提早结束。
这听上去像统筹方法，其实也确实是`统筹方法`。







# 第4章 C++多线程系统编程精要
学习多线程编程面临的最大的思维方式的转变有两点:
* `当前线程`可能随时会`被切换`出去，或者说被抢占(preempt)了。
* 多线程程序中事件的发生顺序`不再有全局统一的先后关系`
当线程被切换回来继续执行下一条语句(指令)的时候，全局数据(包括当前进程在操作系统内核中的状态)可能已经被其他线程修改了。
例如，在没有为指针p加锁的情况下，
```cpp
if (p && p->next)
  { /* ... */ }
```
有可能导致seg fault，因为在逻辑与(&&)的前一个分支evaluate 为 true 之后的一刹那，p可能被其他线程置为 NULL或是被释放，后一个分支就访问了非法地址，直接就崩溃了。
在单CPU系统中，理论上我们可以通过记录CPU上执行的指令的先后顺序来推演多线程的实际交织(interweaving)运行的情况。
在多核系统中，多个线程是并行执行的，我们甚至没有统一的全局时钟来为每个事件编号。
在没有适当同步的情况下，多个 CPU上运行的多个线程中的事件发生先后顺序是无法确定的。
在引入适当`同步`后，事件之间才有了happens-before关系
多线程程序的正确性不能依赖于任何一个线程的执行速度，不能通过原地等待`sleep()`来假定其他线程的事件已经发生，而**必须通过适当的同步来让当前线程能看到其他线程的事件的结果**。
PS: 夸张一点地想，核间通信速度为电信号的速度，2 * 10^8 m /s， 如果CPU主频为2.4GHz，一个时钟周期(0.42 ns)内，无法跨越15cm（假设两个CPU相距如此距离）
无论线程执行得快与慢(被操作系统切换出去得越多执行越慢)，程序都应该能正常工作。
例如下面这段代码就有这方面的问题。
```cpp
bool running = false; //全局标志
void threadFunc()
{
  while (running)
  {
    // get task from queue
  }
}

  void start()
  {
    muduo::Thread t(threadFunc);
    t.start();  // 在start()中，真正创建线程、启动线程
    running = true; // 应该放到 t.start()之前。
  }
```
这段代码暗中假定线程函数的启动慢于running变量的赋值，因此线程函数能进入 while 循环执行我们想要的功能。
如果上机测试运行这段代码，十有八九会按我们预期的那样工作。
但是，假设：直到有一天，系统负载很高，`Thread::start()`调用`pthread_create()`陷人内核后返回时，内核决定换另外一个就绪任务来执行。
于是running的赋值就推迟了，这时线程函数就可能不进入 while 循环而直接退出了
或许有人会认为在 while 之前加一小段延时 (sleep)就能解决问题，但这是错的，无论加多大的延时，系统都有可能先执行 while 的条件判断，然后再执行running的赋值。
正确的做法是把 running 的赋值放到t.start()之前，
这样借助`pthread_create()`的`happens-before`语意来保证running的新值能被线程看到。



## 4.1 基本线程原语的选用
我认为用C/C++编写跨平台的多线程程序不是普遍的需求，因此本书只谈现代Linux下的多线程编程。
POSIX threads的函数有110多个，真正常用的不过十几个。
而且在C++程序中通常会有更为易用的wrapper，不会直接调用 Pthreads函数。
这`11个最基本的Pthreads函数`是:
- 2个:  `线程`的`创建`和`等待结束`(join)。
  封装为 muduo::Thread。
- 4个:  `mutex`的`创建、销毁、加锁、解锁`。
  封装为 muduo::MutexLock。
- 5个:  `条件变量`的`创建、销毁、等待、通知、广播`。
  封装为 muduo::Condition。


多线程系统编程的难点不在于学习线程原语(primitives)，而在于**理解多线程与`现有的C/C++库函数`和`系统调用`的`交互关系`，以进一步学习如何设计并实现线程安全且高效的程序**。



## 4.2 C/C++系统库的线程安全性
现行的C/C++标准(C89/C99/C++03)并没有涉及线程，新版的C/C++标准(C11和 C++11)规定了程序在多线下的语意，C++11 还定义了一个线程库(`std::thread`)。
对于标准而言，关键的不是定义线程库，而是规定`内存模型(memory model)`
特别是：规定一个线程对某个共享变量的修改何时能被其他线程看到，这称为`内存序`(memory ordering)或者`内存能见度`(memory visibility)。
从理论上讲，如果没有合适的内存模型，编写正确的多线程程序属于撞大运行为
从操作系统开始支持多线程到现在已经过去了近20年，人们已经编写了不计其数的运行于关键生产环境的多线程程序

甚至Linux操作系统内核本身也可以是抢占的(preemptive)。
因此 **可以认为：每个支持多线程的操作系统上自带的C/C++ 编译器对本平台的多线程支持都足够好**。
现在多线程程序工作不正常很难归结于编译器 bug，毕竟POSIX threads 线程标准在20世纪90年代中期就制定了。
当然，新标准的积极意义在于让编写跨平台的多线程程序更有保障了。

`Unix系统库(libc和系统调用)`的接口风格是在20世纪70代早期确立的
而第一个支持用户态线程的 Unix 操作系统出现在20世纪90年代早期。线程的出现立刻给系统函数库带来了冲击，破坏了 20年来一贯的编程传统和假定。例如:
* errno不再是一个全局变量，因为每个线程可能会执行不同的系统库函数。
* 有些“纯函数”不受影响，例如`memset/strcpy/snprintf`等等。
* 有些`影响全局状态`或者`有副作用的函数`可以通过`加锁`来实现`线程安全`
  例如：`malloc/free、printf、fread/fseek`等等。
* 有些返回或使用静态空间的函数不可能做到线程安全，因此要提供另外的版本
  例如`asctime_r/ctime_r/gmtime_r、stderror_r、strtok_r`等等。
传统的`fork()`并发模型不再适用于多线程程序(S4.9)。

现在Linux glibc把errno定义为一个宏，注意errno是一个`lvalue`，因此不能简单定义为某个函数的返回值，而必须定义为`对函数返回指针的dereference`。
```c
extern int *__errno_location(void);   // 该函数返回一个指向错误码(int类型)的指针
#define errno (*__errno_location())   // 对指针解引用就得到错误码
```
值得一提的是，操作系统支持多线程已有近 20 年，早先一些性能方面的缺陷都基本被弥补了。
例如最早的SGI STL自己定制了内存分配器，而现在g++自带的STL已经直接使用 malloc 来分配内存，`std::allocator`已经变成了鸡肋(12.2)
原先Google tcmalloc相对于 glibc 2.3 中的ptmalloc2 有很大的性能提升，现在最新的glibc中的ptmalloc3已经把差距大大小了。
**我们不必担心系统调用的线程安全性，因为`系统调用`对于用户态程序来说是`原子的`**。
但是要注意系统调用对于内核状态的改变可能影响其他线程，这个话题留到4.6再细说。

与直觉相反，POSIX标准列出的是一份非线程安全的函数的黑名单8，而不是一份线程安全的函数的白名单(All functions defined by this volume of POSIX.1-2008shall be thread-safe, except that the following functions need not be thread-safe )。在这份黑名单中，`system、getenv/putenv/setenv`等等函数都是不安全的。

因此，可以说现在 glibc 库函数大部分都是线程安全的。
特别是`FILE*`系列函数是安全的，glibc甚至提供了非线程安全的版本，以应对某些特殊场合的性能需求。
尽管单个函数是线程安全的，但两个或多个函数放到一起就不再安全了。
例如`fseek()`和`fread()`都是安全的，但是对某个文件`“先seek再read”`这两步操作中间有可能会被打断，其他线程有可能趁机修改了文件的当前位置，让程序逻辑无法正确执行
在这种情况下，我们可以用`flockfile(FILE*)`和`funlockfile(FILE*)`函数来显式地加锁。
并且由于`FILE*`的锁是`可重入的`，加锁之后再调用`fread()`不会造成死锁。
如果程序直接使用`lseek(2)`和`read(2)`这两个系统调用来随机读取文件，也存在“先seek 再read”这种race condition，但是似乎我们无法高效地对系统调用加锁。
解决办法是改用`pread(2)`系统调用，它不会改变文件的当前位置。
由此可见，编写线程安全程序的一个难点在于线程安全是不可组合的(composable)，一个函数 foo()调用了两个线程安全的函数，而这个 foo()函数本身很可能不是线程安全的。
即便现在大多数 glibc 库函数是线程安全的，我们也不能像写单线程程序那样编写代码。
例如，在单线程程序中，如果我们要临时转换时区，可以用`tzset()`函数，这个函数会改变程序全局的“当前时区”。
```cpp
//获取伦敦的当前时间
string oldTz = getenv("TZ");
putenv("TZ=Europe/London");
tzset();

struct tm localTimeInLN;
time_t now = time(NULL);
localtime_r(&now，&localTimeInLN);
setenv("TZ"，oldTz.c_str(), 1);
tzset();
```
但是在多线程程序中，这么做不是线程安全的，即便tzset()本身是线程安全的。因为它改变了`全局状态(当前时区)`，这有可能影响其他线转换当前时间，
或者被其他进行类似操作的线程影响。
解决办法是使用muduo::TimeZone class，每个`immutable instance`(不变实例)对应一个时区，这样时间转换就不需要修改全局状态了。例如:
```cpp
class TimeZone
{
 public:
  explicit TimeZone(const char* zonefile);
  struct tm toLocalTime(time_t secondsSinceEpoch) const;
  time_t fromLocalTime(const struct tm&) const:

// default copy ctor/assignment/dtor are okay.
// ...
};

const TimeZone kNewYorkTz("/usr/share/zoneinfo/America/New_York");
const TimeZone kLondonTz("/usr/share/zoneinfo/Europe/London");
time_t now = time(NULL);
struct tm localTimeInNY = kNewYorkTz.toLocalTime(now);
struct tm localTimeInLN = kLondonTz.toLocalTime(now);
```

对于C/C++ 库的作者来说，如何设计线程安全的接口也成了一大考验，值得仿效的例子并不多。
一个基本思路是尽量把`class 设计成immutable`的，这样用起来就不必为线程安全操心了。
尽管 C++03 标准没有明说标准库的线程安全性，但我们可以遵循一个基本原则：
  **凡是非共享的对象都是彼此独立的，如果一个对象从始至终只被一个线程用到，那么它就是安全的**。
  另外一个事实标准是：**共享的对象的`read-only`操作是安全的(前提是，标准库容器不能使用splay tree那种数据结构，它在read时会修改状态)**，前提是不能有并发的写操作。
  例如两个线程各自访问自己的局部 vector 对象是安全的，同时访问共享的const vector 对象也是安全的，但是这个 vector 不能被第三个线程修改。
  **一旦有 writer，那么`read-only`操作也必须加锁**。
    例如`vector::size()`，根据S1.1.1对线程安全的定义，`C++的标准库容器`和`std::string`都`不是线程安全`的，只有`std::allocator`保证是`线程安全`的。
    一方面的原因是为了避免不必要的性能开销，另一方面的原因是单个成员函数的线程安全并不具备可组合性(composable)。
    假设有`safe_vector<T>class`，它的接口与std::vector 相同，不过每个成员函数都是线程安全的(类似Java synchronized 方法)。
    但是用`safe_vector<T>`并不一定能写出线程安全的代码。例如:

```cpp
// 全局可见
safe_vector<int> vec;
if (!vec.empty()) //没有加锁保护
{
  int x = vec[0]; // 这两步在多线程下是不安全的，和语义就不符合了
}
```
在if语句判断 vec 非空之后，别的线程可能清空其元素，从而造成`vec[0]`失效。即便`operator[]`是线程安全的，读到的元素也可能不是想要的。
C++标准库中的绝大多数`泛型算法`是`线程安全`的，因为这些都是无状态、纯函数。
只要输入区间是线程安全的，那么泛型函数就是线程安全的。

C++的`iostream`不是线程安全的，因为流式输出
```cpp
std::cout <<"Now is " << time(NULL);
// 等价于两个函数调用
std::cout.operator<<("Now is ")
          .operator<<(time(NULL));
```
即便`ostream::operator<<()`做到了线程安全，也不能保证其他线程不会在两次函数调用之前向`stdout`输出其他字符。
对于“线程安全的 stdout 输出”这个需求，我们可以改用`printf`，以达到安全性和输出的原子性。
但是这等于用了`全局锁`，任何时刻只能有一个线程调用 printf.恐怕不见得高效。
在多线程程序中高效的日志需要特殊设计，见第5章。


## 4.3 Linux上的线程标识
`POSIX threads`库提供了`pthread_self`函数用于返当前进程的标识符，其类型为pthread_t。
`pthread_t`不一定是一个数值类型(整数或指针)也有可能是一个结构体，因此Pthreads专门提供了`pthread_equal`函数用于对比两个线程标识符是否相等。
这就带来一系列问题，包括:
* 无法打印输出 pthread_t，因为不知道其确切类型。也就没法在日志中用它表示当前线程的id。
* 无法比较 pthread_t 的大小或计算其 hash 值，因此无法用作关联容器的 key。
* 无法定义一个非法的 pthread_t 值，用来表示绝对不可能存在的线程id，因此`MutexLock class`没有办法有效判断当前线程是否已经持有本锁。
* `pthread_t`值只在进程内有意义，与操作系统的任务调度之间无法建立有效关联。

比方说在`/proc`文件系统中找不到`pthread_t`对应的task。
另外，`glibc(？？？)的Pthreads`实现实际上把`pthread_t`用作一个结构体指针(它的类型是 unsigned long)，指向一块动态分配的内存，而且这块内存是反复使用的。
这就造成pthread_t的值很容易重复。
Pthreads 只保证同一进程之内同一时刻的各个线程的id 不同:
不能保证同一进程先后多个线程具有不同的 id，更不要说一台机器上多个进程之间的id唯一性了。（也就是说，它不是一个系统层面的全局的概念，甚至同一进程内的线程在多个时刻的线程id，也可能是相同的）

在 Linux上，我建议使用`gettid(2)`系统调用的返回值作为线程id，这么做的好处有:
它的类型是`pid_t`，其值通常是一个`小整数`，便于在日志中输出。
在现代 Linux中，它直接表示`内核的任务调度id`，因此在`/proc`文件系统中可以轻易找到对应项:
`/proc/tid`或`/prod/pid/task/tid`。
在其他系统工具中也容易定位到具体某一个线程，例如在`top(1)`中我们可以按线程列出任务，然后找出CPU 使用率最高的线程id，再根据程序日志判断到底哪一个线程在耗用CPU。
任何时刻都是全局唯一的，并且由于 Linux 分配新 pid 采用递增轮办法，短时间内启动的多个线程也会具有不同的线程 id。
0是非法值，因为操作系统第一个进程`init`的`pid`是`1`
但是glibc并没有封装这个系统调用，需要我们自己实现。
封装`gettid(2)`很简单，但是每次都执行一次系统调用似乎有些浪费，如何才能做到更高效呢?

`muduo::CurrentThread::tid()`采取的办法是用`__thread变量`来缓存`gettid(2)`的返回值，
这样只有在本线程第一次调用的时候才进行系统调用，以后都是直接从thread local缓存的线程id 拿到结果，效率无忧。

还有一个小问题，万一程序执行了`fork(2)`，那么子进程会不会看到 stale 的缓存结果呢?
解决办法是用`pthread_atfork()`注册一个回调，用于`清空缓存的线程id`。
具体代码见muduo/base/CurrentThread.h和Thread.cc。

## 4.4 线程的创建与销毁的守则
线程的`创建`和`销毁`是编写多线程程序的基本要素，线程的`创建`比`销毁`要容易得多，只需要遵循几条简单的原则:
* 程序库不应该在未提前告知的情况下创建自己的“背景线程”。
* 尽量用相同的方式创建线程，例如 muduo::Thread。
* 在进入main()函数之前不应该启动线程。
* 程序中`线程的创建`最好能在`初始化阶段`全部完成。

线程的销毁有几种方式:
* 自然死亡。
  从线程主函数返回，线程正常退出。
* 非正常死亡。
  从线程主函数抛出异常或线程触发segfault信号等非法操作
* 自杀。
  在线程中调用`pthread_exit()`来立刻退出线程。
* 他杀。
  其他线程调用`pthread_cancel()`来强制终止某个线程。
`pthread_ki1l()`是往线程发信号，留到S4.10再讨论。
线程正常退出的方式只有一种，即自然死亡。
任何从外部强行终止线程的做法和想法都是错的。
佐证有:Java的Thread class把stop()、suspend()、destroy()等函数都废弃(deprecated)了，Boost.Threads根本就不提供thread::cancel()成员函数。
**因为强行终止线程的话(无论是自杀还是他杀)，它没有机会清理资源。也没有机会释放已经持有的锁，其他线程如果再想对同一个mutex加锁，那么就会立刻死锁**。
因此我认为不用去研究 cancellation point这种“鸡肋”概念(见下一页 )。如果确实需要强行终止一个耗时很长的计算任务，而又不想在计算期间周期性地检查某个全局退出标志，那么可以考虑把那一部分代码fork()为新的进程，这样杀(`ki11(2)`)一个进程比杀本进程内的线程要安全得多。
当然，fork()的新进程与本进程的通信方式也要慎重选取，最好用文件描述符(`pipe(2)/socketpair(2)/TCP socket`)来收发数据，而不要用共享内存和跨进程的互斥器等IPC，因为这样仍然有死锁的可能。
muduo::Thread不是传统意义上的RAII class，因为它析构的时候没有销毁持有的 Pthreads 线程句柄(pthread_t)，也就是说 Thread 的析构不会等待线程结束。
一般而言，我们会让 Thread 对象的生命期长于线程，然后通过`Thread::join()`来等待线程结束并释放线程资源。
如果 Thread对象的生命期短于线程，那么就没有机会释放pthread_t了。
muduo::Thread 没有提供detach()成员函数，因为我不认为这是必要的。
最后，我认为如果能做到前面提到的“程序中线程的创建最好能在初始化阶段全部完成”，则线程是不必销毁的，伴随进程一直运行，彻底避开了线程安全退出可能面临的各种困难，包括 Thread 对象生命期管理、资源释放等等。


以下分别谈一谈这几个观点。
线程是稀缺资源，
* 一个`进程`可以创建的`并发线程数目`受限于`地址空间的大小`和`内核参数`，
* `一台机器`可以`同时并行运行`的`线程数目`受限于`CPU的数目`。
因此**我们在设计一个服务端程序的时候要精心规划线程的数目，特别是根据`机器的 CPU 数目`来设置`工作线程的数目`，并为`关键任务`保留足够的`计算资源`**。
如果程序库在背地里使用了额外的线程来执行任务，我们这种资源规划就漏算了。
可能会导致高估系统的可用资源，结果处理关键任务不及时，达不到预设的性能指标

还有一个重要原因是，一旦程序中有不止一个`线程`，就很难安全地`fork()`了(S4.9)。
因此“库”不能偷偷建线程。
如果确实有必要使用背景线程，至少应该让使用者知道。
另外，如果有可能，可以让使用者在`初始化库`的时候传入线程池或 event loop 对象，这样`程序`可以`统筹`线程的数目和用途，避免低优先级的任务独占某个线程。

理想情况下，程序里的线程都是用同一个class 创建的(muduo::Thread)，这样容易在线程的启动和销毁阶段做一些统一的簿记 (bookkeeping)工作。
比如说调用一次`muduo::CurrentThread::tid()`把当前线程缓存起来，以后再取线程id 就不会陷人内核了。
也可以统计当前有多少活动线程，进程一共创建了多少线程








## 4.6 多线程与IO
这可算是本章最为关键的一节。
本书只讨论同步IO，包括阻塞与非阻塞，不讨论异步IO(AIO)。
在进行多线程网络编程的时候，几个自然的问题是:
如何处理IO？
能否多个线程同时读写同一个socket文件描述符?
我们知道用多线程同时处理多个socket通常可以提高效率，那么用多线程处理同一个socket也可以提高效率吗?
首先，`操作文件描述符的系统调用`本身是`线程安全的`，我们不用担心多个线程同时操作文件描述符会造成进程崩溃或内核崩溃。
但是，多个线程同时操作同一个socket 文件描述符确实很麻烦，我认为是得不偿失的。需要考虑的情况如下:
如果一个线程正在`阻塞地read(2)`某个socket，而另一个线程`close(2)`了此socket.
如果一个线程正在阻塞地`accept(2)`某个listening socket，而另一个线程`close(2)`了此socket.
更糟糕的是，一个线程正准备`read(2)`某个socket，而另一个线程`close(2)`了此 socket;第三个线程又恰好`open(2)`了另一个文件描述符，其fd号码正好与前面的 socket相同。这样程序的逻辑就混乱了(S4.7)。
我认为以上这几种情况都反映了程序逻辑设计上有问题。
现在假设不考虑关闭文件描述符，只考虑读和写，情况也不见得多好。
因为socket读写的特点是不保证完整性，读100字节有可能只返回20字节，写操作也是一样的。
如果两个线程同时 read 同一个 TCP socket，两个线程几乎同时各自收到（整体的）一部分数据，如何把数据拼成完整的消息？如何知道哪部分数据先到达？(PS: 目前讨论的是，是否可以使用多线程read增加速度，实现比单线程read更快的效果，每个线程读取一部分，但是怎么合并是个问题)
如果两个线程同时 write 同一个 TCP socket，每个线程都只发出去半条消息那接收方收到数据如何处理?
如果给每个 TCP socket 配一把锁，让同时只能有一个线程读或写此 socket，似乎可以“解决”问题，但这样还不如直接始终让同一个线程来操作此socket 来得简单。
对于非阻塞 IO，情况是一样的，而且收发消息的完整性与原子性几乎不可能用锁来保证，因为这样会阻塞其他 IO线程。

如此看来，理论上只有`read`和`write`可以分到两个线程去，因为TCP socket 是双向 IO。问题是真的值得把read 和write 拆开成两个线程吗？
以上讨论的都是网络IO,那么多线程可以加速磁盘IO吗？
首先要避免`lseek(2)`/`read(2)`的race condition(S4.2)。
做到这一点之后，据我看，用多个线程 read 或write同一个文件也不会提速。
不仅如此，**多个线程分别read或write 同一个磁盘上的多个文件也不见得能提速**。
因为**`每块磁盘`都有一个`操作队列`，`多个线程`的`读写请求`到了`内核`是`排队执行`的**。
**只有在内核缓存了大部分数据的情况下，多线程读这些热数据才可能比单线程快**。
`多线程磁盘IO`的一个思路是`每个磁盘`配`一个线程`，把所有`针对此磁盘的IO`都挪到同一个线程，这样`或许 能 避免或减少 内核中的 锁争用`。
我认为应该用“显然是正确”的方式来编写程序，`一个文件`只由`一个进程`中的`一个线程`来读写，这种做法显然是正确的。
为了简单起见，我认为`多线程程序`应该遵循的原则是:
`每个文件描述符`只由`一个线程`操作，从而轻松解决`消息收发的顺序性问题`，也避免了`关闭文件描述符的各种race condition`。
**一个线程可以操作多个文件描述符，但一个线程不能操作别的线程拥有的文件描述符**。
这一点不难做到，muduo网络库已经把这些细节封装了。
`epoll`也遵循相同的原则。
Linux文档并没有说明:
当一个线程正阻塞在`epoll_wait()`上时，另一个线程往此`epoll fd`添加一个新的监视fd 会发生什么。
新fd上的事件会不会在此次`epol1_wait()`调用中返回?
为了稳妥起见，我们应该把对同一个`epoll fd`的操作(添加、删除、修改、等待)都放到`同一个线程`中执行，这正是我们需要 muduo::EventLoop::wakeup()的原因。
当然，一般的程序不会直接使用epoll、read、write，这些底层操作都由网络库代劳了。
这条规则有两个例外:
* 对于`磁盘文件`，在必要的时候多个线程可以同时调用pread(2)/pwrite(2)来读写同一个文件;
* * 对于UDP，由于协议本身保证消息的`原子性`，在适当的条件下(比如消息之间彼此独立)可以多个线程同时读写同一个UDP文件描述符。









# 第11章 反思C++面向对象与虚函数

## 11.7 值语义与数据抽象
**值语义(value semantics)**
指的是`对象的拷贝与原对象无关`，就像拷贝int一样。
C++的内置类型 (`bool/int/double/char`)都是值语义，
标准库里的`complex<>、pair<>、vector<>、map<>、string`等等类型也都是`值语意`，拷贝之后就与原对象脱离关系

与值语义对应的是“`对象语义(object semantics)`”，或者叫做`引用语义`(reference semantics)，

由于“引用”一词在 C++里有特殊含义，所以我在本文中使用“对象语义”这个术语。
**对象语义指的是面向对象意义下的对象，对象拷贝是禁止的**。
`例如 muduo里的 Thread 是对象语义，拷贝 Thread 是无意义的，也是被禁止的:因为Thread代表线程，拷贝一个Thread 对象并不能让系统增加一个一模一样的线程`。

同样的道理，拷贝一个Employee对象是没有意义的，一个雇员不会变成两个雇员，他也不会领两份薪水。
拷贝 Tcp Connection 对象也没有意义，系统中只有一个TCP连接，拷贝Tcp Connection 对象不会让我们拥有两个连接。
Printer 也是不能拷贝的，系统只连接了一个打印机，拷贝 Printer 并不能凭空增加打印机。
凡此总总，面向对象意义下的“对象”是`non-copyable`。


值语义与`不变性(immutable)`无关。
C++中的值语义对象也可以是 mutable，比如complex<>、pair<>、vector<>、map<>、string都是可以修改的。
值语义的对象不一定是 POD，例如 string 就不是 POD，但它是值语义的。
值语义的对象不一定小，例如`vector<int>`的元素可多可少，但它始终是值语义的。
当然，很多值语义的对象都是小的。


### 11.7.2 值语义与生命期
值语义的一个巨大好处是生命期管理很简单，就跟int一样，一你不需要操心int的生命期。
值语义的对象的使用：
* 要么是 stack object，
* 要么直接作为其他 object 的成员，
因此我们不用担心它的生命期(一个函数使用自己stack 上的对象，一个成员函数使用自己的数据成员对象)。

相反，对象语义的 object 由于不能拷贝，因此我们只能通过指针或引用来使用它。

**一旦使用指针和引用来操作对象，那么就要担心所指的对象是已被释放，这度是 C++ 程序 bug 的一大来源**。
此外，由于 C++ 只能通过指针或引用来获得多态性，那么在 C++ 里从事基于继承和多态的面向对象编程有其本质的困难————对象生命期管理(资源管理)。

考虑一个简单的对象建模：`家长与子女`：`a Parent has a Child,a Child knows its Parent`。

在C++中就要为资源管理费一番脑筋:
* Parent 和 child 都代表的是真人，肯定是不能拷贝的，因此具有对象语义。
* Parent 是直接持有 child吗？抑或Parent 和child通过指针互指？
* Xhild的生命期由 Parent 控制吗？
* 如果还有Parentclub和School两个class，分别代表家长俱乐部和学校:ParentClub has many Parent(s)，School has man yChild(ren)，那么如何保证它们始终持有有效的Parent对象和Child对象？
* 何时才能安全地释放Parent 和child ?

以下写法为什么错？
```cpp
class Child;
class Parent : boost::noncopyable {
  Child* myChild;
}

class Child : boost::noncopyable {
  Parent* myParent;
};
```

如果直接使用指针作为成员，那么如何确保指针的有效性？如何防止出现空悬指针？
child和Parent由谁负责释放？
在释放某个Parent 对象的时候，如何确保程序中没有指向它的指针？那么释放某个 child 对象的时候呢？
总之拿到裸指针很危险，毫无信息量，毫无回收机制。
！！！**我们可以通过`智能指针`，把对象语义转为值语义**。
那么让Parent和Child互相持有对方的智能指针，其中一个是`weak_ptr`即可（具体哪一个是`weak_ptr`取决于使用场景）。

如果场景简单，考虑的问题也会减少：
如果Parent 拥有 child，child 的生命期由其 Parent 控制，child 的生命期小于Parent，那么代码就比较简单:
```cpp
class Parent;
class Child : boost::noncopyable {
public:
  explicit Child(Parent* myParent_)
    : myParent(myParent_)
{}

private:
  Parent* myParent;
};

class Parent : boost::noncopyable {
public:
  Parent() : myChild(new Child(this))
  {}

private:
  boost::scoped_ptr<Child> myChild;
};
```
**Child的指针不能泄露给外界，否则仍然有可能出现空悬指针**



### 11.7.3 值语义与标准库
`C++要求凡是能放入标准容器的类型必须具有值语义`
但是，由于 C++ 编译器会为 class 默认提供 copy constructor 和 assignment operator，
因此除非明确禁止，否则class 总是可以作为标准库的元素类型一一尽管程序可以编译通过，但是隐藏了资源管理方面的 bug


### 11.7.4 值语义与C++语言
**C++的class本质上是值语义的**，这才会出现object slicing这种语言独有的问题（即：`将派生类对象赋值给基类对象时，会丢失派生类特有的成员变量和成员函数`），也才会需要程序员注意pass-by-value 和 pass-by-const-reference 的取舍。

值语义是C++语言三大约束之一，C++ 的设计初衷是让用户定义的类型(class) 能像内置类型(int)一样工作，具有同等的地位。
为此 C++ 做了以下设计(妥协):
* class的layout与C struct 一样，**没有额外的开销**。定义一个“只包含一个int成员的class”的对象开销和定义一个int一样。
* 甚至class data member都默认是uninitialized，因为函数局部的int 也是如此。
* class 可以在 stack 上创建，也可以在 heap 上创建。因为int 可以是 stack variable。
* class的数组就是一个个 class 对象挨着，没有额外的indirection。
  因为int数组就是这样的。因此派生类数组的指针不能安全转换为基类指针。
* 编译器会为 class 默认生成copy constructor 和assignment operator。
其他语言没有 copy constructor 一说，也不允许重载assignment operator。
C++的对象默认是可以拷贝的，这是一个尴尬的特性。
* 当class type传人函数时，默认是make a copy(除非参数声明为 reference)。
因为把int传入函数时是make a copy。
C++的“函数调用”比其他语言复杂之处在于：参数传递和返回值传递。
C、Java等语言都是传值，简单地复制几个字节的内存就行了。
但是 C++ 对象是值语义，如果以pass-by-value 方式把对象传人函数，会涉及拷贝构造。
代码里看到一句简单的函数调用，实际背后发生的可能是一长串对象构造操作，`因此减少无谓的临时对象是 C++代码优化的关键之一`。
* 当函数返回一个 class type 时，只能通过make a copy (C++不得不定义RVO来解决性能问题)。
因为函数返回int时是make a copy。
* 以class type 为成员时，数据成员是嵌入的。
  例如`pair<complex<double>，size_t>`的layout 就是`complex<double>`挨着`size_t`。减少通过指针访问的间接性，从而减少访存。
这些设计带来了性能上的好处，原因是 memory locality。


PS: C++返回值优化
返回值优化（Return Value Optimization，简称RVO）是C++编译器在某些情况下对函数返回值的优化技术。
它的目标是`减少函数返回值的拷贝操作`，提高程序的性能和效率。
C++中，函数返回一个对象时，通常会创建一个`临时对象`来存储返回值，然后再将这个`临时对象``拷贝`到函数调用的位置（`直接把返回的对象构造在调用者栈帧上`）。
这个拷贝操作可能会涉及到大量的内存复制和构造函数调用，对于大型对象来说，这可能会带来较大的性能开销。
返回值优化通过避免不必要的拷贝操作，`直接将函数内部创建的临时对象放置到函数调用的位置`，减少了一次拷贝。
具体来说，返回值优化有以下几种情况：
1. 命名返回值优化（Named Return Value Optimization，NRVO）：
  当函数内部创建一个局部对象并将其作为返回值时，编译器会尝试将该对象直接放置到函数调用的位置，而不是进行拷贝操作。
2. 临时对象优化（Temporary Object Optimization，TO）：
  当函数返回一个临时对象时，编译器会将该临时对象直接放置到函数调用的位置，而不是进行拷贝操作。
  这些优化技术可以显著提高程序的性能，尤其是在函数返回大型对象时。但需要注意的是，返回值优化并不是C++标准的要求，它是编译器的一种优化策略，不同的编译器可能会有不同的实现方式和支持程度。

为了充分利用返回值优化，可以遵循以下几个原则：
1. 尽量返回局部对象而不是全局对象或静态对象，因为局部对象更容易被编译器进行优化。
2. 避免在函数内部创建多个临时对象，因为每个临时对象都需要进行构造和析构操作。
3. 使用编译器提供的优化选项，如启用优化级别、开启返回值优化等。




### 11.7.5 什么是数据抽象















