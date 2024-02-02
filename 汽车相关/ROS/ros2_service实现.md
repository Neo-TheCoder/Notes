# 涉及到的TOPIC

## TOPIC 1
`rr/Organization/get_parametersReply`
`rcl_interfaces::srv::dds_::GetParameters_Response_`

## TOPIC 2
`rr/Organization/get_parameter_typesReply`
`rcl_interfaces::srv::dds_::GetParameterTypes_Response_`

## TOPIC 3
`rr/Organization/set_parametersReply`
`rcl_interfaces::srv::dds_::SetParameters_Response_`

## TOPIC 4
`rr/Organization/list_parametersReply`
`rcl_interfaces::srv::dds_::ListParameters_Response_`

## TOPIC 5 "Calculate"，和用户层相关
`rr/CalculateReply`
`service_interfaces::srv::dds_::Organization_Response_`

## TOPIC 6
`rr/Organization/set_parameters_atomicallyReply`
`rcl_interfaces::srv::dds_::SetParametersAtomically_Response_`

## TOPIC 7
`rr/Organization/describe_parametersReply`
`rcl_interfaces::srv::dds_::DescribeParameters_Response_`

## TOPIC 8
`rq/Organization/get_parametersRequest`
`rcl_interfaces::srv::dds_::GetParameters_Request_`

## TOPIC 9
`rr/Poorer/set_parameters_atomicallyReply`
`rcl_interfaces::srv::dds_::SetParametersAtomically_Response_`

## TOPIC 10
`rq/Organization/describe_parametersRequest`
`rcl_interfaces::srv::dds_::DescribeParameters_Request_`

## TOPIC 11
`rq/Poorer/set_parameters_atomicallyRequest`
`rcl_interfaces::srv::dds_::SetParametersAtomically_Request_`

## TOPIC 12
`rr/Poorer/describe_parametersReply`
`rcl_interfaces::srv::dds_::DescribeParameters_Response_`

## TOPIC 13
`rr/Poorer/list_parametersReply`
`rcl_interfaces::srv::dds_::ListParameters_Response_`

## TOPIC 14 ！！！
`rq/CalculateRequest`
`service_interfaces::srv::dds_::Organization_Request_`






# fastddsgen生成产物
```cpp
Organization_RequestPubSubType::Organization_RequestPubSubType()
{
    setName("service_interfaces::srv::Organization_Request");
    auto type_size = Organization_Request::getMaxCdrSerializedSize();
    type_size += eprosima::fastcdr::Cdr::alignment(type_size, 4); /* possible submessage alignment */
    m_typeSize = static_cast<uint32_t>(type_size) + 4; /*encapsulation*/
    m_isGetKeyDefined = Organization_Request::isKeyDefined();
    size_t keyLength = Organization_Request::getKeyMaxCdrSerializedSize() > 16 ?
            Organization_Request::getKeyMaxCdrSerializedSize() : 16;
    m_keyBuffer = reinterpret_cast<unsigned char*>(malloc(keyLength));
    memset(m_keyBuffer, 0, keyLength);
}
```

# 执行 ros2 service list -t
```shell
# server
/Calculate [service_interfaces/srv/Organization]
/Organization/describe_parameters [rcl_interfaces/srv/DescribeParameters]
/Organization/get_parameter_types [rcl_interfaces/srv/GetParameterTypes]
/Organization/get_parameters [rcl_interfaces/srv/GetParameters]
/Organization/list_parameters [rcl_interfaces/srv/ListParameters]
/Organization/set_parameters [rcl_interfaces/srv/SetParameters]
/Organization/set_parameters_atomically [rcl_interfaces/srv/SetParametersAtomically]

# client
/Calculate [service_interfaces/srv/Organization]
/Poorer/describe_parameters [rcl_interfaces/srv/DescribeParameters]
/Poorer/get_parameter_types [rcl_interfaces/srv/GetParameterTypes]
/Poorer/get_parameters [rcl_interfaces/srv/GetParameters]
/Poorer/list_parameters [rcl_interfaces/srv/ListParameters]
/Poorer/set_parameters [rcl_interfaces/srv/SetParameters]
/Poorer/set_parameters_atomically [rcl_interfaces/srv/SetParametersAtomically]
```
格式是：`service_name` + `service_type`




# 问题纪录
## dds发request数据，ROS Server接收数据为空
```cpp
  void
  handle_request(
    std::shared_ptr<rmw_request_id_t> request_header, std::shared_ptr<void> request) override
  {
    auto typed_request = std::static_pointer_cast<typename ServiceT::Request>(request); // 这一步取得request数据，已经是空的了。指针类型强转，只根据提供的类型信息强制转换
    auto response = std::make_shared<typename ServiceT::Response>();
    any_callback_.dispatch(request_header, typed_request, response);
    send_response(*request_header, *response);
  }
```


ROS2对于request数据的处理
```cpp
{
rmw_ret_t
__rmw_send_request(
  const char * identifier,
  const rmw_client_t * client,
  const void * ros_request,
  int64_t * sequence_id)
{
  RMW_CHECK_ARGUMENT_FOR_NULL(client, RMW_RET_INVALID_ARGUMENT);
  RMW_CHECK_TYPE_IDENTIFIERS_MATCH(
    client,
    client->implementation_identifier, identifier,
    return RMW_RET_INCORRECT_RMW_IMPLEMENTATION);
  RMW_CHECK_ARGUMENT_FOR_NULL(ros_request, RMW_RET_INVALID_ARGUMENT);
  RMW_CHECK_ARGUMENT_FOR_NULL(sequence_id, RMW_RET_INVALID_ARGUMENT);

  rmw_ret_t returnedValue = RMW_RET_ERROR;

  auto info = static_cast<CustomClientInfo *>(client->data);
  assert(info);

  eprosima::fastrtps::rtps::WriteParams wparams;
  rmw_fastrtps_shared_cpp::SerializedData data;
  data.is_cdr_buffer = false;
  data.data = const_cast<void *>(ros_request);
  data.impl = info->request_type_support_impl_;
  wparams.related_sample_identity().writer_guid() = info->reader_guid_; // 可能是这里
  if (info->request_writer_->write(&data, wparams)) {
    returnedValue = RMW_RET_OK;
    *sequence_id = ((int64_t)wparams.sample_identity().sequence_number().high) << 32 |
      wparams.sample_identity().sequence_number().low;
  } else {
    RMW_SET_ERROR_MSG("cannot publish data");
  }

  return returnedValue;
}
}
```


### `wparams`
#### WRITE_PARAM_DEFAULT
不管

#### sample_identity_
不管

#### related_sample_identity_
`writer_guid_`
`guidPrefix`
1 '\001'
15 '\017'
121 'y'
65 'A'
186 '\272'
193 '\301'
208 '\320'
210 '\322'
1 '\001'
0 '\000'
0 '\000'
0 '\000'

`entityId`
0 '\000'
0 '\000'
18 '\022'
4 '\004'

`sequence_number_`
high: -1, low: 0

解决了，这个问题无关，而是`export RMW_IMPLEMENTATION=rmw_fastrtps_cpp`的问题




# SOME/IP端调整
`parameter_server`这一app，提供method：
入参：`vector<string>`
```cpp
module dds_types {
module common {
module data_type {
typedef sequence<dds_types::common::data_type::StringType> VectorString;
}; // module data_type
}; // module common
}; // module dds_types
```

出参：`vector<Parameters>`，其中`Parameters`是`struct`:
两个成员：`String`,`ParameterValue`
`ParameterValue`是：`int`,`double`



# ROS2 client收不到fastdds response原因探究
```cpp
void
Executor::execute_any_executable(AnyExecutable & any_exec)
{
  if (!spinning.load()) {
    return;
  }
  if (any_exec.timer) {
    execute_timer(any_exec.timer);
  }
  if (any_exec.subscription) {
    execute_subscription(any_exec.subscription);
  }
  if (any_exec.service) {
    execute_service(any_exec.service);
  }
  if (any_exec.client) {  // 没执行
    execute_client(any_exec.client);
  }
  if (any_exec.waitable) {
    any_exec.waitable->execute(any_exec.data);
  }
  // Reset the callback_group, regardless of type
  any_exec.callback_group->can_be_taken_from().store(true);
  // Wake the wait, because it may need to be recalculated or work that
  // was previously blocked is now available.
  rcl_ret_t ret = rcl_trigger_guard_condition(&interrupt_guard_condition_);
  if (ret != RCL_RET_OK) {
    throw_from_rcl_error(ret, "Failed to trigger guard condition from execute_any_executable");
  }
}
```


说明以下操作失败了：
```cpp
  void
  get_next_client(
    rclcpp::AnyExecutable & any_exec,
    const WeakCallbackGroupsToNodesMap & weak_groups_to_nodes) override
  {
    auto it = client_handles_.begin();
    while (it != client_handles_.end()) { // ！！！实际上是这里的while条件为false，原因是client_handles_为空
      auto client = get_client_by_handle(*it, weak_groups_to_nodes);
      if (client) {
        // Find the group for this handle and see if it can be serviced
        auto group = get_group_by_client(client, weak_groups_to_nodes);
        if (!group) {
          // Group was not found, meaning the service is not valid...
          // Remove it from the ready list and continue looking
          it = client_handles_.erase(it);
          continue;
        }
        if (!group->can_be_taken_from().load()) {
          // Group is mutually exclusive and is being used, so skip it for now
          // Leave it to be checked next time, but continue searching
          ++it;
          continue;
        }
        // Otherwise it is safe to set and return the any_exec
        any_exec.client = client;
        any_exec.callback_group = group;
        any_exec.node_base = get_node_by_group(group, weak_groups_to_nodes);
        client_handles_.erase(it);
        return;
      }
      // Else, the service is no longer valid, remove it and continue
      it = client_handles_.erase(it);
    }
  }
```

`client_handles_`的添加是在这一步：
```cpp
  bool collect_entities(const WeakCallbackGroupsToNodesMap & weak_groups_to_nodes) override
  {
    bool has_invalid_weak_groups_or_nodes = false;
    for (const auto & pair : weak_groups_to_nodes) {
      auto group = pair.first.lock();
      auto node = pair.second.lock();
      if (group == nullptr || node == nullptr) {
        has_invalid_weak_groups_or_nodes = true;
        continue;
      }
      if (!group || !group->can_be_taken_from().load()) {
        continue;
      }
      group->find_subscription_ptrs_if(
        [this](const rclcpp::SubscriptionBase::SharedPtr & subscription) {
          subscription_handles_.push_back(subscription->get_subscription_handle());
          return false;
        });
      group->find_service_ptrs_if(
        [this](const rclcpp::ServiceBase::SharedPtr & service) {
          service_handles_.push_back(service->get_service_handle());
          return false;
        });
      group->find_client_ptrs_if( // 这一步会对client_handles_做push_back，但是后续可能被移除了：memory_strategy_->remove_null_handles(&wait_set_);
        [this](const rclcpp::ClientBase::SharedPtr & client) {
          client_handles_.push_back(client->get_client_handle());
          return false;
        });
      group->find_timer_ptrs_if(
        [this](const rclcpp::TimerBase::SharedPtr & timer) {
          timer_handles_.push_back(timer->get_timer_handle());
          return false;
        });
      group->find_waitable_ptrs_if(
        [this](const rclcpp::Waitable::SharedPtr & waitable) {
          waitable_handles_.push_back(waitable);
          return false;
        });
    }

    return has_invalid_weak_groups_or_nodes;
  }
```

`memory_strategy_->remove_null_handles(&wait_set_);`的调用是在`bool Executor::get_next_executable(AnyExecutable & any_executable, std::chrono::nanoseconds timeout)`函数中的`wait_for_work(timeout);`
!!!注意，正常情况下，根本不执行`wait_for_work`


**关键的外层调用**
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
`take_type_erased_response`是在以下函数中调用，
```cpp
static
void
take_and_do_error_handling(
  const char * action_description,
  const char * topic_or_service_name,
  std::function<bool()> take_action,
  std::function<void()> handle_action)
```

目前发现，如果执行`take_type_erased_response`，这里逻辑是没问题的

`get_next_client`一定要调用进来
```cpp
  void
  get_next_client(
    rclcpp::AnyExecutable & any_exec,
    const WeakCallbackGroupsToNodesMap & weak_groups_to_nodes) override
```

核心是`client_handles_`
```cpp
VectorRebind<std::shared_ptr<const rcl_client_t>> client_handles_;
```



**正常情况的调用**中，
```cpp
rclcpp::spin_until_future_complete(node, result)
```

```cpp
wait_for_work(timeout);
```

```cpp
if (!memory_strategy_->add_handles_to_wait_set(&wait_set_)) {
  ...
}
```

```cpp
    for (auto client : client_handles_) {
      if (rcl_wait_set_add_client(wait_set, client.get(), NULL) != RCL_RET_OK) {
        RCUTILS_LOG_ERROR_NAMED(
          "rclcpp",
          "Couldn't add client to wait set: %s", rcl_get_error_string().str);
        return false;
      }
    }
```

```c
  // Set corresponding rcl client handles NULL.
  for (i = 0; i < wait_set->size_of_clients; ++i) {
    bool is_ready = wait_set->impl->rmw_clients.clients[i] != NULL;     // !!!  client[0]如果非空，则ready
    RCUTILS_LOG_DEBUG_EXPRESSION_NAMED(is_ready, ROS_PACKAGE_NAME, "Client in wait set is ready");
    if (!is_ready) {
      wait_set->clients[i] = NULL;
    }
  }
```



```c
rcl_ret_t
rcl_wait(rcl_wait_set_t * wait_set, int64_t timeout)
// 如果条件未到达，就一直阻塞等待
```
接收一个`struct rcl_wait_set_t`的指针，
如果成员不全空（说明在某处被set了）
计算一波等待时间
调用底层的`rmw_wait`
检查timer

一直没有ready的成员会被置空，相应地更新handles

通过debug variables发现，以下代码执行完，`impl->rmw_clients->clients`就被置空了
```cpp
  rcl_ret_t status =
    rcl_wait(&wait_set_, std::chrono::duration_cast<std::chrono::nanoseconds>(timeout).count());
```


更准确来说，是在此处调用被置空的
```cpp
  // Wait.
  rmw_ret_t ret = rmw_wait(
    &wait_set->impl->rmw_subscriptions,
    &wait_set->impl->rmw_guard_conditions,
    &wait_set->impl->rmw_services,
    &wait_set->impl->rmw_clients,
    &wait_set->impl->rmw_events,
    wait_set->impl->rmw_wait_set,
    timeout_argument);
```


其中的核心判断
```cpp
  if (clients) {
    for (size_t i = 0; i < clients->client_count; ++i) {
      void * data = clients->clients[i];
      auto custom_client_info = static_cast<CustomClientInfo *>(data);
      custom_client_info->listener_->detachCondition();
      if (!custom_client_info->listener_->hasData()) {  // !!! hasData为false
        clients->clients[i] = 0;
      }
    }
  }
```


经过判断，发现是guid的问题
```cpp
// guidPrefix:
1 '\001'
15 '\017'
85 'U'
85 'U'
192 '\300'
188 '\274'
10 '\n'
229 '\345'
1 '\001'
0 '\000'
0 '\000'
0 '\000'
// entityId:
value
0 '\000'
0 '\000'
18 '\022'
4 '\004'
```




设置对端uuid逻辑在这里
```cpp
  void
  on_data_available(eprosima::fastdds::dds::DataReader * reader) final
  {
    assert(reader);

    CustomServiceRequest request;
    request.buffer_ = new eprosima::fastcdr::FastBuffer();

    rmw_fastrtps_shared_cpp::SerializedData data;
    data.is_cdr_buffer = true;
    data.data = request.buffer_;
    data.impl = nullptr;    // not used when is_cdr_buffer is true
    if (reader->take_next_sample(&data, &request.sample_info_) == ReturnCode_t::RETCODE_OK) {
      if (request.sample_info_.valid_data) {
        request.sample_identity_ = request.sample_info_.sample_identity;
        // Use response subscriber guid (on related_sample_identity) when present.
        const eprosima::fastrtps::rtps::GUID_t & reader_guid =
          request.sample_info_.related_sample_identity.writer_guid();
        if (reader_guid != eprosima::fastrtps::rtps::GUID_t::unknown() ) {
          request.sample_identity_.writer_guid() = reader_guid;
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

然后是`sequence_number`不对



```cpp
rcl_ret_t
rcl_take_response(
  const rcl_client_t * client,
  rmw_request_id_t * request_header,
  void * ros_response)
{
  rmw_service_info_t header;
  header.request_id = *request_header;
  rcl_ret_t ret = rcl_take_response_with_info(client, &header, ros_response);
  *request_header = header.request_id;
  return ret;
}
```

```cpp
typedef struct CustomClientResponse
{
  eprosima::fastrtps::rtps::SampleIdentity sample_identity_;
  std::unique_ptr<eprosima::fastcdr::FastBuffer> buffer_;
  eprosima::fastdds::dds::SampleInfo sample_info_ {};
} CustomClientResponse;
```