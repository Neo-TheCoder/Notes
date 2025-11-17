# 前言
粗略分为 SD，Skeleton，Proxy

# note
1. 紧扣`服务发现`的`状态机`
2. 观察服务发现的流程中，是如何和socket API结合起来的，比如TCP建立连接

3. 异步编程
offer service这种操作，不需要注册回调
需要注册回调的是：
server：订阅状态改变时回调，method回调
client：服务可用时回调，收到event时回调
以及过程中对socket监听的读/写事件



# 问题
## 抓包无someip包
```sh
sudo route add -nv 239.255.0.1 dev enp0s8   # 在系统中手动添加一条静态路由规则，目的是让发往多播地址 239.255.0.1 的流量强制通过指定的网络接口 enp0s8 发出
```

## app是否持有socket？似乎不持有。并且域内通信没走网卡，可能某处存在判断以进行优化
```sh
# 使用两个虚拟机进行通信，可见app自己不持有网络socket，而是someipd持有
schaeffler@schaeffler-ubuntu20:~$ sudo netstat -nap
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:36261         0.0.0.0:*               LISTEN      4348/code-e54c774e0
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1017/sshd: /usr/sbi
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      1249/cupsd
tcp        0      0 0.0.0.0:1947            0.0.0.0:*               LISTEN      900/hasplmd_x86_64
tcp        0      0 127.0.0.1:6010          0.0.0.0:*               LISTEN      4898/sshd: schaeffl
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      799/systemd-resolve
tcp        0      0 192.168.56.103:22       192.168.56.1:53857      ESTABLISHED 4249/sshd: schaeffl
tcp        0      0 192.168.56.103:22       192.168.56.1:54602      ESTABLISHED 4826/sshd: schaeffl
tcp        0      0 127.0.0.1:47352         127.0.0.1:36261         ESTABLISHED 4329/sshd: schaeffl
tcp        0     48 192.168.56.103:22       192.168.56.1:54601      ESTABLISHED 4824/sshd: schaeffl
tcp        0      0 127.0.0.1:36261         127.0.0.1:47352         ESTABLISHED 4348/code-e54c774e0
tcp6       0      0 ::1:6010                :::*                    LISTEN      4898/sshd: schaeffl
tcp6       0      0 :::22                   :::*                    LISTEN      1017/sshd: /usr/sbi
tcp6       0      0 ::1:631                 :::*                    LISTEN      1249/cupsd
udp        0      0 0.0.0.0:35933           0.0.0.0:*                           836/avahi-daemon: r
udp        0      0 0.0.0.0:5353            0.0.0.0:*                           836/avahi-daemon: r
udp        0      0 0.0.0.0:30490           0.0.0.0:*                           18493/./routingmana
udp        0      0 192.168.56.103:30490    0.0.0.0:*                           18493/./routingmana
udp        0      0 192.168.56.103:30509    0.0.0.0:*                           18493/./routingmana
udp        0      0 127.0.0.53:53           0.0.0.0:*                           799/systemd-resolve
udp        0      0 192.168.56.103:68       192.168.56.100:67       ESTABLISHED 841/NetworkManager
udp        0      0 10.0.4.15:68            10.0.4.2:67             ESTABLISHED 841/NetworkManager
udp        0      0 0.0.0.0:1947            0.0.0.0:*                           900/hasplmd_x86_64
udp6       0      0 :::5353                 :::*                                836/avahi-daemon: r
udp6       0      0 :::42963                :::*                                836/avahi-daemon: r
raw6       0      0 :::58                   :::*                    7           841/NetworkManager
raw6       0      0 :::58                   :::*                    7           841/NetworkManager

schaeffler@schaeffler-ubuntu20:~$ sudo netstat -nap | grep notify-samp
unix  2      [ ACC ]     STREAM     LISTENING     95806    18539/./notify-samp  /tmp/vsomeip-1277
unix  3      [ ]         STREAM     CONNECTED     92864    18539/./notify-samp  /tmp/vsomeip-1277
unix  3      [ ]         STREAM     CONNECTED     92863    18539/./notify-samp
```

# routing manager daemon
main函数逻辑：
根据命令行参数，判断是否守护进程化（子进程从`fork`之后开始执行，根据返回值知道是父进程还是子进程，如果是子进程，调`setsid()`脱离terminal，如果是父进程就直接退出）
```cpp
  routing_ = std::make_shared<routing_manager_impl>(this);

// routing_初始化时，涉及到 service_discovery_impl的初始化：
discovery_->init();

service_discovery_impl::init() {
  // 读取配置，填充成员变量
}

// 类似地
discovery_->start();

service_discovery_impl::start() {
  // ...
}

#if defined(__linux__) || defined(ANDROID)
    std::shared_ptr<netlink_connector> local_link_connector_;
```




## 主要业务逻辑
主线程 `while循环`，用于判断`信号处理线程`



# routing_manager_client(区别于someipd， someipd是routing_manager_impl)
## `init()`
可见 创建的都是`local endpoint`
```cpp
void routing_manager_client::init() {
    routing_manager_base::init(std::make_shared<endpoint_manager_base>(this, io_, configuration_));
    {
        std::lock_guard<std::mutex> its_lock(sender_mutex_);
        if (configuration_->is_local_routing()) {
            sender_ = ep_mgr_->create_local(VSOMEIP_ROUTING_CLIENT);
        } else {
#if defined(__linux__) || defined(ANDROID)
            auto its_guest_address = configuration_->get_routing_guest_address();
            auto its_host_address = configuration_->get_routing_host_address();
            local_link_connector_ = std::make_shared<netlink_connector>(
                    io_, its_guest_address, boost::asio::ip::address(),
                    (its_guest_address != its_host_address));
            // if the guest is in the same node as the routing manager
            // it should not require LINK to be UP to communicate

            if (local_link_connector_) {
                local_link_connector_->register_net_if_changes_handler(
                    std::bind(&routing_manager_client::on_net_state_change,
                        this, std::placeholders::_1, std::placeholders::_2,
                        std::placeholders::_3));
            }
#else
            receiver_ = ep_mgr_->create_local_server(shared_from_this());
            sender_ = ep_mgr_->create_local(VSOMEIP_ROUTING_CLIENT);
#endif
        }
    }
}
```



# service provider app
## 主要线程
1. `dispatch线程`
  处理应用层回调
2. `io线程`
  处理底层的socket的监听、数据收发

以notify_sample为例，最上层就两个函数，`init()`和`start()`

## 类设计
app必备抽象基类：`class vsomeip::application`，用于各种操作
实际对象是由`vsomeip_v3::application_impl`实例化而来，该类又持有一个`impl指针`，其对象由runtime提供的`工厂函数create_application()`构造

通过`锁`来进行线程间的同步：初始化线程 / run线程（`offer_service`）
## `init()`
读取配置
注册回调
OfferEvent

`application_umpl::init()`
```cpp
bool application_impl::init() {
    if(is_initialized_) {
        VSOMEIP_WARNING << "Trying to initialize an already initialized application.";
        return true;
    }
    // Application name
    if (name_ == "") {
        const char *its_name = getenv(VSOMEIP_ENV_APPLICATION_NAME);
        if (nullptr != its_name) {
            name_ = its_name;
        }
    }

    std::string configuration_path;

    // load configuration from module
    std::string config_module = "";
    const char *its_config_module = getenv(VSOMEIP_ENV_CONFIGURATION_MODULE);
    if (nullptr != its_config_module) {
        // TODO: Add loading of custom configuration module
    } else { // load default module
#ifndef VSOMEIP_ENABLE_MULTIPLE_ROUTING_MANAGERS
        auto its_plugin = plugin_manager::get()->get_plugin(
                plugin_type_e::CONFIGURATION_PLUGIN, VSOMEIP_CFG_LIBRARY);
        if (its_plugin) {
            auto its_configuration_plugin
                = std::dynamic_pointer_cast<configuration_plugin>(its_plugin);
            if (its_configuration_plugin) {
                configuration_ = its_configuration_plugin->get_configuration(name_, path_);
                VSOMEIP_INFO << "Configuration module loaded.";
            } else {
                std::cerr << "Invalid configuration module!" << std::endl;
                std::exit(EXIT_FAILURE);
            }
        } else {
            std::cerr << "1 Configuration module could not be loaded!" << std::endl;
            std::exit(EXIT_FAILURE);
        }
#else
        configuration_ = std::dynamic_pointer_cast<configuration>(
                std::make_shared<vsomeip_v3::cfg::configuration_impl>(configuration_path));
        if (configuration_path.length()) {
            configuration_->set_configuration_path(configuration_path);
        }
        configuration_->load(name_);
#endif // VSOMEIP_ENABLE_MULTIPLE_ROUTING_MANAGERS
    }

    if (configuration_->is_local_routing()) {
        sec_client_.client_type = VSOMEIP_CLIENT_UDS;
#ifdef __unix__
        sec_client_.client.uds_client.user = getuid();
        sec_client_.client.uds_client.group = getgid();
#else
        sec_client_.client.uds_client.user = ANY_UID;
        sec_client_.client.uds_client.group = ANY_GID;
#endif
    } else {
        sec_client_.client_type = VSOMEIP_CLIENT_TCP;
    }

    // Set security mode
    if (configuration_->is_security_enabled()) {
        if (configuration_->is_security_audit()) {
            security_mode_ = security_mode_e::SM_AUDIT;
        } else {
            security_mode_ = security_mode_e::SM_ON;
        }

        if (security::load()) {
        	VSOMEIP_INFO << "Using external security implementation!";
        	security::initialize();
        }
    } else {
        security_mode_ = security_mode_e::SM_OFF;
    }

    const char *client_side_logging = getenv(VSOMEIP_ENV_CLIENTSIDELOGGING);
    if (client_side_logging != nullptr) {
        client_side_logging_ = true;
        VSOMEIP_INFO << "Client side logging for application: " << name_
                << " is enabled";

        if ('\0' != *client_side_logging) {
            std::stringstream its_converter(client_side_logging);
            if ('"' == its_converter.peek()) {
                its_converter.get(); // skip quote
            }
            uint16_t val(0xffffu);
            bool stop_parsing(false);
            do {
                const uint16_t prev_val(val);
                its_converter >> std::hex >> std::setw(4) >> val;
                if (its_converter.good()) {
                    const std::stringstream::int_type c = its_converter.eof()?'\0':its_converter.get();
                    switch (c) {
                    case '"':
                    case '.':
                    case ':':
                    case ' ':
                    case '\0': {
                            if ('.' != c) {
                                if (0xffffu == prev_val) {
                                    VSOMEIP_INFO << "+filter "
                                    << std::hex << std::setw(4) << std::setfill('0') << val;
                                    client_side_logging_filter_.insert(std::make_tuple(val, ANY_INSTANCE));
                                } else {
                                    VSOMEIP_INFO << "+filter "
                                    << std::hex << std::setw(4) << std::setfill('0') << prev_val << "."
                                    << std::hex << std::setw(4) << std::setfill('0') << val;
                                    client_side_logging_filter_.insert(std::make_tuple(prev_val, val));
                                }
                                val = 0xffffu;
                            }
                        }
                        break;
                    default:
                        stop_parsing = true;
                        break;
                    }
                }
            }
            while (!stop_parsing && its_converter.good());
        }
    }

    std::shared_ptr<configuration> its_configuration = get_configuration();
    if (its_configuration) {
        VSOMEIP_INFO << "Initializing vsomeip application \"" << name_ << "\".";
        client_ = its_configuration->get_id(name_);

        // Max dispatchers is the configured maximum number of dispatchers and
        // the main dispatcher
        max_dispatchers_ = its_configuration->get_max_dispatchers(name_) + 1;
        max_dispatch_time_ = its_configuration->get_max_dispatch_time(name_);

        has_session_handling_ = its_configuration->has_session_handling(name_);
        if (!has_session_handling_)
            VSOMEIP_INFO << "application: " << name_
                << " has session handling switched off!";

        std::string its_routing_host = its_configuration->get_routing_host_name();
        if (its_routing_host != "") {
            is_routing_manager_host_ = (its_routing_host == name_);
            if (is_routing_manager_host_ &&
                    !utility::is_routing_manager(configuration_->get_network())) {
#ifndef VSOMEIP_ENABLE_MULTIPLE_ROUTING_MANAGERS
                VSOMEIP_ERROR << "application: " << name_ << " configured as "
                        "routing but other routing manager present. Won't "
                        "instantiate routing";
                is_routing_manager_host_ = false;
                return false;
#else
            is_routing_manager_host_ = true;
#endif // VSOMEIP_ENABLE_MULTIPLE_ROUTING_MANAGERS
            }
        } else {
            auto its_routing_address = its_configuration->get_routing_host_address();
            auto its_routing_port = its_configuration->get_routing_host_port();
            if (its_routing_address.is_unspecified()
                    || is_local_endpoint(its_routing_address, its_routing_port))
                is_routing_manager_host_ = utility::is_routing_manager(configuration_->get_network());
        }

        if (is_routing_manager_host_) {
            VSOMEIP_INFO << "Instantiating routing manager [Host].";
            if (client_ == VSOMEIP_CLIENT_UNSET) {
                client_ = static_cast<client_t>(
                          (configuration_->get_diagnosis_address() << 8)
                        & configuration_->get_diagnosis_mask());
                utility::request_client_id(configuration_, name_, client_);
            }
            routing_ = std::make_shared<routing_manager_impl>(this);
        } else {
            VSOMEIP_INFO << "Instantiating routing manager [Proxy].";
            routing_ = std::make_shared<routing_manager_client>(this, client_side_logging_, client_side_logging_filter_);
        }

        routing_->init();

#ifdef USE_DLT
        // Tracing
        std::shared_ptr<trace::connector_impl> its_connector
            = trace::connector_impl::get();
        std::shared_ptr<cfg::trace> its_trace_configuration
            = its_configuration->get_trace();
        its_connector->configure(its_trace_configuration);
#endif

        VSOMEIP_INFO << "Application(" << (name_ != "" ? name_ : "unnamed")
                << ", " << std::hex << std::setw(4) << std::setfill('0') << client_
                << ") is initialized ("
                << std::dec << max_dispatchers_ << ", "
                << std::dec << max_dispatch_time_ << ").";

        is_initialized_ = true;
    }

#ifdef VSOMEIP_ENABLE_SIGNAL_HANDLING
    if (is_initialized_) {
        signals_.add(SIGINT);
        signals_.add(SIGTERM);

        // Register signal handler
        auto its_signal_handler =
                [this] (boost::system::error_code const &_error, int _signal) {
                    if (!_error) {
                        switch (_signal) {
                            case SIGTERM:
                            case SIGINT:
                                catched_signal_ = true;
                                stop();
                                break;
                            default:
                                break;
                        }
                    }
                };
        signals_.async_wait(its_signal_handler);
    }
#endif

    if (configuration_) {
        auto its_plugins = configuration_->get_plugins(name_);
        auto its_app_plugin_info = its_plugins.find(plugin_type_e::APPLICATION_PLUGIN);
        if (its_app_plugin_info != its_plugins.end()) {
            for (auto its_library : its_app_plugin_info->second) {
                auto its_application_plugin = plugin_manager::get()->get_plugin(
                        plugin_type_e::APPLICATION_PLUGIN, its_library);
                if (its_application_plugin) {
                    VSOMEIP_INFO << "Client 0x" << std::hex << get_client()
                            << " Loading plug-in library: " << its_library << " succeeded!";
                    std::dynamic_pointer_cast<application_plugin>(its_application_plugin)->
                            on_application_state_change(name_, application_plugin_state_e::STATE_INITIALIZED);
                }
            }
        }
    } else {
        std::cerr << "Configuration module could not be loaded!" << std::endl;
        std::exit(EXIT_FAILURE);
    }

    return is_initialized_;
}
```

关于网络socket（有两类，用于服务发现的，以及用于跨域通信的，其中用户服务发现的显然由someipd持有）是app持有，还是someipd持有，通过实例化什么类型的endpoint可以看出来

在使用vsomeipd的情况下，app必然实例化`routing_manager_client`
```cpp
routing_ = std::make_shared<routing_manager_client>(this, client_side_logging_, client_side_logging_filter_);
```

`routing_manager_client::init()`
```cpp
void routing_manager_client::init() {
    routing_manager_base::init(std::make_shared<endpoint_manager_base>(this, io_, configuration_));
    {
        std::lock_guard<std::mutex> its_lock(sender_mutex_);
        if (configuration_->is_local_routing()) {
            sender_ = ep_mgr_->create_local(VSOMEIP_ROUTING_CLIENT);
        } else {
#if defined(__linux__) || defined(ANDROID)
            auto its_guest_address = configuration_->get_routing_guest_address();
            auto its_host_address = configuration_->get_routing_host_address();
            local_link_connector_ = std::make_shared<netlink_connector>(
                    io_, its_guest_address, boost::asio::ip::address(),
                    (its_guest_address != its_host_address));
            // if the guest is in the same node as the routing manager
            // it should not require LINK to be UP to communicate

            if (local_link_connector_) {
                local_link_connector_->register_net_if_changes_handler(
                    std::bind(&routing_manager_client::on_net_state_change,
                        this, std::placeholders::_1, std::placeholders::_2,
                        std::placeholders::_3));
            }
#else
            receiver_ = ep_mgr_->create_local_server(shared_from_this());
            sender_ = ep_mgr_->create_local(VSOMEIP_ROUTING_CLIENT);
#endif
        }
    }
}
```

`strand`（是`io_context`的`adapter`，功能是确保被投递的handler按顺序执行）
```cpp
template<typename Protocol>
class client_endpoint_impl: public endpoint_impl<Protocol>, public client_endpoint,
        public std::enable_shared_from_this<client_endpoint_impl<Protocol> > {
// ...
protected:
  boost::asio::io_context::strand strand_;
};
```

## `start()`
！！！注意，这里的`io_`由多线程共享，所以是多线程在执行`io_.run()`
```cpp
void application_impl::start() {
// ...
      {   /*
            dispatch存在主从的概念：主的一直运行，从的会根据“活跃标记”
            怀疑服务发现过程中，会创建若干从dispatcher，服务发现完成时，一些从dispatcher会退出（这里的清理是在有任务做完时才清理一把，因为线程实际上总是会被条件变量卡住的，不会占住CPU，一定要被notify才有可能退出，而且这样也不必单独开一个线程，这些从dispacher线程wait的条件就是：自身非活跃线程）
            这里的main_dispatch是单线程执行的生产者-消费者模式，任务队列里是初始化的时候注册的各种service的回调sync_handler，有的是周期性任务，需要reschedule。
            当这里的回调触发的时候，当dispatcher数目少的时候，创建线程：执行dispatch
            主dispatch的特点相比从dispatch是还执行notify
          */
          std::lock_guard<std::mutex> its_lock(dispatcher_mutex_);
          is_dispatching_ = true;
          auto its_main_dispatcher = std::make_shared<std::thread>(
                  std::bind(&application_impl::main_dispatch, shared_from_this()));
          dispatchers_[its_main_dispatcher->get_id()] = its_main_dispatcher;      // <thread::id, shared_ptr<thread>>
      }
// ...
    for (size_t i = 0; i < io_thread_count - 1; i++) {
        std::shared_ptr<std::thread> its_thread
            = std::make_shared<std::thread>([this, i, io_thread_nice_level] {
        // ...
                    io_.run();
        // ...
            });
    }
}
```
这里的`main_dispatch`执行的业务逻辑是：
```cpp
void application_impl::main_dispatch() {
    // ...
    std::unique_lock<std::mutex> its_lock(handlers_mutex_);
    while (is_dispatching_) {
        if (handlers_.empty() || !is_active_dispatcher(its_id)) {     // 先notify_all()，再wait()
            // Cancel other waiting dispatcher
            dispatcher_condition_.notify_all();   // 通知其他线程可以参与锁竞争了，因为有小概率在上一行代码判断完，立马来任务了？？？
            // Wait for new handlers to execute
            dispatcher_condition_.wait(its_lock, [this, &its_id] {
                    return !(is_dispatching_
                             && (handlers_.empty() || !is_active_dispatcher(its_id)));
                });
        }
        else {
            while (is_dispatching_  && is_active_dispatcher(its_id)
                   && (its_handler = get_next_handler())) {
                its_lock.unlock();
                invoke_handler(its_handler);      //  ！！！可能新建线程调用dispatch，专门执行dispatch
                
                if (!is_dispatching_)
                    return;

                its_lock.lock();

                reschedule_availability_handler(its_handler);
                remove_elapsed_dispatchers();
            }
        }
    }
    its_lock.unlock();
}
```

从`dispatch()`
```cpp
void application_impl::dispatch() {
    std::unique_lock<std::mutex> its_lock(handlers_mutex_);
    while (is_active_dispatcher(its_id)) {    // 当前线程id
        if (is_dispatching_ && handlers_.empty()) {
             dispatcher_condition_.wait(its_lock,
                                       [this, &its_id] { return !is_active_dispatcher(its_id); });
             // Maybe woken up from main dispatcher
             if (handlers_.empty() && !is_active_dispatcher(its_id)) {
                 if (!is_dispatching_) {
                     return;
                 }
                 std::lock_guard<std::mutex> its_lock(dispatcher_mutex_);
                 elapsed_dispatchers_.insert(its_id);
                 return;
             }
        } else {
            std::shared_ptr<sync_handler> its_handler;
            while (is_dispatching_ && is_active_dispatcher(its_id)
                   && (its_handler = get_next_handler())) {
                its_lock.unlock();
                invoke_handler(its_handler);

                if (!is_dispatching_)
                    return;

                its_lock.lock();

                reschedule_availability_handler(its_handler);
                remove_elapsed_dispatchers();
            }
        }
    }
    if (is_dispatching_) {
        std::lock_guard<std::mutex> its_lock(dispatcher_mutex_);
        elapsed_dispatchers_.insert(its_id);
    }
    dispatcher_condition_.notify_all();
}
```

如何理解这里的**主从dispatch**的设计？？？如果一个主dispatch线程也能处理过来的话，就不会创建新线程了
其中都调到了关键函数`invoke_handler`，主要业务如下：
创建`handler`，主要业务逻辑：
1. 创建定时任务，超过100ms后执行（认为该任务是个长任务，长任务太多会降低系统效率）：如果有dispatcher线程，就唤醒；无，则尝试创建新线程
2. 对锁进行忙等，为了将当前线程标记为`繁忙`
3. **执行用户回调**
4. 关闭定时器，将当前线程标记为`空闲`
```cpp
void application_impl::invoke_handler(std::shared_ptr<sync_handler> &_handler) {
    const std::thread::id its_id = std::this_thread::get_id();

    std::shared_ptr<sync_handler> its_sync_handler
        = std::make_shared<sync_handler>(_handler->service_id_,
            _handler->instance_id_, _handler->method_id_,
            _handler->session_id_, _handler->eventgroup_id_,
            _handler->handler_type_);

    boost::asio::steady_timer its_dispatcher_timer(io_);
    its_dispatcher_timer.expires_from_now(std::chrono::milliseconds(max_dispatch_time_));   // default：100
    its_dispatcher_timer.async_wait([this, its_sync_handler](const boost::system::error_code &_error) {
        if (!_error) {
            print_blocking_call(its_sync_handler);
            if (has_active_dispatcher()) {    // dispatcher都在被占用
                std::lock_guard<std::mutex> its_lock(handlers_mutex_);
                dispatcher_condition_.notify_all();
            } else {
                // If possible, create a new dispatcher thread to unblock.
                // If this is _not_ possible, dispatching is blocked until
                // at least one of the active handler calls returns.
                while (is_dispatching_) {
                    if (dispatcher_mutex_.try_lock()) {
                        if (dispatchers_.size() < max_dispatchers_) {
                            if (is_dispatching_) {
                                auto its_dispatcher = std::make_shared<std::thread>(
                                    std::bind(&application_impl::dispatch, shared_from_this()));
                                dispatchers_[its_dispatcher->get_id()] = its_dispatcher;
                            } else {
                                VSOMEIP_INFO << "Won't start new dispatcher "
                                        "thread as Client=" << std::hex
                                        << get_client() << " is shutting down";
                            }
                        } else {
                            VSOMEIP_ERROR << "Maximum number of dispatchers exceeded. Configuration: "
                                << " Max dispatchers: " << std::dec << max_dispatchers_
                                << " Max dispatch time: " << std::dec << max_dispatch_time_;
                        }
                        dispatcher_mutex_.unlock();
                        break;
                    } else {
                        std::this_thread::yield();    // 相比sleep，有没有可能跑满一个核？？？
                    }
                }
            }
        }
    });
    if (client_side_logging_
        && (client_side_logging_filter_.empty()
            || (1 == client_side_logging_filter_.count(std::make_tuple(its_sync_handler->service_id_, ANY_INSTANCE)))
            || (1 == client_side_logging_filter_.count(std::make_tuple(its_sync_handler->service_id_, its_sync_handler->instance_id_))))) {
        VSOMEIP_INFO << "Invoking handler: ("
            << std::hex << std::setw(4) << std::setfill('0') << client_ <<"): ["
            << std::hex << std::setw(4) << std::setfill('0') << its_sync_handler->service_id_ << "."
            << std::hex << std::setw(4) << std::setfill('0') << its_sync_handler->instance_id_ << "."
            << std::hex << std::setw(4) << std::setfill('0') << its_sync_handler->method_id_ << ":"
            << std::hex << std::setw(4) << std::setfill('0') << its_sync_handler->session_id_ << "] "
            << "type=" << static_cast<std::uint32_t>(its_sync_handler->handler_type_)
            << " thread=" << std::hex << its_id;
    }

    while (is_dispatching_ ) {
        if (dispatcher_mutex_.try_lock()) {
            running_dispatchers_.insert(its_id);
            dispatcher_mutex_.unlock();
            break;
        }
        std::this_thread::yield();
    }

    if (is_dispatching_) {
        try {
            _handler->handler_();
        } catch (const std::exception &e) {
            VSOMEIP_ERROR << "application_impl::invoke_handler caught exception: "
                    << e.what();
            print_blocking_call(its_sync_handler);
        }
    }
    boost::system::error_code ec;
    its_dispatcher_timer.cancel(ec);

    while (is_dispatching_ ) {
        if (dispatcher_mutex_.try_lock()) {
            running_dispatchers_.erase(its_id);
            dispatcher_mutex_.unlock();
            return;
        }
        std::this_thread::yield();
    }
}
```



核心类：`class vsomeip_v3::application_impl`，持有主要的资源
```cpp
    // 类声明
    boost::asio::io_context io_;                            // io_被传入各种boost对象的构造函数，它们会调post
    std::set<std::shared_ptr<std::thread> > io_threads_;
    std::shared_ptr<boost::asio::io_context::work> work_;   // work_由io_构造得来

    // start()
    io_.run();
```
`io_context`也是基于`Impl`范式，其实现类，是基于操作系统，对于linux而言，则是`class scheduler`

`class io_context`
```cpp
class io_context
  : public execution_context
{
public:
  /// Request the io_context to invoke the given function object.
  /**
   * This function is used to ask the io_context to execute the given function
   * object. The function object will never be executed inside @c post().
   * Instead, it will be scheduled to run on the io_context.
   *
   * @param f The function object to be called. The executor will make a copy
   * of the handler object as required. The function signature of the function
   * object must be: @code void function(); @endcode
   *
   * @param a An allocator that may be used by the executor to allocate the
   * internal storage needed for function invocation.
   */
  template <typename Function, typename OtherAllocator>
  void post(BOOST_ASIO_MOVE_ARG(Function) f,
      const OtherAllocator& a) const;

  BOOST_ASIO_DECL count_type run();
  BOOST_ASIO_DECL count_type poll();
  void reset();

private:
  impl_type& impl_;
};
```
能取得io_context的对象：
`work_`
`signals_`
`watchdog_timer_`

(linux环境) io_context的实现类：`scheduler`
（io_context可能被多线程执行，从one_thread_标志位可以看出，还会对单线程场景进行优化）
**单线程（可多线程），任务队列**
```cpp
class scheduler
  : public execution_context_service_base<scheduler>,
    public thread_context
{
  // Initialise the task, if required.
  BOOST_ASIO_DECL void init_task();

  // Run the event loop until interrupted or no more work.
  BOOST_ASIO_DECL std::size_t run(boost::system::error_code& ec);

  // Request invocation of the given operation and return immediately. Assumes
  // that work_started() has not yet been called for the operation.
  BOOST_ASIO_DECL void post_immediate_completion(
      operation* op, bool is_continuation);

  // Enqueue the given operation following a failed attempt to dispatch the
  // operation for immediate invocation.
  BOOST_ASIO_DECL void do_dispatch(operation* op);

ptrivate:
  // Run at most one operation. May block.
  BOOST_ASIO_DECL std::size_t do_run_one(mutex::scoped_lock& lock,
      thread_info& this_thread, const boost::system::error_code& ec);

  // Whether to optimise for single-threaded use cases.
  const bool one_thread_;

  // Mutex to protect access to internal data.
  mutable mutex mutex_;

  // Event to wake up blocked threads.
  event wakeup_event_;

  // The task to be run by this service.
  reactor* task_;

  // Whether the task has been interrupted.
  bool task_interrupted_;

  // The count of unfinished work.
  atomic_count outstanding_work_;

  // The queue of handlers that are ready to be delivered.
  op_queue<operation> op_queue_;

  // Flag to indicate that the dispatcher has been stopped.
  bool stopped_;

  // Flag to indicate that the dispatcher has been shut down.
  bool shutdown_;

  // The concurrency hint used to initialise the scheduler.
  const int concurrency_hint_;

  // The thread that is running the scheduler.
  boost::asio::detail::thread* thread_;
};
```

boost `scheduler::run(ec)`
！！！这里设计**私有队列**的目的是：
避免锁竞争
因为如果只使用公共队列的话，锁竞争会更多，通过私有队列增加一层间接性，私有队列push事件，是无锁无开销的
这里的设计理念是：频繁地小锁 不如 批量的大锁
（？？？私有队列里的任务具体是什么任务？是否都是异步I/O任务？比如得知socket可读，触发回调来读取）
```cpp
struct scheduler_thread_info : public thread_info_base
{
  op_queue<scheduler_operation> private_op_queue;     // 线程私有的队列，存放从reactor获取的I/O事件handler
  long private_outstanding_work;
};

std::size_t scheduler::run(boost::system::error_code& ec)
{
  ec = boost::system::error_code();
  if (outstanding_work_ == 0)
  {
    stop();
    return 0;
  }

  thread_info this_thread;
  this_thread.private_outstanding_work = 0;
  thread_call_stack::context ctx(this, this_thread);

  mutex::scoped_lock lock(mutex_);

  std::size_t n = 0;
  for (; do_run_one(lock, this_thread, ec); lock.lock())  // 锁保护的是scheduler的内部数据，如任务队列，注意do_run_one内部必然会释放锁
    if (n != (std::numeric_limits<std::size_t>::max)())
      ++n;
  return n;
}
```
`scheduler::do_run_one(lock, thread_info, ec)`
从任务队列取任务，注意，有些任务是特殊任务（主任务/周期性任务），带`task_operation_`标记，如果有剩余任务则需要把主任务中断`task_interrupted_`
执行任务的时候，如果发现还有很多任务，则唤醒其他线程（基于`pthread_cond_signal`）
！！！区别于单线程执行`while epoll_wait`，这里的`epoll_wait`是在必要时，才被单线程执行
```cpp
std::size_t scheduler::do_run_one(mutex::scoped_lock& lock,
    scheduler::thread_info& this_thread,
    const boost::system::error_code& ec)
{
  while (!stopped_)
  {
    if (!op_queue_.empty())
    {
      // Prepare to execute first handler from queue.
      operation* o = op_queue_.front();
      op_queue_.pop();
      bool more_handlers = (!op_queue_.empty());

      if (o == &task_operation_)      // 第一次调进来会执行，init_task()时会加入task_operation_，并且立即notify一个线程？？？task_operation_后续仍然会进入队列，什么时候？？？
      {
        task_interrupted_ = more_handlers;

        if (more_handlers && !one_thread_)  // 有任务 & 多线程
          wakeup_event_.unlock_and_signal_one(lock);
        else
          lock.unlock();                    // 无任务 / 单线程 则释放锁

        task_cleanup on_exit = { this, &lock, &this_thread };     // ！！！这里很隐晦地通过 RAII 把私有队列的任务塞进了公共队列，并塞了一个task_operation_进去
        (void)on_exit;

        // Run the task. May throw an exception. Only block if the operation
        // queue is empty and we're not polling, otherwise we want to return
        // as soon as possible.
       /*
          这里的task是reactor*, run实际是去执行一次epoll_wait，如果有任务，timeout为0
          run(timeout, queue&)，收集就绪事件
       */
       task_->run(more_handlers ? 0 : -1, this_thread.private_op_queue);
      }
      else
      {
        std::size_t task_result = o->task_result_;

        if (more_handlers && !one_thread_)
          wake_one_thread_and_unlock(lock);
        else
          lock.unlock();

        // Ensure the count of outstanding work is decremented on block exit.
        work_cleanup on_exit = { this, &lock, &this_thread };
        (void)on_exit;

        // Complete the operation. May throw an exception. Deletes the object.
        o->complete(this, ec, task_result);
        this_thread.rethrow_pending_exception();

        return 1;
      }
    }
    else
    {
      wakeup_event_.clear(lock);
      wakeup_event_.wait(lock);
    }
  }

  return 0;
}
```

### 对应关系
一个`io_context`对应一个`scheduler`
多线程（线程数目是vsomeip中可配置的io_thread_count）都调用到`scheduler::run()`，在任务少的情况下，大多数线程被卡住








# service consumer app
查找服务成功时，势必和创建socket资源有关

## 服务查找 具体如何实现？
someipd肯定会维护local service table






















