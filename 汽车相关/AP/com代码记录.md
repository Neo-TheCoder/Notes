# 1 skeleton端初始化

radar_activity.h 的`Init()`中的`OfferService()`，最后调用到：
```cpp
void Runtime::OfferService(internal::skeleton::ServiceBase& service,
    internal::ServiceId service_id,
    ara::com::InstanceIdentifier instance_id)   // 入参：radarSkeleton类型，建模生成的
{
    bool serviceRegistered{false};
    // 遍历存储service instance的vector
    for (internal::runtime::ServiceRegistry& registry : internal::runtime::ServiceRegistryList::GetInstance()) {
        serviceRegistered |= registry.registerService(service, service_id, instance_id);
    }   // ServiceRegistry对象根据service instance信息调用registerService()方法

    if (!serviceRegistered) {
        std::stringstream s;
        s << "Cannot register service " << service_id << " with instance identifier " << instance_id.ToString();
        throw std::runtime_error(s.str());
    }
}
```

```cpp
bool DdsServiceRegistryImpl::registerService(internal::skeleton::ServiceBase& service,
    ServiceId service_id,
    ara::com::InstanceIdentifier instance_id)
{
    if (!common::ToTransportIdentifier(instance_id)) {
        return false;
    }
    RegisteredObjectsMap::const_iterator registered_object = std::find_if(registered_objects_.begin(),
        registered_objects_.end(),
        [service_id, instance_id](const RegisteredObjectsMap::value_type& o) -> bool {
            return (service_id == o.second.service_id_ && instance_id == o.second.instance_id_);
        }); // 匹配已注册的对象
    // 防止二次注册
    if (registered_object != registered_objects_.end()) {
        // The object was re-registered, so only check if this was done consistently
        common::logger().LogError()
            << "DdsServiceRegistryImpl::registerService(): Registering the same service, service_id: " << service_id
            << ", instance_id:" << instance_id.ToString() << "!";
        return false;
    }

    const DdsServiceMapping::Mapping* mapping = DdsServiceMapping::GetMappingForServiceId(service_id);
    if (!mapping) { // 返回DdsServiceMapping::Mapping类型的指针
        common::logger().LogError()
            << "DdsServiceRegistryImpl::registerService(): there is no mapping for service with id " << service_id;
        return false;
    }
    // 解析MANIFEST得到instance实体
    auto instanceProperties = manifest_.getPrividedInstanceProperties(instance_id);
    if (!instanceProperties) {
        common::logger().LogError()
            << "DdsServiceRegistryImpl::registerService(): there is no manifest for instance with id "
            << instance_id.ToString();
        return false;
    }

    auto participant    // 创建participant，得到domain id和qos profile
        = DomainParticipant::get(instanceProperties.Value()->domainId_, instanceProperties.Value()->qosProfile_);
    if (!participant) {
        common::logger().LogError() << "DdsServiceRegistryImpl::registerService(): participant is nil";
        return false;
    }

    auto deployment_id = mapping->GetDeploymentId();
    common::ServiceInfo info{deployment_id, instance_id};

    auto publisher = participant->GetPublisher(info);
    if (!publisher) {
        common::logger().LogError() << "DdsServiceRegistryImpl::registerService(): publisher is nil";
        return false;
    }

    auto subscriber = participant->GetSubscriber(info);
    if (!subscriber) {
        common::logger().LogError() << "DdsServiceRegistryImpl::registerService(): subscriber is nil";
        return false;
    }

    std::unique_ptr<internal::skeleton::ServiceBase> adapter
        = mapping->GetSkeleton(service, instance_id, {participant, publisher, subscriber});
    if (!adapter) {
        common::logger().LogError() << "DdsServiceRegistryImpl::registerService(): adapter is nil";
        return false;
    }

    registered_objects_.emplace(&service, RegisteredObject{std::move(adapter), service_id, instance_id});
    return true;
}
```
`GetSkeleton`函数：
```cpp
    std::unique_ptr<internal::skeleton::ServiceBase> GetSkeleton(internal::skeleton::ServiceBase& service,
        types::InstanceId instance_id,
        common::HandleInfo handle) const override
    {
        return AdapterBuilder<SkeletonType>().make(service, instance_id, handle);
    }
// 返回值是ServiceBase类型的指针

class ServiceBase   // 抽象基类
{   // 定义了skeleton的接口 除了以下方法，派生类也要提供纯虚函数，它们必须由模型中定义的要实现method调用的server来实现
public:
    virtual ~ServiceBase(){};
    /* method方法中设定了多种模式：
    SetMethodCallProcessingMode设定用于决定何时和如何处理来自server的method调用
    */
    virtual void SetMethodCallProcessingMode(ara::com::MethodCallProcessingMode mode) = 0;
    virtual ara::core::Future<bool> ProcessNextMethodCall() = 0;    // 取得下一个调用并执行
};
```
其中的`GetSkeleton()`方法会调用到渲染生成的代码：radarServiceAdapter：
模板类ServiceMappingImpl在：`AP_Project/event_method_field/apd/RadarFusionMachine/src/radar/net-bindings/fastdds/fastdds_service_mapping-radar.cpp`专门实例化。
```cpp
ServiceMappingImpl<
    ara::com::sample::radar::service_id,
    NoProxy,
    ara::com::sample::radar_binding::dds::radarServiceAdapter>
    ara__com__sample__radar__mapping;
```

PS：由
```cpp
using ServiceInterface = skeleton::radarSkeleton;
```
可以看出，radarSkeleton类就是代码层面的service interface
Adapter其实也体现了适配器模式，接口之间做转换，形如另一种接口
radarSkeleton继承自ara::com::sample::radar和ara::com::internal::skeleton::TypedServiceImplBase<radarSkeleton>类
其中的TypedServiceImplBase<radarSkeleton>继承自ServiceImplBase类


```cpp
// ServiceMappingImpl模板类实例化所需的第三个模板参数
    radarServiceAdapter(ServiceInterface& service,
        ara::com::InstanceIdentifier instance,
        ara::com::internal::dds::common::HandleInfo handle)
        : ::ara::com::internal::dds::skeleton::ServiceImpl(service, ServiceDescriptor::kServiceId, instance, handle)
        , brakeEvent(instance)
        , parkingBrakeEvent(instance)
        , Adjust(instance)
        , UpdateRate(instance)
    {
        Connect(service);
        OfferEvent(brakeEvent);
        OfferEvent(parkingBrakeEvent);
        OfferMethod(Adjust);
        OfferField(UpdateRate);
        OfferService(ServiceDescriptor::kServiceId,
            instance,
            ServiceDescriptor::kMajorVersion,
            ServiceDescriptor::kMinorVersion);
    }

```

// 重点看下其中的构造函数做了什么操作
## 1. Connect(service)
```cpp
    void Connect(ServiceInterface& service)
    {
        service.AddDelegate(*this);
        // 增加代理，即radarServiceAdapter代理ServiceImplBase类
        // DispatcherBase<ServiceBase>提供了AddDelegate()
        service.brakeEvent.AddDelegate(brakeEvent);
        /* service.brakeEvent是radarSkeleton类的成员，是ara::com::internal::skeleton::EventDispatcher<::ara::com::sample::RadarObjects>类型
        RadarObjects是brakeEvent所用的数据类型
        入参brakeEvent是radarServiceAdapter类的成员之一，是EventImpl模板类的实例化
        */
        service.parkingBrakeEvent.AddDelegate(parkingBrakeEvent);
        Adjust.SetInvoker(MethodAdjust::CreateInvoker(dynamic_cast<ServiceInterface&>(service_)));  // service_是ServiceImpl的成员变量 ServiceBase类型（抽象类）
        /*
        Adjust是radarServiceAdapter类的成员（代表method）
        提供了SetInvoker()方法用于接收一个可调用对象：CreateInvoker内部返回的匿名函数
        */
        PS: CreateInvoker的函数体
        template <class T>
        static auto CreateInvoker(T& service)
        {
            return [&](MethodAdjust::request_t const& request) {
                auto input = request.data().Adjust();
                    ::ara::com::sample::Position result_target_position;
                    ara::com::internal::dds::ConvertFromIdl(input.target_position(),result_target_position);    // 数据转换
                return service.Adjust(  // radarSkeleton类的Adjust()是纯虚函数
                    result_target_position
                );
            };
        }

        service.UpdateRate.AddDelegate(UpdateRate);
        // UpdateRate是radarSkeleton类的成员变量，是ara::com::internal::skeleton::MutableFieldDispatcher<::ara::com::sample::RadarObjects_field>类型

    }
// ConvertFromIdl意义在于，将IDL生成的数据类，转换成简单的结构体

```

## 2. OfferEvent(brakeEvent)
```cpp
auto offerRes = event.Offer(handle_);   // 关键调用
PS：本radarServiceAdapter类就是继承自ServiceImpl类
在radarServiceAdapter构造函数中就调用了ServiceImpl提供的各种方法
handle_是common::HandleInfo类型的成员变量
定义如下：
struct HandleInfo
{
    std::shared_ptr<DomainParticipant> participant;
    std::shared_ptr<Publisher> publisher;
    std::shared_ptr<Subscriber> subscriber;
};
Offer中的逻辑：
```cpp
    ara::core::Result<void> Offer(common::HandleInfo handle) override
    {
        if (!handle.participant || !handle.publisher) {
            return ara::core::Result<void>::FromError(ara::com::ComErrorDomainErrc::kBadArguments);
        }   // participant publisher
        auto topic = handle.participant->GetTopic<EventTypesInfo>   (descriptor_type::topic_name);  // 1
        if (!topic) {
            return ara::core::Result<void>::FromError(ara::com::ComErrorDomainErrc::kUnknownError);
        }
        writer_ = handle.publisher->GetDataWriter(topic, descriptor_type::qos_profile);
        // 2
        if (!writer_) {
            return ara::core::Result<void>::FromError(ara::com::ComErrorDomainErrc::kUnknownError);
        }
        return {};
    }

    // 其中的GetTopic()
    template <typename T>
    std::shared_ptr<TypedTopic<T>> DomainParticipant::GetTopic(const char* topicName)
    {   // topicName举例：DdsTopicBrakeEvent
        if (!RegisterType<T>())
            return nullptr;
        /* DDSTypeInfo封装了dds层的TypeSupport  ？？？
        RegisterType的操作：
            在<string, TypeSupport>的map找是否存在typeName
            若无，则临时通过make_type_support()创建
            然后调用register_type()方法注册
        */

        auto topic = topics_.Get(topicName, [this, &topicName]() { return CreateTopic<T>(topicName); });    // 从topic对象池（封装了DDS的topic）取得topic指针，若无则调用第二个参数的函数：调用participant_的create_topic()方法（采用默认QOS）
        return std::dynamic_pointer_cast<TypedTopic<T>>(topic);
        // std::dynamic_pointer_cast用于在运行时确定指针所指向对象的实际类型，并将指针转换为对应类型的智能指针  比如可以将基类智能指针转为派生类指针
    }

    // 其中的GetDataWriter()执行类似操作

```

## 3. OfferMethod(Adjust)
```cpp
void ServiceImpl::OfferMethod(MethodImplBase& method)
{
    if (IsOffered()) {
        common::logger().LogError()
            << "skeleton::ServiceImpl::OfferMethod(): method should be offered before offering service";
        return;
    }
    if (method.IsOffered()) {
        common::logger().LogError() << "skeleton::ServiceImpl::OfferMethod(): method was already offered";
        return;
    }
    method.Offer(handle_, [this](MethodImplBase* methodRef, typename MethodImplBase::requestId_t requestId) {
        Execute(methodRef, requestId);  // 第二个入参：lambda，在收到proxy请求时调用
    });
    // 以下是MethodImpl实现基类方法Offer()
    // ************************************
    // 就是set 当前MethodImpl对象的回调、replyTopic、writer_、requestTopic、reader_
    onRequest_ = onRequest; // 设置回调
    auto replyTopic
        = handle.participant->GetTopic<typename dds_type_construct::reply>(MethodDescriptor::output_topic_name);    // output_topic_name
    
    writer_ = handle.publisher->GetDataWriter(replyTopic, MethodDescriptor::qos_profile);

    auto requestTopic
            = handle.participant->GetTopic<typename dds_type_construct::request>(MethodDescriptor::input_topic_name);   // input_topic_name

    reader_ = handle.subscriber->GetMethodDataReader(requestTopic, MethodDescriptor::qos_profile);
    // !!! method一并创建了MethodDataReader

    reader_->AddMethodHandler(MethodDescriptor::kMethodHash, [this](auto& sample) { EnqueueRequest(sample); });
    /*  !!! 调用MethodDataReader的AddMethodHandler()，往map添加一个lambda
        该函数内容是把sample以及id加入map，并执行onRequest_函数（该函数在Offer()的第二个入参传入
        内容是：
    */
        [this](MethodImplBase* methodRef, typename MethodImplBase::requestId_t requestId) {
                Execute(methodRef, requestId);
            }
        其中Execute(methodRef, requestId)：
        void ServiceImpl::Execute(MethodImplBase* method, typename MethodImplBase::requestId_t requestId)
        {   // ！！！MethodCallProcessingMode有多种模式
            common::logger().LogInfo() << "ServiceImpl::Execute called";
            std::unique_lock<std::mutex> lock(requestsMutex_);
            if (requestProcessingMode_ == MethodCallProcessingMode::kEventSingleThread) {
                common::logger().LogInfo() << "kEventSingleThread: calling method ProcessRequest with ID" << requestId;
                lock.unlock();
                method->ProcessRequest(requestId);  // 直接调用ProcessRequest
            } else if (requestProcessingMode_ == MethodCallProcessingMode::kEvent) {    // 事件触发
                common::logger().LogInfo() << "kEvent: calling method ProcessRequest with ID" << requestId;
                common::logger().LogInfo() << "GetPendingRequestsIds";
                auto requests = method->GetPendingRequestsIds();    // 调用EnqueueRequest时，保存了request ids
                lock.unlock();
                for (auto request : requests) {
                    common::logger().LogInfo() << "request:" << request;
                    pendingRequests_.empty_and_push(std::make_pair(request, method));
                }
                {
                    std::lock_guard<std::mutex> mlock(threadLock_);
                    startProcessing_ = true;
                }
                cond_.notify_one();     // ServiceImpl类的条件变量成员cond_，通知线程处理请求
            } else {    // 对于轮询，不需要在这里处理，实际请求数据 会在用户层去主动调用ServiceImpl::ProcessNextMethodCall()来处理接收的请求数据
                // do nothing in kPoll mode
                lock.unlock();
                common::logger().LogInfo() << "ServiceImpl::Execute kPoll for Id:" << requestId;
            }
        }

            kPoll模式中在用户层代码所做的操作：
            /// @uptrace{SWS_CM_11155, e4a512444fe4c660598f90c4543f23d358c1ad13}
                ara::core::Future<bool> ServiceImpl::ProcessNextMethodCall()
                {
                    std::lock_guard<std::mutex> guard(requestsMutex_);
                    if (requestProcessingMode_ != MethodCallProcessingMode::kPoll) {
                        throw IllegalStateException("ProcessNextMethodCall() called in non-polling mode!");
                    }

                    if (pendingRequests_.get_size() == 0) {
                        CollectPendingRequests();
                    }

                    request_t request;
                    ara::core::Promise<bool> promise;
                    if (pendingRequests_.try_pop(request)) {    // 判断非空，才出队列
                        promise.set_value(request.second->ProcessRequest(request.first));
                    } else {
                        promise.set_value(false);
                    }

                    return promise.get_future();
                }


            kEvent通过条件变量唤醒其他线程所做的操作：
            void ServiceImpl::ProcessRequestTask(uint32_t threadNum)
            {
                common::logger().LogInfo() << "ProcessRequestTask" << threadNum << "started";
                while (true) {
                    std::unique_lock<std::mutex> mlock(threadLock_);
                    cond_.wait(mlock, [this] { return startProcessing_ || shutdownWorker_; });
                    if (!shutdownWorker_) {
                        common::logger().LogInfo() << "ProcessRequestTask" << threadNum << "start processing";
                        startProcessing_ = false;
                        mlock.unlock();
                        request_t request;
                        while (pendingRequests_.try_pop(request)) {
                            common::logger().LogInfo()
                                << "ProcessRequestTask" << threadNum << "processing request id:" << request.first;
                            request.second->ProcessRequest(request.first);  // 调用了具体的method内的ProcessRequest函数
                        }
                        common::logger().LogInfo() << "ProcessRequestTask finished processing";
                    } else {
                        common::logger().LogInfo() << "ProcessRequestTask" << threadNum << "terminates";
                        mlock.unlock();
                        break;
                    }
                }
            }


            kEventSingleThread模式中的ProcessRequest()：
                bool ProcessRequest(requestId_t id) override
                {
                    if (!invoker_) {
                        common::logger().LogError() << "skeleton::MethodImpl::ProcessRequest(): invoker_ is nullptr";
                        return false;
                    }

                    std::unique_lock<std::mutex> lock(pending_requests_lock_);
                    /// @uptrace{SWS_CM_11111, 8b041812e6d247bf5ecb260cb9a9aa4b4a4d9103}
                    auto it = pendingRequests_.find(id);
                    if (it == pendingRequests_.end()) {
                        lock.unlock();
                        common::logger().LogError()
                            << "skeleton::MethodImpl::ProcessRequest(): pendingRequests_ doesn't contain target id";
                        return false;
                    }
                    // 从pendingRequests_去除请求
                    auto request = std::move(it->second);
                    pendingRequests_.erase(it); // 移除该请求
                    lock.unlock();

                    auto result = invoker_(request).GetResult(); // 此处的invoker_(request)返回值为radarImp类（继承了radarSkeleton类）实现的Adjust()方法，radarImp类和Adjust()方法已经是用户层面的
                    整体来看，method就是proxy端调用skeleton端的Adjust()函数，然后把结果返回
                    // request的类型：模板类型MethodTypesInfo::request_t，实际上传入的是定义在adapter_radar.h里的MethodAdjust结构体定义的dds_type_construct::request::dds_type_t
                    typename MethodTypesInfo::reply_t reply;    // dds_type_construct::reply::dds_type_t
                    /// @uptrace{SWS_CM_11112, 8333b7bcc1c505f3e5544511723341e9d33061f5}
                    reply.header().relatedRequestId().sequence_number().low(request.header().requestId().sequence_number().low());
                    // 具有一定格式的reply数据
                    MethodTypesInfo::SetReplyData(reply, result);   // MethodTypesInfo中的静态函数
                    // 作用是对第二个参数调用ConvertToIDL，调用reply.data().Adjust(result);此处的Ajust方法可能在fastdds_impl_type_radar_method_call.cxx中定义
                    std::unique_lock<std::mutex> writerLock(writer_lock_);
                    if (writer_) {
                        writer_->Write(reply);  // 调用data writer 发送应答数据
                    }
                    return true;
                }

    methods_.push_back(&method);
    // OfferMethod最后把调用Offer后的method存入ara::core::Vector<MethodImplBase*>


    // ************************************

}
```
## 4. OfferField(UpdateRate);
Offer的逻辑：
重点在于是event和method的结合
```cpp
    void Offer(ServiceImplBase* service) override
    {
        if (service == nullptr) {
            common::logger().LogError() << "Cannot offer field due to no service instance provided";
            return;
        }

        if (FieldDescriptor::has_notifier) {
            service->OfferEvent(event_);    // notifier意味着event
        }

        if (FieldDescriptor::has_getter) {
            service->OfferMethod(getter_);  // getter意味着method？？？
        }
    }
```

## 5. OfferService(ServiceDescriptor::kServiceId,instance,ServiceDescriptor::kMajorVersion,ServiceDescriptor::kMinorVersion);
```cpp
void ServiceImpl::OfferService(const types::ServiceId serviceId,
    const InstanceIdentifier& instanceIdentifier,
    const ::ara::com::internal::dds::types::ServiceVersionMajor majorServiceVersion,
    const ::ara::com::internal::dds::types::ServiceVersionMinor minorServiceVersion)
{
    if (IsOffered()) {
        return;
    }

    serviceInfo_ = common::ExtendedServiceInfo{serviceId, instanceIdentifier, majorServiceVersion, minorServiceVersion};

    auto participant = handle_.participant;
    auto oldUserData = participant->GetUserData();  // Get USER_DATA QoS Policy of the DDS DomainParticipant
    auto newUserData = AppendServiceToUserData(oldUserData, serviceInfo_);
    participant->SetUserData(newUserData);
    offered_ = true;
}
```

# 2 proxy端初始化
先看用户层面的部分，线程函数中调用了radar_activity.cpp中定义的`init()`函数，
关键调用：
```cpp
    auto res = Proxy::StartFindService(
        [this](ara::com::ServiceHandleContainer<Proxy::HandleType> handles, ara::com::FindServiceHandle handler) {
            RadarActivity::serviceAvailabilityCallback(std::move(handles), handler);
        },
        portSpecifier); // 参数1：callback  参数2：portSpecifier

其中的callback函数：    ？？？这个函数在做什么？
    // Callback to change to radar service offer changes.
    // Callback executed whenever a change for radar service offers happen.
    void serviceAvailabilityCallback(ara::com::ServiceHandleContainer<Proxy::HandleType> handles,
        ara::com::FindServiceHandle handler)
    {
        for (auto it : handles) {
            m_logger.LogInfo() << "Instance " << it.GetInstanceId().ToString() << " is available";
        }
        if (handles.size() > 0) {
            std::lock_guard<std::mutex> lock(m_proxy_mutex);
            if (nullptr == m_proxy) {
                m_proxy = std::make_shared<Proxy>(handles[0]);  // 实例化radarProxy对象
                m_logger.LogInfo() << "Created proxy from handle with instance: "
                                   << m_proxy->GetHandle().GetInstanceId().ToString();
                // Construct some handles (implementation-specific).
                auto first_handle = handles[0];
                auto aux_handle = first_handle;
                for (auto current_handle : handles) {
                    // Call equality operator.
                    m_logger.LogInfo() << "Check handle::operator==: "
                                       << static_cast<uint8_t>(aux_handle == current_handle);
                    // Call copy assignment operator.
                    aux_handle = current_handle;
                    // Call less-than operator.
                    m_logger.LogInfo() << "Check handle::operator<: "
                                       << static_cast<uint8_t>(aux_handle < first_handle);
                }
            }
        }
    }

其中关键调用StartFindService是基类ProxyBase的方法：
template <typename ProxyBinding>
inline ara::core::Result<FindServiceHandle> ProxyBase<ProxyBinding>::StartFindService(
    FindServiceHandler<HandleType> handler,
    InstanceIdentifier instance)
{
    using ResultType = ara::core::Result<FindServiceHandle>;
    FindServiceHandle result = Runtime::StartFindServiceById(ProxyBinding::service_id,
        instance,
        [handler](ServiceHandleContainer<std::shared_ptr<ProxyFactory>> factories, FindServiceHandle handle) {
            ServiceHandleContainer<HandleType> handles;
            for (std::shared_ptr<ProxyFactory> factory : factories) {
                handles.push_back(factory); // handles是全体Proxy的集合
            }
            handler(handles, handle);
        });
    return ResultType::FromValue(std::move(result));
}

其中调用了Runtime层的接口：
    inline static FindServiceHandle StartFindServiceById(internal::ServiceId service_id,
        ara::com::InstanceIdentifier instance_id,
        std::function<void(ServiceHandleContainer<std::shared_ptr<internal::proxy::ProxyFactory>>, FindServiceHandle)>
            handler)
    {
        Init(); // Runtime中的初始化，Initialize()，根据绑定的协议调用Register
        return DoStartFindServiceById(service_id, instance_id, handler);
    }

DoStartFindServiceById：
FindServiceHandle Runtime::DoStartFindServiceById(internal::ServiceId service_id,
    ara::com::InstanceIdentifier instance_id,
    std::function<void(ServiceHandleContainer<std::shared_ptr<ara::com::internal::proxy::ProxyFactory>>,
        FindServiceHandle)> handler)
{
    static std::atomic<std::uint32_t> uid;
    FindServiceHandle handle{service_id, instance_id, uid.fetch_add(1)};
    for (internal::runtime::ProxyFactoryBuilder& factory : internal::runtime::ProxyFactoryBuilderList::GetInstance()) {
        factory.RegisterFindServiceHandle(handle, handler);
    }
    return handle;
}

关键调用：RegisterFindServiceHandle()
void DdsProxyFactoryImpl::RegisterFindServiceHandle(FindServiceHandle handle, ProxyFactoryCallback callback)
{
    std::lock_guard<Mutex> guard(mutex_);
    ServiceHandleContainer<std::shared_ptr<internal::proxy::ProxyFactory>> availableInstances;
    const DdsServiceMapping::Mapping* mapping = DdsServiceMapping::GetMappingForServiceId(handle.service_id);
    if (!mapping) {
        common::logger().LogWarn()
            << "DdsProxyFactoryImpl::RegisterFindServiceHandle(): there is no mapping for service with id "
            << handle.service_id;
        return;
    }

    if (handle.instance_id.IsAny()) {//instance_id   is "any"   ！！！any则
        auto instances = manifest_.getRequiredServiceProperties(handle.service_id); // 遍历domainReqiredServiceProperties_得到的ara::core::Vector<DdsServiceInstanceProperties>
        if (instances.empty()) {
            common::logger().LogWarn()
                << "DdsProxyFactoryImpl::RegisterFindServiceHandle(): there is no service with id "
                << handle.service_id;
        }
        for (auto const itr : instances) {  // 遍历instance
            auto participant = SubscribeOnParticipantUpdates(itr.domainId_, itr.qosProfile_);   // ？？？其中调用OnAvailableServicesChanged(serviceInfoSet)，创建participant
            if (participant) {
                findHandles_.emplace(handle.uid, common::FindServiceInfo{handle, callback, participant});
            } else {
                common::logger().LogError() << "DdsProxyFactoryImpl::RegisterFindServiceHandle(): participant is nil";
            }
        }
        availableInstances = GenerateAnyFactoryList(availableServices_, *mapping);
        // 创建participant、proxy factory
    } else {    // instance id 不为any，为某具体值

        auto instanceProperties = manifest_.getRequiredInstanceProperties(handle.instance_id);  // 返回const DdsServiceInstanceProperties*类型，涉及一些建模时的配置
        if (!instanceProperties) {
            common::logger().LogError()
                << "DdsProxyFactoryImpl::RegisterFindServiceHandle(): there is no manifest for instance with id "
                << handle.instance_id.ToString();
            return;
        }
        auto participant = SubscribeOnParticipantUpdates(
            instanceProperties.Value()->domainId_, instanceProperties.Value()->qosProfile_);
        if (!participant) {
            common::logger().LogError() << "DdsProxyFactoryImpl::RegisterFindServiceHandle(): participant is nil";
            return;
        }

        const auto& emplaceResult
            = findHandles_.emplace(handle.uid, common::FindServiceInfo{handle, callback, participant}); // Id,FindServiceInfo类型的map
        if (emplaceResult.second != true) {
            common::logger().LogWarn() << "DdsProxyFactoryImpl::RegisterFindServiceHandle(): a handle with uid "
                                       << handle.uid << " was already been added";
        }
        auto availableFactory
            = GenerateAvailableInstanceFactory(*mapping, mapping->GetDeploymentId(), handle.instance_id, participant);
            // PS：GenerateAvailableInstanceFactory()
                std::shared_ptr<internal::proxy::ProxyFactory> DdsProxyFactoryImpl::GenerateAvailableInstanceFactory(
                        const DdsServiceMapping::Mapping& mapping,
                        const types::ServiceId serviceId,
                        const types::InstanceId instanceId,
                        std::shared_ptr<DomainParticipant> participant)
                    {
                        if(std::any_of(availableServices_.begin(),
                            availableServices_.end(),
                            [serviceId, instanceId](const common::ServiceInfo& info) {
                                return ((info.serviceId == serviceId) && (info.instanceId == instanceId));
                            })) // 匹配id
                        {
                        return GetProxy(mapping, serviceId, instanceId, participant);  // ！！！ 调用ProxyBuilder对象的make()，得到ProxyFactory对象
                        }
                        else{
                            //don't use builtin reader check service userdata, just use it;
                            std::set<common::ExtendedServiceInfo> serv_info_set;
                            serv_info_set.insert({serviceId, instanceId, mapping.GetServiceVersionMajor(), mapping.GetServiceVersionMinor()});
                            OnAvailableServicesChanged(serv_info_set);
                        }

                        return nullptr;
                    }


        if (availableFactory) {
            availableInstances.push_back(std::move(availableFactory));
        }
    }
    if (!availableInstances.empty()) {
        common::logger().LogDebug()
            << "DdsProxyFactoryImpl::RegisterFindServiceHandle(): services already available, service id: "
            << handle.service_id;
        callback(availableInstances, handle);   // ？？？--> 实际就是执行了serviceAvailabilityCallback函数
                // 该callback的内容是层层传递过来的:
                [handler](ServiceHandleContainer<std::shared_ptr<ProxyFactory>> factories, FindServiceHandle handle) {
                    ServiceHandleContainer<HandleType> handles;
                    for (std::shared_ptr<ProxyFactory> factory : factories) {
                        handles.push_back(factory); // handles是全体Proxy的集合
                    }
                    handler(handles, handle);
                    // handler的内容是:
                        [this](ara::com::ServiceHandleContainer<Proxy::HandleType> handles, ara::com::FindServiceHandle handler) {
                        RadarActivity::serviceAvailabilityCallback(std::move(handles), handler);
                        },
                }
    }
}

```

# 3 skeleton端通信
## 1 event
**从用户层深入**
```cpp
//brakeEvent sample
    auto brakeEvent_allocation_result = m_skeleton->brakeEvent.Allocate();
    if(!brakeEvent_allocation_result){
        m_logger_ctx3.LogError() << "brakeEvent allocation failed with error: " << brakeEvent_allocation_result.Error();
    }else{
        auto brakeEventSample = std::move(brakeEvent_allocation_result).Value();
        // set value:
        // brakeEventSample ->object = value

        // send sample
        auto send_result = m_skeleton->brakeEvent.Send(std::move(brakeEventSample));
        // ！！！此处是调用ara::com::internal::skeleton::EventDispatcher<::ara::com::sample::RadarObjects>的Send()方法
        // 调用栈中的完整类型是这样：
        ara::com::internal::skeleton::EventDispatcher<ara::com::sample::RadarObjects,ara::com::internal::skeleton::Event>::Send(ara::com::internal::skeleton::EventDispatcher<ara::com::sample::RadarObjects, ara::com::internal::skeleton::Event> * const this, ara::com::SampleAllocateePtr data) 
        PS: EventDispatcher模板类是这样定义的：
            template <typename T, template <typename> class Element = Event>
            class EventDispatcher : public DispatcherBase<Element<T>>
            // 这种语法意思是：该模板类有两个模板参数T和Element，T是普通的类型参数，Element是一个模板类型参数，默认为Event，也就是说EventDispatcher继承DispatcherBase<Event<RadarObjects>>

            // 内部实现
                ara::core::Result<void> Send(ara::com::SampleAllocateePtr<T> data)
                {
                    // The magic also must be applied here, for the delegate the allocation has been performed before
                    // should be able to handle the element, for all the other elements, the data shall be sent before
                    if (Dispatcher::delegates_.size() == 1) {   // ？？？1是什么意思
                        return Dispatcher::delegates_.front()->Send(std::move(data));
                                // 内部实现：
                                EventImpl类的Send()方法：
                                ara::core::Result<void> Send(ara::com::SampleAllocateePtr<value_type> data) override
                                {
                                    return Send(*data);
                                }
                                    // 内部实现：
                                        /// \uptrace{SWS_CM_11017, 20a656e2f1bbf3c45a27585325dc58b22d9bc273}
                                        ara::core::Result<void> Send(const_reference data) override
                                        {
                                            using ReturnType = ara::core::Result<void>;
                                            try {
                                                if (IsOffered()) {  // 判断DataWriter是否有效
                                                    ara::core::Optional<uint16_t> optionalId = common::ToTransportIdentifier(instanceId_);
                                                    if (optionalId) {
                                                        // writer_->Write({*optionalId, ConvertToIdl(data)});
                                                        writer_->Write(EventTypesInfo::MakeObject(*optionalId, data));
                                            // 调用DataWriter的Write()方法
                                            // EventTypesInfo实际是EventbrakeEventTypeInfo，它是渲染出的代码，定义如下：
                                            struct EventbrakeEventTypeInfo
                                            {
                                                using ara_type_t = ::ara::com::sample::RadarObjects;
                                                    using dds_type_t = dds_types::ara::com::sample::RadarObjectsEventType;
                                                using dds_type_support_t = dds_types::ara::com::sample::RadarObjectsEventTypePubSubType;
                                                static constexpr char* dds_type_name="dds_types::ara::com::sample::RadarObjectsEventType";
                                            };
                                            // MakeObject内部调用ConvertToIdl，把结构体数据递归地转为dds的类型
                                                        return {};
                                                                }
                                                }
                                            } catch (...) {
                                            }
                                            return ReturnType::FromError({ara::com::ComErrorDomainErrc::kNetworkBindingFailure});
                                        }

                    } else {
                        return Send(*data);
                    }   // 其中的delegates_是Event<RadarObjects>*类型的list
                }

        if (send_result) {
            m_logger_ctx3.LogInfo() << "brakeEvent sent";
        } else {
            m_logger_ctx3.LogError() << "brakeEvent.Send failed with error: " << send_result.Error();
        }
    }
```


## 2 method
在OfferMethod的Offer方法中：
```cpp
    method.Offer(handle_, [this](MethodImplBase* methodRef, typename MethodImplBase::requestId_t requestId) {
        Execute(methodRef, requestId);
    })
```
（初始化阶段已记录了）
创建DataReader，MethodDataReaderListener

**当proxy侧的app向skeleton侧发送请求...**（怎么发送请求的详情见proxy侧的记录）
在创建MethodDataReader时创建了MethodDataDispatcher对象
调用MethodDataReaderListener的`on_data_available()`方法：
```cpp
    /// @uptrace{SWS_CM_11020, 95ae0a0855d8338d23a058118b07bda90315d478}
    inline void on_data_available(eprosima::fastdds::dds::DataReader* reader) override
    {   // 监听proxy端发来的请求
        static_cast<void>(reader);  // 为了消除编译器的未使用警告？
        dispatcher_.Dispatch();
    }
    // 其中Dispatch()的实现：
        void Dispatch() override
    {
        auto samples = reader_.Read(std::numeric_limits<size_t>::max());    // 实质是调用dds reader的take_next_sample接口
        for (auto& sample : samples) {  // 判断收到的请求，是否和offer的service::method关联
            auto discriminator = sample.data()._d();    // 是请求-应答idl数据的_d()方法，作用是返回discriminator的值
            // discriminator是判别器，也就是标识符，看起来应该存储在handlers_中
            auto handler = handlers_.find(discriminator);   // handlers_是在AddMethodHandler时添加的，handlers_存储着method_id
            if (handler == handlers_.end()) {   // handler是std::function<void(SampleType&)>类型
                common::logger().LogError()
                    << "MethodDataDispatcher::Dispatch(): Method not found for sample with discriminator "
                    << discriminator;
            } else {
                handler->second(sample);
                /*
                    调用以下所述的lambda，重点是EnqueueRequest()，以及onRequest_(this, nextId); onRequest_调用的实际是Execute(methodRef, requestId)
                */
            }
        }
    }
                // 此处的second(sample)，实际上是调用了初始化（调用Offer(common::HandleInfo handle, OnRequestType onRequest)）时AddMethodHandler()添加的函数，也就是EnqueueRequest
                        reader_->AddMethodHandler(MethodDescriptor::kMethodHash, [this](auto& sample) { EnqueueRequest(sample); });
                    的EnqueueRequest函数
                    // 内部实现：
                        void EnqueueRequest(typename MethodTypesInfo::request_t& sample)
                        {
                            std::unique_lock<std::mutex> lock(pending_requests_lock_);
                            auto nextId = GetNextId();
                            pendingRequests_[nextId] = std::move(sample);   // 增加<method, method data>
                            lock.unlock();
                            onRequest_(this, nextId);
                            // onRequest_ 该函数在初始化时Offer()的第二个入参传入 ！！！也就是处理请求的逻辑
                            ！！！详情见初始化的分析：Execute(methodRef, requestId)：
                        }

    ！！！MethodDataReaderListener类的on_data_available()方法何时调用？？？
    --->    MethodDataReader的构造函数中：（在skeleton侧初始化时构造）
    this->reader_->set_listener(listener, this->listenerStatusMask_);
    接受了一个我们定义的MethodDataReaderListener对象
```
最后在用户层代码，有`Adjust()`函数的具体定义，将结果`set_value`，然后返回future对象





# 4 proxy端通信
## 1 event
在初始化时订阅，如果有数据处理回调则调用
```cpp
auto brakeEvent_subscription_result = m_proxy->brakeEvent.Subscribe(3); // maxSampleCount
Subscribe内部GetTopic、GetEventDataReader、并设置了两个回调函数，通过这两个方法：
SetSubscriptionStateChangeHandler()、SetReceiveHandler()
入参是maxSampleCount，它是用于存储数据的缓存的大小
对应的成员是internal::proxy::EventDataContainer<ara_type_t>类型（其中有成员std::deque，双端队列，兼备list和vector，用于记录空节点，以及vector类型，用于存储数据本身）
```

在**用户层面**是通过GetNewSamples方法获取数据的
```cpp
m_proxy->brakeEvent.GetNewSamples(callback);
```
内部实现如下：
```cpp
    ara::core::Result<size_t> GetNewSamples(new_samples_callback&& f,
        size_t maxNumberOfSamples = std::numeric_limits<size_t>::max()) override
    {
        std::lock_guard<std::mutex> guard(lock_);

        if (!samplesCache_.AvailableSize()) {
            return ara::core::Result<size_t>::FromError(ara::com::ComErrorDomainErrc::kMaxSamplesExceeded);
        }

        size_t toReadNumber = std::min(samplesCache_.AvailableSize(), maxNumberOfSamples);
        size_t passedToUserSamplesNumber = 0;

        auto samples = reader_->Read(toReadNumber); // 调用DataReader读取数据
        for (auto& sample : samples) {
            value_type data;    // Read得到的是dds层面的数据，转为AP的数据类型
            ConvertFromIdl(sample.data(),data);
            auto dataPtr = samplesCache_.Add(data, ara::com::e2e::ProfileCheckStatus::kCheckDisabled);
            // 把数据加入缓存vector 返回一个用于增加空节点empty_nodes_的函数
            if (dataPtr) {  // 调用该函数
                f(std::move(dataPtr));
                passedToUserSamplesNumber++;    // proxy侧接收到一个数据
            }
        }

        return passedToUserSamplesNumber;
    }
```


## 2 method
事先在初始化时通过MethodImpl构造了所需的东西（MethodDataReader）

### 关于如何发送请求
在用户层代码（即`act()`中）中，调用m_proxy(radarProxy类型)的Adjust对象
```cpp
    auto adjust_future = m_proxy->Adjust(tmp);  // 如同这样的调用
```
其中的tmp是建模时定义，本文举例的工程中service interface定义了Adust方法中：
1. `ara::com::sample::Position`类型的target_position参数 in
2. bool类型的success参数 out
3. `ara::com::sample::Position`类型的effective_position参数 out
`FIRE_AND_FORGET`设置为false  --> `FIRE_AND_FORGET`即无需等待该调用的完成或获取其返回结果。这意味着调用者不关心该调用的结果和执行状态，而是只关注调用操作的触发。

注意此处的`Adjust`是起了别名：
```cpp
using Adjust = ara::com::internal::proxy::Method<ara::com::sample::radar::AdjustOutput(const ::ara::com::sample::APString&)>;
实际上是`Method`对象的派生类`MethodImpl`重载了operator()
由于radarProxy类型的对象在初始化时为成员变量赋值的是派生类对象，因此实现了多态，调用到以下的代码：
```

发送请求的函数：
`MethodImpl`对象对外提供**函数调用**
看起来是普通的函数，实际上做的是：
    创建promise对象，得到future对象，将`<requestId, future>`加入`pendingRequests_`
    Proxy端维护`<requestId, future>`的map，还用在`ReceiveRequestResults(&sample)`时`SetPromiseValue`
    ！！！`ReceiveRequestResults`是在**proxy端的MethodImpl初始化**的时候传给reader的
        `ReceiveRequestResults`的真正调用是在dds的`on_data_available`，里面的`dispatch`方法的实现中，通过`Read()`拿到数据，再触发函数
        `SetPromiseValue`的实现在生成的代码中，负责解DDS的包，真正地给promise `set_value`
        PS：因为`ReceiveRequestResults`的调用是在dds的线程里，所以需要使用promise进行线程间传值（通过promise端维护的`<requestId, future>`map取得相应的promise）
    然后将request组到DDS包调用dds的`Write()`发送出去

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




## 3 field
### 概览
#### skeleton侧
1. 更新、通知一个field的值的改变
2. 为接收到的Get()调用提供服务
2. 为接收到的Set()调用提供服务
```cpp
// 代码如下：
class UpdateRate {
public:
    using FieldType = uint32_t;

    void Update(const FieldType& data); // 等价于event通信中的Send()方法，触发对订阅的clients的notify（如果配置）
    // 对于已配置的Getter，必须至少调用一次Update以设置初值

    /* （可选）收到get请求时，调用该函数
        如果未注册Getter，ara::com要负责返回update设置的上一个值
        这隐式地要求服务初始化后、服务提供前至少一次对update的调用，这取决于service的实现
        get handler函数要返回一个future对象
    */
    void RegisterGetHandler(std::function<ara::core::Future<FieldType>()> getHandler);


    /*
    只要field支持。注册SetHandler是必须的
    该handler会获取发送者请求set的数据，
    要验证设置，执行内部数据的更新，field的新值应该在后续设置
    */
    void RegisterSetHandler(std::function<ara::core::Future<FieldType>(const FieldType& data)> setHandler);
};
```

#### proxy侧
概念上来说，一个field在任何时候都有一个确定的值，这不像event，这就导致相比event，field有以下的不同：
1. 如果订阅了一个field，就会以类似event通知的模式，“立马”把初值就发回给subscriber
可以通过调用`Get()`方法查询当前字段值，也可以通过 Set() 方法更新当前字段值。

注意：一个field提供的所有特征都是可选的：
“on change notification”、Get()、Set()

proxy端field被用来：
调用`Get()`或`Set()`方法
以事件/事件数据的形式访问field更新通知，由proxy所连接的服务实例发送，其机制与常规事件完全相同
```cpp
class UpdateRate {
using FieldType = uint32_t;
void Subscribe(size_t maxSampleCount);
ara::core::Result<size_t> GetFreeSampleCount() const noexcept;
ara::com::SubscriptionState GetSubscriptionState() const;
void Unsubscribe();
void SetReceiveHandler(ara::com::EventReceiveHandler handler);
void UnsetReceiveHandler();
void SetSubscriptionStateChangeHandler(ara::com::SubscriptionStateChangeHandler handler);
void UnsetSubscriptionStateChangeHandler();

template <typename F>
ara::core::Result<size_t> GetNewSamples(F&& f, size_t maxNumberOfSamples = std::numeric_limits<size_t>::max());

// 得到skeleton端的实际值，没有入参，返回future对象
ara::core::Future<FieldType> Get();

/*
    设置新值，取决于服务提供者接收或者修改请求
    新值要被发回给请求者作为应答，因此是有返回值的
*/ 
ara::core::Future<FieldType> Set(const FieldType& value);
};
```

客户端可以通过远程调用Getter方法获取Field的值，
也可以通过远程调用Setter方法设置Field的值。另外和Event相似，当客户端订阅了某个事件组，若Event Group中包含的Field发生变化，服务端会主动的通过Notification消息通知客户端；
当然，用户也可以选择周期发送Notification消息。

**Field和Event的区别是：Field是一个持续存在的变量，比如多媒体音量、车速、环境温度等，这些可以在任何时刻获取；而Event指的是一个事件，事件没有发生就不存在，比如发生碰撞，出现故障等。**




# SOME/IP版本实现
## skeleton
### field
重要类：
`FieldDispatcher`，继承自`EventDispatcher`

#### Notify
```cpp
    /**
     * \brief Sends the referenced data.
     *
     * This version of Update is memory intensive if the event data is large and sending occurs quite
     * frequent due to the time needed to copy the data to the output buffers. However, it does not require a call to
     * \ref Allocate and it can send arbitrary data buffers allocated by the user.
     * \param data Data to send
     *
     * \uptrace{SWS_CM_00116, 5b26bfe8313d5929411ca99e5749075c50a276cc}
     */
    ara::core::Result<void> Update(const_reference data)
    {
        typename LastValueType::LockedPtr last_value{last_value_sent_->Get()};  // 返回一个<std::tuple<T, bool>, unique_lock对象>构造的LockedPtrWrapper对象
        std::get<0>(*last_value) = data;    // 解引用以取得元组对象，更新数据
        std::get<1>(*last_value) = true;    // 这个bool的意义是在FieldDispatcher内部的其他函数里做判断，比如是否初始化
        return Base::Send(data);
    }

// 其中：LastValueType：
    using LastValueType = internal::common::MutexWrapper<std::tuple<T, bool>, std::mutex>;
    /*
        MutexWrapper封装了一个数据结构，从而实现线程安全的访问
        提供Get()，返回LockedPtrWrapper类型的对象
    */

// LockedPtr：
    using LockedPtr = LockedPtrWrapper<T, M>;  ///< Proxy type that manages the associated lock.
    /*
        LockedPtrWrapper将互斥锁（mutex）的guard对象和对同步内部数据结构的引用进行封装，可以理解为线程安全的指针
        如寻常指针，可以调用->，使用assert判断guard_拥有锁的所有权，然后返回&inner_.get()，get()会返回实际对象
        解引用同理
    */
```
该函数用于判断field数据是否有更新。
如果事件数据很大并且发送频率很高，这个版本的Update函数可能会占用较多的内存，因为需要将数据复制到输出缓冲区。但是，它不需要调用Allocate函数，并且可以发送用户分配的任意数据缓冲区。
`FieldDispatcher`在初始化的时候，`last_value_sent_`赋值为false


#### Get
`MutableFieldImpl`对象提供`Get()`方法，其他线程会去取得值，当前线程取得future对象
`MutableFieldImpl`继承自`FieldImpl<FieldDescriptor, T>`、`proxy::MutableField<T>`

include/ara/com/internal/vsomeip/proxy/vsomeip_mutable_field_impl.h
```cpp
/*
    MutableField虚继承了Field
    特点是持有一个MethodSet
*/
template <typename FieldDescriptor, typename T>
/**
 * \brief Implementation for mutable field interface for proxy side
 */
class MutableFieldImpl
    : public FieldImpl<FieldDescriptor, T>
    , public internal::proxy::MutableField<T>
{
public:
    MutableFieldImpl(types::InstanceId instance_id)
        : internal::proxy::Field<T>(FieldImpl<FieldDescriptor, T>::get_method_)
        , FieldImpl<FieldDescriptor, T>(instance_id)
        , internal::proxy::MutableField<T>(FieldImpl<FieldDescriptor, T>::get_method_, set_method_)
        , set_method_(instance_id)
    { }

private:
    MethodImpl<MakeSetMethodDescriptorFrom<FieldDescriptor>,
        typename internal::proxy::MutableField<T>::MethodSet::signature_type>
        set_method_;
};
```


include/ara/com/internal/vsomeip/proxy/vsomeip_field_impl.h
```cpp
/**
 * \brief Field implementation for Proxy Interface
 */
/*
    其中的Field基类虚继承了Event，还继承了bits::FieldBase（只有一个虚析构函数，这样设计意义何在？）
    持有一个Method<T()>类型的引用，！！！Method<T()> 表示一个无参数、返回类型为 T 的函数指针类型，即函数类型为 T()
*/
template <typename FieldDescriptor, typename T>
class FieldImpl
    : public virtual internal::proxy::Field<T>
    , public EventImpl<FieldDescriptor, T>
{
public:
    using MethodGet = typename internal::proxy::Field<T>::MethodGet;
    /**
     * \param instance_id InstanceId to create the FieldImpl object
     */
    explicit FieldImpl(types::InstanceId instance_id)
        : internal::proxy::Field<T>(get_method_)
        , EventImpl<FieldDescriptor, T>(instance_id)
        , get_method_(instance_id)
    { }

    virtual ~FieldImpl()
    { }

    virtual ara::core::Result<void> Subscribe(std::size_t maxSampleCount) override
    {
        return EventImpl<FieldDescriptor, T>::DoSubscribe(maxSampleCount, true);
    }

protected:
    /**
     * \brief This is the get method proxy instance that is used if someone calls Get() on this field.
     */
    MethodImpl<MakeGetMethodDescriptorFrom<FieldDescriptor>,
        typename internal::proxy::Field<T>::MethodGet::signature_type>
        get_method_;
};
```

在skeleton端，`get`和`set`，都是用户自定义函数，注册到`MutableFieldImpl`对象
`FieldImpl`构造时，调用`registerMethodHandler`，把get回调封装到`handleGet`中调用，把`handleGet`传到vsomeip的接口，设置收到消息时触发的回调
```cpp
runtime::VSomeIPConnection::getInstance().getApplication().register_message_handler(
    service_id, instance_id, method_id, callback);
```

skeleton端监听到method request时，触发以下函数，调用用户层传入的field::`get()`函数，取得返回值：future对象


然后调用`SendReply(reply, msg)`，将应答数据发往proxy端
```cpp
    /// \brief This function handles a Get() call from the client to the server.
    /// \param msg Get message sent by the client.
    /// \uptrace{SWS_CM_10302, 8adc98396e0c955435c6d6db492f85f8353c76ea}
    /// \uptrace{SWS_CM_10334, 8d2ec3a6b5530c143de24a8984afc8c226644a32}
    void handleGet(const std::shared_ptr<::vsomeip::message>& msg) const
    {
        bool isValidMessage = common::checkMessageMetaData(*msg,
            FieldDescriptor::service_id,
            FieldDescriptor::get_method_id,
            FieldDescriptor::service_version_major,
            ::ara::com::internal::vsomeip::types::MessageTypeRequest,
            ::ara::com::internal::vsomeip::types::MessageTypeRequestNoReturn,
            true,
            true);

        if (isValidMessage) {
            std::function<ara::core::Future<T>()> get_handler = RetrieveHandler(get_handler_);
            if (get_handler) {
                auto reply = get_handler().GetResult();
                ServiceImplBase::SendReply(reply, msg);
            } else {
                throw IllegalStateException("No Get handler registered!");
            }
        } else {
            common::logger().LogWarn() << "Discarding invalid message";
        }
    }
```


同样地，对于set函数：
因为有了入参，所以稍微复杂一些
解包时要解出要set的数据
```cpp
    /// \brief  This function handles a Set() call from the client.
    /// \param msg The message contains the data the value of the field shall be set to.
    /// \uptrace{SWS_CM_10302, 8adc98396e0c955435c6d6db492f85f8353c76ea}
    /// \uptrace{SWS_CM_10334, 8d2ec3a6b5530c143de24a8984afc8c226644a32}
    void handleSet(const std::shared_ptr<::vsomeip::message>& msg)
    {
        bool isValidMessage = common::checkMessageMetaData(*msg,
            FieldDescriptor::service_id,
            FieldDescriptor::set_method_id,
            FieldDescriptor::service_version_major,
            ::ara::com::internal::vsomeip::types::MessageTypeRequest,
            ::ara::com::internal::vsomeip::types::MessageTypeRequestNoReturn,
            true,
            true);

        if (isValidMessage) {
            std::function<SetMethodSignature> set_handler = Base::RetrieveHandler(set_handler_);
            if (set_handler) {
                common::Unmarshaller<T> unmarshaller(*msg);
                T value_to_set = unmarshaller.template unmarshal<0>();
                auto reply = set_handler(value_to_set).GetResult();
                ServiceImplBase::SendReply(reply, msg);
            } else {
                throw IllegalStateException("No Set handler registered!");
            }
        } else {
            common::logger().LogWarn() << "Discarding invalid message";
        }
    }
```


##### 关于field的值
在skeleton端，一直存在一个`m_update_rate`，代表field的变量，在skeleton端的get()中，为其设置新的值，然后将其`std::move`到promise的`set_value()`，除非被再次赋值，否则就没法使用`m_update_rate`了





## proxy

### field

#### 接收notify
把接收notify消息放在`receive handler`里，在其他线程（vsomeip开的线程）中执行
```cpp
// 在proxy端，DoSubscribe时设置message_handler，收到消息才触发
    app.register_message_handler(EventDescriptor::service_id,
        instance_id_,
        EventDescriptor::event_id,
        [this](const std::shared_ptr<::vsomeip::message>& msg) { onMessage(msg); });
```


#### get
proxy端调用field的get()，最深层调用的是Field初始化时传进来的`MethodGet`（是一个`MethodImpl`类型的对象）。
这个初始化操作是在最外层的`radarImpl`初始化时完成的，
`MutableFieldImpl`的实例化需要两个类型参数：
1. `FieldDescriptor`是一个struct，存储模型中该service中field的相关id，
2. 实际数据类型

`MutableFieldImpl`是field相关类的最外层派生类，继承FieldImpl<FieldDescriptor, T>、MutableField<T>
构造时，同时构造给自己的一个个基类

`MethodImpl`的构造需要一个instance_id，该对象由`FieldImpl`对象持有，传递给`Field`
```cpp
class MutableFieldImpl
    : public FieldImpl<FieldDescriptor, T>
    , public internal::proxy::MutableField<T>
{
public:
    MutableFieldImpl(types::InstanceId instance_id)
        : internal::proxy::Field<T>(FieldImpl<FieldDescriptor, T>::get_method_)
        , FieldImpl<FieldDescriptor, T>(instance_id)
        , internal::proxy::MutableField<T>(FieldImpl<FieldDescriptor, T>::get_method_, set_method_)
        , set_method_(instance_id)
    { }
// ......
}
```

```cpp
// Field的接口
ara::core::Future<T> Get()
{
    return m_get_method_body_ref(); // ！！！
}
```
`MethodGet`重载了`operator()`，持有一个`MethodImplDelegate<MethodDescriptor, result_type>`类型的指针，调用了`startCall(std::forward<Args>(args)...)`：

```cpp
// proxy端，调用field的method接口：get()，最后调用到此处
    template <typename... Args>
    ara::core::Future<ResultType> startCall(Args&&... args)
    {
        using internal::vsomeip::runtime::VSomeIPConnection;

        VSomeIPConnection& connection = VSomeIPConnection::getInstance();
        std::shared_ptr<::vsomeip::message> request = CreateRequest(connection, instance_, std::forward<Args>(args)...);    // ！！！创建了someip request message

        ara::core::Promise<ResultType> promise;
        auto future = promise.get_future();

        std::lock_guard<std::mutex> lock_guard(pending_requests_lock_);
        connection.getApplication().send(request, true);    // 调用vsomeip接口发送请求
        Request r = std::make_pair(request, std::move(promise));
        addPendingRequest(std::move(r));    // ！！！pending_requests_由MethodImplDelegate持有

        return future;
    }
```


当接收到skeleton端发来的vsomeip response应答数据，`onMessage`回调触发，判断当前request id是否存在于自己维护的`std::unordered_map<types::RequestId, Request>`类型的`pending_requests_ map`中，如果是，则解包并set promise
```cpp
    /// \brief This is a callback function
    ///
    /// On message arrival checks if the message already in the map and remove
    /// if match. Also process the response.
    /// \param response Message received as response from vsomeip
    /// \uptrace{SWS_CM_10358, bfb7a30fdb76d2ee20bafd00dcd075403e132147}
    void onMessage(const std::shared_ptr<::vsomeip::message>& response)
    {
        types::RequestId request_id = response->get_request();
        typename PendingRequestMap::iterator it = pending_requests_.find(request_id);
        if (it != pending_requests_.end()) {
            if (validateMessage(*it->second.first, *response)) {
                detail::DeserializeAndSetPromise(std::move(it->second.second), response);
            } else {
                common::logger().LogWarn() << "Discarding invalid message";
                it->second.second.SetError(ara::core::ErrorCode{ara::com::ComErrorDomainErrc::kNetworkBindingFailure});
            }
            pending_requests_.erase(it);
        }
    }
```


