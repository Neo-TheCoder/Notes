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


## 6.1 Instance Identifiers
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





























