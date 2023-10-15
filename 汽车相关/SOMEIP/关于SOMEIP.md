# 汽车软件通信中间件SOME/IP简述
## 1 SOME/IP是中间件吗
准确来说是通信协议，简单来说也可以呃称为通信中间件
全称是**Scalable service-Oriented MiddlewarE over IP**
即：基于IP协议的面向服务的可扩展性通信中间件协议
可以看出有三个要素：
1. 面向服务SOA
2. 基于IP协议之上的通信协议
3. 中间件

## 2 SOME/IP的作用
都和通信相关：
1. 服务发现（Service Discovery）
2. 远程服务调用（RPC，Remote Producer Call）
3. 读写进程信息（Getter & Setter）

## 3 SOME/IP和CAN的区别
CAN是传统的汽车软件通信协议，CAN FD是其扩展
二者的主要区别在于：
1. 通信负荷的差异：即承载的字节数，8Byte -- 1500Byte
2. 通信速率的差异   1Mb/s -- 1000Mb/s
3. CAN是面向服务的，SOME/IP是面向服务的
4. CAN是基于信号在双绞线中传输信号  --  SOME/IP是在车载以太网中传输信号
SOME/IP实际上是位于车载以太网网络协议中的应用层，是基于TCP/UDP协议之上的应用
GENIVI组织针对SOME/IP标准实现了开源vsomeip方案
SOME/IP 协议一般指代具体 SOME/IP、SOME/IP-SD、SOME/IP-TP 3种。

5. 消息格式
一个完整的 SOME/IP 消息，包含以下内容：
MessageID（Event ID / Method ID） -- 32bit
Length -- 32bit -- 消息长度
Request ID（ Client ID & Session ID）-- 32bit
Protocal Version -- 协议版本号
Interface Version -- 接口版本号
Message Type -- 消息类型
Return Code -- 返回编码
Payload -- 数据负载（可变）：SOME/IP底层可以基于TCP或者UDP，使得Payload的容量不一样：如果是UDP，则payload限制在1400B的容量，如果是TCP，数据分段传输，那么SOME/IP可以实现更大容量的传输
PS：所有SOME/IP Header内容采用大端传输，而Payload中的数据存放顺序通过配置设置

6. SOME/IP支持的数据结构类型
基础的uint、int...，对于结构体数据会在内存中按照顺序依次存放

7. SOME/IP消息通信类型
1. R & R（Request & Response）
2. F & F (Fire & Forget)：只请求不应答
3. Notification Event-- 发布订阅机制
但 Notification 分3种情况：
Cyclic Update 周期性的发送相关 value 的变化
Update On Change 如果 value 发生变化，则向外推送
Epsilon Change 如果 value 的值大于相应的 epsilon值，那么对外推送消息
5. Fields
针对的是一种status，并且有一个合法值
一个Fields操作应当包含Setter、Getter与Notification的结合。
执行Setter：客户端发起请求，并在请求消息中的payload设置确定值，并在服务端应答消息中的payload要同步返回这个值
执行Getter：客户端发起请求，请求消息中包含一个空的payload，服务端收到请求消息后将field的值填充到应答消息中的payload

8. SOME/IP-SD
服务发现主要功能：
探测服务、向外提供服务
SD的消息结构是在SOME/IP消息体之上附加了一些参数：比如Entries Array，用于标记Service或者Event Group

服务发现的流程
谈论 Service Discovery 之前，要搞清楚几个基本概念
**Service**：指代 functional entities，代表的是一项功能.
**find**：是一项操作，用于标识 available 状态的 service 及它的们的位置
**event**：Service Discovery 传送的 Message 被称为 event
**Eventgroups**：event 的集合

ervice中有许多细节。
Service有2类：Server Service和Client Service。 Service有2种状态: Down和Available。
一个ECU需要处理两种Service:ServerService和ClientService。
也就是说，一个 ECU 可以对外提供服务,这个时候由它的 Server Service 模块负责，也可以对外请求服务，这个时候由 Client Service 提供。
实际的流程非常复杂，一个 Service 从 Down 状态到 Available 状态之间的切换有一系列的状态阶段转换。

# SOME/IP和DDS的差异
DDS是面向数据的，通信容量在理论上是无限，并且还使用了共享内存，QOS丰富。
