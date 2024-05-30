
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
2. mutex是递归的，由于push_back()可能(但不总是)导致vector 选代器失效（因为被修改了），程序偶尔会crash。

这时候就能体现non-recursive 的优越性：**把程序的逻辑错误暴露出来**。
死锁比较容易 debug，把各个线程的调用栈打出来，只要每个函数不是特别长，很容易看出来是怎么死的，见2.1.2的例子9。
或者可以用`PTHREAD_MUTEX_ERRORCHECK`一下子就能找到错误(前提是 MutexLock 带 debug 选项)。
程序反正要死，不如死得有意义点，留个“全尸”，让验尸(post-mortem)更容易些。
如果确实需要在遍历的时候修改 vector，有两种做法：
1. 一是把修改推后，记住循环中试图添加或删除哪些元素，**等循环结束了再依记录修改 foos**;
2. 二是用**copy-on-write**，见2.8的例子。

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













