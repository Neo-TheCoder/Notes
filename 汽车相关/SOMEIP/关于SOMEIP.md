# �������ͨ���м��SOME/IP����
## 1 SOME/IP���м����
׼ȷ��˵��ͨ��Э�飬����˵Ҳ��������Ϊͨ���м��
ȫ����**Scalable service-Oriented MiddlewarE over IP**
��������IPЭ����������Ŀ���չ��ͨ���м��Э��
���Կ���������Ҫ�أ�
1. �������SOA
2. ����IPЭ��֮�ϵ�ͨ��Э��
3. �м��

## 2 SOME/IP������
����ͨ����أ�
1. �����֣�Service Discovery��
2. Զ�̷�����ã�RPC��Remote Producer Call��
3. ��д������Ϣ��Getter & Setter��

## 3 SOME/IP��CAN������
CAN�Ǵ�ͳ���������ͨ��Э�飬CAN FD������չ
���ߵ���Ҫ�������ڣ�
1. ͨ�Ÿ��ɵĲ��죺�����ص��ֽ�����8Byte -- 1500Byte
2. ͨ�����ʵĲ���   1Mb/s -- 1000Mb/s
3. CAN���������ģ�SOME/IP����������
4. CAN�ǻ����ź���˫�����д����ź�  --  SOME/IP���ڳ�����̫���д����ź�
SOME/IPʵ������λ�ڳ�����̫������Э���е�Ӧ�ò㣬�ǻ���TCP/UDPЭ��֮�ϵ�Ӧ��
GENIVI��֯���SOME/IP��׼ʵ���˿�Դvsomeip����
SOME/IP Э��һ��ָ������ SOME/IP��SOME/IP-SD��SOME/IP-TP 3�֡�

5. ��Ϣ��ʽ
һ�������� SOME/IP ��Ϣ�������������ݣ�
MessageID��Event ID / Method ID�� -- 32bit
Length -- 32bit -- ��Ϣ����
Request ID�� Client ID & Session ID��-- 32bit
Protocal Version -- Э��汾��
Interface Version -- �ӿڰ汾��
Message Type -- ��Ϣ����
Return Code -- ���ر���
Payload -- ���ݸ��أ��ɱ䣩��SOME/IP�ײ���Ի���TCP����UDP��ʹ��Payload��������һ���������UDP����payload������1400B�������������TCP�����ݷֶδ��䣬��ôSOME/IP����ʵ�ָ��������Ĵ���
PS������SOME/IP Header���ݲ��ô�˴��䣬��Payload�е����ݴ��˳��ͨ����������

6. SOME/IP֧�ֵ����ݽṹ����
������uint��int...�����ڽṹ�����ݻ����ڴ��а���˳�����δ��

7. SOME/IP��Ϣͨ������
1. R & R��Request & Response��
2. F & F (Fire & Forget)��ֻ����Ӧ��
3. Notification Event-- �������Ļ���
�� Notification ��3�������
Cyclic Update �����Եķ������ value �ı仯
Update On Change ��� value �����仯������������
Epsilon Change ��� value ��ֵ������Ӧ�� epsilonֵ����ô����������Ϣ
5. Fields
��Ե���һ��status��������һ���Ϸ�ֵ
һ��Fields����Ӧ������Setter��Getter��Notification�Ľ�ϡ�
ִ��Setter���ͻ��˷������󣬲���������Ϣ�е�payload����ȷ��ֵ�����ڷ����Ӧ����Ϣ�е�payloadҪͬ���������ֵ
ִ��Getter���ͻ��˷�������������Ϣ�а���һ���յ�payload��������յ�������Ϣ��field��ֵ��䵽Ӧ����Ϣ�е�payload

8. SOME/IP-SD
��������Ҫ���ܣ�
̽����������ṩ����
SD����Ϣ�ṹ����SOME/IP��Ϣ��֮�ϸ�����һЩ����������Entries Array�����ڱ��Service����Event Group

�����ֵ�����
̸�� Service Discovery ֮ǰ��Ҫ�����������������
**Service**��ָ�� functional entities���������һ���.
**find**����һ����������ڱ�ʶ available ״̬�� service �������ǵ�λ��
**event**��Service Discovery ���͵� Message ����Ϊ event
**Eventgroups**��event �ļ���

service�������ϸ�ڡ�
Service��2�ࣺServer Service��Client Service�� Service��2��״̬: Down��Available��
һ��ECU��Ҫ��������Service:ServerService��ClientService��
Ҳ����˵��һ�� ECU ���Զ����ṩ����,���ʱ�������� Server Service ģ�鸺��Ҳ���Զ�������������ʱ���� Client Service �ṩ��
ʵ�ʵ����̷ǳ����ӣ�һ�� Service �� Down ״̬�� Available ״̬֮����л���һϵ�е�״̬�׶�ת����

# SOME/IP��DDS�Ĳ���
DDS���������ݵģ�ͨ�������������������ޣ����һ�ʹ���˹����ڴ棬QOS�ḻ��




# ʵ������
������**service**Ϊ��
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
## ������
**�㲥**
### OfferService
`Entries Array`����`Offer Service`������Service ID��Instance ID

### FindService
`Entries Array`����`Find Service`������Service ID��Instance ID

һ��**�㲥����**��������`Entries Array`�У�ARRAY1��ʾ`FindService`��ARRAY2��ʾ`OfferService`
Options Array�У�����Ipv4 Endpoint Option��`IP��ַ + �˿ں� + Э��UDP`



**����**
### Subscribe
`Entries Array`����`Subscribe Eventgroup Entry`
ָ��Ҫ���ĵ�`service id - instance id - Eventgroup - version`

### SubscribeAck
`Entries Array`����`Subscribe Eventgroup Ack Entry`

һ��**�㲥����**��������`Entries Array`�У�ARRAY1��ʾ`Subscribe`��ARRAY2��ʾ`SubscribeAck`

## ͨ��
### Event
server����`Message Type`Ϊ`Notification`���͵ı���

### Method
������Ҫ���ģ���**����-Ӧ��ģʽ**��
client����Type`Message Type`Ϊ`Request`���͵ı���
server����Type`Message Type`Ϊ`Response`���͵ı���



# ����`Max Nr Options Per Msg`
This parameter specifies the amount of attached options, which can be handled per incomming message.

## ��ش���
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

�������Ƚ���صĵ���ջ
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
�����Ƕ�ʱ������
ServiceDiscoveryMessageContainer const sd_message_container{sd_message_builder_.BuildMessage(imminent_message_)}; // �õ�sd_message_container

// �ദ���øú�����
SendMulticastMessage(sd_message);

PrepareSdPacket(sd_message);

SerializeSomeIpSdMessage(writer, sd_message);

SerializeSomeIpSdOptions(writer, sd_message.options_);

GetSomeIpSdOptionsSize(options); // ����ServiceDiscoveryOptionContainer�ĳ���



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



## �²�
```cpp
  OptionInterpreter interpreter{};
  ara::core::Result<void> const interpretation_result{options::OptionsInterpreter::InterpretOptions(
      entry.index_1st_opts_, entry.number_1st_opts_, entry.index_2nd_opts_, entry.number_2nd_opts_, options,
      interpreter)};
```



# ����
`Index 1st options`��Option1����Array��ڼ�����Entries Array��Index1�ֶΣ�ָʾ��ǰentries array��ѡ�������Options Array�ĵڼ��������䣩
`Index 2nd options`��Option2����Array��ڼ���
`of opt 1`��Option1����Ŀ
`of opt 2`��Option2����Ŀ
`Service ID`��Entry�����ĸ�����
`Instance ID`��Entry���ڷ�����ĸ�ʵ����0xFFFF��ʾȫ��ʵ��
`Major Version`����������汾��
`TTL`������ڡ����������ڣ���λΪ�룬�����Ϊ���ַ���ʱ������ʱ�䣬�ṩ����ʱ����Чʱ��
`Minor Version`������Ĵΰ汾��

�����֣�˵���ˣ�������취�÷����������ܹ��ҵ������ṩ�ߡ�
����ȷ�����������һ���кܶ��˵Ĺ㳡����һ���ᳪ����ˣ�����Ȼ�����������1. ����ʶ����ˣ���ǰ˵���ˣ���վ��ĳ���ط����㣬����֪���Ǹ��ط���λ�ã�����϶������׾��ҵ����ˣ�����Ǿ�̬���ã�2. �㲢����ʶ����ˣ�����һ���м��ˣ�������м���������һ���ᳪ��ģ����Ǹ���Ҳ������м������ǻᳪ��ģ���վ�ڹ㳡���ĸ�λ�ã�Ȼ���м��˰�λ�ø��㣬��Ϳ����ҵ����ˣ�����Ƕ�̬���֣���SOME/IP-SD�����Ǹ��м��ˡ�


`Entries Array`��Entry�������Ϊ����ڡ��������˷���ʵ���Լ���Ҫ���ĵ��¼������Ϣ����ΪService��Eventgroup�������ͣ�һ��SD���Ŀ��ܰ������Entry��ÿ��Entry��С����16���ֽڣ�һ��Entry���ܰ���`0-2��Option`����ͼΪһ��������SD����ʾ����

`Options Array`��Option�������Ϊѡ�������Type=0x01ʱ�����ڴ���Entry�ĸ�����Ϣ������������ȵȣ�Type=0x04ʱ�����ڴ���IPv4��صĲ�������������IP��ַ��TCP����UDP���˿ںţ�

# һ��UDP��Ϣ��`length of entries array` / `length of options array`����εó��ģ�
��someip����Ӧ��������
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
`SerializeSomeIpSdOptions`������


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


## ��������ջ1��
`MakeOfferServiceEntry` --> `BuildMessage`(`CreateSdMessageFromPayload`) --> `BuildPayload` --> `BuildPayloadServices` --> `InsertEntry` --> `AddToPayload`
`ServiceDiscoveryMessage`�У�`entries_length_`��`options_length_`��Ҫ�����л�ʱ���м���
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

`cyclic offer`��`�̶�����`
`repetition offer`��`������ָ������`

## �ؼ�����ջ2���õ�option��size
```cpp
SerializeSomeIpSdOptions(writer, sd_message.options_);
GetSomeIpSdOptionsSize(options); // ����ServiceDiscoveryOptionContainer�ĳ���
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

�����������ǰһ���������Ȳ������ͷŵ���һ�����ݰ���TCP / UDP����
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


# ����ע���
## subscribeҲ��`��Ч��`
`subscribe entry`�����`TTL`�������`����`�ڷ���˴���ʱ����TTL�룬ʱ�䵽�������Զ�ȡ�����ġ�
�ͻ���Ϊʲô���ظ������أ�
ԭ���Ƿ��������`offer service`����`�ͻ���`��sdģ���`on_message`���洦���յ�offersvice�󴴽�resubscribe.
����`�ͻ��˵�TTL`ʱ��Ҫ `����` `����˵�cyclic_offer_delayʱ��`
�����˾�Ҫ�ظ�������











