# 介绍一下vector ap的使用
AUTOSAR定义了超过300个AR-ELEMENT，VECTOR由自定义了一些，对AP模型进行了扩展

## 供应商。。。使用者。。。（？？？）




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
收到`SubscribeEventGroup`时，发送单播Ack/Nack，**启动此`订阅Entry`的TTL计时器**（如果收到`StopSubscribeEventGroup`，则停止计时器）

当以上阶段发送完N次（可配置次数），进入下一阶段
##### Main Phase
除非服务主动下线，否则一般不会再退出
固定周期（可配置）地发送OfferService
期间就算收到了FindService，也要延迟`（可配置的）一段时间`，发单播`OfferService`给服务请求端
如果服务不可用，返回`Down阶段`，并发送`StopOffer`
收到`SubscribeEventGroup`时，发送单播Ack/Nack，**启动此`订阅Entry`的TTL计时器**（如果收到`StopSubscribeEventGroup`，则停止计时器）

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


# DDS服务发现流程
关键字
发现机制、收发机制、QOS机制

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



















# 看过比较精妙的代码
std::allocator内存分配器
自由链表

vector代码




