# DDS
**数据分发服务**（Data Distribution Service）是一种以数据为中心的通信协议。
用于分布式软件应用通信。该规范定义了应用程序接口 API和通信语义（行为和服务质量），它们能够有效地将信息从信息生产者传递到匹配的消费者。
DDS规范的目的可以概括为：**在正确的时间将正确的信息高效、稳健地传递到正确的地点**。


# RTPS
实时发布订阅协议（Real-Time Publish Subscribe）。
该协议通过 UDP 等不可靠的传输，实现最大努力（Best-Effort）和可靠的发布-订阅通信。
RTPS是DDS实现的标准协议，它的目的和范围是确保基于不同DDS供应商的应用程序可以实现相互操作。


# 一些概念
## DCPS
**Data-Centric Publish Subscribe model**，即以数据为中心的发布订阅模型

## Domain
每个 domain 通过 domain id 来标识。**拥有不同 domain id 的Participant 无法通信**。通过domain id 隔离工作区间，这样可以实现多个 DDS app 相互通信互不干扰。

## Participant
充当其他 DCPS 实体（Publisher, Subscriber and Topic Entities）的容器（container），在域（domain）内作为一个工厂（factory）来创建和
管理其他实体。

## Publisher / Writer
 Publisher 创建和配置 Writer。
 一个 Publisher可以有一个或多个Writer。
Writer通过一个指定的 topic，负责实际的信息的发布。

## Subscriber / Reader
Subscriber 创建和配置 Reader。
一个 Subscriber 可以有一个或多个Reader。
Reader 接收订阅的 topic 发布的数据。

## Topic
绑定 publications 和 subscriptions，
保证了发布方和订阅方数据类型的统
一。
**一个Topic在一个 DDS domain里面是唯一的。**

## Type
描述发布和订阅之间交换的数据类型，即与 Topic 对应。
`Type Support类`提供了序列化、反序列化和获取特定数据类型值等方法。

## Change
DDS 中最基础的通信单元，表示在Topic下写入的数据的更新。

## History
一种用作最近 Change 的缓存的数据结构，Writers / Readers 在其 History中注册这些 Change。

## QoS
**Quality of Service**（服务质量）是控制 DDS 服务行为的某些方面的一组特征。

### Reliability
#### Reliable :
发送方（发布者）期望接收方（订阅者）进行**到达确认**。
速度较慢，但可以防止数据丢失。

#### Best-effort :
发送消息时，接收者（订阅者）没有到达确认。速度快，但是消息可能会丢失。


# 服务发现过程
为了实现通信，DDS提供了Publisher和Subscriber自动发现和匹配的功能。这主要通过`多播通信`实现，**允许Participant自动查找和匹配DataWriters 和 DataReader**。
在具体的实现上，这分为两个步骤来完成：
1. `Participant Discovery Phase`（PDP）：
在这个阶段，参与者互相通知彼此的存在。为了达到这个目的，每个参与者需要定时发送**公告消息**。
公告消息通过周知的多播地址和端口发送（根据domain id计算得到）。

2. `Endpoint Discovery Phase`（EDP）：
在这个阶段，Publisher和Subscriber互相确认。
为此，参与者使用在PDP期间建立的通信通道，彼此共享有关其发布者和订阅者的信息。该信息包含了Topic和数据类型。
为了使两个端点匹配，它们的**Topic和数据类型必须一致**。
一旦发布者和订阅者匹配，他们就发送/接收数据了。

这两个阶段对应了两个独立的协议：
1. `SPDP`（Simple Participant Discovery Protocol）：指定参与者如何在网络中发现彼此。
2. `SEDP`（Simple Endpoint Discovery Protocol）：定义了已经互相发现的参与者交换信息的协议。


# 一些关键接口
```cpp
  class PubListener : public eprosima::fastdds::dds::DataWriterListener {
    public:
        PubListener(SomeipToDdsParticipant& sp) : sp_(sp)
        {}

        ~PubListener() override {}

        void allow_send(SomeipToDdsParticipant& sp){
          std::unique_lock<std::mutex> lock{sp.lock_for_service1_method1_send};
          sp.if_service1_method1_matched = true;
          sp.cv_service1_method1_send.notify_all();
        }

        void on_publication_matched(
                eprosima::fastdds::dds::DataWriter* writer,
                const eprosima::fastdds::dds::PublicationMatchedStatus& info) override {
                  static_cast<void>(writer);
                  // static_cast<void>(info);
                  if (info.current_count_change == 1 && info.current_count == 1)
                  {
                    // 服务发现完成，可以开始发送和接收数据
                    allow_send(sp_);
                  }
              }

    private:
      SomeipToDdsParticipant& sp_;

  }wlistener_{*this};
```

若干`data_writer`在创建时，都会绑定到该`wlistener_`对象
```cpp
    DataWriterQos wqos_routinglocalroadoutputdebugtopic = DATAWRITER_QOS_DEFAULT;
    wqos_routinglocalroadoutputdebugtopic.reliability().kind = RELIABLE_RELIABILITY_QOS;
    wqos_routinglocalroadoutputdebugtopic.history().kind = KEEP_LAST_HISTORY_QOS;
    wqos_routinglocalroadoutputdebugtopic.history().depth = 10;
    // CREATE THE WRITER OF service : planning_routing_debug_service_interface event: RoutingLocalRoadOutputDebugTopic
    writer_routinglocalroadoutputdebugtopic_ = publisher_->create_datawriter(
        wtopic_routinglocalroadoutputdebugtopic_, wqos_routinglocalroadoutputdebugtopic, &wlistener_);
```

`on_publication_matched()`是在`DataWriter`和`DataReader`匹配状态发生变化时触发
***This method is called when the DataWriter is matched (or unmatched) against an endpoint.***
判断当前匹配状态变化的次数是否为1，并且当前匹配状态的数量是否为1。










