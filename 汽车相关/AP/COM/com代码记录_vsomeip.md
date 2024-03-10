# SOME/IP版本实现
相比fastdds版本，更为直接一些

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
        typename LastValueType::LockedPtr last_value{last_value_sent_->Get()};  // ！！！返回一个<std::tuple<T, bool>, unique_lock对象>构造的LockedPtrWrapper对象
        std::get<0>(*last_value) = data;    // 解引用以取得元组对象，更新数据
        std::get<1>(*last_value) = true;    // 这个bool的意义是在FieldDispatcher内部的其他函数里做判断，比如是否初始化
        return Base::Send(data);    // Base即EventDispatcher
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
如果事件数据很大并且发送频率很高，这个版本的Update函数可能会占用较多的内存，因为需要将数据复制到输出缓冲区。但是，它不需要调用`Allocate`函数，并且可以发送用户分配的任意数据缓冲区。
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


