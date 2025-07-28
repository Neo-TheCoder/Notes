
# VECTOR文档



# 1 介绍
EM是AUTOSAR自适应平台的初始流程。
它是`第一个``激活`并`启动`AUTOSAR自适应平台其他部分的进程。
在运行期间，它根据`Machine要求的状态``启动`和`终止`进程。
它与操作系统协同工作，`根据配置`运行进程。

### 1.1.1 依赖
#### 自适应应用程序
如果没有明确禁用，由EM启动的每个进程都必须通过`ApplicationClient API`报告其内部执行状态。

#### 平台健康管理（PHM）
`PHM`可通过`RecoveryActionClient API`将自己注册为`进程`和`功能组状态变化`的观察者。
作为恢复行动，PHM可以请求`重启进程`或`进入安全状态`(？？？)。

#### 状态管理（SM）
SM可以通过改变`现有功能组的当前状态`来启动和停止进程。
通过`StateClient API`（有不同版本，参见第 2.1.2.4 和 2.1.2.5 章）启动和停止进程。

#### 故障处理程序
失败处理程序（`Failure Handler`）向`用户进程`提供`EM启动的进程的终止`的`最新信息`。




### 1.1.2 组件设计
Vector 的 EM 实现：
由一个包含主要server逻辑的`守护进程` 和 几个通过`IPC``与守护进程通信`的`client库`组成。
`EM提供的每项服务`都是作为一个`单独的库`实现的。

* EM 的守护进程启动后，首先执行`初始应用程序`并`等待其终止`。
* 随后，`EM`完成`内部初始化`。
* 它会`解析机器配置`并`查找已安装的可执行文件和可执行文件配置`。
* EM 初始化完成后，它将切换到机器状态`MachineFG “Startup”`，启动配置在机器中的所有进程。
进程，并考虑其依赖关系。
启动从这里开始，EM 是一个被动的服务器守护进程，只有在响应 IPC 服务请求时才会变得活跃。

SM可触发功能组状态转换，PHM可以将自己注册为进程状态变化的观察者。
作为`Recovery Action`，PHM 可以请求进程重启或进入安全状态。


### 1.1.3 Applications, Processes and Startup Configurations
在EM的上下文中，有些术语很容易混淆：
- 应用程序
- 可执行文件
- 进程
- 启动配置
- POSIX 操作系统进程

`应用程序`是一个逻辑概念，EM 并不了解它。
每个`应用程序`都由`多个`可在不同机器上部署的`可执行文件`组成。
每个`可执行文件`在运行时有多个实例，即所谓的`进程`（每个可执行文件会被加载为多个进程执行）。

每个进程可以在多个启动配置下被启动，例如定义进程的启动参数（每个APP，StartupConfigs可以配置多份，在不同时期，执行不同的启动配置）。
`同一进程`只能有`一个启动配置`可以同时执行。
`一个启动配置`会产生一个`正在运行的POSIX操作系统进程实例`


#### 2.1.1.1 Startup Configuration
`Caution`
The Startup Configuration only defines startup attributes of the Process.
Depending on the inherited Process privileges and underlying operating system (i.e. Linux) attribute values could be changed by the Process itself or another Process during runtime.

#### 2.1.1.2 Process States
During runtime a Process has a Process State. AUTOSAR defines the following Process States:
> `Idle`
The Process has not been started by EM yet.

> `Starting`
The Process has been started by EM but did not report Execution State Running yet. 
During Starting state the Process performs initialization activities.

> `Running`
The Process has passed its initialization and is fully functional.

> `Terminating`
The Process is going the terminate. 
Either self initiated or initiated by EM.

> `Terminated`
The Process is terminated. 
Its OS resources have been freed.


#### 2.1.1.3 Process Startup
在功能组状态从一个功能组状态过渡到另一个功能组状态期间，EM会启动所有引用目标功能组状态的进程，或者更准确地说，启动配置引用目标功能组状态的进程。
例如 具有`Startup Configuration SC1`的`进程P1`运行在当前的功能组状态`FG1::S1`，
而具有`Startup Configuration SC2`的`进程 P1`会处于下一个功能组状态`FG1::S2`。
在从`FG1::S1`到`FG1::S2`的功能组状态转换期间，进程将被终止并以不同的启动配置：`Startup Configuration SC2`重新启动。
***不同的`Startup Configuration` 在不同的`FG的state` 下被作为process的启动参数***
如果两个功能组状态中的启动配置相同，则进程不会被触动。
EM 通过分配一个操作系统进程来启动进程，该进程的属性已在启动配置中配置。
如果进程在报告`Execution State`（见 2.1.1.7）运行后，但在功能组状态转换结束前异常终止（即退出状态不是`EXIT_SUCCESS`），则功能组状态转换失败。

`EM不会自动重启进程`
自适应应用程序通常具有守护进程行为：
初始化后，它们在后台等待并提供一些服务，直到收到来自 EM 的终止请求。
但在某些情况下，进程也会执行一些初始化或清理任务并自行终止。
这种进程被称为 “自终止 ”进程。
这被视为正常行为。
EM 不会自动重启进程。

#### 2.1.1.4 Process Shutdown
在从一个功能组状态过渡到另一个功能组状态期间，EM会终止所有未引用目标功能组状态的进程，
更确切地说，是没有引用目标功能组状态的启动配置的进程。
在功能组状态转换期间，如果所有进程不再被活动的功能组状态集引用，EM 就会向它们发送终止请求（POSIX 信号`SIGTERM`）。
如果进程忽略终止请求，没有终止，EM 将`在配置的超时`后`强制终止该进程`。

#### 2.1.1.5 启动/关闭超时监控
EM 监控进程状态的变化。
当进程启动时，EM 会将其状态从`Idle`更改为`Starting`。
从这一刻起，EM 开始`监控 启动进程 报告执行状态 运行的时间`。
！！！**如果进程在配置的时间内没有报告，就会被关闭**。

由于功能组状态转换失败，`SM将收到超时错误`。
在进程关闭过程中，EM会将进程的状态更改为`Terminating`。
然后，EM 开始监控进程终止前的时间。
如果进程没有在终止超时内终止（也就是调用EM client提供的Report接口，报告自己Terminating），EM 就会强制终止进程。
默认超时为 30 秒。



### 2.1.2 State Management
EM provides API which allows `SM` to `request a Function Group state change` as well as `fetching the current state of a given Function Group`.




#### 2.1.2.1 Function Groups
EM提供`功能组`作为控制进程`启动`和`关闭`的机制。
一个功能组由其名称和一组状态定义。
每个功能组都有一个名为`Off`的强制状态，这是`默认状态`。
一台机器可以有多个功能组。
`SM`通过`StateClient API`控制当前状态。
进程可以引用功能组状态。
当前激活的功能组状态定义了EM应运行哪些进程。
从`已定义状态` 进入 `未定义功能组状态` 有两种方法：
> 请求 并 启动 功能组状态转换。
> 当功能组处于 已定义状态 时，一个进程因错误而终止。
如果所有引用新状态的进程的进程状态为`Running`，而所有未引用任何当前活动功能组状态的进程的进程状态为`Terminated`，则 EM 认为功能组状态转换已完成。
`功能组状态`转换分为两个阶段：
* 第一步，`终止`所有未被目标功能组状态引用的进程。
* 第二步是`启动`所有被目标功能组状态引用且尚未运行的进程。
为了加快功能组状态转换的速度，在每个阶段，只要进程之间没有依赖关系，进程的关闭或启动都是`并行`进行的。



#### 2.1.2.2 Errors during Function Group State Transition
A Function Group is only in a `valid and defined` Function Group State,
when the last Function Group State Transition is completed successfully,
otherwise it is in an Undefined Function Group State.

`Note`
The `undefined_state_callback` of the R20.11 State Client API is only called on a Function Group State Transition from a defined Function Group State to an Undefined Function
Group State `caused by an errornous termination of a process of the Function Group`.
That means, the callback it is not called if such a termination takes place during a Function Group State Transition.(进程可能因为timeout而被SIGKILL信号杀死)



#### 2.1.2.3 Machine States
每台机器都必须有一个预定义的功能组`MachineFG`。
功能组 MachineFG 具有以下强制状态：
> `Startup`
> `Shutdown`
> `Restart`
> `SafeState`
当机器启动时，EM将执行从`Off`到`Startup`的功能组状态转换。
`Startup`状态应包含 Vector BSW 的所有必须启动的进程。
`Shutdown`和`Restart`状态用于实现`Machine`的关机或重启。
如果PHM请求进入`SafeState`，则进入`SafeState`。
这是一个最终状态，不能再离开。




### 2.1.3 Application Recovery Actions
#### 2.1.3.1 Process Monitoring
EM provides an API which allows `PHM` to be informed about `Process State changes`.
PHM can fetch a list of all Processes which are known to EM.
After the list has been exchanged EM informs PHM about Processes’ state changes.

#### 2.1.3.2 Function Group State Monitoring
EM provides an API which allows `PHM` to be informed about `Function Group state changes`.

#### 2.1.3.3 Process Restart
Using the information provided by Process Monitoring API `PHM can request to restart a Process`,
if detects an anomaly(exception).
`EM will not consider execution dependencies during restart`.
Be aware that this Process restart can bring a Function Group or whole Machine into an inconsistent state.

#### 2.1.3.4 Enter Safe State
If PHM detects `a serious problem` which affects the whole Machine (e.g. SM is not responding) it can request to enter safe state.
This bypasses(ignore) SM. `After EM has received this request, it will ignore requests from SM`.
AUTOSAR 19-03 does not specify what safe state really means.
**Vector’s implementation of EM sets the MachineFG to SafeState and all other Function Groups to Off**

`Caution`
In order to maximize the chances of successful Function Group State Transition to SafeState, EM tolerates shutdown timeouts and continues with its course of actions(？？？).
On the other hand successful start of Processes assigned to SafeState is considered to be indispensable.
For that reason any occurrence of startup timeout unavoidably leads to inconsistent and unrecoverable state.
For this reason try to keep configuration of SafeState and complexity of assigned Processes as minimal as possible to ensure successful Function Group State Transition.
**因此，应尽量减少安全状态的配置和所分配进程的复杂性，以确保功能组状态转换的成功。**








### 3.7.7 Executing EM Daemon Without Root Privileges
For some projects it might be possible to run EM daemon itself without superuser privileges and still start Processes with different access rights.
This heavily depends on the OS and project constraints.
On some OS’s (e.g. QNX and Linux) it is possible to configure the effective user and primary group ID of an executable on file system basis.
The process loader will use this information to configure user ID and group ID during load time without further activity by EM.
To enable this the integrator has to set the file ownership and permissions correctly on the executable.
In addition, the mounted file system containing the executable has to support it, i.e. must have the ST_NOSUID flag cleared.
The set-user-ID-on-execution bit and set-group-ID-on-execution bit can be used to tell the loader to change the effective/saved user and group ID of a Process to the owner (user/group) of the executable during load time (see listing 3-4).

Please note that usage of this technique implies that `the real user/group ID’s of started Process` are inherited from parent Process (EM) and the child Process can set its `effective user/group ID`’s to the real user/group ID’s becoming by that on the same privilege level as EM is, which presents a security risk in case EM has some special privileges.
To overcome this issue you can start EM by an executable which itself has unprivileged real user/group ID’s.
This way a malicious Process can only change its effective user/group ID back to an unpriviliged ID.






#### 5.3.3.6 Startup Configuration
A Startup Configuration describes `how a Process shall be started`. 
It is configured within the STARTUP-CONFIG element. 
`A Process can have multiple Startup Configurations`.





















