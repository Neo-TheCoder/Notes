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





















