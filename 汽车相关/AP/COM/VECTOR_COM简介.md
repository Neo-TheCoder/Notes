# Machine相关配置
`Machine Design`中可配置`Service Discovery Config`，
它会绑定到Ethernet Connector，可配置成 单播或者多播（Endpoint）
服务发现配置成多播的话，效率会比较高

# SOME/IP
`SOME/IP Daemon`会把所有的`SD消息`通过`多播的Endpoint`发送出去


# IPC
IPC 的通信方式和DDS有异曲同工之妙，IPC 绑定到了一个Local Host：127.0.0.1。
IPC所有的消息，都会放到 127.0.0.1 这个地址上去。
然后，任何IPC方式的 Service，它的 Required Service 和 Provider Service，都会去监控127.0.0.1这个地址
它们所有消息的收发都是通过这个地址进行的。

由于SOME/IP 的 Client 和 Server 之间的所有通信消息，Notify，Field 和 Method，全部都需要通过 **SOME/IP Daemon 做中转**，Client 跟 Server 之间不直接通讯，比如 Client 发送一个请求，它先到达本端的 SOME/IP Daemon，SOME/IP Daemon 再把 Message 发送到 Server 端的 SOME/IP Daemon。
这种集中式的设计，使得C / S之间解耦了，但是如果SOMEIPD挂了，那么所有的通信都挂了。这种设计的目的是：**基于服务的形式**，不是数据驱动，不是为了C/S之间的大量数据交互（如果有大量数据交互，SOME/IP不够高效，需要回归数据模型DDS），而是服务驱动

## Provide
服务提供端，有对应的Provided State Machine
包含以下状态
1. Initial Offer
    该状态下，系统触发一系列的行动，触发`Offer Message`，这条消息激活一个用于调度准备发送的SD消息的Timer
调度这条消息时，需要考虑发送消息的最小间隔、最大间隔，一旦 Timer 激活，且满足该 Timer 的条件，即**满足Max和 Min 区间内的时间**，系统将通过以太网的 Multicast 地址发送发送 Service Discovery 的 Offer消息，并进入Message Scheduler.
发送第一条消息后，系统会检查另一个参数，即 **Repeat 发送的次数**，系统默认会发送三次。第二次发送与第一次的间隔、Delay 发送的时间等，都由系统控制。
例如，发出第一条消息后，经过200毫秒，系统将发送第二条 Service Discovery 消息，再过200毫秒后发送第三条，发送完三条消息后，表明此状态结束，Init Offer 的状态机结束，系统将进入下一个状态机。

因此，发送 Offer Service 的消息，即 Service Discovery 消息的发送规则、频率和次数都已确定





