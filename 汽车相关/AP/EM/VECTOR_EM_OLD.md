# 核心思想
## （至少）两套状态机
EM自身必须维护一个功能组切换的状态机，因为进程启动和关闭的耗时很长，具体状态如下：
```cpp
/*!
 * \brief                 Enumeration for each Function Group Interface state.
 *
 * \trace DSGN-ExecutionManager-StateManagement-FunctionGroupInterface
 */
enum class StateHandle : std::uint_least8_t {
  /*!
   * \brief No state transition in progress.
   */
  kIdle = 0x0,

  /*!
   * \brief Ongoing Function Group State Transition.
   */
  kTransition = 0x1,

  /*!
   * \brief State Transition succeeded but response has not been sent yet.
   */
  kPendingUpdate = 0x2,

  /*!
   * \brief An error has been detected and not reported yet.
   */
  kPendingError = 0x3,

  /*!
   * \brief Notification has been notified. No further errors will be reported.
   */
  kError = 0x4,
};
```
而关于功能组状态的`状态机`，
有专门的Processor负责真正的拉起和关闭行为，集中在`ProcessStartupConfig`
Context负责直接处理StateClient来的请求，负责进行`StartStateTransition`，这里会根据当前EM拉起/关闭进程的阶段，进行相应处理，如果当前在`kTransition`，就执行功能组的状态（本质上是string）的更改（当然会根据配置判断是否合法），然后触发Processor进行`PerformFunctionGroupStateChange`
其实很有可能，SM请求功能组A状态1->2切换，EM进行处理，SM再迅速发同样的消息，是无需处理的

状态机模式的核心思想是，状态可能很多，并且不同的状态迁移的操作各有不同，并且迁移路线也是定死了的
EM所维护的状态机是针对切换状态操作的，因为这个操作时间很长，很复杂，但是功能组状态再怎么切换，还是启动进程/关闭进程那一套


？？？从类间关系可以看出，好几个不同的功能组状态切换请求，不会互相干扰，
因为：
```cpp
// 有 多个 FunctionGroupInterface
StaticMap<StringView, FunctionGroupInterface> func_group_interfaces_;

bool StateManager::ForwardIncomingRequests() noexcept {
  bool request_handled{false};
  for (ipc_server::ConnectionHandle &handle : this->connected_clients_) {
    // ...
    this->ExecuteStateClientRequest(request);
  }
}
```

另外，对于每一个进程还有一个状态机，相关的类都是类似的（如StatePool / Context），**这两个状态机之间是有关联的，它们通过Processor、FunctionGroup和ProcessStartupConfig关联起来！！！**
具体状态如下：
```cpp
/*!
 * \brief      Internal states of a process.
 */
enum class ProcessStateHandle : std::uint8_t {
  kIdle = 0x0,           /*!< Process has not been started.*/
  kStarting = 0x1,       /*!< Process was requested to start but has not yet reported as started. */
  kRunning = 0x2,        /*!< Process is running. */
  kRestarting = 0x3,     /*!< Process was requested to restart and has not yet terminated. */
  kTerminating = 0x4,    /*!< Process was requested to terminate or reported a termination intent itself. */
  kTerminated = 0x5,     /*!< Process has been terminated either by self-termination or by forced termination. */
  kNumberOfStates = 0x6, /*!< Number of Process states. */
};
```

## 数据流向
sm最外层调用
```cpp
/*!
 * \internal
 * - Check precondition.
 * - Run function group interface state machines. This may complete state transisions.
 * - Store Pending updates.
 * - Accept new state clients.
 * - Execute state client requests.
 * - Return whether another call is required or not.
 * \endinternal
 */
bool StateManager::Run() noexcept {
  this->AssertInitialized();
  bool call_again{this->RunFunctionGroupStateMachines()};
  call_again = this->SendOutgoingMessages() || call_again;
  call_again = this->CollectPendingResponses() || call_again;
  this->AcceptNewConnections();
  call_again = this->ForwardIncomingRequests() || call_again;
  return call_again;
}
```

考虑SM给EM发消息以触发（set）功能组状态切换的场景：
源头在EM内部接收SM消息，
`ForwardIncomingRequests()`
假设一开始是Off状态，并且状态机1肯定也是初始状态`kIdle`
`StateIdle`：
然后触发context_获取当前状态，调当前状态的`ExecuteRequest()`：
  触发Context的`ExecuteStateClientRequest(StateClientRequest)`：
      `Context`的`RequestStateTransition`：
        `StartStateTransition(request_information, target_state)`
              1. `UpdateTargetState`：切换function_group_的状态（字符串改变 + 触发ProcessStartupConfig的`OnStateChange`）
              2. Processor去执行`PerformFunctionGroupStateChange()`，**把processor的内部状态由`kIdle` --> `kStartTransition`**
                  OnEnterState():
              涉及到到真正的进程启动关闭的函数调用是这样的：！！！
```cpp
      StateChangeProcessor::Run() {
        // 针对processor的内部状态 进行 swtich-case，也就是说，context_维护的状态 决定了 这个内部状态，然后这个内部状态决定了 process的状态
      }

      // 当StateChangeProcessor::NotifyRecoveryActionUpdateSendCompleted()被触发时，进入下面调用


      bool StateChangeProcessor::ProcessStartAction(ProcessStartupConfig& process) {
        // ...
        process.OnStartRequest();
      }
```

              3. 调request_state_的`StateTransitionStarted`，设置其中成员变量为busy
  然后，进入下一状态，触发context_维护的当前状态指针变为`kTransition`

如果是`StateTransition`，二次进入Run()：
```cpp
  if (this->func_group_->GetStateChangeProcessor().IsStateTransitionFinishedAndNotReported()) {
    this->LogTransitionResult(this->func_group_->GetStateChangeProcessor().GetFinishedStateTransitionResult());
  }

  // -->
  this->context_.GetActiveState().OnCompletedStateTransistion(result);
```
  <!-- 触发Context的`ExecuteStateClientRequest(StateClientRequest)`：
      `Context`的`RequestStateTransition`：
          通过`request_state_`可得知当前已经有一模一样的request，不会进入`StartStateTransition`，因为“A transition to the same state is already in progress” -->


？？？重要
```cpp
/*!
 * \internal
 * - Check precondition.
 * - Run state change processor.
 * - If transition completed in last run:
 *  - Log Transition result.
 * - Request rerun if the state change processor needs a rerun.
 * \endinternal
 */
bool FunctionGroupInterface::Run() noexcept {
  this->AssertInitialized();

  bool const rerun_required{this->func_group_->GetStateChangeProcessor().Run()};  // VCA_EM_CALL_BY_PTR_PASS_VOID
  // VCA_EM_CALL_BY_PTR_PASS_VOID
  if (this->func_group_->GetStateChangeProcessor().IsStateTransitionFinishedAndNotReported()) {
    // VCA_EM_CALL_BY_PTR_PASS_VOID
    this->LogTransitionResult(this->func_group_->GetStateChangeProcessor().GetFinishedStateTransitionResult());
  }

  return rerun_required;
}
```


如果新的功能组状态请求来了话，也可嫩会被响应？
这里可能就是考虑上述场景，需要进行更新请求


# 执行顺序
## 初始化
```cpp
  exit_code = ara::exec::internal::Startup(argc, argv);
```

## `Startup`实现
```cpp
// VECTOR Next Construct AutosarC++17_10-A3.9.1: MD_EM_A3.9.1_main_parameters
// VECTOR Next Construct AutosarC++17_10-A18.1.1: MD_EM_A18.1.1_main_parameters
/*!
 * \internal
 * - Set signal mask
 * - Check plausibility of input parameters
 * - Run MachineInitApplication
 * - Init filesystem
 * - Init reactor
 * - Init logging
 * - Run Executionmanagement.
 * \endinternal
 */
int Startup(int argc, char const* const argv[]) {
  int exit_code{EXIT_FAILURE};  // VECTOR Same Line AutosarC++17_10-A3.9.1: MD_EM_A3.9.1_exit_code
  bool const is_signal_mask_set{ara::exec::internal::SetSignalMask()};
  if (!is_signal_mask_set) {
    ara::core::Abort("Fatal error in EM: Cannot set initial signal mask. Exiting Execution Manager...");
  } else {
    ara::exec::internal::CommandLineArguments args{ara::exec::internal::ParseArguments(argc, argv)};

    osabstraction::process::machine_init_application::Result const init_result{
        osabstraction::process::machine_init_application::RunMachineInitApplication(
            vac::container::CStringView::FromString(args.path_to_machine_init_app))};
    if (!init_result) {
      ara::core::String const machine_init_application_error_string{
          "Fatal error in EM: Init Application run from " + args.path_to_machine_init_app +
          " wasn't successfully executed. Exiting Execution Manager..."};
      ara::core::Abort(machine_init_application_error_string.c_str());
    } else if (!osabstraction::init::InitializeFileSystem()) {
      ara::core::Abort("Fatal error in EM: File system initialization failed. Exiting Execution Manager...");
    } else {
      constexpr std::size_t kMaxLoggingReactorCallbacks{2};
      const std::size_t kMaxNumReactorCallbacks{
          event_manager::IpcConnectionManager::kMaxReactorCallbacks + (args.number_state_clients + 1) +
          recovery_action_manager::RecoveryActionManager::kMaxReactorCallbacks +
          failure_handler_manager::FailureHandlerManager::kMaxReactorCallbacks + kMaxLoggingReactorCallbacks};
      if (kMaxNumReactorCallbacks > std::numeric_limits<std::uint16_t>::max()) {
        std::cerr << "kMaxNumReactorCallbacks value is too big.";
      }

      ara::core::Result<osabstraction::io::reactor1::Reactor1::ConstructionToken> create_reactor_result{
          osabstraction::io::reactor1::Reactor1::Preconstruct(static_cast<std::uint16_t>(kMaxNumReactorCallbacks))};

      if (!create_reactor_result.HasValue()) {
        ara::core::Abort("Fatal error in EM: Reactor creation failed. Exiting Execution Manager...");
      } else {
        osabstraction::io::reactor1::Reactor1 reactor{std::move(create_reactor_result.Value())};

        ara::core::Result<void> const log_init_result{amsr::log::internal::InitializeComponent(
            reactor, static_cast<ara::core::StringView>(args.path_to_logging_config))};

        if (!log_init_result) {
          ara::core::ErrorCode const ec{log_init_result.Error()};
          ara::core::String const ec_message{ec.Message()};
          ara::core::String const ec_user_message{ec.UserMessage()};
          ara::core::String const log_init_error_string{
              "Fatal error in EM: Failed to initialize Logging while reading " + args.path_to_logging_config +
              "\nHint: Option for logging configuration path: '-l'\nError: " + ec_message +
              "\nMore details: " + ec_user_message + "\nExiting Execution Manager...\n"};
          ara::core::Abort(log_init_error_string.c_str());
        } else {
          ara::log::Logger& logger{ara::log::CreateLogger(
              ara::exec::internal::DaemonLogIds::kLogContextIdGlobal.ToString(), "Global logging context")};
          LogCommandLineArguments(args, logger);

          if (RunExecutionmanager(args.path_to_machine_manifest, args.path_to_applications, reactor, logger,
                                  args.number_state_clients)) {
            exit_code = EXIT_SUCCESS;  // VECTOR Same Line AutosarC++17_10-A3.9.1: MD_EM_A3.9.1_exit_code
          }

          static_cast<void>(amsr::log::internal::DeinitializeComponent());
        }
      }
    }
  }

  return exit_code;
}
```

`Startup()`
    `SetSignalMask`
        `sigemptyset(&mask)`
            清空信号集
        `sigaddset(&mask, amsr::signal::internal::kSigChldIdentifier)`
            向信号集中添加特定信号 ———— `SIGCHLD`
                PS：`SIGCHLD信号`是在一个`子进程终止或停止时`发送给其`父进程`的信号。
                `父进程`可以通过`捕获SIGCHLD信号`来等待`子进程的终止状态`，以便进行适当的处理，比如`回收子进程的资源`、`获取子进程的退出状态`等。
        `sigprocmask(SIG_BLOCK, &mask, nullptr)`
            把信号集中的信号进行阻塞

    解析命令行参数



### `RunMachineInitApplication`
是真正`创建子进程`的函数，做了以下逻辑：
得到`executable_path`
然后 通过`stat` 检查可执行文件/程序镜像是否是 `普通文件`
调用`CreateProcess`
    `::fork()`

核心：`waitpid`

### `RunMachineInitApplication`
```cpp
/*!
 * \internal
 * - If empty path is provided use kDefaultMachineInitPath
 * - If file does not exist in the given path
 *  - Create Result with this information and return it
 * - If file is regular and executable
 *  - run process and wait for termination or error
 * - Return Result containing information about file existance, process creation errors, exit status code.
 * \endinternal
 */
Result RunMachineInitApplication(vac::container::CStringView const &path_to_application) {
  bool file_exists{false};
  std::error_code creation_error{0, std::generic_category()}; /* Default value is no error. */
  Result::Status status{EXIT_FAILURE};

  vac::container::CStringView executable_path{path_to_application};
  if (executable_path.empty()) {
    executable_path = kDefaultMachineInitPath;
  }

  /* #00 Check that given program image is a regular file. */
  struct stat file_status {};
  if (::stat(executable_path.c_str(), &file_status) == kApiError) {
    file_exists = false;
    // VECTOR Next Construct AutosarC++17_10-M5.0.21: MD_OSA__M5.0.21_O_NONBLOCK
    // VECTOR Next Construct AutosarC++17_10-M2.13.2: MD_OSA_M2.13.2_O_NONBLOCK
  } else {
    file_exists = true;
    if ((!S_ISREG(file_status.st_mode)) || ((file_status.st_mode & S_IXUSR) == 0)) {
      /* File exists but is not a regular file or is not an exectuable. */
      creation_error.assign(EINVAL, std::generic_category());
    } else {
      OsProcessSettings process_settings{};

      std::string const name{"machine_init_application"};
      std::string const working_dir{"."};

      ara::core::Result<ProcessHandle> result{
          OsProcess::CreateProcess(executable_path.c_str(), name, working_dir, process_settings)};
      OsProcess const process{result.Value()};
      ProcessId const machine_init_app_pid{process.GetId()};

      ProcessId pid{kInvalidProcessId};
      do {
        pid = ::waitpid(machine_init_app_pid, &status, 0);
        if (pid == kApiError) {
          // VECTOR Next Construct AutosarC++17_10-M19.3.1: MD_OSA_M19.3.1_PosixApi
          if (errno != EINTR) {
            /* Allow interruption by signal because waitpid is executed again. Other errors are not allowed. */
            creation_error.assign(errno, std::generic_category());
            break;
          }
        }
      } while (pid != machine_init_app_pid);
    }
  }
  /* Prepare the result. The evaluation of the result is done in the result class. */
  return Result{file_exists, creation_error, status};
}
```

`CreateProcess`
得到可执行文件的路径，调用`realpath()`得到绝对路径
调用`::fork()`
父进程执行完fork直接就可以返回了
子进程还要通过`cgroup`进行一些设置：首先向cgroup中添加pid。然后调用一系列系统调用
设置cwd（当前工作目录）
最后调用`execve`，加载可执行文件镜像，设置
```cpp
      static_cast<void>(
          execve(absolute_binary_path.data(), const_cast<char* const*>(argv), const_cast<char* const*>(envp)));
// 2，3参数分别是：指向命令行参数列表的指针，指向环境变量字符串数组的指针
```

`CreateProcess`实现
```cpp
// VECTOR Next Construct Metric-HIS.PATH: MD_OSA_Metric-HIS.PATH_CreateProcess
/*!
 * \internal
 * - Try to resolve the absolute path to the executable image
 * - If resolving the path failed
 *   - Return an error.
 * - Otherwise call fork.
 * - If creating a child process failed
 *   - Return an error.
 * - If creating a child process succeeded
 *   - In the parent process:
 *     - Return the pid of the child.
 *   - In the child process:
 *     - If setting the Resource Group is required
 *       - Try to assign the Process to the Resource Group. (apply before other settings to ensure assign permissions)
 *       - If assign the Process failed
 *         - End execution of the process as soon as possible.
 *     - If setting scheduling policy and priority is required
 *       - Try to change scheduling policy and priority of the process.
 *       - If changing scheduling policy and priority failed
 *         - End execution of the process as soon as possible.
 *     - Try to change the working directory of the process.
 *     - If changing the working directory failed
 *       - End execution of the process as soon as possible.
 *     - If setting CPU affinity is required
 *       - Try to set the CPU affinity of the process.
 *       - If setting the CPU affinity failed
 *         - End execution of the process as soon as possible.
 *     - If changing the secondary groups is required
 *       - Try to change the secondary groups of the process.
 *       - If changing the seconday groups failed
 *         - End execution of the process as soon as possible.
 *     - If changing the primary group ID is required
 *       - Try to change the primary group ID of the process.
 *       - If changing the primary group ID of the process failed
 *         - End execution of the process as soon as possible.
 *     - If changing the user ID is required
 *       - Try to change the user ID of the process.
 *       - If changing the user ID of the process failed
 *         - End execution of the process as soon as possible.
 *     - If execution of the process has not yet been ended
 *       - Call execve.
 *       - If execve failed
 *         - End execution of the process as soon as possible.
 * \endinternal
 */
ara::core::Result<ProcessId> OsProcess::CreateProcess(PathToExecutable const& executable_path,
                                                      ExecutableName const& name, WorkingDirectory const& working_dir,
                                                      OsProcessSettings& settings) {
  char const* const* const argv{settings.GenerateArgv(name)};
  char const* const* const envp{settings.GenerateEnvp()};

  // kInvalidState will be overwritten by the real error, if not, the function does not work correctly.
  ara::core::Result<ProcessId> result{ara::core::Result<ProcessId>::FromError(
      MakeErrorCode(OsabErrc::kProcessCreationFailed, CreateProcessDetailedError::kInvalidInternalState, nullptr))};

  std::array<char, PATH_MAX> absolute_binary_path{};
  if (::realpath(executable_path.c_str(), absolute_binary_path.data()) == nullptr) {
    // Image path does not exist.

    result = ara::core::Result<ProcessId>::FromError(
        MakeErrorCode(OsabErrc::kProcessCreationFailed, CreateProcessDetailedError::kImagePathInvalid, nullptr));
  } else {
    ProcessId pid{::fork()};
    if (pid < 0) {
      // Creating a child process failed.

      result = ara::core::Result<ProcessId>::FromError(
          MakeErrorCode(OsabErrc::kProcessCreationFailed, CreateProcessDetailedError::kGeneralInteralError, nullptr));
    } else if (pid > 0) {
      // Creating a child process succeeded - path for parent process.

      result = ara::core::Result<ProcessId>::FromValue(pid);
    } else {
      // Creating a child process succeeded - path for child process.

      if (settings.GetResourceGroup().has_value()) {
        const ResourceGroup& resource_group{settings.GetResourceGroup().value()};
        const osabstraction::process::internal::Cgroup cgroup{resource_group};

        // Adding the calling process to resource group by calling AddPid with pid = 0
        if (!cgroup.AddPid(pid)) {
          ara::core::Abort("CreateProcess(): Invalid ResourceGroup configured.");
        }
      }

      if (settings.GetSchedulingSettings().has_value()) {
        if (!SetSchedulerSettings(settings.GetSchedulingSettings().value())) {
          ara::core::Abort("CreateProcess(): SetSchedulerSettings() failed.");
        }
      }

      if (settings.GetNiceValue().has_value()) {
        if (!SetNiceValue(settings.GetNiceValue().value())) {
          ara::core::Abort("CreateProcess(): SetNiceValue() failed.");
        }
      }

      constexpr int32_t kSystemCallFailed{-1};

      if (::chdir(working_dir.c_str()) == kSystemCallFailed) {
        ara::core::Abort("CreateProcess(): chdir() system call failed.");
      }

      if (settings.GetCpuAffinity().has_value() && settings.GetCpuAffinity().value().any()) {
        CpuAffinity const& cpu_affinity{settings.GetCpuAffinity().value()};
        cpu_set_t cpu_set{};

        for (std::size_t cpu_id{}; cpu_id < cpu_affinity.size(); ++cpu_id) {
          if (cpu_affinity.test(cpu_id)) {
            SetCpuId(cpu_id, cpu_set);
          }
        }

        if (sched_setaffinity(0, sizeof(cpu_set), &cpu_set) == kSystemCallFailed) {
          ara::core::Abort(
              "CreateProcess(): sched_setaffinity() system call failed, probably because invalid cores are selected "
              "for core affinity.");
        }
      }

      if (settings.GetSecondaryGroups().has_value()) {
        const std::vector<osabstraction::process::GroupId>& groups{settings.GetSecondaryGroups().value()};
        if (::setgroups(static_cast<size_t>(groups.size()), groups.data()) == kSystemCallFailed) {
          ara::core::Abort("CreateProcess(): setgroups() system call failed.");
        }
      }

      if (settings.GetPrimaryGroupId().has_value()) {
        GroupId const gid{settings.GetPrimaryGroupId().value()};
        if (::setregid(gid, gid) == kSystemCallFailed) {
          ara::core::Abort("CreateProcess(): setregid() system call failed.");
        }
      }

      if (settings.GetUserId().has_value()) {
        UserId const uid{settings.GetUserId().value()};
        if (::setreuid(uid, uid) == kSystemCallFailed) {
          ara::core::Abort("CreateProcess(): setreuid() system call failed.");
        }
      }

      /* VECTOR Next Construct AutosarC++17_10-A5.2.3: MD_OSA_A5.2.3_ConstCastForExecvePosixSpawn */
      static_cast<void>(
          execve(absolute_binary_path.data(), const_cast<char* const*>(argv), const_cast<char* const*>(envp)));

      // Execution only gets past execve if loading the other executable failed.

      ara::core::Abort("CreateProcess(): unexpected path executed after execve() system call.");
    }
  }

  return result;
}
```

### `RunExecutionmanager(path_to_machine_manifest, path_to_applications, reactor, logger, number_state_clients)`
构造`FunctionGroupManager`类型的对象，它维护一个`FunctionGroupList`

读配置，填充以下几个类型的变量
`ProcessListBuilder`，持有`manifest_name_`，持有对`TimerManager`对象的引用、`FunctionGroupManager`对象的引用
`ProcessList`，即`std::vector<std::unique_ptr<ProcessStartupConfig>>`

核心类：
```cpp
// VECTOR Next Construct AutosarC++17_10-M3.4.1: MD_EM_M3.4.1_class
/*!
 * \brief                 Main class representing the Execution Management daemon.
 */
class Executionmanager final : public ExecutionManagerInterface {
 public:
  /*!
   * \brief               Unique pointer to a function group object.
   */
  using FunctionGroupManagerPtr = std::unique_ptr<function_group_manager::FunctionGroupManager>;

  /*!
   * \brief               Constructs the Execution Management daemon.
   *
   * \param[in, out]      logger   Logger instance reference to be used by the Execution Management.
   * \param[in, out]      reactor  Reactor to be used by the Execution Management.
   *
   * \context             ANY
   *
   * \pre                 -
   *
   * \reentrant           FALSE
   * \synchronous         TRUE
   * \threadsafe          TRUE
   */
  explicit Executionmanager(ara::log::Logger& logger, osabstraction::io::reactor1::Reactor1Interface& reactor);

  /*!
   * \brief               Destroys the object.
   *
   * \details             Terminates all adaptive applications before destroying the object.
   */
  // VECTOR Next Line AutosarC++17_10-A15.5.1: MD_EM_A15.5.1_explicit_noexcept_falsepositive
  ~Executionmanager() noexcept final;

  /*!
   * \brief               Initializes the Execution Management.
   *
   * \details             Aborts and logs an error if initialization failed.
   *
   * \param[in]           function_group_manager   Unique pointer R-value to a function group manager.
   * \param[in]           process_list             The R-value list of processes to be managed.
   * \param[in]           number_state_clients     The max. number of state clients that can be connected at the same
   *                      time.
   *
   * \context             InitPhase
   *
   * \pre                 -
   *
   * \reentrant           FALSE
   * \synchronous         TRUE
   * \threadsafe          FALSE
   *
   * \trace               DSGN-ExecutionManager-InitialStateStartup
   */
  void Initialize(FunctionGroupManagerPtr&& function_group_manager, ProcessList&& process_list,
                  std::size_t number_state_clients);

  /*!
   * \brief               Starts the Execution Management state machine.
   *
   * \context             RunningPhase
   *
   * \pre                 -
   *
   * \reentrant           FALSE
   * \synchronous         TRUE
   * \threadsafe          FALSE
   *
   * \trace               DSGN-ExecutionManager-MainLoop
   */
  void Run();

  /*!
   * \copydoc             ExecutionManagerInterface::FindProcessStartupConfigByProcessHandle()
   */
  ProcessStartupConfig* FindProcessStartupConfigByProcessHandle(
      osabstraction::process::ProcessHandle identifier) noexcept final;

  /*!
   * \copydoc             ExecutionManagerInterface::FindProcessStartupConfigByIdentifier()
   */
  ProcessStartupConfig* FindProcessStartupConfigByIdentifier(Identifier pid,
                                                             StartupConfigIndex startup_config_index) noexcept final;

  /*!
   * \copydoc             ExecutionManagerInterface::GetProcessStartupConfigList()
   */
  ProcessList& GetProcessStartupConfigList() noexcept final { return process_all_; }

  /*!
   * \brief               Returns the timer manager object reference.
   *
   * \return              The timer manager object reference.
   *
   * \context             InitPhase
   *
   * \pre                 -
   *
   * \reentrant           FALSE
   * \synchronous         TRUE
   * \threadsafe          FALSE
   */
  vac::timer::TimerManager& GetTimerManager() { return event_manager_.GetTimerManager(); }

  /*!
   * \copydoc             ExecutionManagerInterface::OnEnterSafeState()
   */
  void OnEnterSafeState() noexcept final;

  /*!
   * \brief               Requests to shut down the Execution Management.
   *
   * \details             Sets an internal flag and unblocks the Event Management.
   *
   * \context             ShutdownPhase
   *
   * \pre                 -
   *
   * \reentrant           FALSE
   * \synchronous         TRUE
   * \threadsafe          FALSE
   */
  void RequestShutdown();

 private:
  /*!
   * \brief               Enters the safe state.
   *
   * \context             RunningPhase
   *
   * \pre                 -
   *
   * \reentrant           FALSE
   * \synchronous         TRUE
   * \threadsafe          FALSE
   *
   * \trace               DSGN-ExecutionManager-DaemonStatemachine
   */
  void EnterSafeState();

  /*!
   * \brief               Kills all active adaptive applications.
   *
   * \context             ShutdownPhase
   *
   * \pre                 -
   *
   * \reentrant           FALSE
   * \synchronous         TRUE
   * \threadsafe          FALSE
   */
  void KillAllApps() noexcept;

  /*!
   * \brief               Reactor interface to handle asynchronous operations.
   */
  osabstraction::io::reactor1::Reactor1Interface& reactor_;

  /*!
   * \brief               The logger instance reference.
   */
  ara::log::Logger& log_;

  /*!
   * \brief               Event Manager instance.
   */
  event_manager::EventManager event_manager_;

  /*!
   * \brief Connector between function group manager and recovery action manager.
   */
  ara::exec::internal::state_change_manager::StateChangeManagerConnector scm_connector_{};

  /*!
   * \brief               Manages all recovery decisions issued by the PHM.
   */
  recovery_action_manager::RecoveryActionManager recovery_action_manager_;

  /*!
   * \brief               Manages Failure Handler notifications.
   */
  failure_handler_manager::FailureHandlerManager failure_handler_manager_;

  /*!
   * \brief               Manages function group state changes.
   */
  state_mgmt_srv::StateManager state_change_manager_{};

  /*!
   * \brief               The list of all adaptive applications in the system.
   */
  ProcessList process_all_;

  /*!
   * \brief               Marks if entering safe state has been requested.
   */
  bool safe_state_requested_{false};

  /*!
   * \brief               Marks if a shutdown has been requested.
   *
   * \details             This variable may be accessed from different threads.
   */
  std::atomic_bool shutdown_requested_;

  /*!
   * \brief               The Function Group Manager instance.
   */
  std::unique_ptr<ara::exec::internal::function_group_manager::FunctionGroupManager> function_group_manager_{};
};
```


其中的`StateManager`类，是一个典型的状态机模式，
**核心思想：功能组状态切换时，挂载在该状态下的进程继续执行，否则被kill**
**EM本身要负责把machineFG.startup对应的进程拉起来，然后再被动地通过SM来完成拉起进程、关闭进程的操作**
`StateManager`持有：`map<string, FunctionGroupInterface>`，而`FunctionGroupInterface`持有指向`function_group_manager::FunctionGroup`的指针，通过`FunctionGroup`可以获取到它持有的`StateChangeProcessor`类型指针，
```cpp
  vac::container::StaticMap<amsr::core::StringView, FunctionGroupInterface> func_group_interfaces_{};
```

核心业务：`RunFunctionGroupStateMachines()`
```cpp
/*!
 * \internal
 *  - For each function group interface.
 *    - Run state machine.
 *  - Return whether the state machine is stable or not.
 * \endinternal
 */
bool StateManager::RunFunctionGroupStateMachines() noexcept {
  bool call_again{false};
  for (vac::container::StaticMap<ara::core::StringView, FunctionGroupInterface>::value_type &fg_int_pair :
       this->func_group_interfaces_) {
    FunctionGroupInterface &fg_int{fg_int_pair.second};
    call_again = fg_int.Run() || call_again;
  }
  return call_again;
}
```

监听state_client发来的消息，并进行处理：
```cpp
/*!
 * \internal
 *  - For each connected client in connected_clients_ array.
 *    - If new message is pending.
 *      - Deserialize message.
 *      - If Deserialization fails.
 *        - Close connection
 *      - else
 *        - Execute request.
 *        - Start to receive the next message if connection is still open.
 *     - Else if peer disconnected
 *      Close Connection
 *  - Return true if at least one request has been handled.
 * \endinternal
 */
bool StateManager::ForwardIncomingRequests() noexcept {
  bool request_handled{false};
  for (ipc_server::ConnectionHandle &handle : this->connected_clients_) {
    if (handle != ipc_server::kInvalidHandle) {
      ara::core::Result<ara::core::Span<const std::uint8_t>> new_message{
          this->state_manager_ipc_server_.GetReceiveBuffer(handle)};
      if (new_message.HasValue()) {
        // TODO(vismkk) Const stream not supported yet. It needs to be implemented.
        std::size_t const stream_size{new_message.Value().size()};
        // VECTOR NC AutosarC++17_10-A5.2.4: MD_EM_A5.2.4_const_byte_stream
        // VECTOR NC AutosarC++17_10-A5.2.3: MD_EM_A5.2.3_const_byte_stream
        ara::core::Span<std::uint8_t> const stream{const_cast<std::uint8_t *>(new_message.Value().data()), stream_size};
        ara::core::Result<amsr::exec::internal::communication::Command> const cmd{
            amsr::exec::internal::communication::StateClientProtocolEngine::Deserialize(stream)};
        if (cmd.HasValue()) {
          this->logger_->LogVerbose(
              [&handle](ara::log::LogStream &out) { out << "Received request from state client" << handle; });

          request_handled = true;
          StateClientRequest request;
          request.requester = handle;
          request.request = cmd.Value();
          this->ExecuteStateClientRequest(request);
          if (handle != ipc_server::kInvalidHandle) {
            this->state_manager_ipc_server_.Receive(handle);
          }
        } else {
          this->logger_->LogError() << "State client" << handle
                                    << " sent a corrupted message. Deserialization failed. Terminate connection.";
          this->CloseConnection(handle);
        }
      } else if (new_message == ara::exec::ExecErrc::kDisconnected) {
        this->CloseConnection(handle);
      } else {
        // kBusy error. No pending message.
        // kInvalidInput is not possible as the state manager keeps track of open connection in the connected_clients
        // array.
      }
    }
  }
  return request_handled;
}
```

StateChangeProcessor::`ChangeTransitionState()`
```cpp
/*!
 * \internal
 * - Assert that the state change is valid.
 * - Update current state with next state and enter it.
 * - If the current state is kFinishTransition:
 *   - Change the current state to kIdle and enter it.
 * \endinternal
 */
void StateChangeProcessor::ChangeTransitionState(State next_state) {
  bool is_valid_state_change{false};
  switch (state_) {
    case State::kIdle:
      is_valid_state_change = (next_state == State::kStartTransition) || (next_state == State::kIdle);
      break;
    case State::kStartTransition:
      is_valid_state_change = (next_state == State::kTerminateProcesses) || (next_state == State::kIdle);
      break;
    case State::kTerminateProcesses:
      is_valid_state_change = (next_state == State::kProcessTerminated) || (next_state == State::kStartTransition) ||
                              (next_state == State::kIdle);
      break;
    case State::kProcessTerminated:
      is_valid_state_change = (next_state == State::kStartProcesses) || (next_state == State::kFinishTransition) ||
                              (next_state == State::kIdle) || (next_state == State::kStartTransition);
      break;
    case State::kStartProcesses:
      is_valid_state_change = (next_state == State::kFinishTransition) || (next_state == State::kStartTransition) ||
                              (next_state == State::kIdle);
      break;
    case State::kFinishTransition:
      is_valid_state_change = next_state == State::kIdle;
      break;
    default:
      // is_valid_state_change is already false.
      break;
  }
  assert(is_valid_state_change);
  static_cast<void>(is_valid_state_change);
  state_ = next_state;
  OnEnterState();

  if (state_ == State::kFinishTransition) {
    state_ = State::kIdle;
    OnEnterState();
  }
}
```


功能组状态切换，要内部状态机允许才行：`StateIdle`
```cpp
/*!
 * \internal
 * - Check preconditions.
 * - Execute request.
 * - If transition started:
 *  - Go to state Transition.
 * - else (error occurred)
 *  - Go to state Pending Error.
 * - return response.
 * \endinternal
 */
ara::core::Optional<StateClientResponse> StateIdle::ExecuteRequest(StateClientRequest const &request) noexcept {
  this->AssertInitialized();

  ara::core::Optional<StateClientResponse> response{this->GetContext().ExecuteStateClientRequest(request)};
  if (this->GetContext().GetLastRequestResult() == Context::RequestResult::kSetStateSuccess) {
    this->GetContext().ChangeState(StateHandle::kTransition);
  } else if (this->GetContext().GetLastRequestResult() == Context::RequestResult::kSetStateFailed) {
    this->GetContext().ChangeState(StateHandle::kPendingError);
  } else {
    // GetState request or rejected SetState that does not require error handling. No action required.
  }

  return response;
}
```

真正的功能组状态切换，是由`Context`去做：
`ExecuteStateClientRequest()`
```cpp
/*!
 * \internal
 *  - Check precondition.
 *  - Dispatch command.
 * \endinternal
 */
ara::core::Optional<StateClientResponse> Context::ExecuteStateClientRequest(
    StateClientRequest const& request) noexcept {
  this->AssertInitialized();
  ara::core::Optional<StateClientResponse> result{};
  if (ara::core::holds_alternative<amsr::exec::internal::communication::SetStateRequest>(request.request)) {
    result = this->RequestStateTransition(request);
  } else if (ara::core::holds_alternative<amsr::exec::internal::communication::GetStateRequest>(request.request)) {
    result = this->GetState(
        request.requester,
        ara::core::get<amsr::exec::internal::communication::GetStateRequest>(request.request).GetSequenceNumber());
  } else {
    ara::core::Abort("Command is not part of precondition. Precondition violated.");
  }
  return result;
}
```

修改功能组状态的业务逻辑：Context::`RequestStateTransition()`
```cpp
/*!
 * \internal
 * - Check precondition.
 * - Reject request directly if policy is set to reject all requests.
 * - Reject request directly if target state is SafeState for function group MachineState has been requested by a
 *   state client.
 * - If ongoing request and new target state matches old target state.
 *    - Reject new request with in-transition-to-same-state error.
 * - else
 *   - start the requested state transition
 *   - Output the response resulting from starting the state transition.
 * \endinternal
 */
ara::core::Optional<StateClientResponse> Context::RequestStateTransition(StateClientRequest const& request) noexcept {
  this->AssertInitialized();
  ara::core::Optional<StateClientResponse> result{};

  // The command is a SetState request. This is asserted by precondition.
  amsr::exec::internal::communication::SetStateRequest const& cmd{
      ara::core::get<amsr::exec::internal::communication::SetStateRequest>(request.request)};
  ara::core::StringView const fg_name{this->function_group_->GetName()};
  ara::core::StringView const target_state{cmd.GetTargetStateName()};

  FgRequestState::RequestInformation const request_information{request.requester, cmd.GetSequenceNumber()};

  if (this->reject_state_change_requests_) {
    this->context_logger_->LogInfo() << "Function group's (" << fg_name << ") state transition to "
                                     << this->request_state_.GetTargetState()
                                     << " has been rejected. Further state transitions are not allowed.";
    StateClientResponse response;
    response.response =
        amsr::exec::internal::communication::Command{amsr::exec::internal::communication::SetStateResponse(
            cmd.GetSequenceNumber(), ara::core::Result<void>{ara::exec::ExecErrc::kGeneralError})};
    response.destination = request.requester;
    result.emplace(response);
  } else if ((this->function_group_->GetName() == kManifestFormatMachineState) &&
             (target_state == kManifestFormatMachineModeSafeState) &&
             (request.requester != ipc_server::kSafeStateTransitionRequester)) {
    // SafeState in function group MachineState can only be requested by the execution management itself.
    // Reject all regular state client requests.
    this->context_logger_->LogWarn()
        << "Function group's (" << fg_name << ") state transition to " << this->request_state_.GetTargetState()
        << " has been rejected. This special state cannot be requested for this function group.";

    this->function_group_->GetStateChangeProcessor().CancelTransition();

    // Set error data to be collected later.
    this->request_state_.InvalidTransitionRequested(request_information);

    // Causes state change to Pending Error state.
    this->request_result_ = RequestResult::kSetStateFailed;
  } else if (this->request_state_.StateTransitionOngoing() && (target_state == this->request_state_.GetTargetState())) {
    this->context_logger_->LogDebug() << "Function group's (" << fg_name << ") state transition to "
                                      << this->request_state_.GetTargetState()
                                      << " has been rejected. A transition to the same state is already in progress.";
    StateClientResponse response;
    response.response =
        amsr::exec::internal::communication::Command{amsr::exec::internal::communication::SetStateResponse(
            cmd.GetSequenceNumber(), ara::core::Result<void>{ara::exec::ExecErrc::kInTransitionToSameState})};
    response.destination = request.requester;
    result.emplace(response);
  } else {
    // Can pass target_state because it is not stored by StartStateTransition() in any way that outlives this call
    // so there is no danger that it is used after it is no longer guaranteed to be stable.
    result = this->StartStateTransition(request_information, target_state);
  }
  return result;
}
```

`Context`::`StartStateTransition`
```cpp
/*!
 * \internal
 * - If a transition is ongoing
 *   - cancel it.
 * - If there is a pending response
 *   - fetch it for later output.
 * - Try to start the requested state transition.
 * - If the state requesed by the transition to start exists
 *   - start the transition to this state.
 *   - Update the information about the ongoing state transition to reflect this ongoing transition.
 *   - Output that starting the transition succeeded.
 * - If the state requesed by the transition to start does not exist
 *   - abort ongoing state transition.
 *   - Update the information about the ongoing state transition to reflect this invalid request.
 *   - Output that starting the transition failed.
 * - Output the fetched pending response.
 * \endinternal
 */
ara::core::Optional<StateClientResponse> Context::StartStateTransition(
    FgRequestState::RequestInformation request_information, ara::core::StringView target_state) noexcept {
  ara::core::Optional<StateClientResponse> result{};

  StateHandle const current_state{GetActiveState().GetHandle()};

  if (current_state == StateHandle::kTransition) {
    ara::core::StringView const fg_name{this->function_group_->GetName()};
    ara::core::StringView const previous_target_state{request_state_.GetTargetState()};
    this->context_logger_->LogInfo() << "Function group's (" << fg_name << ") state transition to \""
                                     << previous_target_state
                                     << "\" has been cancelled by a new request. New target state is \"" << target_state
                                     << "\".";

    SetStateTransitionResult(ara::core::Result<void>{ara::exec::ExecErrc::kCancelled});
  }

  if ((current_state == StateHandle::kTransition) || (current_state == StateHandle::kPendingError)) {
    result = ara::core::Optional<StateClientResponse>{CreateResponseMessage()};
  }

  if (this->function_group_->UpdateTargetState(target_state)) {
    // Use a string view to the persistent name of the state.
    ara::core::StringView const new_target_state{this->function_group_->GetTargetState()};

    this->function_group_->GetStateChangeProcessor().PerformFunctionGroupStateChange(new_target_state.ToString());    //  ！！！

    this->request_state_.StateTransitionStarted(new_target_state, request_information);

    // Causes state change to Transition state.
    this->request_result_ = RequestResult::kSetStateSuccess;
  } else {
    // Unknown function group state.
    this->function_group_->GetStateChangeProcessor().CancelTransition();

    // Set error data to be collected later.
    this->request_state_.InvalidTransitionRequested(request_information);

    // Causes state change to Pending Error state.
    this->request_result_ = RequestResult::kSetStateFailed;
  }

  return result;
}
```

FunctionGroup::`UpdateTargetState()`
```cpp
/*!
 * \internal
 * - If parameter state is part of the states in the Function Group:
 *   - Set the active state to the value in the parameter state.
 *   - Notify all Process Startup Configuration registered as listeners about the state change. Use the internally
 *     stored string that will be valid until the object is destroyed.
 *   - Return true.
 * - Otherwise:
 *   - Return false.
 * \endinternal
 */
bool FunctionGroup::UpdateTargetState(StringView state) {
  bool ret_val{false};
  FunctionGroupStates::const_iterator const iter{std::find_if(
      states_.cbegin(), states_.cend(), [state](String const& group_state) { return state == group_state; })};
  if (iter != states_.end()) {
    target_state_ = StringView{*iter};
    for (ListenerType& listener : listener_list_) {
      listener.get().OnStateChange(target_state_);
    }
    ret_val = true;
  }
  return ret_val;
}
```

`ProcessStartupConfig::OnStateChange()`
```cpp
/*!
 * \internal
 * - If one of the assigned Function Group States is equal to the new state:
 *   - Set this process to enabled.
 *   - Set the activating function group and function group state.
 * - Otherwise:
 *   - set this process to disabled.
 * \endinternal
 */
void ProcessStartupConfig::OnStateChange(StringView new_state) {
  std::vector<std::string>::iterator const state_itr{
      std::find(function_group_states_.begin(), function_group_states_.end(), new_state.ToString())};
  is_enabled_ = state_itr != function_group_states_.end();

  if (is_enabled_) {
    activation_cause_function_group_ = function_group_;
    activation_cause_function_group_state_ = *state_itr;
  }
}
```

！！！**真正执行进程启动、关闭**
StateChangeProcessor::`PerformFunctionGroupStateChange()`
```cpp
/*!
 * \internal
 * - Store requested target function group state.
 * - Reset transition failed flag to false.
 * - Increment the transition counter to mark a new transition.
 * - Change state to kStartTransition.
 * \endinternal
 */
void StateChangeProcessor::PerformFunctionGroupStateChange(std::string function_group_state) {
  function_group_state_ = function_group_state;

  finished_transition_error_.reset();

  ++transition_counter_;

  ChangeTransitionState(State::kStartTransition);
}
```

**进程启动/关闭的核心逻辑**：
`StateChangeProcessor::OnEnterState()`
```cpp
/*!
 * \internal
 * - Do nothing in state kIdle.
 * - In state kStartTransition or kProcessTerminated:
 *   - Notify about the Function Group transition.
 * - In state kTerminateProcess:
 *   - Empty the state change action queue.
 *   - Update the list of processes which shall not run.
 * - In state kStartProcesses:
 *   - Update the list of processes which shall run.
 * - In state kFinishTransition:
 *   - Report the state change result to the state change manager.
 * \endinternal
 */
void StateChangeProcessor::OnEnterState() {
  StringView const function_group_name{this->function_group_ref_.GetName()};
  StringView const state_name{function_group_state_};

  switch (state_) {
    case State::kIdle:
      break;
    case State::kStartTransition:
      LogAllActiveProcessStartupConfigurations(log_, function_group_ref_);

      recovery_action_manager_.NotifyFunctionGroupTransition(
          recovery_action_client::FunctionGroupStateTransitionPhase::kStarted, function_group_name, state_name);
      break;
    case State::kTerminateProcesses:
      state_change_action_queue_.clear();

      UpdateListAppsToTerminate();

      break;
    case State::kProcessTerminated:
      // TODO(vismkk): We need a sequence number here now because we can change the direction at any time and we need
      //               the confirmation for the correct termination.
      recovery_action_manager_.NotifyFunctionGroupTransition(
          recovery_action_client::FunctionGroupStateTransitionPhase::kTerminationFinished, function_group_name,
          state_name);
      break;
    case State::kStartProcesses:
      UpdateListAppsToExecute();
      break;
    case State::kFinishTransition:
      this->is_transition_finished_and_not_reported_ = true;
      break;
    default:
      ara::core::Abort("StateChangeProcessor::OnEnterState(): state_ has an invalid value.");
      break;
  }
}
```




`Run()`
```cpp
/*!
 * \internal
 * - If the State Change Processor is in state kIdle, kStartTransition, kProcessTerminated or kFinishTransition:
 *   - Return false.
 * - If the State Change Processor is in state kTerminateProcesses:
 *   - Process termination actions.
 *   - If performing the required actions modifies the state of any process return true, otherwise false.
 * - If the State Change Processor is in state kStartProcesses:
 *   - Process start actions.
 *   - If performing the required actions modifies the state of any process return true, otherwise false.
 * - Otherwise abort because of undefined state.
 * \endinternal
 */
bool StateChangeProcessor::Run() {
  bool modified_state{false};
  switch (state_) {
    case State::kIdle:
      /* Nothing to do. */
      break;
    case State::kStartTransition:
      break;
    case State::kTerminateProcesses:
      modified_state = ProcessTerminateActions();
      break;
    case State::kProcessTerminated:
      break;
    case State::kStartProcesses:
      modified_state = ProcessStartActions();
      break;
    case State::kFinishTransition:
      break;
    default:
      ara::core::Abort("StateChangeProcessor::Run(): state_ has an invalid value.");
      break;
  }

  /* Run should be called again if the a state of a process changed because this may trigger further state changes. */
  return modified_state;
}
```

#### `Initialize()`





#### `Run()`















