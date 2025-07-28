
```cpp
  eprosima::fastrtps::rtps::WriteParams wparams;
  wparams.related_sample_identity().writer_guid() = writer_guid;
  wparams.related_sample_identity().sequence_number().high = high;
  wparams.related_sample_identity().sequence_number().low = low;

  writer_Method_Responsetopic_->write(&data, wparams);
// 和request保持一致
```


# fastdds api reference


## WriteParams
包含一个`CacheChange`的额外信息

## writer_guid：
writer_guid是用于唯一标识数据写入者（writer）的属性。每个写入者都有一个唯一的writer_guid，用于区分不同的数据源。通过writer_guid，接收者可以准确地识别数据的来源，确保数据的可靠传输和正确处理。

## sequence_number：
sequence_number是用于标识数据样本顺序的属性。在数据通信中，每个数据样本都会被分配一个唯一的sequence_number，用于指示数据样本的顺序。通过sequence_number，接收者可以按照正确的顺序重建数据流，确保数据的完整性和准确性。

### high


### low



# 当前代码梳理
ROS2 client（fastdds writer0）发送request数据给
someip_to_dds(fastdds reader0)
`rq/MethodRequest`


someip_to_dds(fastdds writer1)调用fastdds接口发送数据给ROS2 client(fastdds reader1)
`rr/MethodReply`
writer1的write参数`WriteParams`：
1. writer_guid
要和writer0保持一致

2. sequence_number
要和ROS2 client发过来的数据的`info.sample_identity.sequence_number()`保持一致



# 修改建议：
## ros做server
ap的method返回future，其中临时创建promise，可以直接返回future给client app
？如何共享promise
  直接引用传递

检查一下ros2对guid的检测
## ros做client
1. 专门处理一个线程轮询多个future对象，是否Ok，ok就发

2. 
```cpp
  ara::core::Future<service1::proxy::methods::StartApplicationMethod1::Output> future =
      service1_proxy_->StartApplicationMethod1(start_application_method1_value);
```
  可以放在`on_data_available`里面，那样就不用条件变量线程间同步，因为有future对象

# ROS2部分相关源码记录
## ros2 client
### client对于response消息的处理
```cpp
typedef struct CustomClientInfo
{
  eprosima::fastdds::dds::TypeSupport request_type_support_{nullptr};
  const void * request_type_support_impl_{nullptr};
  eprosima::fastdds::dds::TypeSupport response_type_support_{nullptr};
  const void * response_type_support_impl_{nullptr};
  eprosima::fastdds::dds::DataReader * response_reader_{nullptr};
  eprosima::fastdds::dds::DataWriter * request_writer_{nullptr};

  std::string request_topic_;
  std::string response_topic_;

  ClientListener * listener_{nullptr};
  eprosima::fastrtps::rtps::GUID_t writer_guid_;
  eprosima::fastrtps::rtps::GUID_t reader_guid_;

  const char * typesupport_identifier_{nullptr};
  ClientPubListener * pub_listener_{nullptr};
  std::atomic_size_t response_subscriber_matched_count_;
  std::atomic_size_t request_publisher_matched_count_;
} CustomClientInfo;
```

先在触发`on_data_available`回调，验证guid后，将数据存入list
```cpp
  void
  on_data_available(eprosima::fastdds::dds::DataReader * reader)
  {
    assert(reader);

    CustomClientResponse response;
    // Todo(sloretz) eliminate heap allocation pending eprosima/Fast-CDR#19
    response.buffer_.reset(new eprosima::fastcdr::FastBuffer());

    rmw_fastrtps_shared_cpp::SerializedData data;
    data.is_cdr_buffer = true;
    data.data = response.buffer_.get();
    data.impl = nullptr;    // not used when is_cdr_buffer is true
    if (reader->take_next_sample(&data, &response.sample_info_) == ReturnCode_t::RETCODE_OK) {
      if (response.sample_info_.valid_data) {
        response.sample_identity_ = response.sample_info_.related_sample_identity;

        if (response.sample_identity_.writer_guid() == info_->reader_guid_ ||
          response.sample_identity_.writer_guid() == info_->writer_guid_) // response消息对应的writer，要本端client的writer guid一致
        {
          std::lock_guard<std::mutex> lock(internalMutex_);

          if (conditionMutex_ != nullptr) {
            std::unique_lock<std::mutex> clock(*conditionMutex_);
            list.emplace_back(std::move(response));
            // the change to list_has_data_ needs to be mutually exclusive with
            // rmw_wait() which checks hasData() and decides if wait() needs to
            // be called
            list_has_data_.store(true);
            clock.unlock();
            conditionVariable_->notify_one();
          } else {
            list.emplace_back(std::move(response));
            list_has_data_.store(true);
          }
        }
      }
    }
  }
```


在`execute_client`调度逻辑中，
按照以下调用栈执行：
`take_type_erased_response`
`rcl_take_response`
`rcl_take_response_with_info`
`rmw_take_response`
`__rmw_take_response`
```cpp
rmw_ret_t
__rmw_take_response(
  const char * identifier,
  const rmw_client_t * client,
  rmw_service_info_t * request_header,
  void * ros_response,
  bool * taken)
{
  RMW_CHECK_ARGUMENT_FOR_NULL(client, RMW_RET_INVALID_ARGUMENT);
  RMW_CHECK_TYPE_IDENTIFIERS_MATCH(
    client,
    client->implementation_identifier, identifier,
    return RMW_RET_INCORRECT_RMW_IMPLEMENTATION);
  RMW_CHECK_ARGUMENT_FOR_NULL(request_header, RMW_RET_INVALID_ARGUMENT);
  RMW_CHECK_ARGUMENT_FOR_NULL(ros_response, RMW_RET_INVALID_ARGUMENT);
  RMW_CHECK_ARGUMENT_FOR_NULL(taken, RMW_RET_INVALID_ARGUMENT);

  *taken = false;

  auto info = static_cast<CustomClientInfo *>(client->data);
  assert(info);

  CustomClientResponse response;

  if (info->listener_->getResponse(response)) {
    auto raw_type_support = dynamic_cast<rmw_fastrtps_shared_cpp::TypeSupport *>(
      info->response_type_support_.get());
    eprosima::fastcdr::Cdr deser(
      *response.buffer_,
      eprosima::fastcdr::Cdr::DEFAULT_ENDIAN,
      eprosima::fastcdr::Cdr::DDS_CDR);
    if (raw_type_support->deserializeROSmessage(
        deser, ros_response, info->response_type_support_impl_))
    {
      request_header->source_timestamp = response.sample_info_.source_timestamp.to_ns();
      request_header->received_timestamp = response.sample_info_.reception_timestamp.to_ns();
      request_header->request_id.sequence_number =
        ((int64_t)response.sample_identity_.sequence_number().high) <<
        32 | response.sample_identity_.sequence_number().low;

      *taken = true;
    }
  }

  return RMW_RET_OK;
}
```

需要sequence_number保持一致的原因：确保response和request匹配
```cpp
  /// Handle a server response
  /**
    * \param[in] request_header used to check if the secuence number is valid
    * \param[in] response message with the server response
   */
  void
  handle_response(
    std::shared_ptr<rmw_request_id_t> request_header,
    std::shared_ptr<void> response) override
  {
    std::unique_lock<std::mutex> lock(pending_requests_mutex_);
    auto typed_response = std::static_pointer_cast<typename ServiceT::Response>(response);
    int64_t sequence_number = request_header->sequence_number;
    // TODO(esteve) this should throw instead since it is not expected to happen in the first place
    if (this->pending_requests_.count(sequence_number) == 0) {  // 因为client端会记录pending的request，此处的request_header是从对端server得来
      RCUTILS_LOG_ERROR_NAMED(
        "rclcpp",
        "Received invalid sequence number. Ignoring...");
      return;
    }
    auto tuple = this->pending_requests_[sequence_number];
    auto call_promise = std::get<0>(tuple);
    auto callback = std::get<1>(tuple);
    auto future = std::get<2>(tuple);
    this->pending_requests_.erase(sequence_number);
    // Unlock here to allow the service to be called recursively from one of its callbacks.
    lock.unlock();

    call_promise->set_value(typed_response);
    callback(future);
  }
```
pending_requests_的数据结构是：`std::map<int64_t, std::tuple<SharedPromise, CallbackType, SharedFuture>> pending_requests_;`
在把response通过`set_value`传递给上层之前，判断当前request的sequence number是否有效。
？？？`request_header`怎么来的？






#### ros2 client处理response的调用栈
```cpp
void
Executor::execute_client(
  rclcpp::ClientBase::SharedPtr client)
{
  auto request_header = client->create_request_header();
  std::shared_ptr<void> response = client->create_response();
  take_and_do_error_handling(
    "taking a service client response from service",
    client->get_service_name(),
    [&]() {return client->take_type_erased_response(response.get(), *request_header);},
    [&]() {client->handle_response(request_header, response);});
}
```
先临时创建`std::shared_ptr<void> response`，然后通过`take_type_erased_response()`接口取得

##### `take_and_do_error_handling()`的实现
```cpp
static
void
take_and_do_error_handling(
  const char * action_description,
  const char * topic_or_service_name,
  std::function<bool()> take_action,
  std::function<void()> handle_action)
{
  bool taken = false;
  try {
    taken = take_action();
  } catch (const rclcpp::exceptions::RCLError & rcl_error) {
    RCLCPP_ERROR(
      rclcpp::get_logger("rclcpp"),
      "executor %s '%s' unexpectedly failed: %s",
      action_description,
      topic_or_service_name,
      rcl_error.what());
  }
  if (taken) {
    handle_action();
  } else {
    // Message or Service was not taken for some reason.
    // Note that this can be normal, if the underlying middleware needs to
    // interrupt wait spuriously it is allowed.
    // So in that case the executor cannot tell the difference in a
    // spurious wake up and an entity actually having data until trying
    // to take the data.
    RCLCPP_DEBUG(
      rclcpp::get_logger("rclcpp"),
      "executor %s '%s' failed to take anything",
      action_description,
      topic_or_service_name);
  }
}
```
可见，`take response`和`handle response`的操作是同步的：
因为`take_action()`即`take_type_erased_response(response.get(), *request_header)`
`handle_action()`即`handle_response()`

`create_request_header()`：
```cpp
/// An rmw service request identifier
typedef struct RMW_PUBLIC_TYPE rmw_request_id_t
{
  /// The guid of the writer associated with this request
  int8_t writer_guid[16];

  /// Sequence number of this service
  int64_t sequence_number;
} rmw_request_id_t;
```
`create_request_header()`仅仅是构造一个`struct`，它的`sequence_number`是随机的
`create_response()`构造了用户层的response数据类型

`take_type_erased_response()`
```cpp
bool
ClientBase::take_type_erased_response(void * response_out, rmw_request_id_t & request_header_out)
{
  rcl_ret_t ret = rcl_take_response(
    this->get_client_handle().get(),
    &request_header_out,
    response_out);
  if (RCL_RET_CLIENT_TAKE_FAILED == ret) {
    return false;
  } else if (RCL_RET_OK != ret) {
    rclcpp::exceptions::throw_from_rcl_error(ret);
  }
  return true;
}
```

`rcl_take_response`调用`rcl_take_response_with_info()`，在该函数中，set request header
```cpp
rcl_ret_t
rcl_take_response_with_info(
  const rcl_client_t * client,
  rmw_service_info_t * request_header,
  void * ros_response)
{
  RCUTILS_LOG_DEBUG_NAMED(ROS_PACKAGE_NAME, "Client taking service response");
  if (!rcl_client_is_valid(client)) {
    return RCL_RET_CLIENT_INVALID;  // error already set
  }

  RCL_CHECK_ARGUMENT_FOR_NULL(request_header, RCL_RET_INVALID_ARGUMENT);
  RCL_CHECK_ARGUMENT_FOR_NULL(ros_response, RCL_RET_INVALID_ARGUMENT);

  bool taken = false;
  request_header->source_timestamp = 0;
  request_header->received_timestamp = 0;
  if (rmw_take_response(
      client->impl->rmw_handle, request_header, ros_response, &taken) != RMW_RET_OK)
  {
    RCL_SET_ERROR_MSG(rmw_get_error_string().str);
    return RCL_RET_ERROR;
  }
  RCUTILS_LOG_DEBUG_NAMED(
    ROS_PACKAGE_NAME, "Client take response succeeded: %s", taken ? "true" : "false");
  if (!taken) {
    return RCL_RET_CLIENT_TAKE_FAILED;
  }
  return RCL_RET_OK;
}
```


`rmw_take_response()`调用到`__rmw_take_response()`
```cpp
rmw_ret_t
__rmw_take_response(
  const char * identifier,
  const rmw_client_t * client,
  rmw_service_info_t * request_header,
  void * ros_response,
  bool * taken)
{
  RMW_CHECK_ARGUMENT_FOR_NULL(client, RMW_RET_INVALID_ARGUMENT);
  RMW_CHECK_TYPE_IDENTIFIERS_MATCH(
    client,
    client->implementation_identifier, identifier,
    return RMW_RET_INCORRECT_RMW_IMPLEMENTATION);
  RMW_CHECK_ARGUMENT_FOR_NULL(request_header, RMW_RET_INVALID_ARGUMENT);
  RMW_CHECK_ARGUMENT_FOR_NULL(ros_response, RMW_RET_INVALID_ARGUMENT);
  RMW_CHECK_ARGUMENT_FOR_NULL(taken, RMW_RET_INVALID_ARGUMENT);

  *taken = false;

  auto info = static_cast<CustomClientInfo *>(client->data);
  assert(info);

  CustomClientResponse response;

  if (info->listener_->getResponse(response)) {
    auto raw_type_support = dynamic_cast<rmw_fastrtps_shared_cpp::TypeSupport *>(
      info->response_type_support_.get());
    eprosima::fastcdr::Cdr deser(
      *response.buffer_,
      eprosima::fastcdr::Cdr::DEFAULT_ENDIAN,
      eprosima::fastcdr::Cdr::DDS_CDR);
    if (raw_type_support->deserializeROSmessage(
        deser, ros_response, info->response_type_support_impl_))
    {
      request_header->source_timestamp = response.sample_info_.source_timestamp.to_ns();
      request_header->received_timestamp = response.sample_info_.reception_timestamp.to_ns();
      request_header->request_id.sequence_number =
        ((int64_t)response.sample_identity_.sequence_number().high) <<
        32 | response.sample_identity_.sequence_number().low;

      *taken = true;
    }
  }

  return RMW_RET_OK;
}
```
！！！从`info->listener_->getResponse(response)`可以看出，listener_维护从`on_data_available`中取得的response队列，每次处理队头元素。
！！！通过`if (this->pending_requests_.count(sequence_number) == 0)`来匹配request和response



















### server对于request消息的处理
```cpp
typedef struct CustomServiceRequest
{
  eprosima::fastrtps::rtps::SampleIdentity sample_identity_;
  eprosima::fastcdr::FastBuffer * buffer_;
  eprosima::fastdds::dds::SampleInfo sample_info_ {};

  CustomServiceRequest()
  : buffer_(nullptr) {}
} CustomServiceRequest;
```

```cpp
  void
  on_data_available(eprosima::fastdds::dds::DataReader * reader) final
  {
    assert(reader);

    CustomServiceRequest request; // ros2对于service通信方式自定义的结构体，记录request的各种信息
    request.buffer_ = new eprosima::fastcdr::FastBuffer();

    rmw_fastrtps_shared_cpp::SerializedData data;
    data.is_cdr_buffer = true;
    data.data = request.buffer_;
    data.impl = nullptr;    // not used when is_cdr_buffer is true
    if (reader->take_next_sample(&data, &request.sample_info_) == ReturnCode_t::RETCODE_OK) { // dds reader在take数据时，传入SampleInfo类型
      if (request.sample_info_.valid_data) {
        request.sample_identity_ = request.sample_info_.sample_identity;  // 在自定义结构体中，记录request数据的信息
        // Use response subscriber guid (on related_sample_identity) when present.
        const eprosima::fastrtps::rtps::GUID_t & reader_guid =
          request.sample_info_.related_sample_identity.writer_guid();
        if (reader_guid != eprosima::fastrtps::rtps::GUID_t::unknown() ) {
          request.sample_identity_.writer_guid() = reader_guid; // 记录request对应的writer_guid信息到自定义结构体中
        }

        // Save both guids in the clients_endpoints map
        const eprosima::fastrtps::rtps::GUID_t & writer_guid =
          request.sample_info_.sample_identity.writer_guid();
        info_->pub_listener_->endpoint_add_reader_and_writer(reader_guid, writer_guid);

        std::lock_guard<std::mutex> lock(internalMutex_);

        if (conditionMutex_ != nullptr) {
          std::unique_lock<std::mutex> clock(*conditionMutex_);
          list.push_back(request);
          // the change to list_has_data_ needs to be mutually exclusive with
          // rmw_wait() which checks hasData() and decides if wait() needs to
          // be called
          list_has_data_.store(true);
          clock.unlock();
          conditionVariable_->notify_one();
        } else {
          list.push_back(request);
          list_has_data_.store(true);
        }
      }
    }
  }
```


### server对于reponse消息的处理
```cpp
rmw_ret_t
__rmw_send_response(
  const char * identifier,
  const rmw_service_t * service,
  rmw_request_id_t * request_header,
  void * ros_response)
{
  RMW_CHECK_ARGUMENT_FOR_NULL(service, RMW_RET_INVALID_ARGUMENT);
  RMW_CHECK_TYPE_IDENTIFIERS_MATCH(
    service,
    service->implementation_identifier, identifier,
    return RMW_RET_INCORRECT_RMW_IMPLEMENTATION);
  RMW_CHECK_ARGUMENT_FOR_NULL(request_header, RMW_RET_INVALID_ARGUMENT);
  RMW_CHECK_ARGUMENT_FOR_NULL(ros_response, RMW_RET_INVALID_ARGUMENT);

  rmw_ret_t returnedValue = RMW_RET_ERROR;

  auto info = static_cast<CustomServiceInfo *>(service->data);  // 拿到payload
  assert(info);

  eprosima::fastrtps::rtps::WriteParams wparams;  // 临时创建WriteParams变量
  rmw_fastrtps_shared_cpp::copy_from_byte_array_to_fastrtps_guid(
    request_header->writer_guid,
    &wparams.related_sample_identity().writer_guid());            // 把request_header中记录的writer_guid、sequence_number传入wparams
  wparams.related_sample_identity().sequence_number().high =
    (int32_t)((request_header->sequence_number & 0xFFFFFFFF00000000) >> 32);
  wparams.related_sample_identity().sequence_number().low =
    (int32_t)(request_header->sequence_number & 0xFFFFFFFF);

  // TODO(MiguelCompany) The following block is a workaround for the race on the
  // discovery of services. It is (ab)using a related_sample_identity on the request
  // with the GUID of the response reader, so we can wait here for it to be matched to
  // the server response writer. In the future, this should be done with the mechanism
  // explained on OMG DDS-RPC 1.0 spec under section 7.6.2 (Enhanced Service Mapping)

  // According to the list of possible entity kinds in section 9.3.1.2 of RTPS
  // readers will have this bit on, while writers will not. We use this to know
  // if the related guid is the request writer or the response reader.
  constexpr uint8_t entity_id_is_reader_bit = 0x04;
  const eprosima::fastrtps::rtps::GUID_t & related_guid =
    wparams.related_sample_identity().writer_guid();
  if ((related_guid.entityId.value[3] & entity_id_is_reader_bit) != 0) {
    // Related guid is a reader, so it is the response subscription guid.
    // Wait for the response writer to be matched with it.
    auto listener = info->pub_listener_;
    client_present_t ret = listener->check_for_subscription(related_guid);
    if (ret == client_present_t::GONE) {
      return RMW_RET_OK;
    } else if (ret == client_present_t::MAYBE) {
      RMW_SET_ERROR_MSG("client will not receive response");
      return RMW_RET_TIMEOUT;
    }
  }

  rmw_fastrtps_shared_cpp::SerializedData data;
  data.is_cdr_buffer = false;
  data.data = const_cast<void *>(ros_response);
  data.impl = info->response_type_support_impl_;
  if (info->response_writer_->write(&data, wparams)) {
    returnedValue = RMW_RET_OK;
  } else {
    RMW_SET_ERROR_MSG("cannot publish data");
  }

  return returnedValue;
}
```




# 博强
调用ap的method接口要加锁吗？
超时、错误机制

request队列 --> 异步
因为dds也有Buffer，所以`on_data_available`久也没关系，write操作很快，也不会阻塞。



# 胡楠
reader收到第一个数据的`on_data_available`回调没执行完，第二个收包不会进行


# 记录
可以把轮询某个future对象是否ready的操作封装成任务/函数，如果被set_value了，就返回true，











数据队列是必要的
考虑ROS client多次调用同一个method的情况，而request的数据不用队列记录
第一个method，在someip_to_dds这端调用：
```cpp
  ara::core::Future<service1::proxy::methods::StartApplicationMethod1::Output> future =
      service1_proxy_->StartApplicationMethod1(start_application_method1_value);

  ara::core::Result<service1::proxy::methods::StartApplicationMethod1::Output, ara::core::ErrorCode>
      future_result = future.GetResult();
```
迅速拿到了结果，然后又有ROS client再次调用，而`GetResult`等待了很久，而ROS client第三次调用，那第二个数据就丢了






每个data type对应一个data type list，对应一个线程，执行：转发给APP server--等待--发回ROS client，可以让这每个线程，维护一个future队列
```cpp
void StartDDSDataConsumers() {
    dds_data_consumer_threads_.emplace_back([this] {
    ::dds_types::data_type::DynamicObjects data;
    while (!exit_requested_) {
      if ( sensorsmartcameradynamicobjectstopic_events_.Pop(data)) {
          SensorSmartCameraDynamicObjectsTopicDataCallback(data);
        }
      }
    });

    dds_data_consumer_threads_.emplace_back([this] {
    ::dds_types::data_type::LaneLines data;
    while (!exit_requested_) {
      if ( smartcameralanelinestopic_events_.Pop(data)) {
          SmartCameraLaneLinesTopicDataCallback(data);
        }
      }
    });

}

```

# ROS client，关键行为
1. 调用DDS接口监听ROS client发来的request数据
```cpp
void SomeipToDdsParticipant::Method_Requesttopic_SubListener::on_data_available(DataReader* reader) {
// 记录guid、sequence number
}
// 拿到数据，存入request队列
```
2. 将request数据type convert，调用ara::com method接口，采用IPC将request数据发给server app，得到一个future对象，阻塞等待，得到response数据

request队列对应线程执行以下逻辑
```cpp
  static ::StartApplicationMethod1::Output someip_data;
  type_convert::ConvertFromIdl(dds_data, someip_data);

  ara::core::Future<service1::proxy::methods::StartApplicationMethod1::Output> future = service1_proxy_->StartApplicationMethod1(start_application_method1_value);

  ara::core::Result<service1::proxy::methods::StartApplicationMethod1::Output, ara::core::ErrorCode>
      future_result = future.GetResult();
```
3. 调用DDS接口发送response数据
```cpp
  auto participant = SomeipToDdsParticipant::GetInstance();
  ::service_interface::srv::Method_Response response;
  
  response.output_argument(future_result.Value().output_argument);

  participant->SendMethodResponseToROS(response);
```



PS: 当前实现中，`SomeipToDdsParticipant`的构造在`StartServer()`中进行
`SomeipToDds`持有若干service的unique_ptr，可以让`SomeipToDds`增加接口：将数据写入各个service类各自维护的data queue，该接口在`on_data_available`中调用
每个`SubListener`持有一个函数对象，`SomeipToDdsParticipant`持有对这些函数对象set的接口，这些接口的调用是在每个service interface类的`StartServer()`中，通过单例模式直接调用`SomeipToDdsParticipant`提供的set函数对象的接口
当前`SomeipToDds`直接创建在main函数，分配在栈上









# 关于类间关系
## SomeipToDdsParticipant
### 处理dds侧相关的逻辑
DDS各种类的创建
`收`、`发`DDS数据

## StartApplicationCmClientService1
代表skeleton这端的某一个service
对外提供`StartClient()`、`ShutdownClient()`
对应若干种数据类型

## StartApplicationCmClient1
代表skeleton实体，持有若干个service的unique_ptr








# PS
别忘记，还需要type convert
namespace、类型名、变量名






