
# 两个状态机
```cpp
enum class StateHandle
{
    kOff = 0,
    kShutDown = 1,
    kStartup = 2,
    kRunning = 3,
    kRestart = 4
};

enum class RunningState
{
    kIdle = 0,
    kWorking = 1,
    kDrivingMode = 2,
    kParkingMode = 3,
    kEolMode = 4
};
```


# StateManager本体
```cpp
class StateManager
{
public:
    void Init(); // need init log and ara::core and Init function group Manager
    /*
         初始化function_group_manager_：
            其中会初始化context_：
                其中初始化working_mode_为kDrivingMode
                初始化现有的fg以及其状态
                初始化state_pool_：
                    初始化StartupMode、ShutdownMode、RunningMode对象
    */

    void Run(); // need report kRunning, and FunctionGroup::Run()
    void Deinitialize();
    StateManager();
    friend void SignalHandler(int signal);
    
private:
    void InitializeSignalHandling() noexcept;
    ara::core::Optional<ara::exec::ApplicationClient> app_client_;
    FunctionGroupManager function_group_manager_;
    ara::log::Logger& logger_;
    static bool running_; 
    std::atomic<RunningState> running_mode_;
    std::vector<TriggerType> trigger_queue_;
    std::mutex trigger_queue_mutex_;
    StateManagerCom state_manager_com_;
};
```





# 重要类：FunctionGroupManager
```cpp
class FunctionGroupManager
{
private:
    ara::log::Logger* function_group_manager_logger_ = nullptr;
    Context context_;
public:
    FunctionGroupManager(std::atomic<RunningState>& working_mode);
    ~FunctionGroupManager();
    void Init(ara::log::Logger &logger_);
    void Off();
    void WaitStartUp(); 
    // void Run(); // read message and changeState to chang Function Group State
    void Trigger(TriggerType trigger);
};


// FunctionGroupManager也有状态：FunctionGroupManagerState



```



# FunctionGroup具体细节
```cpp
using FunctionGroupStates = std::vector<std::string>;

struct FunctionGroup
{
    std::string function_group_name;    // name
    std::string current_state;  // 当前状态
    FunctionGroupStates function_group_states;  // 该功能具有的所有状态
    FunctionGroup(std::string function_group_name_, std::string current_state_, FunctionGroupStates& function_group_states_)
    {
        function_group_name = function_group_name_;
        current_state = current_state_;
        function_group_states.insert(function_group_states.begin(), function_group_states_.begin(), function_group_states_.end());
    }
};
```

`FunctionGroupManager`的成员：
```cpp
using FunctionGroupList = std::vector<FunctionGroup>;

class Context{
public:
    void Init(ara::log::Logger &logger);    // 初始化logger、设置working_mode_为kDrivingMode、InitFunctionGroup
    
    FunctionGroupManagerState& GetActiveState() noexcept;
    void ChangeState(const StateHandle next_state) noexcept;
    void ChangeFunctionGroupState(::std::string function_group_name, ::std::string state);
    ::std::string GetFunctionGroupState(const std::string& function_group_name);
    Context(std::atomic<RunningState>& working_mode);
    void WaitStartUp();
    void MachineFGOff();
    RunningState GetWorkingMode();
    amsr::exec::r20_11::StateClient state_client_;
private:
    void InitFunctionGroup();   // 初始化function_group_list_，写死了项目需求中涉及到的所有fg以及含有的状态
    void ChangeFunctionGroupList(::std::string function_group_name, ::std::string state);
    FunctionGroupManagerState* active_state_;
    std::atomic<RunningState>& working_mode_;   // ！！！RunningState 关键：状态机
    StatePool state_pool_;
    ara::log::Logger* context_log_;
    FunctionGroupList function_group_list_;
};
```


```cpp
class StatePool
{
public:
    FunctionGroupManagerState& GetState(StateHandle state);
    StatePool(Context &context);
    void Init(ara::log::Logger &logger);
    StartupMode startup_mode;
    ShutdownMode shutdown_mode;
    RunningMode running_mode;
};
```

```cpp
class RunningMode final: public FunctionGroupManagerState
{
private:
    Context &function_group_manager_context_;   // ！！！持有FunctionGroupManager对象的成员：Context对象的引用
    RunningState current_state_;
    bool is_working;
public:
    RunningMode(Context &context);
    void Run(); //need kown how start or off functiongroup
    // 调用ChangeFunctionGroupState切换MachineFg的状态为Running

    ~RunningMode();
};
```

简单看下`ChangeFunctionGroupState`的逻辑
```cpp
void Context::ChangeFunctionGroupState(std::string function_group_name, std::string state) // R17_11
{
    ara::core::Result<amsr::exec::FunctionGroup::CtorToken> fg_result{
    amsr::exec::FunctionGroup::Preconstruct(function_group_name)};
    // Preconstruct可能是先预分配一些东西，具体需要看实现

    amsr::exec::FunctionGroup function_group{std::move(fg_result.Value())}; // 使用std::move的话，只调用移动构造，不使用std::move的话，需要调用拷贝构造：先创建一个新对象，再把原始对象的值复制到新对象中，显然存在内存分配和数据复制的开销，移动构造直接转移所有权，避免了不必要的拷贝
    // 真正构造FunctionGroup对象

    ara::core::Result<amsr::exec::FunctionGroupState::CtorToken> fg_state_result{
    amsr::exec::FunctionGroupState::Preconstruct(function_group, state)};

    amsr::exec::FunctionGroupState function_group_state{std::move(fg_state_result.Value())};    // 构造FunctionGroupState对象

    ara::core::Future<void> set_future = state_client_.SetState(function_group_state);
    /*
        Context对象持有一个amsr::exec::r20_11::StateClient类型的state_client_
        关键调用是调用vector的接口：SetState，是异步调用，返回future对象
    */

    ara::core::Result<void> future_result{set_future.GetResult()};
    while (!future_result.HasValue()) { // 阻塞等待，future返回值
        context_log_->LogInfo() << "Function group transition to " << function_group_state.GetStateName()
                     << " failed. Reason: " << future_result.Error();
      // Try again
        set_future = state_client_.SetState(function_group_state);  // 如果失败，再调用一次
        future_result = set_future.GetResult();
    }
    context_log_->LogInfo([&](ara::log::LogStream &s) {
           s << "set "<< function_group_name << " to " << state << " success" ;
        });
    ChangeFunctionGroupList(function_group_name, state);
    // 该函数的操作是：将某一功能组的当前状态同步到Context对象所持有的function_group_list_

}
```



