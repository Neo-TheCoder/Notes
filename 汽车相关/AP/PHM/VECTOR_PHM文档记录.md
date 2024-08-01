# 1 Introduction
The PHM is a functional cluster of the Adaptive Platform Foundation.
This document describes the functionality, API and configuration of the Microcontroller Open System Architecture (MICROSAR)
Adaptive PHM as specified in [4] as part of the AUTOSAR AP Release 20-11.
The Supervised Entity and Health Channel API is implemented as specified by AUTOSAR AP Release 18-03 [5].
See section 2.1.1 for more details about deviations against AUTOSAR 20-11.
**The PHM supervises `Supervised Entities` and `Health Channels`.**
**It is able to `execute Recovery Actions` in case a failure is detected**.
The expected behavior and corresponding Recovery Actions are defined by the integrator based on the software architecture requirements and can be configured according to
the Manifest Specification [2].


### 1.1.2 Component Design
PHM consists of the PHM Daemon and two static libraries:
* PHM Daemon
The PHM Daemon executes the actual functionality of the PHM.

* PHM library
Adaptive Applications use the PHM library to `establish a connection to the PHM daemon`.
The PHM library consists of two parts, the SupervisedEntity and the HealthChannel.
- **The SupervisedEntity enables the user to `report Checkpoints of Supervised Entities`.**
- **The HealthChannel enables the user to `report Health Statuses of Health Channels`. **
**Based on reported Checkpoints and Health Statuses the PHM `executes supervisions according to the configured requirements`.**
For more information see 4.1.

* WatchdogClient library
The WatchdogClient library enables `execution of watchdog related actions` and also `monitoring of the PHM Daemon process itself`.
For more information see 4.2.

* Phm Base Services library
Adaptive Applications use the Phm Base Services library to `establish an ipc connection to the PHM daemon`.
For more information see 4.3.

* StateManagementClient library
The StateManagementClient library enables `monitoring and state adaption of an Adaptive Application.`
For more information see 4.4.
提供接口给SM，用于监控和APP的状态调整




# 2 Functional Description
## 2.1 Features
The PHM covers features specified in [1] and [4].
The following tables give an overview which features are supported and which are not. Details about each supported feature are given in the next sub-chapters.

`Supported Features`
- Alive Supervision
- Deadline Supervision
- Logical Supervision

- Local Supervision
- Global Supervision
- Health Channel Supervision
- Recovery Notifications

## 2.2 Supervision
Applications are able to report reached Checkpoints to the PHM using a standardized AUTOSAR C++ interface, see chapter 4.

### 2.2.1 Alive Supervision
`Alive Supervision` allows the supervision of `periodically` occurring Checkpoints.
This is done by `counting the occurrences of a specific Checkpoint during a configurable time span`.
Additionally `a minimum and maximum margin` is configurable, see chapter 5.4.6.
If the number of reported Checkpoint occurrences during the timespan is lower or higher than the expected number (with consideration of the margins) than a failure is detected.
也就是检查一段时间内的Chechpoint的次数

`Info`
PHM has no way of directly detecting restarts of supervised applications.
If restarts must be detected by PHM, there has to be an appropriate Alive Supervision configured, which is able to detect a restart of the supervised application.
(从alive supervision的结果侧面得知应用的重启)

All timers of PHM are based on the PHM reference clock, which in the following example has a cycle time of 5 ms.
The clock uses the `WatchdogAliveNotificationCycleTime` configuration parameter as reference cycle time.
`An Alive Supervision timer can be started during the PHM init phase.`
If this is the case the supervision timer starts with the reference timer. 在这种情况下，`监控计时器`与`参考计时器`同时启动。
But if the Alive Supervision timer is started during running phase it could happen that the timer does not start with the reference timer and runs acyclically.
2-1 shows an example of a supervised application which reports two Checkpoints for an Alive Supervision within the accepted time to the PHM.
In this example the Alive Supervision is configured with the following parameters:
* Reference Cycle Time: 5 ms
* AliveReferenceCycle: 20 ms
* ExpectedAliveIndications: 2
* MinMargin/MaxMargin :0





















