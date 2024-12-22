# 类是怎样逐步创建的
起点是`Runtime`，然后是不同Binding对应的`Initializer`，
解析配置的时候就创建了很多对象了，除了`Proxy`端，要到`find service`成功了才创建`Proxy`对象

## someip
### 在`ara::core::Initialize()`时，做了一些事情


### skeleton
比如，`SomeipBindingInitializer`，持有一个很关键的、唯一的类：
`AraComSomeIpBinding`，直接持有很多关键对象：
比如：
`AraComSomeIpBindingServerManager<SomeIpDaemonClient>`
`AraComSomeIpBindingClientManager<SomeIpDaemonClient>`

对`AraComSomeIpBindingServerManager<SomeIpDaemonClient>`而言，关键的成员类有两个：
skeleton Binding类、SkeletonFactoryContainer

**Binding类**（`核心`）：
对于skeleton，是一个skeleton binding
    如`StartApplicationCmService1_ServiceInterfaceSkeletonSomeIpBinding`
    继承自两个类：
1. ::amsr::someip_binding_transformation_layer::internal::`AraComSomeIpSkeletonInterface`
    Interface class for ara::com SOME/IP skeleton implementations.
    ara::com SOME/IP skeleton实现的接口类
    就定义了一个接口：`HandleMethodRequest(header, packet)`

2. ::startapplication::cm::service1::internal::`StartApplicationCmService1_ServiceInterfaceSkeletonImplInterface`
    Skeleton implementation interface of service 'StartApplicationCmService1_ServiceInterface'
    StartApplicationCmService1_ServiceInterface 这一service的 skeleton实现 的接口类（没有强调SOME/IP）
    定义了接口：`GetEventManagerStartApplicationEvent1()`，使得`SkeletonEvent`通过`binding`类拿到`SomeIpSkeletonEventManager`

**SkeletonFactoryContainer**：
Skeleton
    ！！！和`OfferService`之类的外部接口有关，这里的`OfferService`调用极为复杂，`Skeleton`通过`set<InstanceSpecifierLookupTableEntry>`，拿到`AraComSomeIpBinding`，调`OfferService`，从而调到`AraComSomeIpBindingServerManager`::`OfferService`，从而调到`SomeIpDaemonClient`::`OfferService`

PS: `InstanceSpecifierLookupTableEntry`是一个重要纽带，连接Skelton和`SomeipBindingInitializer`::`AraComSomeIpBinding`，从而获取到`AraComSomeIpBindingServerManager`



#### 对于`event`数据的发送
    skeleton和proxy两端都有`SomeIpSkeletonEventManager` / `SomeIpProxyEventManager`（IPC同理）
    真正的实现都放在`SomeIpSkeletonEventBackend ` / `SomeIpProxyEventBackend`

简单梳理函数调用：
1. `StartApplicationCmService1_ServiceInterfaceSkeleton`（继承自`Skeleton`）对象的`events::StartApplicationEvent1`(`SkeletonEvent<...>`)类型的成员变量，调用`Send(SampleType const& data)`：
    通过`Skeleton`->`GetImplementationInterfaces()`，拿到**最为关键的someip binding类**（怎么拿到的：本质是指向派生类`StartApplicationCmService1_ServiceInterfaceSkeletonSomeIpBinding`的基类`StartApplicationCmService1_ServiceInterfaceSkeletonImplInterface`指针），调接口`GetEventManagerStartApplicationEvent1`拿到`SomeIpSkeletonEventManager`对象（每个event都有一个）
    PS: 因为多重绑定的设定，这里是（通过for循环）对每个binding，调`Send(SampleType const& data)`
2. 得到`SomeIpSkeletonEventManager`对象，调用`Send(SampleType const& data)`
具体实现在`backend`类，这里做的操作主要是**序列化数据**：
```cpp
    // 计算 head_size, payload size
    // 分配相应长度的内存
    // 实例化一个序列化Writer
    // 转发{ instance_id, 序列化后的paclet } 给AraComSomeIpBindingServerManager对象
```
3. `AraComSomeIpBindingServerManager`对象，调用`SendEventNotification(instance_id, packet)`
4. `SomeIpDaemonClient`::`Send(instance_id, packet)`
    PS: 这里为什么要把instance_id，往后传递？？？
        因为要构造someip header，在`ActiveConnection`的实现中，再度序列化：`SetUpSomeIpHeader`，`SerializeRoutingSomeIpMessageHeader`
    ？？？为什么再度序列化？？？
    因为还要一些route阶段才知道该怎么填写的字段，如`Message type`

### proxy
`SomeipBindingInitializer`，持有一个很关键的、唯一的类：
`AraComSomeIpBinding`，直接持有很多关键对象：
`AraComSomeIpBindingClientManager<SomeIpDaemonClient>`，两大关键成员：
proxy Binding类、ProxyFactoryContainer


**Binding类**（`核心`）：
对于proxy，是一个proxy binding
    如`StartApplicationCmService1_ServiceInterfaceProxySomeIpBinding`
    继承自两个类：
1. ::amsr::someip_binding_transformation_layer::internal::`AraComSomeIpProxyInterface`
    Interface class for ara::com SOME/IP proxy implementations.
    ara::com SOME/IP proxy实现的接口类
    就定义了一个接口：`HandleMethodResponse(header, packet)`

2. ::startapplication::cm::service1::internal::`StartApplicationCmService1_ServiceInterfaceProxyImplInterface`
    Proxy implementation interface for the Service 'StartApplicationCmService1_ServiceInterface'
    StartApplicationCmService1_ServiceInterface 这一service的 proxy实现 的接口类（没有强调SOME/IP）
    定义了接口：`GetEventManagerStartApplicationEvent1()`，使得`ProxyEvent`通过`binding`类拿到`SomeipProxyEventManager`

**ProxyFactoryContainer**：
Proxy
    ！！！和`StartFindService`之类的外部接口有关，这里的`StartFindService`调用极为复杂，是`注册回调--事件触发--异步调用`的模式。
    注册`（用户传入的）回调`进static的`FindServiceObserversManager`对象，对服务进行监听，当服务offer时，触发。
    同时使用线程池调一次`find_service_function`：其中调到了static函数：`Proxy::FindService`，入参是`InstanceSpecifierLookupTableEntryContainer`，通过`handle_.GetServiceInstances()`可以得到（这个service_instance是在``Proxy`::`StartFindService`::`ResolveInstanceSpecifierMapping`中得到的，这个函数直接访问`Runtime`了！！！）。

通过`InstanceSpecifierLookupTableEntry`，拿到`AraComSomeIpBinding`，调`FindService`，触发`AraComSomeIpBindingClientManager`::`FindService`，从而调到`SomeIpDaemonClient`::`FindService(someip_service_id, instance。GetInstanceId())`

PS: `InstanceSpecifierLookupTableEntry`是一个重要纽带，连接`Proxy`和`SomeipBindingInitializer`::`AraComSomeIpBinding`，从而获取到`AraComSomeIpBindingClientManager`

#### `订阅`流程



#### 对于`event`数据的接收
`SomeIpProxyEventManager`（IPC同理）
真正的实现都放在`SomeIpProxyEventBackend`

简单梳理函数调用：
同样是：`注册回调--事件触发--异步调用`的模式
1. `StartApplicationCmService1_ServiceInterfaceProxy`（继承自`Proxy`）对象的`events::StartApplicationEvent1`(`ProxyEvent<...>`)类型的成员变量，调用`SetReceiveHandler(function)`进行设置用户回调
而库内部也有注册回调，是`SomeIpDaemonClient`注册到osabstraction::socket上，这个回调`OnSomeIpRoutingMessage(instance_id, receive_memory_buffer)`，调用到了：
```cpp
    client_manager_->HandleReceive(instance_id, someip_header, std::move(someip_message));
```

当接收到event数据时，触发：
`SomeIpDaemonClient::OnSomeIpRoutingMessage(instance_id, receive_memory_buffer)`
    `AraComSomeIpBindingClientManager`::`HandleReceive(instance_id, header, packet)`, `RouteEventNotification(instance_id, header, packet)`     （`AraComSomeIpBindingClientManager`持有`map<someip_event_id, SomeipProxyEventBackend>`）
        转发给相应的`SomeipProxyEventBackend`对象，`SomeipProxyEventBackend`::`OnEvent(packet)`
            `SomeipProxyEventManager`::`HandleEventNotification()`，调用到`ProxyEvent`的`Notify()`
                `EventNotificationTask`被`AddTask`到线程池































# 特点
1. 为了方便src-gen这种代码参与编译，大量使用`template`类的函数不被调用（就不会生成代码、参与编译）就可以引入未实现函数的特性

# 哪些设计模式
























