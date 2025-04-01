# 介绍一下vector ap的使用
AUTOSAR定义了超过300个AR-ELEMENT，VECTOR由自定义了一些，对AP模型进行了扩展





# vector代码梳理
## com
### 关键字
1. 为了方便src-gen这种代码参与编译，大量使用 “`template`类的函数不被调用（就不会生成代码、参与编译）就可以引入未实现函数” 的特性
2. 由于多重绑定的存在，用户层对socal调用：看似简单的`OfferService`，`Send`等操作，实际上都要遍历每一种binding的对象，调用下一层的接口
    常见场景：
    一个proxy对应两种binding的skeleton，这里如果有someip binding的skeleton，则是跨ECU的
    并且典型场景下，这里的someipd需要实现端口复用（详情见aracom_api文档）
    回调中，遍历service_handle数组，来针对不同的绑定做不同的处理

3. 第2点是核心思想是：依赖抽象/接口，而不是细节/具体实现
    用户层直接只管调用`socal`层的接口就完事了，底层binding实现要考虑的事情就多了

### vector实现层
其实也是代理模式，谁代理谁？
--> 用户层的Skeleton对象，被`用户层的SkeletonBinding类`所代理
    SomeIpSkeletonEventManager代理SkeletonEvent

socal层、具体binding层
socal层定义了一些绑定无关的抽象类/接口类：如`Skeleton`、`SkeletonEvent`
核心单例：`Runtime`持有关键对象`map<specifier, binding类>`，这里的binding类象征着所使用的绑定（是单例的！！！）
`具体binding类（如AraComSomeIpBinding）`持有`XXXBindingServerManager`和`XXXBindingClientManager`，这两个类和具体协议相关，调用具体协议相关的接口，收发数据
`用户层的SkeletonBinding类`会持有`XXXSkeletonEventManager`，`XXXSkeletonMethodManager`；
`用户层的Proxyinding类`会持有`XXXProxyEventManager`，`XXXProxyMethodManager`
**这里所说的`用户层的SkeletonBinding类`和`用户层的Proxyinding类`其实就相当于`radarServiceAdapter`，持有`EventImpl`等实现类，用户无法直接接触到**（通过指针取到用户层的`SkeletonBinding`或者`ProxyBinding`）



### 用户层
`intance_specifier`（字符串拼接，port的名字，其实someip_config.json或者ipc_config.json里也都是这个`instance_specifier`）/ `instance_identifier`（包含具体binding），二者是一对多的关系（存在多重绑定的概念）

建模集成阶段：
用户先进行服务配置（event、method、field），然后根据需求进行服务部署，选择具体的绑定、部署的平台（一些可能的端口配置）

代码逻辑：
调用`ara::core::Initialize()`，进行统一的初始化
    先读取配置文件，以单例方式实例化一些配置相关的对象，关键类是`map<specifier, binding类>`
    `OfferService`和`FindService`的实现中都需要查表：
        `std::set<InstanceSpecifierLookupTableEntry>`

根据所选择的binding，生成`static` `XXXBindingInitializer`对象（这些类的成员函数的实现在src-gen）

src-gen中包含关键binding类，如果是Skeleton端，就是`SkeletonBinding`，或者是`ProxyBinding`

**AUTOSAR规定**
需要生成出用户可以直接继承的skeleton类，proxy类则不同，先只持有指针，调用FindService静态函数（返回一个service_handle），在回调中构造Proxy对象，赋值给指针
用户可以直接访问到代表event、method、field的对象



## per
KVS，或者FS，Open的时候都是查找``<instance_specifier, 单例的实例>`




# AUTOSAR AP
## com
提供了标准化的API
本身就是对底层所使用的通信中间件的封装，就比如ROS2中通过环境变量来配置使用哪个中间件
AUTOSAR把通信抽象成服务，对于每一个服务提供`OfferService` & `StopOfferService`，`StartFindService`和`StopFindService`接口
对于服务中的event提供`Send`接口或者是`SetReceiveHandler`接口（在回调中可以调用`GetNewSamples`来取数据）
对于method则是提供method回调，或者是`operator()`接口

### 关键字
#### 为什么有这么多中间件，还要重新开发一套AUTOSAR专用通信管理？
需要一个`不受具体网络通信协议约束`的“通信管理”。
它必须支持`OME/IP 协议`，但必须具有改变协议的灵活性。
**应用程序接口应支持`事件驱动`和`轮询`模式，以同样好地访问通信数据。**
**`实时应用程序`通常需要后一种模式，以避免不必要的上下文切换，而前一种模式对于没有实时要求的应用程序来说要方便得多。**
可实现端到端保护的无缝集成，以满足`ASIL`要求。
支持静态（预配置）和动态（运行时）选择服务实例进行通信。

#### 所需的机制，集各家所长
* 代理（或存根）/骨架方法（CORBA、Ice、CommonAPI、Java RMI...）
* 独立于协议的 API（CommonAPI、Java RMI）
* 具有 “可配置接收端缓存 ”的队列通信（DDS、DADDY、经典平台）
* 零拷贝 API，可将 “内存管理 ”转移给中间件（DADDY）
* 数据接收过滤（DDS、DADDY）

**ara::com仅定义了“API签名 ”和“应用程序开发人员可见的行为”。提供这些API和底层中间件传输层的实现是 AUTOSAR AP 供应商的责任。**
应用程序接口设计的一个目标是 “尽可能精简”。
也就是说，它只应提供支持基于服务的通信范例所需的“最小功能集”，包括基本机制：方法、事件和字段。
从根本上说，应用程序接口应只处理服务消费者和服务提供者实施端处理方法、字段和事件通信的功能。
如果我们决定提供比这更多一些的功能，那么理由一般是“如果在我们的应用程序接口之上无法高效地解决某个通信相关问题，我们会将该解决方案作为 ara::com 应用程序接口层的一部分提供”。
核心设计要点之一是**支持`轮询`和`事件驱动编程`范例**。
如果你曾经从程序员的角度接触过中间件技术，那么`代理/骨架（Proxy/Skeleton）`体系结构的方法你可能已经耳熟能详了。
从使用代理/骨架（有时甚至称为存根/骨架）范例的中间件技术的数量来看，称其为 “经典方法”是有道理的。
因此，在 ara::com 中，我们也决定使用这种经典的**代理/骨架架构模式**，并将其命名为ara::com。
这种模式的基本思想是，从正式的service定义中生成两个代码工件：
* 服务代理： 从想要使用可能是远程服务的服务消费者的角度来看，该代码是在代码级别上代表该服务的门面。
在代码层面上。在面向对象的语言绑定中，这通常是一个生成类的实例，为服务提供的所有功能提供方法。因此，服务消费方的应用程序代码会与本地门面进行交互，而本地门面则知道如何将这些调用传播到远程服务实现并返回。
* 服务骨架： 从服务实现（根据服务定义提供功能）的角度来看，这段代码就是服务骨架、它允许将服务实现 “连接到通信管理传输层”，这样分布式服务消费者就能联系到服务实现。

#### 缓存设计
只有接收端有缓存的设计
数据经历了几层拷贝？
接收到数据buffer时，存储到`invisible_cache_`（实际上存储的都是`shared_ptr`）
老版本VECTOR实现中：
    用户主动调`Update()`时，把`invisible_cache_`的buffer数据进行`反序列化`，移到visible_cache_里，用户注册的回调可以决定对指向数据的指针做什么操作（是否延长生命周期）





#### skeleton method调用处理模式
轮询模式（适用于手动控制何时处理下一个请求的情况）、事件触发模式（并发处理）、单线程模式（按顺序处理，潜台词是允许method回调不可并发执行，尽管proxy method并发地调用，避免了skeleton端method回调对于同步机制的要求）
kPoll, kEvent, kEventSingleThread



### skeleton
#### 一些关键调用
对于someip而言，每个app就是一板一眼地给someipd发控制消息，socket连接都是由someipd建立。
##### skeleton
static `Skeleton`::`OfferService()`
    fastdds:    `DomainParticipant`、`Publisher`、`Subscriber`
                而`DataWriter`要到`EventImpl`的`OfferEvent`里才进行`create`，也就是`registerService`（还是在`OfferService()`）

static `Skeleton`::`StopOfferService()`

###### event
`Send()`

###### method 
定义回调

##### proxy
static `Proxy`::`StartFindService([](handles) { UserFindServiceHandler(handles) }, specifier)`
    fastdds:    service proxy端，一个publisher和一个subscriber就够用了
static `Proxy`::`StopFindService(handle)`

###### event
`SetReceiveHandler`

###### method
Future<> `operator()(Args&&... args)`

#### 主要逻辑
skeleton binding和proxy binding都有`factory`

skeleton端，有一个代理类：构造时执行各种`Offer`操作
1. event
用户层调用的`EventDispatcher`类，被`EventImpl`代理（因为多重绑定，所以是一对多的关系，通过list来存储代理类指针），调`Write()`

2. method
数据类型定义中会形成：
```cpp
request::dds_type_t
reply::dds_type_t
```

在`on_data_available`里面，触发一个函数：
```cpp
Execute(MethodImplBase* method, typename MethodImplBase::requestId_t requestId);    // MethodCallProcessingMode判断
```

调用到用户层的method回调
```cpp
enum class MethodCallProcessingMode
{
    kPoll,
    kEvent,             // 线程池
    kEventSingleThread  // 直接在on_data_available的线程执行用户层method回调
};
```

3. field
会维护：
```cpp
  /*!
   * \brief The field on skeleton-side shall always have access to the latest value, which has been set via Update.
   * This is necessary, in case no GetHandler is registered.
   */
  FieldType field_data_{};
```



### proxy
1. event
用户层调`SetReceiveHandler`，这个回调实际上放在`on_data_available`里调用

2. method
用户层直接同步调用`operator()`，`SetRequestData()`，promise存储到`pendingRequests_`，返回future对象（对应的Promise的set是在`on_data_available()`）


## state_manager
需要维护，每个function group的状态
配置文件里配置了，每个app运行在哪一个fg的哪一个状态
在代码里控制什么时候，启动哪些功能组的什么状态（这里的实现是使用em提供的api：`ara::exec::StateClient`，底层是使用管道发消息）

## execution_manager
每个app有一个`exe_config.json`，主要关于：进程的启动、关闭、基于cgroup的资源管理、用户管理
调度策略、调度优先级（静态优先级，是实时进程才使用的）、nice_value（常见的优先级，越小优先级越高）
    三种调度策略：`SCHED_RR`（一个进程吃完时间片，就到队列尾部，如果有优先级高的，就抢占）, `SCHED_FIFO`（先到先得、是非抢占的、有可能长时间运行），`SCHED_OTHER`（公平的分时调度，时间片轮转，时间片长度是动态调整的），前两种是实时调度策略，
环境变量、执行依赖、绑核、进入时间、退出时间、用户id、组id

# powerap
## com
### 关键词
继承接口类进行实现 / 设计新的类
多态、继承、模板、胶水代码
根据模型生成中间文件（`.cpp`，`.camke`）、链接不同的库

主要使用到的设计模式：
**代理模式**，**单例模式**，

对于dds binding而言
    arxml提取数据类型信息生成idl文件，调用fastddsgen

整体来说，就是AUTOSAR AP规定了基于service通信的接口，其中提供不同的绑定（甚至可以多重绑定），让用户无感知地调用接口，想要换绑定，只要改配置文件就行了，达到一种这样的效果

内存管理？？？
`Subscribe(size)`时预分配内存（或者说构造相应数量的对象）
当`GetNewSamples`时，从`invisible_cache_`取一个，反序列化后塞到`visible_cache_`，调用用户注册的回调
所使用的SamplePtr在析构时，会归还给`cache_`

### 架构设计
`DDSSkeletonServiceMananger`，`DDSProxyServiceMananger`（存在多个实例）
工厂





### 业务逻辑




#### OfferService




#### FindService









## per
### 关键词
全局函数`OpenKeyValueStorage(instance_specifier)`，`OpenFileStorage(instance_specifier)`，返回基类指针，对外接口的实现类里的实现当然是加锁了的
存储的时候一并存储版本信息、crc校验信息、冗余备份
调用Sync接口以同步到文件系统

文件存储的话，则是ReadAccessor、ReadWriteAccessor，打开某个名字的文件，读写字符

### 架构设计


### 业务逻辑
#### kvs



#### fs




# SOME/IP服务发现流程
集中式
## 几个阶段
### OfferService、FindService
都是udp多播消息
Offer报文中包含若干个提供的服务（service id, instance id, version, ttl），附加字段里包含ip + port
find报文中包含若干个请求的服务（service id, instance id, version, ttl）
（someipd显然需要维护一个可用服务列表）

### subscribe, subscribe ack（event）
单播
subscribe报文包含若干个请求订阅的event group（service id, instance id, event group id, version, ttl），附加字段里包含ip + port
subscribe ack报文包含对若干个请求订阅的event group（service id, instance id, event group id, version, ttl）的确认
可配置`多播阈值`：0：单播   X：订阅数低于阈值事件，则单播，大于等于阈值，则多播

服务上线了，不代表event准备好了，没准备好就订阅，则返回Subscribe Nack

### request，response
#### 半周期回复offer
`周期发Offer`阶段，前1/2周期收到的Find，回复一条`单播`的Offer
                  后1/2周期收到的Find，回复一条`组播`的Offer（收到得晚，可能网络拥塞）




### 持续发Offer以保活
Offer：当前服务可用，有效事件TTL
client端进行计时，超过了TTL还没收到Offer，则认为服务下线

### SD状态机
#### Server SD
##### Down
下线阶段
服务不可用 / 未注册
服务异常（网络连接中断）


当服务准备完毕后，进入以下状态
#### Ready
每个子状态都含有一个计时器，定时进入下一个阶段
##### Initial Wait Phase
经过`初始delay时间`后，发送第一包`OfferService`，进入下一阶段
如果服务不可用，返回上一阶段

##### Repetition Phase（可通过配置重复时间为0，跳过此阶段）
可配置重复次数、重复时间，发送间隔以指数形式增长（2的n次幂）
期间就算收到了FindService，也要延迟`（可配置的）一段时间`，发单播`OfferService`给服务请求端
如果服务不可用，返回`Down阶段`，并发送`StopOffer`
收到`SubscribeEventGroup`时，发送`单播Ack/Nack`，**启动此`订阅Entry`的TTL计时器**（如果收到`StopSubscribeEventGroup`，则停止计时器）

当以上阶段发送完N次（可配置次数），进入下一阶段
##### Main Phase
除非服务主动下线，否则一般不会再退出
固定周期（可配置）地发送OfferService
期间就算收到了FindService，也要延迟`（可配置的）一段时间`，发单播`OfferService`给服务请求端
如果服务不可用，返回`Down阶段`，并发送`StopOffer`
收到`SubscribeEventGroup`时，发送`单播Ack/Nack`，**启动此`订阅Entry`的TTL计时器**（如果收到`StopSubscribeEventGroup`，则停止计时器）

#### Client SD
##### Down

在Down状态下，收到OfferService，启动TTL计时器，进入下一阶段（如果同时服务被请求直接进入主阶段）
##### Initial Wait Phase
等待一段时间（可配置）
如果收到`OfferService`，则TTL计时器取消，直接进入主阶段
发送第一个`FindService`，进入重复阶段
服务请求被释放的话，返回Down

##### Repetition Phase
以指数增长的间隔发送若干次`FindService`
期间如果收到OfferService（停止发送和计时），直接进入主阶段，延迟一定时间发送`SubscribeEventGroup`
收到`StopOffer`的话，返回Down

##### Main Phase
如果是事件服务，延迟一定时间，触发发送`SubscribeEventGroup`
收到`StopOffer`的话，停止所有计时器，返回Down

client端特别的点在于，一旦收到Offer后，直接进入主阶段
而server端在主阶段不断周期发送Offer，而client不发（因为find是为了激活对端上线，一直发没有意义，浪费网络带宽，Offer不断发送以保活，刷新TTL）


## VECTOR AP SOME/IP协议栈
someipd实际上就一个`主线程`和一个`reactor线程`
someipd，对于每一个**sd endpoint**，维护`server_observers_map_`以及`client_observers_map_`（观察者模式）
`ServiceDiscoveryServer`如果收到`FindService`，所持有的`state_owner_`（持有当前某个状态的状态机，根据实际状态，调用同名的函数）
当收到订阅消息时，`event_manager_`所持有的`message_scheduler_`，调用`ScheduleSubscribeEventgroupAckEntry`以发送`SubscribeAck`
而`scheduler_`持有一个`<AddressPair, UnicastOneshotTimerUniquePtr>`的`TimerMap`（XXXTimer继承自Timer）
在reactor线程中会执行：
```cpp
      timer_manager_.HandleTimerExpiry();   // 相当于reactor线程去轮询计时器是否到期了
```
这个reactor要负责监听sd socket、数据socket、unix domain socket

### 每一个service，都要维护一个状态机（Server SD / Client SD）
使用多态（虚函数）





# DDS服务发现流程
## 关键字
发现机制、收发机制、QOS机制

发现协议的目的是允许每个 RTPS 参与者发现其他相关参与者及其端点。
一旦远程端点已经发现，实现可以相应地配置本地端点以建立通信。
DDS 规范同样依赖于使用发现机制来建立通信匹配的 DataWriters 和 DataReaders。
DDS 实现必须自动发现远程实体，无论是在他们加入和离开网络时。
RTPS 规范将发现协议拆分为两个独立的协议：（大型网络的基本发现协议。）
`Participant Discovery Protocol（PDP）【参与者发现协议】`
`Endpoint Discovery Protocol（EDP）【端点发现协议】`
为了互操作性，所有 RTPS 实现必须至少提供以下发现协议：（中小型网络的基本发现协议。）
Simple Participant Discovery Protocol （SPDP）
Simple Endpoint Discovery Protocol （SEDP）

## 1. DomainParticipant发现阶段（PDP, Participant Discovery Protocol）：
在此阶段，`DomainParticipant`参与者确认彼此的存在。
为此，每个DomainParticipant都会定期发送通知消息。
当两个给定的DomainParticipant存在于同一DDS域中时，它们将匹配。
默认情况下，使用已知的`多播地址`和`端口`（使用`Domain Id`计算）发送通知消息。
此外，还可以指定使用`单播`发送通知的地址列表。
此外，还可以配置此类通知的周期性。



## 2. 端点发现阶段（EDP, Endpoint Discovery Protocol）：
在此阶段，`DataWriter`和`DataReader`相互确认。
为此，`DomainParticipants`使用PDP期间建立的通信信道，彼此共享有关其`DataWriter`和`DataReader`的信息。
除其他外，此信息还包含`主题`和`数据类型`。
要使两个`端点`匹配，**其主题和数据类型必须一致**。
一旦DataWriter和DataReader匹配，它们就可以发送/接收用户数据流量了。

### 四种发现机制
Fast DDS提供以下发现机制：
1. 简单发现：
    这是默认机制。它支持PDP和EDP的RTPS标准，因此提供与任何其他DDS和RTPS实现的兼容性。
2. 静态发现：
    该机制在PDP阶段使用简单参与者发现协议（SPDP）（如RTPS标准所规定），但允许在事先知道所有DataWriter和DataReader的IP和端口、数据类型和主题时跳过简单端点发现协议（SEDP）阶段。
3. 发现服务器：
    此发现机制使用集中式发现架构，其中DomainParticipant（称为服务器）充当元流量发现的中心。
4. 手动发现：
    此机制仅与RTPS层兼容。它禁用PDP，允许用户使用其选择的任何外部元信息通道手动匹配和取消匹配RTPSParticipant、RTPSReader和RTPSWriter。因此，用户必须访问DomainParticipant实现的RTPSParticipant，并直接匹配RTPS实体。



# fastdds
## 关键字
### eProsima Fast DDS-Gen
eProsima Fast DDS-Gen是一个 Java 应用程序，它使用 IDL（接口定义语言）文件中定义的数据类型生成eProsima Fast DDS源代码。
eProsima Fast DDS-Gen生成的源代码使用Fast CDR，这是一个提供数据序列化和编码机制的 C++11 库。
因此，正如RTPS 标准中所述，发送数据时，它们会使用相应的通用数据表示 (CDR) 进行序列化和编码。
CDR 传输语法是代理间传输的低级表示，从 OMG IDL 数据类型映射到字节流。

### foonathan_memory_vendor
an STL compatible C++ memory allocator library.

### fastcdr
a C++ library for data serialization according to the CDR standard (Section 10.2.1.2 OMG CDR).

### fastrtps
the core library of eProsima Fast DDS library.

### RTPS协议
一个domain  --  若干个participant   --  若干个Publisher（和DataWriter是`一对多`关系）、若干个Subscriber（和DataReader是`一对多`关系）
**一个`participant`对应若干个`topic`**
**一个`Publisher`对应若干个`topic`**

Fast DDS中的`RTPS层`实现了`RTPS`协议，它作为DDS标准的⼀部分，负责处理底层⽹络通信、发现服务、数据传输以及QoS策略的具体实现等。
在Fast DDS中，`DCPS层`为`⽤⼾`提供抽象接⼝，隐藏了底层通信机制的复杂性；
⽽`RTPS层`则是`实际执⾏这些操作并保证数据可靠传输`的基础。
实际代码中，存在这样的映射：`DCPS` --> `RTPS`

### 并⾏模型
FastDDS中每个节点（也叫 `DomainParticipant`）具有：
⼀个`主程序线程`（⽤⼾持有）
⼀个`事件和周期性任务`的线程
⼀个`异步发送线程`，⽤于⽤⼾完成写⼊数据后，异步得完成⽹络通信
多个`接收线程`，每个`reception channel`，取决于传输层的实现⽅式

### RTPS的通信传输实现
⽀持SHM，UDP，TCP

### 域（Domain）：
这是用于链接所有发布者和订阅者的概念，属于一个或多个应用程序，它们在不同主题下交换数据。
这些参与域的单个应用程序称为DomainParticipant。DDS域由`domain ID`标识。
DomainParticipant定义Domain ID以指定它所属的DDS域。
具有不同ID的两个DomainParticipants不知道彼此在网络中的存在。
因此，可以创建多个通信通道。
这适用于涉及多个DDS应用程序的场景，它们各自的DomainParticipants相互通信，但这些应用程序不得干扰。
DomainParticipant充当其他 DCPS实体的容器，充当发布者、订阅者和主题实体的工厂，并在域中提供管理服务。

### 主题（Topic）：
**它是将发布者的`DataWriters`与订阅者的`DataReaders`绑定的实体，在DDS域中是`唯一的`**。
它可在进程之间交换的数据的消息，数据表示为可以包含不同数据类型的结构，如整数，字符串等;

### 数据写入器（Data Writer）：
它是负责`发布消息`的实体，用户在创建此实体时必须提供一个主题，该主题将是发布数据的主题；

### 数据读取器（Data Reader）：
它是订阅主题以`接收发布`的实体，用户在创建此实体时必须提供订阅主题；

### 发布者（Publisher）：
它是负责`创建 和 配置`其实现的`DataWriters`的DCPS实体。
`DataWriter`是负责实际`发布消息`的实体。
每个人都有一个分配的主题，在该主题下发布消息;

### 订阅者（Subscriber）：
它是DCPS实体，负责接收在其订阅的主题下发布的数据。
它为一个或多个`DataReader`对象提供服务，这些对象负责将新数据的可用性传达给应用程序;

### 发布消息步骤：
创建要发布数据的 idl ，并通过 fast-gen 生成对应的数据结构，并生成该数据结构序列化的类
通过 `DomainParticipantFactory` 创建 participant ，参数为 `domain id` 和 `qos`
将 participant 注册到 type ，该类型提供 helloworld 的序列化支持
通过 participant 创建 publisher ，用于发布数据
`PubListener` 继承于 `DataWriterListener` ，用于给发布者注册消息通知函数。
DDS 通过服务发现，将消息最终给到应用
publisher 创建 writer ，用于最终的消息发布

### QOS
#### someip_to_dds所使用的
基本上就是ROS2里默认的QOS profile
1. DataWriter
```cpp
  wqos.reliability().kind = RELIABLE_RELIABILITY_QOS;
  wqos.history().kind = KEEP_LAST_HISTORY_QOS;
  wqos.history().depth = 10;                    // 必须配合 KEEP_LAST_HISTORY_QOS
  wqos.endpoint().history_memory_policy = eprosima::fastrtps::rtps::PREALLOCATED_WITH_REALLOC_MEMORY_MODE;
```
2. DataReader
```cpp
  rqos_adasv2positiontopic.reliability().kind = RELIABLE_RELIABILITY_QOS;
  rqos_adasv2positiontopic.history().kind = KEEP_LAST_HISTORY_QOS;
  rqos_adasv2positiontopic.history().depth = 10;
  rqos_adasv2positiontopic.endpoint().history_memory_policy = eprosima::fastrtps::rtps::PREALLOCATED_WITH_REALLOC_MEMORY_MODE;
```

`enumerator PREALLOCATED_MEMORY_MODE`
Preallocated memory.   预分配内存。

Size set to the data type maximum. Largest memory footprint but smallest allocation count.
Size 设置为数据类型 maximum。内存占用量最大，但分配计数最小。

`enumerator PREALLOCATED_WITH_REALLOC_MEMORY_MODE`
Default size preallocated, requires reallocation when a bigger message arrives.
默认大小 `preallocated`，当更大的消息到达时需要重新分配。

Smaller memory footprint at the cost of an increased allocation count.
内存占用更少，但代价是分配计数增加。


#### best effort
减少重传和确认机制


#### reliable
使用了消息确认机制


## 收数据的接口
`on_data_available`由`transport层`的接口调用到`perform_listen_operation(Locator)`
```cpp
void UDPChannelResource::perform_listen_operation(
        Locator input_locator)
{
    Locator remote_locator;

    while (alive())
    {
        // Blocking receive.
        auto& msg = message_buffer();
        if (!Receive(msg.buffer, msg.max_size, msg.length, remote_locator))
        {
            continue;
        }

        // Processes the data through the CDR Message interface.
        if (message_receiver() != nullptr)
        {
            message_receiver()->OnDataReceived(msg.buffer, msg.length, input_locator, remote_locator);
        }
        else if (alive())
        {
            logWarning(RTPS_MSG_IN, "Received Message, but no receiver attached");
        }
    }

    message_receiver(nullptr);
}
```

一种channel对应一个线程
udp的实现：（使用`asio`中的接口，阻塞地收数据）

```cpp
while() {
    //  ...
    receive_from();
}
```

shm的实现：
```cpp
    /**
     * Function to be called from a new thread, which takes cares of performing a blocking receive
     * operation on the ReceiveResource
     * @param input_locator - Locator that triggered the creation of the resource
     */
    void perform_listen_operation(
            Locator input_locator) {
        while() {
            // 阻塞队列Pop
            // ...
            // OnDataReceived();
        }
    }
```

## `write()`如何调用到`serialize()`
可以还原类型
**`DataWriterImpl`可以拿到`TypeSupport`（继承自`shared_ptr`），由`Participant`持有**
序列化的数据，若序列化失败，塞到`payload_pool`
成功则塞进`history_`（由`DataWriterImpl`持有）




## 接收端如何被通知？
底层transport层进行通知，每个通道一个线程

# vsomeip
## boost::asio（已经封装了`epoll`）
整个是一个异步编程的框架
`routing_manager_impl`把所持有的`io_context`到处往外传，构造`service_discovery_impl`时直接把this指针传出去了，也就顺理成章地把`io_context`给出去了
于是可以接收各种someip消息，再根据类型转发给相应的app了
someipd还是和其他app使用unix domain socket来通信，其他的app是使用routing_manager_client来和someipd通信的，其中触发`application_impl：：on_message()`回调，和sd相关的逻辑都集中在`sd::service_discovery`，一个app里根据配置，持有`routing_manager_impl` / `routing_manager_client`中的一个





# SOME/IP 和 DDS的对比
## 集中式SOME/IP协议的好处
SOME/IP（Scalable service-Oriented MiddlewarE over IP）
    是一种专门为汽车工业设计的中间件协议，旨在通过IP网络实现面向服务的通信。
其集中式架构具有以下优点：
简化集成与维护：
    由于采用集中式管理，系统中的各个组件可以通过一个中心节点进行配置和监控，这大大简化了系统的集成和后续维护工作。
易于管理和控制：
    所有服务请求都需经过中央服务器处理，便于实施统一的安全策略、访问控制以及服务质量保证。
高效资源利用：
    通过优化路由和服务调度，可以更有效地利用网络资源和计算资源。

## 分布式DDS协议的好处
DDS（Data Distribution Service）数据分发服务
    是一种用于实时系统中发布/订阅模式的数据连接的中间件标准。
它的分布式特性带来了如下优势：
`高可用性`和可靠性：
    由于没有单点故障，**即使部分节点出现故障，系统仍然能够正常运行**，提高了整体的可靠性和可用性。
灵活性和扩展性：
    每个节点都可以独立地发布或订阅数据，使得系统`非常灵活且容易扩展`。新节点加入网络时不需要对现有系统做大量修改。
低延迟和实时性能：
    DDS协议支持`点对点`直接通信，减少了消息传递的延迟，非常适合需要快速响应的应用场景，如自动驾驶等。

# someip_to_dds
## 关键词
ROS2 rmw通信层也是默认使用fastdds，只要someip_to_dds的QOS配置与ROS2端保持一致，理论上就能够通信。
arxml -- idl保持一致，执行运行时的数据拷贝。
ROS2的`topic`映射到`fastdds`，会含有`rt`字段
ROS2的`service`映射到`fastdds`，会含有`rq`字段

单例、设置回调
一个`DomainParticipant`
一个`Publisher`
一个`Subscriber`

若干组`Topic` 和 `TypeSupport`
若干`DataReader`和若干`DataWriter`，都使用了history

### method
ROS2 as client
生产者消费者模型：收到fastdds request（ROS2 request）数据时，记录`writer_guid`和`sequence_number`，一并存储到阻塞队列中，
线程1不断将数据出队列，调用AP method回调，将future存储到阻塞队列中，
线程2不断地轮询future，将结果转成dds格式数据，发回给ROS2端
```cpp
    write_params.related_sample_identity().writer_guid() = info.related_sample_identity.writer_guid();              // 不set的话，收不到数据
    write_params.related_sample_identity().sequence_number().high = info.sample_identity.sequence_number().high;    // 不set的话，认为服务无效
    write_params.related_sample_identity().sequence_number().low = info.sample_identity.sequence_number().low;
```


### 性能
在合适的地方，针对不包含动态数据类型的数据类型，进行`memcpy`
camera Image数据（几十M），20几Hz
其他数据上百Hz，也能保证收发频率一致

# recorder
debug模式（分钟级别，一分钟一个文件，不接受trigger） + product模式（秒级别，一秒钟一个文件，接受trigger）两种模式可以切换
类似于行车记录仪，自动删除最老的数据

生产者消费者模型、阻塞队列
线程池
**mcap**
```cpp
/**
 * @brief Describes a schema used for message encoding and decoding and/or
 * describing the shape of messages. One or more Channel records map to a single
 * Schema.
 */
struct MCAP_PUBLIC Schema
{
    SchemaId id;
    std::string name;
    std::string encoding;
    ByteArray data;

    Schema() = default;

    Schema(const std::string_view name, const std::string_view encoding, const std::string_view data)
        : name(name),
          encoding(encoding),
          data{reinterpret_cast<const std::byte*>(data.data()),
               reinterpret_cast<const std::byte*>(data.data() + data.size())}
    {
    }

    Schema(const std::string_view name, const std::string_view encoding, const ByteArray& data)
        : name(name), encoding(encoding), data{data}
    {
    }
};

/**
 * @brief Describes a Channel that messages are written to. A Channel represents
 * a single connection from a publisher to a topic, so each topic will have one
 * Channel per publisher. Channels optionally reference a Schema, for message
 * encodings that are not self-describing (e.g. JSON) or when schema information
 * is available (e.g. JSONSchema).
 */
struct MCAP_PUBLIC Channel
{
    ChannelId id;
    std::string topic;
    std::string messageEncoding;
    SchemaId schemaId;
    KeyValueMap metadata;

    Channel() = default;

    Channel(const std::string_view topic,
            const std::string_view messageEncoding,
            SchemaId schemaId,
            const KeyValueMap& metadata = {})
        : topic(topic), messageEncoding(messageEncoding), schemaId(schemaId), metadata(metadata)
    {
    }
};

/**
 * @brief A single Message published to a Channel.
 */
struct MCAP_PUBLIC Message
{
    ChannelId channelId;
    /**
     * @brief An optional sequence number. If non-zero, sequence numbers should be
     * unique per channel and increasing over time.
     */
    uint32_t sequence;
    /**
     * @brief Nanosecond timestamp when this message was recorded or received for
     * recording.
     */
    Timestamp logTime;
    /**
     * @brief Nanosecond timestamp when this message was initially published. If
     * not available, this should be set to `logTime`.
     */
    Timestamp publishTime;
    /**
     * @brief Size of the message payload in bytes, pointed to via `data`.
     */
    uint64_t dataSize;
    /**
     * @brief A pointer to the message payload. For readers, this pointer is only
     * valid for the lifetime of an onMessage callback or before the message
     * iterator is advanced.
     */
    const std::byte* data = nullptr;
};
```
注意到：`Message`中包含有`ChannelId`，而`Channel`和`Schema`是一一对应的



根据topic，大量生成如下代码：用于`McapWriter`
```cpp
// 初始化阶段进行以下操作：

// 需要读取一些.msg文件（ros2所使用的），目的是可供ROS2环境使用、支持plotjuggler    而rosbag play 跟 plotjugger 对schema的data的处理不一样
// schema       确保：发送方和接收方 对于 所交换数据的理解是一致的
schema_test_->name = "sensor_msgs/msg/Image";
schema_test_->encoding = "ros2msg";                 // schema的encoding
auto [format_test, full_text_test] = msgdef_cache.get_full_text(schema_test_->name);                    // 读取.msg文件
schema_test_->data.assign(reinterpret_cast<const std::byte*>(full_text_test.data()),                    // 将消息定义转换成字节流
    reinterpret_cast<const std::byte*>(full_text_test.data() + full_text_test.size()));
writer_->addSchema(*schema_test_);

// channel
channel_test_->topic = "/sensor/camera/front/h264";
channel_test_->messageEncoding = "cdr";             // channel的encoding
channel_test_->schemaId = schema_test_->id;
channel_test_->metadata.emplace("offered_qos_profiles", QosToString(TOPIC_QOS_DEFAULT));                // 遵从metadata.yaml里的配置
available_channels_.push_back(channel_test_);

for (auto channel : available_channels_) {
writer_->addChannel(*channel);
}

template <typename T>
class TypeConverterMsg
{
  public:
    void ConvertToMsg(mcap::Message* msg, T& data, mcap::ChannelId channel_id)
    {
        // serialize DDS structure to binary stream using fastcdr
        eprosima::fastcdr::Cdr serializer(
            fast_buffer_, eprosima::fastcdr::Cdr::DEFAULT_ENDIAN, eprosima::fastcdr::Cdr::DDS_CDR);
        serializer.serialize_encapsulation();
        serializer << data;
        size_t size = serializer.getSerializedDataLength();
        // create the mcap message
        auto timestamp_ns{
            std::chrono::duration_cast<std::chrono::nanoseconds>(std::chrono::system_clock::now().time_since_epoch())
                .count()};

        msg->sequence = 0;
        msg->channelId = channel_id;
        msg->logTime = static_cast<uint64_t>(timestamp_ns);
        msg->publishTime = msg->logTime;                //  用于plotjugger播包

        std::byte* ptr = new std::byte[size];
        std::memcpy(ptr, fast_buffer_.getBuffer(), size);
        msg->data = ptr;
        msg->dataSize = size;
    }
    eprosima::fastcdr::FastBuffer fast_buffer_;
};
/*
    收到数据时，进行两层格式转换：
    1. AP格式的数据   -->  DDS格式的数据
        ConvertToIdl()
    2. DDS格式的数据  -->  Mcap的msg  需要填充信息
        ConvertToMsg()
*/
```

存储池
当收到切片信号时（product特有），
实际上就是从/tmp下的内存中的文件进行选择（根据时间戳选择前M后N的秒级文件），合并，然后调用`copy_file_range`拷贝到磁盘


文件结构
```xml
<Magic>
<Header>
<Data section>[<Summary section>][<Summary Offset section>]
<Footer>
<Magic>
```
**Data Section**
数据部分包含带有消息数据，附件和支持记录的记录。
数据部分允许出现以下记录：
```sh
Schema
Channel
Message
Attachment
Chunk
Message Index
Metadata
Data End
```

# data collector
单线程-单循环模式
数据的收集（根据收集文件的类型，包装成相应任务类，尽量均匀地完成文件的拷贝，`C++17 filesystem接口，copy_file_range(src_fd, des_fd, size)`）、压缩（`gzip`）、上传（`poco`）
线程池
云端通信
读取配置，得到`url`
`SetHeader()`, `SetBody()`组合出一个`request`
```cpp
    BigFileUploadRequestFactory factory = BigFileUploadRequestFactory(info);
    BigFileUploadRequest request = factory.MakeRequest(header, body);

    auto result = request.Execute(header, body);
```
重试和分片是怎么做的？断点续传？


# fastdds实现
## 发数据 `write(void*)`
```cpp
DataWriterImpl::write(void* data)

DataWriterImpl::create_new_change(ALIVE, data)  // ALIVE, 表示样本有效，其他状态还包括：被处置、注销、处置以及注销
{
    WriteParams wparams;
    return create_new_change_with_params(changeKind, data, wparams);
}

ReturnCode_t DataWriterImpl::create_new_change_with_params(ChangeKind_t changeKind, void* data, WriteParams& wparams)
{
    ReturnCode_t ret_code = check_new_change_preconditions(changeKind, data);
    // ...
    InstanceHandle_t handle;
    // ...
    return perform_create_new_change(changeKind, data, wparams, handle);
}

// ！！！PS：用户层可以通过loan_sample(void*& sample, LoanInitializationKind initialization)接口取得
ReturnCode_t DataWriterImpl::perform_create_new_change(ChangeKind_t change_kind, void* data, WriteParams& wparams, const InstanceHandle_t& handle)
{
    std::unique_lock<RecursiveTimedMutex> lock(writer_->getMutex());
    PayloadInfo_t payload;
    bool was_loaned = check_and_remove_loan(data, payload);
    // 如果有借用，则可以通过指针或者说地址得知，从记录的借用列表中移除
    // 如果没有借用。则调用：
        get_free_payload_from_pool(type_->getSerializedSizeProvider(data), payload)     // 先把数据指针转移到CacheChange_t对象中，再转移到payload中
    // 从内存池中取出free payload（这里的内存池实际对象和是否开启data sharing[数据共享交付 DataReader共享DataWriter的history 难道就是零拷贝？？？]有关，DataSharingPayloadPool / TopicPayloadPool。PREALLOCATED_WITH_REALLOC_MEMORY_MODE等QOS配置也影响到此处）
    ！！！free_payloads_其实是vector的结构
    // 调用序列化
    // 构造一个新的CacheChange_t对象，把上面的payload的数据再转移进去
    // 把数据塞进history，调用到rtps层的add_change_(CacheChange_t* a_change, WriteParams& wparams, std::chrono::time_point<std::chrono::steady_clock> max_blocking_time)，其中有一行关键调用：
        // WriterHistory:
        notify_writer(a_change, max_blocking_time);     // RTPSWriter有好几种！！！具体选择哪种实例化应该是createRTPSWriter(...)里面决定的，内部是根据QOS配置来决定的：如果是reliable就是StatefulWriter、如果是unreliable就是StatelessWriter
            // 内部实现：
            mp_writer->unsent_change_added_to_history(a_change, max_blocking_time);
}

// DataWriter提供一个指向内部缓冲区（池）的指针，用户可以直接在此缓冲区内准备数据以供发送。此方法仅适用于平凡数据类型的DataWriter
ReturnCode_t DataWriterImpl::loan_sample(void*& sample, LoanInitializationKind initialization);

        /*
            上面的数据写入到history之后，显然会异步地发送
            先把数据传到FlowController对象，塞进Scheduler，其维护一个Queue
        */
                sched.add_new_sample(writer, change);

                // 专门的线程，跑while循环
                // 通知具体的Writer，数据可以走底层发送了，调接口判断采取哪种方式进行分发
                    fastrtps::rtps::DeliveryRetCode ret_delivery = current_writer->deliver_sample_nts(
                        change_to_process, async_mode.group, locator_selector,
                        std::chrono::steady_clock::now() + std::chrono::hours(24));
```

数据最终如何分发？分为三种情况：
1. 同一进程内的DataReader、
2. 同一个域的
3. 跨域的
```cpp
DeliveryRetCode StatefulWriter::deliver_sample_nts(
        CacheChange_t* cache_change,
        RTPSMessageGroup& group,
        LocatorSelectorSender& locator_selector, // Object locked by FlowControllerImpl
        const std::chrono::time_point<std::chrono::steady_clock>& max_blocking_time)
{
    DeliveryRetCode ret_code = DeliveryRetCode::DELIVERED;

    if (there_are_local_readers_)
    {
        deliver_sample_to_intraprocesses(cache_change);
    }

    // Process datasharing then
    if (there_are_datasharing_readers_)
    {
        deliver_sample_to_datasharing(cache_change);
    }

    if (there_are_remote_readers_)
    {
        ret_code = deliver_sample_to_network(cache_change, group, locator_selector, max_blocking_time);
    }

    check_acked_status();

    return ret_code;
}
```
这里如何通知到udp writer？/ 如何和transport层交互？？？
可能是通过条件变量，notify某一个listner线程，让其执行

？？？locator是什么东西？？？网络定位符？是ip + port吗？



## 收数据
每种通道，都有一个线程在执行`while`
```cpp
void UDPChannelResource::perform_listen_operation(
        Locator input_locator)
{
    Locator remote_locator;

    while (alive())
    {
        // Blocking receive.
        auto& msg = message_buffer();
        if (!Receive(msg.buffer, msg.max_size, msg.length, remote_locator))
        {
            continue;
        }

        // Processes the data through the CDR Message interface.
        if (message_receiver() != nullptr)
        {
            message_receiver()->OnDataReceived(msg.buffer, msg.length, input_locator, remote_locator);
        }
        else if (alive())
        {
            logWarning(RTPS_MSG_IN, "Received Message, but no receiver attached");
        }
    }

    message_receiver(nullptr);
}
```

## 内存池
### WriterPool
调`write()`时，计算数据长度（递归计算），向内存池申请一块内存，
它存储每个序列化后的数据buffer，以`CacheChange_t`的形式存储，存储到`DataWriter`的history中

内存池其实就是：
```cpp
std::vector<PayloadNode*> free_payloads_;
```
每个`PayloadNode`所申请的内存长度是不一的，必要时才重新申请（直接调底层的`realloc(ptr, size)`）
内存池大小和history的深度相关

### ReaderPool


## data sharing
Although Data-sharing delivery uses shared memory, it differs from Shared Memory Transport in that Shared Memory is a full-compliant transport. That means that with Shared Memory Transport the data being transmitted must be copied from the DataWriter history to the transport and from the transport to the DataReader. With Data-sharing these copies can be avoided.

尽管数据共享传输使用共享内存，但它与共享内存传输不同 因为共享内存是完全兼容的传输。这意味着，使用共享内存传输 必须将正在传输的数据从 DataWriter 历史记录复制到传输 以及从运输到 DataReader。通过数据共享，可以避免这些副本。

省略了writer的缓存和reader的缓存（设计了巧妙的数据结构和同步机制）
--> 环形缓冲区
无锁编程 / 轻量级锁 ？





# ROS2
## 架构
从上往下：
用户层（ROS2节点）
rclcpp（ROS Client Library for C++）
rcl（ROS2 client Library）（C语言实现的，可能是为了提供）通信库（和rclcpp合称为ROS2 client层，提供了对ROS话题、服务、参数、Action等接口）
RMW层（DDS抽象层）为了使得封装DDS实现层，保持统一性（其下有rmw_fastrtps, rmw_connextdds）
DDS实现层（封装DDS，封装大量的设置、配置操作）
操作系统



## 基本概念
### Node（节点）：
在 rclcpp 中，节点是最基本的通信单元。节点可以理解为一个执行特定任务的进程中的实例，它通过 ROS 2 中间件进行通信和交互。
节点通过 rclcpp::Node 类来表示和管理。每个节点都有一个全局唯一的名称，以便在 ROS 2 系统中唯一标识它。

### Publisher（发布者）和 Subscriber（订阅者）：
节点通过发布者（Publisher）和订阅者（Subscriber）来实现与其它节点的通信。发布者用于向特定的主题发布消息，而订阅者用于接收来自特定主题的消息。
`rclcpp::Publisher<T>`和`rclcpp::Subscription<T>`类分别用于创建发布者和订阅者，其中 T 是消息类型。

### Service（服务）和 Client（客户端）：
除了通过主题发布和订阅消息外，节点还可以通过服务（Service）和客户端（Client）进行请求和响应式通信。
`rclcpp::Service<T>`和`rclcpp::Client<T>`类分别用于创建服务和客户端，其中 T 是服务请求和响应的类型。

### Actions（动作）：
在 ROS 2 中，动作提供了一种更复杂的通信模式，它允许节点执行长时间运行的目标，并提供反馈和取消功能。
`rclcpp::ActionServer<T>`和`rclcpp::ActionClient<T>`类用于创建动作服务器和动作客户端，其中 T 是动作消息的类型。

## 实现
### topic
`create_publisher`
`create_subscription`

### service
`create_service`
`create_client`





# c++14新特性






# 项目中碰到的问题
1. SOME/IP TTL过短，导致重新订阅
2. 多线程`Send()`，接收端数据错乱（表示udp someip-tp消息重组发生错乱）
    尽管注释里说，同一个类非线程安全，不同的类实例线程安全，但是不同的实例应该也不是线程安全的
    因为虽然`Send()`是这样调用的：
`SkeletonEvent`::`SendInternal(Sample const& data)`
```cpp
    std::lock_guard<std::mutex> const impl_interfaces_guard{skeleton_->GetImplInterfacesLock()};

    // Get the binding implementations (backends) that are retrieved on an OfferService of the skeleton object.
    typename Skeleton::SkeletonImplInterfacePtrCollection const& binding_interfaces{
        skeleton_->GetBindingImplInterfaces()};

    // Send the event sample to all the binding implementations via the binding-specific event manager.
    // VCA_SOCAL_VALID_SKELETON_IMPL_INTERFACE_COLLECTION
    for (typename Skeleton::SkeletonImplInterfacePtr const interface : binding_interfaces) {
      result = SendInternal(interface, data);
    }
```
但是`SendInternal`是这样实现的
```cpp
    return (concrete_impl_interface->*GetEventManagerMethod)()->Send(data);
```
不同实例的`SomeipEventManager`（R20-11中命名为`SkeletonEventXf`），可能是在`ApplicationConnection`那层使用到了临界资源：很可能就是存储message header的buffer

3. 总是需要去看源码实现
需要模拟ROS2端，所以要看ROS2的源码实现
**ROS2 as client**
ROS2发请求，网关程序收到，要立马调用AP的method方法，转发请求给提供method服务的app，得到并存储`future`，塞进队列，
有个线程专门取队列元素，
需要轮询每个future是否可取得值，取得值则发回给ROS2端
**ROS2 as server**
someip_to_dds本地需要维护一个`pending_requests_`的map
当APP发送请求给网关程序时，转发请求给ROS2端，注意塞数据时ROS2端要有效（ROS2端是使用2个fastdds的topic来实现的），
给每个请求编号，在method回调里执行如下逻辑：（注意配置someip_to_dds的线程池个数为多个）
{
    去向ROS2发请求
    从`pending_requests_`根据id来取值，没有就阻塞住（相当于阻塞map）
    然后再把response返回
}

4. 使用VLAN

5. 扩展配置：someipd过载保护
设置连续通知的最小时间间隔，在间隔前到来的event被缓冲，过了时间间隔再转发，在时间间隔内来的event只会被转发最后一个。

6. 只看文档来配置是不够的，为了研究配置参照AUTOSAR AP文档进行了开发

7. 通过perf工具生成火焰图发现，正则表达式效率太差，函数调用栈太深



# 看过比较精妙的代码
`std::allocator`内存分配器针对小内存的内存池设计
按照长度大小分类的自由链表，这个是申请小内存块的时候所使用的，因为直接申请小内存块容易产生内存碎片
配置和回收

vector代码
大量的封装、针对不同平台的封装，从而实现兼容
大量的类设计，单一职责
实现了整体的重构，包括`ara::core`








