# 执行顺序
```cpp
  exit_code = ara::exec::internal::Startup(argc, argv);
```
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



`RunMachineInitApplication`
是真正`创建子进程`的函数，做了以下逻辑：
得到`executable_path`
然后 通过`stat` 检查可执行文件/程序镜像是否是 `普通文件`
调用`CreateProcess`
    `::fork()`

核心：`waitpid`


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

























