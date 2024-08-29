# 2 Introduction
Why did AUTOSAR invent yet another communication middleware API/technology, while there are dozens on the market — the more so as one of the guidelines of Adaptive Platform was to reuse existing and field proven technology?

Before coming up with a new middleware design, we did evaluate existing technologies, which — at first glance — seemed to be valid candidates. Among those were:
* ROS API
* DDS API
* CommonAPI (GENIVI)
* DADDY API (Bosch)

The final decision to come up with a new and AUTOSAR-specific Communication Management API was made due to the fact, that not all of `our key requirements` were met by existing solutions:
* **We need a `Communication Management`, which is NOT bound to a concrete network communication protocol.**
    It has to support `the SOME/IP protocol` but there has to be flexibility to exchange that.
* The AUTOSAR service model, which defines services as a collection of provided methods, events and fields shall be supported naturally/straight forward.
    也就是要支持AUTOSAR模型
* The API shall support an `event-driven` and a `polling` model to get access to communicated data equally well. The latter one is typically needed by real-time ap-
plications to avoid unnecessary context switches, while the former one is much more convenient for applications without real-time requirements.
**应用程序接口应支持 “事件驱动 ”和 “轮询 ”模式，以同样好地访问通信数据。`实时应用程序`通常需要后一种模式，以避免不必要的上下文切换，而前一种模式对于没有实时要求的应用程序来说要方便得多。**

* Possibility for seamless integration of end-to-end protection to fulfill ASIL requirements.
可实现端到端保护的无缝集成，以满足`ASIL`要求。
* Support for static (preconfigured) and dynamic (runtime) selection of service instances to communicate with.

So in the final ara::com API specification, the reader will find concepts (which we will describe in-depth in the upcoming chapters), which might be familiar for him from technologies, we have evaluated or even from the existing Classic Platform:
* Proxy (or Stub) / Skeleton approach (CORBA, Ice, CommonAPI, Java RMI, ...)
* Protocol independent API (CommonAPI, Java RMI)
* Queued communication with `configurable receiver-side caches` (DDS, DADDY, Classic Platform)
* Zero-copy capable API with possibility to shift `memory management` to the middleware (DADDY)
* Data reception filtering (DDS, DADDY)

Now that we have established the introduction of a new middleware API, we go into the details of the API in the following chapters.
The following statement is the basis for basically all `AUTOSAR AP specifications`, but should be explicitly pointed out here again:
**ara::com only defines `the API signatures` and `its behavior visible to the application developer`. Providing an implementation of those APIs and the underlying middleware transport layer is the responsibility of the AUTOSAR AP vendor.**
**ara::com仅定义了 “API签名 ”和 “应用程序开发人员可见的行为”。提供这些 API 和底层中间件传输层的实现是 AUTOSAR AP 供应商的责任。**
For a rough parallel with the AUTOSAR Classic Platform, ara::com can be seen as fulfilling functional requirements in the Adaptive Platform similar to those covered in the Classic Platform by the RTE APIs [1] such as Rte_Write, Rte_Read, Rte_Send, Rte_Receive, Rte_Call, Rte_Result.


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

This allows for support of `strictly real-time scheduled applications`, where the application requires total control of what (amount) is done when and where unnecessary context switches are most critical.
**On the other hand the more relaxed event based applications, which simply want to get notified whenever the communication layer has data available for them is also fully supported.**
The decision within AUTOSAR to genuinely support C++11/C++14 for AP was a very good fit for the ara::com API design.
For enhanced usability, comfort and a breeze of elegance ara::com API exploits C++ features like smart pointers, template functions and classes, proven concepts for asynchronous operations and reasonable operator overloading.


# 5 High Level API Structure
## 5.1 Proxy/Skeleton Architecture
If you’ve ever had contact with middleware technology from a programmer’s perspective, then the approach of a Proxy/Skeleton architecture might be well known to you.
Looking at the number of middleware technologies using the Proxy/Skeleton (sometimes even called Stub/Skeleton) paradigm, it is reasonable to call it the "classic approach".
So with ara::com we also decided to use this classical Proxy/Skeleton architectural pattern and also name it accordingly.

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


示例服务 RadarService 提供了一个`evnet` “BrakeEvent”，它由一个结构组成，其中包含一个标志和一个长度可变的 uint8 数组（作为额外的有效载荷）。
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

为什么这个 API 会返回一个 `InstanceIdentifierContainer`（它代表了ara::com::InstanceIdentifierContainer 的集合）？
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
代码示例：
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
这就意味着，直接使用 ara::com::InstanceIdentifier 和手动解析 ara::core::InstanceSpecifier 更多是针对那些使用案例比较特殊的高级用户。在讨论代理/骨架侧相应的 ara::com API 重载的章节中将给出一些示例。

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
- `StartFindService`是一个类方法，它会`在后台启动一个持续的 “FindService”活动，在服务实例的可用性 发生变化时 通过 给定的回调通知调用者`。
- `FindService`是一次性调用，它会返回调用时间点的可用实例。

根据所采用的`instance identifier`方法（参见第 6.1 节），这两种方法有三种不同的重载：
- 一个使用 `ara::com::InstanceIdentifier`
- 一个使用 `ara::core::InstanceSpecifier`
- 一种不使用参数。

无参数变量的语义很简单： 查找给定类型的`所有服务`，无论其`绑定方式`和`绑定的具体实例标识符`如何。
请注意，只有技术绑定才会被用于查找/搜索，这些技术绑定是以 ServiceInterfaceDeployment 的形式在服务实例清单中为相应服务接口配置的。
请注意，只有技术绑定才能用于查找/搜索，这些技术绑定是以服务接口部署的形式在服务实例清单中为相应服务接口配置的。
同步一次性变体 FindService 会返回匹配服务实例的句柄容器（见第 6.2.1 小节），如果当前没有匹配的服务实例，则句柄容器也可能为空。

与此相反，`StartFindService`会返回一个`FindServiceHandle`，可以通过调用`StopFindService`来停止正在进行的监控服务实例可用性的后台活动（？？？`StartFindService`到底做什么？）。
StartFindService 的第一个参数（也是该变体的特定参数）是用户提供的处理函数，其签名如下：
```cpp
using FindServiceHandler = std::function<void(ServiceHandleContainer<T>, FindServiceHandle)>;
```
一旦绑定检测到与调用 StartFindService 时给定的实例条件相匹配的服务实例的可用性发生了变化，它就会调用用户提供的处理程序，并提供最新的可用服务实例句柄列表。
调用 StartFindService 后，StartFindService 的行为与 FindService 类似，它将使用当前可用的服务实例（也可能是空句柄列表）触发用户提供的处理函数。初始回调触发后，如果初始服务可用性发生变化，它将再次调用提供的处理函数。
请注意，ara::com 用户/开发者在用户提供的处理程序中调用 StopFindService 是明确允许的。
为此，处理程序会明确获取 FindServiceHandle 参数。处理程序不需要重入。
这意味着绑定实现者必须注意对用户提供的处理程序函数的调用进行按顺序排列。

请注意，`ServiceHandleContainer`可以作为分配容器或非分配容器来实现，在用作 FindService 的返回值或 FindServiceHandler 的参数时，只要它满足 C++ 编程语言对容器的一般要求和顺序要求即可。



### 6.2.2.1 Auto Update Proxy instance
无论使用一次性 FindService 还是 StartFindService变体，在这两种情况下，你都能`获得 一个标识服务实例（可能是远程服务实例）的句柄`，然后再从中`创建代理实例`。
但是，如果服务实例宕机了，但后来又恢复了，例如，由于生命周期状态发生了变化，会发生什么情况呢？当服务实例再次可用时，服务消费方现有的代理实例还能重新使用吗？
好消息是 ara::com 设计团队决定从绑定实现中要求这种重用可能性，因为它能减轻实现服务消费者的典型任务。
`在基于服务的通信领域，预计在整个系统（如车辆）的生命周期内，服务提供者和消费者实例会由于其自身的生命周期概念而频繁地启动和关闭。`
`服务提供商和消费者的生命周期在服务提供和服务（再）订阅方面受到监控。`

为此，我们建立了`服务发现基础架构`，从`service offerings`和`service (re)subscriptions`的角度监控 服务提供商 和 消费者 的生命周期！
如果服务消费者应用程序从 “查找服务 ”变体返回的句柄实例化了一个服务代理实例，可能发生的顺序如下图所示。




















# 9 Appendix


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













