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

### vector层
socal层、具体binding层
socal层定义了一些绑定无关的抽象类/接口类：如Skeleton、SkeletonEvent
核心单例：`Runtime`持有关键对象`map<specifier, binding类>`，这里的binding类象征着所使用的绑定
`具体binding类`持有`XXXBindingServerManager`和`XXXBindingClientManager`，这两个类和具体协议相关，调用具体协议相关的接口，收发数据
`XXXBindingServerManager`会持有`XXXSkeletonEventManager`，`XXXSkeletonMethodManager`；
`XXXBindingClientManager`会持有``

### 用户层
`intance_specifier`（字符串拼接，port的名字）/ `instance_identifier`（包含具体binding），二者是一对多的关系（存在多重绑定的概念）
进行service匹配（）

用户先进行服务配置（event、method、field（？？？对应什么样子的数据？？？）），然后根据需求进行服务部署，选择具体的绑定、部署的平台（一些可能的端口配置）
调用`ara::core::Initialize()`，进行统一的初始化
    先读取配置文件，以单例方式实例化一些配置相关的对象，关键类是`map<specifier, binding类>`

根据所选择的binding，生成`static` `XXXBindingInitialzier`对象（这些类的成员函数的实现在src-gen）

src-gen中包含关键binding类，如果是Skeleton端，就是`SkeletonBinding`，或者是`ProxyBinding`

**AUTOSAR规定**
需要生成出用户可以直接继承的skeleton类，proxy类则不同，先只持有指针，调用FindService静态函数（返回一个service_handle），在回调中构造Proxy对象，赋值给指针
用户可以直接访问到代表event、method、field的对象



## per





# AUTOSAR AP
## com
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

### proxy
1. event
用户层调`SetReceiveHandler`，这个回调实际上放在`on_data_available`里调用

2. method
用户层直接同步调用`operator()`，`SetRequestData()`，promise存储到`pendingRequests_`，返回future对象（对应的Promise的set是在`on_data_available()`）



# powerap
## com
### 关键词
多态、继承、模板、胶水代码
根据模型生成中间文件（.camke）、链接不同的库

对于dds binding而言
    arxml提取数据类型信息生成idl文件，调用fastddsgen

整体来说，就是AUTOSAR AP规定了基于service通信的接口，其中提供不同的绑定（甚至可以多重绑定），让用户无感知地调用接口，想要换绑定，只要改配置文件就行了，达到一种这样的效果




### 架构设计
`DDSSkeletonServiceMananger`，`DDSProxyServiceMananger`（存在多个实例）
工厂





### 业务逻辑




#### OfferService




#### FindService









## per
### 关键词


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
默认大小 preallocated，当更大的消息到达时需要重新分配。

Smaller memory footprint at the cost of an increased allocation count.
内存占用更少，但代价是分配计数增加。


#### best effort
减少重传和确认机制


#### reliable
使用了消息确认机制


## 收数据的接口
`on_data_available`是transport层的接口
一个channel对应一个线程
udp的实现：（使用`asio`了）
```cpp
while() {
    //  ...
    receive_from()
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

## 文档介绍
# Intra-process delivery
包含若干要求，每个要求都占有一个章节。

## 识别本地端点
相同进程中端点的`GUID`前缀的前8个八位字节内容相同。

### GUID_t重构
* 提供方法`is_builtin()`、`is_on_same_process_as(other_guid)`和`is_on_same_host_as(other_guid)`
* 考虑其他改进
    * 将`Guid.h`拆分为三个头文件（guid、prefix和entity_id）
    * 将EntityId_t转换为uint32_t
    * 在GuidPrefix_t上使用联合以高效比较其部分

### 获取指向本地端点的指针
* 在RTPSDomain上添加方法，根据其guid返回`reader/writer`的指针
* 应意识到内置端点的存在

## 进程内接收
订阅者上的新数据处理由写入数据的线程执行。
此线程负责将`CacheChange_t`复制到`ReaderHistory`并调用`NotifyChanges()`。
这一决定是为了与传输层来的当前运行的接收线程操作方式保持一致。

用户文档应明确指出，注册`listener`以获取`新更改信息`意味着同时阻塞了传输接收线程和写入数据的线程。
还应解释另一种读取样本的方法是使用用户线程中的`wait_for_unread_samples()`函数，然后进行读取。

当前的Reader API将被本地写入者用于数据传递。本地写入者将直接调用`processDataMsg`。

## Reader：本地写入者的管理
在匹配写入者时，Reader应检查是否在同一进程中。

**销毁时的考虑**
直到现在，Reader可以安全地被销毁，因为我们确信不会有线程访问它。这可能是因为：
* 使用Reader的所有事件首先被销毁。因此，事件线程永远不会访问Reader。
* Reader从`ReceiverResource`对象中注销。因此任何接收线程都不会再访问Reader。

现在，Reader可能会被本地进程中的写入者访问。但是只有当通过发现机制与写入者匹配时才会发生这种情况。因此，在Reader被销毁之前，我们应该确保使用发现机制与写入者取消匹配。由于EDP内置端点将使用进程内机制，这种取消匹配将是即时的。

### 有状态读取器（StatefulReader）
对本地写入者的输出流量不应执行。
在WriterProxy上的更改：
remote_locators_shrinked 应对本地写入者返回一个空向量。
定时事件“心跳响应”对于本地写入者不应启动。
定时事件“初始acknack”应直接调用本地写入者的process_acknack方法。

## 写入者：本地读取器的管理
在匹配到读取器时，写入者应该检查它是否位于同一进程中。

不应对本地读取者执行输出流量。
样本通过processDataMsg发送给本地读取者。
当使用transient_local持久性时，一旦本地读取器被匹配，它应当自动通过processDataMsg接收数据。

### 无状态写入者（StatelessWriter）
在ReaderLocator::start时，如果读者是本地的，则不添加定位符。

### 有状态写入者（StatefulWriter）
发送给本地读取者的样本会自动确认。
发送给本地读取者的Gaps应直接调用processGapMsg。
发送给本地读取者的心跳信息应直接调用processHeartbeatMsg。
对于本地读取者，定时事件不应启动，除了“初始心跳”。

## 发现过程
### 参与者发现(PDP)
为了在同一进程中保持域的分离，我们将继续使用标准机制进行参与者发现。内置PDP读取者和写入者的内部进程交付将不会被使用。

### 端点发现(EDP)
端点发现也将依赖标准机制，但是内置端点将会使用新的机制来发送/接收WriterProxyData和ReaderProxyData到/从本地进程中。

## 其他考虑事项
### 安全性
如果我们依靠比较GUID来识别同一进程中的端点，那么那些属于安全参与者的将不会被考虑在内，因为在这种情况下，GUID将使用整个参与者GUID的哈希重新计算。

### 活跃度（Liveliness）
活跃度声明机制不需要任何更改，只是manual_by_topic声明应该直接调用本地读取者的processHeartbeatMsg方法。

### 副作用
此设计隔离了所有来自网络的消息流量，这意味着像Wireshark™这样的工具将变得无用。考虑到我们的大多数客户实际上运行的是进程内代码，并且使用Wireshark™跟踪报告他们的问题，这些变化会造成支持噩梦。可能的解决办法是提供一个标志以抑制调试目的的过程内行为，并明确指出，在性能测试期间应关闭该标志。
一旦为进程间通信开发了共享内存传输，进程内的机制将变得无关紧要。因为速度增益和源复杂性的权衡将非常昂贵。从内存消耗的角度来看，使用进程间发现数据库比拥有每个进程的发现数据库会有更多优势。



## `write()`如何调用到`serialize()`？？？
可以还原类型
`DataWriterImpl`可以拿到`TypeSupport`（继承自`shared_ptr`），由`Participant`持有
序列化的数据，若序列化失败，塞到`payload_pool`
成功则塞进`history_`（由`DataWriterImpl`持有）

### fastbuffer


## 接收端如何被通知？




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
ROS2 rmw通信层也是默认使用fastdds，只要someip_to_dds的配置与ROS2端保持一致，理论上就能够通信。
arxml -- idl保持一致，执行运行时的数据拷贝。
ROS2的`topic`映射到`fastdds`，会含有`rt`字段
ROS2的`service`映射到`fastdds`，会含有`rq`字段

单例、设置回调
一个`DomainParticipant`
一个`Publisher`
一个`Subscriber`

若干组`Topic` 和 `TypeSupport`
若干`DataReaderListener`

### 性能
camera Image数据，20几Hz
其他数据上百Hz，也能保证收发频率一致

# recorder
debug模式（？？？分钟级别，一分钟一个文件，不接受trigger） + product模式（？？？秒级别，一秒钟一个文件，接受trigger）两种模式可以切换
类似于行车记录仪，自动删除最老的数据

生产者消费者模型、阻塞队列
线程池
mcap（序列化方式）
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
根据topic，大量生成如下代码：用于`McapWriter`
```cpp
// 需要读取一些.msg文件（ros2所使用的），目的是可供ROS2环境使用、支持plotjuggler    而rosbag play 跟 plotjugger 对schema的data的处理不一样。
schema_test_->name = "sensor_msgs/msg/Image";
schema_test_->encoding = "ros2msg";                 // schema的encoding
auto [format_test, full_text_test] = msgdef_cache.get_full_text(schema_test_->name);                    // 读取.msg文件
schema_test_->data.assign(reinterpret_cast<const std::byte*>(full_text_test.data()),                    // 将消息定义转换成字节流
    reinterpret_cast<const std::byte*>(full_text_test.data() + full_text_test.size()));
writer_->addSchema(*schema_test_);

channel_test_->topic = "/sensor/camera/front/h264";
channel_test_->messageEncoding = "cdr";             // channel的encoding
channel_test_->schemaId = schema_test_->id;
channel_test_->metadata.emplace("offered_qos_profiles", QosToString(TOPIC_QOS_DEFAULT));
available_channels_.push_back(channel_test_);

for (auto channel : available_channels_) {
writer_->addChannel(*channel);
}
/*
    收到数据时，进行两层格式转换：
    1. AP格式的数据   -->  DDS格式的数据
    2. DDS格式的数据  -->  Mcap的msg  需要填充信息
        ConvertToMsg()
*/
```

存储池  ？？？
切片    ？？？debug特有
/tmp下的内存中的文件，最后会被写入磁盘

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
数据的收集（根据收集文件的类型，包装成相应任务类，尽量均匀地完成文件的拷贝，`copy_file_range(src_fd, des_fd, size)`）、压缩（`gzip`）、上传（`poco`）
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



# c++14新特性



# 看过比较精妙的代码
`std::allocator`内存分配器
按照长度大小分类的自由链表

vector代码




