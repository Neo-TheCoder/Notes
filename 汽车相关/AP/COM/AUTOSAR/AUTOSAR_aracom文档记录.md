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
* list with one element:
  The common case. Mapping to one concrete instance id of one concrete technical binding.
* list with more than one element: 
  Mapping to multiple technical instances with possibly multiple technical bindings.


从技术上讲，`ResolveInstanceIDs()`的中间件实现是从进程中捆绑的`service instance manifest`中查找`ara::core::InstanceSpecifier`。
因此，在捆绑的`service instance manifest`中，`ara::core::InstanceSpecifier`必须是明确的。























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
因此，作为车内网络主要通信伙伴的 AUTOSAR CP 兄弟姐妹需要采用这种方法，而这种方法与非汽车 IT 的端口使用模式相比并不常见。


通常情况下，这种要求会导致一种架构，即 ECU/网络端点的所有 SOME/IP 流量都通过一个 IP 端口路由！
这就意味着来自/发送到许多不同本地应用程序（服务提供商或服务消费者）的 SOME/IP 报文被（去）复用到/从一个套接字连接。
在经典 AUTOSAR (CP) 中，这是一个简单明了的概念，因为已经有一个共享的通信栈，整个通信都通过该通信栈进行。
`通过一个套接字复用不同的上层 PDU 是集成在 CP SoAd 基本软件模块中的核心功能。`
对于具有 POSIX 套接字应用程序接口的典型 POSIX 兼容操作系统来说，将许多应用程序的 SOME/IP 通信多路复用到一个端口或从一个端口多路复用到另一个端口，意味着需要引入一个单独的/中央（daemon）进程来管理相应的端口。该进程的任务是在 SOME/IP 网络通信和本地通信之间架起桥梁，反之亦然。

从上图中可以看到，启用了 ara::com 的应用程序中的服务代理通过（绿线）SOME/IP 网桥与远程服务实例 2 通信。图中有两点值得注意：
- 我们有意将应用程序到网桥的通信线路部分（绿色）与网桥到服务实例 2 的通信线路部分（蓝色）用不同的颜色表示。
- 我们有意在功能块服务发现和 SOME/IP 网桥周围画了一个框。

将路线的第一部分与第二部分涂上不同颜色的原因很简单：
  这两部分都使用了不同的传输机制。
  代理和网桥之间的第一部分（绿色）使用的是完全由供应商决定的实现方式，而第二部分（蓝色）则必须符合 SOME/IP 规范。这里的 “完全针对特定供应商 ”是指，供应商不仅要决定使用哪种技术（管道、套接字、shm......），还决定在该路径上采用哪种序列化格式（见第 9.1 节）。
在这里，我们显然进入了优化领域： 在优化的 AP 产品中，供应商不会对绿线表示的路径采用不同的（专有）的序列化格式。否则会导致运行时的低效行为。首先，服务消费者应用程序中的代理会采用专有的序列化数据，然后再传输到网桥节点。
必须将数据去序列化并重新序列化为 SOME/IP 序列化格式！因此，即使即使接入点产品供应商为本地通信提供了更有效/更精细的序列化方法在本地通信中使用它也无济于事，因为这样一来，网桥就不能在内部和外部之间简单复制数据。结果是在我们的示例方案中，我们最终还是采用了多绑定设置。
因此，即使用于与其他本地 ara::com 应用程序和桥接节点通信的技术传输（管道、unix 域套接字、共享 mem......）是相同的，绑定的序列化部分也是不同的。
关于图中第二个值得注意的地方： 我们在服务发现和 SOME/IP 桥接功能周围画了一个方框，因为在产品实现中，这很有可能被集成到一个组件中/在一个（恶魔）进程中运行。这两种功能高度相关： 发现/注册部分还包括 ECU 的本地部分（接收本地注册/报价和提供本地查找服务请求）和网络相关功能（基于 SOME/IP 服务发现的报价/查找），其中注册处必须进行仲裁。这种仲裁的核心也是桥接功能。



































