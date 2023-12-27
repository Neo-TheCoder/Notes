# AP com::dds method
## skeleton
fastdds线程监听是否有数据，有则触发`on_data_available`，其中调用以下函数：
解dds包，handlers_是`ara::core::Map<MethodId, Handler>`类型
`using Handler = std::function<void(SampleType&)>;`
```cpp
void Dispatch() override
{
    auto samples = reader_.Read(std::numeric_limits<size_t>::max());    // 实质是调用dds reader的take_next_sample接口
    for (auto& sample : samples) {  // 判断收到的请求，是否和offer的service::method关联
        auto discriminator = sample.data()._d();    // 是请求-应答idl数据的_d()方法，作用是返回discriminator的值
        // discriminator是判别器，也就是标识符，看起来应该存储在handlers_中
        auto handler = handlers_.find(discriminator);   // handlers_是在AddMethodHandler时添加的
        if (handler == handlers_.end()) {   // handler是std::function<void(SampleType&)>类型
            common::logger().LogError()
                << "MethodDataDispatcher::Dispatch(): Method not found for sample with discriminator "
                << discriminator;
        } else {
            handler->second(sample);
            /*
                调用以下所述的lambda，重点是EnqueueRequest()，以及onRequest_(this, nextId); onRequest_调用的实际是Execute(methodRef, requestId)：其中操作是根据用户层设置的method process处理模式来处理请求
            */
        }
    }
}
```

## proxy
维护一个`std::unordered_map<types::RequestId, ara::core::Promise<Output>`类型的`pendingRequests_`，在用户层调用method方法时：
发送请求，每发送一个method request，则创建promise对象，将`<request_id, promise>`存储到map
最后取得一个future对象，调用`get()`来取得结果，如果没收到skeleton端发来的应答消息，则阻塞在这里。
```cpp
    /// @uptrace{SWS_CM_11108, f9ed7f13070d493f8b327596e9f0670b1064a9e5}
    ara::core::Future<Output> operator()(Args&&... args) override
    {
        ara::core::Promise<Output> promise;
        auto future = promise.get_future(); // 返回和promise相关的future对象，后续可以调用get()方法得到值（会阻塞）

        std::unique_lock<std::mutex> lock(pending_requests_lock_);
        auto requestIterator = pendingRequests_.find(lastRequestId_);   // 判断是否有此request，有则是错误
        // 对象类型是std::unordered_map<types::RequestId, ara::core::Promise<Output>>
        if (requestIterator != pendingRequests_.end()) {
            lock.unlock();  // 为什么临时解锁 减小锁的粒度
            promise.SetError(ara::com::ComErrorDomainErrc::kNetworkBindingFailure);
            common::logger().LogError() << "Request Id is already used";
        } else {    // 说明是新的请求
            pendingRequests_.insert(std::make_pair(lastRequestId_, std::move(promise)));
            typename MethodTypesInfo::request_t request;
            /* ？？？最后发现这个类型是类似于Radar_Field_Request这种idl数据类型
            AP_Project/event_method_field/apd/RadarFusionMachine/src/fusion/net-bindings/fastdds/ara/com/sample/type_info_proxy_impl_radar.h

            */
            request.header.requestId.sequence_number.low = lastRequestId_;
            ++lastRequestId_;
            request.header.instanceName = ConvertToIdl(ara::core::String(instanceId_.ToString()));
            lock.unlock();
            MethodTypesInfo::SetRequestData(request, {ConvertToIdl(args)...});
                // 内部实现：
                    static void SetRequestData(request_t& request, input_t const& data)
                    {
                        request.data().Adjust(data);
                        ！！！request是idl生成类，例如fastdds_impl_type_radar_method_request.h中的Radar_Method_Call类，其中data()方法返回专门用于请求-应答的数据类型
                        例如：data()返回dds_types::ara::com::sample::Radar_Method_Call类型的数据，
                        其中的Adjust(data)方法用于为Radar_Method_Call对象的**Radar_Method_Adjust_In类型**的成员赋值
                        声明在fastdds_impl_type_radar_method_adjust_in.h
                        Radar_Method_Adjust_In的私有数据成员为：dds_types::ara::com::sample::Position m_target_position;    // 在建模时定义 实际上建模时定义了target_position为Adjust方法的入参，想必也就是在proxy侧传入的数据
                    }

            writer_->Write(request);    // request装填完就发送出去
        }

        return future;  // 把future对象返回到用户层，用户层可以调用GetResult()得到值
    }
```


fastdds线程监听是否有数据，有则触发`ReceiveRequestResults`：
判断request_id是否在map中，如果有，则说明该请求得到了应答，移除该元素，并`SetPromiseValue`
```cpp
/// @uptrace{SWS_CM_11152, 96b6448a6e34a264203675aae4a1bc2111d97b25}
void ReceiveRequestResults(typename MethodTypesInfo::reply_t& sample)
{
    ara::core::Promise<Output> promise;
    std::unique_lock<std::mutex> lock(pending_requests_lock_);
    auto request = pendingRequests_.find(sample.header().relatedRequestId().sequence_number().low());
    /// @uptrace{SWS_CM_11108, f9ed7f13070d493f8b327596e9f0670b1064a9e5}
    if (request == pendingRequests_.end()) {
        lock.unlock();
        common::logger().LogError() << "MethodImpl::ReceiveRequestResults(): sample's header is missing";
    } else {
        promise = std::move(request->second);
        pendingRequests_.erase(request);
        lock.unlock();
        MethodTypesInfo::SetPromiseValue(promise, sample);
    }
}
```






## SOMEIP_TO_DDS

```cpp
/*
    以下是skeleton端对于method方法的定义：
    出入参都可以在这里发给ROS
*/
auto radarImp::Adjust(const Position& target_position) -> decltype(Skeleton::Adjust(target_position))
{
    radarImp::AdjustOutput output;

    m_adjust_count++;

    m_logger_ctx1.LogInfo() << METHODS_HEADER << "Target position is" << target_position << ", adjusting position ...";

    if ((m_adjust_count % 7) == 0) {
        // Emulate error condition -- Adjusting position took too long
        std::this_thread::sleep_for(std::chrono::seconds(10));

        // set success to false & deviate the effective position.
        output.success = false;
        output.effective_position.x = target_position.x + m_ud_0_100(m_rand_eng);
        output.effective_position.y = target_position.y + m_ud_0_100(m_rand_eng);
        output.effective_position.z = target_position.z + m_ud_0_100(m_rand_eng);

        m_logger_ctx1.LogWarn() << METHODS_HEADER << "Adjusting position took too long time, there is something wrong";
        m_logger_ctx1.LogWarn() << METHODS_HEADER << "Effective position is" << output.effective_position;

    } else {
        // Adjusting position was successful, set success to true and show the effective position.
        output.success = true;
        output.effective_position = target_position;

        m_logger_ctx1.LogInfo() << METHODS_HEADER
                                << "Adjusting position was successful, effective position equals target position";
        m_logger_ctx1.LogWarn() << METHODS_HEADER << "Effective position is" << output.effective_position;
    }

    decltype(Skeleton::Adjust(target_position))::PromiseType promise;
    promise.set_value(std::move(output));
    return promise.get_future();
}
```




