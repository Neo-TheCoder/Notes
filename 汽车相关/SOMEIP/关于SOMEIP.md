# 汽车软件通信中间件SOME/IP简述
## 1 SOME/IP是中间件吗
准确来说是通信协议，简单来说也可以呃称为通信中间件
全称是**Scalable service-Oriented MiddlewarE over IP**
即：基于IP协议的面向服务的可扩展性通信中间件协议
可以看出有三个要素：
1. 面向服务SOA
2. 基于IP协议之上的通信协议
3. 中间件

## 2 SOME/IP的作用
都和通信相关：
1. 服务发现（Service Discovery）
2. 远程服务调用（RPC，Remote Producer Call）
3. 读写进程信息（Getter & Setter）

## 3 SOME/IP和CAN的区别
CAN是传统的汽车软件通信协议，CAN FD是其扩展
二者的主要区别在于：
1. 通信负荷的差异：即承载的字节数，8Byte -- 1500Byte
2. 通信速率的差异   1Mb/s -- 1000Mb/s
3. CAN是面向服务的，SOME/IP是面向服务的
4. CAN是基于信号在双绞线中传输信号  --  SOME/IP是在车载以太网中传输信号
SOME/IP实际上是位于车载以太网网络协议中的应用层，是基于TCP/UDP协议之上的应用
GENIVI组织针对SOME/IP标准实现了开源vsomeip方案
SOME/IP 协议一般指代具体 SOME/IP、SOME/IP-SD、SOME/IP-TP 3种。

5. 消息格式
一个完整的 SOME/IP 消息，包含以下内容：
MessageID（Event ID / Method ID） -- 32bit
Length -- 32bit -- 消息长度
Request ID（ Client ID & Session ID）-- 32bit
Protocal Version -- 协议版本号
Interface Version -- 接口版本号
Message Type -- 消息类型
Return Code -- 返回编码
Payload -- 数据负载（可变）：SOME/IP底层可以基于TCP或者UDP，使得Payload的容量不一样：如果是UDP，则payload限制在1400B的容量，如果是TCP，数据分段传输，那么SOME/IP可以实现更大容量的传输
PS：所有SOME/IP Header内容采用大端传输，而Payload中的数据存放顺序通过配置设置

6. SOME/IP支持的数据结构类型
基础的uint、int...，对于结构体数据会在内存中按照顺序依次存放

7. SOME/IP消息通信类型
1. R & R（Request & Response）
2. F & F (Fire & Forget)：只请求不应答
3. Notification Event-- 发布订阅机制
但 Notification 分3种情况：
Cyclic Update 周期性的发送相关 value 的变化
Update On Change 如果 value 发生变化，则向外推送
Epsilon Change 如果 value 的值大于相应的 epsilon值，那么对外推送消息
5. Fields
针对的是一种status，并且有一个合法值
一个Fields操作应当包含Setter、Getter与Notification的结合。
执行Setter：客户端发起请求，并在请求消息中的payload设置确定值，并在服务端应答消息中的payload要同步返回这个值
执行Getter：客户端发起请求，请求消息中包含一个空的payload，服务端收到请求消息后将field的值填充到应答消息中的payload

8. SOME/IP-SD
服务发现主要功能：
探测服务、向外提供服务
SD的消息结构是在SOME/IP消息体之上附加了一些参数：比如Entries Array，用于标记Service或者Event Group

服务发现的流程
谈论 Service Discovery 之前，要搞清楚几个基本概念
**Service**：指代 functional entities，代表的是一项功能.
**find**：是一项操作，用于标识 available 状态的 service 及它的们的位置
**event**：Service Discovery 传送的 Message 被称为 event
**Eventgroups**：event 的集合

service中有许多细节。
Service有2类：Server Service和Client Service。 Service有2种状态: Down和Available。
一个ECU需要处理两种Service:ServerService和ClientService。
也就是说，一个 ECU 可以对外提供服务,这个时候由它的 Server Service 模块负责，也可以对外请求服务，这个时候由 Client Service 提供。
实际的流程非常复杂，一个 Service 从 Down 状态到 Available 状态之间的切换有一系列的状态阶段转换。

# SOME/IP和DDS的差异
DDS是面向数据的，通信容量在理论上是无限，并且还使用了共享内存，QOS丰富。




# 实例分析
以以下**service**为例
```json
 "services" : [
    {
      "name" : "SomeipStartApplicationCmService1_ServiceInterface",
      "service_id" : 1667,
      "major_version" : 1,
      "minor_version" : 0,
      "methods" : [
        {
          "name" : "StartApplicationMethod1",
          "id" : 5,
          "proto" : "tcp"
        },
        {
          "name" : "StartApplicationField1Getter",
          "id" : 2,
          "proto" : "tcp"
        },
        {
          "name" : "StartApplicationField1Setter",
          "id" : 4,
          "proto" : "tcp"
        }
      ],
      "events" : [
        {
          "name" : "StartApplicationEvent1",
          "id" : 32769,
          "field" : false,
          "proto" : "tcp"
        },
        {
          "name" : "StartApplicationField1Notifier",
          "id" : 32770,
          "field" : true,
          "proto" : "tcp"
        }
      ],
      "eventgroups" : [
        {
          "id" : 1,
          "events" : [
            32769,
            32770
          ]
        }
      ]
    }
 ]
```
## 服务发现
**广播**
### OfferService
`Entries Array`属于`Offer Service`，包含Service ID、Instance ID

### FindService
`Entries Array`属于`Find Service`，包含Service ID、Instance ID

一条**广播报文**，可以在`Entries Array`中，ARRAY1表示`FindService`，ARRAY2表示`OfferService`
Options Array中，包含Ipv4 Endpoint Option：`IP地址 + 端口号 + 协议UDP`



**单播**
### Subscribe
`Entries Array`属于`Subscribe Eventgroup Entry`
指定要订阅的`service id - instance id - Eventgroup - version`

### SubscribeAck
`Entries Array`属于`Subscribe Eventgroup Ack Entry`

一条**广播报文**，可以在`Entries Array`中，ARRAY1表示`Subscribe`，ARRAY2表示`SubscribeAck`

## 通信
### Event
server发送`Message Type`为`Notification`类型的报文

### Method
（不需要订阅，是**请求-应答模式**）
client发送Type`Message Type`为`Request`类型的报文
server发送Type`Message Type`为`Response`类型的报文



# 关于`Max Nr Options Per Msg`
This parameter specifies the amount of attached options, which can be handled per incomming message.

## 相关代码
```cpp
struct OfferServiceEntry {
  /*!
   * \brief Entry identifier
   */
  entries::ServiceEntryId entry_id{};

  /*!
   * \brief Time to live
   */
  amsr::someip_protocol::internal::Ttl ttl{};

  /*!
   * \brief The unicast address of the server, method requests shall be sent to this address if configured to be sent
   * via UDP
   */
  ara::core::Optional<options::IpEndpointOption> udp_endpoint;

  /*!
   * \brief The unicast address of the server, method requests shall be sent to this address if configured to be sent
   * via TCP
   */
  ara::core::Optional<options::IpEndpointOption> tcp_endpoint;
};
```

看起来比较相关的调用栈
```cpp
  entries::OfferServiceEntryContainer imminent_message_;
```

```cpp
/*!
 * \internal
 * Create the SD payload container from the offer service entries.
 * For all payloads:
 *   Create SD message from payload.
 *   Fill in the header of the SD message for multicast sending.
 *   Add the SD message to the messages container.
 * Return the SD message container.
 * \endinternal
 */
ServiceDiscoveryMessageContainer ServiceDiscoveryMessageBuilder::BuildMessage(
    entries::OfferServiceEntryContainer const& offer_service_entries) {
  ServiceDiscoveryMessageContainer sd_message_container{};

  ServiceDiscoveryMessagePayloadContainer payload_container{
      ServiceDiscoveryPayloadBuilder::BuildPayload(offer_service_entries)};

  for (ServiceDiscoveryMessagePayload& payload : payload_container) {
    ServiceDiscoveryMessage sd_message{CreateSdMessageFromPayload(payload)};
    FillMulticastHeader(sd_message);

    sd_message_container.push_back(std::move(sd_message));
  }

  return sd_message_container;
}
```

```cpp
可能是定时器触发
ServiceDiscoveryMessageContainer const sd_message_container{sd_message_builder_.BuildMessage(imminent_message_)}; // 得到sd_message_container

// 多处调用该函数：
SendMulticastMessage(sd_message);

PrepareSdPacket(sd_message);

SerializeSomeIpSdMessage(writer, sd_message);

SerializeSomeIpSdOptions(writer, sd_message.options_);

GetSomeIpSdOptionsSize(options); // 计算ServiceDiscoveryOptionContainer的长度



```


```cpp
/*!
 * \internal
 * If timer is stopped or next expiry is more than half a period away:
 *   Add the offer service entry to the imminent messages container to be scheduled in the next send operation.
 * Else:
 *   Add the offer service entry to the deferred messages container to be scheduled for a later send operation.
 * Start the timer if it is stopped.
 * \endinternal
 */
void CyclicOfferTimer::ScheduleOfferServiceEntry(entries::OfferServiceEntry const &offer_entry) {
  bool const is_stopped{imminent_message_.empty() && deferred_message_.empty()};

  amsr::timer::Timer::Clock::time_point const time_to_half_of_cycle{amsr::timer::Timer::Clock::now() + half_period_ns_};

  if (is_stopped || (Timer::GetNextExpiry() > time_to_half_of_cycle)) {
    imminent_message_.push_back(offer_entry);
  } else {
    deferred_message_.push_back(offer_entry);
  }

  if (is_stopped) {
    Timer::Start();
  }
}
```



## 猜测
```cpp
  OptionInterpreter interpreter{};
  ara::core::Result<void> const interpretation_result{options::OptionsInterpreter::InterpretOptions(
      entry.index_1st_opts_, entry.number_1st_opts_, entry.index_2nd_opts_, entry.number_2nd_opts_, options,
      interpreter)};
```



# 补充
`Index 1st options`：Option1排在Array里第几个（Entries Array的Index1字段，指示当前entries array的选项参数由Options Array的第几个来补充）
`Index 2nd options`：Option2排在Array里第几个
`of opt 1`：Option1的数目
`of opt 2`：Option2的数目
`Service ID`：Entry关于哪个服务
`Instance ID`：Entry关于服务的哪个实例，0xFFFF表示全部实例
`Major Version`：服务的主版本号
`TTL`：“入口”的生命周期，单位为秒，我理解为发现服务时的搜索时间，提供服务时的有效时间
`Minor Version`：服务的次版本号

服务发现，说白了，就是想办法让服务消费者能够找到服务提供者。
打个比方，想象你在一个有很多人的广场上找一个会唱歌的人，很显然有两种情况：1. 你认识这个人，提前说好了，他站在某个地方等你，而你知道那个地方的位置，那你肯定很容易就找到他了，这就是静态配置；2. 你并不认识这个人，存在一个中间人，你告诉中间人你想找一个会唱歌的，而那个人也会告诉中间人我是会唱歌的，我站在广场的哪个位置，然后中间人把位置给你，你就可以找到他了，这就是动态发现，而SOME/IP-SD就是那个中间人。


`Entries Array`，Entry可以理解为“入口”，包含了服务实例以及需要订阅的事件组的信息，分为Service和Eventgroup两种类型，一个SD报文可能包含多个Entry，每个Entry大小都是16个字节，一个Entry可能包含`0-2个Option`。下图为一个完整的SD报文示例：

`Options Array`，Option可以理解为选项参数，Type=0x01时，用于传输Entry的附加信息，比如服务名等等：Type=0x04时，用于传输IPv4相关的参数，比如服务的IP地址、TCP还是UDP、端口号：

# 一个UDP消息里`length of entries array` / `length of options array`是如何得出的？
是someip自适应调整的吗？
```cpp
/*!
 * \brief Alias for SD options length field.
 * \trace SPEC-4981517
 */
using SdOptionsLength = std::uint32_t;

/*!
  * \brief Service discovery options length.
  */
amsr::someip_protocol::internal::SdOptionsLength options_length_{};
```
`SerializeSomeIpSdOptions`？？？


```cpp
/*!
 * \internal
 * Try to find the supplied option among the payload's options.
 * If the option is already in the payload:
 *   Return the index to the option within the payload's options.
 * Else:
 *   Add the option to the payload's options.
 *   Return the index to the newly added option within the payload's options.
 * \endinternal
 */
uint8_t ServiceDiscoveryPayloadBuilder::AddOption(options::ServiceDiscoveryOption&& option,
                                                  ServiceDiscoveryMessagePayload& payload) {
  uint8_t endpoint_idx{};
  options::ServiceDiscoveryOptionContainer::const_iterator const kIt{FindOption(payload.options, option)};
  if (kIt != payload.options.cend()) {
    // Already in the payload, compute index
    endpoint_idx =
        static_cast<amsr::someip_protocol::internal::SdEntryOptionIndex>(std::distance(payload.options.cbegin(), kIt));
  } else {
    // Not yet in the payload, push
    payload.options.emplace_back(std::move(option));
    endpoint_idx = static_cast<amsr::someip_protocol::internal::SdEntryOptionIndex>(payload.options.size() - 1U);
  }

  return endpoint_idx;
}
```

```cpp
/*!
 * \brief SOME/IP SD Message.
 */
struct ServiceDiscoveryMessagePayload {
  /*!
   * \brief Service discovery entries.
   */
  entries::ServiceDiscoveryEntryContainer entries{};
  /*!
   * \brief Service discovery options.
   */
  options::ServiceDiscoveryOptionContainer options{};
};
```


## 关联调用栈1：
`MakeOfferServiceEntry` --> `BuildMessage`(`CreateSdMessageFromPayload`) --> `BuildPayload` --> `BuildPayloadServices` --> `InsertEntry` --> `AddToPayload`
`ServiceDiscoveryMessage`中，`entries_length_`和`options_length_`需要在序列化时进行计算
```cpp
/*!
 * \brief SOME/IP SD Message.
 *
 * \trace SPEC-4981485
 * \trace SPEC-4981486
 */
struct ServiceDiscoveryMessage {
  /*!
   * \brief SOME/IP message header
   */
  amsr::someip_protocol::internal::SomeIpMessageHeader someip_header_{};

  // SD header
  /*!
   * \brief Service discovery flags
   */
  amsr::someip_protocol::internal::SdFlag flags_{};

  /*!
   * \brief Reserved bytes
   */
  std::array<std::uint8_t, 3> reserved_{};
  /*!
   * \brief Service discovery entries length
   */
  amsr::someip_protocol::internal::SdEntriesLength entries_length_{};

  /*!
   * \brief Service discovery entries.
   */
  entries::ServiceDiscoveryEntryContainer entries_{};

  /*!
   * \brief Service discovery options length.
   */
  amsr::someip_protocol::internal::SdOptionsLength options_length_{};

  /*!
   * \brief Service discovery options.
   */
  options::ServiceDiscoveryOptionContainer options_{};

  /*!
   * \brief Service discovery payload.
   */
  message::ServiceDiscoveryMessagePayload payload_{};
};
```

`ServiceDiscoveryOption`
```cpp
/*!
 * \brief Represents a SOME/IP SD option.
 */
struct ServiceDiscoveryOption {
  /*!
   * \brief Type of service discovery option
   */
  SomeIpSdEndpointOptionType type_{};

  /*!
   * \brief The endpoint IP address
   */
  amsr::someip_protocol::internal::IpAddress address_;

  /*!
   * \brief The layer 4 protocol
   */
  SomeIpSdEndpointOptionProto proto_{};

  /*!
   * \brief The layer 4 port
   */
  amsr::someip_protocol::internal::Port port_{};

  /*!
   * \brief Flag to indicate that this option is discardable
   */
  bool discardable{};

  // VECTOR NC VectorC++-V11.0.3: MD_IAM_VectorC++-V11-0-3_struct
  /*!
   * \brief Compares SOME/IP SD options.
   *
   * \param[in] other A SOME/IP SD option to compare to.
   * \return true if both options are equal and false otherwise.
   * \pre -
   * \context ANY
   * \reentrant FALSE
   */
  bool operator==(ServiceDiscoveryOption const& other) const {
    return (type_ == other.type_) && (address_ == other.address_) && (proto_ == other.proto_) && (port_ == other.port_);
  }
};
```

`cyclic offer`是`固定周期`
`repetition offer`是`二进制指数增长`

## 关键调用栈2：得到option的size
```cpp
SerializeSomeIpSdOptions(writer, sd_message.options_);
GetSomeIpSdOptionsSize(options); // 计算ServiceDiscoveryOptionContainer的长度
```

```cpp
/*!
 * \internal
 * - For all find service entries
 *   - Build the payload associated to the entry
 *   - Insert that payload into the message
 * - For all stop offer service entries
 *   - Build the payload associated to the entry
 *   - Insert that payload into the message
 * - For all offer service entries
 *   - Build the payload associated to the entry
 *   - Insert that payload into the message
 * \endinternal
 */
void ServiceDiscoveryPayloadBuilder::BuildPayloadServices(
    ServiceDiscoveryMessagePayloadContainer& payload_container, ServiceDiscoveryMessagePayload& working_payload,
    entries::FindServiceEntryContainer const& find_service_entries,
    entries::OfferServiceEntryContainer const& offer_service_entries,
    entries::StopOfferServiceEntryContainer const& stop_offer_service_entries) {
  // Add FindService entries to the payload
  for (entries::FindServiceEntry const& entry : find_service_entries) {
    // Create entries/options
    ServiceDiscoveryMessagePayload entries_and_options{FindServicePayloadBuilder::CreatePayload(entry)};
    // Insert entries/option into the container
    InsertEntry(std::move(entries_and_options), payload_container, working_payload);
  }

  // Add StopOfferService entries to the payload
  for (entries::StopOfferServiceEntry const& entry : stop_offer_service_entries) {
    // Create entries/options
    ServiceDiscoveryMessagePayload entries_and_options{StopOfferServicePayloadBuilder::CreatePayload(entry)};
    // Insert entries/option into the container
    InsertEntry(std::move(entries_and_options), payload_container, working_payload);
  }

  // Add OfferService entries to the payload
  for (entries::OfferServiceEntry const& entry : offer_service_entries) {
    // Create entries/options
    ServiceDiscoveryMessagePayload entries_and_options{OfferServicePayloadBuilder::CreatePayload(entry)};
    // Insert entries/option into the container
    InsertEntry(std::move(entries_and_options), payload_container, working_payload);
  }
}
```

！！！如果当前一个包，长度不够，就放到下一个数据包（TCP / UDP）里
```cpp
/*!
 * \internal
 * - Check if the entries and options would fit into the current working payload
 * - If they fit
 *   - Add entries and options into the current payload
 * - Otherwise
 *   - Move the current working payload into the payload container
 *   - Create a new empty working payload
 *   - Add entries and options into the new working payload
 * \endinternal
 */
void ServiceDiscoveryPayloadBuilder::InsertEntry(ServiceDiscoveryMessagePayload&& entries_and_options,
                                                 ServiceDiscoveryMessagePayloadContainer& payload_container,
                                                 ServiceDiscoveryMessagePayload& working_payload) {
  // Check if we can insert it in the current payload
  bool const can_insert{EvaluatePayload(entries_and_options, working_payload)};

  if (can_insert) {
    // Insert into our current working payload
    AddToPayload(std::move(entries_and_options), working_payload);
  } else {
    // Working payload is full. Push it into the payload container
    payload_container.emplace_back(std::move(working_payload));
    // Reset working payload
    working_payload = ServiceDiscoveryMessagePayload{};
    // Add entries and options into the new payload and update the options' indices of the entry
    AddToPayload(std::move(entries_and_options), working_payload);
  }
}
```

```cpp
/*!
 * \internal
 * Get current size of supplied payload.
 * Calculate additional size from individual SD entry's header and payload.
 * For each option in the supplied container of entries and options:
 *   If option is not already among the payload's options:
 *     Add the size this option would add to the total.
 * If the total calculated size is less than the allowed maximum SOME/IP SD payload size return true, else false.
 * \endinternal
 */
bool ServiceDiscoveryPayloadBuilder::EvaluatePayload(ServiceDiscoveryMessagePayload const& entries_and_options,
                                                     ServiceDiscoveryMessagePayload const& payload) {
  // Get size of the current payload
  std::size_t const payload_size{CalculateSizeOf(payload)};

  // Check if entries would fit
  std::size_t additional_size{amsr::someip_protocol::internal::kSdEntryHeaderSize +
                              amsr::someip_protocol::internal::kSdEntryPayloadSize};

  // Check if options would fit
  for (options::ServiceDiscoveryOption const& option : entries_and_options.options) {
    // Check if option is already in the option container
    options::ServiceDiscoveryOptionContainer::const_iterator const kOptionIt{FindOption(payload.options, option)};
    if (kOptionIt == payload.options.cend()) {
      additional_size += CalculateSizeOf(option);
    }
  }

  return (payload_size + additional_size) <= kMaxPayloadSize;
}
```

```cpp
  /*!
   * \brief Maximum SOMEIP SD payload size allowed (to be sent in a single UDP PDU)
   * \trace SPEC-4981424
   */
  static std::size_t constexpr kMaxPayloadSize{1392 /*MTU*/ - kSdMessageMinimumSize};
```


# 配置注意点
## subscribe也有`有效期`
`subscribe entry`里面的`TTL`代表这次`订阅`在服务端存活的时间是TTL秒，时间到后服务端自动取消订阅。
客户端为什么会重复订阅呢？
原来是服务端周期`offer service`，在`客户端`的sd模块的`on_message`里面处理：收到offersvice后创建resubscribe.
所以`客户端的TTL`时间要 `大于` `服务端的cyclic_offer_delay时间`
过期了就要重复订阅了











