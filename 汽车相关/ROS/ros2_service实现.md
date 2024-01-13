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
/Calculate [service_interfaces/srv/Organization]
/Organization/describe_parameters [rcl_interfaces/srv/DescribeParameters]
/Organization/get_parameter_types [rcl_interfaces/srv/GetParameterTypes]
/Organization/get_parameters [rcl_interfaces/srv/GetParameters]
/Organization/list_parameters [rcl_interfaces/srv/ListParameters]
/Organization/set_parameters [rcl_interfaces/srv/SetParameters]
/Organization/set_parameters_atomically [rcl_interfaces/srv/SetParametersAtomically]
```






