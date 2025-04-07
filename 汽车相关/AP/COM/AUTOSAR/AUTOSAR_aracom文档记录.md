# 2 Introduction
Why did AUTOSAR invent yet another communication middleware API/technology, while there are dozens on the market — the more so as one of the guidelines of Adaptive Platform was to reuse existing and field proven technology?
为什么 AUTOSAR 要发明另一种通信中间件 API/技术，而市场上已有几十种，更何况自适应平台的指导方针之一就是重复使用现有的、经过现场验证的技术？

Before coming up with a new middleware design, we did evaluate existing technologies, which — at first glance — seemed to be valid candidates. Among those were:
在提出新的中间件设计方案之前，我们对现有技术进行了评估，乍一看，这些技术似乎都是有效的候选技术。其中包括
* ROS API
* DDS API
* CommonAPI (GENIVI)
* DADDY API (Bosch)

The final decision to come up with a new and AUTOSAR-specific Communication Management API was made due to the fact, that not all of `our key requirements` were met by existing solutions:
* **We need a `Communication Management`, which is NOT bound to a concrete network communication protocol.**
    It has to support `the SOME/IP protocol` but there has to be flexibility to exchange that.
* The AUTOSAR service model, which defines services as a collection of provided methods, events and fields shall be supported naturally/straight forward.
之所以最终决定开发新的 AUTOSAR 专用通信管理 API，是因为现有解决方案无法满足我们的所有关键要求：
* 我们需要一个**不受具体网络通信协议约束**的 “通信管理”。
    它必须支持 “SOME/IP 协议”，但必须具有改变协议的灵活性。
* 应自然/直接支持 AUTOSAR 服务模型，该模型将服务定义为所提供方法、事件和字段的集合。
* The API shall support an `event-driven` and a `polling` model to get access to communicated data equally well. The latter one is typically needed by real-time ap-
plications to avoid unnecessary context switches, while the former one is much more convenient for applications without real-time requirements.
**应用程序接口应支持 “事件驱动 ”和 “轮询 ”模式，以同样好地访问通信数据。**
**`实时应用程序`通常需要后一种模式，以避免不必要的上下文切换，而前一种模式对于没有实时要求的应用程序来说要方便得多。**

* Possibility for seamless integration of `end-to-end` protection to fulfill ASIL requirements.
可实现端到端保护的无缝集成，以满足`ASIL`要求。
* Support for static (preconfigured) and dynamic (runtime) selection of service instances to communicate with.
* 支持静态（预配置）和动态（运行时）选择服务实例进行通信。

So in the final ara::com API specification, the reader will find concepts (which we will describe in-depth in the upcoming chapters), which might be familiar for him from technologies, we have evaluated or even from the existing Classic Platform:
* Proxy (or Stub) / Skeleton approach (CORBA, Ice, CommonAPI, Java RMI, ...)
* Protocol independent API (CommonAPI, Java RMI)
* Queued communication with `configurable receiver-side caches` (DDS, DADDY, Classic Platform)
* Zero-copy capable API with possibility to shift `memory management` to the middleware (DADDY)
* Data reception filtering (DDS, DADDY)
因此，在最终的 ara::com API 规范中，读者会发现一些概念（我们将在接下来的章节中进行深入描述），这些概念可能是我们已经评估过的技术或现有经典平台中的技术：
* 代理（或存根）/骨架方法（CORBA、Ice、CommonAPI、Java RMI...）
* 独立于协议的 API（CommonAPI、Java RMI）
* 具有 “可配置接收端缓存 ”的队列通信（DDS、DADDY、经典平台）
* 零拷贝 API，可将 “内存管理” 转移给中间件（DADDY）
* 数据接收过滤（DDS、DADDY）

Now that we have established the introduction of a new middleware API, we go into the details of the API in the following chapters.
The following statement is the basis for basically all `AUTOSAR AP specifications`, but should be explicitly pointed out here again:
**ara::com only defines `the API signatures` and `its behavior visible to the application developer`. Providing an implementation of those APIs and the underlying middleware transport layer is the responsibility of the AUTOSAR AP vendor.**
**ara::com仅定义了 “API签名 ”和 “应用程序开发人员可见的行为”。提供这些 API 和底层中间件传输层的实现是 AUTOSAR AP 供应商的责任。**
For a rough parallel with the AUTOSAR Classic Platform, ara::com can be seen as fulfilling functional requirements in the Adaptive Platform similar to those covered in the Classic Platform by the RTE APIs [1] such as Rte_Write, Rte_Read, Rte_Send, Rte_Receive, Rte_Call, Rte_Result.
既然我们已经确定了新的中间件应用程序接口（API）的引入，我们将在接下来的章节中详细介绍该应用程序接口。
以下声明基本上是所有 “AUTOSAR AP 规范 ”的基础，但应在此再次明确指出：
与 AUTOSAR 经典平台大致相同，ara::com 在自适应平台中满足的功能要求与经典平台中 RTE API [1] 所涵盖的功能要求类似，如 Rte_Write、Rte_Read、Rte_Send、Rte_Receive、Rte_Call、Rte_Result。

# 4 API Design Visions and Guidelines
One goal of the API design was to have it `as lean as possible`(尽可能精简). Meaning, that it should only provide `the minimal set` of functionality needed to support the service based com-munication paradigm consisting of the basic mechanisms: methods, events and fields.
Our definition of the notion "as lean as possible" in this context means: Essentially the API shall only deal with the functionality to handle method, field and event communication on service consumer and service provider implementation side.
**在这种情况下，我们对 “尽可能精简 ”这一概念的定义是指 从根本上说，应用程序接口应只处理`服务消费者`和`服务提供者`实施方处理方法、字段和事件通信的功能。**（总之就是和服务紧密相关）
If we decided to provide a bit more than just that, then the reason generally was "If solving a certain communication-related problem ABOVE our API could not be done efficiently, we provide the solution as part of ara::com API layer."
如果我们决定提供比这更多一些的功能，那么理由一般是 “如果在我们的应用程序接口之上无法高效地解决某个通信相关问题，我们会将该解决方案作为 ara::com 应用程序接口层的一部分提供”。

Consequently, ara::com does not provide any kind of component model or framework, which would take care of things like component life cycle, management of program flow or simply setting up ara::com API objects according to the formal component description of the respective application.
因此，ara::com 并不提供任何类型的组件模型或框架，用于处理组件生命周期、程序流程管理或简单地根据各自应用程序的正式组件描述设置 ara::com API 对象等问题。

All this could be easily built on top of the basic ara::com API and needs not be standardized to support typical collaboration models.
During the design phase of the API we constantly challenged each part of our drafts, whether it would allow for efficient IPC implementations from AP vendors, since we were aware, that you could easily break it already on the API abstraction level, making it hard or almost impossible to implement a well performing binding.
**One of the central design points was — as already stated in the introduction — to support polling and event-driven programming paradigms equally well.**
So you will see in the later chapters, that the application developer, when using ara::com is free to chose the approach, which fits best to his application design, independent whether he implements the service consumer or service provider side of a communication relation.
所有这些都可以很容易地构建在基本的 ara::com 应用程序接口之上，不需要标准化就能支持典型的协作模型。
在应用程序接口的设计阶段，我们不断对草案的每一部分提出质疑，因为我们意识到，在应用程序接口的抽象层次上，你可以很容易地破坏它，从而很难或几乎不可能实现性能良好的绑定。
正如前言所述，核心设计要点之一是同样**支持轮询和事件驱动编程范例。**
因此，在后面的章节中，您将看到应用程序开发人员在使用 ara::com 时，可以自由选择最适合其应用程序设计的方法，而不管他是实现通信关系中的服务消费者还是服务提供者一方。

This allows for support of `strictly real-time scheduled applications`, where the application requires total control of what (amount) is done when and where unnecessary context switches are most critical.
**On the other hand the more relaxed event based applications, which simply want to get notified whenever the communication layer has data available for them is also fully supported.**
The decision within AUTOSAR to genuinely support C++11/C++14 for AP was a very good fit for the ara::com API design.
For enhanced usability, comfort and a breeze of elegance ara::com API exploits C++ features like smart pointers, template functions and classes, proven concepts for asynchronous operations and reasonable operator overloading.
这样就可以支持 “严格的实时计划应用程序”，在这种情况下，应用程序需要完全控制何时完成哪些工作（数量），而且不必要的上下文切换是最关键的。
另一方面，AUTOSAR 也完全支持较为宽松的基于事件的应用程序，这些应用程序只希望在通信层有数据可用时获得通知。
AUTOSAR 决定真正支持 C++11/C++14 AP，这与 ara::com API 的设计非常契合。
为了提高可用性、舒适性和优雅性，ara::com API 利用了 C++ 的各种特性，如智能指针、模板函数和类、异步操作的成熟概念以及合理的操作符重载。



# 5 High Level API Structure
## 5.1 Proxy/Skeleton Architecture
If you’ve ever had contact with middleware technology from a programmer’s perspective, then the approach of a Proxy/Skeleton architecture might be well known to you.
Looking at the number of middleware technologies using the Proxy/Skeleton (sometimes even called Stub/Skeleton) paradigm, it is reasonable to call it the "classic approach".
So with ara::com we also decided to use this classical Proxy/Skeleton architectural pattern and also name it accordingly.
如果你曾经从程序员的角度接触过中间件技术，那么代理/骨架（Proxy/Skeleton）体系结构的方法你可能已经耳熟能详了。
从使用代理/骨架（有时甚至称为存根/骨架）范例的中间件技术的数量来看，称其为 “经典方法 ”是有道理的。
因此，在 ara::com 中，我们也决定使用这种经典的**代理/骨架架构模式**，并将其命名为 ara::com。

The basic idea of this pattern is, that from a formal `service definition` two code artifacts are generated:
* Service Proxy: This code is - from the perspective of the service consumer, which wants to use a possibly remote service - the facade that represents this service
on code level.
In an object-oriented language binding, this typically is an instance of a generated class, which provides methods for all functionalities the service provides. So the service consumer side application code interacts with this local facade, which then knows how to propagate these calls to the remote service implementation and back.
* Service Skeleton: This code is - from the perspective of the service implementation, which provides functionalities according to the service definition - the code,
which allows to `connect the service implementation to the Communication Management transport layer`, so that the service implementation can be contacted by distributed service consumers.
In an object-oriented language binding, this typically is an instance of a generated class. Usually the service implementation from the application developer is connected with this generated class via a subclass relationship.
So the service side application code interacts with this middleware adapter either by implementing abstract methods of the generated class or by calling methods of that generated class.
Further details regarding the structure of ara::com Proxies and Skeletons are shown in section section 6.2 and section 6.3. Regarding this design pattern in general and its role in middleware implementations, see [2] and [3].


## 5.2 Runtime Interface
Beside the APIs provided by proxies and skeletons, the ara::com API contains functionality, which is about crosscutting concerns and therefore cannot really be assigned to proxy/skeleton domain.
The approach in ara::com is to assign this kind of functionality to `a Runtime singleton class` (see 6.4).


## 5.3 Data Type Abstractions
ara::com API introduces specific data types, which are used throughout its various interfaces. They can roughly be divided into the following classes:
* Pointer types: for pointers to data transmitted via middleware
* Collection types: for collections of data transmitted via middleware
* Types for async operation result management: ara::com relies on AUTOSAR AP specific data types (see [4]), which are `specific versions of C++ std::future/std::promise`
* `Function wrappers`: for various application side callback or handler functions to be called by the middleware ara::com defines signature and expected behavior of those types, but does not provide an implementation. The idea of this approach is, that `platform vendors could easily come up with their own optimized implementation of those types.`
**This is obvious for collection and pointer types `as one of the major jobs of an IPC implementation has to deal with memory allocation for the data which is exchanged between middleware users.`**

Being able to provide their own implementations allows to optimize for their chosen memory model.
For most of the types ara::com provides a default mapping to existing C++ types in ara/com/types.h. This default mapping decision could be reused by an AP product vendor.
The default mapping provided by ara::com even has a real benefit for a product vendor, who wants to implement its own variant: He can validate the functional behavior of his own implementation against the implementation of the default mapping.

## 5.4 Error Notification
ara::com API follows the concepts of error handling described in chapter "Error handling" in [4]. Checked Errors will be returned via ara::core::ErrorCode directly or an ara::core::ErrorCode embedded into a ara::core::Result, which either holds a valid return value or the ara::core::ErrorCode.
The functionality provided with ara::core::Result and ara::core::Future (details, see subsubsection 6.2.4.2) allow the user of ara::com to chose between exception based or return code based error handling to some degree.

### 5.4.1 Checked Errors/Exceptions
Checked Errors within ara::com API can only occur in the context of a call of a service interface method and is therefore fully covered in subsection 6.2.4 an subsection 6.3.4.

### 5.4.2 Unchecked Errors/Exceptions
Unchecked Errors within ara::com API can occur in the context of any ara::com API call.
**The ara::com API does not throw any Execption.** The only way to have exceptions is calling the get method of ara::core::Future, if the user decides to use this approach.


示例服务 RadarService 提供了一个`event` “BrakeEvent”，它由一个结构组成，其中包含一个标志和一个长度可变的 uint8 数组（作为额外的有效载荷）。
然后，它提供了一个`field` “UpdateRate”（更新速率），该字段为 uint32 类型，支持 get 和 set 调用，最后还提供了三个方法。
`method` “Adjust”用于定位雷达，包含一个目标位置作为输入参数和两个输出参数。一个是定位成功的信号，另一个是最终有效位置（可能有偏差）的报告。
`method` “Calibrate”方法用于校准雷达，输入参数为配置字符串，输出参数为成功指示符。
如果校准失败，该方法可能会引发两种不同的应用程序错误： “CalibrationFailed“（校准失败）和 ”InvalidConfigString"（配置字符串无效）。
`method`“LogCurrentState”法是一个单向方法，这意味着如果该方法被执行，以及执行的结果如何，都不会向调用者返回反馈。它指示 RadarService 服务将其当前状态输出到本地日志文件中。

# 6 API Elements
## 6.1 Instance Identifiers
speciifer是限定符 --> PS: identifier是标识符
`Instance identifiers`, which get used at proxy and as well at skeleton side, are such a central concept, that their explanation is drawn here — before the detailed description of ara::com proxies and skeletons in upcoming chapters.
Instance identifiers are used within ara::com, on client/proxy side, when a specific instance of a service shall be searched for or — at the server/skeleton side — when `a specific instance of a service` is created.
At ara::com API level the instance identifier is generally a technical binding specific identifier.
在 ara::com API 层级，实例标识符通常是技术绑定的特定标识符。
**Therefore the concrete content/structure of which such an instance identifier consists, is totally technology specific**: So f.i. SOME/IP is using 16 bit unsigned integer identifiers to distinguish different instances of the same service type, while DDS (DDS-RPC) uses string<256> as service_instance_name.




If the unambiguousness is ensured, the integrator/deployer can assign a dedicated technical binding with its specific instance IDs to those "instance specifier" via a "manifest file", which is specifically used for a distinct instantiation/execution of the executable.
This explicitly allows, to start the same executable N times, each time with a different manifest, which maps the same ara::core::InstanceSpecifier differently.
如果能确保明确无误，集成者/部署者就可以通过`MANIFEST`为这些`instance specifier`分配专用技术binding及其特定的`instance IDs`，该文件专门用于可执行文件的不同实例化/执行。
**这就明确允许启动同一个可执行文件 N 次，每次都使用不同的MANIFEST，以不同的方式映射同一个 ara::core::InstanceSpecifier。**

```json
  "services" : [
    {
      "service_id" : 2134,
      "shortname_path" : "/smart/service_interface/data_collector_status_2icc_service_interface",
      "major_version" : 1,
      "minor_version" : 0,
      "events" : [
        {
          "id" : 45056,
          "shortname" : "DataCollectorStatusTopic"
        }
      ],
      "fields" : [
      ],
      "methods" : [
      ],
      "provided_service_instances" : [
        {
          "instance_id" : 6134,
          "instance_specifier" : [
            "data_collector/RootSwComponentPrototype/data_collector_status_2icc_service_provided_port"
          ]
        }
      ]
    },
//   ...
  ]
```

PS: `instance_specifier`等价于`p-port`
```json
      "provided_service_instances" : [
        {
          "instance_id" : 2093,
          "instance_specifier" : [
            "someip_to_dds/RootSwComponentPrototype/openspace_trajectory_service_provided_port",
            "someip_to_dds/RootSwComponentPrototype/openspace_trajectory_service_test_provided_port"
          ],
          "domain" : 5,
          "port" : 56093,
          "ttl" : 3,
          "expected_client_integrity_level" : "QM"
        }
      ]
```

`InstanceSpecifier` -> `InstanceIdentifier`
```cpp
namespace ara {
namespace com {
namespace runtime {
      ara::com::InstanceIdentifierContainer ara::com::runtime::ResolveInstanceIDs(ara::core::InstanceSpecifier modelName);
    }
  }
}
```

为什么这个 API 会返回一个 `InstanceIdentifierContainer`（它代表了`ara::com::InstanceIdentifierContainer` 的集合）？
这需要解释一下：
AUTOSAR 支持集成商在软件组件开发人员可见的`一个抽象标识符`后面配置`多个技术绑定`。
此功能称为`多重绑定`，在本文档的不同部分均有提及（更详细的解释请参见第 9.3 节）。
**在`skeleton`/`server`使用`多重绑定`是一种常见的用例，因为它允许不同的客户端在与服务器联系时使用自己喜欢的绑定。**
与此相反，在`proxy/client`使用多重绑定则比较奇特。
例如，它可以用来支持一些`故障转移方法（如果绑定 A 不工作，则使用绑定 B）`。

So the possible returns for a call of `ResolveInstanceIDs()` are:
* empty list:
  The integrator failed to provide a mapping for the abstract identifier. This most likely is a configuration error.
* list with `one element`:
  The common case. Mapping to one concrete instance id of one concrete technical binding.
* list with `more than one element:` 
  Mapping to multiple technical instances with possibly multiple technical bindings.


从技术上讲，`ResolveInstanceIDs()`的中间件实现是从进程中捆绑的`service instance manifest`中查找`ara::core::InstanceSpecifier`。
因此，在捆绑的`service instance manifest`中，`ara::core::InstanceSpecifier`必须是明确的。
VECTOR代码示例：
```cpp
// VECTOR NC AutosarC++17_10-M9.3.3, VectorC++-V5.0.1: MD_SOMEIPBINDING_AutosarC++17_10-M9.3.3_Method_can_be_declared_const
// VECTOR NC AutosarC++17_10-A15.5.3: MD_SOMEIPBINDING_AutosarC++17_10-A15.4.2_A15.5.3_Exception_caught
// VECTOR NC AutosarC++17_10-A15.4.2: MD_SOMEIPBINDING_AutosarC++17_10-A15.4.2_A15.5.3_Exception_caught
void SomeipBindingInitializer::RegisterServiceInstances() noexcept {
  ::ara::com::Runtime& runtime_instance{::ara::com::Runtime::getInstance()};
  {
    // ---- Register all known R-Port InstanceSpecifiers ----
    {
      // Map R-Port /vector/StartApplication/cm/Client1/StartApplicationCmClient1/RPort_StartApplicationCmService1_ServiceInterface to instance /deployment/Required/RequiredSomeipStartApplicationCmService1_ServiceInterface 
      ::ara::core::InstanceSpecifier const instance_specifier{"startapplication_cm_client1/RootSwComponentPrototype/RPort_StartApplicationCmService1_ServiceInterface"_sv};
      ::ara::com::InstanceIdentifier const instance_identifier{"SomeIp:1404"_sv};

        runtime_instance.MapInstanceSpecifierToInstanceId(
            &(aracom_someip_binding_.value()), instance_specifier, instance_identifier,
            "/vector/StartApplication/cm/ServiceInterface/StartApplicationCmService1_ServiceInterface"_sv);
    }
  }
  {
    // No P-Ports configured
  }
}
```
`MapInstanceSpecifierToInstanceId`的内部实现就是：
  使用`instance_identifier`构造出`InstanceSpecifierLookupTableEntry`，
  把`<instance_specifier, entry>`插入该table
`instance_specifier`和`instance_identifier`，是一对多的


### 6.1.1 When to use InstanceIdentifier versus InstanceSpecifier
根据前面的解释，软件开发人员可能会产生这样的印象，即在使用需要实例标识符信息的 ara:com API 之前，总是要先手动（通过调用 `ResolveInstanceIDs()`）将 ara::core::InstanceSpecifier 解析为 ara::com::InstanceIdentifier。

这确实有点尴尬，因为我们已经提到过，`对于 软件开发人员 来说，实现 自适应 AUTOSAR SWC 的 “典型”方法 是 使用 软件组件模型 中的抽象 “instnace speciifers”。`
在接下来详细介绍代理和骨架侧 API 的章节中，您将看到 ara::com 提供了典型的函数重载，这些函数要么使用 ara::com::InstanceIdentifier，要么使用 ara::core::InstanceSpecifier，从而在最常见的使用案例中解放了开发人员，开发人员只需使用 ara::core::InstanceSpecifier 即可，无需明确调用 ResolveInstanceIDs()。
这就意味着，直接使用 `ara::com::InstanceIdentifier` 和手动解析 `ara::core::InstanceSpecifier` 更多是针对那些使用案例比较特殊的高级用户。
在讨论代理/骨架侧相应的 ara::com API 重载的章节中将给出一些示例。

这两种变体之间的根本区别就在于此： `ara::com::InstanceIdentifier` 可以更方便地在自适应应用程序/进程之间交换！
因为它们`已经完全包含了所有特定技术信息`，不需要通过服务实例清单的内容进行任何进一步的解析，这样一个序列化的 ara::com::InstanceIdentifier 可以在不同的进程中重新构建，只要他的进程可以访问与 ara::com::InstanceIdentifier 基于相同的绑定技术，就可以使用。


#### 6.1.1.1 Transfer of an InstanceIdentifier
如前所述，ara::com::InstanceIdentifier 只能用于 “高级用户”，因为它的格式依赖于供应商，并且包含技术绑定信息。
因此，传输或存储 ara::com::InstanceIdentifier 可能会有很大风险。因为转移或重新存储后，转移绑定可能不复存在，或者堆栈供应商 A 的 ara::com::InstanceIdentifier 可能会被使用供应商 B 的堆栈的应用程序解释。



## 6.2 Proxy Class
The Proxy class is generated from the service interface description of the AUTOSAR meta model.

ara::com 对生成的代理类的接口进行了标准化。
AP 产品供应商的工具链将生成一个完全实现该接口的代理实现类。

注：`由于代理类必须提供的接口是由 ara::com 定义的，因此通用（独立于产品）生成器可以生成一个抽象类或模拟类`，应用程序开发人员可以根据这些类实现其服务消费者应用程序。ara::com 希望在 “proxy ”命名空间内生成与代理相关的工件。
该命名空间通常包含在根据服务定义及其上下文推导出的命名空间层次结构中。

```cpp
class RadarServiceProxy {
public:

/*
  brief Implementation is platform vendor specific
* A Handletype must contain the information that is needed to create
* a proxy.
* This information shall be hidden.
* Since the platform vendor is responsible for creation of handles,
  the ctor signature is not given as it is not of interest to the user.
*/

class HandleType {
/**
* brief Two ServiceHandles are considered equal if they represent
* the same service instance.
\param other
\return bool
*/

inline bool operator==(const HandleType &other) const;
const ara::com::Instanceidentifier &GetInstanceId() const;

};

/**
* StartFindService does not need an explicit version parameter as this
* is internally available in ProxyClass.
* That means only compatible services are returned.
* param handler this handler gets called any time the service
* availability of the services matching the given
* instance criteria changes. If you use this variant of
* FindService, the Communication Management has to
* continuously monitor the availability of the services
* and call the handler on any change.
* \param instanceId which instance of the service type defined
* by T shall be searched/found.
* \return a handle for this search/find request, which shall
* be used to stop the availability monitoring and related
* firing of the given handler. ( see StopFindService())
*/
static ara::com::FindServiceHandle StartFindService(
ara::com::FindServiceHandler<RadarServiceProxy::HandleType> handler,
ara::com::InstanceIdentifier instanceId);

/**
* This is an overload of the StartFindService method using an
instance specifier, which gets resolved via service instance
manifest.
* \param instanceSpec instance specifier
*/
static ara::com::FindServiceHandle StartFindService(
ara::com::FindServiceHandler<RadarServiceProxy::HandleType> handler, ara::core::InstanceSpecifier instanceSpec);

/**
This is an overload of the StartFindService method using neither
* instance specifier nor instance identifier.
* Semantics is, that ALL instances of the service shall be found, by
* using all available/configured technical bindings.
*/
static ara::com::FindServiceHandle StartFindService(
ara::com::FindServiceHandler<RadarServiceProxy::HandleType> handler);

/**
*
Method to stop finding service request (see above)
*/
static void StopFindService(ara::com::FindServiceHandle handle);

/**
Opposed to StartFindService(handler, instance) this version
* is a "one-shot" find request, which is:
* - synchronous i.e. it returns after the find has been done
and a result list of matching service instances is available.(list may be empty, if no matching service
instances currentlv exist)
- does reflect the availability at the time of the method call. No further (background) checks of availability are done.
* \param instanceld which instance of the service type defined
* by T shall be searched/found.
*/
static ara::com::ServiceHandleContainer<RadarServiceProxy::HandleType> Findservice(
ara::com::InstanceIdentifier instanceId);


/**
* This is an overload of the FindService method using an
* instance specifier, which gets resolved via service instance
* manifest.
*/
static ara::com::ServiceHandleContainer<RadarServiceProxy::HandleType>
FindService(ara::core::InstanceSpecifier instanceSpec);

/**
* This is an overload of the StartFindService method using neither
* instance specifier nor instance identifier.
Semantics is, that ALL instances of the service shall be found, by
* using all available/confiqured technical bindings.
*/
static ara::com::ServiceHandleContainer<RadarServiceProxy::HandleType> Findservice();

/*
brief The proxy can only be created using a specific
* handle which identifies a service.
This handle can be a known value which is defined at
deployment or it can be obtained using the
ProxyClass::Findservice method.
param handle The identification of the service the
* proxy should represent .
*/
explicit RadarServiceProxy(HandleType &handle);
/**
* proxy instances are not copy constructible.
*/
RadarServiceProxy(RadarServiceProxy &other) = delete;
/**
* proxy instances are not copy assignable.
*/
RadarServiceProxy& operator=(const RadarServiceProxy &other) = delete;

/**
*
\brief Public member for the BrakeEvent
*/
events::BrakeEvent BrakeEvent;

fields::UpdateRate UpdateRate;

methods::Calibrate Calibrate;

methods::Adjust Adjust;

methods::LogCurrentState LogCurrentState;
};
```

### 6.2.1 Constructor and Handle Concept
如图 6.5 所示，ara::com 规定代理类必须提供一个构造函数。
这意味着开发人员负责创建一个代理实例，以便与可能的远程服务进行通信。
构造函数接收一个`RadarServiceProxy::HandleType`类型的参数————这是`生成的代理类的 一个内部类`。
那么直接的问题可能是："这个句柄是什么，如何创建它/从哪里获取它？”
这个句柄是什么，应该很简单：
  在调用 ctor 之后，你就有了一个代理实例，它允许你与服务进行通信，因此句柄必须包含必要的地址信息，COM binding的实现才能够和服务连接。

`地址信息的具体内容` 完全取决于 `绑定实施/技术传输层`！
这已经部分回答了 “如何创建/从何处获取”的问题： 应用程序开发人员不可能真正创建地址信息，因为根据 AUTOSAR 核心理念，他是在实施其应用程序 AP 产品，因此通信管理是独立的。
解决办法是，ara::com 为应用程序开发人员提供了一个用于查找服务实例的 API，该 API 可返回此类句柄。
这里将详细介绍 API 的这一部分：6.2.2 小节。
这种方法的共同优点是，`代理实例只能从 “FindService ”API 生成的句柄中创建，因此只能创建真正由现有服务实例支持的代理`。

#### AUTOSAR Binding Implementer Hint
在实现与 ara::com 兼容的绑定时，您必须决定将哪些信息嵌入到句柄类的实现中，以及如何在代理类 ctor 的实现中对嵌入到句柄实现中的信息作出反应。

要了解更多信息，就必须结合`服务发现机制`（见第 9.2 节）来研究句柄类型，并理解多重绑定的含义（见第 9.3 节）。
当您在 AP 产品中实现了服务发现功能，并因此实现了第 6.2.2 小节中的功能时，您很可能会遇到 ara::com 应用程序调用 ProxyClass::FindService 的典型情况：
* the found service is located on a different node on the network
* the found service is located within a different application on the same node (within
the same AP infrastructure)
* the found service is located within the same process

可能的组合会增加复杂性：
  对于现有的服务类型，上述任何一种情况都可能同时适用 ———— 与应用程序对话的服务实例有一个在本地的同一进程中（如果考虑到代码重复使用较多的大型应用程序，这并不奇怪），一个在不同进程的同一ECU 中，一个在远程 ECU 中。

我们（ara::com 设计团队）要求这种设置能让 ara::com 用户无缝使用。
顺便说一下：这种功能称为多重绑定，因为您有一个代理类形式的服务抽象，它绑定到多个不同的传输绑定。

在所有情况下，使用 ara::com 的应用程序开发人员都会与同一个代理类的实例进行交互，而`代理类的实现`是由您提供的。
现在，人们对 AP 产品的期望很明显，那就是它能提供在这些不同情况下进行高效通信的方法。
这意味着，如果开发人员使用由 HandleType 实例构建的代理实例（该实例表示代理用户本地的服务实例），那么 代理实现 应使用 与由 HandleType 实例构建的代理实例（该实例表示远程服务实例）不同的技术解决方案（在这种情况下，例如简单的本地函数调用/本地地址空间副本）。
一言以蔽之： `接入点产品供应商必须提供一个代理类实现，它能根据 ctor 中给出的 HandleType 实例中包含的信息，委托给完全不同的传输层实现`。

因此，这里可能会出现一个问题： ara::com 本可以直接返回一个代理实例，而不是从 “FindService” 功能中返回一个句柄。
在阅读了 ara::com 如何处理事件访问（6.2.3 小节）之后，我们就能更好地理解其中的原因了。
但目前只需说明以下几点即可、 代理实例包含某些状态。

正因为如此，`在一些用例中，应用程序开发人员希望使用不同的代理实例，而所有代理实例都“连接”到同一个服务实例`。
因此，如果您承认存在这种情况，那么通过句柄进行间接的决定就变得很清楚了：ara::com 无法知道应用程序开发人员是希望始终使用同一个代理实例（显式共享状态），还是希望每次触发 “FindService” 功能时都使用一个新实例，因为该功能返回的代理与服务实例完全相同。
因此，通过提供这种间接/解耦，决定权掌握在 ara::com 用户手中。

另一方面，`代理类的实例既不可复制构造，也不可复制分配`！
这是一个明确的设计决定，与通过 HandleType 强制构造的想法相辅相成。
由于拥有事件/字段缓存、注册处理程序、复杂状态......等，代理类的实例可能会非常耗费资源。
因此，如果允许复制构造/复制赋值，就有可能在无意中完成复制。
因此，一言以蔽之，强迫用户通过 HandleType 来创建代理，会让用户意识到这一决定是经过深思熟虑的。

### 6.2.2 Finding Services
代理类提供类（静态）方法来查找与代理类兼容的服务实例。
由于服务实例有生命周期，其可用性本质上是动态的，因此 ara::com 提供了两种不同的 “FindService ”方法，以方便用户：
- `StartFindService`是一个类方法，它会`在后台启动一个持续的 “FindService”活动，在 “服务实例的可用性” 发生变化时 通过 给定的回调通知调用者`。
- `FindService`是一次性调用，它会返回调用时间点的可用实例。

根据所采用的`instance identifier`方法（参见第 6.1 节），这两种方法有三种不同的重载：
- 一个使用 `ara::com::InstanceIdentifier`
- 一个使用 `ara::core::InstanceSpecifier`
- 一种`不使用参数`。

无参数变量的语义很简单： 查找给定类型的`所有服务`，无论其`绑定方式`和`绑定的具体实例标识符`如何。
请注意，只有技术绑定才会被用于查找/搜索，这些技术绑定是以 ServiceInterfaceDeployment 的形式在服务实例清单中为相应服务接口配置的。
请注意，只有技术绑定才能用于查找/搜索，这些技术绑定是以服务接口部署的形式在服务实例清单中为相应服务接口配置的。
同步一次性变体 FindService 会返回匹配服务实例的句柄容器（见第 6.2.1 小节），如果当前没有匹配的服务实例，则句柄容器也可能为空。

与此相反，`StartFindService`会返回一个`FindServiceHandle`，可以通过调用`StopFindService`来停止正在进行的监控服务实例可用性的后台活动。
`StartFindService` 的第一个参数（也是该变体的特定参数）是用户提供的处理函数，其签名如下：
```cpp
using FindServiceHandler = std::function<void(ServiceHandleContainer<T>, FindServiceHandle)>;
```
一旦 `绑定`检测到与调用 `StartFindService` 时给定的实例条件相匹配的服务实例的`可用性`（？？？）发生了变化，它就会调用用户提供的处理程序，并提供最新的可用服务实例句柄列表。
调用 `StartFindService` 后，`StartFindService` 的行为与 `FindService` 类似，它将使用`当前可用的服务实例`（也可能是空句柄列表）触发用户提供的处理函数。
  初始回调触发后，如果初始服务可用性发生变化，它将再次调用提供的处理函数。
请注意，ara::com 用户/开发者在用户提供的处理程序中调用 StopFindService 是明确允许的。
为此，处理程序会明确获取 FindServiceHandle 参数。处理程序不需要重入。
这意味着绑定实现者必须注意对用户提供的处理程序函数的调用进行按顺序排列。

请注意，`ServiceHandleContainer`可以作为分配容器或非分配容器来实现，在用作 FindService 的返回值或 FindServiceHandler 的参数时，只要它满足 C++ 编程语言对容器的一般要求和顺序要求即可。



### 6.2.2.1 Auto Update Proxy instance
无论使用一次性 `FindService` 还是 `StartFindService`变体，在这两种情况下，你都能`获得 一个标识服务实例（可能是远程服务实例）的句柄`，然后再从中`创建代理实例`。
但是，如果服务实例宕机了，但后来又恢复了，例如，由于生命周期状态发生了变化，会发生什么情况呢？
当服务实例再次可用时，服务消费方现有的代理实例还能重新使用吗？
好消息是 ara::com 设计团队决定从绑定实现中要求这种重用可能性，因为它能减轻实现服务消费者的典型任务。
`在基于服务的通信领域，预计在整个系统（如车辆）的生命周期内，服务提供者和消费者实例会由于其自身的生命周期概念而频繁地启动和关闭。`
`服务提供商和消费者的生命周期在服务提供和服务（再）订阅方面受到监控。`

为此，我们建立了`服务发现基础架构`，从`service offerings`和`service (re)subscriptions`的角度监控 服务提供商 和 消费者 的生命周期！
如果服务消费者应用程序从 “查找服务 ”变体返回的句柄实例化了一个服务代理实例，可能发生的顺序如下所示。
Explanation of figure 6.1:
* T0: The service consumer may successfully call a service method of that proxy (and GetSubscriptionState() on subscribed events will return kSubscribed according to 6.2.3.2).
* T1: The service instance goes down, correctly `notified via service discovery`.
* T2: A call of a service method on that proxy will lead to a checked exception
(ara::com::ServiceNotAvailableException), since the targeted service instance of the call does not exist anymore. Correspondingly `GetSubscriptionState()` on any subscribed event will return `kSubscriptionPending` (see also 6.2.3.2) at this point even if the event has been successfully subscribed (kSubscribed) before.
* T3: The service instance comes up again, `notified via service discovery infrastructure.` The Communication Management at the proxy side will be notified and will `silently update the proxy object instance with a possibly changed transport layer addressing information.`(**比如说，`port number`发生变化**) This is illustrated in the figure with transport layer part of the proxy, which changed the color from blue to rose. The Binding implementer hint part below discusses this topic more detailed.
* T4: Consequently service method calls on that proxy instance will succeed again and GetSubscriptionState() on events which the service consumer had subscribed before, will return kSubscribed again.

图6.1描述了一个服务消费者与服务实例交互的时间线，展示了从正常通信到服务实例中断再到恢复的整个过程。以下是各个时间点的具体解释：
`T0`: 在这个初始阶段，服务消费者可以成功调用代理上的服务方法，并且如果订阅了事件，调用`GetSubscriptionState()`方法将返回`kSubscribed状态`，表明事件已经成功订阅。
`T1`: 服务实例在这个时间点宕机，但是通过服务发现机制正确地通知了服务消费者。
  这意味着系统能够感知到服务实例的不可用，并向相关组件发送通知。
`T2`: 当尝试调用宕机服务实例的代理上的服务方法时，会抛出一个检查异常`ara::com::ServiceNotAvailable` Exception，因为目标服务实例已不存在。
  同时，任何之前已成功订阅的事件，在调用`GetSubscriptionState()`时将返回`kSubscriptionPending`状态，这表明订阅处于等待状态，直到服务恢复可用性。
`T3`: 服务实例重新上线，并通过服务发现基础设施通知所有相关方。
  此时，代理端的通信管理器会被通知，并自动更新代理对象实例中可能改变的传输层地址信息（例如`端口号`）。
  这一变化在图中通过代理传输层部分的颜色从蓝色变为粉红色来表示。
`T4`: 随着服务实例的恢复，对代理实例的服务方法调用将再次成功，对于之前已订阅的事件，调用`GetSubscriptionState()`方法将重新返回`kSubscribed`状态，表示这些事件的订阅状态已恢复正常。



代理实例的这种便利行为（因为有相应的服务发现机制对service的状态进行通知）使服务消费者的实现者不必：
- 通过`GetSubscriptionState()`对事件进行轮询，这表明服务实例宕机
- 重新触发一次性 `FindService()` 以获取新句柄：
或者是：
- 注册一个 `FindServiceHandler`，在服务实例宕机或有新句柄时调用它。
然后从新句柄重新创建代理实例（并重新执行所需的事件订阅调用）。

此外，如果您已注册了`FindServiceHandler`，那么 绑定实现 必须确保在调用已注册的 `FindServiceHandler` 之前对现有代理实例进行 `“自动更新”`（使之可用）！
这样做的原因是：在调用中给出代理实例的句柄时，应用程序开发人员就可以在 FindServiceHandler 中与现有的代理实例成功交互，从而表明服务实例已重新启动。
```cpp
/**
* Reference to radar instance, we work with,
* initialized during startup
*/
RadarServiceProxy *myRadarProxy;
void radarServiceAvailabilityHandler(ServiceHandleContainer<RadarServiceProxy::HandleType> curHandles, FindServiceHandle handle) {
  for (RadarServiceProxy::HandleType handle : curHandles) {
      if (handle.GetInstanceId() == myRadarProxy->GetHandle().GetInstanceId()) {
        /**
        * This call on the proxy instance shall NOT lead to an exception,
        * regarding service instance not reachable, since proxy instance
        * should be already auto updated at this point in time.
        */
        ara::core::Future<Calibrate::Output> out = myRadarProxy->Calibrate("test");
        // ... do something with out.
    }
  }
}
```

AUTOSAR 绑定实现者提示
对于绑定实现者来说，重要的是要明白，当服务实例的底层传输层寻址发生变化时，现有代理实例的这种 `“自动更新 ”`也会起作用！
这种情况是否会发生，完全取决于传输层绑定的实现！
例如，如果我们在 `代理实例 和 服务实例实施` 之间建立了 SOME/IP 网络绑定，那么在服务实例重启后，`服务实例的端口号`可能确实发生了变化。
尽管如此，代理实例的“自动更新”仍将无缝运行！
如果您还记得前面的讨论（见表 6.2.1 和第 9.3 节），我们在其中给出了一些提示，说明绑定实现者可以/可以在代理句柄实例中嵌入哪些内容，那么您可能会提出这样的问题：它是如何干扰 “自动更新 ”的？
在`binding/discovery`生成句柄时，服务实例的`初始传输层寻址信息`很可能会被编码到句柄中，以便由此创建的`proxy instance`能够与`service instance`取得联系。

请注意，对于 `服务实例的 传输层 寻址信息` 在整个生命周期内 `保持不变`的设置，这也是一种`性能优化`！
在这种情况下，可以在生命周期内进行一次服务查找，并将返回的句柄持久地存储在某个地方。
只要服务消费者再次启动（而不是触发查找服务变体），它就可以直接重新使用持久化句柄来创建代理实例。
`其优化之处在于，无需首先进行代价高昂的发现`。
如果代理实例按照 ara::com 的要求在幕后 “自动更新”，当服务实例被重新提供时，可能会出现传输层寻址信息发生变化的情况（如上文所述）。这显然意味着，更新后的代理实例使用的传输层寻址信息与以前构建实例的句柄中包含的信息不同！
另一方面，这也意味着允许用户使用过时的句柄（过时的含义是传输层寻址信息已失效）创建代理实例。
这里需要区分两种不同的情况：
- 在用过时的句柄创建代理实例时，绑定实现不知道新的传输层地址。其结果是，在使用过时的地址信息创建代理实例后，对服务实例的调用可能会失败。
但是，当服务实例被（重新）提供，并且 AP 产品的绑定实现可以看到/知道实例的新传输层地址信息时，它就会对代理实例应用 “自动更新”（用新的传输层地址进行更新）。
- 在使用过时的句柄创建代理实例时，绑定实现已知道新的传输层地址，并使用该地址代替。

**如果服务实例完全改变了传输层机制，“自动更新 ”机制也必须发挥作用。**


### 6.2.3 Events
For each event the remote service provides, `the proxy class contains a member of a event specific wrapper class`. In our example the member has the name BrakeEvent and is of type events::BrakeEvent.
As you see in 6.5 all the event classes needed for the proxy class are generated inside a specific namespace events, which is contained inside the proxy namespace.
The member in the proxy is used to access events/event data, which are sent by the service instance our proxy is connected to. Let’s have a look at the generated event class for our example:

```cpp
void SetReceiveHandler(ara::com::EventReceiveHandler handler);
```
用户通过`SetReceiveHandler`传入的handler不需要重入，因为通信管理实现必须对处理程序的调用进行串行处理： 当上次调用 `GetNewSamples()`后有新事件发生时，MW 会调用一次处理程序。

```cpp
/*
  设置订阅状态更改处理程序，一旦该事件的订阅状态发生变化，通信管理实现就会调用该处理程序。
  通信管理实现将序列化对已注册处理程序的调用。
  如果如果在前一次调用处理程序的运行期间订阅状态发生了多次变化，通信管理会将所有变化汇总到一次调用中，并以最后一次/有效的变化为准。
*/
void SetSubscriptionStateChangeHandler(
  ara::com::SubscriptionStateChangeHandler handler);
```
多次订阅状态的变化，以最后一次有效的为准

```cpp
template <typename F>
ara::core::Result<size_t> GetNewSamples(
  F&& f,
  size_t maxNumberOfSamples = std::numeric_limits<size_t>::max());
  // 默认参数表示处理所有可用的free sample slots
```


#### 6.2.3.1 Event Subscription and Local Cache
仅凭代理实例中存在一个事件封装类成员这一事实，并不意味着用户可以即时访问由服务实例引发/发出的事件。
首先，您必须 “订阅 ”事件，以便告诉通信管理部门，您现在有兴趣接收事件。
```cpp
void Subscribe(size_t maxSampleCount);
```
This method expects a parameter maxSampleCount, which basically informs Communication Management implementation, how many event samples the application intends to hold at maximum.
`Therefore — with calling this method, you not only tell the Communication Management, that you now are interested in receiving event updates, but you are at the same time setting up a "local cache" for those events bound to the event wrapper instance with the given maxSampleCount.`
This cache is allocated and filled by the Communication Management implementation, which hands out smartpointers to the application for accessing the event sample data.
How that works in detail is described in subsubsection 6.2.3.3.
该方法需要一个`maxSampleCount`参数，该参数主要用于通知通信管理实现，应用程序打算最多保存多少个事件样本。
因此，通过调用此方法，您不仅可以告诉 “通信管理 ”实现，您现在有兴趣接收事件更新，同时还可以为绑定到事件包装器实例的事件建立 “本地缓存”，缓存的最大采样数是给定的。
该缓存由 “通信管理 ”实现分配和填充，并将智能指针分配给应用程序，以便其访问事件样本数据。
具体工作原理将在 6.2.3.3 小节中详细介绍。


#### 6.2.3.2 Monitoring Event Subscription
The call to the Subscribe method is `asynchronous` by nature. This means that at the point in time Subscribe returns, it is just the indication, that the Communication Management has accepted the order to care for subscription.
The subscription process itself may (most likely, but depends on the underlying IPC implementation) involve the event provider side. `Contacting the possibly remote service for setting up the subscription might take some time.`

So the binding implementation of the subscribe is allowed to return immediately after accepting the subscribe, even if for instance the remote service instance has not yet acknowledged the subscription (in case the underlying IPC would support mechanism like acknowledgment at all). If the user — after having called Subscribe — wants to get feedback about the success of the subscription, he might call:
```cpp
ara::com::SubscriptionState GetSubscriptionState() const;
```
如果底层 IPC 实现使用了某种机制，如服务端的订阅确认，那么在订阅后立即调用 GetSubscriptionState，如果确认尚未到达，可能会返回 kSubscriptionPending。
否则，在底层 IPC 实现获得即时反馈的情况下（这在本地通信中很有可能），调用也可能已经返回 kSubscribed。
如果用户需要监控订阅状态，他有两种选择：
- 通过 GetSubscriptionState 轮询
- 注册一个处理程序，在订阅状态发生变化时调用该处理程序
```cpp
  void SetSubscriptionStateChangeHandler(ara::com::
SubscriptionStateChangeHandler handler);

enum class SubscriptionState { kSubscribed, kNotSubscribed, kSubscriptionPending };
using SubscriptionStateChangeHandler = std::function<void(SubscriptionState)>;
```
Anytime the subscription state changes, the Communication Management implementation calls the registered handler. A typical usage pattern for an application developer, who wants to get notified about latest subscription state, would be to register a handler before the first call to Subscribe.
After having accepted the “subscribe order” the Communication Management implementation will call the handler first with argument `SubscriptionState.kSubscriptionPending` and later — as it gets acknowledgment from the service side —
it will call the handler with argument `SubscriptionState.kSubscribed`.
PS：用户注册的回调是在订阅状态发生变化时触发，不管是pending还是subscribe都会触发

再次说明：如果底层实现不支持来自服务端的订阅确认，那么实现也可以跳过对带有参数 SubscriptionState.kSubscriptionPending 的处理程序的第一次调用，而直接调用带有参数 SubscriptionState.kSubscribed 的处理程序。
对已注册的 “订阅状态更改 ”处理程序的调用是完全异步的。
也就是说，它们甚至可以在调用 Subscribe 尚未返回时发生。用户必须意识到这一点！

一旦用户为某个事件注册了 “订阅状态改变”处理程序，他可能会收到多次对该处理程序的调用。
不仅是最初，当状态从 SubscriptionState.kNotSubscribed 变为 SubscriptionState.kSubscribed（最终通过中间步骤 SubscriptionState.kSubscriptionPending）时，而且以后的任何时候，因为提供此事件的服务可能有一定的生命周期（可能与某些车辆模式绑定）。
因此，**服务可能会在可用和（暂时）不可用之间切换，甚至可能意外崩溃并重新启动。**
提供事件的服务实例的可用性发生变化时，代理方的通信管理实现可能会看到。
因此，只要检测到对事件订阅状态有影响的变化，通信管理程序就会触发已注册的 “订阅状态更改 ”处理程序。
此外（也许更重要），`通信管理功能还能在需要时更新用户的事件订阅、`
这一机制与上文（6.2.2.1）所述的 “自动更新代理实例 ”机制密切相关： 由于通信管理实现会监控服务实例的可用性，因此一旦服务可用，`service proxy`就会自动连接到通信管理实现。

**该机制不仅会在需要时 “自动更新 ”其代理，还会在更新代理实例后 “悄悄地” 重新订阅用户已完成的任何事件订阅。**
这可以粗略地看作是一个非常有用的安慰功能--如果没有 “更新后重新订阅”功能，仅靠 “自动更新”似乎是一种半心半意的做法。
通过注册 “订阅状态更改” 处理程序，用户现在又多了一种监控服务当前可用性的可能性！
除了 6.2.2 中所述的注册 FindServiceHandler 外，注册了 “订阅状态更改 ”处理程序的用户还可以通过调用其处理程序来间接监控服务的可用性。
如果代理所连接的服务实例出现故障，通信管理会调用带有参数 SubscriptionState.kSubscriptionPending 的处理程序。
一旦 “更新后重新订阅”成功，通信管理程序就会调用带有参数 SubscriptionState.kSubscribed 的处理程序。

符合 ara::com 标准的通信管理实现必须将对用户注册处理程序的调用进行串行化处理。
即：**如果发生新的订阅状态变更，而用户提供的处理程序仍在运行，通信管理实现必须推迟下一次调用，直到前一次调用返回。在用户注册的状态变更处理程序运行期间发生的数次订阅状态变更，应汇总为对用户注册的处理程序的一次调用，`其有效/最后一次调用应为对用户注册的处理程序的一次调用`。**

`AUTOSAR Binding Implementer Hint`
Depending on the used IPC or transport layer technology the lifetime/availability of the service as a whole (represented by the proxy instance) and the availability of its subparts (e.g. events, fields methods) may be distinguishable or not.
With SOME/IP f.i., there is the contract, that the service availability as a whole is notified and the expectation/contract is, that then automatically all subparts are available as well.
Here in ara::com we do not require this tight coupling! So generally it would be supported/allowed, that a service instance could be found (see subsection 6.2.2) and methods could be called on it (via the proxy), but the “subscription state” switches to SubscriptionState.kNotSubscribed, because the service has withdrawn just the event, which the user has subscribed to. The mechanism of registering the “subscription state change” handler with the expectation to steadily monitor state changes in the background is similar or related to the mechanism of Proxy::FindService (see subsection 6.2.2), where the user can also register a handler to monitor availability changes of service instances. So from implementation view point — depending on the used transport layer technology —
those mechanisms may depend on each other or may be tightly coupled implementation-wise.
根据所使用的 IPC 或传输层技术，整个服务（由代理实例表示）的生命周期/可用性和其子部分（如事件、字段方法）的可用性可能是可区分的，也可能是不可区分的。
对于 SOME/IP f.i.，有一个契约，即服务的整体可用性会得到通知，而期望/契约是，所有子部分也会自动可用。
在 ara::com 中，我们不需要这种紧密耦合！
因此，一般情况下，我们支持/允许找到服务实例（见第 6.2.2 小节）并调用其method（通过代理），但 “订阅状态 ”会切换为 `SubscriptionState.kNotSubscribed`，因为服务只撤回了用户订阅的事件。（可见，粒度是比较细的）
注册 “订阅状态更改 ”处理程序以期望在后台稳定监控状态更改的机制与 Proxy::FindService 的机制类似或相关（见第 6.2.2 小节），在后者中，用户也可以注册一个处理程序来监控服务实例的可用性更改。
因此，从实现的角度看--取决于所使用的传输层技术--这些机制可能相互依赖，也可能相互关联。



#### 6.2.3.3 Accessing Event Data — aka Samples
那么，在根据前面的章节成功订阅事件后，如何访问接收到的事件数据样本呢？
`在典型的 IPC 实现中，从事件发出者（服务提供者）发送到订阅代理实例的事件数据会在一些缓冲区（如内核缓冲区、特殊 IPC 实现控制的共享内存区域等）中累积/排队。`
缓冲区、特殊 IPC 实现控制的共享内存区域......）。
因此，必须采取明确的行动，从这些缓冲区获取/取回这些事件样本、
最终进行反序列化，然后以正确的样本缓存形式将其放入`event封装类实例的特定缓存中`。
触发此操作的 API 是`GetNewSamples`.

第二个 size_t 类型的参数控制着事件采样的最大数量，这些采样将从中间件缓冲区获取/反序列化，然后以调用 f 的形式呈现给应用程序。

在调用`GetNewSamples()`时，ara::com 实现会首先检查 应用程序持有的事件采样数 是否 已超过 其在上次调用 `Subscribe()` 时承诺的最大值。
* 如果是，则返回 ara::core::ErrorCode。
* 否则，ara::com 实现会检查底层缓冲区是否包含`新的事件样本`:
  如果是，则将其反序列化到样本槽中，然后调用应用程序提供的带有指向新事件样本的 SamplePtr 的 f。
  这一处理过程（检查缓冲区中的其他样本并回调应用程序提供的回调 f）会重复进行，直到：
- 缓冲区中没有任何新样本
- 缓冲区中有更多样本，但已达到调用`GetNewSamples()`(的入参)时应用程序提供的最大样本数参数。
- 缓冲区中有更多样本，但应用程序已超过在 Subscribe() 中承诺的 maxSampleCount。

在应用程序/用户提供的回调 f 的执行过程中，可以决定如何处理传入的 SamplePtr 参数（即最终对事件数据进行深入检查）： 是将新样本 “扔掉”（因为不感兴趣），还是将其保留备用。
要了解保留/丢弃事件样本的含义，必须充分理解作为事件样本数据访问/入口的 SamplePtr 的语义。
下一章将对此进行说明。返回的 ara::core::Result 包含一个 ErrorCode 或`（在成功情况下）对 f 的调用次数`，这些调用是在 GetNewSamples 调用的上下文中完成的。


#### 6.2.3.4 Event Sample Management via SamplePtrs
A `SamplePtr`, which is handed over from the ara::com implementation to application/user layer is — from a semantical perspective — a unique-pointer (very similar to a `std::unique_ptr`): When the ara::com implementation hands it over an ownership transfer takes place. 
From now on the application/user is responsible for the lifetime management of the underlying sample. As long as the user doesn’t free the sample by destroying the SamplePtr or by calling explicit assignment-ops/modifiers on the SamplePtr instance, the ara::com implementation can not reclaim the memory slot occupied by this sample.

事件样本数据所在的内存插槽由 ara::com 实现分配。
这通常发生在调用`Subscribe()`时，用户/应用程序通过参数 maxSampleCount 定义了希望同时访问的事件数据样本的最大数量。
在以后的 GetNewSamples() 调用中，ara::com 实现将填充这样一个 “样本槽”（如果有空闲），并在用户/应用程序回调 f 中传递指向它的 SamplePtr。
在回调实现中，用户/应用程序决定如何处理传入的 SamplePtr。如果用户/应用程序希望保留样本以供日后访问（即在回调返回后），那么它将在某个外层作用域位置复制样本，以适应其软件组件架构。决定是否复制样本（即保留样本）可能仅仅取决于事件样本数据的属性/值。在这种情况下，回调实现基本上是对接收到的事件样本进行 “过滤”。
由于我们说过，SamplePtr 的行为类似于 std::unique_ptr），因此上面的语句必须稍作修改： `在决定保留事件采样时，实现显然不是复制传入的 SamplePtr，而是将其移动到外层作用域位置。移动到外层作用域位置。`
6.8 中的小示例除其他外，还展示了方法`handleBrakeEventReception()`中的回调实现如何实现简单的过滤，以及如何将样本移动到具有 “LastN ”语义的全局存储中，供以后使用/处理。

`AUTOSAR 绑定实现者提示`
如前几章所述，在事件接收代理实例的进程空间中为样本分配内存是`绑定实现`的工作。
它应在 `<event>::Subscribe` 调用的上下文中完成，因为可能存在资源关键型应用程序（如符合 ASIL 要求的应用程序？？？），这些应用程序明确将其对 Subscribe 的调用转移到初始化阶段，在该阶段允许从内核分配内存。
因此，将真正的内存分配转移到稍后阶段（执行 Subscribe 调用后）的绑定实现应考虑到，可能会有应用程序无法接受这样的实现、
因为这样一来，应用程序就无法控制从操作系统/内核分配内存的时间（例如通过 mmap 或 brk）。
另外，在前面几章中，我们提到了一个事实，即应用程序在调用 Subscribe 时可能会超出其占用的样本数。
这种情况基本上会发生，因为一个典型的绑定实现会在一个名为 “Subscribe ”的样本数之上分配一个额外的 "空闲槽"！
对于使用 LastN 策略的典型应用程序，在处理样本时，会在 GetNewSamples() 过程中将现有/持有的 SamplePtr 替换为新的 SamplePtr。
`为了实现这一基本语义，绑定实现需要一个额外的备用槽，以便在将事件数据序列化之前将其提供给应用程序`


#### 6.2.3.5 Event-Driven vs Polling-Based access
正如我们所承诺的，我们完全支持以`事件驱动`和`轮询方式`访问新数据。

！！！为什么需要轮询
对于轮询方法，除了我们已经讨论过的那些应用程序接口外，不需要其他应用程序接口。
典型的用例是，您有一个应用程序，会被周期性地触发以进行某些处理，并在特定期限内提供其输出。
这是`regulator/control算法`的典型模式--循环激活可能还由`实时定时器`驱动，以确保最小的抖动。
在这种情况下，每个激活周期都要调用 GetNewSamples()，然后将更新的缓存数据作为当前处理迭代的输入。
在这里，只需在处理算法调度时获取要处理的最新数据即可。
如果通信管理程序在新数据可用时随时通知您的应用程序，则会适得其反： 这将意味着应用程序进程会出现`不必要的上下文切换`，`因为在收到通知时，您并不想处理新数据，因为还不到处理的时候。`
不过，也有其他使用情况。如果您的应用程序没有这种：以截止日期为驱动的方法，而只是在某些事件发生时做出反应、那么设置周期性警报并通过调用 GetNewSamples() 轮询新事件 的调用来设置周期性警报和轮询新事件，这就有点不妥了，而且效率极低。

（即：异步模式）
在这种情况下，您明确希望通信管理程序通知您的应用程序，从而向您的应用程序进程发出异步上下文切换。
我们通过以下 API 机制支持这种情况：
```cpp
  void SetReceiveHandler(ara::com::EventReceiveHandler handler)；
```
通过该 API，您可以注册一个用户定义的回调函数，当上次调用 GetNewSamples() 后`出现新的事件数据时`，通信管理程序必须调用该函数。
注册的函数无需重入，因为通信管理程序必须将对注册回调的调用序列化。
明确允许在已注册的回调中调用`GetNewSamples()`！


`AUTOSAR Binding Implementer Hint`
如果绑定实现调用了注册的用户函数，并且在执行此函数期间有新事件到达，但应用程序尚未再次从正在运行的用户函数中调用GetNewSamples()，则不需要触发新的回调！
但是，新到达的数据必须在应用程序下一次调用GetNewSamples()时可见。如果在执行此函数期间用户调用了GetNewSamples()，并且在或之后此GetNewSamples()仍在注册的接收处理程序内部到达新事件数据，则通信管理实现必须延迟对接收处理程序的下一次调用，直到运行的调用结束。
因此，直观的绑定实现方式是设置一个`标志`，当新的事件数据到达且用户定义的接收处理程序当前正在运行时。当用户接收处理程序结束时，通信管理实现只需检查标志是否已设置，并在需要时发出对接收处理程序的下一次调用。


注意，用户可以随时通过event wrapper提供的`UnsetReceiveHandler()`方法`更改事件驱动和轮询样式之间的行为`，因为用户还可以撤销用户特定的“接收处理程序”。
 以下简短的代码片段是如何在代理/客户端端处理事件的简单示例。
 在此示例中，在main函数中创建了一个RadarService类型的代理实例，并注册了一个接收处理程序，当接收到新的BrakeEvent事件时，ara::com实现会调用该处理程序。这意味着，在这个示例中，我们使用了“事件驱动”方法。
在我们的示例接收处理程序中，我们使用新接收的事件更新本地缓存，从中筛选出所有不符合特定属性的BrakeEvent事件。之后，我们调用一个处理函数，处理我们决定保留的样本。

```cpp
myRadarProxy->BrakeEvent.GetNewSamples(
  [](SamplePtr<proxy::events::BrakeEvent::SampleType> samplePtr) {
  if(samplePtr->active) {
    lastNActiveSamples.push_back(std::move(samplePtr));
    if (lastNActiveSamples.size() > 10)
      lastNActiveSamples.pop_front();
  }
});
```
策略是：获取这一批数据中，最新的N个数据
```cpp
/* Note: If the entity we would subscribe to, would be a field instead of an event, it would be crucial, to register our reception handler BEFORE subscribing, to avoid race conditions. After a field subscription, you would get instantly so called "initial events" and to be sure not to miss them, you should care for that your reception handler is registered before. * /
```
对于`field`要避免`race condition`
要把注册receive handler放在订阅操作前面，
由于field有初值，为了确保不错过这些事件，应确保接收处理程序在订阅之前已经注册好。如果在订阅后才set handler，那么初值可能丢失。


#### 6.2.3.6 Buffering Strategies
`AUTOSAR Binding Implementer Hint`
At this point it surely makes sense to talk about reasonable buffering strategies for binding implementations. So this entire subsection is mainly of interest for an AP product vendor/binding implementer.

The following figure sketches a simple deployment, where we have a service providing an event, for which two different local adaptive SWCs have subscribed through their respective ara::com proxies/event wrappers. As you can see in the picture both proxies have a local event cache. This is the cache, which gets filled via `GetNewSamples()`.
What this picture also depicts is, that the service implementation sends its event data to a Communication Management buffer, which is apparently outside the process space of the service implementation — the picture here assumes, that `this buffer is owned by kernel or it is realized as a shared memory between communicating proxies and skeleton or owned by a separate binding implementation specific “demon” process.`
图中假设的背景如下： 自适应应用程序是作为具有独立/受保护内存/地址空间的进程来实现的。
服务实现（通过skeleton）发送的事件数据不能在service/skeleton进程私有地址空间内缓冲：如果是这样的话，代理对事件数据的访问通常会导致服务应用进程的上下文切换。
我们希望通过方法调用处理模式（`MethodCallProcessingMode`）（参见 6.3.3 小节）对服务端进行完全控制，因此任意服务消费者的通信行为不应触发这种情况。
现在，让我们粗略地看看作为 “发送事件 ”目标的缓冲区可能位于以下三个不同位置：
* `Kernel Space`:
  Data is sent to a memory region not mapped directly to an application process.
  This is typically the case, when binding implementation uses IPC primitives like pipes or sockets, where data written to such a primitive ends up in `kernel buffer space`.
* `Shared Memory`:
  (需要实现数据同步)
  Data is sent to a memory region, which is also directly readable from receivers/proxies.
  Writing/reading between different parties is synchronized specifically (lightweight with mem barriers or with explicit mutexes).
* `IPC-Daemon Space`:
  Data is sent to an explicit non-application process, which acts as a kind of demon for the IPC/binding implementation.
  Note, that technically this approach might be built on an IPC primitive like communication via kernel space or shared memory to get the data from service process to demon process.


在`缓冲区空间的灵活性/大小`、`访问速度/开销方面的效率`以及`防止恶意访问/写入缓冲区方面`，每种方法都可能有不同的优缺点。
因此，考虑到 AP 产品及其使用中的不同限制，可能会产生不同的解决方案。
本例中要强调的是，明确鼓励 AP 产品供应商使用基于参考的方法来访问事件数据： `event wrapper的 ara::com API 故意通过 SamplePtr 来建立访问模型，并将其传递给回调而不是值！`
在`1:N事件通信`的典型场景中，这将允许在 “本地事件缓存 ”中拥有的不是事件数据值本身，而是指向中央通信管理缓冲区中数据的`指针/引用`。
这样，通过`GetNewSamples()`对本地缓存进行更新时，就`不是以值复制的方式，而是以引用更新的方式来实现`。
老实说：这显然只是对缓冲区使用方面优化可能性的粗略描述。
正如这里（第 9.1 节）所提示的，传输到应用程序进程的数据通常必须在首次访问应用程序之前进行去序列化。

由于去序列化必须专门针对消费应用程序的对齐方式，因此集中共享已经去序列化的表示可能会很棘手。
但至少你明白了一点，那就是代理/服务端的事件数据访问 API 设计为消费者之间共享事件数据提供了空间。


### 6.2.4 Methods
For each method the remote service provides, the proxy class contains a member of a method specific wrapper class.
In our example, we have three methods and the corresponding members have the name Calibrate (of type methods::Calibrate), Adjust (of type methods::Adjust) and LogCurrentState (of type methods::LogCurrentState). Just like the event classes the needed method classes of the proxy class are generated inside a specific namespace methods, which is contained inside the proxy namespace.
The method member in the proxy is used to call a method provided by the possibly remote service instance our proxy is connected to.


```cpp
class Adjust {
public:
  struct Output {
    bool success;
    Position effective_position;
};

ara::core::Future<Output> operator()(const Position &target_position);
};
```

#### 6.2.4.1 One-Way aka Fire-and-Forget Methods
在继续介绍“普通”方法提供的功能之前，我们简要介绍“单向方法”，因为我们在前一节中已经提到了这个术语。
ara::com支持一种特殊类型的方法，我们称之为“单向”或`“fire-and-forget”`。
从技术上讲，这是一种`只有IN参数而没有OUT参数`且`不允许引发错误`的方法。
与服务器之间也不可能进行握手/同步！
因此，客户端/调用方在服务器/被调用方是否处理了“单向”调用时将得不到任何反馈。

有些通信模式中，这种`尽力而为`的方法是完全足够的。
在这种情况下，“单向/发出并忘记”的语义在资源方面非常`轻量级`。
这意味着在某些情况下，不需要等待服务器的响应或确认，只需发送请求并继续执行即可，这种轻量级的通信方式可以更高效地利用资源。

`AUTOSAR Binding Implementer Hint`
To support the notion of a “one-way/fire-and-forget” method perfectly, implementation of such a call should be very asynchronously by nature. I.e. blocking the caller for some time to do a lot of housekeeping/setting up the call, is not expected by the caller of a “one-way/fire-and-forget” method! Since he isn’t even prepared for any error handling in this case, it is explicitly encouraged to do minimal processing in the callers context and shift as much as possible in an asynchronous context.
为了完美支持“单向/发出并忘记”方法的概念，这种调用的实现应该是非常异步的。即，阻塞调用方一段时间来进行大量的清理/设置调用并不是“单向/发出并忘记”方法的调用方所期望的！`由于在这种情况下甚至没有准备好进行任何错误处理，因此鼓励在调用方的上下文中进行最少的处理，并尽可能将尽可能多的工作转移到异步上下文中。`
`LogCurrentState`示例类，可能用于在特定时间点或事件发生时记录系统、应用程序或对象的当前状态，以便后续分析、调试或监控。就是一个比较轻量级的类。


#### 6.2.4.2 Event-Driven vs Polling access to method results
`Event-Driven`
Like in the event data access, `event-driven` here means, that the caller of the method (the application with the proxy instance) `gets notified by the Communication Management implementation` as soon as the method call result has arrived.

For a Communication Management implementation of ara::com this means, it has to setup some kind of `waiting mechanism (WaitEvent)` behind the scene, which gets woken up as soon as the method result becomes available, to notify the ara::com user. So how do the different usage patterns of the ara::core::Future work then?

```cpp
enum class future_status : uint8_t
{
  ready, ///< the shared state is ready
  timeout, ///< the shared state did not become ready before the specified timeout has passed
};

template <typename T, typename E = ErrorCode>
class Future {
public:
// ...
  Result<T, E> GetResult() noexcept;

  template <typename Rep, typename Period>
  future_status wait_for(std::chrono::duration<Rep, Period> const& timeoutDuration) const;

  template <typename Clock, typename Duration>
  future_status wait_until(std::chrono::time_point<Clock, Duration> const& deadline) const;

  template <typename F>
  auto then(F&& func) -> SEE_COMMENT_ABOVE;

};
```
根据您提供的信息，AUTOSAR绑定实现者需要确保在future对象上调用get()或wait()的变体之一时，用户会在方法结果（有效结果或异常）可用时立即收到通知。这是他们对事件驱动的定义。在所有这些情况下，绑定实现者必须设置一个机制，确保用户通过Future::then注册的回调被调用，或者在服务方法结果可用或在调用过程中检测到错误时，阻塞的wait()/get()调用立即恢复。

“立即”这个概念显然有点模糊！ara::com设计团队设想的一般方法可以通过一个简单的例子来最好地解释：
假设使用的底层传输机制基于Unix域套接字（或类似的基于fd的I/O）。在方法结果准备就绪时，服务方法的骨架端实现会将其写入相应的套接字文件描述符。
在域套接字的接收端，代理实例将具有相应的描述符，并通过select或poll等待它。`也就是说，在服务端向套接字写入时，会“立即”唤醒在select/poll调用中等待的代理端通信管理代码。`

正如您所看到的，“立即”取决于机器负载和所使用的低级机制提供的延迟。
`另一方面，我们不会排除一个偏向于基于轮询的机制的低级实现！`
因此，代理实现也可以周期性地检查数据，而不是从服务实例传播OS信号到代理实例，这会导致恢复线程执行。
如果轮询频率足够高，这可能会导致用户调用服务方法时具有相当低且可接受的延迟！
在底层传输机制类似基于文件描述符的读/写I/O的情况下，这样的方法可能没有太多意义，因为您会对每个描述符发出读取。
但在基于共享内存的实现或支持异步I/O的情况下，允许每个系统调用提交多个I/O操作，这可能是一个有效的用例！
如果您有一个具有极端通信负载的自适应应用程序，那么在通信管理实现级别上采用这样一种基于轮询的解决方案，甚至为了实现事件驱动的应用程序行为可能是有意义的，如果您的平台/选择的传输机制提供了有效的批量操作，并且您可以以一些可接受的延迟成本应用这些操作。

在某些情况下，ara::com用户可能根本不希望他的应用程序（进程）被某个方法调用返回事件激活！
想象一下一个典型的`实时(Real Time)应用程序`，它必须完全控制自己的执行。
我们已经在事件数据访问的上下文中讨论过这种实时/轮询用例（子小节6.2.3.3）。
对于方法调用，同样的方法适用！
这意味着对于实时应用程序，用户可能希望完全控制方法调用的执行，而不希望由于方法调用返回事件而激活应用程序。

对于ara::core::Future，我们预见了以下的使用模式：
在通过`operator()`调用服务方法之后，您只需使用`ara::core::Future::is_ready()`来轮询方法调用是否已经完成。
这个调用被定义为非阻塞的。
当然，它可能`涉及一些系统调用/上下文切换（例如查看一些内核缓冲区）`，这`并非免费，但它不会阻塞`！
在ara::core::Future::is_ready()返回true之后，可以保证下一次调用ara::core::Future::get()不会阻塞，而会立即返回有效值或在出现错误时抛出异常。


#### 6.2.4.3 Canceling Method Result
在使用服务方法调用时，如果你对结果不再感兴趣，应该如何处理
。如果你已经通过()运算符调用了一个服务方法，返回了一个ara::core::Future对象，但是你对结果不再感兴趣，甚至可能已经通过ara::core::Future::then()注册了一个回调函数。在这种情况下，你应该明确告诉通信管理模块。
让ara::core::Future对象超出作用域是一种简单的方式来表明你不再对方法调用结果感兴趣，这样它的析构函数就会被调用。
`ara::core::Future的析构函数的调用是一个信号，告诉绑定实现，任何针对该future注册的回调都不应再被调用，为方法调用结果保留/分配的内存可能会被释放，等待方法结果的事件机制也将被停止。`

`AUTOSAR Binding Implementer Hint`
这段文字主要讨论了在服务方法调用中，如果用户通过触发 ara::core::Future 的析构函数表明不再对服务方法调用结果感兴趣，那么跳过该方法调用的工作是有意义的。
在极端情况下，这意味着将方法调用的取消传播到实现服务方法的服务端。
然而，他们故意不要求这样做，因为这可能会对应用程序级别的实现产生很大影响。如果他们要求或预见应用程序级别的服务方法随时可能被中止，那么就会涉及到高级应用程序协议的领域（类似`事务系统`），并且会给服务端应用程序开发人员带来很大负担，这是超出范围的。
但是，绑定实现者可以自由地`将 取消操作 传播到 服务端框架`，这样应用程序级别方法实现返回的方法调用结果可能会直接在服务端/框架端被丢弃！
当然，这样的高效实现需要一个适当的`控制通道/协议`，以便从代理传播取消到框架。
例如，SOME/IP协议并没有提供这样的机制，因此在使用SOME/IP传输时，方法结果不能在框架端被丢弃。
这是否会造成实质性影响还有待商榷。
取消通知会带来额外的网络流量，只有在从框架到代理的传输节省的资源要多得多时，才会有可衡量的效益。

`To trigger the call to the dtor` you could obviously let the future go out of scope.
Depending on the application architecture this might not be feasible, as you already might have assigned the returned ara::core::Future to some variable with greater scope.


```cpp
  calibrateFuture = service.Calibrate(myConfigString);

  calibrateFuture = Future<Calibrate::Output>();  // 直接把future无效化
```



### 6.2.5 Fields
Conceptually a field has — unlike an event — a certain value at any time.
That results in the following additions compared to an event:
* if a subscription to a field has been done, “immediately” `initial values are sent back` to the subscriber in an event-like notification pattern.
* the current field value can be queried via a call to a `Get()` method or could be updated via a `Set()` method.
总之，field表示的量，一直存在

`在`field`的配置中，你可以决定字段是否具有“on-change-notification”、Get() 或 Set() 这三种机制中的哪些`。
在这个例子中，字段同时配置了这三种机制。
对于每个字段，远程服务提供的代理类包含一个特定字段包装类的成员。在这个例子中，成员的名称为UpdateRate（类型为fields::UpdateRate）。
与事件和方法类一样，代理类的字段类是在特定命名空间fields内生成的，该命名空间包含在代理命名空间内。
在解释字段的概念时，它被故意放在事件和方法的解释之后，因为字段的概念大致上是一个具有相关get()/set()方法的`event`的聚合。
因此，从技术上讲，我们也将ara:com`field`表示实现为ara:com`event`和`method`的组合。

Consequently the field member in the proxy is used to
* call `Get()` or `Set()` methods of the field with exactly the same mechanism as regular method
* access field update notifications `in the form of events/event data`, which are sent by the service instance our proxy is connected to with exactly the same mechanism as regular events





## 6.3 Skeleton Class
The Skeleton class is generated from the service interface description of the AUTOSAR meta model. ara::com does standardize the interface of the generated Skeleton class. The toolchain of an AP product vendor will generate a Skeleton implementation class exactly implementing this interface.

The generated Skeleton class is an `abstract class`. It cannot be instantiated directly, because it does not contain implementations of the service methods, which the service shall provide.
`Therefore the service implementer has to subclass the skeleton and provide the service method implementation within the subclass.`

Note: Equal to the Proxy class the interfaces the Skeleton class has to provide are defined by ara::com, a generic (product independent) generator could generate an abstract class or a mock class against which the application developer could implement his service provider application. This perfectly suits the platform vendor independent development of Adaptive AUTOSAR SWCs.
！！！开发者可以根据这个生成的类来实现自己的服务提供者应用程序。这种方法非常适合于`平台供应商无关的 Adaptive AUTOSAR SWCs 开发`。
ara::com expects skeleton related artifacts inside a namespace "skeleton". This namespace is typically included in a namespace hierarchy deduced from the service definition and its context.
这个命名空间通常是从`服务定义`和`上下文中推导出来的命名空间层级结构中的一部分`。这样做有助于组织和管理代码，使得代码结构清晰且易于维护。

```cpp
class RadarServiceSkeleton {
public:
  RadarServiceSkeleton(ara::com::InstanceIdentifier instanceId,
    ara::com::MethodCallProcessingMode mode = ara::com::MethodCallProcessingMode::kEvent);

  /*
    This specifically supports multi-binding.
  */
  RadarServiceSkeleton(ara::com::InstanceIdentifierContainer instanceIds,
    ara::com::MethodCallProcessingMode mode = ara::com::MethodCallProcessingMode::kEvent);

  RadarServiceSkeleton(ara::com::InstanceSpecifier instanceSpec,
    ara::com::MethodCallProcessingMode mode = ara::com::MethodCallProcessingMode::kEvent);

  RadarServiceSkeleton(const RadarServiceSkeleton& other) = delete;

  RadarServiceSkeleton& operator=(const RadarServiceSkeleton& other) = delete;

/*
  The Communication Management implementer should care in his dtor implementation, 
  that the functionality of StopOfferService() is internally triggered in case this service instance has been offered before.
  This is a convenient cleanup functionality.
*/
~RadarServiceSkeleton();  // 需要触发StopOfferService() 确保在服务实例被销毁时，相关的服务提供也会被停止。


void OfferService();

void StopOfferService();

struct CalibrateOutput {
  bool result;
};

/*
  This fetches the next call from the Communication Management and executes it.
  The return value is a ara::core::Future. n case of an Application Error, 
  an ara::core::ErrorCode is stored in the ara::core::Promise from which the ara::core::Future is returned to the caller.
  Only available in polling mode.
*/
ara::core::Future<bool> ProcessNextMethodCall();

events::BrakeEvent BrakeEvent;

virtual ara::core::Future<CalibrateOutput> Calibrate(std::string configuration) = 0;
};
```

### 6.3.1 Instantiation
Since you could deploy many different instances of the same type (and therefore same skeleton class) it is straightforward, that you have to give an instance identifier upon creation.(可以部署多个service instance的不同实现)
This identifier has to be unique. 
In the exception-less creation of a service skeleton a static member function Preconstruct checks the provided identifier.
The construction token is embedded in the returned ara::core::Result if the identifier was unique. Otherwise it returns ara::core:.ErrorCode.

在无例外地创建服务骨架时，静态成员函数 `Preconstruct` 会`检查 所提供的标识符`。
如果标识符是唯一的，则在返回的 ara::core::Result 中嵌入构造标记。否则返回 ara::core:.ErrorCode。

如果要创建具有相同`identifier`的新实例，必须先销毁现有实例。
正是出于这个原因，骨架类（就像代理类一样）既不支持复制构造，也不支持复制赋值！
否则，两个 “完全相同”的实例就会以相同的实例标识符存在一段时间，方法调用的路由就会变得不确定。
关于实例标识符定义的不同 ctors 变体反映了它们的不同性质，这在第 6.1 节中介绍过。
* variant with ara::com::`InstanceIdentifier`:
  Service instance will be created with exactly `one binding specific instance identifier`.
* variant with ara::com::`InstanceIdentifierContainer`:
  Service instance will be created with bindings to `multiple distinct instance identifiers`.
  This is mentioned as "multi-binding" throughout this document and also explained in more detail in section 9.3
* variant with ara::core::`InstanceSpecifier`:
  Service instance will be created with bindings to the instance identifier(s) found after "`service manifest`" lookup with the given ara::core::InstanceSpecifier.
  Note, that this could also imply a "multi-binding" as the integrator could have mapped the given ara::core::InstanceSpecifier to multiple technical/binding specific instance identifiers within the "service manifest".

这也可能意味着“多绑定”，因为集成者可以将给定的 ara::core::InstanceSpecifier 映射到“服务清单”中的多个技术/绑定特定实例标识符。
换句话说，集成者可以在服务清单中将`一个抽象的实例标识符`映射到`多个具体的技术或绑定相关的实例标识符上`。
这种做法可以帮助在不同的技术或绑定层面上管理和识别相同的实例，从而实现更灵活和多样化的系统集成和管理。

Note: Directly after creation of an instance of the subclass implementing the skeleton,
this instance will not be visible to potential consumers and therefore no method will be called on it.
This is only possible after the service instance has been made visible with the `OfferService` API (see below).


### 6.3.2 Offering Service instance
The skeleton provides the method `OfferService()`.
After you — as application developer for the service provider side — have instantiated your custom service implementation class and initialized/set up your instance to a state, where it is now able to serve requests (method calls) and provide events to subscribing consumers, you will call this OfferService() method on your instance.
From this point in time, where you call it, method calls might be dispatched to your service instance — even if the call to OfferService() has not yet returned.
If you decide at a certain point (maybe due to some state changes), that you do not want to provide the service anymore, you call StopOfferService() on your instance. The contract here is: After StopOfferService() has returned no further method calls will be dispatched to your service instance.
For sanity reasons ara::com has the requirement for the AP vendors implementation of the skeleton dtor, that it internally does a StopOfferService() too, if the instance is currently offered.

这段话描述了 在作为服务提供方的应用程序开发者 实例化 自定义服务实现类 并将实例初始化/`method`调用）并向订阅的消费者提供`event`之后，会调用 `OfferService()` 方法。
从调用此方法的时间点开始，即使调用 `OfferService()` 方法尚未返回，方法调用也可能会被分派到您的服务实例。

如果在某个时刻（可能是由于某些状态变化），您决定不再提供服务，可以在实例上调用`StopOfferService()`。
在这里的约定是：在`StopOfferService()`返回后，不会再将进一步的方法调用分派到您的服务实例。
出于健全性考虑，ara::com 对于骨架类析构函数的API供应商实现有一个要求，即如果实例当前正在提供服务，那么在内部也要执行`StopOfferService()`。
这样做是为了确保在销毁实例时，服务的提供会被正确地停止，避免潜在的问题和混乱。

So — “`stop offer`” needs only be called on an instance which lives on and during its lifetime it switches between states, where it is visible and provides its service, and states, where it does not provide the service.
是的，“停止提供服务”只需要在一个实例上调用，该实例在其生命周期内会在`可见并提供服务的状态`和`不提供服务的状态`之间切换。
当您的服务实例需要停止提供服务时，您可以调用`StopOfferService()`方法来实现这一目的。
这种设计允许服务实例在需要时灵活地切换服务提供状态，确保系统的稳定性和可控性。

### 6.3.3 Polling and event-driven processing modes
Now let’s come to the point, where we deliver on the promise to support event-driven and polling behavior also on the service providing side.
From the viewpoint of the service providing instance — here our skeleton/skeleton subclass instance — requests (service method or field getter/setter calls) from service consumers may come in at arbitrary points in time.
In a `purely event-driven setup`, this would mean, that the Communication Management generates corresponding call events and transforms those events to concrete method calls to the service methods provided by the service implementation.
在一个纯粹的事件驱动设置中，通信管理会生成相应的`调用事件`，并将这些 `事件` 转换为`对 由服务实现提供的 服务方法 的 具体方法调用`。
换句话说，当某个事件发生时，通信管理会触发相应的调用事件，
然后将这些事件转换为实际的方法调用，以便服务实现可以对这些事件做出响应并执行相应的操作。
这种设计模式通常用于异步和松散耦合的系统中，以实现更灵活和可扩展的架构。
The consequences of this setup are clear:
* general reaction to a service method call might be fast, since `the latency is only restricted by general machine load and intrinsic IPC mechanism latency.`
* rate of context switches to the OS process containing the service instance might be high and non-deterministic, decreasing overall throughput.（如果触发频率高，上下文切换多，显得处理的吞吐量差）

As you see — there are pros and cons(优缺点) for an event-driven processing mode at the service provider side. However, we do support such a processing mode with ara::com. The other bookend we do support, is a pure polling style approach. Here the application developer on the service provider side explicitly calls an ara::com provided API to process explicitly one call event.
With this approach we again support the typical RT-application developer. His application gets typically activated due to a low jitter cyclical alarm.
通过这种方法，我们再次支持典型的实时应用程序开发人员。他的应用程序通常由于低抖动周期性警报而被激活。
PS: "抖动"（jitter）指的是事件发生的时间上的不确定性或波动性。

When his application is active, it checks event queues in a non-blocking manner and decides explicitly how many of those accumulated (since last activation time) events it is willing to process. Again: Context switches/activations of the application process are only accepted by specific (RT) timers. Asynchronous communication events shall not lead to an application process activation.
So how does ara::com allow the application developer to differentiate between those processing modes? The behavior of a skeleton instance is controlled by the second parameter of its ctor, which is of type ara::com::MethodCallProcessingMode.
在这种情况下，当应用程序处于活动状态时，它以`非阻塞`的方式`检查 事件队列`，并明确决定要处理多少个（自上次激活以来积累的）事件。
PS: 这意味着应用程序的执行是由特定的定时器控制的，只有在定时器触发时才会激活应用程序进程进行处理。
异步通信事件不应该导致应用程序进程的激活，这样可以确保应用程序的执行是在可控的时间点进行的，从而提高系统的可预测性和稳定性。

再次强调：应用程序进程的`上下文切换/激活`仅由特定（实时）定时器接受。异步通信事件不应导致应用程序进程激活。
那么，ara::com如何允许应用程序开发人员区分这些处理模式呢？
skeleton实例的行为由其构造函数的第二个参数控制，该参数的类型为`ara::com::MethodCallProcessingMode`。
```cpp
enum class MethodCallProcessingMode { kPoll, kEvent, kEventSingleThread };
```

That means the processing mode is set for `the entire service instance` (i.e. all its provided methods are affected) and is fix for the whole lifetime of the skeleton instance.
The default value in the ctor is set to `kEvent`, which is explained below.


#### 6.3.3.1 Polling Mode
If you set it to `kPoll`, the Communication Management implementation will not call any of the provided service methods asynchronously!
If you want to process the next (assume that there is `a queue` behind the scenes, where incoming service method calls are stored) pending service-call, you have to call the following method on your service instance:
```cpp
ara::core::Future<bool> ProcessNextMethodCall();
```
We are using the mechanism of `ara::core::Future` again to return a result, which will be fulfilled in the future. What purpose does this returned ara::core::Future serve?
It allows you to get notified, when the “next request” has been processed. That might be helpful to chain service method calls one after the other. A simple use case for a typical RT application could be:
* RT application gets scheduled.
* it calls `ProcessNextMethodCall` and registers a callback with `ara::core::Future::then()`
* the callback is invoked after the service method called by the middleware corresponding to the outstanding request has finished.
* in the callback the RT application decides, if there is enough time left for serving a subsequent service method. If so, it calls another ProcessNextMethodCall.

返回的ara::core::Future用于在将来返回结果，从而实现异步处理。它允许您在“下一个请求”被处理时得到通知。
这对于`按顺序链接服务方法调用`可能很有帮助。
一个典型的实时应用程序的简单用例可能是：
- 实时应用程序被`调度`。
- 它调用`ProcessNextMethodCall`并在`ara::core::Future::then()`中注册一个回调函数。
- 在中间件调用的服务方法对应的未完成请求处理完毕后，回调函数被调用。
- 在回调函数中，实时应用程序`决定 是否 有足够的时间 来处理后续的服务方法`。
    如果有，它会调用另一个`ProcessNextMethodCall`。
    这种机制可以帮助实现服务方法的链式调用，以便更有效地处理多个请求。

Sure - this simple example assumes, that the RT application knows worst case runtime of its service methods (and its overall time slice), but this is not that unlikely! The bool value of the returned ara::core::Future is set to true by the Communication Management in case there really was an outstanding request in the queue, which has been dispatched, otherwise it is set to false. This is a somewhat comfortable indicator to the application developer, not to call repeatedly ProcessNextMethodCall although the request queue is empty. So calling ProcessNextMethodCall directly after a previous call returned an ara::core::-
Future with the result set to false might most likely do nothing (except that incidentally in this minimal time frame a new request came in).
Note that the binding implementation is free to decide, whether it dispatches the method call event to your service method implementation within the thread context in which you called ProcessNextMethodCall, or whether it does spawn a separate thread for this method call.

这个简单的例子假设实时应用程序知道其`服务方法`的`最坏运行时间（以及总体时间片）`，但这并非不可能！
**返回的ara::core::Future的布尔值由通信管理设置为true，表示队列中确实有一个未完成的请求已经被处理，否则设置为false。**
这是一个相对方便的指示，告诉应用程序开发人员，即使请求队列为空，也不要重复调用ProcessNextMethodCall。
因此，在之前的调用返回一个结果为`false`的`ara::core::Future`后，直接调用`ProcessNextMethodCall`可能大多数情况下不会执行任何操作（除非在这个极短的时间段内恰好有新的请求进来）。
请注意，绑定实现可以自由决定是否在调用`ProcessNextMethodCall`的线程上下文中，将方法调用事件分派到您的服务方法实现中，或者是否为此方法调用生成一个单独的线程。

`AUTOSAR Binding Implementer Hint`
The explanation up to this point regarding the request processing mode `MethodCallProcessingMode.kPoll` will have a huge impact on the binding implementation!
The fundamental idea of this mode to rule out context switches to a process containing a service implementation caused by Communication Management events (incoming service method calls) has some consequences for AP products based on typical operating systems: There are constraints for the location of the queue, which has to collect the service method call requests until they are consumed by the polling service implementation.
The queue must be realized either outside of the address space of the service provider application or it must be located in a shared memory like location, so that the sending part is able to write directly into the queue. Typical solutions of placing the queue outside of the service provider address space would be
• Kernel space. If the binding implementation would use socket or pipe mechanisms, the kernel buffers being the target of the write-call would resemble the queue. Adapting/configuring maximal sizes of those buffers might in typical OS mean recompiling the kernel.
• User address space of a different binding/Communication Management demon-application. Buffer space allocation for queues allocated within user space could typically be done more dynamic/flexible.
In comparison to a shared memory solution the access from the polling service provider to those queue location might come with higher costs/latency.

到目前为止关于请求处理模式`MethodCallProcessingMode.kPoll`的解释对于绑定实现将产生巨大影响！
这种模式的基本思想是：`排除`由通信管理事件（传入的服务方法调用）引起的对包含服务实现的进程的`上下文切换`，这对基于典型操作系统的AP产品有一些影响：对于收集服务方法调用请求直到它们被轮询服务实现消耗的队列的位置有一些约束。
队列必须要么实现在服务提供者应用程序的地址空间之外，要么必须位于共享内存位置，以便发送方能够直接写入队列。
将队列放在服务提供者地址空间之外的典型解决方案可能包括：
- 内核空间。
  如果绑定实现使用套接字或管道机制，作为写调用目标的内核缓冲区将类似于队列。
  调整/配置这些缓冲区的最大大小在典型操作系统中可能意味着重新编译内核。
- 不同绑定/通信管理守护应用程序的用户地址空间。
  在`用户空间`内`分配用于队列的缓冲区空间`可以更加动态/灵活地完成。
与共享内存解决方案相比，从轮询服务提供者到队列位置的访问可能会带来更高的成本/延迟。
`ProcessNextMethodCall()`，用于处理当前队列头部的一个任务
// provided service instance端，处理future队列中的任务
// 返回值的true：表示任务执行成功，false表示失败、也表示没任务
```cpp
    virtual ara::core::Future<bool> ProcessNextMethodCall() override
    {
        if (delegates_.size() != 0) {
            std::list<ServiceBase*>::iterator last_iterator = it_;
            do {
                if (it_ == delegates_.end()) {
                    it_ = delegates_.begin();
                }

                ara::core::Future<bool> result = (*it_)->ProcessNextMethodCall();   // ProcessNextMethodCall()是skeleton端需要重载的函数
                it_++;

                if (!result.is_ready()) {
                    return result;
                } else if (result.get()) {
                    ara::core::Promise<bool> promise;
                    promise.set_value(true);
                    return promise.get_future();
                }
            } while (it_ != last_iterator);
        }

        ara::core::Promise<bool> promise;
        promise.set_value(false);
        return promise.get_future();
    }
```
在`ProcessNextMethodCall()`方法中，根据底层通信协议的不同，一般应包含如下步骤：
- 将method`request`数据`反序列化`
- 将反序列化后的参数列表传入`用户的method实现`，得到一个Future：在用户的方法实现可能是同步的；也可能是异步的
通过用户method实现返回的Future`拿到处理结果`
- 将这个结果`序列化`成`response`，发送给Proxy
- 检查`是否存在下一个method请求`，如果是返回true，否则返回false

在以上的步骤中，有两次返回Future：`用户method处理实现返回`和`ProcessNextMethodCall`返回，这两处只要有一处是异步的，则整个method请求的处理就是异步的。
序列化、反序列化、检查是否有下一个请求等步骤 可以在ProcessNextMethodCall的当前线程，也可以异步执行；
用户method处理方法中，可以先返回Future，然后异步执行处理，也可以在ProcessNextMethodCall同一个线程中同步完成。ProcessNextMethodCall所在的线程、序列化/反序列化/调用用户实现/返送返回值的线程、用户实际处理method的线程，这三个上下文可以是同一个线程，可以是两个线程，也可以是三个不同的线程。


#### 6.3.3.2 Event-Driven Mode
If you set the processing mode to `kEvent` or `kEventSingleThread`, the Communication Management implementation will `dispatch events asynchronously` to the service method implementations at the time the service call from the service consumer comes in.

Opposed to the `kPoll` mode, here the service consumer implicitly controls/triggers service provider process activations with their method calls!
What is then the difference between kEvent and kEventSingleThread? kEvent means, that the Communication Management implementation may call the service method implementations concurrently.
That means for our example: If — at the same point in time — one call to method Calibrate and two calls to method Adjust arrive from different service consumers, the Communication Management implementation is allowed to take three threads from its internal thread-pool and do those three calls for the two service methods concurrently.
那么kEvent和kEventSingleThread之间的区别是什么呢？
**`kEvent`表示通信管理实现可以`并发 调用服务方法实现`。**
对于我们的例子来说：如果在同一时间点，来自不同服务消费者的一个对Calibrate方法的调用和两个对Adjust方法的调用同时到达，通信管理实现可以从其内部线程池中获取三个线程，并同时执行这两个服务方法的三个调用。

On the contrary the mode `kEventSingleThread` assures, that on the service instance `only one service method at a time will be called by` the Communication Management implementation.
That means, Communication Management implementation has to `queue incoming service method call events for the same service instance and dispatch them one after the other.`
Why did we provide those two variants? 
From a functional viewpoint only `kEvent` would have been enough! 
A service implementation, where certain service methods could not run concurrently, because of shared data/consistency needs, could simply do its synchronization (e.g. via std::mutex) on its own!
The reason is “efficiency”. If you have a service instance implementation, which has extensive synchronization needs, i.e. would synchronize almost all service method calls anyways, it would be a total waste of resources, if the Communication Management would “spend” N threads from its thread-pool resources, which directly after get a hard sync, sending N-1 of it to sleep.
`！！！为什么要提供单线程模式？？？`
一个服务实现，由于`共享数据/一致性需求`，某些服务方法`不能并发运行`，可以简单地在自己的服务方法上进行同步（例如通过std::mutex）！原因是“效率”。
如果您有一个服务实例实现，它具有广泛的同步需求，即几乎所有服务方法调用都需要同步，那么如果通信管理“花费”了N个线程从其线程池资源中，然后直接进行严格的同步，**将N-1个线程置于休眠状态，这将是一种资源的浪费**。
（PS：大量的线程间同步，使得并行退化为串行了）

For service implementations which lie in between — i.e. some methods can be called concurrently without any sync needs, some methods need at least partially synchronization — the service implementer has to decide, whether he uses kEvent and does synchronization on top on his own (possibly optimizing latency, responsiveness of his service instance) or whether he uses kEventSingleThread, which frees him from synchronizing on his own (possibly optimizing ECU overall throughput).

对于介于两者之间的服务实现——即一些方法可以并发调用而无需同步，而一些方法至少部分需要同步——服务实现者必须决定是使用`kEvent`并在自己的基础上进行同步（可能优化其服务实例的延迟和响应性），还是使用`kEventSingleThread`，这样就不需要自行进行同步（可能优化ECU的整体吞吐量）。
这取决于服务实现者对于性能和资源利用的不同考量，以满足其特定的需求和优化目标。

### 6.3.4 Methods
Service methods on the skeleton side are abstract methods, which have to be `overwritten` by the service implementation sub-classing the skeleton. Let’s have a look at the Adjust method of our service example:
```cpp
struct AdjustOutput {
  bool success;
  Position effective_position;
};

virtual ara::core::Future<AdjustOutput> Adjust(const Position& position) = 0;
```
在这种情况下，服务方法的抽象定义中的输入参数直接映射到骨架抽象方法签名的方法参数中。
在这里，它是来自类型Position的position参数，由于它是一个`非原始类型`，因此被建模为`“const ref”`。
方法签名中有趣的部分是返回类型。
服务方法的实现必须返回我们之前讨论过的ara::core::Future。
这个想法很简单：我们不想强迫服务方法的实现者通过简单返回这个“入口点”方法来表示服务方法的完成。
通过返回`ara::core::Future`，可以更灵活地处理服务方法的完成状态，而不仅仅是通过简单的返回值来表示。

也许服务实现者决定将服务调用的实际处理分派给一个中央工作线程池！
当“入口点”方法的返回信号服务调用的完成给通信管理时，这将会非常麻烦。
在我们的工作线程池场景中，我们将不得不在服务方法内部的某个等待点阻塞，并等待来自工作线程的通知，表明它已经完成，然后我们才会从服务方法返回。
`在这种情况下，我们将在服务方法内部有一个被阻塞的线程！`
从现代多核CPU的高效利用的角度来看，这是不可接受的。
因此，通过返回`ara::core::Future`，可以避免在服务方法内部阻塞线程，提高CPU的利用率和系统的性能。

(这段描述指出了在服务方法实现中避免阻塞线程的重要性。通过返回ara::core::Future而不是在服务方法内部阻塞线程等待处理完成的通知，可以提高系统的效率和性能。如果服务方法的实现需要将实际处理分派给工作线程池，通过Future可以更好地管理异步操作，避免在服务方法内部阻塞线程。这种设计可以更好地利用现代多核CPU的性能，避免不必要的资源浪费和性能下降。因此，通过返回Future，服务实现者可以更灵活地处理服务调用的完成状态，同时保持系统的高效性和性能。
主要说明用future的好处
)
The returned ara::core::Future contains a structure as template parameter, which aggregates all the OUT-parameters of the service call.
The following two code examples show two variants of an implementation of Adjust.
- In the first variant the service method is directly processed synchronously in the method body, so that an ara::core::Future with an already set result is returned, 
- while in the second example, the work is dispatched to an asynchronous worker, so that the returned ara::core::Future may not have a set result at return.

(The referenced object is provided by the Communication Management implementation until the service method call has set its promise (valid result or error). If the service implementer needs the referenced object beyond that, he has to make a copy.)

返回的ara::core::Future包含一个结构作为模板参数，该结构聚合了服务调用的所有输出参数。
下面的两个代码示例展示了Adjust方法的两种实现变体。
在第一种变体中，服务方法`直接在函数体中同步处理`，因此返回一个已经设置结果的`ara::core::Future`；
```cpp
class RadarServiceImpl : public RadarServiceSkeleton {
public:
  Future<AdjustOutput> Adjust(const Position& position) {
    ara::core::Promise<AdjustOutput> promise;
    struct AdjustOutput out = doAdjustInternal(position, &out.effective_position);
      promise.set_value(out);
    // we return a future from an already set promise...
    return promise.get_future();
  }
};
```

而在第二个示例中，`工作被分派给一个异步工作线程`，因此返回的ara::core::Future在返回时可能没有设置结果。
（通信管理实现提供了引用对象，直到服务方法调用设置了其承诺（有效结果或错误）。
如果服务实现者需要在此之后使用引用对象，他必须进行复制。）
```cpp
Future<AdjustOutput> Adjust(const Position& position) {
  ara::core::Promise<AdjustOutput> promise;
  auto future = promise.get_future();
  std::thread t(
      [this] (const Position& pos, ara::core::Promise prom) {
        prom.set_value(doAdjustInternal(pos));
      },
      std::cref(position), std::move(promise)).detach();

    return future;
}
```

#### 6.3.4.1 One-Way aka Fire-and-Forget Methods
“One-way/fire-and-forget” methods on the server/skeleton side do have (like on the proxy side) a simpler signature compared to normal methods.
Since there is no feedback possible/needed towards the caller it is a simple void method:
```cpp
virtual void LogCurrentState() = 0;
```


#### 6.3.4.2 Raising Application Errors
Whenever on the implementation side of a service method, an `ApplicationError` — according to the interface description — is detected, `the Checked Exception representing this ApplicationError simply has to be stored into the Promise`, from which the Future is returned to the caller:
```cpp
Future<CalibrateOutput> Calibrate(const std::string& configuration) {
  ara::core::Promise<CalibrateOutput> promise;
  auto future = promise.get_future();
  if (!checkConfigString(configuration)) {
      // given arg is invalid:
      // assume that in ARXMLs we have ErrorDomain with name SpecificErrors
      // which contains InvalidConfigString error.
      // Note that numeric error code will be casted to ara::core::ErrorCode implicitly. promise.SetError(SpecificErrorsErrc::InvalidConfigString); }
  }
  else {...}

return future;
}
```

### 6.3.5 Events
On the skeleton side the service implementation is in charge of `notifying about occurrence of an event`.
As shown in 6.15 the skeleton provides a member of an event wrapper class per each provided event.
The event wrapper class on the skeleton/event provider side looks obviously different than on the proxy/event consumer side.
```cpp
class BrakeEvent {
public:
  using SampleType = RadarObjects;

  void Send(const SampleType& data);

  ara::com::SampleAllocateePtr<SampleType> Allocate();

  /*
    After sending data, 
    you loose ownership
    and can’t access the data through the SampleAllocateePtr anymore.
    Implementation of SampleAllocateePtr will be with the semantics of std::unique_ptr (see types.h)
  */
  void Send(ara::com::SampleAllocateePtr<SampleType> data);
};
```
The using directive — analogue to the Proxy side — just introduces the common name SampleType for the concrete data type of the event. We provide two different variants of a “Send” method, which is used to send out new event data. The first one takes a reference to a SampleType.
This variant is straight forward: 
The event data has been `allocated` somewhere by the service application developer and is given via reference to the binding implementation of `Send()`.
After the call to send returns, the data might be removed/altered on the caller side. The binding implementation will make a copy in the call.
The second variant of ’Send‘ also has a parameter named “data”, but this is now of a different type `ara::com::SampleAllocateePtr<SampleType>`. According to our general approach to only provide abstract interfaces and eventually provide a proposed mapping to existing C++ types (see section 5.3) this pointer type, we introduced here, shall behave like a `std::unique_ptr<T>`.
这段文本描述了在一个系统中使用代理模式时的一种情况。
在这个系统中，使用指令指示符（using directive）引入了一个名为 SampleType 的通用名称，用于表示事件的具体数据类型。
系统提供了两种不同的“Send”方法变体，用于发送新的事件数据。
* 第一种变体接受一个`对 SampleType 的引用`。
  这种方式很直接：事件数据 由服务应用程序开发人员在某处分配，并通过引用传递给`Send()`的绑定实现。
  在调用 Send 后，数据可能会在调用方被移除/更改。绑定实现将在调用中进行复制。

* 第二种“Send”方法的变体也有一个名为“data”的参数，但这次是一个不同类型的参数 `ara::com::SampleAllocateePtr<SampleType>`。
  根据系统提供的一般方法，只提供抽象接口并最终提供对现有 C++ 类型的映射，这里引入的指针类型应该表现得像一个 `std::unique_ptr<T>`。

That roughly means: Only one party can hold the pointer - if the owner wants to give it away, he has to explicitly do it via `std::move`.
So what does this mean here? Why do we want to have `std::unique_ptr<T>` semantics here?
To understand the concept, we have to look at the third method within the event wrapper class first:
```cpp
ara::com::SampleAllocateePtr<SampleType> Allocate();

// 旧版VECTOR的实现：
// VECTOR Next Line VectorC++-V6-6-1: MD_SOCAL_VectorC++-V6-6-1_AbortWithSingleReturnStatement
::ara::com::SampleAllocateePtr<SampleType> Allocate() noexcept { return std::make_unique<SampleType>(); }
```
The event wrapper class provides us here with a method to `allocate memory` for one sample of event data.
It returns a smart pointer `ara::com::SampleAllocateePtr <SampleType>`, which points to the allocated memory, where we then can write an event data sample to.
And this returned smart pointer we can then give into an upcoming call to the second version of “Send”.
So — the obvious question would be — why should I let the binding implementation do the memory allocation for event data, which I want to notify/send to potential consumers?
The answer simply is: `Possibility for optimization of data copies.`
The following “over-simplified” example makes things clearer: Let’s say the event, which we talk about here (of type RadarObjects), could be quite big, i.e. it contains a vector, which can grow very large (say hundreds of kilobytes).
In the first variant of “Send”, you would allocate the memory for this event on your own on the `heap` of your own application process.
Then — during the call to the first variant of “Send” — the binding implementation has to copy this event data from the (private) process heap to a memory location, where it would be accessible for the consumer.
If the event data to copy is very large and the frequency of such event occurrences is high, the sheer runtime of the data copying might hurt.
The idea of the combination of `Allocate()` and the second variant to send event data (`Send(SampleAllocateePtr<SampleType> data)`) is to eventually avoid this copy!
A smart binding implementation might implement the Allocate() in a way, that it allocates memory at a location, where writer (service/event provider) and reader (service/event consumer) can both directly access it! So an `ara::com::SampleAllocateePtr<SampleType>` is a pointer, which points to memory nearby the receiver.
Such locations, where two parties can both have direct access to, are typically called `“shared memory”.`
The access to such regions should — for the sake of data consistency — be synchronized between readers and writers.
This is the reason, that `the Allocate()` method returns such a smart pointer with the aspects of single/solely user of the data, which it points to: After the potential writer (service/event provider side) has called `Allocate()`, he can access/write the data pointed to as long as he hands it over to the second send variant, where he explicitly gives away ownership!
This is needed, because after the call, the readers will access the data and need a consistent view of it.
这段文本描述了事件包装类提供的一种方法，用于为事件数据的一个样本分配内存。
它返回一个智能指针 `ara::com::SampleAllocateePtr <SampleType>`，指向已分配的内存，我们可以在其中写入事件数据样本。然后，我们可以将这个返回的智能指针传递给下一个调用第二版本的“Send”。
为什么要让绑定实现来为我想要通知/发送给潜在消费者的事件数据进行内存分配呢？答案很简单：数据拷贝的优化可能性。
举个简单的例子来解释：假设我们这里讨论的事件（类型为 RadarObjects）可能非常庞大，比如包含一个可以变得非常大（比如数百千字节）的向量。
在第一个“Send”变体中，您将在您自己应用程序进程的堆上为此事件分配内存。然后，在调用第一个“Send”时，绑定实现必须将此事件数据从（私有）进程堆复制到一个内存位置，以便消费者可以访问。
如果要复制的事件数据非常大，并且事件发生的频率很高，那么数据复制的运行时间可能会影响性能。
`Allocate()` 和第二个发送事件数据的变体（`Send(SampleAllocateePtr<SampleType> data)`）的组合的想法是`最终避免这种复制`！
一个智能的绑定实现可能会以一种方式实现 `Allocate()`，在这种方式下，它会在一个位置分配内存，写入者（服务/事件提供者）和读取者（服务/事件消费者）都可以直接访问！因此，`ara::com::SampleAllocateePtr<SampleType>` 是一个指针，指向接收者附近的内存。
这种双方都可以直接访问的位置通常被称为“共享内存”。为了数据一致性，这些区域的访问应该在读取者和写入者之间进行同步。
这就是为什么 `Allocate()` 方法返回具有单一用户/`独占`用户方面的智能指针的原因：潜在的写入者（服务/事件提供者方）调用 `Allocate()` 后，他可以访问/写入指向的数据，只要他将其移交给第二个发送变体，那里他明确放弃所有权！
这是必要的，因为在调用之后，读取者将访问数据并需要一个一致的视图。

```cpp
using namespace ara::com;

RadarServiceImpl myRadarService;

// Handler called at occurrence of a BrakeEvent
void BrakeEventHandler() {
  // let the binding allocate memory for event data...
  SampleAllocateePtr<BrakeEvent::SampleType> curSamplePtr = myRadarService.BrakeEvent.Allocate();
  
  // fill the event data ...
  curSamplePtr->active = true;
  fillVector(curSamplePtr->objects);
  
  // Now notify event to consumers ...
  myRadarService.BrakeEvent.Send(std::move(curSamplePtr));

  // Now any access to data via curSamplePtr would fail -
  // we’ve given up ownership!
}
```

`AUTOSAR Binding Implementer Hint`
The idea behind the concept of providing a binding specific “Allocate” functionality was greatly driven by the “zero-copy” buzzword.
Having a shared memory based IPC transport mechanism the “zero-copy” axiom might be easily fulfill-able at first glance. The `ara::com::SampleAllocateePtr<SampleType>` mechanism foresees/assumes a hard synchronization between readers/writers anyway, so the challenge in implementation isn’t that big, if the platform provides shared memory concepts anyway.
But in reality you have to be aware of serialization needs (section 9.1), which can ruin any “zero-copy” attempts, which we did also hint at explicitly in subsection 9.1.1.
The work-around would be to either rule out serialization needs between ara::com communication partners in an deployment by prescribing compile settings in a way, that exchanged data types are binary compatible or at least to implement some smart checking logic to detect, between which ara::com communication partners in fact serialization is not needed.

这一理念主要受到`“zero-copy”`这个概念的推动。
通过基于`共享内存`的IPC传输机制，一开始看起来“zero-copy”是可以轻松实现的。
`ara::com::SampleAllocateePtr<SampleType>`机制预见/假设读者/写者之间存在严格的同步，因此在实现上的挑战并不大，只要平台本身提供了共享内存的概念。
但实际上，你必须意识到`序列化的需求`（第9.1节），这可能会破坏任何“zero-copy”的尝试，我们在第9.1.1小节中也明确指出了这一点。解决方法可以是在部署中通过编译设置规定交换数据类型是二进制兼容的，或者至少实现一些智能检查逻辑来检测哪些 ara::com 通信伙伴之间实际上不需要序列化。



### 6.3.6 Fields
On the skeleton side the service implementation is in charge of
* updating and notifying about changes of the value of a field.
* serving incoming Get() calls.
* serving incoming Set() calls.
As shown in 6.15 the skeleton provides a member of a field wrapper class per each provided field. The field wrapper class on the skeleton/field provider side looks obviously different than on the proxy/field consumer side.
On the service provider/skeleton side the service specific field wrapper classes are defined within the namespace fields directly beneath the namespace skeleton. Let’s have a deeper look at the field wrapper in case of our example event UpdateRate:
```cpp
class UpdateRate {
public:
  using FieldType = uint32_t;

  void Update(const FieldType& data); // 等价于event中的Send()

/*
  If no Getter is registered ara::com is responsible for responding to the request using the last value set by update.
  This implicitly requires at least one call to update after initialization of the Service, before the service is offered. This is up to the implementer of the service.
  如果没有注册Getter，ara::com 将负责使用 update 设置的最后一个值来响应请求。
  这就隐含地要求在初始化服务后、提供服务前至少调用一次 update。这取决于服务的实现者。
*/ 
  void RegisterGetHandler(std::function<ara::core::Future<FieldType>()> getHandler);

/*
  Registering a SetHandler is mandatory, if the field supports it.
  The handler gets the data the sender requested to be set. It has to validate the settings and perform an update of its internal data. The new value of the field should than be set in the future.
  The returned value is sent to the requester and is sent via notification to all subscribed entities.
*/
  void RegisterSetHandler(std::function<ara::core::Future<FieldType>(const FieldType& data)> setHandler);
};
```
The using directive — again as in the Event Class and on the Proxy side — just introduces the common name FieldType for the concrete data type of the field.
We provide an `Update `method by which the service implementer can update the current value of the field.
It is very similar to the simple/first variant of the `Send` method of the event class: The field data has been allocated somewhere by the service application developer and is given via reference to the binding implementation of Update.
After the call to Update returns, the data might be removed/altered on the caller side. The binding implementation will make a (typically serialized) copy in the call.
In case “on-change-notification” is configured for the field, notifications to subscribers of this field will be triggered by the binding implementation in the course of the Update call.
通过using——就像在事件类和代理端一样——引入了一个通用名称 FieldType，用于表示字段的具体数据类型。
我们提供了一个 Update 方法，通过这个方法，服务的实现者可以更新字段的当前值。
这个方法与事件类的简单/第一个变体的 Send 方法非常相似：
field数据在服务应用程序开发人员的某个地方被分配，并通过引用传递给 Update 的绑定实现。
在调用 Update 后，数据可能会在调用方被移除/修改。绑定实现将在调用中创建一个（通常是序列化的）副本。
如果为字段配置了“`在变化时通知`”，则在 Update 调用过程中，绑定实现将触发对该字段订阅者的通知。

#### 6.3.6.1 Registering Getters
The `RegisterGetHandler` method provides the possibility to register a method implementation by the service implementer, which gets then called by the binding implementation on an incoming Get() call from any proxy instance.
The RegisterGetHandler method in the generated skeleton does only exist in case availability of “field getter” has been configured for the field in the IDL!（模型中配置的话，才生成相应的函数接口）
`Registration` of such a “GetHandler” is fully `optional`! 
Typically there is no need for a service implementer to provide such a handler. The binding implementation always has access to the latest value, which has been set via Update. So any incoming Get() call can be served by the Communication Management implementation standalone.
A theoretical reason for a service implementer to still provide a “GetHandler” could be: Calculating the new/current value of a field is costly/time consuming. Therefore the service implementer/field provider wants to defer this process until there is really need for that value (indicated by a getter call). In this case he could calculate the new field value within its “GetHandler” implementation and give it back via the known ara:com promise/future pattern.
If you look at the bigger picture, then such a setup with the discussed intention, where the service implementer provides and registers a “GetHandler” will not really make sense, if the field is configured with “on-change-notification”, too.
In this case, new subscribers will get potentially outdated field values on subscription, since updating of the field value is deferred to the explicit call of a “GetHandler”.
通常情况下，服务实现者无需提供这样的处理程序。
绑定实现始终可以访问通过 Update 设置的最新值。因此，任何传入的 Get() 调用都可以由通信管理实现独立提供。
服务实现者仍需提供 “GetHandler ”的一个理论原因可能是 `计算field的新值/当前值耗费大量时间`。
因此，服务实现者/字段提供者希望将这一过程推迟到真正需要该值（由获取器调用指示）时再进行。
在这种情况下，他可以在其 “GetHandler ”实现中计算新的字段值，并通过已知的 ara:com promise/future 模式将其返回。
从更大的角度来看，如果field也被配置为 “on-change-notification”（变更时通知），那么服务实现者提供并注册一个 “GetHandler”（获取处理程序）的这种设置就不太合理了。
在这种情况下，`由于字段值的更新会推迟到显式调用 “GetHandler”时进行，因此新用户在订阅时可能会得到过时的field值`。

You also have to keep in mind: In such a setup, with enabled “on-change-notification” together with a registered “GetHandler” the Communication Management implementation will not automatically care for, that the value the developer returns from the “GetHandler” will be synchronized with value, which subscribers get via “on-change-notification” event!
If the implementation of “GetHandler” does not internally call Update() with the same value, which it will deliver back via ara:com promise, then the field value delivered via “on-change-notification” event will differ from the value returned to the Get() call. I.e.
the Communication Management implementation will not automatically/internally call Update() with the value the “GetHandler” returned.
Bottom line: Using RegisterGetHandler is rather an exotic use case and developers should be aware of the intrinsic effect.
Additionally a user provided “GetHandler”, which only returns the current value, which has already been updated by the service implementation via Update(), is typically very inefficient! The Communication Management then has to call to user space and to additionally apply field serialization of the returned value at any incoming Get() call.
Both things could be totally “optimized away” if the developer does not register a “GetHandler” and leaves the handling of Get() calls entirely to the Communication Management implementation.


您还必须记住： 在这种情况下，如果启用了“on-change-notification”（变更通知）和已注册的 “GetHandler”（获取处理程序），通信管理实现将不会自动考虑开发人员`从 “GetHandler”（获取处理程序）返回的值是否与订阅者通过 “on-change-notification”（变更通知）事件获取的值同步`！
如果 “GetHandler”的实现没有在内部使用相同的值调用`Update()`（它将通过 ara:com 承诺返回该值），那么通过 “on-change-notification”事件返回的字段值将与调用`Get()`返回的值不同。
也就是说`通信管理实现不会自动/内部调用 Update() 并使用 “GetHandler”返回的值`。
一句话 使用 RegisterGetHandler 是一个比较特殊的用例，开发人员应注意其内在影响。
此外，用户提供的 “GetHandler ”只返回服务实现已通过 Update() 更新的当前值，效率通常很低！通信管理必须调用用户空间，并在任何传入的 Get()调用中额外应用返回值的字段序列化。
如果开发人员不注册 “GetHandler”，而是将 Get() 调用的处理完全交给通信管理实现，那么这两件事都可以完全 “优化掉”。


#### 6.3.6.2 Registering Setters
Opposed to the RegisterGetHandler the RegisterSetHandler API has to be called by the service implementer in case it exists (i.e. field has been configured with setter support).
The reason, that we decided to make the registration of a “GetHandler” mandatory is simple: We expect, that the server implementation will always need to check the validity of a new/updated field values set by any anonymous client. A look at the signature of the “SetHandler” `std::function<ara::core::Future<FieldType>(const FieldType& data)>` reveals that the registered handler does get the new value as input argument and is expected to return also a value.
The semantic behind this is: In case the “SetHandler” always has to return the effective (eventually replaced/corrected) value. This allows the service side implementer to validate/overrule the new field value provided by a client.
The effective field value returned by the “SetHandler” is implicitly taken over by the Communication Management implementation as if the service implementer had called Update() explicitly with the effective value on its own. That means: An explicit Update() call within the “SetHandler” is superfluous as the Communication Management would update the field value with the returned value of the “SetHandler” anyways.

相对于 RegisterGetHandler，`RegisterSetHandler` API 必须由服务实现者调用（如果存在的话，即字段已配置支持设置操作）（即必须set）。
注册“GetHandler”是`强制性`的原因很简单：
我们期望 服务器实现 始终需要 检查由任何匿名客户端设置的 新/更新field值的有效性。
通过观察“SetHandler”的签名 `std::function<ara::core::Future<FieldType>(const FieldType& data)>`，可以看到注册的处理程序会将新值作为输入参数，并且预期也会返回一个值。
这背后的语义是：`SetHandler` 必须始终返回`有效的`（可能被替换/校正的）值。
这使得服务端实现者可以验证/覆盖客户端提供的新字段值。
由 SetHandler 返回的有效字段值会被通信管理实现隐式接管，就好像服务实现者自己显式调用了`Update()`并传入了有效值一样。
这意味着：在 SetHandler 中显式调用 Update() 是多余的，因为`通信管理会使用 SetHandler 返回的值更新字段值`。


#### 6.3.6.3 Ensuring existence of “SetHandler”
The existence of a registered “SetHandler” is ensured by an ara:com compliant implementation by returning an unchecked error: If a developer calls OfferService() on a skeleton implementation and had not yet registered a “SetHandler” for each of its fields, which has setter enabled, the Communication Management implementation shall return an unchecked error indicating this programming error.
符合 ara:com 规范的实现会通过返回`unchecked错误`来确保已注册的“SetHandler”的存在： 
  如果开发者在骨架实现上调用`OfferService()`，但尚未为每个启用了`setter`的字段注册 “SetHandler”，则通信管理实现应返回一个未选中的错误，表明编程错误。

#### 6.3.6.4 Ensuring existence of valid Field values
Since the most basic guarantee of a field is, that it has a valid value at any time, ara:com has to somehow ensure, that a service implementation providing a field has to provide a value before the service (and therefore its field) becomes visible to potential consumers, which — after subscription to the field — expect to get initial value notification event (if field is configured with notification) or a valid value on a Get call (if getter is enabled for the field).
An ara::com Communication Management implementation needs therefore behave in the following way: If a developer calls OfferService() on a skeleton implementation and had not yet called Update() on any field, which
* has notification enabled
* or has getter enabled but not yet a “GetHandler” registered
the Communication Management implementation shall return an unchecked error indicating this programming error.
Note: The AUTOSAR meta-model supports the definition of such initial values for a field in terms of a so called FieldSenderComSpec of a PPortPrototype. So this model element should be considered by the application code calling `Update()`.

由于`field`最基本的保证是在任何时候都具有有效值，ara::com 必须确保服务实现在服务（以及其字段）对潜在消费者可见之前必须提供一个值。
消费者在订阅字段后，期望收到初始值通知事件（如果字段配置了通知）或在 Get 调用时收到有效值（如果字段启用了 Getter）。
ara::com 的通信管理实现需要按照以下方式行事：如果开发人员在骨架实现上调用`OfferService()`，但尚未对任何字段调用 Update()，而这些字段
* 启用了通知
* 或启用了 Getter 但尚未注册“GetHandler”
通信管理实现应返回一个未经检查的错误，指示这个编程错误。
值得注意的是：AUTOSAR AP元模型支持通过 `PPortPrototype` 的 `FieldSenderComSpec` 定义字段的初始值。
因此，应用代码在调用 `Update()` 时应考虑这个模型元素。
也就是说，skeleton端，要在`OfferService`之前，在SetHandler那些初始化阶段就调用`Update`来设置初值



#### 6.3.6.5 Access to current field value from Get/SetHandler
Since the underlying field value is only known to the middleware, the current field value is not accessible from the “`Get/SetHandler`” implementation, which are on application level.
If the “Get/SetHandler” needs to read the current field value, the skeleton implementation must provide a field value replica accessible from application level.
用户层无法直接获取当前的初值，因为用户只能通过用户层回调获取（skeleton底层实现要把维护的底层的field的值拷贝出来）。




## 6.4 Runtime
Note: `A singleton` called Runtime may be needed to collect cross-cutting functionalities.
Currently there are no requirements for such functionalities, so this chapter is empty. This might change until the 1st release.









# 8 Raw Data Streaming Interface
## 8.1 Introduction
The Adaptive AUTOSAR Communication Management is based on `Service Oriented communication`.
This is good for implementing platform independent and dynamic applications with a service-oriented design. For ADAS applications, it is important to be able to transfer raw binary data streams over Ethernet efficiently between applications and sensors, where service oriented communication (e.g. SOME/IP, DDS) either creates unnecessary overhead for efficient communication, or the sensors do not even have the possibility to send anything but
raw binary data.
The Raw Data Binary Stream API provides a way to send and receive Raw Binary Data Streams, which are sequences of bytes, without any data type. They enable efficient communication with external sensors in a vehicle (e.g. sensor delivers video and map data in "Raw data" format). The communication is performed over a network using sockets.
From the ara::com architecture point of view, Raw Data Streaming API is static, i.e. its is not generated. It is part of the ara::com namespace, but is independent of the ara::com middleware services.
Adaptive AUTOSAR Communication Management基于“面向服务的通信”。
这对于实现基于服务设计的平台无关和动态应用程序非常有用。
对于`ADAS应用程序`，能够在`应用程序`和`传感器之间`高效地通过以太网传输`原始二进制数据流`非常重要，而面向服务的通信（例如SOME/IP、DDS）可能会为高效通信带来不必要的开销，或者传感器甚至没有发送除原始二进制数据之外的任何可能性。
`原始数据二进制流API`提供了一种发送和接收原始二进制数据流的方式，这些数据流是`字节序列`，`没有任何数据类型`。它们使得能够与车辆中的外部传感器进行高效通信（例如，传感器以“原始数据”格式提供视频和地图数据）。
通信是通过`套接字`在`网络上`执行的。
从ara::com架构的角度来看，原始数据流API是静态的，即它不是生成的。
`它是ara::com命名空间的一部分，但独立于ara::com中间件服务。



### 8.1.1 Functional description
`The Raw Data Binary Stream API` can be used in both the client or the server side.
The functionality of both client and server allow to send and receive.
The only difference is that `the server` can `wait for connections` but `cannot actively connect to a client`.
On the other side, the client can `connect to a server` (`that is already waiting for connections`)
but the client cannot wait for connections.
The usage of the Raw Data Binary Streams API from Adaptive Autosar must follow this sequence:
* As client
1. `Connect`:
  Establishes connection to sensor
2. `ReadData/WriteData`:
  Receives or sends data
3. `Shutdown`:
  Connection is closed.

* As server
1. `WaitForConnection`:
  Waits for incoming connections from clients
2. `ReadData/WriteData`:
  Receives or sends data
3. `Shutdown`:
  Connection is closed and stops waiting for connections.

## 8.2 Class and Model
### 8.2.1 Class and signatures
The class `ara::com::raw` defines a `RawDataStream` class for reading and writing binary data streams over a network connection using sockets. The client side is an object of the class class ara::com::raw::RawDataStreamClient and the server side is `ara::com::raw::RawDataStreamServer`

#### 8.2.1.1 Constructor
The constructor takes as input the instance specifier qualifying the network binding and parameters for the instance.
```cpp
RawDataStreamClient(const ara::com::InstanceSpecifier& instance);
RawDataStreamServer(const ara::com::InstanceSpecifier& instance);
```

#### 8.2.1.2 Destructor
Destructor of RawDataStream.
If the connection is still open, it will be shut down before destroying the RawDataStream object.
```cpp
~RawDataStreamClient();
~RawDataStreamServer();
```











# 9 Appendix
本节主要面向 ara::com 绑定实现者和 AP 产品供应商。
因此，我们没有把所有内容都放在一个方框中，而是在前面的注释中加以说明。
当然，我们也欢迎 ara::com API 用户阅读本节。

序列化（见 [7]）是将`某些数据结构`转换为`标准格式`，以便在发送方与接收方（可能是不同的接收方）之间交换的过程。
从一个网络节点向另一个网络节点传输数据时，通常会有这种概念。
当把数据放在网线上并读取回来时，你必须遵循准确的、约定俗成的规则，才能在接收方正确解释数据。
对于网络通信用例来说，显然需要一种确定的方法来将进程中的数据表示转换成线格式，然后再转换回来。
通信的盒子可能基于不同的微控制器，具有不同的内码和不同的数据字大小（16 位、32 位、64 位），因此采用完全不同的排列方式。
在 AUTOSAR CP 中，序列化通常不用于`平台内部/节点内部`通信！
在这里，内部内存数据表示可以直接从发送方复制到接收方。
这是可能的，因为在典型的 CP 产品中有三个假设：
- 所有本地SWCs的`字节序`相同。
- 所有本地SWCs的某些数据结构的`对齐方式`是一致的。
- 在内存中交换的数据结构是`连续的`。


## 9.2 Service Discovery Implementation Strategies
如前几章所述，ara::com 希望由产品供应商实现服务发现的功能。
由于服务发现功能基本上是在 API 层面上定义的（参见第 6.4 节），其方法包括`FindService, OfferService and StopOfferService`（见第 6.4 节），因此协议和实施细节部分是开放的。
当 AP 节点（更具体地说是`AP SWC`）通过网络`offer`一个service 或`require`另一个网络节点提供服务时，`service discovery/service registry`显然是通过线路进行的。
所使用的通信协议必须完全规定通过网络发现服务的协议。
对于 SOME/IP，SOME/IP 服务发现协议规范[9]对此进行了规定。
但是，如果一个 ara::com 应用程序想与同一厂商 AP 中同一节点上的另一个 ara::com 应用程序通信，就必须有一个可用的本地服务发现变体。
这里唯一的区别是，`本地服务发现的协议实现`完全由接入点产品供应商决定。

一个AP产品供应商可以选择两种方法之间的区别：`集中式方法`和`分布式方法`。
！！！`集中式方法`是指供应商决定拥有一个中央实体（例如一个守护进程），该实体：
- 维护所有`服务实例`及其`位置信息`的`注册表`
- 处理`本地`ara::com应用程序发出的所有`FindService、OfferService和StopOfferService`请求，从而`更新注册表（OfferService、StopOfferService）`或`查询注册表（FindService）`
- 处理`来自网络的`所有SOME/IP SD消息，从而`更新其注册表（收到SOME/IP Offer Service）`或`查询注册表（收到SOME/IP Find Service）`
- 通过`发送SOME/IP SD消息``将本地更新传播到网络`












## 9.3 Multi-Binding implications
As shortly discussed in subsection 6.2.1 Multi-Binding describes the solution to support setups, where the technical transport/connection between different instances of a certain proxy class/skeleton class are different.
service的实例1可以使用ipc，实例2可以使用someip
There might be various technical reasons for that:
* `proxy class` uses different transport/IPC to communicate with different skeleton instances.
  Reason: `Different service instances` support `different transport mechanisms` because of `deployment decisions`.
* symmetrically it may also be the case, that different proxy instances for the same skeleton instance uses different transport/IPC to communicate with this instance: The skeleton instance supports multiple transport mechanisms to get contacted.
- 代理类使用不同的传输/IPC 与不同的骨架实例通信。原因是什么？
  由于部署决定，不同的服务实例支持不同的传输机制。
- 对称情况下，同一skeleton实例的不同proxy实例也可能使用不同的传输/IPC与该实例通信：skeleton实例支持多种传输机制。

### 9.3.1 简单的多重绑定用例
下图描述了一个明显和/或相当简单的案例。
在这个示例中，服务消费者（代理）和服务提供者（骨架）之间的通信只涉及本地节点（在一个 AP 产品/ECU 内），服务消费者侧有两个相同代理类的实例。
从图中可以看到，`服务 消费者 应用程序`首先触发了 “`FindService`”（查找服务），然后返回了两个句柄，分别代表 所搜索服务类型 的 两个 不同服务实例。
服务消费者应用程序 为 每个句柄 实例化了 一个代理实例。
在本例中，服务实例 1 与服务消费者（代理实例 1）位于同一自适应应用程序（同一进程/地址空间）中、
而服务实例 2 位于不同的自适应应用程序中（不同的进程/地址空间）。


```cpp
FindService(ServiceType, AnyInstance)
returns Handle1, Handle2
```

在这幅图中，象征代理和骨架之间传输层的线条颜色不同：
  实例 1 的代理类实例的传输层（绑定实现）为红色，而实例 2 的传输层为蓝色。
  之所以颜色不同，是因为在代理实现的层面上，所使用的技术已经不同。至少，如果你希望 AP 产品供应商（作为 IPC 绑定实现者）努力提供性能良好的产品的话！
在这种情况下，代理实例 1 和服务实例 1（红色）之间的通信应优化为普通方法调用，因为代理实例和骨架实例 1 都包含在一个进程中。
！！！service的provide和require甚至不一定使用IPC手段
代理实例 2 和服务实例 2（蓝色）之间的通信是真正的 IPC。
因此，这里的操作成本要高得多，很可能涉及各种`系统调用`/`内核上下文切换`，以便将 调用/数据 从 服务消费者应用程序的 进程 传输到 服务应用程序（通常使用`管道、套接字或共享 mem`等基本技术，并在上面添加一些信号进行控制）。

因此，从 服务消费方应用程序 开发人员的角度来看，这是完全透明的：
  从供应商的`ProxyClass::FindService`实现中，他获得了两个服务实例的不透明句柄，并由此创建了同一个代理类的两个实例。
  但 “神奇”的是，这两个代理的行为方式完全不同，它们分别与各自的服务实例进行联系。
  因此，这个句柄中必须`包含一些信息`，代理类实例才能从中知道要选择哪种`技术传输方式`。
  虽然这个用例乍一看很简单，但仔细一看却并非如此...... 
  问题是：谁会在句柄中写入这样的信息：由 句柄 创建的 代理实例 应使用直接的方法/函数调用，而不是更复杂的 IPC 机制，反之亦然？
当服务实例 1 在注册表/服务发现处通过`SkeletonClass::OfferService`进行注册时，这一点无法确定！
因为这取决于以后使用它的服务消费者。
因此，AP 供应商的`SkeletonClass::OfferService`实现很可能从参数（AP 供应商生成的skeleton）中获取所需的信息，并通过 AP 供应商特定的 IPC 通知 AP 供应商的注册表/服务发现实现。
前一句中的许多 “AP 厂商 ”都是有意为之。
这只是表明，这里的所有机制都不是标准化的，因此可以由接入点供应商有意设计和优化。

不过，基本步骤仍将保留。
因此，在使用`SkeletonClass::OfferService`的过程中，服务实例方面 通常会 向`注册表/Discovery`通报的是`技术寻址信息`，即如何通过 AP 产品的本地 IPC 实现到达实例。


通常情况下，一个 AP 产品/AP 节点只使用一种 IPC 机制！
如果产品供应商已经在自适应应用程序之间实现了高度优化/高效的本地 IPC，那么这种 IPC 将被普遍使用。
因此--在我们的例子中，假设底层 IPC 机制是`unix domain socket`--skeleton instance 1 将 获取/创建 其套接字端点 所连接的文件描述符，并在`SkeletonClass::OfferService`过程中将此描述符传递给`注册表/服务发现`。
`skeleton instance 2`也是如此，只是描述符不同而已。
当 服务消费者应用部分 稍后执行`ProxyClass::FindService`时，`注册表`会将这两个服务实例的`寻址信息`发送给服务消费者，在服务消费者中，这两个服务实例作为两个不透明句柄可见。

因此，在这个例子中，显然句柄的外观完全相同，但有一点不同，即所包含的`file descriptor`值不同，因为它们引用的是不同的 unix 域套接字。
因此，在这种情况下，必须以某种方式在实例 1 的`proxy`中检测到`直接方法/函数调用`的优化可能性。
一个可能的小窍门是，在skeleton instance 1 提供给注册/发现系统的`寻址信息`中，也包含`进程 ID（pid）`，可以是明示的，也可以是将其包含在套接字描述符文件名中的。
因此，服务消费方代理实例 1 只需检查句柄内的 PID 是否与自己的进程相同，然后就可以使用优化路径。
顺便说一下： 检测进程的本地优化潜力是一件微不足道的小事，如今几乎所有现有的中间件实现都能做到这一点，因此无需再强调这个话题。
现在，如果我们后退一步，就必须认识到，我们这里的简单示例并不能完全反映多重绑定的含义。
它确实描述了 `同一代理类的 两个实例` 使用不同传输层联系服务实例的情况，但正如示例所示，这并没有反映在 表示不同实例的 句柄中，而只是一种优化！
在我们的具体例子中，使用`proxy instance 1` 从而与服务实例 1 通信的`服务消费者`也可以像`proxy instance 2` 一样使用 Unix 域套接字传输，而不会有任何功能上的损失，只是从非功能性能的角度来看，这显然是不好的。
尽管如此，这个简单的场景还是值得在此提及因为它是现实世界中的一种情况，在许多部署中都很可能发生，因此必须得到很好的支持！




### 9.3.2 本地/网络多重绑定用例
在上一节中，我们了解了多重绑定的一个特殊变体、现在我们来看一个变体，它也可以被视为现实世界中的一个案例。
假设我们有一个与前一章非常相似的设置。
唯一不同的是，服务实例 2 位于不同的与带有 AP 产品的 ECU 连接在同一个以太网网络上，其中服务消费者（及其实例 1 和实例 2 的代理）所在的以太网网络。
由于 AP 以太网的标准协议是 SOME/IP，因此两个 ECU 之间的通信也应基于 SOME/IP。
在我们的具体示例中，这意味着代理 1 通过 unix 域套接字与服务 1 通信（如果 AP 供应商/IPC 实现者做足了功课，可能会对进程本地通信进行优化，以直接调用方法），而代理 2 则通过符合 SOME/IP 消息格式的网络套接字与服务 2 通信。

因此，在这种情况下，我们 AP ECU 上的注册表/服务发现daemon看到了实例 2 的服务提议，该提议包含基于 IP 网络端点的寻址信息。
关于实例 1 的服务提议，没有任何变化
该提议仍与某个 Unix 域名套接字名称相关联，该名称实质上是一个文件名。
本质上是一个文件名。在这个例子中，从 ProxyClass::FindService 内部返回的实例 1 和实例 2 的两个句柄看起来非常不同：实例 1 的句柄包含 Unix 域套接字和名称信息，而句柄 2 则包含网络套接字、IP 地址和端口号信息。因此，与第一个例子（第 9.3.1 小节）不同的是，在这里我们确实有一个全面的多重绑定，我们的代理类 ctor 从句柄 1 中实例化/创建了
在这里，我们的代理类 ctor 从句柄 1 和句柄 2 中实例化/创建了两种完全不同的传输机制！如何在调用 ctor 时动态决定使用哪种传输机制，在技术上是由中间件实现者决定的： 生成的代理类实现可能已经包含了任何支持的机制，而句柄中包含的信息只是用来在不同行为之间进行切换，或者在运行时通过共享库机制从给定的句柄中检测到某种需要后，加载所需的传输功能（又称绑定）。



### 9.3.3 Typical SOME/IP Multi-Binding use case
在上一节中，我们简要提到，在使用 SOME/IP 作为网络协议的典型部署场景中，自适应 SWC（即在其上下文中运行的语言和网络绑定）自己打开套接字连接与远程服务通信的可能性很小。
为什么不太可能？因为 SOME/IP 在设计之初就明确要求使用尽可能少的端口。之所以有这样的要求，是因为嵌入式 ECU 功耗低、资源少： 并行管理大量 IP 套接字意味着内存（和运行时）资源方面的巨大成本。
因此，作为车内网络主要通信伙伴的 AUTOSAR CP 兄弟姐妹需要采用这种方法，而这种方法与非汽车行业的端口使用模式相比并不常见。

通常情况下，这种要求会导致一种架构，即 ECU/网络端点的`所有 SOME/IP 流量`都通过`一个 IP 端口`路由！
这就意味着来自/发送到许多不同本地应用程序（服务提供商或服务消费者）的 SOME/IP 报文被（去）复用到/从一个套接字连接。
在经典 AUTOSAR (CP) 中，这是一个简单明了的概念，因为已经有一个共享的通信栈，整个通信都通过该通信栈进行。
`通过一个套接字复用不同的上层 PDU 是集成在 CP SoAd 基本软件模块中的核心功能。`
对于具有 POSIX 套接字应用程序接口的典型 POSIX 兼容操作系统来说，将许多应用程序的 SOME/IP 通信多路复用到一个端口或从一个端口多路复用到另一个端口，意味着需要引入一个单独的/中央（daemon）进程来管理相应的端口。该进程的任务是在 SOME/IP 网络通信和本地通信之间架起桥梁，反之亦然。

从上图中可以看到，启用了 ara::com 的应用程序中的服务代理通过（绿线）SOME/IP 网桥与远程服务实例 2 通信。图中有两点值得注意：
- 我们有意将 APP 到 网桥 的通信线路部分（绿色）与 网桥 到服务实例 2 的通信线路部分（蓝色）用不同的颜色表示。
- 我们有意在 功能块服务发现 和 SOME/IP网桥 周围画了一个框。

将路线的第一部分与第二部分涂上不同颜色的原因很简单：
  这两部分都使用了不同的传输机制。
  代理和网桥之间的第一部分（绿色）使用的是完全由供应商决定的实现方式(IPC)，而第二部分（蓝色）则必须符合 SOME/IP 规范。
  这里的 “完全针对特定供应商 ”是指，供应商不仅要决定使用哪种技术（管道、套接字、shm......），还决定在该路径上采用哪种序列化格式（见第 9.1 节）。
在这里，我们显然进入了优化的范畴：
  在优化的 AP 产品中，供应商不会对绿线表示的路径采用不同的（专有）的序列化格式。否则会导致运行时的低效行为。
  首先，服务消费者应用程序中的代理会采用专有的序列化数据，然后再传输到网桥节点。
必须将数据`反序列化`并`重新序列化`为 `SOME/IP` 序列化格式！
  因此，即使 接入点产品供应商 为 本地通信 提供了 更有效/更精细的序列化方法，在本地通信中使用它也无济于事，因为这样一来，网桥就不能在内部和外部之间简单复制数据。
  结果是在我们的示例方案中，我们最终还是采用了多绑定设置。
因此，即使用于与其他本地 ara::com 应用程序和桥接节点通信的技术传输（管道、unix 域套接字、共享 mem......）是相同的，绑定的序列化部分也是不同的。
关于图中第二个值得注意的地方：
  我们在服务发现和 SOME/IP 桥接功能周围画了一个方框，因为在产品实现中，这很有可能被集成到一个组件中/在一个（daemon）进程中运行。
  这两种功能高度相关：`发现/注册部分`还包括 ECU 的本地部分（接收本地注册/offer和提供本地查找服务请求）和网络相关功能（基于 SOME/IP 服务发现的offer/find），其中注册处必须进行`仲裁`。
  这种仲裁的核心也是桥接功能。







### 9.4.2 Software Component
通过软件组件，AUTOSAR方法论定义了比接口更高阶的元素。
`软件组件`的概念是描述`软件的可重用部分`，并具有`明确定义的接口`。
为此，AUTOSAR 清单规范定义了一个模型元素 `SoftwareComponentType`，它是一个`抽象`元素，具有多个具体子类型，其中 `AdaptiveApplicationSwComponentType` 子类型对于自适应应用软件开发人员来说最为重要。SoftwareComponentType 模型元素由 C++ 代码实现。
该组件 “向 ”外部 “提供 ”或 “要求 ”外部提供哪些服务接口由端口表示。
端口由 ServiceInterfaces 类型化。
P 端口表示端口类型的服务接口已被提供，而 R 端口表示软件组件类型需要端口类型的服务接口。


对于上半部分（元模型层）示例中 不同的软件组件类型 A 和 B，在`(代码)实现层`（图中的下半部分）都有具体的实现。
软件组件类型 A 的 R 端口的实现基于 ara::com 代理类在实现层的实例，而软件组件类型 B 的 P 端口的实现则使用 ara::com 骨架类的实例。
代理类和骨架类都是从服务接口定义 ServiceInterface 生成的，相应的端口都引用了服务接口定义 ServiceInterface。
本例中使用的是服务接口 “RadarService”，我们已在整个文档中使用过。这样一个实现软件组件类型的代码片段显然可以重复使用。
在 C++ 实现层面上，`一个 AdaptiveApplicationSwComponentType 的实现` 通常可以归结为`一个或几个 C++ 类`。

因此，重用仅仅意味着在不同的上下文中多次实例化这个类/那些类。在这里，我们基本上可以区分以下几种情况：
* 在代码中多次实例化 C++ 类。
* 通过多次启动/运行同一可执行文件进行隐式多次实例化。
第一种情况仍属于 “实现级 ”领域。

上图展示了一个任意示例，其中 A 和 B 的实现在不同的上下文中实例化。
* 左下方是可执行文件 1，它直接使用了两个 As impl 实例和一个 Bs impl 实例。
* 与此相反，右侧显示的是可执行文件 2，它 “直接”（即在最顶层）使用了一个 Bs 植入实例和一个复合软件组件实例，而该组件本身 “在其主体中 ”又实例化了一个 As 和 Bs 植入实例。
注：`AUTOSAR 元模型中以 CompositionSwComponentType 的形式充分反映了将软件组件与其他组件组合成更大/复合人工制品的自然实现概念`，CompositionSwComponentType 本身就是软件组件类型，允许软件组件的任意递归嵌套/组合。
第二种情况则属于`“部署级别 ”`的范畴，将在下一章中加以说明。


### 9.4.3 Adaptive Application/Executables and Processes
AP中的`可部署软件单元`被称为`自适应应用程序`（相应的元模型元素为 AdaptiveAutosarApplication）。
这种`自适应应用程序`由 `1...n` 个可执行文件组成，而这些可执行文件又是通过`实例化 CompositionSwComponentType`（具有任意嵌套）建立起来的，如前一章所述。
**通常情况下，集成者会决定`以1...n个可执行文件的形式`启动哪些自适应应用程序，以及`启动某个自适应应用程序/其相关可执行文件的次数。`**
这就意味着，这种隐式实例化无需编写特定代码！
集成商只需处理机器配置，配置应用程序的启动次数。
启动后的自适应应用程序会变成 1...n 个进程（取决于可执行文件的数量）。
我们称之为 “部署级别”。

上图显示了一个简单的示例，我们有两个自适应应用程序，其中每个应用程序都包含一个可执行文件。
包含可执行文件 1 的自适应应用程序 1 部署了两次，在可执行文件启动后会产生进程 1 和进程 2，而包含可执行文件 2 的应用程序 2 部署了一次，在启动后会产生进程 3。




### 9.4.4 Usage of meta-model identifiers within ara::com based application code
到此为止对元模型/ara::com 关系的解释应有助于理解 6.4 中描述的 ResolveInstanceID 中使用的实例标注符的结构。
如前一章所述和图 9.6 所示，`实例说明符`以某种方式与`软件组件类型模型中的相应端口`相关联。
**在代码中，同一个组件的实现可能会被实例化多次，并最终在不同进程中启动多次。**
`显然，必须为对象分配实例 ID，使其最终在部署中具有独特的身份`。

上图以一个简单的例子概述了“问题”。
在可执行文件 2 中，`SoftwareComponentType B`在不同的上下文（嵌套层）中有三个实例。
所有实例都提供了 SI RadarService 的特定实例。
应用流程 2 服务实例清单的集成商必须在 ara::com 层面上进行技术映射。
也就是说，他必须决定在每个 B 实例中使用哪种技术传输绑定，然后再决定使用哪种技术传输绑定的特定实例 ID。
在我们的示例中，集成商希望通过 SOME/IP 绑定提供 SI RadarService，并在 B 实例（嵌套在右侧的复合组件内）中使用 SOME/IP 特定实例 ID “1”、 而他决定通过本地 IPC（Unix 域套接字）绑定和 Unix 域套接字特定实例 ID“/tmp/Radar/3 ”和“/tmp/Radar/4 ”来提供 SI RadarService。
很明显，在`Service Instance Manifest`（允许将进程中的端口实例映射到技术绑定及其具体实例 ID）中，仅使用模型中的`端口名称`是不足以区分的。
要在一个可执行文件（因此也是一个流程）中获得唯一的标识符，嵌套实例化和再实例化的特性是必不可少的。

每次软件组件类型实例化时，它都会在其实例化上下文中获得一个唯一的名称。
这一概念同时适用于 C++ 实现级和 AUTOSAR 元模型级！在我们的具体示例中，这意味着：
* 顶层的 B 实例在其层级上有唯一的名称： “B_Inst_1”和 “B_Inst_2”
* 复合组件类型中的 B 实例在这一层获得唯一名称： “B_Inst_1”
* 位于顶层的复合组件实例在其层级上获得唯一名称： “Comp_Inst_1”

* 因此，从可执行文件/进程的角度来看，B 的所有实例都有唯一的标识符：
- “B_Inst_1”
- “B_Inst_2”
- “Comp_Inst_1::B_Inst_1”

对于自适应软件组件开发人员来说，这就是一言以蔽之的意思：
  如果您构建了一个实例说明符，通过 `ResolveInstanceIDs()` 转换为 `ara::com::InstanceIdentifier` 或直接与 `FindService()` 一起使用（从模型角度看是 R 端口侧），或作为骨架的 ctor 参数（从模型角度看是 P 端口侧），那么它应该是这样的：
```cpp
<context identifier> / <port name>
```
端口名称取自模型，该模型描述了要开发的AdaptiveApplicationSwComponentType。
由于您不一定是决定组件部署位置和频率的人，因此您应该预见到，您的 AdaptiveApplicationSwComponentType 实现可以通过一个字符串化的`<context identifier>`，您可以在构建组件时直接使用该标识符：
- 您可以在构造 ara::core::InstanceSpecifier 时直接使用该字符串，以实例化代理/骨架，以反映您自己的组件端口。
- 将其 “移交 ”给其他 AdaptiveApplicationSwComponentType 实现。
类型实现（即从自己的 AdaptiveApplicationSwComponentType 实现实例化（即创建一个新的嵌套层）

```cpp
/*! The instance specifier to be used for Service. */
ara::core::StringView const kInstanceSpecifierService{
    "data_collector/RootSwComponentPrototype/someip_recorder_service_required_port"};
```











