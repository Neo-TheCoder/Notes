# hello world
## pub端
构造了：
一个participant     带QOS
一个publisher       带QOS
一个TypeSupport     需要注册participant
一个topic           带QOS
一个DataWriter      带QOS，以及DataWriterListener

一个participant对应多个publisher，一个publisher对应多个DataWriter

```cpp
// participant and its qos
    DomainParticipantQos pqos = PARTICIPANT_QOS_DEFAULT;    // default qos object
    pqos.name("Participant_pub");
    auto factory = DomainParticipantFactory::get_instance();    // factory模式

    if (use_env)
    {
        factory->load_profiles();
        factory->get_default_participant_qos(pqos);
    }

    participant_ = factory->create_participant(0, pqos);    // create participant

// publisher and its qos
    PublisherQos pubqos = PUBLISHER_QOS_DEFAULT;    // publisher QOS

    if (use_env)    // 无特别输入，则为false
    {
        participant_->get_default_publisher_qos(pubqos);
    }

    publisher_ = participant_->create_publisher(
        pubqos,
        nullptr);

// TypeSupport
type_.register_type(participant_);

//  Topic
topic_ = participant_->create_topic(
    "HelloWorldTopic",  // topic_name
    "HelloWorld",   // type_name
    tqos);  // TopicQos对象

// DataWriter
writer_ = publisher_->create_datawriter(
    topic_, // participant_创建的topic
    wqos,   // DataWriter qos
    &listener_);    // PubListener对象

// DataReaderListener需要重写 on_publication_matched 方法
```

## sub端
构造了：
一个participant     带QOS
一个subscriber      带QOS
一个TypeSupport     需要注册participant
一个topic           带QOS
一个DataReader      带QOS，以及DataReaderListener


```cpp
// participant and its qos
    DomainParticipantQos pqos = PARTICIPANT_QOS_DEFAULT;
    pqos.name("Participant_sub");
    auto factory = DomainParticipantFactory::get_instance();

    if (use_env)
    {
        factory->load_profiles();
        factory->get_default_participant_qos(pqos);
    }

    participant_ = factory->create_participant(0, pqos);    // domain ID / qos

// subscriber and its qos
    //CREATE THE SUBSCRIBER
    SubscriberQos sqos = SUBSCRIBER_QOS_DEFAULT;

    if (use_env)
    {
        participant_->get_default_subscriber_qos(sqos);
    }

    subscriber_ = participant_->create_subscriber(sqos, nullptr);

// TypeSupport
    //REGISTER THE TYPE
    type_.register_type(participant_);  // type_是TypeSupport对象

// Topic
    topic_ = participant_->create_topic(    // pub和sub创建的topic需要匹配
        "HelloWorldTopic",  // Topic_name
        "HelloWorld",       // Type_name
        tqos);

// DataReader
    // CREATE THE READER
    DataReaderQos rqos = DATAREADER_QOS_DEFAULT;
    rqos.reliability().kind = RELIABLE_RELIABILITY_QOS; // DataReaderQos：设置了 reliability.kind 为 RELIABLE_RELIABILITY_QOS

    if (use_env)
    {
        subscriber_->get_default_datareader_qos(rqos);
    }

    reader_ = subscriber_->create_datareader(topic_, rqos, &listener_); // 读取数据的逻辑都在listener_内部

// DataReaderListener需要重写 on_subscription_matched 方法
```













