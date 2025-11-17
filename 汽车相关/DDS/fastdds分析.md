# 关键字
## eProsima Fast DDS-Gen
eProsima Fast DDS-Gen是一个 Java 应用程序，它使用 IDL（接口定义语言）文件中定义的数据类型生成eProsima Fast DDS源代码。
eProsima Fast DDS-Gen生成的源代码使用Fast CDR，这是一个提供数据序列化和编码机制的 C++11 库。
因此，正如RTPS 标准中所述，发送数据时，它们会使用相应的通用数据表示 (CDR) 进行序列化和编码。
CDR 传输语法是代理间传输的低级表示，从 OMG IDL 数据类型映射到字节流。

## foonathan_memory_vendor
an STL compatible C++ memory allocator library.

## fastcdr
a C++ library for data serialization according to the CDR standard (Section 10.2.1.2 OMG CDR).

## fastrtps
the core library of eProsima Fast DDS library.

## RTPS协议
一个domain  --  若干个participant   --  若干个Publisher（和DataWriter是`一对多`关系）、若干个Subscriber（和DataReader是`一对多`关系）
**一个`participant`对应若干个`topic`**
**一个`Publisher`对应若干个`topic`**

Fast DDS中的`RTPS层`实现了`RTPS`协议，它作为DDS标准的⼀部分，负责处理底层⽹络通信、发现服务、数据传输以及QoS策略的具体实现等。
在Fast DDS中，`DCPS层`为`⽤⼾`提供抽象接⼝，隐藏了底层通信机制的复杂性；
⽽`RTPS层`则是`实际执⾏这些操作并保证数据可靠传输`的基础。
实际代码中，存在这样的映射：`DCPS` --> `RTPS`

## 并⾏模型
FastDDS中每个节点（也叫 `DomainParticipant`）具有：
⼀个`主程序线程`（⽤⼾持有）
⼀个`事件和周期性任务`的线程
⼀个`异步发送线程`，⽤于⽤⼾完成写⼊数据后，异步得完成⽹络通信
多个`接收线程`，每个`reception channel`，取决于传输层的实现⽅式

## RTPS的通信传输实现
⽀持SHM，UDP，TCP

## 域（Domain）：
这是用于链接所有发布者和订阅者的概念，属于一个或多个应用程序，它们在不同主题下交换数据。
这些参与域的单个应用程序称为DomainParticipant。DDS域由`domain ID`标识。
DomainParticipant定义Domain ID以指定它所属的DDS域。
具有不同ID的两个DomainParticipants不知道彼此在网络中的存在。
因此，可以创建多个通信通道。
这适用于涉及多个DDS应用程序的场景，它们各自的DomainParticipants相互通信，但这些应用程序不得干扰。
DomainParticipant充当其他 DCPS实体的容器，充当发布者、订阅者和主题实体的工厂，并在域中提供管理服务。

## 主题（Topic）：
**它是将发布者的`DataWriters`与订阅者的`DataReaders`绑定的实体，在DDS域中是`唯一的`**。
它可在进程之间交换的数据的消息，数据表示为可以包含不同数据类型的结构，如整数，字符串等;

## 数据写入器（Data Writer）：
它是负责`发布消息`的实体，用户在创建此实体时必须提供一个主题，该主题将是发布数据的主题；

## 数据读取器（Data Reader）：
它是订阅主题以`接收发布`的实体，用户在创建此实体时必须提供订阅主题；

## 发布者（Publisher）：
它是负责`创建 和 配置`其实现的`DataWriters`的DCPS实体。
`DataWriter`是负责实际`发布消息`的实体。
每个人都有一个分配的主题，在该主题下发布消息;

## 订阅者（Subscriber）：
它是DCPS实体，负责接收在其订阅的主题下发布的数据。
它为一个或多个`DataReader`对象提供服务，这些对象负责将新数据的可用性传达给应用程序;

## 发布消息步骤：
创建要发布数据的 idl ，并通过 fast-gen 生成对应的数据结构，并生成该数据结构序列化的类
通过 `DomainParticipantFactory` 创建 participant ，参数为 `domain id` 和 `qos`
将 participant 注册到 type ，该类型提供 helloworld 的序列化支持
通过 participant 创建 publisher ，用于发布数据
`PubListener` 继承于 `DataWriterListener` ，用于给发布者注册消息通知函数。
DDS 通过服务发现，将消息最终给到应用
publisher 创建 writer ，用于最终的消息发布

## QOS
## someip_to_dds所使用的
基本上就是ROS2里默认的QOS profile
1. DataWriter
```cpp
  wqos.reliability().kind = RELIABLE_RELIABILITY_QOS;
  wqos.history().kind = KEEP_LAST_HISTORY_QOS;
  wqos.history().depth = 10;                    // 必须配合 KEEP_LAST_HISTORY_QOS
  wqos.endpoint().history_memory_policy = eprosima::fastrtps::rtps::PREALLOCATED_WITH_REALLOC_MEMORY_MODE;
```
2. DataReader
```cpp
  rqos_adasv2positiontopic.reliability().kind = RELIABLE_RELIABILITY_QOS;
  rqos_adasv2positiontopic.history().kind = KEEP_LAST_HISTORY_QOS;
  rqos_adasv2positiontopic.history().depth = 10;
  rqos_adasv2positiontopic.endpoint().history_memory_policy = eprosima::fastrtps::rtps::PREALLOCATED_WITH_REALLOC_MEMORY_MODE;
```

`enumerator PREALLOCATED_MEMORY_MODE`
Preallocated memory.   预分配内存。

Size set to the data type maximum. Largest memory footprint but smallest allocation count.
Size 设置为数据类型 maximum。内存占用量最大，但分配计数最小。

`enumerator PREALLOCATED_WITH_REALLOC_MEMORY_MODE`
Default size preallocated, requires reallocation when a bigger message arrives.
默认大小 `preallocated`，当更大的消息到达时需要重新分配。

Smaller memory footprint at the cost of an increased allocation count.
内存占用更少，但代价是分配计数增加。


## best effort
减少重传和确认机制


## reliable
使用了消息确认机制


# 收数据的接口
`on_data_available`由`transport层`的接口调用到`perform_listen_operation(Locator)`
```cpp
void UDPChannelResource::perform_listen_operation(
        Locator input_locator)
{
    Locator remote_locator;

    while (alive())
    {
        // Blocking receive.
        auto& msg = message_buffer();
        if (!Receive(msg.buffer, msg.max_size, msg.length, remote_locator))
        {
            continue;
        }

        // Processes the data through the CDR Message interface.
        if (message_receiver() != nullptr)
        {
            message_receiver()->OnDataReceived(msg.buffer, msg.length, input_locator, remote_locator);
        }
        else if (alive())
        {
            logWarning(RTPS_MSG_IN, "Received Message, but no receiver attached");
        }
    }

    message_receiver(nullptr);
}
```

一种channel对应一个线程
udp的实现：（使用`asio`中的接口，阻塞地收数据）

```cpp
while() {
    //  ...
    receive_from();
}
```

shm的实现：
```cpp
    /**
     * Function to be called from a new thread, which takes cares of performing a blocking receive
     * operation on the ReceiveResource
     * @param input_locator - Locator that triggered the creation of the resource
     */
    void perform_listen_operation(
            Locator input_locator) {
        while() {
            // 阻塞队列Pop
            // ...
            // OnDataReceived();
        }
    }
```

# `write()`如何调用到`serialize()`
可以还原类型
**`DataWriterImpl`可以拿到`TypeSupport`（继承自`shared_ptr`），由`Participant`持有**
序列化的数据，若序列化失败，塞到`payload_pool`
成功则塞进`history_`（由`DataWriterImpl`持有）




# 接收端如何被通知？
底层transport层进行通知，每个通道一个线程







# fastdds实现
## 发数据 `write(void*)`
```cpp
DataWriterImpl::write(void* data)

DataWriterImpl::create_new_change(ALIVE, data)  // ALIVE, 表示样本有效，其他状态还包括：被处置、注销、处置以及注销
{
    WriteParams wparams;
    return create_new_change_with_params(changeKind, data, wparams);
}

ReturnCode_t DataWriterImpl::create_new_change_with_params(ChangeKind_t changeKind, void* data, WriteParams& wparams)
{
    ReturnCode_t ret_code = check_new_change_preconditions(changeKind, data);
    // ...
    InstanceHandle_t handle;
    // ...
    return perform_create_new_change(changeKind, data, wparams, handle);
}

// ！！！PS：用户层可以通过loan_sample(void*& sample, LoanInitializationKind initialization)接口取得
ReturnCode_t DataWriterImpl::perform_create_new_change(ChangeKind_t change_kind, void* data, WriteParams& wparams, const InstanceHandle_t& handle)
{
    std::unique_lock<RecursiveTimedMutex> lock(writer_->getMutex());
    PayloadInfo_t payload;
    bool was_loaned = check_and_remove_loan(data, payload);
    // 如果有借用，则可以通过指针或者说地址得知，从记录的借用列表中移除
    // 如果没有借用。则调用：
        get_free_payload_from_pool(type_->getSerializedSizeProvider(data), payload)     // 先把数据指针转移到CacheChange_t对象中，再转移到payload中
    // 从内存池中取出free payload（这里的内存池实际对象和是否开启data sharing[数据共享交付 DataReader共享DataWriter的history 难道就是零拷贝？？？]有关，DataSharingPayloadPool / TopicPayloadPool。PREALLOCATED_WITH_REALLOC_MEMORY_MODE等QOS配置也影响到此处）
    ！！！free_payloads_其实是vector类型
    // 调用序列化
    // 构造一个新的CacheChange_t对象，把上面的payload的数据再转移进去
    // 把数据塞进history，调用到rtps层的add_change_(CacheChange_t* a_change, WriteParams& wparams, std::chrono::time_point<std::chrono::steady_clock> max_blocking_time)，其中有一行关键调用：
        // WriterHistory:
        notify_writer(a_change, max_blocking_time);     // RTPSWriter有好几种！！！具体选择哪种实例化应该是createRTPSWriter(...)里面决定的，内部是根据QOS配置来决定的：如果是reliable就是StatefulWriter、如果是unreliable就是StatelessWriter
            // 内部实现：
            mp_writer->unsent_change_added_to_history(a_change, max_blocking_time);
}

// DataWriter提供一个指向内部缓冲区（池）的指针，用户可以直接在此缓冲区内准备数据以供发送。此方法仅适用于“平凡数据类型”的DataWriter
ReturnCode_t DataWriterImpl::loan_sample(void*& sample, LoanInitializationKind initialization);

        /*
            上面的数据写入到history之后，显然会异步地发送
            先把数据传到FlowController对象，塞进Scheduler，其维护一个Queue
        */
                sched.add_new_sample(writer, change);

                // 专门的线程，跑while循环
                // 通知具体的Writer，数据可以走底层发送了，调接口判断采取哪种方式进行分发
                    fastrtps::rtps::DeliveryRetCode ret_delivery = current_writer->deliver_sample_nts(
                        change_to_process, async_mode.group, locator_selector,
                        std::chrono::steady_clock::now() + std::chrono::hours(24));
```

数据最终如何分发？分为三种情况：
1. 同一进程内的DataReader、
2. 同一个域的
3. 跨域的
```cpp
DeliveryRetCode StatefulWriter::deliver_sample_nts(
        CacheChange_t* cache_change,
        RTPSMessageGroup& group,
        LocatorSelectorSender& locator_selector, // Object locked by FlowControllerImpl
        const std::chrono::time_point<std::chrono::steady_clock>& max_blocking_time)
{
    DeliveryRetCode ret_code = DeliveryRetCode::DELIVERED;

    if (there_are_local_readers_)
    {
        deliver_sample_to_intraprocesses(cache_change);
    }

    // Process datasharing then
    if (there_are_datasharing_readers_)
    {
        deliver_sample_to_datasharing(cache_change);
    }

    if (there_are_remote_readers_)
    {
        ret_code = deliver_sample_to_network(cache_change, group, locator_selector, max_blocking_time);
    }

    check_acked_status();

    return ret_code;
}
```
这里如何通知到udp writer？/ 如何和transport层交互？？？
可能是通过条件变量，notify某一个listner线程，让其执行

？？？locator是什么东西？？？网络定位符？是ip + port吗？



## 收数据
每种通道，都有一个线程在执行`while`
```cpp
void UDPChannelResource::perform_listen_operation(
        Locator input_locator)
{
    Locator remote_locator;

    while (alive())
    {
        // Blocking receive.
        auto& msg = message_buffer();
        if (!Receive(msg.buffer, msg.max_size, msg.length, remote_locator))
        {
            continue;
        }

        // Processes the data through the CDR Message interface.
        if (message_receiver() != nullptr)
        {
            message_receiver()->OnDataReceived(msg.buffer, msg.length, input_locator, remote_locator);
        }
        else if (alive())
        {
            logWarning(RTPS_MSG_IN, "Received Message, but no receiver attached");
        }
    }

    message_receiver(nullptr);
}
```

## 内存池
### WriterPool
调`write()`时，计算`数据长度（递归计算）`，向内存池申请一块内存，
它存储每个序列化后的数据buffer，以`CacheChange_t`的形式存储，存储到`DataWriter`的history中

内存池其实就是：
```cpp
std::vector<PayloadNode*> free_payloads_;
```
每个`PayloadNode`所申请的内存长度是不一的，必要时才重新申请（直接调底层的`realloc(ptr, size)`）
内存池大小和history的深度相关

### ReaderPool
？？？



## data sharing
Although Data-sharing delivery uses shared memory, it differs from Shared Memory Transport in that Shared Memory is a full-compliant transport. That means that with Shared Memory Transport the data being transmitted must be copied from the DataWriter history to the transport and from the transport to the DataReader. With Data-sharing these copies can be avoided.

尽管数据共享传输使用共享内存，但它与共享内存传输不同 因为共享内存是完全兼容的传输。这意味着，使用共享内存传输 必须将正在传输的数据从 DataWriter 历史记录复制到传输 以及从运输到 DataReader。通过数据共享，可以避免这些副本。

省略了writer的缓存和reader的缓存（设计了巧妙的数据结构和同步机制）
--> 环形缓冲区
无锁编程 / 轻量级锁 ？







