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
## 几个阶段
### OfferService、FindService
都是多播消息
Offer报文中包含若干个提供的服务（service id, instance id, version, ttl），附加字段里包含ip + port
find报文中包含若干个请求的服务（service id, instance id, version, ttl）

### subscribe, subscribe ack
单播
subscribe报文包含若干个请求订阅的event group（service id, instance id, event group id, version, ttl），附加字段里包含ip + port
subscribe ack报文包含对若干个请求订阅的event group（service id, instance id, event group id, version, ttl）的确认

### 持续发Offer以保活？？？




# DDS服务发现流程
## 1. DomainParticipant发现阶段（PDP）：
在此阶段，`DomainParticipant`参与者确认彼此的存在。
为此，每个DomainParticipant都会定期发送通知消息。
当两个给定的DomainParticipant存在于同一DDS域中时，它们将匹配。
默认情况下，使用已知的`多播地址`和`端口`（使用DomainId计算）发送通知消息。
此外，还可以指定使用单播发送通知的地址列表。
此外，还可以配置此类通知的周期性。



## 2. 端点发现阶段（EDP）：
在此阶段，`DataWriter`和`DataReader`相互确认。
为此，DomainParticipants使用PDP期间建立的通信信道，彼此共享有关其DataWriter和DataReader的信息。
除其他外，此信息还包含`主题`和`数据类型`。
要使两个端点匹配，其主题和数据类型必须一致。
一旦DataWriter和DataReader匹配，它们就可以发送/接收用户数据流量了。


























# 看过比较精妙的代码
std::allocator内存分配器
自由链表

vector代码




