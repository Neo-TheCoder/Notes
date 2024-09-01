# 文档介绍
# AUTOSAR
# 6 Functional specification
This chapter contains the specification of the internal functional behavior of the E2E supervision, this includes how the layout of the E2E-Header is defined, how the E2E-Header is created and how the E2E-Header is evaluated, and how the E2E-Statemachine is defined. For general introduction of the E2E supervision, see chapter 1.

## 6.1 Overview of communication protection
An important aspect of a communication protection mechanism is its `standardization` and `its flexibility for different purposes`.
This is resolved by having a set of E2E Profiles, that define a combination of protection mechanisms, a message format, and a set of configuration parameters.
Moreover, some E2E Profiles have standard `E2E variants`.
An E2E variant is simply a set of configuration options to be used with `a given E2E Profile`.
For example, in E2E Profile 1, the positions of CRC and counter are configurable. The E2E variant 1A requires that CRC starts at bit 0 and counter starts at bit 8.
E2E communication protection works as follows:
* Sender: addition of control fields like CRC or counter to the transmitted data;
* Receiver: evaluation of the control fields from the received data, calculation of control fields (e.g. CRC calculation on the received data), comparison of calculated control fields with an expected/received content.

Each E2E Profile has a specific set of control fields with a specific functional behavior and with specific properties for the detection of communication faults.

## 6.2 Overview of E2E Profiles
The E2E Profiles provide a consistent set of data protection mechanisms, designed to protecting against the faults considered in the fault model.
Each E2E Profile provides an alternative way to protect the communication, by means of different algorithms. However, E2E Profiles have similar interfaces and behavior.
### Each E2E Profile uses a subset of the following data protection mechanisms:
1. `A CRC`, provided by CRC Supervision;
2. `A Sequence Counter incremented at every transmission request`, the value is checked at receiver side for correct incrementation;
3. `An Alive Counter incremented at every transmission request`, the value checked at the receiver side if it changes at all, but correct incrementation is not checked;(只检查是否置位，不检查是否正确地增长)
4. `A specific ID for every port data element` sent over a port or a specific ID for every message-group (global to system, where the system may contain potentially several ECUs);
5. `A specific ID for every source` (e.g., client) of a data element or message group
6. `A message type` distinguishing between requests and responses in case of E2E communication protection for methods
7. `A message result` distinguishing between normal and error responses in case of E2E communication protection for methods
8. Timeout detection:
(a) Receiver communication timeout.
(b) Sender acknowledgement timeout.

Depending on the used communication and network stack, appropriate subsets of these mechanisms are defined as E2E communication profiles.
Some of the above mechanisms are implemented in RTE, COM, and/or communication stacks. However, to reduce or avoid an allocation of safety requirements to these modules, they are not considered: E2E Supervision provides all mechanisms internally (only with usage of CRC Supervision).
The E2E Profiles can be used for both inter and intra ECU communication. The E2E Profiles were specified for specific communication infrastructure, such as CAN, CAN FD, FlexRay, LIN, Ethernet. Depending on the system, the user selects which E2E Profile is to be used, from the E2E Profiles provided by E2E Supervision.


### 6.2.1 Error detection

### 6.2.3 General Functionality of an E2E-Profile
Each E2E-Profile provides the following 3 functionalities:
1. Protect
2. Forward
3. Check
The ’protect’ functionality, simply called the ’protect function’ creates the E2E-Header and therefore protects the data to be sent over a communication medium.
The ’forward’ functionality, simply called the ’forward function’, is similar to the protect function and creates the header for the data to be transmitted but allows the additional replication of a received E2E-State. The main use-case for this function is Signal-Service-Translation where e.g. a E2E-protected signal is received, and the E2E-Status shall be replicated on the outgoing side.
The ’check’ functionality, simply called the ’check function’, evaluates the E2E-Header of the received message and checks for occurred communication faults. These faults are mirrored in the returned E2E-States.
In addition to the single E2E-Profiles a E2E-Statemachine evaluates the returned E2E-States over a longer period.






# VECTOR
### 2.7.5 E2E Communication Protection




















# 先看下下proxy端接收数据时的链路
当`Runtime`、`Skeleton`、`Proxy`对象（find service时成功时构造）都成功构造时
Proxy端的reactor线程，通过`epoll`，监听若干和someipd建立的unix domain socket的可读事件.
当监听到可读事件时，逐步触发receive handler，是这样逐步触发的：
当监听到可读事件时，`EventNotificationTask`任务被构造：
```cpp
    /*!
     * \brief Initialize the functor to call on event notification.
     *
     * \param[in] event The event to notify.
     */
    explicit EventNotificationTask(ProxyEventBase& event) : Task{&event}, event_{event} {}
```
然很被塞入线程池处理。
在用户层进行`SetReceiveHandler`时，进行注册用户层的receive回调


## `GetNewSamples`的细节
```cpp
  /*!
   * \brief       Reads the serialized samples from underlying receive buffers and deserializes them.
   * \tparam      F User provided callable function with the signature void(ara::com::SamplePtr<SampleType const>).
   * \param[in]   f Callable to be invoked on every deserialized sample.
   * \param[in]   max_samples Maximum number of samples that can be processed within this call.
   * \return      Result containing the number of successfully processed events within this call or an error.
   * \error       ara::com::ComErrc::kMaxSamplesReached if all slots from visible sample cache are used.
   * \pre         The event is subscribed.
   * \context     App
   * \threadsafe  FALSE for same class instance, TRUE for different instances.
   *              This API can be called from event receive handler when subscription / unsubscription is requested in
   *              parallel from the application thread without the need of additional synchronization measures.
   * \reentrant   FALSE for same class instance, TRUE for different instances.
   * \vpublic
   * \synchronous TRUE
   * \trace       SPEC-8053568
   * \spec
   *   requires true;
   * \endspec
   *
   * \internal
   * - Create a result with the valid event processed set to zero.
   * - If the event is subscribed
   *   - Define a callable which
   *     - Create a SamplePtr from the deserialized data.
   *     - Invoke the user provided callable f.
   *   - Invoke ReadSamples on the proxy_event_backend_ and emplace its return value into result.
   * - Otherwise
   *    - Log a warning that API is called prior to subscription.
   * - Return the result.
   *
   * Calls "ara::core::Abort()" if:
   * - The event has been not subscribed to.
   * \endinternal
   */
  template <typename F>
  auto GetNewSamples(F&& f, std::size_t max_samples = std::numeric_limits<size_t>::max()) noexcept
      -> GetNewSamplesResult {
    // PTP-B-Socal-ProxyEvent_GetNewSamples
    GetNewSamplesResult result{GetNewSamplesResult::FromValue(0UL)};
    if (is_subscribed_) {
      auto callable_sample_result = [this, &f](SampleData&& deserialized_data) {
        // VCA_SOCAL_CALLING_NON_STATIC_FUNCTION_FROM_CALLBACK_SYNCHRONOUSLY
        f(GetSamplePtr(std::move(deserialized_data)));
      };
      // VCA_SOCAL_FUNCTION_CALL_ON_VALID_OBJECTS_ADHERING_TO_FUNCTION_CONTRACT
      result = proxy_event_backend_.ReadSamples(max_samples, callable_sample_result);
    } else {
      logger_.LogFatal(
          [](::ara::log::LogStream& s) { s << "API called before subscription or after unsubscription of the event."; },
          static_cast<char const*>(__func__), __LINE__);
      ::ara::core::Abort(
          "ProxyEventBase::GetNewSamples: API called before subscription or after unsubscription of the event.");
    }
    // PTP-E-Socal-ProxyEvent_GetNewSamples
    return result;
  }
```

### `SampleReader`的`ReadSamples`的实现：
```cpp
/*!
   * \brief Reads serialized samples from the given sample cache container, deserializes them and calls the provided
   *        callback function.
   *
   * \context     App
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \synchronous TRUE
   *
   * \pre   -
   * \param[in] visible_sample_cache         shared pointer to the visible sample cache from which free sample slots
   *                                         are retrieved for each read sample
   * \param[in] serialized_samples_container container which has the enqueued event samples cache entries
   * \param[in] max_samples                  maximum number of samples to process
   * \param[in] callable_sample_result       Callable to be invoked on successful deserialization
   *
   * \return The number of event samples which were successfully deserialized and processed
   *
   * \internal
   * - Calculate how many samples shall be processed by using the min of max_samples or the available serialzied samples
   * - Do repeatedly for the serialized_samples in sample_cache_container
   *   - Retrieve one slot from the visible cache
   *   - When a slot is availble
   *     - Deserialize the sample.
   *     - If deserialization is successful
   *       - Increase the number of successfully processed events
   *       - Invoke callable_sample_result with wrapped, deserialized sample, e2e check status
"NotAvailable" and time stamp.
   *     - Otherwise
   *       - Return the visible cache slot.
   *   - Otherwise
   *     - Stop further processing of samples
   * \endinternal
   *
   */
  std::size_t ReadSamples(std::shared_ptr<VisibleSampleContainer> visible_sample_cache,
                          SampleCacheContainer& serialized_samples_container, std::size_t const max_samples,
                          CallableReadSamplesResult const& callable_sample_result) const noexcept override {
    std::size_t nr_valid_events_processed{0};
    std::size_t const samples_to_process{std::min(max_samples, serialized_samples_container.size())};
    // VECTOR NL AutosarC++17_10-A6.5.1: MD_SOMEIPBINDING_AutosarC++17_10-A6.5.1_loop_counter
    for (std::size_t process_index{0U}; process_index < samples_to_process; ++process_index) {
      // Get free slot for deserialization
      // VECTOR NL AutosarC++17_10-A18.5.8: MD_SOMEIPBINDING_AutosarC++17_10_A18.5.8_false_positive
      std::shared_ptr<socal::internal::events::MemoryWrapperInterface<SampleType>> visible_cache_slot{
          // VCA_SOMEIPBINDING_ACCESSING_MEMBERS_OF_REFERENCE_CLASS_ATTRIBUTES
          visible_sample_cache->GetNextFreeSample()};

      if (visible_cache_slot != nullptr) {
        // Retrieve serialized event
        std::unique_ptr<SomeIpSampleCacheEntry> const& serialized_event{
            // VCA_SOMEIPBINDING_POSSIBLY_CALLING_NULLPTR_METHOD_CALL_ON_REF
            serialized_samples_container.front()};

        // VCA_SOMEIPBINDING_ACCESSING_MEMBERS_OF_REFERENCE_CLASS_ATTRIBUTES
        ::ara::core::Optional<TimeStamp> const time_stamp{serialized_event->GetTimeStamp()};

        // VCA_SOMEIPBINDING_ACCESSING_MEMBERS_OF_REFERENCE_CLASS_ATTRIBUTES
        std::size_t const buffer_size{serialized_event->GetBufferSize()};
        ::vac::memory::MemoryBuffer<osabstraction::io::MutableIOBuffer>::MemoryBufferView const buffer_view{
            // VCA_SOMEIPBINDING_ACCESSING_MEMBERS_OF_REFERENCE_CLASS_ATTRIBUTES
            serialized_event->GetBufferView()};
        // VCA_SOMEIPBINDING_POSSIBLY_CALLING_NULLPTR_METHOD_CALL_ON_REF
        bool const deserialized_successfully{DeserializeSample(**visible_cache_slot, buffer_size, buffer_view)};

        // VCA_SOMEIPBINDING_POSSIBLY_CALLING_NULLPTR_METHOD_CALL_ON_REF
        serialized_samples_container.pop_front();

        if (deserialized_successfully) {
          ++nr_valid_events_processed;
          // VCA_SOMEIPBINDING_TRIVIAL_FUNCTION_CONTRACT
          callable_sample_result(SampleData{std::move(visible_cache_slot), visible_sample_cache,
                                            ara::com::E2E_state_machine::E2ECheckStatus::NotAvailable, time_stamp});
        } else {
          // VCA_SOMEIPBINDING_POSSIBLY_CALLING_NULLPTR_METHOD_CALL_ON_REF
          visible_sample_cache->ReturnEntry(std::move(visible_cache_slot));

          logger_.LogError(
              [](::ara::log::LogStream& s) {
                s << "Deserialization error occurred. Please check that the event datatype for proxy and skeleton "
                     "side are compatible.";
              },
              static_cast<char const*>(__func__), __LINE__);
        }
      } else {  // VCA_SOMEIPBINDING_TRIVIAL_FUNCTION_CONTRACT
        // This is not an error case, we only process until no more free slot is available.
        logger_.LogDebug([](::ara::log::LogStream& s) { s << "No free slot is available anymore."; },
                         static_cast<char const*>(__func__), __LINE__);
        break;
      }
    }  // VCA_SOMEIPBINDING_TRIVIAL_FUNCTION_CONTRACT

    return nr_valid_events_processed;
  }
```





`GetSamplePtr`，传入的`deserialized_data`带有`e2e_check_status`
```cpp
  /*!
   * \brief       Construct the SamplePtr from the deserialization result when the parameter of time stamp is disabled.
   * \tparam      Config  Configures if the time stamp is enabled or disabled.
   * \param[in]   deserialized_data The deserialized event data.
   * \return      SamplePtr containing the sample data and e2e check status.
   * \pre         -
   * \context     App
   * \threadsafe  FALSE
   * \reentrant   TRUE
   * \synchronous TRUE
   */
  // VECTOR NC AutosarC++17_10-M9.3.3: MD_SOCAL_AutosarC++17_10-M9.3.3_Method_can_be_declared_static
  template <typename Config = TimestampConfiguration>
  auto GetSamplePtr(SampleData&& deserialized_data) const noexcept
      -> std::enable_if_t<Config::IsEnabled != true, SamplePtr> {
    // VCA_SOCAL_CALLING_STL_APIS
    return SamplePtr{std::move(deserialized_data.memory_wrapper_if_ptr), std::move(deserialized_data.cache_ptr),
                     deserialized_data.e2e_check_status};
  }
```


`SamplePtr`是什么？
```cpp
  using SamplePtr = ::amsr::socal::r20_11::events::SamplePtr<SampleType const, TimestampConfiguration>;
```

`SamplePtr`的构造函数，！！！表明了`ProfileCheckStatus::Error`时，构造直接失败！调用`::ara::core::Abort`
```cpp
  /*!
   * \brief       Generic constructor for storing the deserialized sample and E2E check status.
   * \tparam      Config  Configures if time stamp is enabled or disabled.
   * \param[in]   memory_wrapper_if_ptr Pointer to the memory wrapper interface which can be used to access the
   *              sample data from the corresponding binding.
   *              Must be nullptr, if and only if, e2e_check_status equals to 'Error'.
   * \param[in]   cache_ptr Pointer to the cache interface where the memory_ptr was taken from.
   * \param[in]   e2e_profile_check_status E2E profile check status for the sample.
   * \pre         -
   * \context     App
   * \threadsafe  TRUE
   * \reentrant   TRUE
   * \vprivate
   * \synchronous TRUE
   * \internal
   * - If given memory_ptr is invalid,
   *   - Reset the cache_ptr_.
   *
   * - Calls "ara::core::Abort()" if:
   *   - memory_ptr is nullptr and e2e_check_status is not equal to ProfileCheckStatus::Error.
   *   - memory_ptr is not equal to nullptr and e2e_check_status is equal to ProfileCheckStatus::Error.
   * \endinternal
   */
  template <typename Config = TimestampConfiguration>
  // VECTOR NC AutosarC++17_10-A12.1.5: MD_SOCAL_AutosarC++17_10-A12.1.5_delegating_constructor_for_common_init
  explicit SamplePtr(
      MemoryWrapperInterfacePtrType&& memory_wrapper_if_ptr, std::weak_ptr<CacheType> cache_ptr,
      std::enable_if_t<Config::IsEnabled == false, ProfileCheckStatus> const e2e_profile_check_status) noexcept
      : memory_ptr_{std::move(memory_wrapper_if_ptr)},
        // VCA_SOCAL_VALID_MOVE_CONSTRUCTION
        cache_ptr_{std::move(cache_ptr)},
        e2e_profile_check_status_{e2e_profile_check_status} {
    if (memory_ptr_ == nullptr) {
      cache_ptr_.reset();  // VCA_SOCAL_CALLING_STL_APIS

      if (e2e_profile_check_status_ != ProfileCheckStatus::Error) {
        ::amsr::socal::internal::logging::AraComLogger const logger{
            ::amsr::socal::internal::logging::kAraComLoggerContextId,
            ::amsr::socal::internal::logging::kAraComLoggerContextDescription, "SamplePtr20-11"_sv};

        logger.LogFatal(
            [](::ara::log::LogStream& s) {
              s << "Creating SamplePtr with nullptr is only allowed, if E2E profile check status is 'Error'.";
            },
            static_cast<char const*>(__func__), __LINE__);
        ::ara::core::Abort(
            "SamplePtr20-11::SamplePtr: Creating SamplePtr with nullptr is only allowed, if E2E profile check status "
            "is 'Error'.");
      }
    } else if (e2e_profile_check_status_ == ProfileCheckStatus::Error) {  // Memory Ptr must be nullptr if e2e profile
                                                                          // check status is Error.
      ::amsr::socal::internal::logging::AraComLogger const logger{
          ::amsr::socal::internal::logging::kAraComLoggerContextId,
          ::amsr::socal::internal::logging::kAraComLoggerContextDescription, "SamplePtr20-11"_sv};

      logger.LogFatal(
          [](::ara::log::LogStream& s) {
            s << "Invalid construction of SamplePtr with E2E profile check status 'Error'.";
          },
          static_cast<char const*>(__func__), __LINE__);
      ::ara::core::Abort(
          "SamplePtr20-11::SamplePtr: Invalid construction of SamplePtr with E2E profile check status 'Error'.");
    } else {
      // Do nothing.
    }
  }
```

#### PS: `E2ESampleReader`的`DeserializeE2ESample`
```cpp
  /*!
   * \brief       Performs deserialization of event sample payload with an E2E protected region.
   * \param[out]  sample_placeholder Sample placeholder
   * \param[in]   payload_size       Size of the payload
   * \param[in]   packet_view        View to the payload of the received sample
   * \param[in, out] e2e_result      E2E result to write to.
   * \return      A bool indicating the success of the deserialization
   * \context     App
   * \threadsafe  FALSE
   * \reentrant   FALSE
   * \synchronous TRUE
   * \trace SPEC-13650588
   */
  // VECTOR NC AutosarC++17_10-A8.4.4: MD_SOMEIPBINDING_A8.4.4_useReturnValueInsteadOfOutputParameter
  bool DeserializeE2ESample(
      SampleType& sample_placeholder, std::size_t const payload_size,
      ::vac::memory::MemoryBuffer<osabstraction::io::MutableIOBuffer>::MemoryBufferView const& packet_view,
      ::amsr::e2e::Result& e2e_result) const noexcept {
    bool deserialized_successfully{false};
    // VECTOR Next Line AutosarC++17_10-M5.2.8:MD_SOMEIPBINDING_AutosarC++17_10-M5.2.8_conv_from_voidp
    ::ara::core::Span<::std::uint8_t> const message{static_cast<::std::uint8_t*>(packet_view[0U].base_pointer),
                                                    payload_size};
    ::ara::core::Optional<::ara::core::Span<::std::uint8_t>> const e2e_region{
        // VCA_SOMEIPBINDING_POSSIBLY_CALLING_NULLPTR_METHOD_CALL_ON_REF
        deserializer_.GetE2EProtectedArea(message.subspan(no_check_header_size_))};
    if (e2e_region.has_value()) {
      // In case of E2E, first perform E2E check
      Reader e2e_reader{e2e_region.value()};
      // VCA_SOMEIPBINDING_POSSIBLY_ACCESSING_NULLPTR_RETRIEVED_FROM_EXTERNAL_FUNCTION
      bool const ret_val{e2e_reader.VerifySize(e2e_transformer_.GetHeaderSize())};
      if (ret_val) {
        if (!is_e2e_check_disabled_) {
          e2e_result =
              // VCA_SOMEIPBINDING_POSSIBLY_ACCESSING_NULLPTR_RETRIEVED_FROM_EXTERNAL_FUNCTION
              e2e_transformer_.Check(ara::core::Span<std::uint8_t const>{e2e_reader.Data(), e2e_reader.Size()});
        }
      }

      // Deserialize if E2E check passes
      if (e2e_result.GetCheckStatus() != ::amsr::e2e::state_machine::CheckStatus::Error) {
        // VCA_SOMEIPBINDING_PASSING_REFERENCE
        deserialized_successfully = DeserializeSample(sample_placeholder, payload_size, packet_view);
      }
    } else {
      // Skipped E2E check (e.g. when updateBit is set to false)
      // VCA_SOMEIPBINDING_PASSING_REFERENCE
      deserialized_successfully = DeserializeSample(sample_placeholder, payload_size, packet_view);
    }
    return deserialized_successfully;
  }
```






PS：其中`SampleData`的定义：
固定地存在一个成员变量，表示`ProfileCheckStatus`
```cpp
  /*!
   * \brief Sample data containing the memory pointer, e2e check status and time stamp.
   */
  struct SampleData {  // VCA_SOCAL_CALLING_STL_APIS
    /*!
     * \brief Memory wrapper pointer to access the deserialized samples.
     */
    MemoryWrapperInterfacePtr memory_wrapper_if_ptr{};

    /*!
     * \brief Cache pointer memory wrapper was taken from.
     */
    std::weak_ptr<::amsr::socal::internal::events::CacheInterface<SampleType>> cache_ptr;

    /*!
     * \brief E2ECheckStatus for the sample.
     */
    ::ara::com::e2e::ProfileCheckStatus e2e_check_status{::ara::com::e2e::ProfileCheckStatus::Error};

    /*!
     * \brief The time stamp for when receiving a message.
     */
    ::amsr::core::Optional<TimeStamp> time_stamp{};
  };
```




# 配置e2e的情况下，类有什么不同？
































