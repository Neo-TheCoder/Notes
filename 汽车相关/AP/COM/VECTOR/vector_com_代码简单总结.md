# 类是怎样逐步创建的
起点是`Runtime`，然后是**不同Binding**对应的`Initializer`，
解析配置的时候就创建了很多重要对象了！
关键调用：
`SomeipBindingInitializer`::`RegisterServiceInstances` -->  `Runtime`::`MapInstanceSpecifierToInstanceId(binding, instance_specifier, instance_identifier, service_shortname_path)`
`Runtime`::`MapInstanceSpecifierToInstanceId(binding, instance_specifier, instance_identifier, service_shortname_path)`
这个调用颇为关键：创建了`multimap<instance_specifier， { binding, instance_identifier, service_shortname_path }>`
```cpp
void Runtime::MapInstanceSpecifierToInstanceId(
    amsr::socal::internal::BindingInterface* binding, ara::core::InstanceSpecifier const& instance_specifier,
    ara::com::InstanceIdentifier const& instance_identifier,
    amsr::socal::internal::ServiceShortNamePath const& service_shortname_path) noexcept {
  instance_specifier_table_.MapInstanceSpecifierToInstanceId(binding, instance_specifier, instance_identifier,
                                                             service_shortname_path);
}
```
除了`Proxy`端，要到`find service`成功了才创建`Proxy`对象

## someip
### 在`InitializeComponent()`时，构造了一些对象
构造：
* `AraComSomeIpBinding`对象
* `{ServiceInterface}SomeIpSkeletonFactory`
* event_backend：`SomeIpSkeletonEventBackend<{ServiceInterfaceSkeletonSomeipEventConfigEvent1}>`，添加到`static`的`map<instance_id, event_backend>`
* 调接口（怎么个调用法，都是src-gen生成的）：`MapInstanceSpecifierToInstanceId(aracom_someip_binding, instance_specifier, instance_identifier, service_name)`
    `Runtime`维护一个`InstanceSpecifierLookupTable`类型的`instance_specifier_table_`，往里构造对象`<aracom_someip_binding, instance_specifier, instance_identifier, service_name>`
PS: `instance_specifier`和`instance_identifier`是一对多的关系（很多时候是一对一），后者会显露具体的binding



### skeleton
比如，`SomeipBindingInitializer`，持有一个很关键的、唯一的类：
`AraComSomeIpBinding`，直接持有很多关键对象：
比如：
`AraComSomeIpBindingServerManager<SomeIpDaemonClient>`
`AraComSomeIpBindingClientManager<SomeIpDaemonClient>`

#### 对`AraComSomeIpBindingServerManager<SomeIpDaemonClient>`而言，关键的成员类有两个：
skeleton Binding类、SkeletonFactoryContainer

##### **Binding类**（`核心`）：
对于skeleton，是一个`skeleton binding`
    如`StartApplicationCmService1_ServiceInterfaceSkeletonSomeIpBinding`
    继承自两个类：
1. ::amsr::someip_binding_transformation_layer::internal::`AraComSomeIpSkeletonInterface`
    > Interface class for ara::com SOME/IP skeleton implementations.
    > ara::com SOME/IP skeleton实现的接口类
    就定义了一个接口：`HandleMethodRequest(header, packet)`

2. ::startapplication::cm::service1::internal::`StartApplicationCmService1_ServiceInterfaceSkeletonImplInterface`
    > Skeleton implementation interface of service 'StartApplicationCmService1_ServiceInterface'
    > StartApplicationCmService1_ServiceInterface 这一service的 skeleton实现 的接口类（没有强调SOME/IP）
    定义了接口：`GetEventManagerStartApplicationEvent1()`，使得`SkeletonEvent`通过`binding`类拿到`SomeIpSkeletonEventManager`

##### **SkeletonFactoryContainer**：
Skeleton
    ！！！和`OfferService`之类的外部接口有关，这里的`OfferService`调用极为复杂，`Skeleton`通过`set<InstanceSpecifierLookupTableEntry>`，拿到`AraComSomeIpBinding`，调`OfferService`，从而调到`AraComSomeIpBindingServerManager`::`OfferService`，从而调到`SomeIpDaemonClient`::`OfferService`

PS: `InstanceSpecifierLookupTableEntry`是一个重要纽带，连接Skelton和`SomeipBindingInitializer`::`AraComSomeIpBinding`，从而获取到`AraComSomeIpBindingServerManager`



#### `OfferService`
##### 先看`Skeleton`的构造
###### 拿到`ConstructToken`
`user_namespace::Preconstruct(instance_specifier)` -> `src_gen::Preconstruct(instance_specifier, MethodCallProcessingMode::kEvent)` --> `socal::Base::PreConstruct(instance, mode)` -> `socal::Base::PreConstruct(instance_container, threrad_pool_id, mode)` -> 
（中途变成`instance_container`是因为多重绑定）
`socal::Base::PreConstruct(instance, mode)`调用`ConstructEntryContainerFromSpecifier(instance)（这里）`：（！！！关键`Runtime::ResolveInstanceIDs`）从`Runtime`那里得到`instance_specifier_table`，根据入参`instance_id`进行查找，如果有，则把结果拷贝到Skeleton类中定义的`static`的`std::set<InstanceSpecifierLookupTableEntry>`，最后返回**`ConstructToken`对象**：
```cpp
using ConstructionResult = ::ara::core::Result<ConstructionToken>;
// ...
return ConstructionResult::FromValue(offered_instances, thread_pool_id, mode);
```

###### 拿到`ConstructToken`后构造
    使用初始化列表：构造基类部分，成员变量

##### ``OfferService`的实现
在构造`Skeleton`时，通过上述`Preconstruct`返回的`ConstructToken`，调这个token的`ConsumeOfferedInstances()`方法，得到`std::set<InstanceSpecifierLookupTableEntry>`类型的`instance_container`：
```cpp
    for (InstanceSpecifierLookupTableEntry const& offered_instance : offered_instances_) {
        SkeletonImplPtr skeleton_implementation{offered_instance.GetBinding()->OfferService(
            kServiceIdentifier, offered_instance.GetInstanceIdentifierString(), *this)};
    // ...
    }
```
PS: 这里的`kServiceIdentifier`也是不区分具体binding的
针对当前的service，对每一个`offered_instance`调用`GetBinding()->OfferService(kServiceIdentifier, offered_instance.GetInstanceIdentifierString(), *this)`
PS: 这里传进去的this指针，肯定是最调用对象（即最外层派生类）的地址（不可能把继承关系截断）
从而调到`AraComSomeIpBindingServerManager`::`OfferService(instance, skeleton)`
    （此处，通过传进来的`instance`得到：`someip_instance_id`；通过传进来的`skeleton.GetInstanceId()`得到：`skeleton_id`
    而`AraComSomeIpBindingServerManager`内部维护一个`skeleton_factories`，通过`factory->GetSkeletonId()`查找skeleton_id相同的），查找成功，则调用`someip_posix_.OfferService(someip_service_id, someip_instance_id);`，当该函数返回成功，则调用以下接口来构造`skeleton_someip_binding`对象（！！！）：
```cpp
        result = it->get()->create(instance.GetInstanceId(), skeleton);     // 构造skeleton_someip_binding对象
```
从而调到`SomeIpDaemonClient`::`OfferService`
PS: 每个`Skeleton`都有一个`static`的`skeleton_id_`（这个对象的类型仅仅用于比较是否相等）
而`someip_instance_id`是`std::uint16_t`类型的
而`factory`实际是`AraComSomeIpSkeletonFactory<ServiceInterfaceSkeleton, ServiceInterfaceSkeletonSomeIpBinding>`类型的，直接通过第一个模板参数拿到`skeleton_id`，通过第二个模板实参可以拿到`service_id`
以上`GetBinding()->OfferService(kServiceIdentifier, offered_instance.GetInstanceIdentifierString(), *this)`返回指向具体派生的`skeleton_binding`的基类指针。
    然后使用这行代码进行指针类型转换：（把socal层的基类指针，转为了src-gen中的基类指针，多个`GetEventManager`的纯虚函数，然后把这个升级过的指针塞进`Skeleton所维护的binding_instance_implementations_`）
```cpp
    ConcreteSkeletonImplInterface* concrete_impl_interface{
        dynamic_cast<ConcreteSkeletonImplInterface*>(skeleton_implementation.release())};
```




#### 对于`event`数据的发送
skeleton和proxy两端都有`SomeIpSkeletonEventManager` / `SomeIpProxyEventManager`（IPC同理）
真正的实现都放在`SomeIpSkeletonEventBackend ` / `SomeIpProxyEventBackend`

简单梳理函数调用：
1. `StartApplicationCmService1_ServiceInterfaceSkeleton`（继承自`Skeleton`）对象的`events::StartApplicationEvent1`(`SkeletonEvent<...>`)类型的成员变量，调用`Send(SampleType const& data)`：
    通过`Skeleton`->`GetImplementationInterfaces()`，拿到**最为关键的`skeleton someip binding`类**（怎么拿到的：本质是指向派生类`StartApplicationCmService1_ServiceInterfaceSkeletonSomeIpBinding`的基类`StartApplicationCmService1_ServiceInterfaceSkeletonImplInterface`指针），调接口`GetEventManagerStartApplicationEvent1`拿到`SomeIpSkeletonEventManager`对象（每个event都有一个）
    PS: 因为多重绑定的设定，这里是通过`socal::Skeleton`，拿到`src-gen中的指向skeleton具体派生binding对象的基类指针`，这个指针实际指向的最上层对象实现了`GetEventManager`方法，拿到相应的`EventManager`
    然后，（通过for循环）对每个binding，调`Send(SampleType const& data)`


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



#### 对于`method`数据的处理






### proxy
#### `订阅`流程
##### find service流程
注意，用户操作的Proxy对象要find service成功了，才创建，对于多重绑定，`StartFindService(function, instance_specifier)`，这里的function的函数签名是`入参是ServiceHandleContainer<HandleType>，函数体是FindService1Handler(handles);`，也就是说，会有多个proxy对象产生？用户通过instance_identifier进行区分
Proxy对象的构造和proxy_someip_binding对象的构造如何联系在一起？
Proxy类持有成员：
```cpp
    std::shared_ptr<ProxyImplInterface> proxy_impl_interface_ptr_;  // 这里的ProxyImplInterface类型和下面Skeleton中的情况一样，是src-gen层面的一个抽象类（这个类和具体binding无关），比最底层的基类高一级别
```

PS：Proxy端对这个指针还提供了对外访问接口`GetServiceProxyImplInterface()`，返回这个`share_ptr<interface>`的拷贝

PS: 在Skeleton端是`std::vector<std::unique_ptr<ConcreteSkeletonImplInterface>> binding_instance_implementations_;`
    也相应地提供了访问指针的接口`GetImplementationInterfaces`，返回是`std::vector<std::unique_ptr<ConcreteSkeletonImplInterface>>&`
！！！skeleton这里使用**unique_ptr**，是因为不需要拷贝指针，也不需要转移所有权（感觉普通的引用即可）
      proxy这里使用**shared_ptr**，是因为`proxy_method`的实现需要拷贝shared_ptr：可能是为了延长生命周期？？？
      proxy端的多重绑定是每一个绑定，对应一个`instance_identifier`，对应一个`StartApplicationCmService1_ServiceInterfaceProxy`类型的指针

`proxy_someip_binding`的构造
```cpp
  static std::shared_ptr<ProxyImplInterface> ConstructInterface(HandleType const& handle) noexcept {
    // ...
        std::shared_ptr<amsr::socal::internal::ProxyImplInterface> const proxy_interface{
          factory_interface->Create(resolve_result.Value().GetInstanceIdentifierString())};
    // ...
  }
```
要注意这里的`handle`是怎么来的：
    需要观察find service的链路：能够调用到`AracomClientManager`
```cpp
  static ara::com::ServiceHandleContainer<HandleType> FindService(
      InstanceSpecifierLookupTableEntryContainer const& service_instances) noexcept {
    ara::com::ServiceHandleContainer<HandleType> handles{};
    try {
      // Call FindService() of the related binding
      for (InstanceSpecifierLookupTableEntry const& service_instance : service_instances) {
        ara::com::ServiceHandleContainer<InstanceHandle> const instance_handles{
            service_instance.GetBinding()->FindService(
                kServiceIdentifier, service_instance.GetInstanceIdentifierString(), GetProxyIdStatic())};

        // Convert the instance handles into HandleType objects
        for (InstanceHandle const& instance_handle : instance_handles) {
          handles.emplace_back(service_instance.GetInstanceIdentifier(), instance_handle.GetProxyFactory());
        }
      }
    }  // VECTOR Next Line AutosarC++17_10-A15.3.4: MD_SOCAL_AutosarC++17_10-A15.3.4_Caught_exception_is_too_general
    catch (std::exception const& e) {
      ::ara::core::Abort(e.what());
    }
    // VECTOR NC AutosarC++17_10-A15.3.4: MD_SOCAL_AutosarC++17_10-A15.3.4_Using_catch_all
    // VECTOR NC AutosarC++17_10-M0.3.1: MD_SOCAL_AutosarC++17_10-M0.3.1_Dead_exception_handler
    catch (...) {
      ::ara::core::Abort("Proxy::FindService: Unknown exception.");
    }
    return handles;
  }
```

`SomeipBindingInitializer`，持有一个很关键的、唯一的类：
`AraComSomeIpBinding`，直接持有很多关键对象：
#### `AraComSomeIpBindingClientManager<SomeIpDaemonClient>`，两大关键成员：
proxy Binding类、ProxyFactoryContainer


##### **Binding类**（`核心`）：
对于proxy，是一个proxy binding
    如`StartApplicationCmService1_ServiceInterfaceProxySomeIpBinding`
    继承自两个类：
1. ::amsr::someip_binding_transformation_layer::internal::`AraComSomeIpProxyInterface`
    > Interface class for ara::com SOME/IP proxy implementations.
    > ara::com SOME/IP proxy实现的接口类
    就定义了一个接口：`HandleMethodResponse(header, packet)`

2. ::startapplication::cm::service1::internal::`StartApplicationCmService1_ServiceInterfaceProxyImplInterface`
    > Proxy implementation interface for the Service 'StartApplicationCmService1_ServiceInterface'
    > StartApplicationCmService1_ServiceInterface 这一service的 proxy实现 的接口类（没有强调SOME/IP）
    定义了接口：`GetEventManagerStartApplicationEvent1()`，使得`ProxyEvent`通过`binding`类拿到`SomeipProxyEventManager`

##### **ProxyFactoryContainer**：
Proxy
    ！！！和`StartFindService`之类的外部接口有关，这里的`StartFindService`调用极为复杂，是`注册回调--事件触发--异步调用`的模式。
    注册`（用户传入的）回调`进static的`FindServiceObserversManager`对象，对服务进行监听，当服务offer时，触发。
    同时使用线程池调一次`find_service_function`：其中调到了static函数：`Proxy::FindService`，入参是`InstanceSpecifierLookupTableEntryContainer`，通过`handle_.GetServiceInstances()`可以得到（这个service_instance是在``Proxy`::`StartFindService`::`ResolveInstanceSpecifierMapping`中得到的，这个函数直接访问`Runtime`了！！！）。

通过`InstanceSpecifierLookupTableEntry`，拿到`AraComSomeIpBinding`，调`FindService`，触发`AraComSomeIpBindingClientManager`::`FindService`，从而调到`SomeIpDaemonClient`::`FindService(someip_service_id, instance。GetInstanceId())`

PS: `InstanceSpecifierLookupTableEntry`是一个重要纽带，连接`Proxy`和`SomeipBindingInitializer`::`AraComSomeIpBinding`，从而获取到`AraComSomeIpBindingClientManager`





#### `订阅`流程




#### 对于`event`数据的接收
注意，这里老版本VECTOR需要在proxy端主动调用`Update()`方法！！！
    ？？？更新什么？？？

因为proxy端收到数据时，是采用调用用户的回调的模式，是异步的
当某一种binding的数据到达，则触发相应binding的event的`receivehandler`
    不管是`IpcProxyEventManager`还是`SomeipProxyEventManager`都会持有：
```cpp
    ::amsr::socal::internal::ProxyEventBindingInterface* service_event_{};  // 在用户层ProxyEvent::Subscribe(policy, cache_size)
    
    // SomeipProxyEventManager::Subscribe深层调用
    Subscribe(ProxyEventBindingInterface*, cache_size); // 这里的ProxyEventBindingInterface*是通过ProxyEvent的proxy_ptr_拿到具体proxy binding的src-gen层的抽象类指针，调用GetEventManagerMethod方法拿到的
```

`SomeIpProxyEventManager`（IPC同理）
真正的实现都放在`SomeIpProxyEventBackend`

简单梳理函数调用：
同样是：`注册回调--事件触发--异步调用`的模式
1. `StartApplicationCmService1_ServiceInterfaceProxy`（继承自`Proxy`）对象的`events::StartApplicationEvent1`(`ProxyEvent<...>`)类型的成员变量，调用`SetReceiveHandler(function)`进行设置用户回调

而**库内部也有注册回调**，是`SomeIpDaemonClient`注册到`osabstraction::socket`上
    这个回调：`OnSomeIpRoutingMessage(instance_id, receive_memory_buffer)`，调用到了：
```cpp
    client_manager_->HandleReceive(instance_id, someip_header, std::move(someip_message));
```

2. 当接收到event数据时，
    在ProxyEvent中（对于ipc和someip的binding，各有一个实例）
    通过指向`Proxy`的指针，拿到指向`proxy_someip_binding派生类`的指针，拿到event manager对象

连起来看，对于someip_binding，触发：
`SomeIpDaemonClient::OnSomeIpRoutingMessage(instance_id, receive_memory_buffer)`
    `AraComSomeIpBindingClientManager`::`HandleReceive(instance_id, header, packet)`, `RouteEventNotification(instance_id, header, packet)`
    （`AraComSomeIpBindingClientManager`持有`map<someip_event_id, SomeipProxyEventBackend>`）
        转发给相应的`SomeipProxyEventBackend`对象，`SomeipProxyEventBackend`::`OnEvent(packet)`
            `SomeipProxyEventManager`::`HandleEventNotification()`，调用到`ProxyEvent`的`Notify()`
                `EventNotificationTask`被`AddTask`到线程池







# 特点
1. 为了方便src-gen这种代码参与编译，大量使用 “`template`类的函数不被调用（就不会生成代码、参与编译）就可以引入未实现函数” 的特性
2. 由于多重绑定的存在，用户层对socal调用：看似简单的`OfferService`，`Send`等操作，实际上都要遍历每一种binding的对象，调用下一层的接口
    常见场景：
    一个proxy对应两种binding的skeleton，这里如果有someip binding的skeleton，则是跨ECU的
    并且典型场景下，这里的someipd需要实现端口复用（详情见aracom_api文档）

3. 第2点是核心思想是：依赖抽象/接口，而不是细节/具体实现
    用户层直接只管调用`socal`层的接口就完事了，底层binding实现要考虑的事情就多了




# 哪些设计模式













