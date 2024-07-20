# 一些关键逻辑
当someipd收到`控制信息`，根据类型，调用相应的`Handler`（就是简单的switch-case逻辑）

`client app`通过IPC向someipd发送两种消息，其发送的时机是这样的：
# `find service`
## client app
`StartFindService`() -> ... -> `send control message`

## someipd
`OnControlMessage()`
someipd得知client app想要find service，发送`源IP是单播，目的IP是多播的SD--Find消息`

# request service
## client app
`proxy obejct creation` -> ... -> `Proxy()`, and then `Preconstruct()` -> `CreateBackend()` -> `CreateClient()` -> `RequestService()`, `send control message`
是method的请求

## someipd
`OnControlMessage()` -> `RequestService()`
(当`IsOffered()`为true，才执行往下执行) -> `Connect()` -> `ConnectTCP()`，当`socket可写`，触发`回调`，改变`状态机状态`，设置为`Connected`

# 关于someip_config.json的解析
```cpp
/*!
 * \internal
 * - Open the config file in the supplied path as an input file stream
 * - Check if the file with supplied path is opened and has content
 *   - Initialize configuration structs to be filled by the parser
 *   - Create a configuration parser and initialize it with the application config JSON data
 *   - Parse the the application config file until the current parser is finished
 *   - Check if the application parser result is valid
 *     - Merge the parsed JSON config with the global configuration
 *   - Otherwise, indicate that the parsed configuration is not valid and log an error that parsing failed
 * - Otherwise, log a warning that application configuration JSON not found at the supplied path
 * \endinternal
 */
void JsonConfiguration::ParseApplicationGenConfigJson(std::string const& file_path) {
  // PTP-B-JsonConfiguration_ParseApplicationGenConfigJson
  amsr::stream::filestream::InputFileStream appl_gen_config_fs{};
  ara::core::Result<void> const open_result{appl_gen_config_fs.Open(file_path)};

  if (open_result.HasValue()) {
    /* Initialize configuration structs to be filled by the parser */
    ConfigurationTypesAndDefs::ServiceContainer services_container{};
    ConfigurationTypesAndDefs::RequiredServiceInstanceContainer required_service_instances{};
    ConfigurationTypesAndDefs::ProvidedServiceInstanceContainer provided_service_instances{};
    ConfigurationTypesAndDefs::NetworkEndpointContainer network_endpoints{};
    ConfigurationTypesAndDefs::SecureCom secure_com{};
    ConfigurationTypesAndDefs::GeneratorVersion generator_version{};

    /* Parse someip_config.json file */
    amsr::json::JsonData appl_gen_config_json{appl_gen_config_fs};
    parsing::ConfigurationParser application_gen_parser{
        appl_gen_config_json, services_container, required_service_instances, provided_service_instances,
        network_endpoints,    secure_com,         generator_version};
    ara::core::Result<void> const application_gen_parser_result{application_gen_parser.Parse()};
    if (application_gen_parser_result) {
      logger_.LogVerbose(
          [&file_path](ara::log::LogStream& s) {
            s << "Merging parsed someip_config.json file to global configuration '" << file_path << "'.";
          },
          __func__, __LINE__);

      bool const add_services_result{AddServices(services_container)};
      bool const add_network_endpoints_result{AddNetworkEndpoints(network_endpoints)};
      bool const add_required_service_instances_result{AddRequiredServiceInstances(required_service_instances)};
      bool const add_provided_service_instances_result{AddProvidedServiceInstances(provided_service_instances)};
      bool const add_secure_endpoints{AddSecureEndpoints(secure_com.secure_endpoints)};
      bool const add_machine_psk_identity_hint_result{AddMachinePskIdentityHint(secure_com.psk_identity_hint)};
      // Final validation of configuration consistency
      bool const validate_configuration_consistency_result{ValidateConfigurationConsistency()};
      is_valid_ = (((is_valid_ && add_services_result) &&
                    (add_required_service_instances_result && add_provided_service_instances_result)) &&
                   ((add_network_endpoints_result && add_secure_endpoints) &&
                    (add_machine_psk_identity_hint_result && validate_configuration_consistency_result)));

    } else {
      is_valid_ = false;
      logger_.LogError(
          [&file_path](ara::log::LogStream& s) { s << "Parsing of JSON file '" << file_path << "' failed."; }, __func__,
          __LINE__);
    }
  } else {
    logger_.LogWarn(
        [&file_path](ara::log::LogStream& s) {
          s << "Skipped reading in application configuration. Application configuration JSON not found at: "
            << file_path;
        },
        __func__, __LINE__);
  }
  // PTP-E-JsonConfiguration_ParseApplicationGenConfigJson
}
```

先使用输入流打开文件，然后把配置信息存入各个配置类对象：
1、  `std::<Service>vector`类型
`Service`类的成员包括：
service id、major_version、minor_version、methods、events、eventgroups（包含event group id和event）

2、  **RequiredServiceInstance**  类型的  **vector**
**RequiredServiceInstance类的成员包括：**
**service_id、**  **instance_id**  **、major_version、minor_version、（MachineMapping**  类型的  **）port_mapping、RequiredServiceInstanceServiceDiscovery**

**其中MachineMapping**  类的成员包括：

network endpoint   **IpAddress（实际是string类型）**  、  **TCP Port**  、  **UDP Port**  、  **event multicast IpAddress**  、  **network mask**  、  **IpAddressPrefixLength（即网络号的位数）**  、  **Network（ip + mask + prefix）**

其中  **RequiredServiceInstanceServiceDiscovery**  类的成员包括：

（required service的SD参数）

**Ttl**  （FindService entry的TTL）、  **InitialRepetitionsMax**  、initial_delay_min_、initial_delay_max_（FindService entry的最大延迟）、initial_repetitions_base_delay_、  **RequiredServiceInstanceSdEventgroup**  类型的  **vector**

**RequiredServiceInstanceSdEventgroup是SD eventgroup 参数，其成员包括：EventgroupId**  、  **Ttl（**  SubscribeEventgroup entry的TTL  **）**  、request_response_delay_min_（SubscribeEventgroup entry的最小延迟）、request_response_delay_max_



3、  **ProvidedServiceInstance**  类型的  **vector**

**ProvidedServiceInstance类的成员包括：**

**service_id、**  **instance_id**  **、major_version、minor_version、（MachineMapping**  类型的  **）port_mapping、**  ProvidedServiceInstanceServiceDiscovery

**其中MachineMapping**  类的成员包括：

network endpoint   **IpAddress（实际是string类型）**  、  **TCP Port**  、  **UDP Port**  、  **event multicast IpAddress**  、  **network mask**  、  **IpAddressPrefixLength（即网络号的位数）**  、  **Network（ip + mask + prefix）**

其中  **ProvidedServiceInstanceServiceDiscovery**  类的成员包括：

（required service的SD参数）

**Ttl**  （FindService entry的TTL）、  **InitialRepetitionsMax**  、initial_delay_min_、initial_delay_max_（FindService entry的最大延迟）、initial_repetitions_base_delay_、cyclic_offer_delay_、request_response_delay_min_、request_response_delay_max_、  **ProvidedServiceInstanceSdEventgroup**  类型的  **vector**

**ProvidedServiceInstanceSdEventgroup**  是provided service的SD eventgroup参数  **，其成员包括：EventgroupId**  、event_multicast_threshold_、request_response_delay_min_、request_response_delay_max_



4、  **NetworkEndpoint**  类型  **vector**

其成员包括：

**IpAddress**  、  **MTU**  、  **NetworkEndpointServiceDiscovery（用于多播SD消息的多播**  IpAddress、Port  **）**  、  **NetworkEndpointPort**  类型的vector（  **Port**  、  **Protocol**  、  **SocketOptions（QoS、TcpKeepAliveOption、EnableSocketOptionLinger）**  、  **ComRequestedType**  （枚举类，用于表示一个endpoint是否被标记为仅使用通信或服务发现。））





# 调用顺序 SOME/IP daemon
**PS：Run()函数做了什么？**
![image.png](https://atlas.pingcode.com/files/public/65460c103a27284c5ca127f4/origin-url)
**解析并验证配置（若无效，则直接终止）、启动signal handling（为了恰当地终止）、启动reactor线程（用于通信和时间处理）、创建用于APP在运行时来进行的BasicIPC通信连接、启动服务发现（多播通信）**

***PS：个别应用程序使用Basic IPC 通信通道与 SOME/IP 守护进程进行交互。通过这个Basic IPC 通信通道，应用程序可以提供、查找服务接口。***
***SOME/IP 数据包的发送和接收也通过该通信通道完成。***


## **Initialize**  ()
### SomeIpDaemonClass的`Initialize()`
```cpp
 /*!
   * \brief      Initializes the SOME/IP daemon.
   * \param[in]  args The command line arguments.
   * \return     ara::core::Result with either void value in case of success or an "InitializationError".
   * \pre        -
   * \context    Init
   * \threadsafe FALSE
   * \reentrant  FALSE
   */
  ara::core::Result<void, InitializationError> Initialize(CommandLineArguments const& args) {
    ::amsr::someip_daemon::logging::AraComLogger logger{amsr::someip_daemon::logging::kSomeIpDLoggerContextId,
                                                        amsr::someip_daemon::logging::kSomeIpDLoggerContextDescription,
                                                        "Initialize()"};

    logger.LogInfo([](ara::log::LogStream& s) { s << "Starting the SOME/IP Daemon"; }, __func__, __LINE__);

    ara::core::Result<void, InitializationError> result{InitializationError::kBadConfig};

    // Parse configuration
    config_ = std::make_unique<JsonConfigurationClass>(args.cfg_path_);
    if (config_->IsValid()) {
      member_config_ = std::make_unique<SomeipdMemberClass>(*config_);
      try {
        someip_daemon_ = std::make_unique<SomeIpDaemonClass>(*member_config_);
        someipd_reactor_thread_ =
            std::make_unique<ReactorThreadClass>(member_config_->reactor_, member_config_->timer_manager_);

        if (args.enable_em_) {
          someipd_state_listener_ = std::make_shared<RealStateListener>();
        } else {
          someipd_state_listener_ = std::make_shared<DummyStateListener>();
        }

        someip_daemon_->Initialize();

        logger.LogInfo([](ara::log::LogStream& s) { s << "Starting reactor thread."; }, __func__, __LINE__);
        // PTP-B-SomeIpDaemon-SomeIpDaemonMain_StartReactor
        someipd_reactor_thread_->StartReactor();
        // PTP-E-SomeIpDaemon-SomeIpDaemonMain_StartReactor
        someipd_state_listener_->Notify(state_listener::SomeIpdState::kRunning);

        // Add dummy result to indicate success
        result.EmplaceValue();
      } catch (std::runtime_error const& e) {
        result.EmplaceError(InitializationError::kNotOk);
      }
    }
    return result;
  }
```

在  **Initialize**  ()中创建多个对象，让  **SomeIpDaemonClass**  的那些成员指针指向它们

**JsonConfigurationClass**  类在构造的时候就解析了  **etc/someipd-posix.json**  配置，

然后根据配置创建  **SomeipdMemberClass**  对象（初始化列表）

再把创建的  **SomeipdMemberClass**  传入  **SomeIpDaemonClass**  构造函数，创建出  **SomeIpDaemonClass（**  实际传入的模板参数为  **SomeIpd，**  看起来是用于管理  **SomeipdMemberClass**  对象的  **）**  对象，

然后根据：  **SomeipdMemberClass**  对象的  **reactor_**  成员、  **timer_manager_**  成员来初始化

接着根据入参（是否启动EM）判断  **SomeIpdListener**  成员指针指向的对象，若为  **true**  ，创建  **RealStateListener**  对象（实质是  **DummyStateListener**  类）

然后调用  **SomeIpd**  对象的  **Initialize**  ()：

首先调用成员  **ServiceDiscovery**  对象的  **Initialize**  ()

而  **ServiceDiscovery**  类采用了接口和实现分离：

其内部实现调用了  **DynamicServiceDiscovery**  类的  **Initialize**  ()函数：

**CreateSdEndpoints**  ();    // 创建service discovery endpoint

**CreateServerServiceInstanceStateMachines**  ();

**CreateClientServiceInstanceStateMachines**  ();

**StartServiceInstanceStateMachines**  ();



再调用成员  **ApplicationManager**  对象的  **Listen**  (  **UnicastAddress**  )

```
  /*!
   * \brief initialize the SomeIP Deamon an notify the execution manager about someip daemon state.
   */
  void Initialize() {
    service_discovery_.Initialize();
    application_manager_.Listen(
        osabstraction::io::ipc1::UnicastAddress{config_.GetSomeIpdIpcDomain(), config_.GetSomeIpdIpcPort()});
  }
```

分别调用  **ServiceDiscoveryInterface**  对象的  **Initialize**  ()、  **ApplicationManager**  对象的  **Listen**  ()方法

#### **ServiceDiscoveryInterface**  对象的  **Initialize**  ()方法

创建了  **service discovery endpoints**  、以及  **server**  和  **client service instance**  的  **状态机**  ，并启动它们

```
  /*!
   * \brief Initializes connection manager, endpoints and state machines.
   *
   * \context Network
   * \reentrant FALSE
   *
   * \internal
   * - Create service discovery endpoints.
   * - Create server service instance state machines.
   * - Create client service instance state machines.
   * - Start service instance state machines.
   * \endinternal
   */
  void Initialize() override {
    logger_.LogDebug([](ara::log::LogStream& s) { s << "Static SD is disabled"; }, __func__, __LINE__);

    // Create service discovery endpoints and state machines.
    // PTP-B-SomeIpDaemon-DynamicServiceDiscovery_CreateSdEndpoints
    CreateSdEndpoints();
    // PTP-E-SomeIpDaemon-DynamicServiceDiscovery_CreateSdEndpoints
    // PTP-B-SomeIpDaemon-DynamicServiceDiscovery_CreateServerServiceInstanceStateMachines
    CreateServerServiceInstanceStateMachines();
    // PTP-E-SomeIpDaemon-DynamicServiceDiscovery_CreateServerServiceInstanceStateMachines
    // PTP-B-SomeIpDaemon-DynamicServiceDiscovery_CreateClientServiceInstanceStateMachines
    CreateClientServiceInstanceStateMachines();
    // PTP-E-SomeIpDaemon-DynamicServiceDiscovery_CreateClientServiceInstanceStateMachines
    // PTP-B-SomeIpDaemon-DynamicServiceDiscovery_StartServiceInstanceStateMachines
    StartServiceInstanceStateMachines();
    // PTP-E-SomeIpDaemon-DynamicServiceDiscovery_StartServiceInstanceStateMachines
  }
```

创建了  **service discovery endpoints**

遍历config的  **NetworkEndpoints**  （config是把各个app的someip config.json合起来了，还有默认IPC配置：domain和port都是42，然后NetworkEndpoint就是特指network endpoint字段，而本例的接收/发送端采用了不同的port（从端口号看出来的区别））

**填充了sd_endpoints_**  数组的内容，因为本例配置采用的是相同的sd endpoint，所以只有一个元素

然后调用  **CreateCyclicTimers**  ();    目的是为每个  **sd endpoint**  创建循环计时器。其内部实现会把  **cyclic_offer_delay**  作为参数。

调用  **CreateRepetitionOfferTimers**  ();    目的是为了每个  **SD endpoint**  创建reptition offer timers。



创建了  **server**  和  **client service instance**  的  **状态机**

#### CreateServerServiceInstanceStateMachines();    // 创建sd server service instance

遍历provided service instances，再遍历其中的所有machine mapping，

一个provided service instance配备一个单独的state machine，创建SchedulerInterface()对象

调用  **CreateServerStateMachine**  ()

得到通信模式（  **kSdAndCommunication**  ）

通过  **MakeRemoteClient**  ()创建一个remote client，是  **RemoteClient**  <  **DefaultTemplateConfiguration**  >类型

并将其作为  **MakeServerServiceInstanceStateMachine**  ()的参数，该函数创建了  **ServiceDiscoveryServer**  对象，其构造函数入参包括：

service_id, instance_id, major_version, minor_version, address, timer_manager, message_scheduler, config

其实际上是state_machine::server::  **ServiceDiscoveryServer**  <  **RemoteClientSharedPtrType**  >类型的参数



如果不是该通信模式，那么就调用无remote client版本的  **MakeServerServiceInstanceStateMachine**  ()函数

PS：此处涉及到  **DynamicServiceDiscovery**  的模板参数  **TemplateConfiguration**  ，可以是  **DefaultTemplateConfiguration**  ，其内部是一些默认的别名

然后往std::multimap<someip_protocol::internal::  **IpAddress**  ,   **ServiceDiscoveryServiceInstanceStateMachinePtr**  >这个map里插入当前记录<  **address**  ,      **server_state_machine**  >



#### CreateClientServiceInstanceStateMachines();

特殊之处：endpoint，  **OnSocketRequired**  ()：用于监听多播地址（在客户端这边，需要立即启动以防错过任何offer）。



#### StartServiceInstanceStateMachines();

```
  /*!
   * \brief Start service discovery client and server service instance state machines.
   *
   * \context Network
   * \reentrant FALSE
   *
   * \internal
   * - Start all server state machines.
   * - Start all client state machines.
   * \endinternal
   */
  virtual void StartServiceInstanceStateMachines() {
    {
      // Start server state machines
      typename ServiceDiscoveryServiceInstanceServerStateMachineContainer::iterator const kBegin{
          sd_server_service_instance_state_machines_.begin()};

      typename ServiceDiscoveryServiceInstanceServerStateMachineContainer::iterator const kEnd{
          sd_server_service_instance_state_machines_.end()};

      static_cast<void>(std::for_each(
          kBegin, kEnd,
          [](typename ServiceDiscoveryServiceInstanceServerStateMachineContainer::reference state_machine_ptr) {
            state_machine_ptr->OnStart();
            state_machine_ptr->OnNetworkUp();
          }));
    }
```

遍历了所有的state_machine::server::  **ServiceDiscoveryServer**  <RemoteClientSharedPtrType>对象，执行一个匿名函数，做了以下操作：

            state_machine_ptr->  **OnStart**  ();    // 内部实现是打印一行Log

            state_machine_ptr->  **OnNetworkUp**  ();    // 内部实现是打印一行Log，并调用  **ServiceDiscoveryServerStateOwner**  类型的成员state_owner_的  **OnNetworkUp**  ()方法。



#### **ApplicationManager**  对象的  **Listen**  ()方法

**Listen**  ()传入的参数是：  **UnicastAddress**  类型的对象

用于监听指定地址到来的IPC连接



内部实现：

判断当前state_，如果为  **kStopped，**  而  **emplace_acceptor**  标志位为true

传入ara::core::  **Optional**  <  **ConnectionAcceptor**  >类型的成员

调用  **ConnectionAcceptor**  对象的  **Listen**  ()方法

入参是匿名函数：

**OnAccept**  (std::move(application_connection)    // 当潜在IPC通信向server建立连接时调用

其中调用了  **CreateApplication**  (std::move(handle))    // 创建  **ApplicationType**  的实例，并传给新接收的IPC连接，其中的入参handle代表新到来的IPC连接

调用  **ApplicationAcceptor**  对象的  **Listen**  ()方法：接收连接，开始监听，当新连接接收时，调用入参的函数



```
  /*!
   * \brief Start listening for incoming IPC connections on the given address.
   * \param[in] address The IPC address of the SOME/IP daemon.
   * \param[in] emplace_acceptor Invoke the acceptor constructor before calling its Listen() API.
   * \pre Listen has never been called before.
   * \context Init
   * \reentrant FALSE
   *
   * \internal
   * - If the manager is in a stopped state.
   *   - If the emplace_acceptor is set to true.
   *     - Create a new connection acceptor with the given address.
   *   - Start listening to the connection acceptor.
   *   - Change to a listening state.
   * \endinternal
   */
  void Listen(UnicastAddress const& address, bool emplace_acceptor = true) {
    logger_.LogVerbose(
        [&address](ara::log::LogStream& s) {
          s << "Address (Domain: 0x" << ara::log::HexFormat(address.domain_) << ", Port: 0x"
            << ara::log::HexFormat(address.port_) << ")";
        },
        __func__, __LINE__);
    if (state_ == AppManState::kStopped) {
      if (emplace_acceptor) {
        connection_acceptor_.emplace(reactor_, address, media_type_info_);
      }
      connection_acceptor_->Listen([this](std::unique_ptr<ApplicationConnection> application_connection) {
        OnAccept(std::move(application_connection));
      });
      state_ = AppManState::kListening;
      logger_.LogInfo(
          [&address](ara::log::LogStream& s) {
            s << "Start accepting application connections from IPC Address (Domain: 0x"
              << ara::log::HexFormat(address.domain_) << ", Port: 0x" << ara::log::HexFormat(address.port_) << ")";
          },
          __func__, __LINE__);
    }
  }
```

其中的  **ApplicationConnection**  类：

 Manage connection, message reception and transmission using "BasicIPC".

用于管理连接，消息的接收，以及使用  **BasicIPC**  进行的传输。







### ReactorThreadClass的StartReactor()

```cpp
/*!
 * \internal
 * - Construct an object of reactor thread.
 *  - While the flag is set to keep the reactor thread active.
 *    - Get pair of bool(if valid NextExpiry) and timeval struct for the next expiring timer.
 *    - If NextExpiry is true, ergo valid AND the expiry time is less than maximum allowed value.
 *      - Calculate timeout value.
 *      - Call HandleEvents with the calculated timeout value.
 *    - Else checks whether some events are pending on any registered NativeHandle and dispatches the corresponding
 *      callbacks.
 *  - Call the Callback function inside the timer manager to trigger firing timers.
 * - Construct a ara::result to set the name for the reactor thread.
 *  - Log whether the construction of the thread name was successful or not.
 * \endinternal
 */
void ReactorThread::StartReactor() noexcept {
  using vac::container::literals::operator""_sv;
  vac::container::CStringView const thread_name_{"vSomeipdReactor"_sv};

  reactor_thread_.emplace([this]() {
    logger_.LogDebug([](ara::log::LogStream &s) { s << "Reactor thread started."; }, __func__, __LINE__);

    while (!reactor_done_) {
      // PTP-B-SomeIpDaemon-ReactorThread_GetNextExpiry
      std::pair<bool, struct timeval> const expiry{timer_manager_.GetNextExpiry()};
      // PTP-E-SomeIpDaemon-ReactorThread_GetNextExpiry
      if ((expiry.first) &&
          // Conversion must not lead to overflow. In case of overflow we should take the maximum allowed value.
          (std::chrono::duration_cast<std::chrono::seconds>(std::chrono::milliseconds::max()) >
           std::chrono::seconds{expiry.second.tv_sec})) {
        // Calculate timeout value
        std::chrono::milliseconds const timeout{
            std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::seconds{expiry.second.tv_sec}) +
            std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::microseconds{expiry.second.tv_usec})};
        // PTP-B-SomeIpDaemon-ReactorThread_HandleEvents
        static_cast<void>(reactor_.HandleEvents(timeout));
        // PTP-E-SomeIpDaemon-ReactorThread_HandleEvents
      } else {
        // PTP-B-SomeIpDaemon-ReactorThread_HandleEvents
        static_cast<void>(reactor_.HandleEvents(std::chrono::milliseconds::max()));
        // PTP-E-SomeIpDaemon-ReactorThread_HandleEvents
      }
      // PTP-B-SomeIpDaemon-ReactorThread_HandleTimerExpiry
      timer_manager_.HandleTimerExpiry();
      // PTP-E-SomeIpDaemon-ReactorThread_HandleTimerExpiry
    }

    logger_.LogDebug([](ara::log::LogStream &s) { s << "Reactor thread stopped."; }, __func__, __LINE__);
  });

  osabstraction::thread::Result const result{
      osabstraction::thread::SetNameOfThread(reactor_thread_->native_handle(), thread_name_)};

  if (!result.HasValue()) {
    logger_.LogError([](ara::log::LogStream &s) { s << "Unable to set name of reactor thread."; }, __func__, __LINE__);
  }
}
```

在此处，真正实例化了一个线程，

  `  reactor_thread_.emplace([this]() {...}`  （在后面SetNameOfThread才真正给线程改名Reactor）

负责处理后续逻辑：HandleEvents，而主线程的控制流就分开了（阻塞在Signal处理函数）



**HandleEvents**  ()方法用于等待events，事件入参传入  **epoll_wait**  ()系统调用

**epoll_wait**  ()的入参：

** **  1  **、int    **  epoll实例的handle，可用于监控多个fd是否有I/O

 2  **、struct epoll_event**  数组的首地址

 3、  **struct epoll_event**  元素个数

 4、  **int    **  返回epoll wait时间



#### 关于如何使用epoll处理event

当event事件到来，epoll_wait退出，通过  **epoll_events_[i]**  取得事件，调用  **HandleOneEvent**  ()开始处理

```
/*!
 * \internal
 * - Enter exclusive area of callback entry.
 * - If the callback entry is registered to the reactor and the callback is registered for at least some of the
 *     received events.
 *   - The callback should be called now. Mark it as currently executing.
 * - Leave exclusive area of callback entry.
 * - If the callback should be called now
 *   - Call the callback only with received events that it is currently also registered for.
 *   - Enter exclusive area of callback entry.
 *   - If the file descriptor of the callback should be closed
 *     - Close the file descriptor and reset the flag that signals this.
 *   - Mark the callback as not currently executing.
 *   - Leave exclusive area of callback entry.
 * \endinternal
 */
void Reactor1::HandleOneEvent(CallbackHandle callback_handle, struct epoll_event events) noexcept {
  bool call_callback{false};
  EventTypes events_to_report{};
  internal::CallbackHandleStruct const handle{internal::UnpackCallbackHandle(callback_handle)};
  internal::CallbackEntry& entry{callbacks_[handle.index_]};

  {
    std::lock_guard<std::mutex> const entry_lock{entry.mutex_};
    if (IsHandleOfCallback(handle, entry) && entry.valid_) {
      events_to_report = GetEventsToReport(events, entry.registered_events_);
    }
    if (events_to_report.HasAnyEvent()) {
      call_callback = true;
      entry.in_callback_ = true;
    }
  }
  // The callback entry mutex is released to avoid the callback being locked while the callback is executing as this
  // would prevent calling Reactor1 API functions on this callback from the callback.
  if (call_callback) {
    entry.callback_(std::move(callback_handle), std::move(events_to_report));

    std::lock_guard<std::mutex> const entry_lock{entry.mutex_};
    if (close_current_callback_fd_) {
      static_cast<void>(close(entry.io_source_));
      close_current_callback_fd_ = false;
    }
    entry.in_callback_ = false;
  }
}
```

检测callback是否注册到reactor，以及是否为已经收到的事件注册，







**事件何时注册？**



















### SomeIpdListener的Notify(State)

**SomeIpdListener**  类型的对象

```
  /*!
   * \brief         Report state to the execution manager.
   * \param[in]     currentState the state to report.
   * \pre           -
   * \context       ANY
   * \threadsafe    FALSE
   * \reentrant     FALSE
   * \vprivate      Vector component internal API
   *
   * \internal
   * - Construct a default application client return type as kGeneralError.
   * - Store the return status when invoking ReportApplicationState of the AppClientDependency.
   * - Log a message whether the current state has been successfully reported, otherwise log an error.
   * \endinternal
   */
  void Notify(const state_listener::SomeIpdState currentState) final {
    ApplicationReturnType appClientReturnType{ApplicationReturnType::kGeneralError};

    switch (currentState) {
      case SomeIpdState::kRunning: {
        appClientReturnType = app_client_.ReportApplicationState(ApplicationState::kRunning);
        break;
      }
      case SomeIpdState::kTerminating: {
        appClientReturnType = app_client_.ReportApplicationState(ApplicationState::kTerminating);
        break;
      }
      default:
        logger_.LogError([](ara::log::LogStream& s) { s << "Undefined state encountered"; }, __func__, __LINE__);
        break;
    }

    switch (appClientReturnType) {
      case ApplicationReturnType::kGeneralError: {
        logger_.LogError([](ara::log::LogStream& s) { s << "Application Return Type: General Error"; }, __func__,
                         __LINE__);
        break;
      }
      case ApplicationReturnType::kSuccess: {
        logger_.LogDebug([](ara::log::LogStream& s) { s << "Application Return Type: Success"; }, __func__, __LINE__);
        break;
      }
      default:
        logger_.LogError([](ara::log::LogStream& s) { s << "Undefined application client return type encountered"; },
                         __func__, __LINE__);
        break;
    }
  }
```

目的是根据  **SomeIpdState**  向EM汇报  **ApplicationState**







## APP如何发送数据
![image.png](https://atlas.pingcode.com/files/public/65462c693a27284c5ca12872/origin-url)
发送数据前，计算header size、payload size，
```cpp
  /*!
   * \brief Calculate the size of the required packet header in case E2E protection is used for for a Pdu event.
   * \tparam E2eProfileConfig E2E profile configuration used to enable this API in case the event is E2E
   * protected.
   * \tparam EventConfig Event configuration used to enable this API in case the event is PDU serialized.
   * \return Size of the packet header in bytes.
   *
   * \context App
   * \threadsafe FALSE
   * \reentrant FALSE
   * \synchronous TRUE
   */
  template <typename T1 = E2eProfileConfig, typename T2 = EventConfig>
  auto CalculateHeaderSize()
      -> std::enable_if_t<!std::is_void<T1>::value &&
                              (T2::kMessageType == ::amsr::someipd_app_protocol::internal::MessageType::kPdu),
                          std::size_t> {
    // PDU header and E2E header for a E2E protected event.
    std::size_t const e2e_header_size{e2e_transformer_.value().GetHeaderSize()};
    std::size_t const header_size{::amsr::someip_protocol::internal::kPduHeaderSize + e2e_header_size};
    return header_size;
  }
```

总体  **header**  为两部分：

**kPduHeaderSize（ServiceId(2) **  +   **MethodId(2) **  +   **LengthField(4)）= 8**
**e2e_header_size **  =   **8（Profile和另一个值与运算）**


PayloadSerializer::  **GetRequiredBufferSize**  (data)

得到  **payload**  的长度


而  **alloc_size**  就是二者之和

![image.png](https://atlas.pingcode.com/files/public/654634d93a27284c5ca1288a/origin-url)

此处填充SOME/IP头部


```cpp
  /*!
   * \brief     Send event sample.
   * \param[in] data The event sample value to be transmitted.
   *
   * \context      App
   * \threadsafe   FALSE
   * \reentrant    FALSE
   * \synchronous  TRUE
   *
   * \internal
   * - Calculate size of headers and payload required for memory allocation.
   * - Allocate memory for the packet headers and payload.
   * - Instantiate a writer wrapper for the allocated memory and serialize headers and payload into the buffer.
   * - Forward serialized packet including the instance ID to the ServerManager for further transmission handling.
   * \endinternal
   */
  void Send(SampleType const& data) {
    // Allocate required memory size
    std::size_t const header_size{CalculateHeaderSize()};
    std::size_t const payload_size{PayloadSerializer::GetRequiredBufferSize(data)};
    std::size_t const alloc_size{header_size + payload_size};

    // Allocate memory for the serialization
    MemoryBufferPtr packet{tx_buffer_allocator_.Allocate(alloc_size)};

    // Get linear access to allocated MemoryBuffer via Writer
    // Due to the limitation that only flexible memory with a single view is used we can safely cast.
    MemoryBuffer::MemoryBufferView packet_view{packet->GetView(0)};
    // VECTOR Next Line AutosarC++17_10-M5.2.8:MD_SOMEIPBINDING_AutosarC++17_10-M5.2.8_conv_from_voidp
    BufferView body_view{static_cast<std::uint8_t*>(const_cast<void*>(packet_view[0].base_pointer)), packet->size()};
    Writer writer{body_view};

    // Serialize the headers + payload
    Serialize(writer, body_view, payload_size, data);

    // Finally transmit the serialized packet via ServerManager
    if (EventConfig::kMessageType == ::amsr::someipd_app_protocol::internal::MessageType::kPdu) {
      someip_binding_server_manager_.SendPduEventNotification(instance_id_, std::move(packet));
    } else {
      someip_binding_server_manager_.SendEventNotification(instance_id_, std::move(packet));
    }
  }
```

分配空间，实例化someip的writer，调用  **Serialize**  ()，然后调用  **AraComSomeIpBindingServerManagerInterface**  类对象的  **SendEventNotification**  (instance_id_, std::move(packet))方法将数据发送

其中调用了  **SomeIpDaemonClient**  类对象的  **Send**  (instance_id_, std::move(packet))

`Send`内部实现：
```cpp
  /* ---- Routing channel API -------------------------------------------------------------------------------------- */

  /*!
   * \copydoc amsr::someip_daemon_client::internal::ActiveConnection::SendSomeIpMessage
   * \pre Start() has been called.
   * \synchronous TRUE
   */
  bool Send(amsr::someip_protocol::internal::InstanceId instance_id,
            vac::memory::MemoryBufferPtr<osabstraction::io::MutableIOBuffer> packet) {
    CheckIsRunning();
    return routing_controller_.SendSomeIpMessage(instance_id, std::move(packet));
  }

```

可见，调用了  **RoutingController**  类对象的  **SendSomeIpMessage**  (instance_id, std::move(packet))，其内部实现

```cpp
  /*!
   * \brief     Initiates the transmission of a SomeIp routing message.
   * \details   The method may return before the given message has been transmitted.
   *            Outgoing routing messages might be queued.
   * \param[in] instance_id SOME/IP service instance identifier.
   * \param[in] packet A memory buffer containing a routing message.
   * \return    True if the operation has been successful and false otherwise.
   * \pre       -
   * \context   ANY
   * \reentrant FALSE
   *
   * \internal
   * - Set the result (return value of the function) to false.
   * - Enter the critical section.
   * - Check if the connection has been established.
   *   - Enqueues a routing message for transmission with the passed instance id and the packet.
   *   - Update the result to true.
   * - Otherwise, log an error that the connection is disconnected.
   * - Leave the critical section.
   * - Return the result.
   * \endinternal
   */
  bool SendSomeIpMessage(amsr::someip_protocol::internal::InstanceId instance_id,
                         vac::memory::MemoryBufferPtr<osabstraction::io::MutableIOBuffer> packet) {
    logger_.LogDebug(
        [&instance_id, &packet](ara::log::LogStream& s) {
          ara::log::LogHex16 const instance_id_hex{ara::log::HexFormat(instance_id)};
          vac::memory::MemoryBuffer<osabstraction::io::MutableIOBuffer>::size_type const memory_buffer_size{
              (packet != nullptr) ? packet->size() : 0};

          s << "Instance_id 0x" << instance_id_hex << " memory buffer length " << memory_buffer_size;
        },
        __func__, __LINE__);
    bool result{false};
    std::lock_guard<std::mutex> const lock_guard{lock_};
    if (connection_state_ == ConnectionState::kConnected) {
      EnqueueSomeIpMessage(instance_id, std::move(packet));
      result = true;
    } else {
      logger_.LogError([](ara::log::LogStream& s) { s << "Trying to send a SOME/IP message in disconnected state"; },
                       __func__, __LINE__);
    }
    return result;
  }
```

将SOME/IP消息入队，
```cpp
  /*!
   * \brief Enqueues a routing message for transmission.
   * \param[in] instance_id SOME/IP service instance identifier.
   * \param[in,out] memory_buffer A memory buffer containing a routing message.
   * \pre -
   * \context ANY
   * \reentrant FALSE
   *
   * \internal
   * - Emplace the passed memory buffer to the transmit queue.
   * - Check if the transmit queue has only one message, schedule the next transmission.
   * \endinternal
   */
  void EnqueueSomeIpMessage(amsr::someip_protocol::internal::InstanceId const instance_id,
                            vac::memory::MemoryBufferPtr<osabstraction::io::MutableIOBuffer> memory_buffer) {
    transmit_queue_.emplace_back(instance_id, std::move(memory_buffer));
    // Only schedule the next transmission if the transmit queue was empty before
    if (transmit_queue_.size() == 1U) {
      TransmitNextMessage();
    }
  }
```

根据入参创建  **TransmitQueueEntry**  对象，然后入队，

如果传输队列只有一个元素（说明之前传输队列为空），就调用  **TransmitNextMessage**  ()进行下一次传输

**TransmitNextMessage()**
```cpp
  /*!
   * \brief Starts transmission of all messages waiting in the transmit queue.
   * \details Transmit queue is emptied as long as the latest transmission was done immediately.
   * \pre -
   * \context ANY
   * \reentrant FALSE
   *
   * \internal
   * - Keep sending messages as long as queue is filled and latest message was send immediately
   *   - Only send if we are connected
   *     - Go for next packet to send
   *     - On immediate sent message, delete element in queue
   *   - On disconnected state, leave sending loop
   * \endinternal
   */
  void TransmitNextMessage() {
    ActiveConnectionTransmissionState tx_state_result{ActiveConnectionTransmissionState::kImmediate};

    bool is_queue_empty{transmit_queue_.empty()};
    bool is_connected{connection_state_ == ConnectionState::kConnected};
    bool is_tx_possible{(!is_queue_empty) && (is_connected)};

    // Keep sending messages as long as queue is filled and latest message was sent immediately
    while (is_tx_possible) {
      // Go for next packet to send
      typename TransmitQueue::const_iterator const it{transmit_queue_.cbegin()};
      TransmitQueueEntry const& transmit_queue_entry{*it};
      bool const is_control_message{transmit_queue_entry.IsControlMessage()};
      if (is_control_message) {
        tx_state_result =
            TransmitControlCommand(transmit_queue_entry.GetMessageType(), transmit_queue_entry.GetMemoryBuffer().get());
      } else if (transmit_queue_entry.IsRoutingPduMessage()) {
        tx_state_result =
            TransmitPduMessage(transmit_queue_entry.GetInstanceId(), transmit_queue_entry.GetMemoryBuffer().get());
      } else {
        tx_state_result =
            TransmitSomeIpMessage(transmit_queue_entry.GetInstanceId(), transmit_queue_entry.GetMemoryBuffer().get());
      }
      // On immediate sent message, delete element in queue
      if (tx_state_result == ActiveConnectionTransmissionState::kImmediate) {
        static_cast<void>(transmit_queue_.erase(it));
      }

      is_queue_empty = transmit_queue_.empty();
      is_connected = connection_state_ == ConnectionState::kConnected;
      is_tx_possible =
          (!is_queue_empty) && (tx_state_result == ActiveConnectionTransmissionState::kImmediate) && is_connected;
    }
  }
```



传输SOME/IP非控制消息，调用  **ActiveConnection**  类对象的以下函数
```cpp
  /*!
   * \brief Initiates the transmission of a SOME/IP routing message for transmission.
   * \param[in] instance_id SOME/IP service instance identifier.
   * \param[in] memory_buffer A memory buffer containing a routing message.
   * \return kImmediate if transmission was done immediately, kDeferred if transmission is deferred of underlaying layer
   * or otherwise kError.
   * \pre -
   * \context ANY
   * \reentrant FALSE
   *
   * \internal
   * - Configures the generic header for transmission.
   * - Configures the someip header for transmission.
   * - Create the I/O vector container view.
   * - Transmit prepared message over active connection.
   * \endinternal
   */
  ActiveConnectionTransmissionState TransmitSomeIpMessage(
      amsr::someip_protocol::internal::InstanceId const instance_id,
      vac::memory::MemoryBuffer<osabstraction::io::MutableIOBuffer> const* memory_buffer) {
    logger_.LogDebug(
        [&instance_id, &memory_buffer](ara::log::LogStream& s) {
          vac::memory::MemoryBuffer<osabstraction::io::MutableIOBuffer>::size_type const memory_buffer_size{
              (memory_buffer != nullptr) ? memory_buffer->size() : 0U};
          ara::log::LogHex16 const instance_id_hex{ara::log::HexFormat(instance_id)};

          s << "Instance_id 0x" << instance_id_hex << " memory buffer length " << memory_buffer_size;
        },
        __func__, __LINE__);

    SetUpGenericHeader(transmit_generic_header_,
                       amsr::someipd_app_protocol::internal::kRoutingSomeIpMessageHeaderLength,
                       serialization::MessageType::kSomeIp, memory_buffer);
    SetUpSomeIpHeader(transmit_someip_header_, instance_id);
    SetUpIovecContainer(transmit_iovec_container_, transmit_someip_header_, memory_buffer);

    return TransmitOverConnection();
  }
```

其中调用了  **TransmitOverConnection**  ()
```cpp
  /*!
   * \brief Transmit internal filled IOVec over active connection.
   * \pre transmit_iovec_container_ needs to be filled with a valid message to send.
   * \return kImmediate if transmission was done immediately, kDeferred if transmission is deferred of underlaying
   * layer or otherwise kError.
   * \context ANY
   * \reentrant FALSE
   *
   * \internal
   * - Create the I/O vector container view.
   * - Send the container view asynchronously.
   * - Check if the asynchronous transmission immediately completed.
   * - Check if the transmission failed.
   *   - Log a sending error.
   * \endinternal
   */
  ActiveConnectionTransmissionState TransmitOverConnection() {
    ActiveConnectionTransmissionState result{ActiveConnectionTransmissionState::kError};
    // Trigger asynchronous transmission via BasicIpc connection
    ConstIOBufferContainerView const view{transmit_iovec_container_.data(), transmit_iovec_container_.size()};
    ara::core::Result<osabstraction::io::ipc1::SendResult> send_result{
        connection_.Send(view, [this](ara::core::Result<void>&& send_completion_result) {
          OnSendCompletion(std::move(send_completion_result));
        })};

    if (send_result.HasValue()) {
      // Handle immediate message send
      if (send_result.Value() == osabstraction::io::ipc1::SendResult::kSendCompleted) {
        logger_.LogDebug([](ara::log::LogStream& s) { s << "Completing immediate message send."; }, __func__, __LINE__);
        result = ActiveConnectionTransmissionState::kImmediate;
      } else {
        logger_.LogDebug([](ara::log::LogStream& s) { s << "Message sending was deferred."; }, __func__, __LINE__);
        result = ActiveConnectionTransmissionState::kDeferred;
      }
    } else {
      logger_.LogError(
          [&send_result](ara::log::LogStream& s) {
            ::ara::core::StringView const err_str{send_result.Error().Message()};
            s << "Asynchronous message send request failed with error: " << err_str;
          },
          __func__, __LINE__);
    }
    return result;
  }
```

其中调用osabstraction::io::ipc1::  **Connection**  类型对象的了  **Send**  ()
  


```cpp
/*!
 * \internal
 *  - Check for usage errors:
 *    - Connection's native handle should not be in kClosed state.
 *    - Connection should not be in kReceiveOnly state, otherwise return with OsabErrc::kDisconnected error.
 *    - Connection should in kConnected state, otherwise return with OsabErrc::kUninitialized error.
 *  - Forward the call to Writer.
 * \endinternal
 */
auto Connection::Send(ara::core::Span<ConstIOBuffer> message, SendCompletionCallback &&callback) noexcept
    -> ara::core::Result<SendResult> {
  std::lock_guard<std::mutex> const lock{this->mutex_};
  return internal::CheckOr(this->CheckIsOpenInternal(), OsabErrc::kUninitialized)
      .AndThen([this]() {
        return internal::CheckOr(this->connection_state_ != ConnectionState::kReceiveOnly, OsabErrc::kDisconnected);
      })
      .AndThen([this]() {
        return internal::CheckOr(this->connection_state_ == ConnectionState::kConnected, OsabErrc::kUninitialized);
      })
      .AndThen([this, &message, &callback]() {
        return writer_.TryToSendMessage(message, std::move(callback), native_handle_,
                                        send_media_type_info_implemented_.GetMediaBuffer());
      });
}
```



```cpp
/*!
 * \internal
 *  - Check pre-conditions:
 *    - Number of buffers in message parameter should not exceed kMaximumNumberOfIoBuffers.
 *    - Total size of all buffers in message parameter should not exceed 32bit.
 *  - Check for usage errors:
 *    - WriterState should be kIdle.
 *  - Set WriterState to kSendingMessage.
 *  - Save Send completion callback.
 *  - Call to internal API to prepare message for sending.
 *  - If it is polling mode then try to send via SHM.
 *  - Otherwise try to send via event driven media.
 *  - If message_ has been sent in full then set WriterState back to kIdle.
 *  - Otherwise if it is event driven mode then enable continuation of write attempts by enabling write events.
 *  - Return:
 *    - SendResult::kSendCompleted if message_ has been sent in full.
 *    - SendResult::kAsyncProcessingNecessary otherwise.
 * \endinternal
 */
template <typename Handler, void (Handler::*ChangeWriteObservation)(bool)>
auto MessageWriter<Handler, ChangeWriteObservation>::TryToSendMessage(ara::core::Span<ConstIOBuffer> message,
                                                                      SendCompletionCallback&& callback,
                                                                      NativeHandle handle,
                                                                      ara::core::Optional<RingBuffer>& buffer) noexcept
    -> ara::core::Result<SendResult> {
  if (message.size() > kMaximumNumberOfIoBuffers) {
    ara::core::Abort("Span of IO buffers used in Send contains more than kMaximumNumberOfIoBuffers entries.");
  }

  if (io::internal::CalculateAlloverSize(message) > std::numeric_limits<std::uint32_t>::max()) {
    ara::core::Abort("The overall message length passed to Send must not exceed the maximum value of uint32_t");
  }

  return CheckOr(this->writing_state_ == WriterState::kIdle, OsabErrc::kBusy)
      .AndThen([this, &message, &callback, &handle, &buffer]() -> ara::core::Result<void> {
        this->writing_state_ = WriterState::kSendingMessage;

        this->send_completion_callback_ = std::move(callback);

        this->PrepareMessage(message);

        ara::core::Result<void> result{};
        // When polling mode is active, buffer will be not empty
        if (buffer.has_value()) {
          static_cast<void>(ProcessShm(buffer.value(), this->message_));
          // Polling is always successful and does not change result
        } else {
          result = WriteSome(handle);
        }
        return result;
      })
      .Map([this, buffer]() -> SendResult {
        bool const message_sent{this->message_.IsEmpty()};
        if (message_sent) {
          this->writing_state_ = WriterState::kIdle;
        } else if (!buffer.has_value()) {
          // If asynchronous processing is necesary AND pure UDS is used
          // then activate the Reactor.
          (this->handler_.*ChangeWriteObservation)(true);
        } else {
          // Make Bauhaus happy.
        }
        return message_sent ? SendResult::kSendCompleted : SendResult::kAsyncProcessingNecessary;
      });
}
```

轮询模式采用IPC



非轮询模式采用Socket
关键调用为  **sendmsg**
```cpp
/*!
 * \internal
 *  - Check pre-conditions: x
 *  - Check for usage errors: x
 *  - Prepare msghdr structure based on buffer formal parameter.
 *  - Call ::sendmsg() with native_handle formal parameter and the prepared msghdr structure.
 *  - Map the errors of ::sendmsg() call if any.
 *  - Return:
 *    - Value:
 *      - Number of sent bytes.
 *    - Errors:
 *      - See the CDD header.
 * \endinternal
 */
auto Send(NativeHandle native_handle, ara::core::Span<ConstIOBuffer> buffer) noexcept
    -> ara::core::Result<TransferredBytes> {
  ara::core::Result<TransferredBytes> result{OsabErrc::kApiError};

  // MSG_NOSIGNAL Requests not to send SIGPIPE on errors on stream oriented sockets when the other end breaks the
  // connection. An error is reported by return value and errno and not by signal.
  // VECTOR Next Line AutosarC++17_10-A3.9.1: MD_OSA_A3.9.1_PosixApi
  int const flags{MSG_NOSIGNAL};

  struct msghdr message {};

  static_assert(sizeof(struct iovec) == sizeof(ConstIOBuffer), "ConstIOBuffer is not compatible with iovec.");
  // VECTOR Next Construct AutosarC++17_10-A5.2.3: MD_OSA_A5.2.3_ConstCastForSendmsg
  // VECTOR Next Line AutosarC++17_10-A5.2.4: MD_OSA_A5.2.4_ReinterpretCastIoBufferConversion
  message.msg_iov = const_cast<struct iovec*>(reinterpret_cast<struct iovec const*>(buffer.data()));

  // osabstraction::io::kMaxIOBufferArraySize has already been checked. It depends on the IOV_MAX limit. Thus the
  // value never exceeds a supported type. The cast can be done without narrowing converion.
  static_assert(kMaxIOBufferArraySize <= std::size_t{std::numeric_limits<decltype(msghdr::msg_iovlen)>::max()},
                "osabstraction::io::kMaxIOBufferArraySize exceeds msghdr::msg_len max limit.");
  message.msg_iovlen = static_cast<decltype(msghdr::msg_iovlen)>(buffer.size());

  message.msg_flags = flags;

  // VECTOR Next Construct AutosarC++17_10-M27.0.1: MD_OSA_M27.0.1_PosixApi
  ssize_t const transferred_bytes{::sendmsg(native_handle, &message, flags)};
  // VECTOR Next Construct AutosarC++17_10-M27.0.1: MD_OSA_M27.0.1_PosixApi
  static_assert(sizeof(ssize_t) <= sizeof(std::size_t), "std::size_t can not fit ssize_t value.");

  /* Process transferred_bytes since it may designate error or number of bytes sent */
  if (transferred_bytes == kApiError) {
    result = ara::core::Result<TransferredBytes>{MapSendmsgStreamSocketError(GetErrorNumber())};
  } else {
    // Note: The value of transferred_bytes is guaranteed to be positive at this point as the only negative value that
    //       is documented for the return value of sendmsg (which is kApiError) has been checked. That means the cast
    //       to an unsigned type is safe.
    result = ara::core::Result<TransferredBytes>{TransferredBytes{static_cast<std::size_t>(transferred_bytes)}};
  }

  return result;
}
```



#### 问题记录
![image.png](https://atlas.pingcode.com/files/public/6546358a3a27284c5ca1288b/origin-url)

此处的message为什么是3？根据调用栈发现是一个元素数量为3的数组，应该是对应三个  **topic**  （代码逻辑中和  **kMaximumNumberOfIoBuffers**  (  **32**  )比较）

**ConstIOBufferContainer **  transmit_iovec_container_;






# 调用顺序    parameter server（skeleton）
skeleton侧会调用
**SomeIpDaemonClient**  类型的成员的方法：  **Connect**  ()、  **Start**  () 以及
**ClientManager**  类型的成员的方法  **StartServiceDiscovery**  ()

## InitializeCommunication(config)

![image.png](https://atlas.pingcode.com/files/public/6530ce36b3a56a8dd49b042d/origin-url)

**InitializeInternal()内部实现：**

判断Runtime实例是否存活

**InitializeReactorAndTimerManager**  (max_reactor_callback_count);

入参是“reactor要处理的最大callback数量”。

**InitializeReactorThread**  (GetReactorThreadConfig());

**InitializeBindings**  ();

调用::amsr::someip_binding_transformation_layer::internal::  **InitializeComponent**  ()

```cpp
ara::core::Result<void> InitializeComponent() noexcept {
  // Someip binding initializer/deinitializer class that holds the AraComSomeIpBinding instance
  ::amsr::someip_binding_transformation_layer::internal::SomeipBindingInitializer& initializer{
      SomeipBindingInitializer::getInstance()};

  // Register all service instances into SOME/IP Binding. Create all ara::com / Binding Transformation objects.
  // Registers all service instances into Socal.
  ara::core::Result<void> result{initializer.Initialize()};

  return result;
}
```

重点是其中的initializer的  **Initialize**  ()调用

```cpp
::ara::core::Result<void> SomeipBindingInitializer::Initialize() noexcept {
  ::ara::core::Result<void> result{::amsr::someip_binding::internal::SomeIpBindingErrc::error_not_ok};

  ::ara::com::Runtime& runtime_instance{::ara::com::Runtime::getInstance()};
  ::amsr::socal::internal::configuration::Configuration const& config{runtime_instance.GetConfig()};
  ::osabstraction::io::reactor1::Reactor1& reactor{runtime_instance.GetReactor()};
  ::vac::timer::ThreadSafeTimerManager& timer_manager{runtime_instance.GetTimerManager()};
  bool is_processing_mode_polling{runtime_instance.GetProcessingMode() ==
                                   ::amsr::socal::internal::configuration::RuntimeProcessingMode::kPolling};

  aracom_someip_binding_.emplace(config, [&runtime_instance]() { static_cast<void>(runtime_instance.ProcessPolling()); },
                                          &reactor, timer_manager, is_processing_mode_polling);

  // Register all service instances into SOME/IP Binding.
  InitializeRequiredServiceInstances();

  // Create all ara::com / Binding Transformation objects.
  InitializeServiceInterfaceProxyFactories();
  InitializeServiceInterfaceSkeletonFactories();
  InitializeProxySomeIpEventBackends();
  InitializeSkeletonSomeIpEventBackends();

  // Registers all service instances into Socal.
  RegisterServiceInstances();

  result.EmplaceValue();
  return result;
}
```

通过getXXX函数获取单例对象，存入`ara::core::Optional<SomeIpBindingType>`类型的对象aracom_someip_binding_。（SomeIpBindingType是SOME/IP binding的全局实例化）



注意，无论是接收端还是发送端，都是用同一份代码，可能是便于模板生成。  **InitializeRequiredServiceInstances**  ()的实现是根据模型配置的，在server端根本不会有实现。  **InitializeServiceInterfaceProxyFactories**  ();同理。

**InitializeServiceInterfaceSkeletonFactories**  ()内部实现：

**AraComSomeIpBindingInitializeServiceInterfaceSkeletonFactoriesparameter_service_interface**  (aracom_someip_binding_->  **GetServerManager**  ());

```cpp
void AraComSomeIpBindingInitializeServiceInterfaceSkeletonFactoriesparameter_service_interface(
    AraComSomeIpBindingSpecializationSkeleton::ServerManager& server_manager) {
  // Instantiate and register a skeleton factory for the ServiceInterface '/smart/service_interface/parameter_service_interface'.
  server_manager.AddSkeletonFactory(std::move(std::make_unique<parameter_service_interfaceSomeIpSkeletonFactory>(server_manager)));
}
```

操作：往server_manager中添加当前service interface对应的Skeleton factory。

**ServerManager**  是  **AraComSomeIpBindingServerManager**  <  **SomeIpDaemonClient**  >

（  **ServerManager**  持有SkeletonFactoryContainer对象，用于存储someip skeleton factory实例。）



**InitializeSkeletonSomeIpEventBackends**  ()的内部实现：

```cpp
 void SomeipBindingInitializer::InitializeSkeletonSomeIpEventBackends() noexcept {
  // Initialize skeleton event backends for ServiceInterface '/smart/service_interface/parameter_service_interface'
  ::amsr::someip_binding_transformation_layer::internal::parameter_service::
      AraComSomeIpBindingInitializeSkeletonSomeIpEventBackendsparameter_service_interface(aracom_someip_binding_->GetServerManager());
}
```



```cpp
void AraComSomeIpBindingInitializeSkeletonSomeIpEventBackendsparameter_service_interface(
    AraComSomeIpBindingSpecializationSkeleton::ServerManager& server_manager) {
  { // ServiceInstance: 0xC
    { // Event: ParameterNotificationEvent

      // SOME/IP Skeleton event backend type for event 'ParameterNotificationEvent'.
      using parameter_service_interfaceSkeletonSomeIpEventBackendParameterNotificationEvent = ::amsr::someip_binding_transformation_layer::internal::SomeIpSkeletonEventBackend<parameter_service_interfaceSkeletonSomeIpEventConfigurationParameterNotificationEvent>;

      std::unique_ptr<parameter_service_interfaceSkeletonSomeIpEventBackendParameterNotificationEvent> event_backend{
        std::make_unique<parameter_service_interfaceSkeletonSomeIpEventBackendParameterNotificationEvent>(12U, server_manager)};
      parameter_service::parameter_service_interfaceSkeletonSomeIpEventManagerParameterNotificationEvent::EmplaceBackend(12U, std::move(event_backend));
    }

  }
}
```

创建了  **SomeIpSkeletonEventBackend**  <  **生成的具体event结构体**  >类型的unique_ptr，

传入SomeIpSkeletonEventManager的std::map<::amsr::someip_protocol::internal::  **InstanceId**  ,  std::unique_ptr<  **EventBackend**  >>类型的event_map_。



**RegisterServiceInstances**  ();的内部实现：

```cpp
void SomeipBindingInitializer::RegisterServiceInstances() noexcept {
  ::ara::com::Runtime& runtime_instance{::ara::com::Runtime::getInstance()};
  {
    // No R-Ports configured
  }
  {
    // ---- Register all known P-Port InstanceSpecifiers ----
    {
      // Map P-Port /smart/excecutables/parameter_server/parameter_server_component/parameter_service_provided_port to instance /smart/applications/parameter_server_process/ServiceInterfaceDeployment_service_instance
      ::ara::core::InstanceSpecifier const instance_specifier{"parameter_server/RootSwComponentPrototype/parameter_service_provided_port"_sv};
      ::ara::com::InstanceIdentifier const instance_identifier{"SomeIp:12"_sv};
      runtime_instance.MapInstanceSpecifierToInstanceId(
          &(aracom_someip_binding_.value()), instance_specifier, instance_identifier,
          "/smart/service_interface/parameter_service_interface"_sv);
    }
  }
}
```

将instance名字和instance的id对应，

调用  **Runtime**  对象的  **MapInstanceSpecifierToInstanceId**  ()：

操作是把  **<instance_name, instance id>**  存入  **InstanceSpecifierLookupTable**  类型的对象（其注释说明了到底存储的是什么类型的map：<ara::core::  **InstanceSpecifier**  ,   **InstanceSpecifierLookupTableEntry**  >

**InstanceSpecifierLookupTableEntry**  ：  **InstanceIdentifier**  、  **InstanceIdentifierString**  、  **BindingInterface**  、  **ServiceShortNamePath**

在本例中：

**InstanceSpecifier**  是：  **parameter_server/RootSwComponentPrototype/parameter_service_provided_port**

（如何拼接：ROOT-SW-COMPONENT-PROTOTYPE，EXECUTABLE的port prototype的名字。）

**InstanceIdentifierString**  是：SomeIp:12

**BindingInterface**  是：someip binding对象

**service_shortname_path**  是：/smart/service_interface/parameter_service_interface

）。



**InitializeThreadPools**  ();

初始化default_thread_pool_

初始化用户定义线程池  **ThreadPoolConfigContainer**  ，用于proxy侧method方法响应。

```cpp
/*! \internal
 * - Initialize default thread-pool.
 * - Initialize user-defined thread-pools.
 * - Validate the InstanceSpecifiers configured in thread pool AssignmentConfig.
 * - Create thread-pool config for proxy method response.
 * \endinternal
 */
// VECTOR NC AutosarC++17_10-A15.5.3: MD_SOCAL_AutosarC++17_10-A15.5.3_STL_exceptions
// VECTOR NC AutosarC++17_10-A15.4.2: MD_SOCAL_AutosarC++17_10-A15.4.2_STL_exceptions
void Runtime::InitializeThreadPools() noexcept {
  logger_.LogDebug([](ara::log::LogStream const&) {}, __func__, __LINE__);

  /* Initialize default thread-pool */
  default_thread_pool_.emplace();
  static_cast<void>(default_thread_pool_->Initialize(GetDefaultThreadPoolConfig()));

  /* Initialize user-defined thread-pools */
  amsr::socal::internal::configuration::ThreadPoolConfigContainer thread_pool_configs{config_.GetThreadPools()};

  /* Validate the InstanceSpecifiers configured in thread pool AssignmentConfig */
  ValidateThreadpoolAssignmentConfig();

  /* Create thread-pool config for proxy method response */
  thread_pool_configs.emplace_back(GetProxyMethodThreadPoolConfig());
  thread_pools_.reserve(thread_pool_configs.size());
  for (const amsr::socal::internal::configuration::ThreadPoolConfig& thread_pool : thread_pool_configs) {
    logger_.LogDebug(
        [&thread_pool](ara::log::LogStream& s) {
          std::uint16_t const pool_id{static_cast<std::uint16_t>(thread_pool.GetPoolId())};

          s << "Initializing thread-pool id: " << pool_id;
        },
        __func__, __LINE__);

    /* Create new thread-pool */
    thread_pools_.emplace_back();

    /* We can call back() here directly - static_vector throws an exception if emplacing isn't possible. */
    static_cast<void>(thread_pools_.back().first.Initialize(thread_pool));

    /* Initialize thread-pool assignments */
    thread_pools_.back().second.reserve(thread_pool.GetAssignmentConfigs().size());
    for (amsr::socal::internal::configuration::AssignmentName const& name : thread_pool.GetAssignmentConfigs()) {
      thread_pools_.back().second.emplace_back(name);
    }
  }
}
```




### **StartBindings**  ();的内部实现：

连接到SOME/IP daemon，为所有需要的service instance开始Service Discovery。

```cpp
  void Start() {
    logger_.LogInfo([](::ara::log::LogStream& s) { s << "Starting SOME/IP binding."; }, __func__, __LINE__);

    // Connect to the SOME/IP daemon
    someip_daemon_client_.Connect();
    // Start the SOME/IP daemon client.
    someip_daemon_client_.Start();
    logger_.LogInfo([](::ara::log::LogStream& s) { s << "Connection to SOME/IP Daemon has been established."; },
                    __func__, __LINE__);

    client_manager_.StartServiceDiscovery();
    logger_.LogInfo([](::ara::log::LogStream& s) { s << "SOME/IP Service Discovery started."; }, __func__, __LINE__);
  }
```

调用  **SomeIpDaemonClient**  对象的几个方法：

#### **Connect()**

根据给定配置的地址，连接SOME/IP daemon。

初始化了和SOME/IP daemon的一个新连接，并一直阻塞，除非连接建立或者发生错误。（前提是SOME/IP daemon在运行）

```cpp
  void Connect() {
    logger_.LogDebug(
        [this](ara::log::LogStream& s) {
          s << "(0x" << ara::log::HexFormat(config_.someipd_ipc_domain) << ", 0x"
            << ara::log::HexFormat(config_.someipd_ipc_port) << ")";
        },
        __func__, __LINE__);
    bool const success{ipc_connection_.ConnectSync(
        osabstraction::io::ipc1::UnicastAddress{config_.someipd_ipc_domain, config_.someipd_ipc_port})};
    if (success) {
      if (ipc_connection_.IsConnectionPollingRequired() && (!is_processing_mode_polling_)) {
        // Create polling timer only if processing mode is NOT kPolling, since the IPC polling call is then
        // propagated through
        ipc_polling_timer_.emplace(timer_manager_, ipc_connection_, config_.someipd_ipc_polling_period, logger_);
        logger_.LogDebug([](ara::log::LogStream& s) { s << "Timer for BasicIPC polling registered."; }, __func__,
                         __LINE__);
      } else {
        logger_.LogDebug([](ara::log::LogStream& s) { s << "BasicIPC polling is not required."; }, __func__, __LINE__);
      }
    } else {
      char const* error_msg{"Connection establishment between SomeIpDaemonClient and SomeIpDaemon failed."};
      logger_.LogFatal([&error_msg](ara::log::LogStream& s) { s << error_msg; }, __func__, __LINE__);
      ara::core::Abort(error_msg);
    }
  }
```



如果连接尚未建立，更新连接状态为“连接中”，

建立异步连接，

1. 如果程序采用轮询模式（polling mode），则需要主动触发反应器线程（定期检查连接状态）。
1. 如果不是使用轮询模式，那么程序需要等待连接的异步建立。这意味着程序会暂停执行，直到连接成功建立为止，而不是主动轮询。


通过调试发现，该函数的入参实际上是：

osabstraction::io::ipc1::  **UnicastAddress**  类型的对象，domain和port都是42（config来源于SomeIpDaemonClient的成员  **SomeIpDaemonClientConfigModel**  对象，猜测是默认的）

```cpp
bool ConnectSync(osabstraction::io::ipc1::UnicastAddress const& address) {
    logger_.LogDebug(
        [this, &address](ara::log::LogStream& s) {
          ara::log::LogHex32 const address_domain_hex{ara::log::HexFormat(address.domain_)};
          ara::log::LogHex32 const address_port_hex{ara::log::HexFormat(address.port_)};
          ConnectionState const connection_state{connection_state_};
          ConnectionStateUnderlyingType const connection_state_select{
              static_cast<ConnectionStateUnderlyingType>(connection_state)};

          s << "Connecting to the SOME/IP daemon (BasicIPC domain: 0x" << address_domain_hex << ", port: 0x"
            << address_port_hex << ", connection state: " << connection_state_select << ")";
        },
        __func__, __LINE__);

    bool result{false};
    std::unique_lock<std::mutex> unique_lock{lock_, std::defer_lock};
    unique_lock.lock();

    if (connection_state_ == ConnectionState::kDisconnected) {
      // Update own connection state already before starting the async connection establishment.
      // This is necessary because the OnConnectCompletion callback may be executed even before we can update the state
      // after ConnectAsync returned.
      connection_state_ = ConnectionState::kConnecting;

      // Start asynchronous connection establishment
      ara::core::Result<void> const connect_result{
          connection_.ConnectAsync(address,
                                   [this](ara::core::Result<void>&& connect_complete_result) {
                                     OnConnectCompletion(std::move(connect_complete_result));
                                   },
                                   media_type_info_)};

      // Check if the asynchronous connection establishment has been started successfully.
      if (connect_result.HasValue()) {
        bool const is_polling_mode{config_model_.processing_mode == SomeIpBindingProcessingMode::kPolling};
        logger_.LogDebug(
            [this, &address, &is_polling_mode](ara::log::LogStream& s) {
              ara::log::LogHex32 const address_domain_hex{ara::log::HexFormat(address.domain_)};
              ara::log::LogHex32 const address_port_hex{ara::log::HexFormat(address.port_)};
              std::uint64_t const response_timeout_val{static_cast<std::uint64_t>(
                  std::chrono::duration_cast<std::chrono::milliseconds>(response_timeout_).count())};

              s << "Waiting for connection establishment to the SOME/IP daemon (BasicIPC domain: 0x"
                << address_domain_hex << ", port: 0x" << address_port_hex << " max timeout=" << response_timeout_val
                << "ms).";
            },
            __func__, __LINE__);

        // Wait until the connection to the SOME/IP daemon has been established.
        if (is_polling_mode) {
          // In polling mode we must wait actively by periodically triggering the reactor thread
          WaitForConnectCompletionInPollingMode(unique_lock);
        } else {
          WaitForConnectCompletionInNonPollingMode(unique_lock);
        }

        // In case of timeout, return the state to kDisconnected.
        if (connection_state_ == ConnectionState::kConnecting) {
          logger_.LogError(
              [&address, &connect_result](ara::log::LogStream& s) {
                ara::log::LogHex32 const address_domain_hex{ara::log::HexFormat(address.domain_)};
                ara::log::LogHex32 const address_port_hex{ara::log::HexFormat(address.port_)};

                s << "Timeout occured while establishing connection to the SOME/IP daemon. BasicIPC domain: 0x"
                  << address_domain_hex << ", port: 0x" << address_port_hex;
              },
              __func__, __LINE__);
          connection_state_ = ConnectionState::kDisconnected;
        }
        result = connection_state_ == ConnectionState::kConnected;
      } else {
        // ConnectAsync failed.
        connection_state_ = ConnectionState::kDisconnected;

        logger_.LogError(
            [&address, &connect_result](::ara::log::LogStream& s) {
              ara::log::LogHex32 const address_domain_hex{ara::log::HexFormat(address.domain_)};
              ara::log::LogHex32 const address_port_hex{ara::log::HexFormat(address.port_)};
              ::ara::core::StringView const err_str{connect_result.Error().Message()};

              s << "Error occurred while establishing connection to the SOME/IP daemon. BasicIPC domain: 0x"
                << address_domain_hex << ", port: 0x" << address_port_hex << ", ConnectAsync error: " << err_str;
            },
            __func__, __LINE__);
      }
    } else {
      logger_.LogError(
          [](ara::log::LogStream& s) {
            s << "Connection to SOME/IP daemon is already established or is currently being established.";
          },
          __func__, __LINE__);
    }
    return result;
  }
```

```cpp
/*!
 * \internal
 *  - Check for usage errors:
 *    - Connection's native handle should be in kClosed state.
 *    - Connection should be in kDisconnected state.
 *  - Store completion callback.
 *  - Create communication media channels.
 *  - Resolve platform specific addresses.
 *  - Store media type information.
 *  - Issue connection request using OS interface service.
 *  - Register to the Reactor with no active events.
 *  - In case of success:
 *    - Set the state of Connection's native handle to kOpen.
 *    - Set Connection's state to kConnecting.
 *    - Activate write events in the Reactor.
 *  - In case of any error:
 *    - Close the Connection's native handle if it was opened.
 *    - Set the state of Connection's native handle back to kClosed.
 * \endinternal
 */
auto Connection::ConnectAsync(const UnicastAddress remote_address, ConnectCompletionCallback &&callback,
                              const MediaTypeInfo s2c_media_type_info_external) noexcept -> ara::core::Result<void> {
  std::lock_guard<std::mutex> const lock{this->mutex_};
  internal::os_interface::ClientIndex client_index{};
  return internal::CheckOr(this->CheckIsClosedInternal(), OsabErrc::kAlreadyConnected)
      .AndThen([this]() {
        return internal::CheckOr(this->connection_state_ == ConnectionState::kDisconnected,
                                 OsabErrc::kAlreadyConnected);
      })
      .AndThen([this, &remote_address, &callback, &s2c_media_type_info_external, &client_index]() {
        this->on_connect_completion_callback_ = std::move(callback);

        client_index = internal::os_interface::ClientIndex{clients_current_count_.fetch_add(1)};

        return internal::os_interface::CreateCommMedia(remote_address, client_index,
                                                       s2c_media_type_info_external.GetType(),
                                                       s2c_media_type_info_external.GetSizeBytes());
      })
      .AndThen([this, &remote_address, &client_index](internal::TwoDirectionMediaTypeInfoImplemented media_info) {
        this->send_media_type_info_implemented_ = media_info.c2s_media_type_info_implemented;
        this->receive_media_type_info_implemented_ = media_info.s2c_media_type_info_implemented;

        internal::os_interface::UnicastAddressNative const native_remote_address{
            internal::os_interface::ResolveUnicastAddress(remote_address)};
        internal::os_interface::UnicastAddressNative const native_local_address{
            internal::os_interface::ConstructClientAddress(remote_address, client_index)};

        return internal::os_interface::Connect(native_remote_address, native_local_address);
      })
      .AndThen([this](NativeHandle handle) {
        this->native_handle_ = handle;
        return this->RegisterToReactor();
      })
      .Inspect([this]() {
        this->open_state_ = OpenState::kOpen;
        this->connection_state_ = ConnectionState::kConnecting;
        ChangeWriteObservation(true);
      })
      .InspectError([this](ara::core::ErrorCode) {
        if (this->native_handle_ != kInvalidNativeHandle) {
          // outcome of Close() is ignored, we report the original error and don't want to overwrite it.
          static_cast<void>(internal::os_interface::Close(this->native_handle_));
          this->SetStateClosed();
        }
      });
}
```

**ConnectSync**  (  **UnicastAddress**  )：

创建  **unique_lock**  <std::  **mutex**  >对象

将状态修改为  **kConnecting**  ：

调用  **connection_**  成员的  **ConnectAsync**  (  **address**  ,   **callable**  )函数开始进行异步连接：

入参  **s2c_media_type_info_external**  为  **kUds（unix domain socket，PS：还定义了其他IPC枚举：shm）**

（PS：  **AndThen**  就是连续调用）



创建native_remote_address、native_local_address

(创建了vector_basic_ipc_domain-42_port-42（socket）文件)

调用  **Connect**  (native_remote_address，native_local_address)函数

```cpp
/*!
 * \internal
 *  - Check pre-conditions: x
 *  - Check for usage errors: x
 *  - Create stream socket.
 *    - Note: Since this function creates the socket only it has a right to close it in case of an error.
 *            Socket close is done in BindAndConnect during its error handling since
 *            effectively BindAndConnect is an exclusive integral part of Connect.
 *            It should be ensured that none of other API's called from Connect closes the socket.
 *  - Define the created socket as a non-blocking one.
 *  - Bind the socket to the client_address and issue the connection request to the server_address.
 *  - Return:
 *    - Value:
 *      - A native handle to the created socket.
 *    - Errors:
 *      - See the CDD header.
 * \endinternal
 */
auto Connect(UnicastAddressNative const& server_address, UnicastAddressNative const& client_address) noexcept
    -> ara::core::Result<NativeHandle> {
  return CreateStreamSocket().AndThen(
      [&client_address, &server_address](NativeHandle native_handle) -> ara::core::Result<NativeHandle> {
        SetBlockingMode(native_handle, SocketBlockingMode{false});
        return BindAndConnect(native_handle, server_address, client_address);
      });
}
```

**CreateStreamSocket**  ()

**BindAndConnect**  (native_handle, server_address, client_address)（没细看，总之最后调用到系统调用：  **bind**  ）

最后返回相应的文件描述符



根据  **is_polling_mode**  标志判断：

如果为  **polling**  模式，则需要周期性地触发reactor thread，以活跃地等待。

（没细看）







#### **Start()**

开始消息传输

```cpp
  void Start() {
    is_running_ = true;
    // Start receive path of IPC connection
    ipc_connection_.StartReceiving();
  }
```



**StartReceiving()**

**触发对下一个package的异步接收**



```cpp
  /*!
   * \brief     Trigger an asynchronous reception.
   * \details   If message reception is not successfully completed, the error will be handled in the OnReceiveCompletion
   *            callback.
   *            If message reception was successfull the next asynchronous reception will be started immediately.
   * \pre       -
   * \context   ANY (currently called from: Init | Reactor | Callback)
   * \reentrant FALSE
   *
   * \internal
   * - Start the asynchronous reception of the next packet
   * - Log an error and disconnect from the SOME/IP daemon in case the asynchronous reception could not be started.
   * \endinternal
   */
  void StartReceiving() {
    osabstraction::io::ipc1::MessageAvailableCallback available_callback{
        [this](std::size_t const& message_size) -> MutableIOBufferContainerView {
          return OnMessageAvailable(message_size);
        }};
    osabstraction::io::ipc1::ReceiveCompletionCallback completion_callback{
        [this](ara::core::Result<std::size_t>&& receive_complete_result) {
          OnReceiveCompletion(std::move(receive_complete_result));
        }};

    // Start asynchronous reception via BasicIPC connection
    ara::core::Result<void> const receive_async_result{
        connection_.ReceiveAsync(std::move(available_callback), std::move(completion_callback))};

    if (!receive_async_result.HasValue()) {
      logger_.LogError(
          [&receive_async_result](::ara::log::LogStream& s) {
            ::ara::core::StringView const err_str{receive_async_result.Error().Message()};
            s << "An error occurred while starting an asynchronous message reception. Error message: " << err_str;
          },
          __func__, __LINE__);

      Disconnect();
    }
  }
```



**ReceiveAsync**  (MessageAvailableCallback &&  **msg_available_callback**  , ReceiveCompletionCallback &&  **receive_completion_callback**  )

调用Connection对象的MessageReader类型的成员reader_的  **ReceiveAsync**  ()方法

```cpp
/*!
 * \internal
 *  - Check for usage errors:
 *    - Connection's native handle should not be in kClosed state.
 *    - Connection should be in kConnected or in kReceiveOnly state.
 *  - Forward the call to the Reader.
 * \endinternal
 */
auto Connection::ReceiveAsync(MessageAvailableCallback &&msg_available_callback,
                              ReceiveCompletionCallback &&receive_completion_callback) noexcept
    -> ara::core::Result<void> {
  std::lock_guard<std::mutex> const lock{this->mutex_};
  return internal::CheckOr(this->CheckIsOpenInternal(), OsabErrc::kUninitialized)
      .AndThen([this]() {
        /* VECTOR Next Construct AutosarC++17_10-M5.14.1: MD_OSA_M5.14.1_EnumSideEffect */
        return internal::CheckOr((this->connection_state_ == ConnectionState::kConnected) ||
                                     (this->connection_state_ == ConnectionState::kReceiveOnly),
                                 OsabErrc::kUninitialized);
      })
      .AndThen([this, &msg_available_callback, &receive_completion_callback]() {
        return this->reader_.ReceiveAsync(std::move(msg_available_callback), std::move(receive_completion_callback));
      });
}
```



#### StartServiceDiscovery()

```cpp
/*!
   * \brief       Start Service Discovery for all registered required service instances.
   * \pre         SomeIpDaemon must be connected to the application.
   * \context     Init, App
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \vprivate
   * \synchronous TRUE
   *
   * \internal
   * - For each required service instance.
   *   - Send Request Service command to SOME/IP Daemon (this will trigger the corresponding Client SM to start).
   * \endinternal
   */
  void StartServiceDiscovery() {
    typename RequiredServiceInstanceContainer::const_iterator const it_begin{
        required_service_instance_container_.cbegin()};
    typename RequiredServiceInstanceContainer::const_iterator const it_end{required_service_instance_container_.cend()};

    static_cast<void>(std::for_each(
        it_begin, it_end,
        [this](amsr::someip_binding::internal::RequiredServiceInstanceId const& required_service_instance) {
          // Note: Having different versions is currently not yet supported. Only Service ID and Instance ID
          // are passed here.
          ::amsr::someip_daemon_client::internal::ControlMessageReturnCode const request_service_return_code{
              someip_posix_.RequestService(required_service_instance.service_id,
                                           required_service_instance.instance_id)};
          if (request_service_return_code == ::amsr::someip_daemon_client::internal::ControlMessageReturnCode::kOk) {
            logger_.LogDebug(
                [&required_service_instance](ara::log::LogStream& s) {
                  s << "SD is successfully requested for ";
                  amsr::someip_binding::internal::logging::LogBuilder::LogRequiredServiceInstanceId(
                      s, required_service_instance);
                  s << ".";
                },
                __func__, __LINE__);
          } else {
            // Error while requesting the service -> SD will not start for this service.
            logger_.LogError(
                [&required_service_instance, &request_service_return_code](ara::log::LogStream& s) {
                  s << "SD failed for ";
                  amsr::someip_binding::internal::logging::LogBuilder::LogRequiredServiceInstanceId(
                      s, required_service_instance);
                  s << ". Reason: ";
                  amsr::someip_binding::internal::logging::LogBuilder::LogControlMessageReturnCodeAsString(
                      s, request_service_return_code);
                },
                __func__, __LINE__);
          }
        }));  // ignore std::for_each return value. The returned lambda function is not required for further processing.
  }
```

遍历  **required_service_instance_container_**  （是std::  **vector**  <::amsr::someip_binding::internal::  **RequiredServiceInstanceId**  >类型），

调用  **AraComSomeIpBindingClientManager**  对象someip_posix_的  **RequestService**  (  **service_id**  ,   **instance_id**  )方法



**RequestService**  (  **service_id**  ,   **instance_id**  )

创建函数对象，其中调用  **RequiredServiceInstanceManagerType**  的  **Request**  (  **RequestedServiceInstance **  const&)方法，

然后传入  **ReactorSyncTask**  对象task，并执行，会阻塞

```cpp
/*!
   * \brief      Requests a service instance.
   * \details    This function is called on proxy instantiation.
   * \param[in]  service Service identifier to requested service.
   * \pre        StartHandlingEvents() has been called.
   * \context    App
   * \threadsafe TRUE
   * \reentrant  TRUE for different requested service instances.
   *
   * \internal
   * - Synchronize with the reactor and request the service instance.
   * - If request was  successful
   *   - Wait until a unicast connection to the remote server has been established.
   *   - If connection got disconnected, report the failure to the required instance manager.
   * \endinternal
   */
  void RequestService(RequestedServiceInstance const& service) {
    logger_.LogVerbose([](::ara::log::LogStream const&) {}, __func__, __LINE__);
    std::function<bool()> const func{[this, &service] { return RequestServiceSync(service); }};

    ReactorSyncTask<bool> task{timer_manager_, func};
    // We will block here until reactor is done with the task
    bool const result{task()};

    if (result) {
      // Request will start a connection, we need to wait here until connection is done.
      connection_manager::ConnectionState const connection_state{
          connection_manager_->WaitForConnectionEstablishment(service)};

      switch (connection_state) {
        case connection_manager::ConnectionState::kConnecting: {
          logger_.LogWarn(
              [&service](::ara::log::LogStream& s) {
                s << "Timeout while connecting to remote service instance 0x"
                  << ::ara::log::HexFormat(service.service_id_) << ":0x" << ::ara::log::HexFormat(service.instance_id_)
                  << ":0x" << ::ara::log::HexFormat(service.major_version_);
              },
              __func__, __LINE__);
          break;
        }

        case connection_manager::ConnectionState::kDisconnected: {
          required_service_instance_manager_.OnConnectionFailed(service);
          logger_.LogWarn(
              [&service](::ara::log::LogStream& s) {
                s << "Failed to connect to remote service instance 0x" << ::ara::log::HexFormat(service.service_id_)
                  << ":0x" << ::ara::log::HexFormat(service.instance_id_) << ":0x"
                  << ::ara::log::HexFormat(service.major_version_);
              },
              __func__, __LINE__);
          break;
        }

        case connection_manager::ConnectionState::kConnected: {
          logger_.LogDebug(
              [&service](::ara::log::LogStream& s) {
                s << "Already connected to remote service instance 0x" << ::ara::log::HexFormat(service.service_id_)
                  << ":0x" << ::ara::log::HexFormat(service.instance_id_) << ":0x"
                  << ::ara::log::HexFormat(service.major_version_);
              },
              __func__, __LINE__);
          break;
        }
      }
    }
  }



```

**RequestService**  最终调用到此处：

将request package序列化，然后调用  **SendControlCommand**  ()

该函数用于向SOME/IP daemon请求一个service instance

```cpp
  /*!
   * \brief Requests a service instance from the SOME/IP daemon.
   * \details After calling this function, an application must be ready to process requests destined to this service
   * instance.
   * \param[in] service_id SOME/IP service id of the Requested service.
   * \param[in] instance_id SOME/IP instance id of the Requested service.
   * \return Return code to indicate whether the request is successfully transmitted or not.
   * \error amsr::someipd_app_protocol::internal::ControlMessageReturnCode::kNotOk The error is returned if the control
   * command is not sent successfully.
   * \pre -
   * \context ANY
   * \threadsafe TRUE
   * \reentrant FALSE
   *
   * \internal
   * - Lock mutex.
   * - Initialize a SendControlCommandResult with return code kNotOk.
   * - Serialize a command control request packet with service_id and instance_id. Store returned MemoryBufferPtr.
   * - Synchronously send a RequestService command with the previously serialized request packet.
   *   Overwrite the SendControlCommandResult with the result.
   * - Return the SendControlCommandResult.
   * \endinternal
   */
  ControlMessageReturnCode RequestService(ControlMessageServiceId service_id, ControlMessageInstanceId instance_id) {
    std::lock_guard<std::mutex> const lock{command_lock_};
    SendControlCommandResult result{amsr::someipd_app_protocol::internal::ControlMessageReturnCode::kNotOk, nullptr};

    // Serialize request packet
    /* VECTOR Next Line AutosarC++17_10-A18.5.8: MD_SOMEIPDAEMONCLIENT_AutosarC++17_10-A18.5.8_use_of_smartpointer: */
    MemoryBufferPtr request_packet{SerializeServiceInstanceRequestPacket(service_id, instance_id)};

    // Synchronous command request to SOME/IP daemon
    result =
        SendControlCommand(MessageType::kRequestService, ControlMessageReturnCode::kOk,
                           amsr::someipd_app_protocol::internal::kControlMessageRequestServiceResponsePayloadLength,
                           std::move(request_packet));

    return result.first;
  }
```



#### OfferService()

如果没有提供服务，

执行field初始化检查

收集所有的成功提供服务的skeleton实例（通过查表：  **InstanceSpecifierLookupTable**  对象），更新binding实例

如果skeleton在至少一个binding中被提供，

则为拥有notifier的field发送所有初始的event

更新“service提供”的状态为true



该方法是由src-gen生成出的service interface的skeleton对象来调用的，底层实现是调用了基类：skeleton的方法，然后调用  **BindingInterface**  对象的  **OfferService**  ()方法，入参是：

1、ServiceIdentifierType（service name以及id）

2、string（instance id）

3、SkeletonInterface对象

这个部分是socal（  **Service Oriented Communication Abstraction Layer**  ，不区分具体传输协议）层的，

然后其内部实现是采用了ipc_binding里的版本，创建IpcInstanceIdentifier对象，调用  **ServerManager**  对象的  **OfferService**  (  **aracom_service_id**  ,   **ipc_instance_id**  ,   **skeleton**  )方法

```cpp
/*!
   * \brief Offer the service.
   * \pre The instance identifier must be valid and must not be malformed.
   * \pre All of the fields of the Skeleton with Setter must have a SetHandler registered.
   * \pre If SOME/IP is configured as the communication binding the application must be connected to the Daemon.
   * \pre All fields of the Skeleton must be initialized that have a notifier or a getter with a registered
   *      "GetHandler".
   * \pre IAM access must be granted.
   * \context App
   * \threadsafe FALSE
   * \reentrant TRUE for for different ServiceSkeleton class instances, FALSE otherwise.
   * \vpublic
   * \synchronous TRUE
   * \trace SPEC-4980347
   * \trace SPEC-4980348
   * \trace SPEC-4980364
   * \trace SPEC-4980365
   *
   * \internal
   * - Lock mutex.
   * - If no service has been offered
   *   - Do field initialization check.
   *   - Collect all skeleton implementations which are offered service successfully and update the binding instance
   *     implementations.
   *   - If Skeleton is offered in at least one binding
   *     - Sends all initial events for the fields that have notifiers.
   *     - Update the service offered state to true.
   *   - Otherwise, log an error for offering service failure.
   * - Otherwise, abort with an error that the service has been offered.
   *
   * Calls "ara::core::Abort()" if:
   * - Any exception derived from "std::exception" is caught.
   * - An unknown exception is caught.
   * \endinternal
   */
  void OfferService() noexcept {
    try {
      std::lock_guard<std::mutex> const guard{GetOfferServiceLock()};
      if (!service_offered_) {
        // Loop over all mapped service instances and offer them
        using SkeletonImplPtr = std::unique_ptr<SkeletonImplInterface>;
        DoFieldInitializationChecks();

        std::unique_lock<std::mutex> impl_interfaces_unique_lock{GetImplInterfacesLock()};

        for (InstanceSpecifierLookupTableEntry const& offered_instance : offered_instances_) {
          SkeletonImplPtr skeleton_implementation{offered_instance.GetBinding()->OfferService(
              kServiceIdentifier, offered_instance.GetInstanceIdentifierString(), *this)};
          if (skeleton_implementation != nullptr) {
            // Make a downcast to the concrete ServiceSkeletonImplInterface, to allow the usage of events from the
            // concrete service instance.
            assert(skeleton_implementation->GetSkeletonId() == GetSkeletonId());
            ConcreteSkeletonImplInterface* concrete_impl_interface{
                dynamic_cast<ConcreteSkeletonImplInterface*>(skeleton_implementation.release())};

            assert(concrete_impl_interface != nullptr);
            binding_instance_implementations_.emplace_back(concrete_impl_interface);
          }
        }

        if (binding_instance_implementations_.size() > 0U) {
          impl_interfaces_unique_lock.unlock();

          SendInitialFieldNotifications();

          service_offered_ = true;
        } else {
          logger_.LogError(
              [this](::ara::log::LogStream& s) {
                std::string const service_identifier{kServiceIdentifier.toString()};
                s << "Failed to offer service '" << service_identifier << "' with instance ID(s) '";
                for (InstanceSpecifierLookupTableEntry const& entry : offered_instances_) {
                  ara::com::InstanceIdentifier const instance_identifier{entry.GetInstanceIdentifier()};
                  ara::core::StringView const instance_id{instance_identifier.ToString()};
                  s << " " << instance_id;
                }
                s << "'";
              },
              __func__, __LINE__);
        }
      } else {
        logger_.LogFatal(
            [this](::ara::log::LogStream& s) {
              std::string const service_identifier{kServiceIdentifier.toString()};
              s << "'" << service_identifier << "' service with instance ID(s) '";
              for (InstanceSpecifierLookupTableEntry const& entry : offered_instances_) {
                ara::com::InstanceIdentifier const instance_identifier{entry.GetInstanceIdentifier()};
                ara::core::StringView const instance_id{instance_identifier.ToString()};
                s << " " << instance_id;
              }
              s << "' is already offered. Aborting.";
            },
            __func__, __LINE__);
        ::ara::core::Abort("Skeleton::OfferService: service already offered.");
      }
    }  // VECTOR Next Line AutosarC++17_10-A15.3.4: MD_SOCAL_AutosarC++17_10-A15.3.4_Caught_exception_is_too_general
    catch (std::exception const& e) {
      ::ara::core::Abort(e.what());
    }  // VECTOR NC AutosarC++17_10-A15.3.4: MD_SOCAL_AutosarC++17_10-A15.3.4_Using_catch_all
       // VECTOR NC AutosarC++17_10-M0.3.1: MD_SOCAL_AutosarC++17_10-M0.3.1_Dead_exception_handler
    catch (...) {
      ::ara::core::Abort("Skeleton::OfferService: Unexpected fatal error.");
    }
  }
```



**ServerManager**  对象的  **OfferService**  (  **aracom_service_id**  ,   **ipc_instance_id**  ,   **skeleton**  )方法

```cpp
  /*!
   * \brief          Offer a service over the IPC binding from a skeleton.
   * \details        Ensures uniqueness of offered services by checking all offered services in one process.
   * \param[in]      aracom_service_id the ara::com::ServiceIdentifierType of the corresponding service interface.
   * \param[in]      instance The IPC InstanceIdentifier to offer a service.
   * \param[in,out]  skeleton The related frontend skeleton which is offered.
   * \return         A SkeletonImplInterface for the given Service, configured for the given instance_identifier or
   *                 nullptr, if no such SkeletonImplInterface is available from the binding.
   * \pre            Initialization finished: Initialize() called, all skeleton factories registered.
   *
   * \context        App
   * \threadsafe     TRUE
   * \trace          SPEC-4980348
   *
   * \internal
   * - Search the related skeleton factory for the service instance to be offered.
   * - If the skeleton factory is found,
   *   - Use the factory to create a new provided service instance.
   *   - If the provided service instance is created successfully,
   *     - Offer the service instance via ServiceDiscovery.
   *     - If any error occurs during service offer/instance creation destroy the just created provided service instance
   *       and log errors.
   * \endinternal
   */
  std::unique_ptr<::amsr::socal::internal::SkeletonImplInterface> OfferService(
      ::ara::com::AraComServiceId const& aracom_service_id,
      ::amsr::ipc_binding_transformation_layer::internal::IpcInstanceIdentifier const& instance,
      ::amsr::socal::internal::SkeletonInterface& skeleton) {
    // PTP-B-IpcBinding-ServerManager_OfferService

    logger_.LogDebug([](::ara::log::LogStream const&) {}, __func__, __LINE__);
    std::unique_ptr<::amsr::socal::internal::SkeletonImplInterface> skeleton_impl{nullptr};

    // Search the skeleton factory for the service instance to be offered.
    SkeletonFactoryInterface const* skeleton_factory{FindOfferedSkeletonFactory(aracom_service_id, instance, skeleton)};
    if (skeleton_factory != nullptr) {
      ServiceDiscoveryServiceInstanceIdentifier const service_instance{
          skeleton_factory->GetIpcServiceId(), instance.GetInstanceId(), skeleton_factory->GetIpcMajorVersion(),
          skeleton_factory->GetIpcMinorVersion()};

      bool destroy_skeleton_impl{false};

      // Create a new provided service instance via factory.
      // The factory directly emplaces the new provided service instance into the skeleton_binding_instances_ map.
      skeleton_impl = skeleton_factory->create(skeleton, service_instance.instance_id_);
      if (skeleton_impl != nullptr) {
        // Once skeleton is created, it registers itself in this manager
        IpcSkeletonBindingIdentity const skeleton_identity{service_instance.service_id_, service_instance.instance_id_,
                                                           service_instance.major_version_};

        // Protect modification of skeleton_binding_instances_.
        std::lock_guard<std::mutex> const skeleton_binding_instances_guard{skeleton_binding_instances_lock_};

        // Search the provided service instance just created by the factory.
        ServiceSkeletonIpcBindingInstances::iterator const created_provided_service_instance_it{
            skeleton_binding_instances_.find(skeleton_identity)};
        if (created_provided_service_instance_it != skeleton_binding_instances_.cend()) {
          // Notify the  service discovery about the local offer.
          bool const offered{ipc_sd_.OfferLocalService(service_instance, created_provided_service_instance_it->second)};
          if (!offered) {
            logger_.LogError(
                [this, &service_instance](::ara::log::LogStream& s) {
                  s << "Could not offer service ";
                  PrettyPrintServiceInstance(s, service_instance);
                },
                __func__, __LINE__);

            // Remove the just created provided service instance from the map in case the offer via SD failed.
            static_cast<void>(skeleton_binding_instances_.erase(created_provided_service_instance_it));
            destroy_skeleton_impl = true;
          }
        } else {
          logger_.LogError(
              [this, &service_instance](::ara::log::LogStream& s) {
                s << "Could not create a skeleton IPC instance ";
                PrettyPrintServiceInstance(s, service_instance);
              },
              __func__, __LINE__);
          destroy_skeleton_impl = true;
        }
      } else {
        logger_.LogError(
            [this, &service_instance](::ara::log::LogStream& s) {
              s << "Could not create a skeleton IPC instance ";
              PrettyPrintServiceInstance(s, service_instance);
            },
            __func__, __LINE__);
      }

      // In case an error occured, the temporary allocated provided service instance must be destroyed.
      // Destruction must be done outside of skeleton_binding_instances_guard to avoid deadlocks.
      if (destroy_skeleton_impl) {
        skeleton_impl = nullptr;
      }
    }

    // PTP-E-IpcBinding-ServerManager_OfferService
    return skeleton_impl;
  }
```

通过IPC binding，从skeleton offer一个service

首先调用  **FindOfferedSkeletonFactory**  (aracom_service_id, instance, skeleton)方法得到  **SkeletonFactoryInterface**  对象，  **FindOfferedSkeletonFactory**  ()内部实现是：

遍历  **SkeletonFactoryContainer**  这一  **vector**  <  **SkeletonFactoryInterface**  >，匹配其中的skeleton id相同的元素。只要找到了，就创建  **ServiceDiscoveryServiceInstanceIdentifier**  类型的对象：入参是ipc service id、instance id、ipc major ver、ipc min ver。

遍历当前  **provided_service_instance_map_**  （是std::map<  **ServiceDiscoveryServiceInstanceIdentifier**  , ::ara::com::  **AraComServiceId**  >类型），根据  **service_identifier**  得到其中的provided service instance，

然后还要匹配，根据入参的aracom_service_id（  **ServiceIdentifierType**  类型，实质只是存了个service id的字符串），匹配到则返回  **SkeletonFactoryInterface**  类型的裸指针

（  **SkeletonIdObject***  就是  **SkeletonId、SkeletonFactoryInterface**  是抽象基类，  **ServiceDiscoveryServiceInstanceIdentifier**  类维护一个<ipc_protocol::  **ServiceId**  , ipc_protocol::  **InstanceId**  , ipc_protocol::  **MajorVersion**  , ipc_protocol::  **MinorVersion**  >元组）

（精髓在于：先根据skeleton id找到相应的  **skeleton factory**  ，然后再搜索  **service instance -- ara com service id的mapping**  ，判断ara com service id是否匹配map里的，最后返回这个  **skeleton factory，一定要跑到map里再匹配一下**  ）

得到  **skeleton_factory**  后，可以调用  **create**  (  **skeleton**  ,   **service_instance_id**  )

然后调用  **IpcServiceDiscovery**  对象的  **OfferLocalService**  (  **service_instance**  ,   **created_provided_service_instance_it**  -  **>second**  )

```cpp
  /*!
   * \brief      Offers a local service by registering the service in the packet router and
   *             informing all remote nodes by sending an offer over the multicast network.
   * \param[in]  service  Service identifier to offer.
   * \param[in]  provider Pointer to the server sink that provides the service.
   * \return     True if offered or False if this instance could not be offered.
   * \pre        StartHandlingEvents() has been called.
   *
   * \context    App
   * \threadsafe TRUE
   * \reentrant  TRUE for different offered service instances and providers.
   *
   * \internal
   * - Start offering the service from the reactor context.
   * \endinternal
   */
  bool OfferLocalService(OfferedServiceInstance const& service, LocalServerSinkPtr provider) {
    logger_.LogVerbose([](::ara::log::LogStream const&) {}, __func__, __LINE__);
    std::function<bool()> const func{[this, &service, &provider] { return OfferLocalServiceSync(service, provider); }};

    ReactorSyncTask<bool> task{timer_manager_, func};
    // We will block here until reactor is done with the task
    return task();
  }
```

其中调用了：

```cpp
  /*!
   * \brief      Offers a local service by registering the service in the packet router and
   *             informing all remote nodes by sending an offer over the multicast network.
   * \param[in]  service  Service identifier to offer.
   * \param[in]  provider Pointer to the server sink that provides the service.
   * \return     True if offered or False if this instance could not be offered.
   * \pre        StartHandlingEvents() has been called.
   * \context    Reactor
   * \reentrant  FALSE
   * \trace      SPEC-4980348
   *
   * \internal
   * - Add an entry to the ProviderSinkTable and packet router to register the offered service with a
   *   destination sink.
   * - If connection manager has been initialized
   *   - Start offering the given service remotely via multicast endpoint.
   * \endinternal
   */
  bool OfferLocalServiceSync(OfferedServiceInstance const& service, LocalServerSinkPtr provider) {
    logger_.LogVerbose([](::ara::log::LogStream const&) {}, __func__, __LINE__);

    bool start_offering_remotely{false};
    ipc_protocol::ServiceInstanceIdentifier const offered_service{service.service_id_, service.instance_id_,
                                                                  service.major_version_};

    // Add an entry to the ProviderSinkTable and packet router to register the offered service with a
    // destination sink.
    AddLocalProvidedSink(offered_service, provider);

    if (connection_manager_ != nullptr) {
      /* Try to offer the given service remotely */
      start_offering_remotely = provided_service_instance_manager_.StartOffering(service);
      if (!start_offering_remotely) {
        logger_.LogError(
            [&service](::ara::log::LogStream& s) {
              s << "Failed to offer service 0x" << ::ara::log::HexFormat(service.service_id_) << ":0x"
                << ::ara::log::HexFormat(service.instance_id_) << ":0x" << ::ara::log::HexFormat(service.major_version_)
                << ":0x" << ::ara::log::HexFormat(service.minor_version_) << " remotely";
            },
            __func__, __LINE__);
      }
    }

    return start_offering_remotely;
  }
```

```cpp
  /*!
   * \brief Register a local service with a destination provided sink.
   * \param[in] service Id of the offered service.
   * \param[in] sink The local destination sink where subscribe / unsubscribe message should be routed to
   * \pre       -
   * \context   Reactor
   * \reentrant FALSE
   *
   * \internal
   * - If the service is not registered with the local server
   *   - Register the service with the local server.
   *   - Add a request route in the packet router to this local server.
   * \endinternal
   */
  void AddLocalProvidedSink(RequestedServiceInstance service, LocalServerSinkPtr sink) {
    bool added{false};
    logger_.LogDebug(
        [&service](::ara::log::LogStream& s) {
          s << "Register with provided sink (0x" << ::ara::log::HexFormat(service.service_id_) << ":0x"
            << ::ara::log::HexFormat(service.instance_id_) << ":0x" << ::ara::log::HexFormat(service.major_version_)
            << ")";
        },
        __func__, __LINE__);

    std::pair<typename LocalServerSinkTable::iterator, bool> const ret{local_server_sink_table_.emplace(service, sink)};
    added = ret.second;
    if (added) {
      packet_router_.AddRequestRoute(service, sink);
    }
  }
```













# 调用顺序    sensor_data_service（proxy）

## InitializeCommunication(config)

和skeleton侧类似，会调用到

**SomeIpDaemonClient**  类型的成员的方法：  **Connect**  ()、  **Start**  () 以及

**ClientManager**  类型的成员的方法  **StartServiceDiscovery**  ()



初始化时调用  **Runtime**  的`InitializeInternal()`，
```cpp
/*! \internal
 * - If the Runtime instance is not alive for multi-threaded applications
 *   - Call HandleErrorNotRunning.
 * - Otherwise, Initialize the reactor thread.
 * - Instantiating the binding pool and initialize all bindings.
 * - Flag that signalizes that runtime and bindings have been initialized.
 * - Start all dynamic actions of all bindings.
 * \endinternal
 */
void Runtime::InitializeInternal() noexcept {
  if (is_running_) {
    HandleErrorAlreadyRunning();
  }

  // Maximum number of callbacks the reactor needs to handle.
  constexpr std::uint16_t max_reactor_callback_count{1024};

  // Initialize the reactor thread. Reactor required before binding initialization because bindings may initialize
  // connections (e.g. SOME/IP).
  // The real communication via the reactor must only be started after StartBindings() is called.
  InitializeReactorAndTimerManager(max_reactor_callback_count);
  InitializeReactorThread(GetReactorThreadConfig());

  // Instantiating the binding pool and initialize all bindings.
  logger_.LogDebug([](ara::log::LogStream& s) { s << "InitializeBindings"; }, __func__, __LINE__);
  InitializeBindings();

  InitializeThreadPools();

  // Flag that signalizes that runtime and bindings have been initialized.
  is_running_ = true;

  // Start all dynamic actions of all bindings (receive / transmit paths, timers etc.)
  logger_.LogDebug([](ara::log::LogStream& s) { s << "StartBindings"; }, __func__, __LINE__);
  StartBindings();
}
```

`InitializeThreadPools()`
```cpp
/*! \internal
 * - Initialize default thread-pool.
 * - Initialize user-defined thread-pools.
 * - Validate the InstanceSpecifiers configured in thread pool AssignmentConfig.
 * - Create thread-pool config for proxy method response.
 * \endinternal
 */
// VECTOR NC AutosarC++17_10-A15.5.3: MD_SOCAL_AutosarC++17_10-A15.5.3_STL_exceptions
// VECTOR NC AutosarC++17_10-A15.4.2: MD_SOCAL_AutosarC++17_10-A15.4.2_STL_exceptions
void Runtime::InitializeThreadPools() noexcept {
  logger_.LogDebug([](ara::log::LogStream const&) {}, __func__, __LINE__);

  /* Initialize default thread-pool */
  default_thread_pool_.emplace();
  static_cast<void>(default_thread_pool_->Initialize(GetDefaultThreadPoolConfig()));

  /* Initialize user-defined thread-pools */
  amsr::socal::internal::configuration::ThreadPoolConfigContainer thread_pool_configs{config_.GetThreadPools()};

  /* Validate the InstanceSpecifiers configured in thread pool AssignmentConfig */
  ValidateThreadpoolAssignmentConfig();

  /* Create thread-pool config for proxy method response */
  thread_pool_configs.emplace_back(GetProxyMethodThreadPoolConfig());
  thread_pools_.reserve(thread_pool_configs.size());
  for (const amsr::socal::internal::configuration::ThreadPoolConfig& thread_pool : thread_pool_configs) {
    logger_.LogDebug(
        [&thread_pool](ara::log::LogStream& s) {
          std::uint16_t const pool_id{static_cast<std::uint16_t>(thread_pool.GetPoolId())};

          s << "Initializing thread-pool id: " << pool_id;
        },
        __func__, __LINE__);

    /* Create new thread-pool */
    thread_pools_.emplace_back();

    /* We can call back() here directly - static_vector throws an exception if emplacing isn't possible. */
    static_cast<void>(thread_pools_.back().first.Initialize(thread_pool));

    /* Initialize thread-pool assignments */
    thread_pools_.back().second.reserve(thread_pool.GetAssignmentConfigs().size());
    for (amsr::socal::internal::configuration::AssignmentName const& name : thread_pool.GetAssignmentConfigs()) {
      thread_pools_.back().second.emplace_back(name);
    }
  }
}
```

1. `默认线程池`的初始化（调用完，线程就启动了）
名字是`vComDef`，默认构造函数是把  **State**  设置为  **kStopped**  ，  **SpawnWorkerThreads()**  中创建该thread pool中的所有  **worker thread**  （默认为1），worker thread的名字是  **pool_prefix **  +   **pool_id **  +   **index**  。
**WorkerThread**  构造的时候，调用  **Create**  ()启动线程


2. `user-defined`线程池  **container**  的创建和初始化，采用的是  **ProxyMethodThreadPool**  的全体线程池Config
（  **Runtime**  类持有一个  **StaticVector**  <  **ThreadPoolMapElement**  >对象  **thread_pools_**  ，其中  **ThreadPoolMapElement**  是<  **ThreadPool**  ,    **ServiceInstanceAssignmentContainer**  >，每有一个  **service instance**  就对应一个  **work thread**  ）

根据这个config的（写死在代码里的）配置，只创建一个线程池，并且只含有一个线程，名字是  **vComResp**  。

根据线程池中的配置，实例化线程池，并通过  **Initialize**  启动所有的worker thread，遍历当前线程池config的vector<AssignmentName>，将若干个  **assignment name**  传入当前线程池的  **ThreadPoolMapElement**

其中  **Assignment Config**  的vector只有一个元素，类型是string，"  **ProxyResponseMethod**  "
```cpp
    /* Initialize thread-pool assignments */
    thread_pools_.back().second.reserve(thread_pool.GetAssignmentConfigs().size());
    for (amsr::socal::internal::configuration::AssignmentName const& name : thread_pool.GetAssignmentConfigs()) {
      thread_pools_.back().second.emplace_back(name);
    }
```





















和skeleton相同，会调用到  **Runtime**  的  **StartBindings**  ()函数
![image.png](https://atlas.pingcode.com/files/public/65363c49b3a56a8dd49b4455/origin-url)

socal这层提供函数  **StartFindService**  ()，用于接收函数对象

```
  // VECTOR NC VectorC++-V6.6.1: MD_SOCAL_VectorC++-V6-6-1_AbortWithSingleReturnStatement
  /*!
   * \brief Start an asynchronous FindService notification about service updates.
   * \details The callback handler may be executed even before the return of this function call. Therefore calling
   *          StopFindService within the callback is then not allowed, because it requires the concrete
   *          FindServiceHandle returned from this function.
   * \param[in] handler Gets called upon detection of a matching service. Note that the handles received by this
   *            callback shall be released before the Runtime is destroyed. I.e. they cannot be stored in variables
   *            with longer life period than the application's main(). If not followed, it's not guaranteed that the
   *            communication middleware is shut down properly and may lead to segmentation fault. This param can not
   *            be nullptr. Any exception thrown by the callback will lead to a termination through "std::terminate()".
   * \param[in] instance The instance specifier for which a matching service instance should be searched.
   * \return FindServiceHandle for this search/find request, which is needed to stop the service availability
   *         monitoring and related firing of the given handler.
   * \pre SOME/IP daemon must be running (SOME/IP binding only).
   * \pre Instance specifier ( \p instance ) must be configured in the ARXML model.
   * \pre IAM access must be granted.
   * \context App
   * \threadsafe FALSE
   * \reentrant FALSE
   * \synchronous FALSE
   * \trace SPEC-4980368
   *
   * \internal
   * - Create an InstanceSpecifierLookupTableEntry from \p instance.
   * - Call ScheduleInitialSnapshotTask() with \p handler and the InstanceSpecifierLookupTableEntry.
   * - Return the FindServiceHandle that was returned by the call.
   *
   * Calls "ara::core::Abort()" if:
   * - \p handler is "nullptr".
   *
   * \endinternal
   */
  static ara::com::FindServiceHandle StartFindService(ara::com::FindServiceHandler<HandleType> handler,
                                                      ara::core::InstanceSpecifier instance) noexcept {
    if (handler == nullptr) {
      char const* error_msg{"StartFindService() called with handler=nullptr. Please use a valid handler function."};

      amsr::socal::internal::logging::AraComLogger const logger{
          amsr::socal::internal::logging::kAraComLoggerContextId,
          amsr::socal::internal::logging::kAraComLoggerContextDescription, "Proxy"};
      logger.LogFatal([&error_msg](ara::log::LogStream& s) { s << error_msg; }, __func__, __LINE__);

      ara::core::Abort(error_msg);
    }

    InstanceSpecifierLookupTableEntryContainer const service_instances{ResolveInstanceSpecifierMapping(instance)};
    return ScheduleInitialSnapshotTask(handler, service_instances);
  }
```

重点是其中的  **ScheduleInitialSnapshotTask**  (handle, service_instance)

```
  /*!
   * \brief Register a new observer and schedule execution of a new InitialSnapShot task.
   * \details The InitialSnapShot will be executed via a dedicated ThreadPool thread.
   * \param[in] handler The application callback handle to be notified by found service instances.
   * \param[in] service_instances Container of searched service instances.
   * \return A FindService handle for the StartFindService request.
   * \pre -
   * \context App
   * \threadsafe FALSE
   * \reentrant FALSE
   * \synchronous FALSE
   *
   * \internal
   * Calls "std::terminate()" if:
   *  - "std::bad_alloc" is thrown from "vac::language::make_unique".
   * \endinternal
   */
  static ara::com::FindServiceHandle ScheduleInitialSnapshotTask(
      ara::com::FindServiceHandler<HandleType> const& handler,
      InstanceSpecifierLookupTableEntryContainer const& service_instances) noexcept {
    HandleAndHandler handle{findservice_observers_manager_.AddObserver(GetProxyIdStatic(), service_instances, handler)};

    RuntimeSelected& runtime{RuntimeSelected::getInstance()};
    ThreadPool& pool = runtime.RequestThreadPoolAssignment();
    // VECTOR NC AutosarC++17_10-A0.1.2: MD_SOCAL_AutosarC++17_10-A0.1.2_ScheduleInitialSnapshotTask
    // VECTOR NC AutosarC++17_10-M0.3.2: MD_SOCAL_AutosarC++17_10-M0.3.2_ScheduleInitialSnapshotTask
    pool.AddTask(vac::language::make_unique<InitialSnapshotTask>(handle));

    return handle.GetHandle();
  }
```

将任务加入线程池。  **DynamicWork**  对象的  **Run**  ()方法会执行函数



而真正传入  **StartFindService**  ()的函数对象为（用户层生成代码：）：

```
void SensorDataService::FindService1Handler(
    ara::com::ServiceHandleContainer<parameter_service::proxy::parameter_service_interfaceProxy::HandleType>
        service1_handles) {
  if (service1_handles.empty()) {
    log_.LogInfo() << "[Service1] StartFindService handler called: no instance found.";
  } else if (service1_handles.size() == 1) {
    // In some rare cases the callback can be called again before the StopFindService call has finished.
    // In this case the already found proxy should not be overwritten, because it may be already in use.
    if (!service1_proxy_) {
      std::lock_guard<std::mutex> lock(service1_proxy_mutex_);
      log_.LogInfo() << "[Service1] StartFindService handler called: 1 instance found.";
      service1_proxy_ =
          std::make_shared<parameter_service::proxy::parameter_service_interfaceProxy>(service1_handles[0]);
      notify_on_service1_found_.notify_all();
    }
  } else {
    // The example doesn't handle this case.
    log_.LogInfo() << "[Service1] StartFindService handler called: " << service1_handles.size()
                   << " instances found. Not handled by Start Application.";
  }
}
```

其入参：  **parameter_service_interfaceHandleType**  是生成出来的代码

使用条件变量  **notify_on_service1_found_**  阻塞对于服务发现时未在1s内找到服务的判断。

其中的service1_handles在何时得到？很可能是把服务发现的操作放在线程里

**在（线程池的）线程里**  执行函数对象  **FindService1Handler**  ()

而入参是在函数void   **ExecuteFindServiceAndHandler**  (  **F**  && find_service_function) const中传入的

通过调用  **FindServiceHandleAndHandler**  对象的成员handle_的  **GetServiceInstances**  ()方法得到，是amsr::socal::internal::  **InstanceSpecifierLookupTableEntryContainer**  类型，想必是在初始化时得到















  


## FindService

在Proxy对象调用  **StartClient**  ()方法之后，

1. 调用  **parameter_service_interfaceProxy**  类对象的（继承自  **Proxy**  对象）  **StartFindService**  ()函数，传入了以下参数：  
     (1) 调用了  **FindService1Handler**  的函数对象（它会作为  **ScheduleInitialSnapshotTask**  的入参，把它包装成  **FindServiceHandleAndHandler对象**  放进  **FindServiceObserversManager**  类对象的vector<  **FindServiceHandleAndHandler**  <  **ProxyHandleType**  >>成员：  **pending_observers_**  ，通过  **AddTask()**  方法放在线程里执行）(2)   **InstanceSpecifier**


**StartFindService**  ()的重点是调用  **ScheduleInitialSnapshotTask**  (handler, service_instances)，它会请求线程池分配（没有指定的话，即使默认线程池线程  **vComDef**  去做），把  **handle**  的任务分给线程池去做。



**然后Find Service的操作就放在另一个线程去异步地执行：**

1. Find service后开始设置调用  **parameter_service_interfaceProxy**  类对象  **SubscriptionStateHandler**  、  **ReceiveHandler**
1. 然后开始调用  **parameter_service_interfaceProxy**  类对象的  **ProxyEvent**  成员的  **Subscribe**


```
ara::core::Result<void> SensorDataService::StartClient() {
  ara::core::Result<void> result{};
  if (!client_started_) {
    log_.LogInfo() << "[Service1] StartFindService for StartApplicationService1.";
    ara::com::FindServiceHandle find_service_handle =
        parameter_service::proxy::parameter_service_interfaceProxy::StartFindService(
            [this](const ara::com::ServiceHandleContainer<
                   parameter_service::proxy::parameter_service_interfaceProxy::HandleType>& handles) {
              FindService1Handler(handles);
            },
            ara::core::InstanceSpecifier{kInstanceSpecifierService1});

    {
      std::unique_lock<std::mutex> lock{service1_proxy_mutex_};
      if (!notify_on_service1_found_.wait_for(lock, std::chrono::seconds{1},
                                              [&]() { return service1_proxy_ != nullptr; })) {
        result.EmplaceError(SensorDataServiceErrc::kProxyNotFound);
        log_.LogError() << "[Service1] StartFindService did not find a proxy in the given time.";
      }
    }

    log_.LogInfo() << "[Service1] StopFindService for StartApplicationService1.";
    parameter_service::proxy::parameter_service_interfaceProxy::StopFindService(find_service_handle);

    if (result.HasValue()) {
      log_.LogInfo() << "[Service1] [Event1] Set subscription state handler for the event.";
      service1_proxy_->ParameterNotificationEvent.SetSubscriptionStateHandler(
          [this](const ara::com::SubscriptionState subscription_state) {
            SubscriptionStateHandlerService1Event1(subscription_state);
          });

      log_.LogInfo() << "[Service1] [Event1] Set receive handler for the event.";
      service1_proxy_->ParameterNotificationEvent.SetReceiveHandler([this]() { ReceiveHandlerService1Event1(); });

      log_.LogInfo() << "[Service1] [Event1] Subscribe to the event.";
      service1_proxy_->ParameterNotificationEvent.Subscribe(ara::com::EventCacheUpdatePolicy::kLastN, 1);
      client_started_ = true;
    }
  } else {
    result.EmplaceError(SensorDataServiceErrc::kClientAlreadyStarted);
  }
  return result;
}
```



而在  **vComDef0-0**  线程中，取出任务队列的任务开始执行，会调用  **（Proxy**  对象中嵌套的）  **InitialSnapshotTask对象（这个类型会持有一个HandleAndHandler**  对象  **）的operator()**  方法：

初始化find service request，并调用注册的find service handler（为所有活跃的observer）

**PS**  ：  **观察者**  （行为设计模式）模式：定义一种订阅机制，在对象发生时通知多个观察该对象的其他对象

其中调用了handle_的  **ExecuteFindServiceAndHandler()方法**

```
 /*!
     * \brief Initiates a find service request and calls the registered find service handler for all active observers.
     * \pre           -
     * \context       Callback
     * \threadsafe    FALSE
     * \reentrant     FALSE
     * \synchronous   TRUE
     *
     * \internal
     * - Update observers.
     * - For all active observers, trigger a FindService request and call the respective user handler.
     * \endinternal
     */
    void operator()() override {
      findservice_observers_manager_.UpdateObservers();

      handle_.ExecuteFindServiceAndHandler(
          [](InstanceSpecifierLookupTableEntryContainer const& service_instances)
              -> ara::com::ServiceHandleContainer<HandleType> { return FindService(service_instances); });
    }
```



在  **ExecuteFindServiceAndHandler**  函数中，传入service handle的vector

```
  /*!
   * \brief     Executes a FindService request and the associated handler (if it is active).
   * \tparam F  Type of find_service_function lambda function.
   * \param[in] find_service_function Lambda executing FindService.
   * \pre -
   * \context Callback
   */
  template <typename F>
  void ExecuteFindServiceAndHandler(F&& find_service_function) const {
    std::lock_guard<std::recursive_mutex> const guard(state_->lock_);
    if (state_->active_) {
      ara::com::ServiceHandleContainer<HandleType> const handles{find_service_function(handle_.GetServiceInstances())};
      handler_(handles);
    }
  }
```

这时真正调用用户层通过StartFindService传入的handle（本例中即：  **FindService1Handler**  ，操作是：判断  **FindService**  是否成功，成功则根据service_handle创建  **parameter_service_interfaceProxy**  对象），并通过条件变量唤醒其他阻塞的线程（实际上就是主线程的StartClient()中阻塞等待了一秒钟）



**GetServiceInstances**  ()用于得到  **find_service_function**  函数的入参

返回一个std::  **set**  <  **InstanceSpecifierLookupTableEntry**  >

**PS：**

**find_service_function**  的调用，实际上是调用  **FindService**

```
/*!
   * \brief Call binding-specific FindService operation and convert returned InstanceHandles back into HandleTypes.
   * \param[in] service_instances Container of searched service instances (InstanceSpecifier lookup table entries).
   * \return Found service instances.
   * \pre The instance identifier must be valid and must not be malformed.
   * \context App, Callback
   * \threadsafe FALSE
   * \reentrant FALSE
   * \synchronous TRUE
   *
   * \internal
   * Calls "ara::core::Abort()" if:
   * - SOME/IP binding only:
   *   If the SOME/IP binding specific instance identifier cannot not be parsed (called within the binding FindService).
   *
   * - IPC binding only:
   *   If instance is not valid (called within the binding FindService).
   *
   * - ara::com::IamAccessDeniedException is caught:
   *   SOME/IP binding only:
   *   If "FindService()" request is denied by IAM.
   *
   * - An unknown exception is caught.
   * \endinternal
   */
  static ara::com::ServiceHandleContainer<HandleType> FindService(
      InstanceSpecifierLookupTableEntryContainer const& service_instances) noexcept {
    ara::com::ServiceHandleContainer<HandleType> handles{};
    try {
      // Call FindService() of the related binding
      for (InstanceSpecifierLookupTableEntry const& service_instance : service_instances) {
        ara::com::ServiceHandleContainer<InstanceHandle> const instance_handles{
            service_instance.GetBinding()->FindService(
                kServiceIdentifier, service_instance.GetInstanceIdentifierString(), GetProxyIdStatic())};

        // Convert the instance handles into HandleType objects
        for (InstanceHandle const& instance_handle : instance_handles) {
          handles.emplace_back(service_instance.GetInstanceIdentifier(), instance_handle.GetProxyFactory());
        }
      }
    }  // VECTOR Next Line AutosarC++17_10-A15.3.4: MD_SOCAL_AutosarC++17_10-A15.3.4_Caught_exception_is_too_general
    catch (std::exception const& e) {
      ::ara::core::Abort(e.what());
    }
    // VECTOR NC AutosarC++17_10-A15.3.4: MD_SOCAL_AutosarC++17_10-A15.3.4_Using_catch_all
    // VECTOR NC AutosarC++17_10-M0.3.1: MD_SOCAL_AutosarC++17_10-M0.3.1_Dead_exception_handler
    catch (...) {
      ::ara::core::Abort("Proxy::FindService: Unknown exception.");
    }
    return handles;
  }
```

遍历  **InstanceSpecifierLookupTableEntry**  的集合

创建  **vector**  <  **InstanceHandle>，返回BindingInterface**  （抽象基类）指针（指向的是  **AraComSomeIpBinding**  对象），调用  **FindService**  ()函数

```
  /*!
   * \brief           Execute a synchronous FindService call.
   * \details         If the instance identifier could not be parsed, an error is logged and we abort.
   * \param[in]       instance The binding specific instance identifier in string representation.
   * \param[in]       proxy_id The amsr::socal::internal::ProxyId to search for
   * \return          A container of InstanceHandles for the currently known service instances.
   * \throws          ara::com::IamAccessDeniedException if FindService request was denied by IAM.
   * \pre             Fully initialized SOME/IP binding.
   * \context         App, Callback
   * \threadsafe      FALSE
   * \reentrant       FALSE
   * \exceptionsafety No side effects.
   * \vprivate
   * \synchronous     TRUE
   *
   * \internal
   * - Parse instance identifier string
   * - Call FindService on the client manager
   *   - In case of parse error, log error and abort
   * - Return container of found instance
   * \endinternal
   */
  ara::com::ServiceHandleContainer<::amsr::socal::internal::InstanceHandle> FindService(
      ara::com::AraComServiceId const&, ::amsr::socal::internal::InstanceIdentifierString const& instance,
      ::amsr::socal::internal::ProxyId proxy_id) override {
    ara::com::ServiceHandleContainer<::amsr::socal::internal::InstanceHandle> found_instances{};

    ara::core::Result<SomeIpInstanceIdentifier> const someip_instance_id{ParseInstanceIdentifierString(instance)};
    if (someip_instance_id.HasValue()) {
      found_instances = client_manager_.FindService(proxy_id, someip_instance_id.Value());
    } else {
      char const* error_msg{"Failed to parse SOME/IP binding instance identifier string."};
      logger_.LogFatal(
          [&instance, &someip_instance_id, &error_msg](ara::log::LogStream& s) {
            ara::core::StringView const error_message{someip_instance_id.Error().Message()};
            s << error_msg << instance << "': " << error_message << ".";
          },
          __func__, __LINE__);

      ara::core::Abort(error_msg);
    }

    return found_instances;
  }
```

其中调用了  **AraComSomeIpBinding**  的  **ClientManager**  类型的成员的  **FindService(ServiceIdentifierType**  ，   **InstanceIdentifierString**  ，  **ProxyId)**

**AraComSomeIpBindingClientManager：**

```
 /*!
   * \brief           Execute a synchronous FindService call.
   * \param[in]       proxy_id The amsr::socal::internal::ProxyId to search for
   * \param[in]       instance The SOME/IP instance identifier for which to find a service
   * \return          A container of found instance handles.
   * \throws          ara::com::IamAccessDeniedException if FindService request was denied by IAM.
   * \pre             -
   * \context         App, Callback
   * \threadsafe      FALSE
   * \reentrant       FALSE
   * \exceptionsafety No side effects.
   * \vprivate
   * \synchronous     TRUE
   *
   * \internal
   * - Find its matching proxy factory
   * - Forward the find service request to the SOME/IP Daemon
   *   - If the daemon returns kOk
   *     - Build handles for every found service instances
   *   - If the daemon returns kFindServiceAccessDenied
   *     - Throw ara::com::IamAccessDeniedException
   *   - If the daemon returns kNotConnected
   *     - Log error
   *   - For other return values
   *     - Log generic error
   * \endinternal
   */
  ara::com::ServiceHandleContainer<::amsr::socal::internal::InstanceHandle> FindService(
      ::amsr::socal::internal::ProxyId const proxy_id, SomeIpInstanceIdentifier const& instance) override {
    logger_.LogDebug([](ara::log::LogStream const&) {}, __func__, __LINE__);
    ara::com::ServiceHandleContainer<::amsr::socal::internal::InstanceHandle> handles{};

    // Find its matching ServiceSomeIpProxyFactory to create the binding-related proxy-part, when the user
    // creates one proxy instance.
    ProxyFactoryContainer::iterator const it_begin{proxy_factories_.begin()};
    ProxyFactoryContainer::iterator const it_end{proxy_factories_.end()};
    ProxyFactoryContainer::iterator const it{
        std::find_if(it_begin, it_end, [proxy_id](std::unique_ptr<AraComSomeIpProxyFactoryInterface> const& factory) {
          return factory->GetProxyId() == proxy_id;
        })};

    if (it != it_end) {
      ::amsr::someip_protocol::internal::ServiceId const someip_service_id{it->get()->GetSomeIpServiceId()};

      // Forward the FindService request to the SOME/IP daemon and get currently known service instances.
      ::amsr::someip_daemon_client::internal::FindServiceResult const find_service_result{
          someip_posix_.FindService(someip_service_id, instance.GetInstanceId())};
      ::amsr::someip_daemon_client::internal::ControlMessageReturnCode const return_code{
          find_service_result.return_code};
      if (return_code == ::amsr::someip_daemon_client::internal::ControlMessageReturnCode::kOk) {
        // Build handles for every found service instances
        for (::amsr::someip_protocol::internal::ServiceInstance const& service_instance :
             find_service_result.service_instances) {
          // Create a new InstaceHandle containing
          // - A string representation of the found service. String representation because the InstanceHandle is an
          // binding independent object.
          // - A pointer to a proxy factory used during Proxy construction to create the binding specific 'backend'
          // object for the service instance.
          handles.emplace_back(std::to_string(static_cast<std::uint32_t>(service_instance.instance_id_)), (*it).get());
        }
      } else if (return_code ==
                 ::amsr::someip_daemon_client::internal::ControlMessageReturnCode::kFindServiceAccessDenied) {
        logger_.LogError(
            [&someip_service_id, &instance](ara::log::LogStream& s) {
              ::amsr::someip_protocol::internal::InstanceId const instance_id{instance.GetInstanceId()};
              s << "FindService request (0x" << ara::log::HexFormat(someip_service_id) << ", 0x"
                << ara::log::HexFormat(instance_id) << ") denied by identity and access management.";
            },
            __func__, __LINE__);
        throw ara::com::IamAccessDeniedException("FindService");
      } else if (return_code == ::amsr::someip_daemon_client::internal::ControlMessageReturnCode::kNotConnected) {
        logger_.LogError(
            [](ara::log::LogStream& s) {
              s << "Failed to look up service because a connection to the SOME/IP daemon has not been established. "
                   "Returning an empty container.";
            },
            __func__, __LINE__);
      } else {
        logger_.LogError(
            [&return_code](ara::log::LogStream& s) {
              s << "FindService request failed with return code 0x"
                << ara::log::HexFormat(
                       static_cast<std::underlying_type<
                           ::amsr::someip_daemon_client::internal::ControlMessageReturnCode>::type>(return_code))
                << ". Returning an empty list of offered service instances.";
            },
            __func__, __LINE__);
      }
    }
    return handles;
  }
```

查找  **ProxyFactoryContainer**  （vector<  **AraComSomeIpProxyFactoryInterface**  >）是否有proxy_id为入参的

然后得到service id

**重点**  ：然后给  **SOME/IP daemon**  发送  **FindService**  请求

接着遍历  **FindService**  返回的  **FindServiceResult**  类对象的结构体成员：std::  **vector**  <  **ServiceInstance**  >（包含了找到的service instance）：

根据  **instance_id**  ,     **ServiceProxyFactoryInterface***  （抽象类，它被  **AraComSomeIpProxyFactoryInterface**  继承，然后这个类又被  **AraComSomeIpProxyFactory**  继承）创建  **InstanceHandle**  （作为返回值）



给  **SOME/IP daemon**  发送  **FindService**  请求  是通过调用  **SomeIpDaemonClient**  成员  **someip_posix_**  的  **FindService**  (  **service_id**  ,      **instance_id**  )来实现的，返回  **FindServiceResult**  类型的变量，其内部实现是：调用了  **SomeIpDaemonClient类的成员：CommandController**  类对象的  **FindService**  (  **service_id**  ,      **instance_id**  )，其内部实现如下：

```
/*!
   * \brief Returns service instances offered at the moment from the SOME/IP daemon.
   * \param[in] service_id SOME/IP service id of the service.
   * \param[in] instance_id SOME/IP instance id of the service.
   * \return A FindService result structure containing a return code and a container of service instances offered at the
   * time of the call.
   * \error amsr::someipd_app_protocol::internal::ControlMessageReturnCode::kNotOk The error is returned if the
   * serialization of the command control packet fails.
   * \pre -
   * \context ANY
   * \threadsafe TRUE
   * \reentrant FALSE
   *
   * \internal
   * - Lock mutex.
   * - Initialize a FindServiceResult with return code kNotOk.
   * - Serialize a command control request packet with service_id and instance_id. Store returned MemoryBufferPtr.
   * - Synchronously send a FindService command with the previously serialized request packet. Store return code and
   *   response packet.
   * - Set return_code of FindServiceResult to return code of send command.
   * - If return code is kOk
   *   - If size of response packet is greater than 0.
   *     - Deserialize the response packet.
   *     - If the deserialization is not successful
   *       - Log a warning of incorrect size of FindService response package.
   *   - else
   *     - Log debug message that FindService request returned without found service instance.
   * - Return FindServiceResult
   * \endinternal
   */
  FindServiceResult FindService(ControlMessageServiceId service_id, ControlMessageInstanceId instance_id) {
    std::lock_guard<std::mutex> const lock{command_lock_};
    FindServiceResult result{ControlMessageReturnCode::kNotOk, {}};

    // Serialize request packet
    /* VECTOR Next Line AutosarC++17_10-A18.5.8: MD_SOMEIPDAEMONCLIENT_AutosarC++17_10-A18.5.8_use_of_smartpointer: */
    MemoryBufferPtr request_packet{SerializeServiceInstanceRequestPacket(service_id, instance_id)};

    // Synchronous command request to SOME/IP daemon
    SendControlCommandResult const response{
        SendControlCommand(MessageType::kFindService, ControlMessageReturnCode::kOk, 0U, std::move(request_packet))};

    result.return_code = response.first;
    // Deserialize ServiceInstance pairs
    if (result.return_code == ControlMessageReturnCode::kOk) {
      // Multiple of single ServiceInstance in payload?
      std::size_t const response_packet_size{response.second->size()};
      if (response_packet_size > 0U) {
        MemoryBufferView packet_view{response.second->GetView(0U)};
        /* VECTOR Disable AutosarC++17_10-M5.2.8:
        MD_SOMEIPDAEMONCLIENT_AutosarC++17_10-M5.2.8_Conversion_from_void*_or_integer_to_pointer_type */
        Reader::BufferView const buffer_view{static_cast<Reader::BufferView::pointer>(packet_view[0U].base_pointer),
                                             packet_view[0U].size};
        // VECTOR Enable AutosarC++17_10-M5.2.8
        Reader reader{buffer_view};

        bool const deserialize_result{
            amsr::someipd_app_protocol::internal::DeserializeControlMessageFindServiceResponsePayload(
                reader, result.service_instances)};
        if (!deserialize_result) {
          logger_.LogWarn(
              [](ara::log::LogStream& s) {
                s << "Incorrect size of FindService response message. Payload size is not a multiple of single "
                     "ServiceInstance entry.";
              },
              __func__, __LINE__);
        }
      } else {
        logger_.LogDebug(
            [](ara::log::LogStream& s) { s << "FindService request returned without found service instances."; },
            __func__, __LINE__);
      }
    }

    return result;
  }

```

序列化出一个  **request_packet**  ，然后调用  **SendControlCommand**  ()将其（  **同步**  地）发送出去，接收  **response**  也是同步的，然后判断返回的  **response**  是否有效，这是通过调用  **DeserializeControlMessageFindServiceResponsePayload**  来判断的，反序列化失败则说明数据有问题





# Proxy SubscriptionState
**调用顺序：**

**SubscriptionStateUpdateTask**  （嵌套于class   **ProxyEventBase**  中）

线程池中线程调用其  **operator()**  函数，调用  **ProxyEventBase&**  成员的  **NotifySubscriptionStateUpdateSync**  ()函数，其中调用了注册的  **subscription_state_handler_**  (  **notified_state**  )






# Proxy Subscribe
```cpp
// 用户层调用：ProxyEvent对象的Subscribe()方法
service1_proxy_->StartApplicationEvent1.Subscribe(ara::com::EventCacheUpdatePolicy::kLastN, 1);
    kLastN: 是EventCacheUpdatePolicy，表示抛弃最近最少使用的，cache为1

// 其中调用了SomeipProxyEventManager对象的Subscribe()方法
  /*!
   * \brief Subscribe to the event.
   * \details Forward subscribe to EventBackend
   * \param[in] event Related frontend event object. Used for notification handling
   * \param[in] cache_size    Size limit of the event cache
   * \return true if subscription is successful, otherwise false.
   * \pre         SOME/IP binding must be initialized
   * \context     App
   * \threadsafe  FALSE When used with same proxy instance on same event.
   * \reentrant   FALSE
   * \vprivate
   * \synchronous FALSE
   *
   * \internal
   * - Guard against call to Update()
   *   - If the event is not subscribed
   *     - Update cache capacity
   *     - Set subscribe flag to true
   *     - Trigger Subscribe on the backend
   *   - Otherwise, log a warning
   * \endinternal
   */
  bool Subscribe(::amsr::socal::internal::ProxyEventBindingInterface* event, std::size_t cache_size) override {
    bool is_subscription_successful{false};
    std::lock_guard<std::mut  ex> const guard{subscription_lock_};

    // Check for validity of event. (This is passed by ara::com, so it is ok to assert its correctness)
    assert(event != nullptr);  // No event frontend for subscription state handling given.

    if (!subscribed_) {
      cache_capacity_ = cache_size;  // Update Cache Capacity
      service_event_ = event;

      // set subscribed_ to true before calling SubscribeEvent, as HandleEventSubscriptionStateUpdate might then be
      // called at any moment even before SubscribeEvent is returned.
      subscribed_ = true;

      base_sequence_ = backend_->Subscribe(this, cache_size);
      last_known_sequence_ = base_sequence_;

      is_subscription_successful = true;
    } else {
      logger_.LogWarn(
          [](ara::log::LogStream& s) { s << "Ignoring subscription: Event has already been subscribed to."; }, __func__,
          __LINE__);
    }
    return is_subscription_successful;
  }

// 把订阅消息转发给event后端
// 其操作的重点是创建了ReactorSyncTask任务，执行SubscribeSync()，
  /*!
   * \brief       Adds the provided event manager to the subscriber list and sends SubscribeEvent
   *              to the ClientManager if this is the first subscriber.
   * \details     Called from Event Manager.
   * \param[in]   subscriber A pointer to the event manager that subscribes this event backend.
   * \param[in]   cache_size Size limit of the event cache
   * \return      The (currently) highest assigned sequence number, i.e. the sequence number of the latest event to
   *              have already arrived
   * \pre         -
   * \context     App
   * \threadsafe  TRUE
   * \reentrant   FALSE
   * \vprivate
   * \synchronous TRUE
   *
   * \internal
   * - Guard against parallel access to the last event sequence
   *   - Make a local copy of the last event sequence
   * - Guard against parallel Subscriptions or Unsubscriptions
   *   - Update size of the invisible sample cache
   *   - Create a task to perform subscription from the Reactor
   *   - Wait until the Reactor has performed the task
   *   - Trigger the client manager to subscribe to the event
   * \endinternal
   */
  std::size_t Subscribe(ServiceProxySomeIpEventInterface* subscriber, std::size_t cache_size) {
    logger_.LogVerbose([](ara::log::LogStream const&) {}, __func__, __LINE__);

    std::size_t result;

    {
      std::lock_guard<std::mutex> const cache_guard{sample_cache_lock_};
      result = last_event_sequence_;
    }

    {  // Guard against parallel subscriptions/unsubscriptions
      std::lock_guard<std::mutex> const guard{subscriber_lock_};

      // Update size of the InvisibleSampleCache as the largest over all subscribers
      if (cache_size > cache_capacity_) {
        cache_capacity_ = cache_size;
      }

      ReactorSyncTask<bool> task{timer_manager_, [this, &subscriber] { return SubscribeSync(subscriber); }, false};
      if (!task()) {    // ReactorSyncTask的operator()：执行传入的函数
        someip_binding_client_manager_.SubscribeEvent(kServiceId, instance_id_, kEventId);
      }
    }

    return result;
  }

// operator中的逻辑：
  /*!
   * \brief       Trigger the task to be executed via the reactor thread and get back its result.
   * \details     The caller will be blocked until the task finishes.
   * \return      The result of the task, or the default result in case a timeout occurred.
   * \pre         -
   * \context     App
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \vprivate
   * \synchronous TRUE
   */
  Result operator()() {
    if (ara::com::Runtime::getInstance().GetProcessingMode() ==
        ::amsr::socal::internal::configuration::RuntimeProcessingMode::kPolling) {
      // No reactor thread is running, we must do the job ourselves.
      std::lock_guard<std::recursive_mutex> const lock{ara::com::Runtime::getInstance().GetPollingModeLock()};
      PerformTask();
    } else {
      /* Trigger reactor thread to do the work via a zero-length timer */
      Timer::Stop();
      Timer::SetPeriod(std::chrono::seconds::zero());
      Timer::Start();
    }

    ara::core::future_status const status{future_.wait_for(kTimeout)};
    if (status == ara::core::future_status::ready) {
      result_ = future_.get();
    }
    return result_;
  }


// ReactorSyncTask会执行以下函数，阻塞直到完成
/*!
   * \brief       Adds the given subscriber to the subscriber list and notify the updated subscription state if
   *              subscription is completed.
   * \tparam      EventConfig  Event configuration struct.
   * \param[in]   subscriber The subscriber to be added from the list.
   *              If subscription is completed, it will be notified with the current subscription state.
   * \return      true if Subscription is completely done (no further steps are needed).
   *              false if subscriber is added to the list but further action is required (Subscribe shall be
   *              forwarded to the daemon).
   * \pre         -
   * \context     Reactor
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \vprivate
   * \synchronous TRUE
   *
   * \internal
   * - Add subscriber to the list of subscribers
   * - If this is not the first subscription, update the subscriber
   *   with the current subscription state
   * \endinternal
   */
  template <typename T = EventConfig>
  auto SubscribeSync(ServiceProxySomeIpEventInterface* subscriber) ->
      typename std::enable_if<!T::kIsFieldEvent, bool>::type {
    logger_.LogVerbose([](ara::log::LogStream const&) {}, __func__, __LINE__);

    bool result{false};
    subscriber_list_.push_back(subscriber);

    // If we are not the first subscriber, nothing more shall be done.
    std::size_t const subscriber_count{subscriber_list_.size()};
    if (subscriber_count > 1U) {
      // Subscription is completed.
      result = true;
      subscriber->HandleEventSubscriptionStateUpdate(state_);
    }
    return result;
  }
```

然后调用SOME/IP binding类型的someip_binding_client_manager_的`SubscribeEvent(kServiceId, instance_id_, kEventId)`方法：

**重点是向SomeIP Daemon发送订阅请求**
```cpp
/*!
   * \brief       Lets the SOME/IP binding know that a proxy wishes to receive an event of a service instance.
   * \details     This function will abort in case the connection to the SOME/IP daemon has not been established.
   * \param[in]   service_id  SOME/IP service id of the service.
   * \param[in]   instance_id SOME/IP instance id of the service.
   * \param[in]   event_id    SOME/IP event id of the service.
   * \pre         Connection to the SOME/IP daemon is established.
   * \context     App
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \vprivate
   * \synchronous TRUE
   *
   * \internal
   * - Send subscribe request to the SOME/IP daemon
   * - In case of IAM access denied, log error
   * - In case there is no connection with the SOME/IP daemon, log error and abort
   * \endinternal
   */
  void SubscribeEvent(::amsr::someip_protocol::internal::ServiceId const service_id,
                      ::amsr::someip_protocol::internal::InstanceId const instance_id,
                      ::amsr::someip_protocol::internal::EventId const event_id) override {
    ::amsr::someip_daemon_client::internal::ControlMessageReturnCode const return_code{
        someip_posix_.SubscribeEvent(service_id, instance_id, event_id)};
    switch (return_code) {
      case ::amsr::someip_daemon_client::internal::ControlMessageReturnCode::kOk: {
        // event subscription successful. No more actions necessary.
        break;
      }
      case ::amsr::someip_daemon_client::internal::ControlMessageReturnCode::kNotConnected: {
        logger_.LogError(
            [&return_code, &service_id, &event_id, &instance_id](ara::log::LogStream& s) {
              s << "Failed to subscribe to ";
              amsr::someip_binding::internal::logging::LogBuilder::LogEventId(s, service_id, event_id, instance_id);
              s << ". Reason: ";
              amsr::someip_binding::internal::logging::LogBuilder::LogControlMessageReturnCodeAsString(s, return_code);
            },
            __func__, __LINE__);
        ara::core::Abort("Failed to subscribe to event. No connection to the SOME/IP Daemon has been established.");
        break;
      }
      default: {
        logger_.LogError(
            [&return_code, &service_id, &event_id, &instance_id](ara::log::LogStream& s) {
              s << "Failed to subscribe to ";
              amsr::someip_binding::internal::logging::LogBuilder::LogEventId(s, service_id, event_id, instance_id);
              s << ". Reason: ";
              amsr::someip_binding::internal::logging::LogBuilder::LogControlMessageReturnCodeAsString(s, return_code);
            },
            __func__, __LINE__);
        break;
      }
    }
  }
```
`SomeIpPosix&`的`SubscribeEvent()`，是通过`CommandController`的`SubscribeEvent()`，它是真正发送SOME/IP控制消息




# Skeleton端发送数据

## Event


## Method
当Proxy端发起method request时，
触发一系列回调：
`OnReceiveCompletion()`、`ProcessReceivedMessage()`

往线程池增加`AsyncRequest`类型的任务：
即`SkeletonStartApplicationMethod1AsyncRequest`类型

最终触发以下函数：
（PS：显然是src-gen中生成的）
`SkeletonStartApplicationMethod1AsyncRequest`对象的`operator()`被调用
```cpp
  /*!
   * \brief   Operator gets called when method invocation is planned in the frontend.
   * \details It shall be called only once for each instance.
   * \pre -
   * \context     Callback
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \synchronous TRUE
   *
   */
  void operator()() override {
    // VECTOR Next Line AutosarC++17_10-A18.5.8: MD_SOMEIPBINDING_AutosarC++17_10-A18.5.8_Local_object_allocated_in_the_heap
    std::unique_ptr<Input> const input{std::make_unique<Input>()};
    bool const deserialization_ok{DeserializeInput(packet_.get(), *input)};
    if (deserialization_ok) {
    std::uint8_t const& arg_input_argument{input->input_argument};

    ara::core::Result<::startapplication::cm::service1::internal::methods::StartApplicationMethod1::Output> result{skeleton_->StartApplicationMethod1(arg_input_argument).GetResult()};
    if (result.HasValue()) {
      response_handler_.SerializeAndSendMethodResponse<Serializer>(header_, result.Value());
    } else {
      response_handler_.SerializeAndSendApplicationErrorMethodResponse(header_, result.Error());
    }
    } else { // Deserialization failed
      response_handler_.SendErrorResponse(header_,
                                          ::amsr::someip_protocol::internal::SomeIpReturnCode::kMalformedMessage);
    }
  }
```
首先进行反序列化，解出数据后，直接通过`::startapplication::cm::service1::skeleton::StartApplicationCmService1_ServiceInterfaceSkeleton`类型的指针，调用用户层传入的`method回调`，调用`GetResult`进行阻塞等待，得到结果后，直接调用`response_handler_.SerializeAndSendMethodResponse<Serializer>(header_, result.Value());`，将结果返回

所以，可以认为：当proxy端连续发来多个相同类型的method request时，skeleton端每收到一个，就创建一个相应的任务



# Proxy端初始化






# Proxy端接收数据
通过`Unix Domain Socket`和Someipd通信，接收来自skeleton的消息

**调用顺序：**
任务类`*EventNotificationTask`（嵌套于class`ProxyEventBase`中）

线程池中线程调用该任务类的`operator()`函数，
其中调用`ProxyEventBase&`成员的`NotifySync()`函数，
其中调用了注册的receive handler（出于性能考虑，只在处理程序自上次调用后发生变化时才copy handler？？？）



## 接收`event`类型的数据
每个app的com模块代码中，会创建reactor线程，专门处理unix domain socket读写事件。
当可读事件发生时，触发初始化阶段注册的回调：
```cpp
      case ConnectionState::kConnected:
        if (events.HasReadEvent()) {
          reader_.OnReactorEvent(native_handle_);
        }
```
经过一系列调用
`StartReceiving()`
```cpp
    osabstraction::io::ipc1::ReceiveCompletionCallback completion_callback{
        [this](ara::core::Result<std::size_t>&& receive_complete_result) {
          OnReceiveCompletion(std::move(receive_complete_result));
        }};
```
`OnReceiveCompletion`
```cpp
      // Verify that we received at least generic header and specific header
      if (received_length >= kHeaderLength) {
        ProcessReceivedMessage();

        // Trigger an asynchronous reception again.
        StartReceiving();
      }
```
`OnSomeIpRoutingMessage`
```cpp
        someip_daemon_client_.OnSomeIpRoutingMessage(routing_someip_header.instance_id_,
                                                     std::move(reception_buffer_.receive_message_body));
```

```cpp
client_manager_->HandleReceive(instance_id, someip_header, std::move(someip_message));
```

`HandleReceive`
```cpp
  void HandleReceive(::amsr::someip_protocol::internal::InstanceId const instance_id,
                     ::amsr::someip_protocol::internal::SomeIpMessageHeader const& header,
                     MemoryBufferPtr packet) override {
    logger_.LogDebug([](ara::log::LogStream const&) {}, __func__, __LINE__);

    ::amsr::someip_protocol::internal::SomeIpReturnCode const error_code{DoInfrastructuralChecks(header)};
    if (error_code == ::amsr::someip_protocol::internal::SomeIpReturnCode::kOk) {
      switch (header.message_type_) {
        case ::amsr::someip_protocol::internal::kNotification: {
          RouteEventNotification(instance_id, header, std::move(packet));  // delegate to concrete proxy-binding
          break;
        }
```

```cpp
      it->second->OnEvent(std::move(packet));
```

`SomeipProxyEventBackend`的`OnEvent`
！！！和proxy event的`sample cache`相关联了，`sample cache`是多级的，分为用户可见 和 用户不可见

该回调是为了把新读取到的数据存储到`invisible_sample_cache_`
需要记录序号(从0开始自增)，用于更新策略
为每一个`event_manager`调用`HandleEventNotification()`
同时维护一个`last_event_entry_`，只存储最新的条目

```cpp
  void OnEvent(MemoryBufferPtr packet) override {
    // Create Entry
    // VECTOR NL AutosarC++17_10-A18.5.8: MD_SOMEIPBINDING_AutosarC++17_10-A18.5.8_Large_packets_allocated_on_stack
    SomeIpSampleCacheEntrySharedPtr entry{std::make_shared<SomeIpSampleCacheEntry<SampleType>>(std::move(packet))};

    // Store a shared pointer of event sample in an invisible cache and removes the oldest cache element if full.
    if (cache_capacity_ != 0U) {  // 即Subscribe(...)的输入，当前为1
      std::lock_guard<std::mutex> const guard{sample_cache_lock_};
      assert(invisible_sample_cache_.size() <= cache_capacity_);  // invisible_sample_cache_的size被限制在cache_capacity_下
      if (invisible_sample_cache_.size() == cache_capacity_) {  // 说明invisible_sample_cache_满了
        invisible_sample_cache_.pop_front();
      }
      invisible_sample_cache_.push_back(entry);

      last_event_sequence_ = EventSequenceUtil::GetNextSequence(last_event_sequence_);
    }

    // Call HandleEventNotification for all event managers in the list
    for (ServiceProxySomeIpEventInterface* const event_manager : subscriber_list_) {
      event_manager->HandleEventNotification();
    }

    // Update last received entry
    last_event_entry_.emplace(entry);
  }
```

`HandleEventNotification()`
```cpp
  void HandleEventNotification() override {
    if (subscribed_) {
      // Notify EventReceiveHandler
      service_event_->Notify();
    }
  }
```

`Notify()`，异步调用到`receive handler`
```cpp
  void Notify() noexcept override {
    /* Only schedule an event notification task, if a receive handler is set, indicated by
    receive_handler_set_ equals true. This is done for performance reasons. */
    if (receive_handler_set_.load()) {
      ara::com::Runtime& runtime{ara::com::Runtime::getInstance()};

      ThreadPool& pool{runtime.RequestThreadPoolAssignment()};
      // VECTOR NC AutosarC++17_10-A0.1.2: MD_SOCAL_AutosarC++17_10-A0.1.2_Notify
      // VECTOR NC AutosarC++17_10-M0.3.2: MD_SOCAL_AutosarC++17_10-M0.3.2_Notify
      pool.AddTask(vac::language::make_unique<EventNotificationTask>(*this));
    }
  }
```
注意到，检测到set了receive handler，就构造`EventNotificationTask`对象，`AddTask()`
`EventNotificationTask`嵌套在`ProxyEventBase`类中
`EventNotificationTask`
其构造函数，先构造基类对象，再将引用成员绑定到入参
`operator()`只不过是调用`NotifySync()`
```cpp
  class EventNotificationTask final : public Task {
   public:
    /*!
     * \brief Initialize the functor to call on event notification.
     *
     * \param[in] event The event to notify.
     */
    explicit EventNotificationTask(ProxyEventBase& event) : Task{&event}, event_{event} {}

    /*!
     * \brief Execute the event notification handler.
     * \details     Called in the context of the Default Thread Pool.
     * \pre -
     * \context     Callback
     * \threadsafe  FALSE
     * \reentrant   FALSE
     * \synchronous TRUE
     */
    void operator()() override { event_.NotifySync(); }

   private:
    /*!
     * \brief The event to notify.
     */
    ProxyEventBase& event_;
  };
```

！！！从而调用到了注册的`Receive handler`
注意，客户在应用层代码可能多次调用`SetReceiveHandler`，
而`receive_handler_`这个函数对象由`ProxyEventBase`对象来维护，在调用前通过标志位判断receive handler是否有更新
`It is set once Set/UnsetReceiveHandler is called, and reset once NotifySync is being called.`
`NotifySync()`如果不调用，就不重新为`receive_handler_`赋值，毕竟客户可能连续调用`SetRcvHandler`，只取最后一次即可

```cpp
  /*!
   * \brief Notify event receive handler on reception of a new event sample. Internal use only.
   * \details     Called in the context of the Default Thread Pool.
   * \pre         -
   * \context     Callback
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \synchronous TRUE
   * \trace       SPEC-4980082, SPEC-4980117, SPEC-4980378
   *
   * \internal
   * - Lock mutex.
   * - If the receive handler has changed since last call
   *   - Set receive handler to the new handler.
   * - If receive handler is valid
   *   - Call the handler.
   *
   * Calls "std::terminate()" if:
   * - Mutex acquisition fails.
   * \endinternal
   */
  void NotifySync() noexcept {
    std::lock_guard<std::recursive_mutex> const guard{receive_handler_lock_};
    /* For performance reason we copy handler only if it is changed since last call */
    const bool& receive_handler_changed{new_receive_handler_pair_.first}; // SetReceiveHandler中会置为true

    if (receive_handler_changed) {
      receive_handler_ = new_receive_handler_pair_.second;
      new_receive_handler_pair_.first = false;
    } // 接收了receive_handler_后就置位false, 如果再次调用set receive handler则receive_handler_changed重新置为true

    /* We call receive handler within the receive_handler_lock_ just to make it deterministic,
     that once the user call returns from Set/Unset-ReceiveHandler, old receive_handler_ is no more used. */
    if (receive_handler_ != nullptr) {
      /* Do not use receive_handler_ after call of handler. Due to recursive mutex the handler could replace itself.
       */
      receive_handler_();
    }
  }
```
总之，，每次该proxy event有可读事件时，就最终触发到客户注册的`Receive Handler`，可以看作是同步的

```cpp
void StartApplicationCmClientService1::ReceiveHandlerService1Event1() {
  log_.LogInfo() << "[Service1] [Event1] Receive handler was called.";
  log_.LogInfo() << "[Service1] [Event1] Update cache and get new samples.";
  service1_proxy_->StartApplicationEvent1.Update();
  const service1::proxy::events::StartApplicationEvent1::SampleContainer& samples =
      service1_proxy_->StartApplicationEvent1.GetCachedSamples();
  for (std::size_t sample_idx{0}; sample_idx < samples.size(); ++sample_idx) {
    log_.LogInfo() << "[Service1] [Event1] Received value " << *samples[sample_idx] << ".";
  }
}
```

## `ReceiveHandlerService1Event1`
### `Update()`, 后续被取消
`ProxyEvent`的接口，调用了更底层的`Update`，注意IPC SOME/IP都有自己的Update()实现
更新用户可见的内存
```cpp
  /*!
   * \brief Updates the event cache container visible to the user via GetCachedSamples()
   * \details Calling Update() will invalidate any reference to a SampleContainer that has been acquired via
   *          GetCachedSamples() API before, after that GetCachedSamples has to be called again to get a new valid
   *          reference to the SampleContainer.
   * \param[in] filter A function that indicates whether an event should be copied to the event cache
   * \return true if at least one new event was transferred to the event cache
   * \pre Subscribe has been called, but Unsubscribe has not been called yet.
   * \context App, Callback
   * \threadsafe TRUE
   * \reentrant FALSE
   * \vpublic
   * \synchronous TRUE
   * \trace SPEC-8053568
   *
   * \internal
   * - Lock mutex.
   * - Get the binding-related proxy implementation.
   * - Call Update() on the event manager.
   *
   * Calls "ara::core::Abort()" if:
   * - Mutex locking fails.
   * - Construction of a new shared pointer fails.
   * - Any exception derived from "std::exception" is caught.
   * - An unknown exception is caught.
   * \endinternal
   */
  bool Update(const ara::com::FilterFunction<SampleType>& filter = {}) noexcept {
    bool result{false};
    try {
      if (is_subscribed_.load()) {
        // Protect concurrent modification of visible_sample_cache_ by Update(), GetCachedSamples(), Cleanup() APIs.
        std::lock_guard<std::mutex> const guard(visible_sample_cache_lock_);
        result = (proxy_ptr_->GetServiceProxyImplInterface().get()->*GetEventManagerMethod)()->Update(
            filter, event_cache_update_policy_, visible_sample_cache_);
      }
    }
    // VECTOR NC AutosarC++17_10-A15.3.4: MD_SOCAL_AutosarC++17_10-A15.3.4_Caught_exception_is_too_general
    // VECTOR NC AutosarC++17_10-M0.3.1: MD_SOCAL_AutosarC++17_10-M0.3.1_Dead_exception_handler
    catch (std::exception const& e) {
      ::ara::core::Abort(e.what());
    }
    // VECTOR NC AutosarC++17_10-A15.3.4: MD_SOCAL_AutosarC++17_10-A15.3.4_Using_catch_all
    // VECTOR NC AutosarC++17_10-M0.3.1: MD_SOCAL_AutosarC++17_10-M0.3.1_Dead_exception_handler
    catch (...) {
      ::ara::core::Abort("ProxyEvent::Update: Unknown exception.");
    }
    return result;
  }
```
以SOME/IP为例
#### 更新策略`kNewestN`: 
在这种策略下，每次调用 "更新 "时，缓存都会首先被清除，然后填入新的可用事件。
即使自上次调用`Update`后没有任何事件发生，缓存也会被清除。
```cpp
  /*!
   * \brief       Transfers the received events to the passed event container taking into account the current policy
   *              and the passed filter function
   *
   * \param[in]   filter                    A filter function indicating whether an event should be copied.
   * \param[in]   event_cache_update_policy The event cache update policy.
   * \param[out]  visible_cache             An event container which will receive events.
   *
   * \return      true if at least one new event has been copied into the visible cache
   *
   * \pre         -
   * \context     App
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \synchronous TRUE
   */
  bool Update(FilterFunction const& filter, ara::com::EventCacheUpdatePolicy const event_cache_update_policy,
              SampleContainer& visible_cache) override {
    std::lock_guard<std::mutex> const guard{subscription_lock_};
    bool result;

    if (event_cache_update_policy == ara::com::EventCacheUpdatePolicy::kNewestN) {
      result = UpdateNewest(filter, visible_cache);
    } else {
      result = UpdateLast(filter, visible_cache);
    }

    return result;
  }
```

`SomeipProxyEventManager`的`UpdateNewest`
```cpp
  /*!
   * \brief        Apply the filter provided with the first argument to the last cache_capacity_
   *               samples in the invisible cache and copy all those which pass the filter into
   *               the cache provided with the second argument.
   * \details      The 'last cache_capacity_  samples' only refers to those events which arrived EITHER
   *               after the current subscription became effective OR after the previous call to Update(),
   *               depending on which of these two occurred later. Samples which had already arrived before
   *               the call to Subscribe() or before a previous call to Update() will neither be passed
   *               to the filter nor copied.
   *
   * \param[in]    filter        The filter function to pass or nullptr
   * \param[out]   visible_cache The cache to copy into. The samples are ordered by age with the sample
   *                             at the highest position being the one which arrived last among all samples copied.
   *
   * \return       If at least one sample has been copied into the cache provided with the
   *               second argument, return true. Otherwise, return false.
   *
   * \pre          -
   * \context      App
   * \threadsafe   FALSE
   * \reentrant    FALSE
   * \synchronous  TRUE
   */
  bool UpdateNewest(FilterFunction const& filter, SampleContainer& visible_cache) {
    bool result{false};

    // Clear the visible cache
    visible_cache.clear();

    std::size_t const last_known_sequence{
        backend_->GetSamples(filter, last_known_sequence_, visible_cache, cache_capacity_)};

    if (visible_cache.size() > 0) {
      result = true;
    }
    last_known_sequence_ = last_known_sequence;
    return result;
  }
```
该策略确保数据是最新的，使得`GetCachedSamples`时`Get`到的数据绝不重复

##### `kNewestN`的具体实现
注意到：
```cpp
    std::size_t const last_known_sequence{
        backend_->GetSamples(filter, last_known_sequence_, visible_cache, cache_capacity_)};
```
`UpdateNewest`不断维护`last_known_sequence_`，并且将其传入`GetSamples`







#### 更新策略`kLastN`
`SomeipProxyEventManager`的`UpdateLast`
通过`SomeipProxyEventBackend<EventConfig>`的`GetSamples()`得到`last_known_sequence_`，
将其与`base_sequence_`作差
`base_sequence_`是`活动订阅（如果有）生效前到达的最后一个事件的序列号。该值仅在更新策略为 EventCacheUpdatePolicy::kLastN 时使用。`
`last_known_sequence_`是`该 SomeipProxyEventManager 已"知道"的最高指定序列号`
更新`base_sequence_`和`last_known_sequence_`（经过观察，发现这两个序号一直凝固在最后两位(如果`Subscribe(1)`)）


```cpp
  /*!
   * \brief       Apply the filter provided with the first argument to the last cache_capacity_
   *              samples in the invisible cache and copy all those which pass the filter into
   *              the cache provided with the second argument.
   * \details     The 'last cache_capacity_  samples' only refers to those events which arrived after
   *              the current subscription became effective. Samples which had already arrived before
   *              the call to Subscribe() will neither be passed to the filter nor copied.
   * \param[in]   filter         The filter function to pass or nullptr
   * \param[out]  visible_cache  The cache to copy into. The samples are ordered by age with the sample at
   *                             the highest position being the one which arrived last among all samples copied.
   * \return      If at least one new event sample arrived between the previous call to Update() and
   *              this one AND at least one sample has been copied into the cache provided with the
   *              second argument, return true. Otherwise, return false.
   * \pre         -
   * \context     App
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \vprivate
   * \synchronous TRUE
   */
  bool UpdateLast(FilterFunction const& filter, SampleContainer& visible_cache) {
    bool result{false};

    // Clear the visible cache
    visible_cache.clear();

    std::size_t const last_known_sequence{backend_->GetSamples(filter, base_sequence_, visible_cache, cache_capacity_)};

    std::size_t const sequence_difference{
        EventSequenceUtil::GetSequenceDifference(base_sequence_, last_known_sequence)};

    if (last_known_sequence != last_known_sequence_) {
      result = (visible_cache.size() > 0);
      if (sequence_difference >= cache_capacity_) {
        base_sequence_ = EventSequenceUtil::GetSequenceDifference(cache_capacity_, last_known_sequence);
      }
    }
    last_known_sequence_ = last_known_sequence;
    return result;
  }
```


`SomeipProxyEventBackend<EventConfig>`的`GetSamples()`
PS:
从`invisible_sample_cache_`中取数据到`visible_cache`
`newest_undesired_sequence`是被认为太旧的event中，最新的的序号
(PS: 实际上，历史的历史的每一个event都需要有一个编号，这些序号都是用来作差的)
比如说，skeleton先启动一顿狂发5个event，`invisible_sample_cache_`全存储下来了: 0, 1, 2, 3, 4, 5
而`Subscribe`时设置的`visible_cache`大小为1，则只保留1个
然后设置一个skip值，便于for循环，跳过不需要管的老的几个数据
因为`invisible_sample_cache_`存储时，是新的在最后面
`invisible_sample_cache_`中很可能是未经反序列化的数据，因此调用更底层的`GetSample`得到反序列化后的数据
然后经过`filter`筛选，得到实际数据，经过筛选条件后塞入`visible_cache`(实际上是把`shared_ptr`塞进去了，塞进去可以延长生命周期)
该函数的目的是得到一个序号
```cpp
  /*!
   * \brief       Transfers the received events to the passed event container
   * \param[in]   filter                    A filter function indicating whether an event should be copied.
   * \param[in]   newest_undesired_sequence Sequence number of the newest among those events considered to be too old
   * \param[out]  visible_cache             An event container which will receive events.
   * \param[in]   max_samples               Maximum number of samples to transfer.
   * \return      The sequence number of the last event received
   * \pre         -
   * \context     App
   * \threadsafe  TRUE
   * \reentrant   FALSE
   * \vprivate
   * \synchronous TRUE
   *
   * \internal
   * - Guard against parallel access to the invisible sample cache, e2e state and last event sequence
   *   - Get the amount of new available samples
   *   - Compute the amount of requested samples
   *   - Skip to the newest events requested in the invisible cache
   *   - For all the new events requested
   *     - Get sample. This may do deserialization (log error in case of deserialization error)
   *     - Apply filter
   *     - If pass through filter
   *       - Update E2E state
   *       - Emplace sample pointer in the visible cache
   * \endinternal
   */
  std::size_t GetSamples(FilterFunction const& filter, std::size_t const newest_undesired_sequence,
                         SampleContainer& visible_cache, std::size_t max_samples) {
    std::lock_guard<std::mutex> const guard{sample_cache_lock_};

    std::size_t const nr_available_events{static_cast<std::size_t>(
        EventSequenceUtil::GetSequenceDifference(newest_undesired_sequence, last_event_sequence_))};  // 后 - 前，简单作差
    std::size_t const nr_requested_events{std::min(nr_available_events, max_samples)};

    // Skip to the newest events requested
    typename SomeIpSampleEntryContainer::size_type skip{0};
    if (nr_requested_events < invisible_sample_cache_.size()) {
      skip = invisible_sample_cache_.size() - nr_requested_events;
    }

    for (size_t it{skip}; it < invisible_sample_cache_.size(); ++it) {
      // Get sample. This call may do deserialization
      SomeIpSampleCacheEntryResult get_sample_result{GetSample(invisible_sample_cache_[it])};

      // Case of sample successfully deserialized
      if (get_sample_result.HasValue()) {
        typename SomeIpSampleCacheEntryResult::value_type const& deserialization_result{get_sample_result.Value()};
        // Apply filter
        bool const have_filter{filter != nullptr};
        bool relevant{};
        if (have_filter) {
          relevant = filter(*deserialization_result.sample_ptr);
        } else {
          relevant = true;
        }
        if (relevant) {
          // Update E2E state
          e2e_state_ = deserialization_result.e2e_result.GetState();
          // Emplace sample pointer in the visible cache
          visible_cache.emplace_back(deserialization_result.sample_ptr,
                                     deserialization_result.e2e_result.GetCheckStatus());
        }
      } else {
        logger_.LogError([](ara::log::LogStream& s) { s << "Deserialization error occurred."; }, __func__, __LINE__);
      }
    }
    return last_event_sequence_;
  }
```

`SomeIpSampleCacheEntry`的`GetSample()`
`invisible_sample_cache_`存储的全是`std::shared_ptr`，`invisible_sample_cache_`这一`deque`持有的元素，什么时候`erase`？
观察到序号一直增长，但是`invisible_sample_cache_`的size应该是限定在Subscribe的长度了(通过`assert`)，也就是一直为1，但是序号仍然在记录，是什么意思？



```cpp
  /*!
   * \brief       Retrieves stored deserialized sample or performs deserialization
   * \tparam      DeserializerConfiguration configuration for intializing the deserializer without E2E configuration
   * \return      A pair holding the deserialized sample and E2E default result
   * \pre         -
   * \context     App
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \vprivate
   * \synchronous TRUE
   *
   * \internal
   * - If event sample is serialized
   *   - Deserialize event sample
   *   - If deserialization is successful
   *     - Return deserialized sample and default E2E result
   *   - Else
   *     - Reset deserialized sample container and return error
   *   - Release memory of serialized sample
   * - Else if event sample is deserialized
   *   - Return deserialized sample and default E2E result
   * - Else if deserialization error
   *   - Return error
   * \endinternal
   */
  template <typename DeserializerConfiguration,
            std::enable_if_t<std::is_void<typename DeserializerConfiguration::E2eProfileConfigType>::value,
                             std::uint8_t> = 0U>
  auto GetSample() -> GetSampleResult {
    GetSampleResult sample_data{DeserializationState::kDeserializationError};

    ::amsr::e2e::Result const e2e_result{ara::com::E2E_state_machine::E2EState::NoData,
                                         ara::com::E2E_state_machine::E2ECheckStatus::NotAvailable};

    // Protect concurrent access to the shared resources
    {
      std::lock_guard<std::mutex>{deserialization_mutex_};
      switch (state_) {
        case DeserializationState::kSerialized: {
          // Deserialize event sample
          SampleDeserializationResult deserialization_result{DeserializeSample<DeserializerConfiguration>(
              serialized_sample_->GetView(0U), serialized_sample_->size())};

          if (deserialization_result.HasValue()) {
            deserialized_sample_ = std::move(deserialization_result.Value());
            sample_data.EmplaceValue(DeserializationResult{deserialized_sample_, e2e_result});
            state_ = DeserializationState::kDeserialized;
          } else {
            state_ = DeserializationState::kDeserializationError;
          }
          // Release memory buffer of serialized sample
          serialized_sample_.reset();
          break;
        }
        case DeserializationState::kDeserialized: {
          sample_data.EmplaceValue(DeserializationResult{deserialized_sample_, e2e_result});
          break;
        }
        case DeserializationState::kDeserializationError:
        default: {
          // Return deserialization error.
          break;
        }
      }
    }
    return sample_data;
  }
```











### `GetCachedSamples()`
直接就把`visible_sample_cache_`返回出去
```cpp
  // VECTOR NL VectorC++-V6.6.1: MD_SOCAL_VectorC++-V6.6.1_AbortWithSingleReturnStatement
  const SampleContainer& GetCachedSamples() const noexcept {
    // Protect concurrent modification of visible_sample_cache_ by Update(), GetCachedSamples(), Cleanup() APIs.
    std::lock_guard<std::mutex> guard(visible_sample_cache_lock_);
    return visible_sample_cache_;
  }
```




#### 总结
首先，`OnEvent`接口每次调用维护队尾元素的序号`last_event_sequence_`。

# New
`UpdateNewest`不断维护一个`last_known_sequence_`

# Last
`UpdateLast`不断维护一个`base_sequence_{0}`和`last_known_sequence_{0}`

二者都先通过`GetSamples`得到一个`last_known_sequence`，只不过传入函数的入参`newest_undesired_sequence`的含义不同

# `GetSamples`
`nr_available_events` = `last_event_sequence_` - `newest_undesired_sequence`
表示**目前可用的events的个数**

`nr_requested_events` = std::min(`nr_available_events` - `max_samples`)，毕竟最多只能提供`nr_available_events`个
PS: `max_samples`是`Subscirbe()`的入参
表示**实际需要的events的个数**

因为要遍历`invisible_sample_cache_`
设置循环的起点: `skip` = `invisible_sample_cache_.size()` - `nr_requested_events`
把`[skip, invisible_sample_cache_.size()]`个元素塞到`visible_sample_cache_`

最后返回`last_event_sequence_`

## `Newest`
简单地记录了上一批到`invisible_sample_cache_`的数据的最后一个序号
永远是取没取过的数据

## `Last`
取的是当前`invisible`里面的`n`个数据(`n`为`Subscribe的入参`)

**核心在于，给`OnEvent`接口收到的event数据每一个进行了编号**










**`ReceiveHandlerService1Event1()`的实现**

调用  **parameter_service_interfaceProxy**  类型的指针指向对象的  **ParameterNotificationEvent**  成员的`GetCachedSamples()`，得到cache（vector）对象的引用。


### 何时触发？
在初始化时创建了**ThreadPool**::**WorkerThread**
（`默认线程池`的线程，名为`vComDef`，
在`runtime`中创建）， 
**EventNotificationTask**  对象被分配给线程池，调用函数调用运算符  **operator()**  时，调用  **ProxyEventBase**  对象的  **NotifySync()**  ，其中调用了设置的  **receive_handler_**

该任务何时分配？
分配任务的直接原因是此处的代码，往线程池添加任务。是接收到event数据时，通过一系列调用触发的函数。
```cpp
  /*!
   * \brief Schedule a task to notify event receive handler on reception of a new event sample.
   * \details Called from "reactor" thread. Invocations of the event receive handler are serialized.
   * \pre Receive handler is set.
   * \context Reactor
   * \threadsafe FALSE
   * \reentrant FALSE
   * \synchronous FALSE
   * \trace SPEC-4980117, SPEC-4980378
   * \trace SPEC-8053498, SPEC-8053499
   *
   * \internal
   * - If receive handler is set
   *   - Get runtime instance.
   *   - Add an EventNotificationTask to the threadpool with a pointer to this object as key.
   *
   * Calls "std::terminate()" if:
   * - "std::bad_alloc" is caught due to "vac::language::make_unique".
   * \endinternal
   */
  void Notify() noexcept override {
    /* Only schedule an event notification task, if a receive handler is set, indicated by
    receive_handler_set_ equals true. This is done for performance reasons. */
    if (receive_handler_set_.load()) {
      ara::com::Runtime& runtime{ara::com::Runtime::getInstance()};

      ThreadPool& pool{runtime.RequestThreadPoolAssignment()};
      // VECTOR NC AutosarC++17_10-A0.1.2: MD_SOCAL_AutosarC++17_10-A0.1.2_Notify
      // VECTOR NC AutosarC++17_10-M0.3.2: MD_SOCAL_AutosarC++17_10-M0.3.2_Notify
      pool.AddTask(vac::language::make_unique<EventNotificationTask>(*this));
    }
  }
```

`ProcessReceivedMessage(); `  ->   `OnSomeIpRoutingMessage`  

## 发送method类型的数据
是在自己创建的`Periodic线程中`，
通过`StartApplicationCmService1_ServiceInterfaceProxy`类型指针（代表该service，且是proxy端）得到`PrxoyMethod`类型的`StartApplicationMethod1`对象，重载了`operator()(ArgsT... args)`，传入参数`method1_value_`，最终返回`ara::core::Future<Output`对象，
（PS：当前示例中，该Method的入参、出参都是`uint8_t`）
```cpp
    /*!
     * \brief Operation will call the method of the ServiceProxyImplInterface.
     * \param[in] args The parameters of this method call.
     * \return A future object for the application user which is used to query for method completion.
     * \pre -
     * \context App
     * \threadsafe FALSE
     * \reentrant FALSE
     * \vpublic
     * \synchronous FALSE
     * \trace SPEC-4980380, SPEC-5951256
     */
    Return operator()(ArgsT... args) noexcept {
      Return return_value;
      // VECTOR NC AutosarC++17_10-A18.5.8: MD_SOCAL_AutosarC++17_10-A18.5.8_FalsePositive
      ProxyImplInterfacePtr const proxy_impl_ptr{proxy_ptr_->GetServiceProxyImplInterface()};
      return_value = (proxy_impl_ptr.get()->*Method)(std::forward<ArgsT>(args)...);
      return return_value;
    }
```
调用`std::shared_ptr<CommunicationBindingInterface>`类型的指针的`GetServiceProxyImplInterface()`方法得到`ProxyImplInterface`（是Proxy的第二个模板参数）类型的裸指针，通过成员函数指针调用传递参数，
以上类对象的构造是在`FindService1Handler`中：
```cpp
      service1_proxy_ =
          std::make_shared<service1::proxy::StartApplicationCmService1_ServiceInterfaceProxy>(service1_handles[0]);
```

`StartApplicationCmService1_ServiceInterfaceProxy`的构造
```cpp
// ====================== Proxy constructor (deprecated) ======================
StartApplicationCmService1_ServiceInterfaceProxy::StartApplicationCmService1_ServiceInterfaceProxy(
  StartApplicationCmService1_ServiceInterfaceProxy::HandleType const& handle) noexcept
  : Base{ConstructInterface(handle)},
    logger_(amsr::socal::internal::logging::kAraComLoggerContextId, amsr::socal::internal::logging::kAraComLoggerContextDescription,
            "StartApplicationCmService1_ServiceInterfaceProxy"),
    StartApplicationMethod1{this},
    StartApplicationEvent1{this},
    StartApplicationField1{this} {

  // Register the proxy. After this call, callbacks might be received.
  // This must be the LAST thing to do while creating the proxy.
  // Otherwise, race-conditions during
  // construction might happen.
  GetServiceProxyImplInterface()->Initialize();
}
```

`ProxyMethod`模板类的声明
注意：有一个模板类型参数是：`成员函数指针`
```cpp
  template <typename ServiceInterface, typename CommunicationBindingInterface, typename ProxyHandleType,
            typename ConcreteMethod,
            ara::core::Future<typename ConcreteMethod::Output> (CommunicationBindingInterface::*Method)(ArgsT...)>
  class ProxyMethod final {
// ...
explicit ProxyMethod(ProxyPtr proxy_ptr) noexcept : proxy_ptr_{proxy_ptr} {}

// ...
  }
```

PS：此处模板参数的实例化是在**用户生成代码部分**：
！！！这个成员函数指针类型的模板类型参数联系实际传入的是：
此处的模板参数实例化是通过`using声明`，在外层的`MethodParameters`用`std::uint8_t const&*`实例化，然后通过`::`访问到内部的`ProxyMethod`，并实例化模板参数（其中的成员函数指针的名字Method如同形参名，没有意义）
```cpp
/*!
 * \brief Type alias for service method 'StartApplicationMethod1', that is part of the proxy.
 *
 * \trace SPEC-4980346
 */
using StartApplicationMethod1 = ::amsr::socal::internal::methods::MethodParameters<
  
    std::uint8_t const&>::
        ProxyMethod<startapplication::cm::service1::StartApplicationCmService1_ServiceInterface, startapplication::cm::service1::internal::StartApplicationCmService1_ServiceInterfaceProxyImplInterface, startapplication::cm::service1::internal::StartApplicationCmService1_ServiceInterfaceHandleType, startapplication::cm::service1::internal::methods::StartApplicationMethod1,
                    &startapplication::cm::service1::internal::StartApplicationCmService1_ServiceInterfaceProxyImplInterface::MethodStartApplicationMethod1>;
```

！！！总之，`return_value = (proxy_impl_ptr.get()->*Method)(std::forward<ArgsT>(args)...);`的调用实际是调用到了**src-gen**的`StartApplicationCmService1_ServiceInterfaceProxyImplInterface`类的`MethodStartApplicationMethod1`函数
内部实现：
```cpp
  /*!
   * \brief Realized getter for the method 'StartApplicationMethod1' on SOME/IP level.
   *
   * \param[in] input_argument Input argument of type std::uint8_t.
   * \return An ara::core::Future for return value of the method.
   *
   * \pre              -
   * \context          App
   * \threadsafe       FALSE
   * \reentrant        FALSE
   * \synchronous      FALSE
   */
  ::ara::core::Future<::startapplication::cm::service1::proxy::methods::StartApplicationMethod1::Output> MethodStartApplicationMethod1(const std::uint8_t& input_argument) override {
    // Build struct with all input arguments
    ::startapplication::cm::service1::internal::methods::StartApplicationMethod1::Input const input_struct{input_argument};
    return method_StartApplicationMethod1_.HandleMethodRequest(input_struct);
  }
```

此处调用对象为建模产生的method对象的`HandleMethodRequest()`方法，说明对于同一个method，`每一次method request`，`session_id`递增
关键调用`HandleMethodRequest`的内部实现：
```cpp
  /*!
   * \brief Serialization of a method.
   * \details In case of memory allocation failure log and exit.
   *
   * \param[in] input_arguments Input struct containing all input arguments.
   * \return ara::core::Future for return value of the method.
   *
   * \pre              -
   * \context          App
   * \threadsafe       TRUE
   * \reentrant        TRUE
   * \synchronous      TRUE
   * \exceptionsafety  STRONG No undefined behavior or side effects.
   *
   * \internal
   * - Get the session ID for this method request.
   * - Construct the SOME/IP header using session ID and other relevant information.
   * - Instantiate packet builder for SOME/IP packet to allocate memory and serialization of the packet contents.
   * - Pass the serialized request to "ProxyRequestHandler" for further transmission.
   * \endinternal
   *
   * \trace SPEC-4981292, SPEC-4980087, SPEC-4980122
   * \trace SPEC-4981447, SPEC-4981448
   * \trace SPEC-8053501
   */
  Future HandleMethodRequest(Input const input_arguments) {
    logger_.LogDebug([](ara::log::LogStream const&) {}, __func__, __LINE__);

    // Build SOME/IP header
    ::amsr::someip_protocol::internal::SomeIpMessageHeader header{};
    header.service_id_ = RequiredSomeIpServiceInstance::kServiceId;
    header.method_id_ = kMethodId;
    header.client_id_ = proxy_binding_.GetClientId();
    header.message_type_ = ::amsr::someip_protocol::internal::SomeIpMessageType::kRequest;
    header.protocol_version_ = ::amsr::someip_protocol::internal::kProtocolVersion;
    header.interface_version_ = RequiredSomeIpServiceInstance::kMajorVersion;
    header.return_code_ = ::amsr::someip_protocol::internal::SomeIpReturnCode::kOk;
    header.length_ = 0U;  // Finally updated by the serializer.

    // Compute next session ID for header
    {
      // Ensure thread-safe increment of session id
      std::lock_guard<std::mutex> const guard{session_id_lock_};
      ::amsr::someip_protocol::internal::SessionId session_id{session_};
      // Update the header
      header.session_id_ = session_id;
      // Increment the session id
      ++session_;
    }

    // Allocate and build SOME/IP packet
    PacketBuilderType packet_builder{logger_, tx_buffer_allocator_};
    BufferPtrType result_buffer{packet_builder.BuildAndSerializePacket(header, input_arguments)};

    // Transmit request
    ProxyRequestHandler<Output> request_handler{header.session_id_, logger_};
    // Ensure thread-safe request transmission
    std::lock_guard<std::mutex> const guard{transmission_lock_};
    return request_handler.HandleSerializedMethodRequest(proxy_binding_, std::move(result_buffer),
                                                         pending_request_map_);
  }
```

创建`SomeIpMessageHeader`，发送`SOME/IP请求消息`
计算`session_id`（修改`ProxyMethodManager`的操作是原子的）
然后通过`BuildAndSerializePacket()`构建`SOME/IP packet`。
使用`ProxyRequestHandler对象`来传递request
后续，直接在用户层代码对`future对象`使用`GetResult()`取得RPC的返回值

`HandleMethodRequest`中的关键调用`HandleSerializedMethodRequest`
```cpp
  /*!
   * \brief Create a future object for an already serialized method request.
   *
   * \details The assembled request is being forwarded. If forwarding fails, this will be
   *          signaled via an appropriate exception set in the promise.
   *
   * \tparam RequiredServiceInstance Deduced type of the sink to forward the request to.
   * \tparam PendingRequestMap Deduced type of the map containing the logic for storing the pending request.
   *
   * \param[in]      proxy_binding     Reference is used to forward the message.
   * \param[in]      packet            SOME/IP message packet
   * \param[in, out] pending_requests  Reference to a map of pending requests
   *
   * \return ara::core::Future for return value of this service method
   *
   * \pre              A serialized SOME/IP request packet is available.
   * \context          App
   * \threadsafe       FALSE
   * \reentrant        FALSE
   * \synchronous      TRUE
   * \exceptionsafety  STRONG No undefined behavior or side effects.
   *
   * \internal
   * - Store a promise for this request into the pending requests map
   * - If the promise was added successfully to the map:
   *   - Send the method request
   *   - Handle transmission error if transmission failed
   * - Otherwise handle error due to duplicated session id
   * \endinternal
   *
   *
   * \trace SPEC-4980140
   */
  template <typename RequiredServiceInstance, typename PendingRequestMap>
  auto HandleSerializedMethodRequest(RequiredServiceInstance& proxy_binding,
                                     ::vac::memory::MemoryBufferPtr<osabstraction::io::MutableIOBuffer> packet,
                                     PendingRequestMap& pending_requests) -> ara::core::Future<ReturnType> {
    ara::core::Optional<ara::core::Future<ReturnType>> added_request{MovePromiseToPendingRequestMap(pending_requests)};
    ara::core::Future<ReturnType> result{};
    if (added_request.has_value()) {
      result = std::move(added_request.value());
      bool const request_sent{proxy_binding.SendMethodRequest(std::move(packet))};
      if (!request_sent) {
        OnTxError(pending_requests);
      }
    } else {
      result = OnSessionIdDuplicated();
    }

    return result;
  }
```
`StoreRequestInMap`的操作是：
PS：该操作是原子的
创建`std::pair<RequestKey, Promise<Output>>`
返回`optional_future`
把当前session_id对应的request存入map。

`StoreRequestInMap`
```cpp
  ara::core::Optional<ara::core::Future<Output>> StoreRequestInMap(RequestKey const& request_key) noexcept {
    std::lock_guard<std::mutex> const protect_access{map_access_protection_};

    ara::core::Promise<Output> promise{};
    ara::core::Optional<ara::core::Future<Output>> optional_future{};
    std::pair<typename MapType::iterator, bool> const pending_request_stored{
        pending_requests_.emplace(request_key, std::move(promise))};

    if (pending_request_stored.second) {
      typename MapType::iterator const iterator{pending_request_stored.first};
      ara::core::Promise<Output>& moved_promise{iterator->second};
      optional_future.emplace(moved_promise.get_future());
    }

    return optional_future;
  }
```
注意：该函数的入参其实就是`session_id_`，如果`session_id`重复，
如果已经存在具有相同键的元素，则可以通过`if (pending_request_stored.second)`判断得出，那么就会返回空的`optional_future`
！！！即便是相同类型的request，也是不同的`session_id`
（`session_id`何时会重复？--> 多线程发起request时，session_id会重复？现在的处理逻辑是：会丢弃数据）

如果发送失败，则调用`OnTxError`：
```cpp
  template <typename PendingRequestMap>
  void OnTxError(PendingRequestMap& pending_requests) {
    std::pair<ara::core::Promise<ReturnType>, bool> request{pending_requests.MoveOutRequest(session_id_)};
    bool const promise_found{request.second};
    if (promise_found) {
      ara::core::Promise<ReturnType> promise{std::move(request.first)};
      promise.SetError(
          {ara::com::ComErrc::network_binding_failure, "Transmission error: Method request was not sent."});
    }
  }
```
移除`PendingRequestMap`对象中当前session_id_对应的`ara::core::Promise<Output>`类型的request，并取出对应的`promise`，调用`SetError`
如果`session_id`重复，则调用`OnSessionIdDuplicated`


## 接收method类型的数据
### 分配解包任务
PS：要解包后，才得到`method_id`以及`session_id`

！！！在`vComReactor`线程中：
使用`epoll`技术，等待事件触发，在`StartReceiving()`中注册回调`OnReceiveCompletion`，在层层调用栈中，
当`ProcessSomeIpMessage`调用时，执行：
`client_manager_->HandleReceive(instance_id, someip_header, std::move(someip_message));`
根据`header.message_type_`判断消息类型，得知是`KResponse`类型，
执行`proxy_binding->HandleMethodResponse(header, std::move(packet));`
然后在src-gen中，对`header_id_.method_id_`做判断，判断是预设的哪一个method
```cpp
  switch (method_id) {
    case methods::ProxyStartApplicationMethod1::kMethodId: {
      method_StartApplicationMethod1_.HandleMethodResponse(header, std::move(packet));
      break;
    }
    // ...其他method
  }
```
`HandleMethodResponse()`：
是`ProxyMethodManager`类的方法
```cpp
  void HandleMethodResponse(::amsr::someip_protocol::internal::SomeIpMessageHeader const& header,
                            ::vac::memory::MemoryBufferPtr<osabstraction::io::MutableIOBuffer> packet) {
    response_handler_.HandleResponse(header, std::move(packet), pending_request_map_);
  }
```
在`HandleResponse`中，
根据`session_id`，从`ProxyMethodManager`维护的`pending_requests`取出相应的`Promise`
！！！即便`request类型`相同，`session_id`也是不同的，这样就能确保相同类型的两个request，分别得到处理（放到线程池多线程地处理）

`AddTask`：`PositiveResponseTask`
```cpp
  /* Schedule a PositiveResponseTask to set the method response to the promise to notify user */
  bool const task_scheduled{thread_pool.AddTask(vac::language::make_unique<PositiveResponseTask>(
      std::move(packet), std::move(request.first), logger_, this))};
```
PS：其中`request.first`是`ara::core::Promise<Output>`类型


### 解析response包
！！！在`vComResp0-0`线程中：
`PositiveResponseTask`重载了`operator()()`，作为任务被分配给线程池的线程执行
```cpp
    void operator()() override {
      ProxyResponseHandler::HandlePositiveResponseSync(std::move(packet_), std::move(pending_request_), logger_);
    }
```

`HandlePositiveResponseSync`：
```cpp
  static void HandlePositiveResponseSync(::vac::memory::MemoryBufferPtr<osabstraction::io::MutableIOBuffer> packet,
                                         Promise&& pending_request,
                                         ::amsr::someip_binding::internal::logging::AraComLogger const& logger) {
    // Prepare Reader
    ::vac::memory::MemoryBuffer<osabstraction::io::MutableIOBuffer>::MemoryBufferView packet_view{packet->GetView(0U)};
    ::amsr::someip_protocol::internal::deserialization::BufferView const body_view{
        // VECTOR Next Line AutosarC++17_10-M5.2.8:MD_SOMEIPBINDING_AutosarC++17_10-M5.2.8_conv_from_voidp
        static_cast<std::uint8_t*>(packet_view[0U].base_pointer), packet->size()};

    // Skip the header
    ::amsr::someip_protocol::internal::deserialization::BufferView const buffer_view{
        body_view.subspan(::amsr::someip_protocol::internal::kHeaderSize,
                          body_view.size() - ::amsr::someip_protocol::internal::kHeaderSize)};
    ::amsr::someip_protocol::internal::deserialization::Reader reader{buffer_view};

    // DeserializePositiveResponse
    // VECTOR NL AutosarC++17_10-A18.5.8: MD_SOMEIPBINDING_AutosarC++17_10-A18.5.8_Large_packets_allocated_on_stack
    std::shared_ptr<Output> output{std::make_shared<Output>()};
    bool const deserialization_status{ResponseOkDeserializer::Deserialize(reader, *output)};

    if (deserialization_status) {
      pending_request.set_value(*output);
    } else {
      logger.LogError([](ara::log::LogStream& s) { s << "Deserialization error occurred"; }, __func__, __LINE__);
      pending_request.SetError({ara::com::ComErrc::network_binding_failure, "Deserialization error occurred"});
    }
  }
```
对`packet`解包，执行`Promise`的`set_value`












## skeleton端method处理
而在skeleton端是这样实现的：
默认线程池线程处理该任务：（是生成出来的代码）
```cpp
  /*!
   * \brief   Operator gets called when method invocation is planned in the frontend.
   * \details It shall be called only once for each instance.
   * \pre -
   * \context     Callback
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \synchronous TRUE
   *
   */
  void operator()() override {

    ara::core::Result<::startapplication::cm::service1::internal::fields::StartApplicationField1::Output> result{skeleton_->StartApplicationField1.Get().GetResult()};
    if (result.HasValue()) {
      response_handler_.SerializeAndSendMethodResponse<Serializer>(header_, result.Value());
    } else {
      response_handler_.SerializeAndSendApplicationErrorMethodResponse(header_, result.Error());
    }
  }
```

调用  **SkeletonField**  <std::uint8_t, FieldConfigStartApplicationField1>对象的  **Get**  ()，其中会调用为field注册的  **getHandler**  ，该  **Get**  ()方法返回一个future对象，对其调用  **GetResult**  ()即可取得值。

这里的逻辑是先触发（skeleton端的）在用户层代码设置的field的get handler，然后把当前field的值打包到Method Response消息里。







# SOME/IP daemon接收数据
![image.png](https://atlas.pingcode.com/files/public/654639b70bd6b6cbbf7c63e9/origin-url)

在reactor线程中，执行  **StartReactor**  ()，其中调用  **HandleEvents**  (std::chrono::milliseconds::max())

```cpp
/*!
 * \internal
 * - Wait for events.
 * - If waiting was interrupted by a signal
 *   - Return kSignal.
 * - If an unrecoverable error was encountered
 *   - Abort the execution of the process.
 * - If no events were reported
 *   - Return kTimeout.
 * - Otherwise if events were reported
 *   - For each event
 *     - If it is an unblock event.
 *       - Handle the unblock event.
 *     - Otherwise handle the IO event.
 *   - Return kEventsHandledOrUnblock.
 * \endinternal
 */
Result<UnblockReason> Reactor1::HandleEvents(std::chrono::milliseconds timeout) noexcept {
  Result<UnblockReason> result{Result<UnblockReason>::FromError(OsabErrc::kFatal)};

  // The following cast is valid because the maximum size of epoll_events_ is limited to kMaxNumCallbacks(8191) during
  // the construction of the Reactor1 object.
  // VECTOR Next Line AutosarC++17_10-A3.9.1: MD_OSA_A3.9.1_EpollApi
  int const num_events{epoll_wait(epoll_fd_, epoll_events_.data(), static_cast<int>(epoll_events_.size()),
                                  GetEpollWaitTimeout(timeout))};

  if (num_events < 0) {
    if (errno == EINTR) {  // VECTOR Same Line AutosarC++17_10-M19.3.1: MD_OSA_M19.3.1_PosixApi
      result.EmplaceValue(UnblockReason::kSignal);
    } else {
      // The other errors EBADF(epoll_fd_ is invalid), EFAULT(No permissions to write to epoll_events_) and
      // EINVAL(epoll_fd_ does not point to epoll instance) can only happen as an effect of another major error, are
      // not recoverable and are therefore treated as a fatal error.
      ara::core::Abort("Reactor1::HandleEvents(): Fatal error reported by epoll_wait().");
    }
  } else if (num_events == 0) {
    result.EmplaceValue(UnblockReason::kTimeout);
  } else {
    // The following cast is valid because num_events has to be larger than 0 to execute this code.
    for (std::size_t i{0}; i < static_cast<std::size_t>(num_events); ++i) {
      const struct epoll_event& epoll_ev{epoll_events_[i]};
      CallbackHandle const callback_handle{epoll_ev.data.u64};
      if (callback_handle == internal::kUnblockCallbackHandle) {
        HandleUnblock();
      } else {
        HandleOneEvent(callback_handle, epoll_ev);
      }
    }
    result.EmplaceValue(UnblockReason::kEventsHandledOrUnblock);
  }
  return result;
}
```

（PS：调试someipd而不启动别的app的话，会发现最后会阻塞于此处：  **epoll_wait()**

此处  **HandleEvents()**  函数传入的是std::max()（如果未配置的话）

**此处使用了系统调用：epoll_wait(epoll_fd, epoll_events.data(), events_num, time)**

参数说明：
-   `epfd`  ：epoll实例的文件描述符，通过  **epoll_create**  函数创建。
-   `events`  ：用于存储就绪事件的数组，每个事件对应一个  **文件描述符**  。
-   `maxevents`  ：  **events**  数组的大小，表示最多可以存储多少个就绪事件。
-   `timeout`  ：等待事件发生的  **超时时间**  ，单位是毫秒。传入-1表示无限等待，传入0表示立即返回，传入正整数表示等待指定时间。

函数返回值：
- 成功时，返回就绪事件的个数。
- 失败时，返回-1，并设置errno来指示错误原因。

PS：使用  **epoll_wait**  函数的一般步骤如下：
1. 创建一个epoll实例，通过  **epoll_create**  函数。
1. 向epoll实例中注册需要监听的文件描述符和事件，通过  **epoll_ctl**  函数。
1. 调用  **epoll_wait**  函数等待事件发生。
1. 处理就绪的事件。




在  **Reactor1**  的  **Preconstruct**  ()函数中：

有调用  **epoll_create1**  ()

而在osabstraction模块中定义Reactor1类时，嵌套定义了  **ConstructionToken类**  ，并再度嵌套定义了结构体  **ConstructionResources**  ：（  **int**  类型的，虽然起了别名叫  **NativeHandle**  ，用于表示I/O操作相关的文件描述符）epoll_fd_、unblock_event_fd_



有调用  **epoll_ctl**  ()

（信号处理也会注册事件，通过SignalManager的各种方法）

在  **ServiceDiscovery**  对象的  **Initialize**  ()中调用  **CreateClientServiceInstanceStateMachines**  ()中，find出sd endpoint（通过  **ServiceDiscoveryEndpointUniquePtr访问**  ），监听多播地址

![image.png](https://atlas.pingcode.com/files/public/6551bb560bd6b6cbbf7cf8df/origin-url)

其中打开了单播地址和多播地址对应的socket，在  **UDPSocket**  ::  **OpenSocket**  ()中，创建  **EventTypes**  对象，调用  **SetReadEvent**  ，通过指向要注册的reactor对象的指针，调用  **Register**  ()，传入：

**NativeHandle**  、  **EventTypes**  、  **lambda表达式**  （其中调用了  **HandleRead**  ，可处理Sd消息）

通过将函数对象封装到自定义结构体中，并将该结构体与文件描述符关联，可以实现将函数对象转换为epoll可以监听的事件。



还有注册事件是在：  **SomeIpDaemonClass**  对象调用  **Initialize**  ()中，

调用了  **ApplicationManager**  对象的  **Listen**  (ipc1::  **UnicastAddress**  )，实际上是打开一个(42,42)Unix Domain Sokcet，监听其他APP发来的数据，其中调用了  **ConnectionAcceptor**  对象的  **Listen**  ()方法，传入了一个function，  **ApplicationAcceptor**  对象set这个function后，（function的内容是：  **OnAccept**  ，而  **OnAccept**  的内容是：  **CreateApplication**  (std::unique_ptr<  **ApplicationConnection**  >)，主要做的是：创建  **ApplicationType**  实例，传给新接收的IPC连接（遍历给定的application pool，搜索一个空闲的entry来创建一个新的application））

调用  **ListenAsync**  ()：（开始异步接收消息）

调用  **AcceptAsync**  (function)，它会根据状态判断，调用  **ChangeReadObservation**  ()：

  `this->reactor_.AddMonitoredEvents(this->reactor_handle_, events);`  







在  **Reactor1**  ::  **HandleEvents**  ()的下面的逻辑：

调用了  **HandleOneEvent(callback_handle, epoll_ev)**

就是开始处理连接的：





**OnMessageAvailable**  ()（该函数何时调用？）

```cpp
/*!
 * \internal
 * - Log a message containing the passed-in message length.
 * - Set the return value as a reference to an empty buffer.
 * - If no memory is allocated for the new message that is to be received,
 *   - Allocate the required memory and set the return value as a reference to the new buffer.
 * - Else, log an error about the reception of a new message while the previous one is not handled yet.
 * - Return the return value.
 * \endinternal
 */
ara::core::Span<osabstraction::io::MutableIOBuffer> ApplicationConnection::OnMessageAvailable(
    std::size_t message_length) {
  logger_.LogVerbose([&message_length](ara::log::LogStream& s) { s << "Message length " << message_length; }, __func__,
                     __LINE__);

  ara::core::Span<osabstraction::io::MutableIOBuffer> buffer{};

  if (receive_memory_buffer_ == nullptr) {
    if (message_length >= (amsr::someipd_app_protocol::internal::kGenericMessageHeaderLength +
                           amsr::someipd_app_protocol::internal::kSpecificMessageHeaderLength)) {
      buffer = PrepareReceiveMemoryBuffer(message_length);
    }
  } else {
    logger_.LogError(
        [](ara::log::LogStream& s) {
          s << "Trying to receive a new message while the previous one has not been completed yet";
        },
        __func__, __LINE__);
  }

  return buffer;
}
```





**其中会调用：PrepareReceiveMemoryBuffer**  ()（可以看出SomeipDaemon是怎么接收数据的）

```cpp
  /*!
   * \brief       Sets up a new memory buffer for the next incoming message.
   *
   * \param[in]   message_length The total length of the message expected next.
   * \return      A view to the I/O vector container pointing to the memory regions for the next incoming message.
   * \pre         message_length >= kHeaderLength
   * \context     Callback
   * \reentrant   FALSE
   *
   * \internal
   * - Reset the current reception buffer.
   * - Calculate payload memory buffer size (passed message length - header length)
   * - Check if the body message exists:
   *   - Allocate new memory buffer with size of message_length using receive_message_allocator_.
   *   - If memory allocation succeeded (set to receive_message_body):
   *     - Create and fill the container of {receive_generic_header, receive_specific_header, receive_message_body}
   *   - Otherwise:
   *     - Log a warning that memory allocation failed.
   *     - Abort the execution
   * - Otherwise (no body exists):
   *   - Set the result to the default I/O vector container without body.
   * - Return the result.
   * \endinternal
   */
  MutableIOBufferContainerView PrepareReceiveMemoryBuffer(std::size_t const message_length) {
    MutableIOBufferContainerView view;

    reception_buffer_.receive_iovec_container.clear();
    reception_buffer_.receive_message_body.reset();

    // Allocate memory only for the message payload, the generic message header and the specific header are stored
    // separately
    std::size_t const payload_memory_buffer_length{message_length - kHeaderLength};

    if (payload_memory_buffer_length > 0U) {  // Memory Allocation Necessary
      // Try to allocate required size
      reception_buffer_.receive_message_body = receive_message_allocator_.Allocate(payload_memory_buffer_length);
      reception_buffer_.receive_iovec_container.reserve(
          3U);  // 3U is enough to hold the generic_header_view, specific_header_view and receive_message_view

      // Instantiate a I/O vector container with the entry for the message body/payload
      vac::memory::MemoryBuffer<osabstraction::io::MutableIOBuffer>::MemoryBufferView const receive_message_body_view{
          reception_buffer_.receive_message_body->GetView(0U)};
      assert(receive_message_body_view.size() == 1);  // Only MemoryBuffer objects with a single view are support.
      osabstraction::io::MutableIOBuffer const receive_message_body_iovec{receive_message_body_view[0]};

      reception_buffer_.receive_iovec_container = reception_buffer_.receive_iovec_container_no_body;
      reception_buffer_.receive_iovec_container.push_back(
          {receive_message_body_iovec.base_pointer, receive_message_body_iovec.size});

      view = MutableIOBufferContainerView{reception_buffer_.receive_iovec_container.data(),
                                          reception_buffer_.receive_iovec_container.size()};
    } else {  // No Memory Allocation necessary (No body)
      view = MutableIOBufferContainerView{reception_buffer_.receive_iovec_container_no_body.data(),
                                          reception_buffer_.receive_iovec_container_no_body.size()};
    }

    return view;
  }
```

**用于为接收到的message分配内存**

此处是  **ApplicationConnection**  类的  **FlexibleUniqueMemoryBufferAllocator**  类型的成员receive_message_allocator_分配内存，分配的长度是：  **接收到的长度**   -   **kGenericMessageHeaderLength **  -   **kSpecificMessageHeaderLength**  ，调用  **Allocator**  的  **Create**  (length)函数，得到  **MemoryBufferPtr**  类型的指针，然后调用  **GetView**  (0)方法（作用：从view_storage_中根据key找到相应的view对象？？？）得到  **MemoryBufferView**  类型的对象，然后使用view[0]对象创建  **MutableIOBuffer**  对象，还使用  **MutableIOBuffer**  的  **base_pointer**  和  **size**  创建了  **MutableIOBufferContainer**  对象。

（PS：  **MutableIOBufferContainer**  是std::  **array**  <osabstraction::io::  **MutableIOBuffer**  , 1U>的别名）

然后把  **receive_specific_header_**  信息导入  **receive_iovec_container_**





























调用MessageReader<>类的  **HandleHeaderOrPayloadReadCompleted**  ()函数

其中调用了  **PrepareDiscardIterationOrReportCompletion**  ()、接着是  **ReportReceptionComplete**  (ara::core::Result<std::size_t>{this->message_stored_bytes_}

之后经过一系列调用：

![image.png](https://atlas.pingcode.com/files/public/65463cb63a27284c5ca1289a/origin-url)

调用到ApplicationConnection::  **ReceiveAsync**  ()

何时传入长度参数？

someip daemon持有  **ApplicationConnection**  类对象，它持有一个osabstraction::io::ipc1::  **Connection**  类对象，用于IPC连接，此处的代码就是把函数对象通过  **ReceiveAsync**  传给子成员  **MessageReader**  对象，在后续的  **MessageReader**  类的  **ReportReceptionComplete**  函数里，该函数被赋给临时对象cb，  **ReportReceptionComplete函数**  具有入参是在此时传入的：

![image.png](https://atlas.pingcode.com/files/public/654744d30bd6b6cbbf7c6550/origin-url)

是  **MessageReader**  的成员  **message_stored_bytes_**  ，那么它是在什么时候赋值的呢？

是在  **MessageReader**  的成员函数  **HandleHeaderOrPayloadReadCompleted**  ()

该函数体现了  **MessageReader**  处于  **状态机模式**

![image.png](https://atlas.pingcode.com/files/public/654746153a27284c5ca12a21/origin-url)

this->  **message_stored_bytes_ **  = this->  **header_**  .  **payload_size_ **  - this->  **remaining_to_read_bytes_**  ;

那么再考察header_.payload_size是何时赋值的：

发现是  **MessageWriter**  去赋值的：

![image.png](https://atlas.pingcode.com/files/public/6547474d3a27284c5ca12a26/origin-url)

在Prepare函数中通过  **CalculateAlloverSize**  ()函数得出：

**CalculateAlloverSize()：**

简单地计算所占字节数

而  **TryToSendMessage**  ()函数的实现：

检查前提条件：

 * 消息参数中的  **缓冲区数量**  不应超过   **kMaximumNumberOfIoBuffers**  。

 * - 消息参数中  **所有缓冲区的总大小**  不应超过  **32**  位。

 检查使用错误：

 * -   **WriterState **  应为   **kIdle**  。

 将   **WriterState **  设置为   **kSendingMessage**  。

保存  **Send completion callback**  。

 * - 调用内部 API 准备发送信息。

 * -   **如果是轮询模式，则尝试通过 SHM 发送。**

 * -   **否则尝试通过事件驱动媒体发送。**

 * - 如果消息已全部发送完毕，则将   **WriterState **  设回   **kIdle**  。

 * - 否则，如果是事件驱动模式，则通过启用write event来继续尝试写入。

 * - 返回：

 * - 发送结果::  **kSendCompleted**  （如果信息已全部发送）。

 * - 否则：SendResult::  **kAsyncProcessingNecessary**  。

（PS：这部分调用MessageWriter是在  **kOfferService**  消息的前提下，发送响应报文的情况

它是从上层的  **ApplicationConnection**  ::  **SendMessage()**  调用过来的，再往上就是  **CommandController**  类的  **OnControlMessage**  ()方法，再往上就是  **Application**  类的  **StartReceive()，再往上会发现是在ApplicationConnection**  ::  **ProcessReceivedMessage()**  中调用的，准确来说是其中的  **reception_function_**  被触发了，也就是接收到消息时触发的回调，它做了什么呢？

--> 在  **ProcessReceivedMessage()函数中触发了回调：**  判断出接收到的消息为  **Control Message**  ，然后开始调用关于“回复SOME/IP消息”的逻辑：准备  **response_packet_**  ，并调用  **Send**  函数，把应答消息发送出去（发送到buffer里）



**而对于普通数据：**

**ProcessReceivedMessage()**  函数有一行注释：将所有剔除了通用头信息的信息直接传递给  **Application**  实例，以便进一步路由到特定的  **Controller**  ，是在  **reception_function_**  ()触发中实现的，而  **reception_function_**  会被赋值为：  **OnMessage()，它会根据消息类型判断后续操作。**

**PS：**  每个  **RoutingControllerType**  类对象持有一个  **RoutingControllerType**  类型的指针

那么对于常规的非控制消息，  **OnMessage**  ()调用的是：  **OnRoutingSomeIpMessage**  (  **SpecificHeaderView**  &,   **MemoryBufferPtr**  )：  
其内部实现：

将消息头部反序列化出来

分配一个’packet router‘的packet

将消息转发给packet router（路由的目的是最终根据required someip service instance的配置，让请求服务的APP收到消息）




## 接收缓冲区是怎样的

如果是多线程发送数据给someip daemon的话，内核的接收缓冲区会一并接受，造成数据混乱

多进程的话，数据不会混乱 --> 内核支持

当多个进程同时使用Unix域套接字向单个进程发送数据时，每个发送进程的数据首先会被复制到内核中的发送缓冲区。  **内核会负责管理这些发送缓冲区，并根据接收进程的状态和可用性来进行数据传输。**

接收进程的缓冲区由内核维护，当接收进程调用接收函数（比如recv()）时，  **内核会将来自不同发送进程的数据复制到接收缓冲区中。这意味着内核需要处理多个发送进程的数据，并确保数据在接收端被正确组装和传递。**

在底层实现中，内核会使用进程间通信的机制来协调多个发送进程和单个接收进程之间的数据传输。这可能涉及到进程调度、数据复制、缓冲区管理等操作，以确保数据的正确传输和接收。



**UdpSocketDataSource**  类中的  **Receive**  ()方法表明someip-daemon使用socket同步地收到的UDP数据，如果数据正确，则把消息长度和数据本身拷贝出来。

（外层是由  **ReadDatagram**  调用的）

创建内存是在  **DatagramMessageReader**  类的  **ReadDatagram**  ()中通过

  `datagram_buffer_.resize(max_datagram_length_);`  做到的，其中  **max_datagram_length_**  是  **1500**

而  **DatagramMessageReader**  是  **ServiceDiscoveryEndpoint**  的成员，  **是一一对应的**

而  **UDPSocket**  类有  **ServiceDiscoveryEndpoint**  类型的引用。









# 整体时序
![image.png](https://atlas.pingcode.com/files/public/6555b8c7279ebff629408a14/origin-url)

![image.png](https://atlas.pingcode.com/files/public/65560a3e82fc7f759fcaac6a/origin-url)
## 在proxy端：
### reactor线程：
用于处理事件（  **BasicIPC**  ：  **Unix Domain Socket**  的连接）
比如，发现  **ReadEvent**  时：一路调用到  **StartReceiving**  ()中传给  **ReceiveAsync**  ()的  **ReceiveCompletionCallback**  ，这个函数主体是  **OnReceiveCompletion**  ()，其内部的重点是  **ProcessReceivedMessage**  ()（PS：不知道为什么紧接着又调用了  **StartReceiving**  ();？？？），后续的处理自然是解包，在  **RoutingController**  类的  **ReceiveServiceDiscoveryServiceInstanceUpdate**  ()方法中，调用  **ClientInterface**  指针的  **HandleFindService**  ()，其内部实现是检查是否有任何异步  **FindService**  任务注册给当前服务，然后给reactor线程分配  **ServiceInstanceUpdateTask**  任务。

### 默认线程池的线程：

大多数proxy回调（  **Event receive handler**  、  **subscription state change handler**  和  **FindServiceHandler callbacks**  ）都由默认线程池触发。


### proxy response thread pool：
感觉没用起来？




# 关于vector如何使用`epoll`
在`ara::core::Initialize()`解析多个配置文件，解析到`./etc/com_application.json`时，
```cpp
amsr::socal::internal::InitializeComponent(ara_com_json_file_path);
```

`Runtime层`
```cpp
return ara::com::Runtime::getInstance().InitializeCommunication(json_config_path);
```

`Runtime层`的`InitializeInternal()`
```cpp
/*! \internal
 * - If the Runtime instance is not alive for multi-threaded applications
 *   - Call HandleErrorNotRunning.
 * - Otherwise, Initialize the reactor thread.
 * - Instantiating the binding pool and initialize all bindings.
 * - Flag that signalizes that runtime and bindings have been initialized.
 * - Start all dynamic actions of all bindings.
 * \endinternal
 */
void Runtime::InitializeInternal() noexcept {
  if (is_running_) {
    HandleErrorAlreadyRunning();
  }

  // Maximum number of callbacks the reactor needs to handle.
  constexpr std::uint16_t max_reactor_callback_count{1024};

  // Initialize the reactor thread. Reactor required before binding initialization because bindings may initialize
  // connections (e.g. SOME/IP).
  // The real communication via the reactor must only be started after StartBindings() is called.
  InitializeReactorAndTimerManager(max_reactor_callback_count);
  InitializeReactorThread(GetReactorThreadConfig());

  // Instantiating the binding pool and initialize all bindings.
  logger_.LogDebug([](ara::log::LogStream& s) { s << "InitializeBindings"; }, __func__, __LINE__);
  InitializeBindings();

  InitializeThreadPools();

  // Flag that signalizes that runtime and bindings have been initialized.
  is_running_ = true;

  // Start all dynamic actions of all bindings (receive / transmit paths, timers etc.)
  logger_.LogDebug([](ara::log::LogStream& s) { s << "StartBindings"; }, __func__, __LINE__);
  StartBindings();
}
```


`InitializeReactorAndTimerManager()`
```cpp
/*! \internal
 * - Create reactor construction token.
 * - If the reactor construction result does not have value
 *   - Set the error code from the construction error.
 *   - Log the fatal message.
 *   - Invoke abortion.
 * - Otherwise, create the reactor and timer manager.
 * \endinternal
 */
void Runtime::InitializeReactorAndTimerManager(std::uint16_t num_of_callbacks) noexcept {
  /* Create reactor construction token */
  ara::core::Result<osabstraction::io::reactor1::Reactor1::ConstructionToken> reactor_construct_result{
      osabstraction::io::reactor1::Reactor1::Preconstruct(num_of_callbacks)};
  if (!reactor_construct_result.HasValue()) {
    ara::core::ErrorCode const error_code{reactor_construct_result.Error()};
    logger_.LogFatal(
        [&num_of_callbacks, &error_code](ara::log::LogStream& s) {
          s << "Failed to construct the reactor with num_of_callbacks = " << num_of_callbacks << ". " << error_code;
        },
        __func__, __LINE__);

    ara::core::Abort("Failed to construct the reactor.");
  }

  reactor_.emplace(std::move(reactor_construct_result.Value()));
  timer_manager_.emplace(&(reactor_.value()));
}
```

`Preconstruct`是把什么预创建了？
`Preconstruct`
```cpp
/*!
 * \internal
 * - Check plausibility of input parameter.
 * - Create epoll instance.
 * - Create eventfd instance.
 * - Set up the unblock event.
 * - If any of the above steps fails skip the remaining steps, close open files and return a resource error.
 * - Otherwise construct and return a ConstructionToken with the allocated resources.
 * \endinternal
 */
Result<Reactor1::ConstructionToken> Reactor1::Preconstruct(std::uint16_t num_callbacks) noexcept {
  Result<Reactor1::ConstructionToken> result{Result<Reactor1::ConstructionToken>::FromError(OsabErrc::kResource)};
  ConstructionToken::ConstructionResources resources;

  if (num_callbacks > internal::kMaxNumCallbacks) {
    ara::core::Abort("Reactor1 construction failed: Invalid number of callbacks specified.");
  }
  resources.epoll_fd_ = epoll_create1(EPOLL_CLOEXEC);
  if (resources.epoll_fd_ == kInvalidNativeHandle) {
    // VECTOR Next Line AutosarC++17_10-M19.3.1: MD_OSA_M19.3.1_PosixApi
    result.EmplaceError(MakeErrorCode(OsabErrc::kResource, errno, "epoll_create1() failed."));
  } else {
    // VECTOR Next Construct AutosarC++17_10-A4.5.1: MD_OSA_A4.5.1_SysCallEnumAsFlags
    // VECTOR Next Line AutosarC++17_10-M5.0.21: MD_OSA_M5.0.21_SysCallEnumAsFlags
    resources.unblock_event_fd_ = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    if (resources.unblock_event_fd_ == kInvalidNativeHandle) {
      // VECTOR Next Line AutosarC++17_10-M19.3.1: MD_OSA_M19.3.1_PosixApi
      result.EmplaceError(MakeErrorCode(OsabErrc::kResource, errno, "eventfd() failed."));
      static_cast<void>(close(resources.epoll_fd_));
    } else {
      struct epoll_event epoll_ev;
      EventTypes events;
      events.SetReadEvent(true);
      epoll_ev = InitEpollStruct(events, internal::kUnblockCallbackHandle);
      if (epoll_ctl(resources.epoll_fd_, EPOLL_CTL_ADD, resources.unblock_event_fd_, &epoll_ev) == -1) {
        // VECTOR Next Line AutosarC++17_10-M19.3.1: MD_OSA_M19.3.1_PosixApi
        result.EmplaceError(MakeErrorCode(OsabErrc::kResource, errno, "epoll_ctl() failed."));
        static_cast<void>(close(resources.epoll_fd_));
        static_cast<void>(close(resources.unblock_event_fd_));
      } else {
        result.EmplaceValue(num_callbacks, resources);
      }
    }
  }
  return result;
}
```
先判断入参：`num_callbacks`是否合规：不能大于8191，当前是1024.
调用`epoll_create1`，创建了一个`epoll实例`。
调用`eventfd`，创建了一个eventfd实例。
如果都创建成功，构造vector定义的`epoll_event`对象，`EventTypes`对象（`SetReadEvent`），
使用``EventTypes`对象来set`epoll_event`对象（epoll_event需要绑定到event上）
调用`InitEpollStruct`，创建系统路径下的头文件中定义的`epoll_event`对象，根据传入的vector定义的`epoll_event`对象，设置这个系统下的`epoll_event`对象的成员：`EPOLLIN`，传入一个`CallbackHandle`，
然后调用：系统调用`epoll_ctl`，

`epoll_ctl`函数用于控制`某一个epoll实例`上的事件，具体是向`epoll实例`中添加一个文件描述符和对应的事件。
文件描述符和对应事件此时还是空的。

`Preconstruct`函数的返回值是`Result<ConstructionToken>`类型，包含信息：`num_callbacks`、`ConstructionResources`对象
`InitializeReactorAndTimerManager`通过`Preconstruct`得到返回值后，从而构造`osabstraction::io::reactor1::Reactor1`类型的对象`reactor_`（由`Runtime`持有）
`Reactor1`的构造函数：（PS：是osabsrataction层的）
```cpp
/*!
 * \internal
 * - Initialize all fields.
 * \endinternal
 */
Reactor1::Reactor1(ConstructionToken&& token) noexcept
    : epoll_fd_(token.ExtractEpollFd()),
      // The size of the epoll_events_ vector has to be at least 1, otherwise epoll_wait() fails.
      epoll_events_(std::max(token.GetNumCallbacks(), std::uint16_t{1})),  // Can throw std::bad_alloc
      unblock_event_fd_(token.ExtractUnblockEventFd()),
      callbacks_(token.GetNumCallbacks()),  // Can throw std::bad_alloc
      callbacks_end_{},
      registration_mutex_{} {
  callbacks_end_ = callbacks_.begin();
}
```
set以下的成员变量
1. epoll本身的fd
2. `std::vector<struct epoll_event>`类型的`epoll_events_`，外层代码里创建的`max_reactor_callback_count`：1024
3. `unblock_event_fd_`，表示非阻塞事件
4. `std::vector<internal::CallbackEntry>`，存储函数指针的集合
5. 函数指针集合的迭代器
6. `registration_mutex_`

（PS：fd由操作系统预先分配，但是应该还没绑定到具体的函数）


`ConstructionToken`类，是资源的集合
```cpp
class ConstructionToken{
// ...
   private:
    /*!
     * \brief Number of callbacks the Reactor should be able to handle.
     */
    std::uint16_t num_callbacks_{};

    /*!
     * \brief Handle for the epoll instance.
     */
    FileDescriptor epoll_fd_{};

    /*!
     * \brief Handle for the unblock eventfd.
     */
    FileDescriptor unblock_event_fd_{kInvalidNativeHandle};
}

```
`epoll_fd_`成员表示epoll实例，`unblock_event_fd_`表示非阻塞类型的event

`InitializeReactorAndTimerManager`还实例化了`vac::timer::ThreadSafeTimerManager`对象：
以上面构造出的`reactor_`对象作为输入进行构造，`ThreadSafeTimerManager`得以构造基类`TimerManager`
`InitializeReactorAndTimerManager`调用完后，调用`InitializeReactorThread`，创建了reactor线程
```cpp
/*! \internal
 * - If the process mode of runtime is KSingleThreaded
 *   - Create a reactor thread in single threaded mode.
 *   - Log error information if thread creation fails.
 * - Otherwise, do not instantiate a reactor thread in polling mode.
 *
 * Calls "ara::core::Abort()" if:
 *  - Thread creation through amsr::thread::Thread::Create() fails
 * \endinternal
 */
void Runtime::InitializeReactorThread(amsr::thread::ThreadConfig const& reactor_thread_config) noexcept {
  logger_.LogDebug([](ara::log::LogStream const&) {}, __func__, __LINE__);

  if (config_.GetProcessingMode() == amsr::socal::internal::configuration::RuntimeProcessingMode::kSingleThreaded) {
    /* Instantiate reactor thread in single threaded mode */
    this->reactor_done_ = false;
    ara::core::Result<amsr::thread::Thread> thread_result{amsr::thread::Thread::Create(reactor_thread_config, [this]() {  // 创建线程，执行以下逻辑
      while (!this->reactor_done_) {
        std::pair<bool, struct timeval> const expiry{timer_manager_.value().GetNextExpiry()};
        if ((expiry.first) &&
            // Conversion must not lead to overflow. In case of overflow we should take the maximum allowed value.
            (std::chrono::duration_cast<std::chrono::seconds>(std::chrono::milliseconds::max()) >
             (std::chrono::seconds{expiry.second.tv_sec}) - std::chrono::seconds{1U})) {
          // Calculate timeout value
          std::chrono::milliseconds const timeout{
              std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::seconds{expiry.second.tv_sec}) +
              std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::microseconds{expiry.second.tv_usec})};
          static_cast<void>(reactor_.value().HandleEvents(timeout));
        } else {
          static_cast<void>(reactor_.value().HandleEvents(std::chrono::milliseconds::max()));
        }
        timer_manager_.value().HandleTimerExpiry();
      }
    })};
    if (thread_result.HasValue()) {
      reactor_thread_ = std::move(thread_result.Value());
    } else {
      logger_.LogFatal(
          [&reactor_thread_config](ara::log::LogStream& s) {
            s << "Could not create reactor thread: " << reactor_thread_config.name;
          },
          __func__, __LINE__);
      ara::core::Abort("Could not create reactor thread.");
    }
  } else {
    /*
     * Do not instantiate a reactor thread in polling mode.
     * User will control event handling by ProcessPolling() API
     */
    reactor_done_ = true;
  }
}
```
创建`reactor`线程，执行以下逻辑：
调用`Timer`提供的接口`GetNextExpiry()`，
执行：
```cpp
static_cast<void>(reactor_.value().HandleEvents(std::chrono::milliseconds::max()));
```

`HandleEvents`
```cpp
/*!
 * \internal
 * - Wait for events.
 * - If waiting was interrupted by a signal
 *   - Return kSignal.
 * - If an unrecoverable error was encountered
 *   - Abort the execution of the process.
 * - If no events were reported
 *   - Return kTimeout.
 * - Otherwise if events were reported
 *   - For each event
 *     - If it is an unblock event.
 *       - Handle the unblock event.
 *     - Otherwise handle the IO event.
 *   - Return kEventsHandledOrUnblock.
 * \endinternal
 */
Result<UnblockReason> Reactor1::HandleEvents(std::chrono::milliseconds timeout) noexcept {
  Result<UnblockReason> result{Result<UnblockReason>::FromError(OsabErrc::kFatal)};

  // The following cast is valid because the maximum size of epoll_events_ is limited to kMaxNumCallbacks(8191) during
  // the construction of the Reactor1 object.
  // VECTOR Next Line AutosarC++17_10-A3.9.1: MD_OSA_A3.9.1_EpollApi
  int const num_events{epoll_wait(epoll_fd_, epoll_events_.data(), static_cast<int>(epoll_events_.size()),
                                  GetEpollWaitTimeout(timeout))};

  if (num_events < 0) {
    if (errno == EINTR) {  // VECTOR Same Line AutosarC++17_10-M19.3.1: MD_OSA_M19.3.1_PosixApi
      result.EmplaceValue(UnblockReason::kSignal);
    } else {
      // The other errors EBADF(epoll_fd_ is invalid), EFAULT(No permissions to write to epoll_events_) and
      // EINVAL(epoll_fd_ does not point to epoll instance) can only happen as an effect of another major error, are
      // not recoverable and are therefore treated as a fatal error.
      ara::core::Abort("Reactor1::HandleEvents(): Fatal error reported by epoll_wait().");
    }
  } else if (num_events == 0) {
    result.EmplaceValue(UnblockReason::kTimeout);
  } else {
    // The following cast is valid because num_events has to be larger than 0 to execute this code.
    for (std::size_t i{0}; i < static_cast<std::size_t>(num_events); ++i) {
      const struct epoll_event& epoll_ev{epoll_events_[i]};
      CallbackHandle const callback_handle{epoll_ev.data.u64};
      if (callback_handle == internal::kUnblockCallbackHandle) {
        HandleUnblock();
      } else {
        HandleOneEvent(callback_handle, epoll_ev);
      }
    }
    result.EmplaceValue(UnblockReason::kEventsHandledOrUnblock);
  }
  return result;
}
```
调用系统调用：`epoll_wait`，想要监听的事件，是通过`epoll_ctl`系统调用注册

## 何时set要监听的事件？
调用栈截取如下：
`someip_daemon_client_.Connect();`

`bool ConnectSync(osabstraction::io::ipc1::UnicastAddress const& address)`

`ChangeWriteObservation(true);`


### fd的mode的设置
`SetBlockingMode(native_handle, SocketBlockingMode{false});`
设置了fd的模式
```cpp
/*!
 * \brief Sets native handle's blocking mode.
 *
 * \note  Since this function doesn't create the native handle it doesn't close it in case of an error.
 *
 * \param[in] native_handle  The native handle.
 * \param[in] enable         The blocking mode to be set. True to enable blocking and false to turn blocking off.
 *
 * \return          -
 *
 * \error           -
 *
 * \context         ANY
 *
 * \pre             Valid native handle.
 *
 * \reentrant       FALSE
 * \synchronous     TRUE
 * \threadsafe      FALSE
 *
 * \vprivate        Vector component internal API
 */
/*!
 * \internal
 *  - Check pre-conditions: x
 *  - Check for usage errors: x
 *  - Get file attributes and abort on any error.
 *  - Set the blocking flag to the passed value.
 *  - Set the file attributes according to the updated flag and abort on any error.
 *  - Return: x
 * \endinternal
 */
auto SetBlockingMode(NativeHandle native_handle, SocketBlockingMode enable) noexcept -> void {
  // VECTOR Next Line AutosarC++17_10-A3.9.1: MD_OSA_A3.9.1_PosixApi
  int flags{::fcntl(native_handle, F_GETFL, 0)};
  if (flags == kApiError) {
    // Function calls ara::core::Abort()
    MapNonSpecialFdOperationError(GetErrorNumber());
  } else {
    if (enable.value) {
      // VECTOR Next Construct AutosarC++17_10-M5.0.21: MD_OSA__M5.0.21_O_NONBLOCK
      // VECTOR Next Construct AutosarC++17_10-M2.13.2: MD_OSA_M2.13.2_O_NONBLOCK
      flags = flags & ~O_NONBLOCK;
    } else {
      // VECTOR Next Construct AutosarC++17_10-M2.13.2: MD_OSA_M2.13.2_O_NONBLOCK
      // VECTOR Next Line AutosarC++17_10-M5.0.21: MD_OSA__M5.0.21_O_NONBLOCK
      flags = flags | O_NONBLOCK;
    }
    if (::fcntl(native_handle, F_SETFL, flags) == kApiError) {
      // Function calls ara::core::Abort()
      MapNonSpecialFdOperationError(GetErrorNumber());
    }
  }
}
```










