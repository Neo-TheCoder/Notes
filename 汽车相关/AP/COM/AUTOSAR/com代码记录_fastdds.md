# 前言
## 涉及到的设计模式
### 代理模式
`代理模式`可以为其他对象提供一个代理，以便在访问对象时添加额外的功能或控制访问的方式。

**三个关键角色**
1. `目标对象`
定义了代理对象和真实对象的共同接口，代理对象和真实对象都实现了这个接口。

2. `代理对象`
持有一个对真实对象的引用，并在其上执行操作。代理对象通常在执行操作之前或之后执行额外的逻辑，例如权限验证、缓存、延迟加载等。

3. `真实对象`
实际执行操作的对象。

代理模式的**核心思想**是通过代理对象来控制对真实对象的访问。
当客户端需要访问真实对象时，它并不直接与真实对象交互，而是通过代理对象来完成操作。代理对象可以在执行操作前后添加额外的逻辑，或者在必要时延迟创建真实对象。

而在AP的框架中，由于有些实现类要根据用户输入的模型来生成，需要涉及到一定的代码生成，而这些生成的类负责实际的一些操作，就涉及到代理模式，而ara-api层面的对象是代理对象，而他们都继承自某些抽象类，就是目标对象。举例如下：

1. `目标对象`
是生成出来的
```cpp
/// @uptrace{SWS_CM_00002, 4d3a2b51c78573d1d34cd3d2aff93527b4a97d63}
class radarSkeleton
    : public ara::com::sample::radar
    , public ara::com::internal::skeleton::TypedServiceImplBase<radarSkeleton>
{
public:
    /// @uptrace{SWS_CM_00130, 296a1309a7a0c7ed9c22860bac1f8e8da1d6d781}
    radarSkeleton(ara::com::InstanceIdentifier instance_id,
        ara::com::MethodCallProcessingMode mode = ara::com::MethodCallProcessingMode::kEvent)
        : ara::com::internal::skeleton::TypedServiceImplBase<radarSkeleton>(instance_id, mode)
    { }

    /// @uptrace{SWS_CM_00152, b772a2873b407e2731b5b05529fb74cda8bce3df}
    radarSkeleton(ara::core::InstanceSpecifier instanceSpec,
        ara::com::MethodCallProcessingMode mode = ara::com::MethodCallProcessingMode::kEvent)
        : ara::com::internal::skeleton::TypedServiceImplBase<radarSkeleton>(std::move(instanceSpec), mode)
    { }

    /// @uptrace{SWS_CM_00153, 8b59a23d227f152aad94749801c8ddea75d1622d}
    radarSkeleton(ara::com::InstanceIdentifierContainer instanceIDs,
        ara::com::MethodCallProcessingMode mode = ara::com::MethodCallProcessingMode::kEvent)
        : ara::com::internal::skeleton::TypedServiceImplBase<radarSkeleton>(std::move(instanceIDs), mode)
    { }

    virtual ~radarSkeleton()
    {
        StopOfferService();
    }

    /// @brief Skeleton shall be move constructable.
    ///
    /// @uptrace{SWS_CM_00135, 3e590d73cd77c5afbc592cec60094e43c8dfbf97}
    explicit radarSkeleton(radarSkeleton&&) = default;

    /// @brief Skeleton shall be move assignable.
    ///
    /// @uptrace{SWS_CM_00135, 3e590d73cd77c5afbc592cec60094e43c8dfbf97}
    radarSkeleton& operator=(radarSkeleton&&) = default;

    /// @brief Skeleton shall not be copy constructable.
    ///
    /// @uptrace{SWS_CM_00134, 988e39db973396943ac710bbeb3a6e4af033b727}
    explicit radarSkeleton(const radarSkeleton&) = delete;

    /// @brief Skeleton shall not be copy assignable.
    ///
    /// @uptrace{SWS_CM_00134, 988e39db973396943ac710bbeb3a6e4af033b727}
    radarSkeleton& operator=(const radarSkeleton&) = delete;

    void OfferService()
    {
        if (!UpdateRate.IsInitialized()) {
            throw ara::com::IllegalStateException(
                "Attempt to offer service \"/apd/serviceinterfaces/radar\" with uninitialized field \"UpdateRate\"");
        }

        ara::com::internal::skeleton::TypedServiceImplBase<radarSkeleton>::OfferService();
    }

    /// @uptrace{SWS_CM_00191, e3ee8aca56f9a3b371df808324769584093419f3}
    using ara::com::sample::radar::AdjustOutput;
    virtual ara::core::Future<AdjustOutput> Adjust(
        const ::ara::com::sample::Position& target_position
    ) = 0;
    fields::UpdateRate UpdateRate;
    events::brakeEvent brakeEvent;
    events::parkingBrakeEvent parkingBrakeEvent;
};
```


2. `代理对象`
是生成出来的，继承了`radarSkeleton`
```cpp
class radarImp : public ara::com::sample::skeleton::radarSkeleton
{
    using Skeleton = ara::com::sample::skeleton::radarSkeleton;

public:
    radarImp(ara::core::InstanceSpecifier instanceSpec, ara::com::MethodCallProcessingMode mode)
        : Skeleton(std::move(instanceSpec), mode)   // 调用基类构造函数
        , m_worker(&radarImp::ProcessRequests, this)
    { } // 构造时启了一个method专用的线程

    virtual ~radarImp()
    {
        m_finished = true;
        m_worker.join();    // 主线程会调用该析构，然后调用join()，等待m_worker结束
    }

    virtual auto Adjust(
        const ::ara::com::sample::Position& target_position
        ) -> decltype(Skeleton::Adjust(target_position)) override;
private:

private:
    /*!
     * \brief Defines how the incoming service method invocations are processed.
     *
     * \uptrace{SWS_CM_00198}
     * \uptrace{SWS_CM_00199}
     */
    void ProcessRequests();

    std::atomic<bool> m_finished{false};
    std::thread m_worker;

    ara::log::Logger& m_logger_ctx1{
        ara::log::CreateLogger("CTX1", "context for adjustment", ara::log::LogLevel::kVerbose)};
    ara::log::Logger& m_logger_ctx2{
        ara::log::CreateLogger("CTX2", "context for calibration", ara::log::LogLevel::kVerbose)};
    ara::log::Logger& m_logger_ctx5{ara::log::CreateLogger("CTX5", "context for echo", ara::log::LogLevel::kVerbose)};
};
```

3. `真实对象`
也是生成出来的，是许多操作的直接执行者
```cpp
class radarServiceAdapter : public ::ara::com::internal::dds::skeleton::ServiceImpl
{
public:
    using ServiceInterface = skeleton::radarSkeleton;
    using ServiceDescriptor = descriptors::internal::Service;

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

    virtual ~radarServiceAdapter()
    {
        StopOfferService();
        StopOfferField(UpdateRate);
        StopOfferMethod(Adjust);
        StopOfferEvent(brakeEvent);
        StopOfferEvent(parkingBrakeEvent);
        Disconnect(dynamic_cast<ServiceInterface&>(service_));
    }

private:
    ara::com::internal::dds::skeleton::EventImpl<
        EventbrakeEventTypeInfo,
        descriptors::brakeEvent>
        brakeEvent;
    ara::com::internal::dds::skeleton::EventImpl<
        EventparkingBrakeEventTypeInfo,
        descriptors::parkingBrakeEvent>
        parkingBrakeEvent;
    ara::com::internal::dds::skeleton::MethodImpl<
        MethodAdjust,
        descriptors::Adjust>
        Adjust;
    ara::com::internal::dds::skeleton::MutableFieldImpl<
        FieldUpdateRate>
        UpdateRate;

    void Connect(ServiceInterface& service)
    {
        service.AddDelegate(*this);
        service.brakeEvent.AddDelegate(brakeEvent);
        service.parkingBrakeEvent.AddDelegate(parkingBrakeEvent);
        Adjust.SetInvoker(MethodAdjust::CreateInvoker(dynamic_cast<ServiceInterface&>(service_)));
        service.UpdateRate.AddDelegate(UpdateRate);
    }

    void Disconnect(ServiceInterface& service)
    {
        service.RemoveDelegate(*this);
        service.brakeEvent.RemoveDelegate(brakeEvent);
        service.parkingBrakeEvent.RemoveDelegate(parkingBrakeEvent);
        Adjust.SetInvoker(nullptr);
        service.UpdateRate.RemoveDelegate(UpdateRate);
    }
};
```




# 用户层代码基本结构
创建`RadarActivity`对象（准确来说，是`ServiceInterface_name` + `Activity`）
## skeleton
1. 调用`init()`
```cpp
m_logger_ctx3.LogDebug() << "enter init()";

// Register
m_skeleton->UpdateRate.RegisterGetHandler(std::bind(&RadarActivity::getUpdateRate, this));

m_skeleton->UpdateRate.RegisterSetHandler(
    std::bind(&RadarActivity::setUpdateRate, this, std::placeholders::_1));
// Initialize Fields Values before Offering Service.
::ara::com::sample::RadarObjects_field UpdateRate_value;
//TODO:Need assign value
m_skeleton->UpdateRate.Update(UpdateRate_value);    // 直接开始持续发送数（是field的特性，有初值的设定）
//Offering
m_skeleton->OfferService(); // 这里有很多逻辑
```
！！！注意此处，`RadarActivity`所持有的调用对象`m_skeleton`是`radarSkeleton*`类型，实际指向的是`radarImp`类型的对象，也就是持有一个基类指针，指向派生类对象，并且`radarImp`类是skeleton侧`method的实现类`，是method专属的
而`radarSkeleton`类则是代表了skeleton侧的service interface的概念，持有除了method以外的event、field对象


2. 在`while`循环中调用`act()`
```cpp
void RadarActivity::act()
{
    m_logger_ctx3.LogInfo() << "radar active";
    static uint8_t i = 0;

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
        if (send_result) {
            m_logger_ctx3.LogInfo() << "brakeEvent sent";
        } else {
            m_logger_ctx3.LogError() << "brakeEvent.Send failed with error: " << send_result.Error();
        }
    }

//parkingBrakeEvent sample
    auto parkingBrakeEvent_allocation_result = m_skeleton->parkingBrakeEvent.Allocate();
    if(!parkingBrakeEvent_allocation_result){
        m_logger_ctx3.LogError() << "parkingBrakeEvent allocation failed with error: " << parkingBrakeEvent_allocation_result.Error();
    }else{
        auto parkingBrakeEventSample = std::move(parkingBrakeEvent_allocation_result).Value();
        // set value:
        // parkingBrakeEventSample ->object = value

        // send sample
        auto send_result = m_skeleton->parkingBrakeEvent.Send(std::move(parkingBrakeEventSample));
        if (send_result) {
            m_logger_ctx3.LogInfo() << "parkingBrakeEvent sent";
        } else {
            m_logger_ctx3.LogError() << "parkingBrakeEvent.Send failed with error: " << send_result.Error();
        }
    }
  
// UpdateRate send notification
    //TODO:Need assign value
    ::ara::com::sample::RadarObjects_field UpdateRate_value;
    // UpdateRate send notification
    //TODO:Need assign value
    auto UpdateRate_result = m_skeleton->UpdateRate.Update(UpdateRate_value);
    if (UpdateRate_result) {
        m_logger_ctx3.LogInfo() << "UpdateRate notify ";
    } else {
        m_logger_ctx3.LogError() << "UpdateRate notify failed with error: " << UpdateRate_result.Error();
    }
    i++;
}
```

以上方法调用的对象都是`RadarActivity`对象，用户层直接接触
## skeleton侧的`RadarActivity`
```cpp
class RadarActivity
{
public: // 以下函数的实现都在用户层
    RadarActivity();    // ！！！构造函数中，new了一个radarImp对象，赋给m_skeleton（基类指针指向派生类对象）
    ~RadarActivity();

    /*!
     *  \brief Initializes radar activity.
     *
     *  Initializes radar activity. This is called during initialization of the runtime.
     */
    void init();

    /*!
     *  \brief Runs radar activity.
     *
     *  Executable unit triggered to perform radar activity.
     */
    void act();

protected:

ara::core::Future<ara::com::sample::skeleton::fields::UpdateRate::value_type> getUpdateRate();

ara::core::Future<::ara::com::sample::RadarObjects_field> setUpdateRate(::ara::com::sample::RadarObjects_field field);

    /*!
     * \brief A pointer to the skeleton object.
     */
    ara::com::sample::skeleton::radarSkeleton* m_skeleton;  // ！！！持有radarSkeleton*类型的指针

    enum class internalStates
    {
        READY,
        NOT_READY
    };
    internalStates m_internal_state_for_update_rate_set_handler = internalStates::READY;    // 判断field的set handler的状态

    ara::log::Logger& m_logger_ctx3{
        ara::log::CreateLogger("CTX3", "context for update rate", ara::log::LogLevel::kVerbose)};
    ara::log::Logger& m_logger_ctx4{
        ara::log::CreateLogger("CTX4", "radar activity main context", ara::log::LogLevel::kVerbose)};
};
```


## `radarImp`
```cpp
class radarImp : public ara::com::sample::skeleton::radarSkeleton
{
    using Skeleton = ara::com::sample::skeleton::radarSkeleton;

public:
    radarImp(ara::core::InstanceSpecifier instanceSpec, ara::com::MethodCallProcessingMode mode)
        : Skeleton(std::move(instanceSpec), mode)   // 调用基类构造函数
        , m_worker(&radarImp::ProcessRequests, this)
    { } // 构造时启了一个method专用的线程

    virtual ~radarImp()
    {
        m_finished = true;
        m_worker.join();    // 主线程会调用该析构，然后调用join()，等待m_worker结束
    }

    virtual auto Adjust(
        const ::ara::com::sample::Position& target_position
        ) -> decltype(Skeleton::Adjust(target_position)) override;
private:

private:
    /*!
     * \brief Defines how the incoming service method invocations are processed.
     *
     * \uptrace{SWS_CM_00198}
     * \uptrace{SWS_CM_00199}
     */
    void ProcessRequests();

    std::atomic<bool> m_finished{false};
    std::thread m_worker;

    ara::log::Logger& m_logger_ctx1{
        ara::log::CreateLogger("CTX1", "context for adjustment", ara::log::LogLevel::kVerbose)};
    ara::log::Logger& m_logger_ctx2{
        ara::log::CreateLogger("CTX2", "context for calibration", ara::log::LogLevel::kVerbose)};
    ara::log::Logger& m_logger_ctx5{ara::log::CreateLogger("CTX5", "context for echo", ara::log::LogLevel::kVerbose)};
};
```



## proxy
1. 调用`RadarActivity::init()`
```cpp
void RadarActivity::init()
{
    m_logger.LogInfo() << "init() enter";

    ara::core::InstanceSpecifier portSpecifier{"fusion/fusion/radar_RPort"};
    m_logger.LogInfo() << "Port In Executable Ref:" << portSpecifier.ToString();

    auto instanceIDs = ara::com::runtime::ResolveInstanceIDs(portSpecifier);    // 调用rapidjson接口，解析Json，搜索<portSpecifier, instance_id>，此处是：<fusion/fusion/radar_RPort, DDS:19>
    if (instanceIDs.empty()) {
        throw std::runtime_error{"No InstanceIdentifiers resolved from provided InstanceSpecifier"};
    }
    m_logger.LogInfo() << "Searching for Service Instance:" << instanceIDs[0].ToString();

    auto res = Proxy::StartFindService(
        [this](ara::com::ServiceHandleContainer<Proxy::HandleType> handles, ara::com::FindServiceHandle handler) {
            RadarActivity::serviceAvailabilityCallback(std::move(handles), handler);
        },
        portSpecifier); // 注册了一个找到service时调用的函数
    if (!res) {
        m_logger.LogError() << "StartFindService failed with error: " << res.Error();
        throw std::runtime_error{"StartFindService failed"};
    }

    m_logger.LogInfo() << "init() exit";
}
```
！！！注意此处，proxy侧的`RadarActivity`类，持有的`radarProxy`指针，是在`FindService`过程中构造出来的

## proxy侧的`RadarActivity`类
```cpp
/*!
 *  \brief Class implementing radar activity.
 *
 *  Radar activity implementing function of data radar.
 */
class RadarActivity
{
    using Proxy = ara::com::sample::proxy::radarProxy;  // ！！！使用到了radarProxy，是组合关系，fusion端的RadarActivity持有radarProxy类型的指针

public:
    RadarActivity();

    /*!
     *  \brief Initializes radar activity.
     *
     *  Initializes radar activity. This is called during initialization of the runtime.
     */
    void init();

    /*!
     *  \brief Runs radar activity.
     *
     *  Executable unit triggered to perform radar activity.
     */
    void act();

    /*!
     *  \brief Callback to change to radar service offer changes.
     *
     *  Callback executed whenever a change for radar service offers happen.
     *
     */

    void serviceAvailabilityCallback(ara::com::ServiceHandleContainer<Proxy::HandleType> handles,
        ara::com::FindServiceHandle handler)
    {
        for (auto it : handles) {
            m_logger.LogInfo() << "Instance " << it.GetInstanceId().ToString() << " is available";
        }
        if (handles.size() > 0) {
            std::lock_guard<std::mutex> lock(m_proxy_mutex);
            if (nullptr == m_proxy) {
                m_proxy = std::make_shared<Proxy>(handles[0]);  // !!! 服务发现完成时，才构造radarProxy对象
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
    /*!
     * \brief Callback Received when the Field UpdateRate is changed.
     *
     * Callback Received when the Field UpdateRate is changed.
     */
    void UpdateRateReceived()
    {   // 该函数作为field对象的SetReceiveHandler函数的参数
        ara::log::LogStream logMsg{m_logger.LogVerbose()};

        m_proxy->UpdateRate.GetNewSamples(
            [&logMsg](auto sample) {
                logMsg << FIELDS_HEADER << "Callback: UpdateRate Field ";
            },
            1);
    }
protected:
    std::shared_ptr<Proxy> m_proxy;
    std::mutex m_proxy_mutex;
    std::uint32_t m_act_count;
    std::random_device m_rd;

    /*!
     * Subscribe to UpdateRate Field.
     */
    void UpdateRateSubscription();

    /*!
     * Testing Field Getter.
     */
    void fieldGetter();

    /*!
     * Testing Field Setter.
     */
    void fieldSetter();

    // this class own logger
    // this class own logger
    ara::log::Logger& m_logger{
        ara::log::CreateLogger("FACT", "RadarActivity class own context", ara::log::LogLevel::kVerbose)};
};
```


2. 在`while`循环中调用`act()`
```cpp
void RadarActivity::act()
{
    m_logger.LogInfo() << "radar alive";

    if (nullptr != m_proxy) {
        if(m_proxy->brakeEvent.IsSubscribed()){ 
            auto e2eState = ara::com::e2e::internal::GetSMState(m_proxy->brakeEvent);
            bool stateResult = (e2eState == ara::com::e2e::SMState::kNoData);
            // LogStream logMsg{m_logger.LogVerbose()};
            // logMsg << "brakeEvent E2E state:" << (stateResult ? "ok (NoData)" : "not ok");
            // logMsg.Flush();  
            
            auto callback = [&](auto sample) {
            m_logger.LogInfo() << "recive brakeEvent message";
            // logMsg << "recive brakeEvent message";

            // logMsg.Flush();    
            };
            // execute callback for every samples in the context of GetNewSamples
            m_proxy->brakeEvent.GetNewSamples(callback);    // 得到event数据，此时早已订阅        
        }
        else{
            m_logger.LogInfo() << "not subscribed to brakeEvent yet";
            // subscribe to event
            auto brakeEvent_subscription_result = m_proxy->brakeEvent.Subscribe(3); // 设置接收端 cache size为3
            if (brakeEvent_subscription_result.HasValue()) {
                m_logger.LogInfo() << "brakeEvent Callback registered.";
            } else {
                m_logger.LogError() << "brakeEvent Subscription failed with error: " << brakeEvent_subscription_result.Error();
                return;
            }
            m_logger.LogInfo() << "brakeEvent subscription complete";
        }
        if(m_proxy->parkingBrakeEvent.IsSubscribed()){
            auto e2eState = ara::com::e2e::internal::GetSMState(m_proxy->parkingBrakeEvent);
            bool stateResult = (e2eState == ara::com::e2e::SMState::kNoData);
            // LogStream logMsg{m_logger.LogVerbose()};
            // logMsg << "parkingBrakeEvent E2E state:" << (stateResult ? "ok (NoData)" : "not ok");
            // logMsg.Flush();  
            
            auto callback = [&](auto sample) {
            m_logger.LogInfo() << "recive parkingBrakeEvent message";
            // logMsg << "recive parkingBrakeEvent message";

            // logMsg.Flush();    
            };
            // execute callback for every samples in the context of GetNewSamples
            m_proxy->parkingBrakeEvent.GetNewSamples(callback);            
        }
        else{
            m_logger.LogInfo() << "not subscribed to parkingBrakeEvent yet";
            // subscribe to event
            auto parkingBrakeEvent_subscription_result = m_proxy->parkingBrakeEvent.Subscribe(3);
            if (parkingBrakeEvent_subscription_result.HasValue()) {
                m_logger.LogInfo() << "parkingBrakeEvent Callback registered.";
            } else {
                m_logger.LogError() << "parkingBrakeEvent Subscription failed with error: " << parkingBrakeEvent_subscription_result.Error();
                return;
            }
            m_logger.LogInfo() << "parkingBrakeEvent subscription complete";
        }

        m_logger.LogInfo() << "Subscribe to UpdateRate Field";
        this->UpdateRateSubscription();

        // "Testing Field Getter.";
        this->fieldGetter();

        // "Testing Field Setter.";
        this->fieldSetter();

        ::ara::com::sample::Position target_position;

        // proxy method call
        auto getUpdateRateFuture = m_proxy->Adjust(target_position);    // 发送请求，实现RPC，不需要订阅

        auto res = getUpdateRateFuture.get();

        m_logger.LogInfo() << "method call result is : "
            << "if sucess: " << res.success
            << "position" << res.effective_position.x << res.effective_position.y << res.effective_position.z;

    }
    m_act_count++;
}

```
## `radarProxy`
持有对proxy端的`event、method、field`对象的引用（属于基类引用指向派生类对象，派生类对象实际是`MethodImpl`等类型）
构造函数中，得到`ProxyFactoryWrapper<ProxyBinding>`类型的对象，从而构造基类`ProxyBase`
调用了基类`ProxyBase<radarProxyBase>`对象的成员指针`proxy_base_`提供的各种方法来将自己持有的各种引用初始化
而`proxy_base_`实际指向的对象是`ara::com::sample::radar_binding::dds::radarImpl`类型

```cpp
/// @uptrace{SWS_CM_00004, bedfd6edb7e40a5e31639e01621800cfc077b498}
class radarProxy
    : public ara::com::internal::proxy::ProxyBase<radarProxyBase>
    , public ara::com::sample::radar
{
public:
    /// @brief Proxy constructor.
    ///
    /// @uptrace{SWS_CM_00131, a4bb0196ec1ba17aed6d599f4dc3259f7b7ed02b}
    explicit radarProxy(const HandleType& proxy_base_factory)
        : ara::com::internal::proxy::ProxyBase<radarProxyBase>(proxy_base_factory)
        , Adjust(proxy_base_->GetAdjust())
        , UpdateRate(proxy_base_->GetUpdateRate())
        , brakeEvent(proxy_base_->GetbrakeEvent())
        , parkingBrakeEvent(proxy_base_->GetparkingBrakeEvent())
    { }

    /// @brief Proxy shall be move constructable.
    ///
    /// @uptrace{SWS_CM_00137, c198eaaf815fde7c1ebe1f7c31a75bf1ccc57f19}
    explicit radarProxy(radarProxy&&) = default;

    /// @brief Proxy shall be move assignable.
    ///
    /// @uptrace{SWS_CM_00137, c198eaaf815fde7c1ebe1f7c31a75bf1ccc57f19}
    radarProxy& operator=(radarProxy&&) = default;

    /// @brief Proxy shall not be copy constructable.
    ///
    /// @uptrace{SWS_CM_00136, 2565b350c03a33dff1af61b8fc65aacbccb7ea9a}
    explicit radarProxy(const radarProxy&) = delete;

    /// @brief Proxy shall not be copy assignable.
    ///
    /// @uptrace{SWS_CM_00136, 2565b350c03a33dff1af61b8fc65aacbccb7ea9a}
    radarProxy& operator=(const radarProxy&) = delete;

    /// @uptrace{SWS_CM_00196, 2bdfebdbb1957ea558963073c804f27d13405c80}
    methods::Adjust& Adjust;
    fields::UpdateRate& UpdateRate;
    events::brakeEvent& brakeEvent;
    events::parkingBrakeEvent& parkingBrakeEvent;
};
```


# 1 skeleton端初始化
```cpp
m_skeleton = new radarImp(instanceSpec, ara::com::MethodCallProcessingMode::kPoll); // ！！！重要的初始化
```
在`radarImp`的初始化过程中，
初始化了基类对象`radarSkeleton`，这里涉及到很多的构造：
```cpp
/// @uptrace{SWS_CM_00002, 4d3a2b51c78573d1d34cd3d2aff93527b4a97d63}
class radarSkeleton
    : public ara::com::sample::radar    // radar类记录了service相关信息：id、version、AdjustOutput结构体
    , public ara::com::internal::skeleton::TypedServiceImplBase<radarSkeleton>
{
    /// @uptrace{SWS_CM_00152, b772a2873b407e2731b5b05529fb74cda8bce3df}
    radarSkeleton(ara::core::InstanceSpecifier instanceSpec,
        ara::com::MethodCallProcessingMode mode = ara::com::MethodCallProcessingMode::kEvent)
        : ara::com::internal::skeleton::TypedServiceImplBase<radarSkeleton>(std::move(instanceSpec), mode)
    { }

    /// @uptrace{SWS_CM_00191, e3ee8aca56f9a3b371df808324769584093419f3}
    using ara::com::sample::radar::AdjustOutput;
    virtual ara::core::Future<AdjustOutput> Adjust(const ::ara::com::sample::Position& target_position) = 0;
}
// ！！！这里还涉及到radarSkeleton对象的成员的初始化：
    fields::UpdateRate UpdateRate;
    events::brakeEvent brakeEvent;
    events::parkingBrakeEvent parkingBrakeEvent;
```

`radarSkeleton`的几个重要成员（涉及field、event）
1. field
```cpp
/// @uptrace{SWS_CM_00007, 42d730a95751bac3ee6019e706f92c406b2499b8}
using UpdateRate = ara::com::internal::skeleton::MutableFieldDispatcher<::ara::com::sample::RadarObjects_field>;
```

2. event
```cpp
using brakeEvent = ara::com::internal::skeleton::EventDispatcher<::ara::com::sample::RadarObjects>;
```
！！！重要特点：这里的`EventDispatcher类`继承的`DispatcherBase`所持有的主体都是`list<EventImpl>`，对这个list的填充是在调用`OfferService`时，对`Adapter类`初始化时才做的

PS：注意到，`MethodCallProcessingMode`默认为`kEvent`
再看一下`radarSkeleton`的重要基类：`TypedServiceImplBase`的构造
```cpp
// Defines the methods for all skeletons that require knowledge about the service type that they offer.
TypedServiceImplBase(ara::com::InstanceIdentifier instance, ara::com::MethodCallProcessingMode mode)
    : ServiceImplBase(instance)
{
    SetMethodCallProcessingMode(mode);
    // 调用到基类ServiceImplBase的同名函数：
}
        基类ServiceImplBase的SetMethodCallProcessingMode：
        virtual void SetMethodCallProcessingMode(ara::com::MethodCallProcessingMode mode) override
        {
            mode_ = mode;   // 其实就是kEvent
            for (ServiceBase* delegate : Dispatcher::delegates_) {
                delegate->SetMethodCallProcessingMode(mode);    // 尚未OfferService时，这里是空的
            }
        }

// 基类ServiceImplBase构造函数（Base class for all generated skeleton classes.）
ServiceImplBase(ara::core::InstanceSpecifier instanceSpec)
    : ServiceImplBase(::ara::com::runtime::ResolveInstanceIDs(instanceSpec))
{ } // 委托构造

    // 委托构造调用的构造函数
    ServiceImplBase(ara::com::InstanceIdentifierContainer instanceIDs)
        : instanceIDs_(std::move(instanceIDs))
        , it_(Dispatcher::delegates_.end())
    {
        // Passing an empty container is treated as a Violation
        // unless otherwise stated in SWS
        if (instanceIDs_.empty()) {
            throw std::invalid_argument("Unable to create service, instance identifier container is empty");
        }
    }


    // ServiceImplBase的两个基类：
    DispatcherBase<ServiceBase>、ServiceBase
！DispatcherBase：
server端的所有dispatcher的基类，每个dispatcher为所有clients（即delegate）分配method调用，该基类提供对外接口，用于添加或删除代理
本质上是list<ServiceBase>

！ServiceBase：
定义skeleton实现的接口（全是纯虚函数），除了声明在其中的method，生成出的从属类也添加了纯虚函数（必须在派生类中根据模型的定义来实现）
```



`radar_activity.h`的`Init()`中的`OfferService()`，外层调用是这样的：
`radarImp`对象调用了基类`radarSkeleton`的方法！！！
```cpp
    void OfferService()
    {
        if (!UpdateRate.IsInitialized()) {
            throw ara::com::IllegalStateException(
                "Attempt to offer service \"/apd/serviceinterfaces/radar\" with uninitialized field \"UpdateRate\"");
        }

        ara::com::internal::skeleton::TypedServiceImplBase<radarSkeleton>::OfferService();  // 调用下一级基类方法
    }
```
调用了基类`TypedServiceImplBase<radarSkeleton>`的`OfferService`
（！！！`radarSkeleton`是`TypedServiceImplBase`的“派生类模板”，相当于是一种特化的Base类，比单纯继承更灵活）
```cpp
// TypedServiceImplBase
    /**
     * \brief Offer service to all bindings that are able to offer the service.
     *
     * It is OK to call this method repeatedly.
     *
     * \uptrace{SWS_CM_00101, a4b7416191b6d3795f020d8c96b607408cbe6d3f}
     */
    void OfferService()
    {
        Runtime::OfferService(static_cast<Service&>(*this));    // *this已经是指向radarImp对象了
    }
```
！！！`TypedServiceImplBase<radarSkeleton>`这层直接调用`Runtime`提供的`static`方法：`OfferService`，注意到其中做了`static_cast`，将当前`radarImp`指针类型强转为`radarSkeleton&`类型（派生类指针转为基类指针），并传入了radarImp对象作为入参
```cpp
// Runtime，Service是src-gen的radarSkeleton类型
    template <typename Service>
    /**
     * @brief Offer the service instance.
     *
     * method is idempotent - could be called repeatedly.
     * @param service service to offer
     **/
    inline static void OfferService(Service& service)
    {
        static_assert(std::is_base_of<internal::skeleton::ServiceBase, Service>::value,
            "OfferService only accepts classes derived from generated service skeletons");  // radarSkeleton必须是ServiceBase的后代
        Init();     // 调用ara::com::internal::runtime的Initialize（只调用一次），！！！其实现在src-gen生成出的ara_com_main-radar.cpp中，根据绑定的协议，调用各自的Register()
        for (const auto& instanceId : service.GetInstanceIDs()) {   // instance id为"DDS:19"
            OfferService(service, Service::service_id, instanceId);
        }
    }
```

在src-gen生成出的`ara_com_main-radar.cpp`中，根据绑定的协议，调用各自的`Register()`，本质上就是：
解析配置，得到`DdsProxyFactoryImpl`对象和`DdsServiceRegistryImpl`对象（这两个对象都有一个manifest_成员，存储着`domainProvidedServiceProperties`），构造`vector<ProxyFactoryBuilder>`、`vector<ServiceRegistry>`
```cpp
void Register()
{
    static runtime::ServiceInstanceManifest manifest;

    if (!manifest.parseJsonFile("./etc/service_instance_manifest.json")) {
        common::logger().LogError() << "runtime::Register(): Failed service instance manifest:";
    }

    static runtime::DdsProxyFactoryImpl proxyFactory(manifest);
    internal::runtime::ProxyFactoryBuilderList& proxyFactoryBuilderList
        = internal::runtime::ProxyFactoryBuilderList::GetInstance();
    proxyFactoryBuilderList.push_back(proxyFactory);

    static runtime::DdsServiceRegistryImpl serviceRegistry(manifest);
    internal::runtime::ServiceRegistryList& serviceRegistryList = internal::runtime::ServiceRegistryList::GetInstance();
    serviceRegistryList.push_back(serviceRegistry);
}
```


然后调用到`Runtime`提供的`static`的`OfferService`：
```cpp
void Runtime::OfferService(internal::skeleton::ServiceBase& service,
    internal::ServiceId service_id,
    ara::com::InstanceIdentifier instance_id)   // 入参：radarImp类型，62303，"DDS:19"
{
    bool serviceRegistered{false};
    // 遍历存储service instance的vector
    for (internal::runtime::ServiceRegistry& registry : internal::runtime::ServiceRegistryList::GetInstance()) {
        serviceRegistered |= registry.registerService(service, service_id, instance_id);
    }   // ！！！ServiceRegistry对象根据service instance信息调用registerService()方法

    if (!serviceRegistered) {
        std::stringstream s;
        s << "Cannot register service " << service_id << " with instance identifier " << instance_id.ToString();
        throw std::runtime_error(s.str());
    }
}
```
`ServiceRegistry`就是用来做`Network Binding`

`registerService`的实现：
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
        });     // registered_objects_是std::unordered_map<internal::skeleton::ServiceBase*, RegisteredObject>
        // 匹配已注册的对象，防止二次注册
    if (registered_object != registered_objects_.end()) {
        // The object was re-registered, so only check if this was done consistently
        common::logger().LogError()
            << "DdsServiceRegistryImpl::registerService(): Registering the same service, service_id: " << service_id
            << ", instance_id:" << instance_id.ToString() << "!";
        return false;
    }

    const DdsServiceMapping::Mapping* mapping = DdsServiceMapping::GetMappingForServiceId(service_id);  //拿到ServiceMappingImpl<ara::com::sample::radar::service_id,NoProxy,ara::com::sample::radar_binding::dds::radarServiceAdapter>类型的ara__com__sample__radar__mapping对象（在src-gen中），匹配静态常量service id（同样生成在src-gen）
    if (!mapping) {
        common::logger().LogError()
            << "DdsServiceRegistryImpl::registerService(): there is no mapping for service with id " << service_id;
        return false;
    }

    auto instanceProperties = manifest_.getPrividedInstanceProperties(instance_id);
    if (!instanceProperties) {
        common::logger().LogError()
            << "DdsServiceRegistryImpl::registerService(): there is no manifest for instance with id "
            << instance_id.ToString();
        return false;
    }

    auto participant
        = DomainParticipant::get(instanceProperties.Value()->domainId_, instanceProperties.Value()->qosProfile_);   // // 创建participant，得到domain id和qos profile
    if (!participant) {
        common::logger().LogError() << "DdsServiceRegistryImpl::registerService(): participant is nil";
        return false;
    }

    auto deployment_id = mapping->GetDeploymentId();    // 是"Radar"  来自建模信息，存储在service_desc_radar.h，并且取得这个信息是来自ara__com__sample__radar__mapping，该对象的类型的第三个模板参数是radarServiceAdapter，注意到radarServiceAdapter对象在这里只是类型参数，不需要构造
    common::ServiceInfo info{deployment_id, instance_id};   // "Radar","DDS:19"

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
        = mapping->GetSkeleton(service, instance_id, {participant, publisher, subscriber});     // ！！！重要调用，生成了Adapter对象
    if (!adapter) {
        common::logger().LogError() << "DdsServiceRegistryImpl::registerService(): adapter is nil";
        return false;
    }

    registered_objects_.emplace(&service, RegisteredObject{std::move(adapter), service_id, instance_id});   // map增加一个元素，就等于注册好了，实际存储的是<radarImp类型对象, RegisteredObject对象>，然后RegisteredObject的第一个成员是指向radarServiceAdapter类型对象的指针
    return true;
}
这里的 radarImp 和 radarServiceAdapter 都是ServiceBase的后代，但是意义却不同！！！
    RegisteredObject的定义：
    struct RegisteredObject
    {
        std::unique_ptr<internal::skeleton::ServiceBase> service_;
        ServiceId service_id_;
        ara::com::InstanceIdentifier instance_id_;
    };
```
注意到：
`radarServiceAdapter`继承自`ServiceImpl`，负责skeleton端的service的真正实现
`radarServiceAdapter`的构造需要有`radarImp`对象作为入参

`GetSkeleton`函数：
```cpp
    std::unique_ptr<internal::skeleton::ServiceBase> GetSkeleton(internal::skeleton::ServiceBase& service,
        types::InstanceId instance_id,
        common::HandleInfo handle) const override
    {
        return AdapterBuilder<SkeletonType>().make(service, instance_id, handle);
    }   // SkeletonType在src-gen里传入
// 返回值是ServiceBase类型的指针
        此处调用的make（是为了捕获dynamic_cast时可能产生的异常）：
            std::unique_ptr<Adapter> make(internal::skeleton::ServiceBase& service,
            types::InstanceId instance_id,
            common::HandleInfo handle)
        {
            try {
                typename Adapter::ServiceInterface& interface = dynamic_cast<typename Adapter::ServiceInterface&>(service); // 把绑定到radarImp对象的引用，强转为radarSkeleton类型的引用（派生类指针转为基类指针）
                std::unique_ptr<Adapter> adapter = std::make_unique<Adapter>(interface, instance_id, handle);
                return adapter;
            } catch (std::bad_cast&) {
                throw IllegalStateException("Given service interface does not fit to the binding adapter!");
            }
        }

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
可以看出，`radarSkeleton`类就是代码层面的`service interface`（持有几个event、method、field）
Adapter其实也体现了`适配器模式`，接口之间做转换，形如另一种接口
`radarSkeleton`继承自ara::com::sample::radar和ara::com::internal::skeleton::TypedServiceImplBase<radarSkeleton>类
其中的TypedServiceImplBase<radarSkeleton>继承自`ServiceImplBase`类

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

// 持有以下关键对象
    ara::com::internal::dds::skeleton::EventImpl<
        EventbrakeEventTypeInfo,
        descriptors::brakeEvent>
        brakeEvent;
    ara::com::internal::dds::skeleton::EventImpl<
        EventparkingBrakeEventTypeInfo,
        descriptors::parkingBrakeEvent>
        parkingBrakeEvent;
    ara::com::internal::dds::skeleton::MethodImpl<
        MethodAdjust,
        descriptors::Adjust>
        Adjust;
    ara::com::internal::dds::skeleton::MutableFieldImpl<
        FieldUpdateRate>
        UpdateRate;

```

// 重点看下其中构造函数执行了什么操作
## 1. Connect(service)
```cpp
    void Connect(ServiceInterface& service)
    {
        service.AddDelegate(*this);
        /* 
            ！！！为radarImp对象，增加代理：即当前的radarServiceAdapter对象
            DispatcherBase<ServiceBase>提供了AddDelegate()，ServiceImplBase继承了DispatcherBase，DispatcherBase有list<ServiceBase*>，也就是说，！！！radarServiceAdapter代理了radarImp（虽然是实现类），真正承载了service的功能
        */ 
        service.brakeEvent.AddDelegate(brakeEvent);     // 同样地，EventImpl代理了EventDispatcher
        /* 
            service.brakeEvent是radarSkeleton类的成员，是ara::com::internal::skeleton::EventDispatcher<::ara::com::sample::RadarObjects>类型
            RadarObjects是brakeEvent所用的数据类型
            入参brakeEvent是radarServiceAdapter类的成员之一，是EventImpl模板类的实例化
        */
        service.parkingBrakeEvent.AddDelegate(parkingBrakeEvent);
        Adjust.SetInvoker(MethodAdjust::CreateInvoker(dynamic_cast<ServiceInterface&>(service_)));  // service_是ServiceImpl的成员变量，是ServiceBase类型的引用，绑定的是radarImp对象
        /*
            Adjust是radarServiceAdapter类的成员（代表method）
            提供了SetInvoker()方法用于接收一个可调用对象：CreateInvoker内部返回的匿名函数
        */
        PS: CreateInvoker的函数体
        template <class T>
        static auto CreateInvoker(T& service)   // service实际上是radarImp类型
        {
            return [&](MethodAdjust::request_t const& request) {    // 入参是模型定义产生的dds产生的入参类型
                auto input = request.data().Adjust();   // 得到引用
                    ::ara::com::sample::Position result_target_position;
                    ara::com::internal::dds::ConvertFromIdl(input.target_position(),result_target_position);    // 数据转换
                return service.Adjust(  // 调用用户层传入的radarImp类的Adjust方法
                    result_target_position
                );
            };
        }

        service.UpdateRate.AddDelegate(UpdateRate);
        // UpdateRate是radarSkeleton类的成员变量，是ara::com::internal::skeleton::MutableFieldDispatcher<::ara::com::sample::RadarObjects_field>类型

    }
// ConvertFromIdl意义在于，将IDL生成的数据类，转换成c++中可操作的结构体
```

对于method而言，调用`AddDelegate`时，会调用`SetMethodCallProcessingMode(mode_)`
代码模板默认生成：
```cpp
m_skeleton = new radarImp(instanceSpec, ara::com::MethodCallProcessingMode::kPoll);
```

`OfferService`中，构造`radarServiceAdapter`对象，最后构造`ServiceImpl`，该类的构造函数设定`requestProcessingMode_`的初值为`MethodCallProcessingMode::kPoll`
可见，skeleton端处理request默认采用`kPoll`

1. `requestProcessingMode_`若为`kPoll`
    若入参不是`kPoll`
        就调用用户层传入的函数处理请求（先处理剩余的请求）
        是`kEventSingleThread`
        是`kEvent`
            启动专用线程，处理请求

2. `requestProcessingMode_`若为`kEvent`
    若入参不是`kEvent`（才有切换的必要），即是`kEventSingleThread`、或`kPoll`
    调用`StopWorkers();`

```cpp
void ServiceImpl::SetMethodCallProcessingMode(ara::com::MethodCallProcessingMode mode)
{
    common::logger().LogInfo() << "SetMethodCallProcessingMode";
    std::lock_guard<std::mutex> guard(requestsMutex_);
    // execute remaining requests in case we're switching away from kPoll
    if ((requestProcessingMode_ == MethodCallProcessingMode::kPoll) && (mode != MethodCallProcessingMode::kPoll)) {
        CollectPendingRequests();
        request_t request;
        while (pendingRequests_.try_pop(request)) {
            request.second->ProcessRequest(request.first);
        }

        if (mode == MethodCallProcessingMode::kEvent) {
            common::logger().LogInfo() << "SetMethodCallProcessingMode to kEvent";
            StartWorkers();
        } else {
            common::logger().LogInfo() << "SetMethodCallProcessingMode to kEventSingleThread";
        }
    } else if ((requestProcessingMode_ == MethodCallProcessingMode::kEvent)
        && (mode != MethodCallProcessingMode::kEvent)) {
        StopWorkers();
        if (mode == MethodCallProcessingMode::kEvent) { // Bug: 无意义的判断
            common::logger().LogInfo() << "SetMethodCallProcessingMode to kEvent";
        } else {
            common::logger().LogInfo() << "SetMethodCallProcessingMode to kPoll";
        }
    }

    requestProcessingMode_ = mode;
}
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
        if (!handle.participant || !handle.publisher) { // 判断是否在之前实例化，registerService时应该创建了对象（其实当前还在registerService，这个函数的调用栈很深）
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
        = handle.participant->GetTopic<typename dds_type_construct::reply>(MethodDescriptor::output_topic_name);    // output_topic_name    PS：此处的topic_name都记录在src-gen的AP_Project/event_method_field/apd/RadarFusionMachine/src/radar/net-bindings/fastdds/ara/com/sample/service_desc_radar.h中，此处的topic_name是DdsTopicRadarMethodRequest，是模型输入的
    
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
真正在DDS层面提供服务
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
PS：以下回调函数的传入看似很复杂，核心就是延迟调用，`serviceAvailabilityCallback`一直要到服务发现完成才调用

先看用户层面的部分，线程函数中调用了`radar_activity.cpp`中定义的`init()`函数，
## 关键调用：`StartFindService`：
参数1：lambda函数：接收1个handles，1个handler，函数体是用户层实现的`serviceAvailabilityCallback`，接收两个入参
参数2：portSpecifier：`"fusion/fusion/radar_RPort"`
此处调用了`ara::com::sample::proxy::radarProxy`类所继承的`ara::com::internal::proxy::ProxyBase<radarProxyBase>`的`static`的`StartFindService`函数
```cpp
    auto res = Proxy::StartFindService(
        [this](ara::com::ServiceHandleContainer<Proxy::HandleType> handles, ara::com::FindServiceHandle handler) {
            RadarActivity::serviceAvailabilityCallback(std::move(handles), handler);
        },
        portSpecifier); // 参数1：callback  参数2：portSpecifier
```

其中的callback函数：`serviceAvailabilityCallback`
```cpp
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
                m_proxy = std::make_shared<Proxy>(handles[0]);  // 实例化radarProxy对象！！！
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
```

其中关键调用`Proxy::StartFindService`是基类`ara::com::internal::proxy::ProxyBase<radarProxyBase>`的方法：
```cpp
template <typename ProxyBinding>
inline ara::core::Result<FindServiceHandle> ProxyBase<ProxyBinding>::StartFindService(
    FindServiceHandler<HandleType> handler,
    ara::core::InstanceSpecifier instance)
{
    auto instanceIDs = ::ara::com::runtime::ResolveInstanceIDs(instance);

    // According to SWS_CM_00623 InstanceSpecifier validation errors are treated as Violations
    if (instanceIDs.empty()) {
        throw std::invalid_argument("Unable to find service, instance identifier container is empty");
    }

    // Currently it's assumed that only one Service Instance mapped to the Port:
    // InstanceIdentifier Container contains only one element
    if (instanceIDs.size() != 1) {
        throw std::invalid_argument(
            "Implementation constraints: not able to handle multiple service instances mapped to r-port-prototype");
    }
    return StartFindService(std::move(handler), instanceIDs.front());
}
```
以上函数的重点就是拿到`instance id`
更深层调用是：当前所在类`ara::com::internal::proxy::ProxyBase<radarProxyBase>`的另一个重载函数（区别是第二个参数是`InstanceIdentifier`）
其重点是调用`Runtime`层的`StartFindServiceById`，拿到当前类的模板类型参数`ProxyBinding::service_id`，传入，然后拿到结果返回出去
PS: 当前的调用栈都是static函数，但是由于using的过程中，模板类的模板类型参数已经确定了
在`RadarActivity`中`using Proxy = ara::com::sample::proxy::radarProxy;`，而`ara::com::sample::proxy::radarProxy`继承自`ara::com::internal::proxy::ProxyBase<radarProxyBase>`，那么此处的`ProxyBinding`就是`radarProxyBase`，继承了`ara::com::sample::radar`，自然能拿到static的`ServiceId`（jinja模板生成）

```cpp
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
```

### `Runtime`层的`StartFindServiceById`：
参数：
1. service_id
2. instance id
3. lambda函数，入参为：
    1. `ServiceHandleContainer<std::shared_ptr<ProxyFactory>>`
    2. `FindServiceHandle`
    函数体为：
    遍历通过入参拿到的`ServiceHandleContainer`，将其一个个地`push_back`进临时创建的handles，再传入`handler`回调
    PS：此处的handler回调，即最上层用户层传入的函数体中含有`serviceAvailabilityCallback`的lambda，该lmabda什么时候调用要看`StartFindServiceById`的后续调用

Runtime层的`StartFindServiceById`：
```cpp
    inline static FindServiceHandle StartFindServiceById(internal::ServiceId service_id,
        ara::com::InstanceIdentifier instance_id,
        std::function<void(ServiceHandleContainer<std::shared_ptr<internal::proxy::ProxyFactory>>, FindServiceHandle)>
            handler)
    {
        Init(); // Runtime中的初始化，Initialize()，根据绑定的协议调用Register！！！这里开始的后续调用就确定了哪一种协议的调用栈
        return DoStartFindServiceById(service_id, instance_id, handler);
    }
```

#### `Init()`:
```cpp
void Runtime::Init()
{
    static std::once_flag initialized;
    std::call_once(initialized, internal::runtime::Initialize);
}
```

`internal::runtime::Initialize`
编译时传入宏定义`HAS_FASTDDS_BINDING`
```cpp
void Initialize()
{
#ifdef HAS_VSOMEIP_BINDING
    vsomeip::runtime::Register();
#endif  // HAS_VSOMEIP_BINDING

#ifdef HAS_WRSOMEIP_BINDING
    ara::com::internal::wrsomeip::runtime::Register();
#endif  // HAS_WRSOMEIP_BINDING

#ifdef HAS_OPENDDS_BINDING
    dds::runtime::Register();
#endif  // HAS_OPENDDS_BINDING

#ifdef HAS_FASTDDS_BINDING
    dds::runtime::Register();
#endif  // HAS_FASTDDS_BINDING

#ifdef HAS_RTESHM_BINDING
    rte::runtime::Register();
#endif 
}
```

`dds::runtime::Register();`
初始化全局的``ProxyFactoryBuilderList`，`ServiceRegistryList`
```cpp
void Register()
{
    static runtime::ServiceInstanceManifest manifest;

    if (!manifest.parseJsonFile("./etc/service_instance_manifest.json")) {
        common::logger().LogError() << "runtime::Register(): Failed service instance manifest:";
    }

    static runtime::DdsProxyFactoryImpl proxyFactory(manifest);
    internal::runtime::ProxyFactoryBuilderList& proxyFactoryBuilderList
        = internal::runtime::ProxyFactoryBuilderList::GetInstance();
    proxyFactoryBuilderList.push_back(proxyFactory);

    static runtime::DdsServiceRegistryImpl serviceRegistry(manifest);
    internal::runtime::ServiceRegistryList& serviceRegistryList = internal::runtime::ServiceRegistryList::GetInstance();
    serviceRegistryList.push_back(serviceRegistry);
}
```

！！！此处`Runtime`层的对象是如何和具体协议相关的对象产生联系的呢？是因为在`Runtime`的`Init()`中，有一串宏定义，决定执行：
```cpp
dds::runtime::Register();
```
而这个`Register()`是ara-api中的。

#### `DoStartFindServiceById`：
遍历`Runtime::Init`时初始化的`ProxyFactoryBuilderList`对象
为其中每一个元素调用`RegisterFindServiceHandle(handle, handler)`


```cpp
FindServiceHandle Runtime::DoStartFindServiceById(internal::ServiceId service_id,
    ara::com::InstanceIdentifier instance_id,
    std::function<void(ServiceHandleContainer<std::shared_ptr<ara::com::internal::proxy::ProxyFactory>>,
        FindServiceHandle)> handler)
{
    static std::atomic<std::uint32_t> uid;  // 表示什么的原子变量？？？
    FindServiceHandle handle{service_id, instance_id, uid.fetch_add(1)};
    for (internal::runtime::ProxyFactoryBuilder& factory : internal::runtime::ProxyFactoryBuilderList::GetInstance()) {
        factory.RegisterFindServiceHandle(handle, handler);
    }
    return handle;
}
```

`FindServiceHandle`
标识一个被触发的`FindService请求`，只是个标识：记录`service_id`、`instance_id`
```cpp
/**
 * @brief Identifier for a triggered FindService request.
 *
 * This identifier is needed to later cancel the FindService request.
 *
 * @uptrace{SWS_CM_00303, 74731364104a5921eb782e7e2f78c97f58f6d07f}
 */
struct FindServiceHandle
{
    FindServiceHandle()
        : instance_id{InstanceIdentifier::MakeAny()}
    { }

    FindServiceHandle(internal::ServiceId srvc_id, ara::com::InstanceIdentifier instnc_id, std::uint32_t id)
        : service_id{srvc_id}
        , instance_id{instnc_id}
        , uid{id}
    { }

    internal::ServiceId service_id;
    ara::com::InstanceIdentifier instance_id;
    std::uint32_t uid;
};
```


关键调用：`RegisterFindServiceHandle()`
调用`GetMappingForServiceId`，根据`service_id`，拿到`ServiceMappingImpl`全局对象
```cpp
ServiceMappingImpl<ara::com::sample::radar::service_id, ara::com::sample::radar_binding::dds::radarImpl, NoAdapter>
```
关键：和`radarImpl`联系起来了
拿到`dds service配置：domain、qos、service id、instance_id`，调用`SubscribeOnParticipantUpdates`，拿到participant
然后调用`GenerateAvailableInstanceFactory`，得到`availableFactory`  ？？？有待深挖

！！！其中调用`OnServiceAvailabilityChanged`时，把service信息（包括回调函数）组织成`ServiceInfo`，会真正调用最外层用户层传入的函数体中含有`serviceAvailabilityCallback`的lambda

再把`availableFactory`存入`availableInstances`
调用`callback(availableInstances, handle);`

`RegisterFindServiceHandle`
```cpp
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

    if (handle.instance_id.IsAny()) {//instance_id   is "any"，表示需要接收所有service instance id的消息
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

        auto instanceProperties = manifest_.getRequiredInstanceProperties(handle.instance_id);  // 返回const DdsServiceInstanceProperties*类型，涉及一些建模时的dds service配置：domain、qos、service id、instance_id
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
            = findHandles_.emplace(handle.uid, common::FindServiceInfo{handle, callback, participant}); // Id, FindServiceInfo类型的map
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
                        return GetProxy(mapping, serviceId, instanceId, participant);  // ！！！ 调用ProxyBuilder类的make()，得到ProxyFactoryImpl对象
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

`OnServiceAvailabilityChanged()`
```cpp
void DdsProxyFactoryImpl::OnServiceAvailabilityChanged(std::set<common::ServiceInfo> const& availableServices,
    common::ServiceInfo service,
    bool available)
{
    common::logger().LogInfo() << "Service {" << service.serviceId << "}, instance {" << service.instanceId.ToString()
                               << "} become " << (available ? "available." : "unavailable.");

    DdsServiceMapping::MultiMapping mappings = DdsServiceMapping::GetMappingForFastDDSServiceId(service.serviceId);

    for (const DdsServiceMapping::MultiMapping::value_type& mappingKeyval : mappings) {
        ServiceId mappingId = mappingKeyval.first;
        const DdsServiceMapping::Mapping& mapping = mappingKeyval.second;
        // This is the loop that calls any handlers that registered for just one instance of a service
        for (auto const& info : findHandles_) {
            auto const& handle = info.second;
            if (handle.findHandle.service_id == mappingId && handle.findHandle.instance_id == service.instanceId) {
                const types::ServiceId deploymentId = mapping.GetDeploymentId();
                if (available) {
                    auto factory = GetProxy(mapping, deploymentId, service.instanceId, handle.participant);
                    if (factory) {
                        handle.callback({factory}, handle.findHandle);
                    } else {
                        common::logger().LogError()
                            << "DdsProxyFactoryImpl::OnServiceAvailabilityChanged(): proxy factory is nil";
                    }
                } else {
                    handle.callback({}, handle.findHandle);
                }
            }
        }

        // This loop calls handlers that registered for any instance of a given service.
        ServiceHandleContainer<std::shared_ptr<internal::proxy::ProxyFactory>> allInstances
            = GenerateAnyFactoryList(availableServices, mapping);
        for (auto const& info : findHandles_) {
            auto const& handle = info.second;
            if (handle.findHandle.service_id == mappingId && handle.findHandle.instance_id.IsAny()) {
                handle.callback(allInstances, handle.findHandle);
            }
        }
    }
}
```


有待深挖：
`ProxyBase`
```cpp
/**
 * \brief This is the base class that all proxies derive from.
 *
 * It provides all the methods that are common for service discovery. It also serves as the link from the user-visible
 * API to the binding. The template parameter designates the respective base class that has to provide getters for all
 * interface elements declared.
 *
 */
template <typename ProxyBinding>
class ProxyBase
{
public:
    /**
     * \brief Type of the handle that identifies one particular instance of the service.
     * \uptrace{SWS_CM_00312, 5375d8c7e5188912edeb968a872caf5bfa9991bd}
     *
     * The underlying type is lightweight and can therefore be freely copied and stored by the application developer.
     */
    using HandleType = ProxyFactoryWrapper<ProxyBinding>;

    /**
     * \brief Creates a proxy using the instance information from the given handle. See \ref FindService() and
     * \ref StartFindService() for means to obtain a handle.
     *
     * \param proxy Instance the proxy shall be connected to.
     *
     * \uptrace{SWS_CM_00131, 4dbc501350571efc37be84e68c998c95a73adbda}
     */
    ProxyBase(const HandleType& proxy)
        : proxy_base_(proxy.Create())
        , handle_(proxy)
    { }
    virtual ~ProxyBase()
    { }

    /**
     * \brief Returns the handle from which the Proxy instance has been created.
     *
     * \remark since the proxy may be "auto updated" during runtime, in case the (remote) service has been restarted
     * with a different transportlayer adressing, the handles returned at time t0 and t1 from the same proxy instance
     * need not be binary-equal!
     *
     * \return handle The handle that may be used to create another instance of a proxy for the same instance.
     *
     * \uptrace{SWS_CM_10383, 17f578b19b796bdb8dbafd9b287f6dbd0f013139}
     */
    HandleType GetHandle() const
    {
        return handle_;
    }

    /**
     *
     * \brief Retrieves a list of compatible instances for the service.
     *
     * Opposed to \ref StartFindService() this version is a "one-shot" find request, which is
     * - synchronous, i.e. it returns after the find has been done and a result list of matching service instances is
     *   available. (list may be empty, if no matching service instances currently exist)
     * - does reflect the availability at the time of the method call. No further (background) checks of availability
     *   are done.
     *
     * \param instance Instance of the service type that shall be searched/found. A wildcard may be given.
     *  Default value is wildcard.
     *
     * \uptrace{SWS_CM_00122, d485774f111eb58f6cfa351c02570b425400aca6}
     */
    static ara::core::Result<ServiceHandleContainer<HandleType>> FindService(
        InstanceIdentifier instance = InstanceIdentifier::MakeAny());

    /**
     *
     * \brief Retrieves a list of compatible instances for the service.
     *
     * Opposed to \ref StartFindService() this version is a "one-shot" find request, which is
     * - synchronous, i.e. it returns after the find has been done and a result list of matching service instances is
     *   available. (list may be empty, if no matching service instances currently exist)
     * - does reflect the availability at the time of the method call. No further (background) checks of availability
     *   are done.
     *
     * \param instance Instance Specifier qualifying the service instances that shall be searched/found.
     *
     * \uptrace{SWS_CM_00622, b102b02dce909ff9796a59e634e9f5ad9370806a}
     */
    static ara::core::Result<ServiceHandleContainer<HandleType>> FindService(ara::core::InstanceSpecifier instance);

    /**
     * \brief Installs a handler that is called whenever the set of offered instances for this service changes.
     *
     * If you use this method, the middleware will continuously monitor the availability of the services and call the
     * handler on any change. This means also that service instances that are not available anymore simply disappear
     * from the list that is given to the callback.
     *
     * StartFindService does not need an explicit version parameter as this is internally available in ProxyClass
     * That means only compatible services are returned.
     *
     * \param handler Handler that gets called any time the service availability of the services matching the given
     * instance criteria changes.
     *
     * \param instance Which instance of the service shall be searched/found. A wildcard may be given to
     * look for all compatible instances. Default value is wildcard.
     *
     * \return a handle for this search/find request, which shall be used to stop the availability monitoring and
     * related firing of the given handler. (\ref StopFindService())
     *
     * \uptrace{SWS_CM_00123, 01fa2c1b9fac60d7ae46b7af48403600fb2e683b}
     **/
    static ara::core::Result<FindServiceHandle> StartFindService(FindServiceHandler<HandleType> handler,
        InstanceIdentifier instance = InstanceIdentifier::MakeAny());

    /**
     * \brief Installs a handler that is called whenever the set of offered instances for this service changes.
     *
     * If you use this method, the middleware will continuously monitor the availability of the services and call the
     * handler on any change. This means also that service instances that are not available anymore simply disappear
     * from the list that is given to the callback.
     *
     * StartFindService does not need an explicit version parameter as this is internally available in ProxyClass
     * That means only compatible services are returned.
     *
     * \param handler Handler that gets called any time the service availability of the services matching the given
     * instance criteria changes.
     *
     * \param instance Instance Specifier qualifying the service instances that shall be searched/found.
     *
     * \return a handle for this search/find request, which shall be used to stop the availability monitoring and
     * related firing of the given handler. (\ref StopFindService())
     *
     * \uptrace{SWS_CM_00623, d290d7f3ab8ff20337af911263cd3aaf107e32e9}
     **/
    static ara::core::Result<FindServiceHandle> StartFindService(FindServiceHandler<HandleType> handler,
        ara::core::InstanceSpecifier instance);

    /**
     * \brief Unregisters the previously installed handler using the returned handle.
     *
     * \param handle
     *
     * \uptrace{SWS_CM_00125, 623b1cf4ca84f3c9b7b93e895546caf037666b21}
     */
    static void StopFindService(FindServiceHandle handle);

protected:
    std::unique_ptr<ProxyBinding>
        proxy_base_;  ///< Binding specific class that holds references to all interface elements.
    HandleType handle_;  ///< Handle that was used to create this instance.
};

```

！！！关键类`ara::com::sample::radar_binding::dds`的`radarImpl`，继承自`radarProxyBase`
`radarProxy`继承自`ProxyBase<radarProxyBase>`，而`radarImpl`是继承自`radarProxyBase`

`radarImpl`类
```cpp
class radarImpl
    : public proxy::radarProxyBase
    , public ::ara::com::internal::dds::proxy::ProxyImplBase
{
public:
    using ProxyInterface = proxy::radarProxy;
    using ServiceDescriptor = descriptors::internal::Service;

    explicit radarImpl(
        ara::com::InstanceIdentifier instance,
        ara::com::internal::dds::common::HandleInfo handle)
        : ::ara::com::internal::dds::proxy::ProxyImplBase(ServiceDescriptor::kServiceId,
            instance,
            ServiceDescriptor::kMajorVersion,
            ServiceDescriptor::kMinorVersion,
            handle)
        , brakeEvent(instance, handle)
        , parkingBrakeEvent(instance, handle)
        , Adjust(instance, handle)
        , UpdateRate(instance, handle)
    { }

    virtual ~radarImpl()
    { }

    virtual proxy::events::brakeEvent& GetbrakeEvent() override
    {
        return brakeEvent;
    }

    virtual proxy::events::parkingBrakeEvent& GetparkingBrakeEvent() override
    {
        return parkingBrakeEvent;
    }

    virtual proxy::methods::Adjust& GetAdjust() override
    {
        return Adjust;
    }

    virtual proxy::fields::UpdateRate& GetUpdateRate() override
    {
        return UpdateRate;
    }

private:
    ara::com::internal::dds::proxy::EventImpl<descriptors::brakeEvent,
        EventbrakeEventTypeInfo>
        brakeEvent;
    ara::com::internal::dds::proxy::EventImpl<descriptors::parkingBrakeEvent,
        EventparkingBrakeEventTypeInfo>
        parkingBrakeEvent;
  ara::com::internal::dds::proxy::MethodImpl<descriptors::Adjust,
      MethodAdjust,
      typename MethodAdjust::signature_t>
      Adjust;
  ara::com::internal::dds::proxy::MutableFieldImpl<FieldUpdateRate>
      UpdateRate;
};
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
在创建`MethodDataReader`时创建了`MethodDataDispatcher`对象
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

在**用户层面**是通过`GetNewSamples`方法获取数据的
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
事先在初始化时通过MethodImpl构造了所需的对象（MethodDataReader）

### 关于method如何发送请求
在用户层代码（即`act()`中）中，调用m_proxy(`radarProxy`类型)的`Adjust`对象提供的`operator()`
```cpp
    auto adjust_future = m_proxy->Adjust(tmp);  // 如同这样的调用
```
其中的tmp是用户层传入，本文举例的工程中service interface定义了Adust方法中：
1. `ara::com::sample::Position`类型的target_position参数 in
2. bool类型的success参数 out
3. `ara::com::sample::Position`类型的effective_position参数 out
`FIRE_AND_FORGET`设置为false
    PS：`FIRE_AND_FORGET`即无需等待该调用的完成或获取其返回结果。这意味着调用者不关心该调用的结果和执行状态，而是只关注调用操作的触发。

注意此处的`Adjust`是起了别名：
```cpp
using Adjust = ara::com::internal::proxy::Method<ara::com::sample::radar::AdjustOutput(const ::ara::com::sample::APString&)>;
```
实际上是`Method`对象的派生类`MethodImpl`重载了`operator()`
由于radarProxy类型的对象在初始化时为成员变量赋值的是派生类对象，因此实现了多态，调用到以下的代码：

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
field的特点在于：
在skeleton端初始化时，需要一并设置初值（毕竟field是用于表示温度这种持续存在的变量的）。

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
“on change notification”、`Get()`、`Set()`

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







