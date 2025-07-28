## 1.1 Architectural Overview
### 1.1.1 Dependencies
Time Synchronization has direct dependencies to ExecutionManager, libVAC, libOsAbstraction, Msr4Base, ApplicationBase, Thread and logAPI.

### 1.1.2 Component Design
This component consists of two main artifacts/modules:
* `timesync daemon executable`: tsyncd
* `client library`: timesync
`The timesync daemon` is responsible for `managing Time Bases`, and for communication with offboard timesync masters and slaves.
时间同步守护进程 “负责 ”管理时基"，并与板外时间同步主站和从站通信。

The client library provides the ara::tsync API and handles communication with the timesync daemon.
AMSR client applications may link against the client library to make use of ara::tsync. 
The component also includes a small example client application tsynce, which periodically prints the current time and status information.







