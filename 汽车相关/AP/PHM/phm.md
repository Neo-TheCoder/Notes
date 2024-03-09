# 包含的几大模块
## 提供的库、接口、以及基本功能
PHM为`Adaptive Application`提供了一些lib：`daemon client`。让AA报告App自身的状态（通过IPC方式），反馈给PHM Daemon。

PHM还提供了`watchdog client library`：
PHM**将watchdog的权限放给了外部**，让用户决定行动，而不是PHM自己。

PHM还提供了`PHM Base Service Library`：
让应用能够获取当前的状态（包括当前的监控状态）。
可以通过注册、注销 去启用或禁用监控功能，以及控制是否接收`State Manager`的状态消息。

此外，PHM还和`Execution Manager`交互，因为`PHM Daemon`会通过`RecoveryActionClient`，获取当前EM的所有进程的状态（该接口显然基于EM的对外接口）。当某个进程崩溃后，会通过这个接口获取相关信息。
同时也要向EM报告`PHM Daemon`的状态（通过`ApplicationClient`的库，也是由EM提供）。

`Adaptive Application`定期通过调用库，报告其`Checkpoint`状态给`PHM Daemon`。`PHM Daemon`通过连接接收到状态后，会收集并管理这些状态。

## base_connection_manager
`SM`作为client通过该接口请求`PHM Daemon`的所有状态，以及获得`Local PHM Notification`。
`SM`对于PHM发来的通知是可以关闭的。
只有`State Manager`需要收集PHM的一些状态变化的信息，如何使用这些信息，取决于用户的 State Manager的定义，**根据用户的策略去执行相应的动作**。
然后是`watchdog connection manager`，前面有提到，当PHM发生了问题，它会把 Watchdog 的权限放给外部应用，由外部的应用来决定。PHM自身并不做执行策略的主观性判断，因为在不同的项目里面，主观性判断会有所不同。它只是将状态传给外部的APP，由APP决定要执行的策略。
例如，如果Crash了，平台是否要Restart，这是由APP来触发的。`watchdog_connection_manager`是作为IPC的 Client 存在的。而外部 APP，才是IPC的 Server。
**PHM 将会把当前跟 Watchdog相关的一些状态，主动Report给外部APP，由用户决定执行什么操作**。然后是Model，我们要去做PHM的Model Design，然后生成配置文件。PHM 通过 Configuration 这个模块，去解析模型生成的配置文件，比如说在模型中定义的 Checkpoint，Action List，逻辑表达式以及Rule的定义等。

## Time Manager和Supervision State Manager
属于PHM的管理类，当事件发生时，它们管理相关内容并与之交互。

# 监控类型
## Alive Supervision
监控`周期性上报的Checkpoint`的状态

## Deadline Supervision
监控从`Source Checkpoint`到`Target Checkpoint`所耗费的事件

## Logical Supervision
监控多个Checkpoints是否按照`状态迁移的顺序进行切换`

## Local Supervision
组合以上三种Supervision
而Global Supervision是Local Supervision的集合

## Health Channel Supervision
每个`Health Channel`，由多个`Health`状态组成，Application需要报告当前`Channel ID`的`Health`状态给`PHM Daemon`；
`Health Channel`的监控结果可以由配置的规则推导，用于`Arbitration（仲裁）`的输入（？？？）

Alive使用最为普遍，大多数项目的监控中，都是使用Alive，其次是Deadline监控。




