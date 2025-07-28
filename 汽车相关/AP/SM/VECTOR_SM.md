

# 职责1 `Function Group`的管理
```cpp
amsr::exec::r20_11::StateClient state_client_;
```

## `获取 MachineFG 从 初始状态 过渡到 Startup 的结果`
```cpp
void Context::WaitStartUp()
{
    ara::core::Future<void> future = state_client_.GetInitialMachineStateTransitionResult();
    ara::core::Result<void> result{future.GetResult()};
    if(!result.HasValue()) 
    {
       context_log_->LogError() << "Get MachineFG startup failed. Reason: " << result.Error();
       ara::core::Abort("Get MachineFG startup failed.");
    }
}
```

## `R20-11`
```cpp
void StatePool::Init(ara::log::Logger &logger)
{
    simple_start_mode_.Init(logger);
    off_mode_.Init(logger);
}
```

## 切换状态
当前工程都是简单场景：`Off` -> `Running`
### `OffMode`
```cpp
void OffMode::Run()
{
    // change DrivingFG to Off
    log_->LogInfo() << "SetState for 'DrivingFG' to 'Off'.";
    function_group_manager_context_.ChangeFunctionGroupState("DrivingFG", "Off");

    // change CommonFG to Off
    log_->LogInfo() << "SetState for 'CommonFG' to 'Off'.";
    function_group_manager_context_.ChangeFunctionGroupState("CommonFG", "Off");

    // change MachineFg to Off
    log_->LogInfo() << "SetState for 'MachineFG' to 'Off'.";
    function_group_manager_context_.MachineFGOff();
}
```
切换

### `SimpleStartMode` 
```cpp
void SimpleStartMode::Run() 
{
    // change MachineFg to Running
    log_->LogInfo() << "SetState for 'MachineFG' to 'Running'.";
    function_group_manager_context_.ChangeFunctionGroupState("MachineFG", "Running");

    // change DrivingFG to Running
    log_->LogInfo() << "SetState for 'DrivingFG' to 'Running'.";
    function_group_manager_context_.ChangeFunctionGroupState("DrivingFG", "Running");

    // change CommonFG to Running
    log_->LogInfo() << "SetState for 'CommonFG' to 'Running'.";
    function_group_manager_context_.ChangeFunctionGroupState("CommonFG", "Running");

    // change ParkingFG to Running
    log_->LogInfo() << "SetState for 'ParkingFG' to 'Running'.";
    function_group_manager_context_.ChangeFunctionGroupState("ParkingFG", "Running");

    // change DataInfoFg to Runnning
    log_->LogInfo() << "SetState for 'DataInfoFg' to 'Running'.";
    function_group_manager_context_.ChangeFunctionGroupState("DataInfoFG","Running");
}
```
把若干FG切到`Running`状态





# `SM`和`EM`的通信

























