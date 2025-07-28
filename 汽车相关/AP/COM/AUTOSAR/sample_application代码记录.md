# 关于fusion中method的调用方式
```cpp
void FusionActivity::asyncMethodCall()
{
    // Calling "Adjust" method asynchronously (GetResult() should not be used unless results are ready).
    // Set random target position from -500 to 500.
    Position target_position;   // 填充Position变量
    target_position.x = m_ud_min_500_500(m_rand_eng);
    target_position.y = m_ud_min_500_500(m_rand_eng);
    target_position.z = m_ud_min_500_500(m_rand_eng);

    m_async_count++;

    auto adjust_future = m_proxy->Adjust(target_position);  // 调用Skeleton端Adjust方法，返回fututre对象，是异步的

    // Every 14 calls set a callback to the async call, no need to poll the results.
    bool poll_mode = (m_async_count % 14 == 0) ? false : true;

    if (poll_mode) {    // 每14次执行1次
        m_logger.LogInfo() << METHODS_HEADER << "Adjust() have been called, poll for completion";
        // Polling the results.
        while (!adjust_future.is_ready()) { // 每500ms轮询一下
            // Results are not ready yet.
            std::this_thread::sleep_for(std::chrono::milliseconds(500));
            m_logger.LogInfo() << METHODS_HEADER << "Radar is adjusting position, please wait ...";
        }

        // GetResult will return immediately.
        auto r = adjust_future.GetResult();
        if (r.HasValue()) {
            auto adjust_output = r.Value();
            if (adjust_output.success) {
                m_logger.LogInfo() << METHODS_HEADER << "Adjusting position was successful, effective position is ("
                                   << adjust_output.effective_position;

            } else {
                m_logger.LogWarn() << METHODS_HEADER
                                   << "Adjusting position was not successful, deviation from the target position is ("
                                   << adjust_output.effective_position.x - target_position.x << ","
                                   << adjust_output.effective_position.y - target_position.y << ","
                                   << adjust_output.effective_position.z - target_position.z << ")";
            }
        } else {
            auto err = r.Error();
            m_logger.LogWarn() << ERROR_HEADER << "Code: " << err.Value();
            m_logger.LogWarn() << ERROR_HEADER << "Message: " << err.Message();
        }
    } else {    // 每14次调用中，大多数的情况
        // Registering a notification callback.
        // TODO: Current ara::com::future.then() doesn't support arguments to be passed to the Lambda
        // TODO: then output can't be retrieved. According to the explanatory, the use example is
        // TODO: then([] (Future<Calibrate::Output> f) { f.GetResult()}.
        m_logger.LogInfo() << METHODS_HEADER << "Radar is adjusting position, you will be notified when done ...";

        adjust_future.then([&]() {  // 因为前面判断了是否ready，所以立即执行回调
        /*
            then的实现：
                先用一个写锁保护以下操作：
                先set callback
                然后判断future对象是否ready，执行回调
        */
            // it's safe to access the logger instance even if it's parent goes out of scope,
            // as long as the application process is alive, the reference is valid, because of
            // strong ownership inside the logging framework. 
            m_logger.LogInfo() << METHODS_HEADER << "Notification! Adjusting position has finished";
        });
    }
}
```

```cpp
void FusionActivity::syncMethodCall()
{
    m_sync_count++;

    FusionVariant variant;

    if ((m_sync_count % 3) == 0) {  // 设置variant不同的值
        variant = FusionVariant::FV_CHINA;
    } else if ((m_sync_count % 5) == 0) {
        variant = FusionVariant::FV_USA;
    } else if ((m_sync_count % 7) == 0) {
        variant = FusionVariant::FV_RUSSIA;
    } else {
        variant = FusionVariant::FV_EUROPE;
    }

    // Prepare the calibration configuration string.
    CoreString configuration = "calibration_config_" + std::to_string(m_sync_count);

    // Calling "Calibrate" method synchronously (GetResult() should be used).
    auto calibrate_future = m_proxy->Calibrate(configuration, variant);

    m_logger.LogInfo() << METHODS_HEADER << "Radar is calibrating, please wait ...";

    auto r = calibrate_future.GetResult();  // 直接同步调用GetResult()
    if (r.HasValue()) {
        auto calibrate_output = r.Value();
        (void)calibrate_output;
        m_logger.LogInfo() << METHODS_HEADER << "Calibration was successful";
    } else {
        auto err = r.Error();
        m_logger.LogWarn() << ERROR_HEADER << "Code: " << err.Value();
        m_logger.LogWarn() << ERROR_HEADER << "Message: " << err.Message();
        // reset the count and continue
        m_sync_count = 0;
    }
}

```

```cpp
void FusionActivity::fireAndForgetMethodCall()
{
    CoreString text = "Echo has been called.";

    m_proxy->Echo(text);    // 直接调用，默认没有返回值，不需要skeleton端应答
    m_logger.LogInfo() << METHODS_HEADER << text;
}
```
















