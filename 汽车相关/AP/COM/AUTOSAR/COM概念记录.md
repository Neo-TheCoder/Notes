# UDP port
UdpPort 配置，在 IP-Unicast 情况下用于方法和事件通信。
## 在 SOME/IP 服务发现期间： 在 SD-Offer 消息中发送给客户端（SD-find 应答）或客户端（SD-offer）的端口号。
方法： 这是服务器接受方法调用消息（来自客户端）的目的端口。这是服务器发送方法响应信息（给客户端）的源端口。
事件： 在 IP-Unicast 情况下，这是服务器向订阅客户端发送事件消息的事件源端口。

# TCP port
tcp connection的两端，server和client都可以指定端口，只不过client可以不显式指定port，而是让操作系统（在AP里就是让someipd）分配。

# socket
可以配置：`< 源IP地址, 目标IP地址, 源端口号, 目标端口号, 传输层协议 >`的五元组
不显示配置的话，操作系统会分配默认的
IP可以配置为`INADDR_ANY`

# SOME/IP服务发现过程
服务发现（SD）主要有两大作用：
1. 实现服务的发布和订阅（service id, instance id），Find Service和Offer Service可以塞在一个包里（塞在Entries Array字段里面）
2. 管理某个服务实例（包括服务端和客户端）是否需要运行或者是否能发送报文

## 整体流程
1. server端广播消息：提供哪些服务
2. client端收到通知后，选择订阅哪些服务
3. server端向client端发送订阅的数据（无人订阅则无需发送，节省资源）

### 服务发现具体行为
当一人进程需要访问另一个进程的服务时，它会向本地的SOMEMP服务代理发送一个请求，服务代理会在本地的服务目录中查找并返回符合条件的服务列表。然后，请求方可以通过发送基于UDP的点对点消息来与服务提供方进行通信，从而实现进程间通信。
#### 整体过程:
当服务请求者发起服务请求时，服务代理会先在本地维护的服务目录中查找是否有符合条件的服务提供者。
1. 如果本地服务目录中存在对应的服务提供者，则服务请求者可以直接通过UDP点对点通信方式向服务提供者发送服务请求和响应消息，完成服务调用的过程。
2. 如果本地服务目录中没有找到对应的服务提供者，则服务代理会通过UDP广播方式向网络中的其他节点发送服务请求，以便寻找服务提供者。
    在这种情况下，服务请求者需要**等待一定时间**，等待其他节点的响应，如果在规定的时间内没有收到响应，则服务请求者会抛出服务不可用的异常。
    如果找到了服务提供者，则服务请求者会通过UDP点对点通信方式向服务提供者发送服务请求和响应消息，完成服务调用的过程.

### 服务发现的流程主要分为两种
1. `服务的寻找与发布流程`，主要用于Method通信。
2. `服务的寻找、发布与订阅流程`，主要用于Event通信.

#### 服务发布与Method通信
服务的发布流程即某个服务上线后，可以在运行时动态的通知到大家，然后需要使用该服务的客户遍可以请求该服务了。
someip协议对于服务的发布规定使用ofer报文来表示。服务的发布，往往也伴随着服务的寻找(或者说发现)流程，这里的服务的寻找流程主要是为了催促能提供服务的一方，尽快提供服务，在someip协议中规定使用find报文来表示。

有了上述的服务寻找与发现流程后，就可以method通信.
1. Client端需要某个服务，那么就可以通过find报文广播出去：Client一般不知道服务到底是谁能提供，所以广发英雄帖(组播)，谁能提供服务Client不需要关心(该部分由SD管理)。

2. Server端收到了需要服务的英雄帖后，对比一下自己是否能提供该服务
如果自己提供不了，就忽略该find报文
如果自己能提供，便发出offer报文，告诉Client有该服务(该部分由SD管理)。
这里的Offer同时还会携带自己业务服务所在的IP和port，以便Client能知道去哪里请求服务。

3. Client端收到offer报文后，就知道server在提供自己需要的服务了:
就可以按照自己的需求去请求该服务了(即`Request， Response流程`)。
这里补充一点:在实际代码实现中，Client在收到Offer后，会由Cient SD先通知Client，Client才会启动去Request，否则Client没有收到Offer，也不知道去哪个地址请求服务。
**如果是TCP通信，在Client SD通知Client后，会由Client发起TCP三次握手建立连接。**

##### 半周期回复Offer:
半周期回复Offer设定:
在周期发送offer阶段中，如果是1/2周期前收到的find，会回复一条`单播`的offer;
如果是1/2周期后收到find，会回复一条`组播`的offer。
这里回复单播就是单独告诉发送find的Client端;
而**发送组播，是为了节约网络带宽**。
stop offer用于主动停止服务.

#### 服务订阅与Event通信
服务的订阅也需要服务发布流程，但是多了订阅的步骤，服务的订阅流程服务的订阅由Client SD在收到Offer确认有人能提供服务后主动发起。
someip中规定服务的订阅需要两步：`subscribe报文`和`subscribe ack报文`

与Method流程相同的是在TCP建链及之前的部分。
Clien的SD主动发起订阅，Server的SD收到订阅后，通知业务服务，可以发送event了。
然后Server的SD立即回复一个ACK消息，让Clent业务做好接收的准备，然后Server根据自身需要开始发送event

##### 多个client订阅一个server的服务是否可以使用组播方式?
###### 订阅服务的组播、单播:
`threshold`: 指定何时使用多播以及何时使用单播发送通知事件。
此值必须设置为非负数。如果设置为零，则eventoroup的所有事件将通过单播发送，否则，只要订阅数低干阈值事件将通过单播发送；如果订阅数大于或等于阈值，将多播发送，这意味着，阈值为1将导致所有事件正在通过多播发送。默认值为0。

###### Subscribe也有相应的stop机制:
Subscribe也有相应的stop机制，也可以发送`Stop Subscribe`和超时，作用就是停止订阅，让服务端不再给取消订阅的cient发送event.
一个服务上线了不代表event就准备好了，如果event没有准备好，就有人来订阅了，那么会回复subscribe nack。

##### SD保活作用:
offer的作用: 告诉大家当前的服务可用，服务的有效时间为TTL(TTL时间之后不可用)。
那么要让服务一直可用，就要在TTL之前一直发送offer，来保持这个server的服务对这个client一直有效。
也就是说，server发送了一次offer之后，如果在TTL时间内不再发送offer，就认为服务无效了。需要重新发送offer。

#### SD状态机
1. Down Phase
2. Initial Wait Phase
3. Repetition Phase
4. Main Phase

2, 3, 4统称为Ready阶段

2, 3, 4状态的内部会含有一个`计时器`，大多数情况会**定时进入下一个阶段**。
当进入到`Main Phase`后，除非服务主动下线，否则一般不会再退出。

服务的`Down Phase`表示`服务不可用`或`未注册`。
服务进入Down状态通常有以下两种情况:
1. 服务未注册:
当服务未成功注册时，服务的状态会被设置为Down。未注册的服务无法被客户端访问。

2. 服务异常:
当服务在运行过程中出现异常，例如网络连接中断、服务崩溃等，服务的状态也会被设置为Down。
在此情况下，服务会被认为是不可用的，并将从服务列表中移除，直到服务重新注册。



##### Server SD
1. Down Phase
当服务准备完毕后，进入`Initial Wait Phase`

2. Initial Wait Phase
`INITIAL_DELAY`时间（是`INITIAL_DELAY_MIN`和`INITIAL_DELAY_MAX`之间的随机值）
当server收到find service报文，服务端将其忽略，不做处理
当定时器超时后（经过`INITIAL_DELAY`时间，或者如果设置了`INITIAL_DELAY_MAX` == 0，则直接进入`MAIN_PHASE`），才发送第一帧`offer service`，进入下一阶段`Repetition phase`
如果此时服务不可用，则进入`Down Phase`

3. Repetition Phase
重复发送`Offer Service`，重复次数由`REPETITIONS_MAX`决定，发送间隔以`REPETITIONS_BASE_DELAY`为基本时间，每发送一次，发送间隔为上一次的`2倍`
收到某client发来的`Find Service`（也是通过广播发来的，但src IP是单播的，所以指示了来源），不影响当前阶段的发送计数和计时，延迟一定时间（`REQUEST_RESPONSE_DELAY`）后，单独发送单播`OfferService`给服务请求端（使用单播地址，port仍然是服务发现专用的port，例如30490）
如果此时服务不可用，则进入`Down Phase`，并发送`StopOfferService`通知所有客户端

如果收到`Subscribe Event Group`，发送单播Ack/Nack，启动此订阅Entry的TTL计时器
如果收到`Stop Subscribe Event Group`，停止此订阅Entry的TTL计时器

发送完`REPETITIONS_BASE_DELAY`次offer后，进入下一阶段`MAIN_PHASE`

4. Main Phase
周期性发送Offer Service，以`CYCLIC_OFFER_DELAY`为周期
收到某客户端的find service，不影响当前阶段的发送计数，延迟`REQUEST_RESPONSE_DELAY`时间，发送单播`OfferService`给服务请求端
如果此时服务不可用，则进入`Down Phase`，并发送`StopOfferService`通知所有客户端

如果收到`Subscribe Event Group`，发送单播Ack/Nack，启动此订阅Entry的TTL计时器
如果收到`Stop Subscribe Event Group`，停止此订阅Entry的TTL计时器


#### 上电初始化:
如果上电后通信不通或者服务没有上线，那么之后就不用发offer等信息，就进入到Not Ready状态待命。
当状态机在Not Ready中时，如果这时检测到通信状态发生改变或者服务状态发生改变，那么就尝试去判断通信是否通了且服务是否上线`if(status == up_and configured && service-status == up )`
如果上述条件都满足，那么状态机会从Not Ready切换到Ready。
反之，如果不满足上述if条件，那么就要从Ready状态机回到Not Ready状态


##### Client SD
1. Down Phase
服务未被应用请求
收到OfferService，存储当前服务实例状态，启动TTL计时，此时服务若被应用请求，直接进入`MAIN_PHASE`

2. Initial Wait Phase
等待`INITIAL_DELAY`时间
client收到新的offer service，取消计时器，直接进入`MAIN_PHASE`。如果设置了`INITIAL_DELAY_MAX` == 0，则直接进入`MAIN_PHASE`
计时器超时后，发送第一个`Find Service`，进入`Repetition Phase`
如果服务请求被释放，进入`Down Phase`

3. Repetition Phase
    1. 重复发送Find Serviee，重复次数由`REPETITIONS MAX`决定，发送间隔以`REPETITIONS BASE DELAY`为基时间，每发送一次，间隔加倍
    2. 收到`OfferService`，停止发送计数和计时，立即进入`Main Phase`，触发发送SubscribeEventGroup(延迟一定时间)
    3. 收到`StopOffer Service`，取消当前计数，立即进入`Main Phase`

4. Main Phase
1. `收到Offer Service`，如果是事件服务触发发送`Subscribe Eventgroup`(延迟一定时间)
2. 如果收到`StopOfferService`，则停止所有计时器:
    若有订阅，则发送`StopSubscribeEventgreup`

##### 服务端与client区别:
Client状态机与Server状态机最大的不同就是一旦收到Offer后，会直接进入Main阶段。
另一个区别是Server在Main中会不断周期发送Offer,但是Client在Main中不需要发送Find(因为Find的作用的触发激活Server上线，对端不上线的话，不断触发没有太大意义，只在Repetition Phase里发就行了，也浪费总线带宽；而Offer的作用是保活，是要不断发送才能刷新TTL)。
这也是为啥**Client为什么一收到offer后，就不会再发送Find了**
每个服务都有一个SD状态机，用于处理服务的SD过程。
服务的SD状态机向括多个状态，服务在不同的状态之间进行状态转换，以完成服务的SD过程。


在AUTOSAR AP中，`eventgroups`还有独立于service的TTL，准确来说是`Subscribe Eventgroup Entry`（是SOME/IP协议栈的一部分）。




# MulticastThreshold

## AUTOSAR
### multicast Threshold
`SomeipProvidedEventGroup`具有属性：`eventMulticast UdpPort`
Specifies the number of subscribed clients that trigger the server to change the transmission of events to multicast.

Example:
If configured to 0 only unicast will be used.
If configured to 1 the first client will be already served by multicast.
If configured to 2 the first client will be served with unicast and as soon as the 2nd client arrives both will be served by multicast.
This does not influence the handling of initial events, which are served using unicast only.

当multicast Threshold设置成x（x >= 1），则前 x-1 个client使用单播，当第x个client到达时，全体使用多播

PS：在只有一个客户端时使用单播，可以减少网络带宽的占用；而在多个客户端需要接收相同数据时，使用多播可以减少数据传输的重复和网络负担


### eventMulticast UdpPort
UdpPort configuration that is used for Event communication in the IP-Multicast case.
During SOME/IP Service Discovery: Send in the SD-SubscribeEventGroupAck Message to client (answer to SD-SubscribeEventGroup).
Event: This is the destination-port where the server sends the multicast event messages if the multicastThreshold is exceeded.


### 其他相关说明

[SWS_CM_80064]{DRAFT} Transport protocol for sending of an event message
The event message shall be transmitted using UDP if the threshold defined by the multicastThreshold attribute of the SomeipProvidedEventGroup that is aggregated by the ProvidedSomeipServiceInstance in the role eventGroup in the Manifest has been reached (see [PRS_SOMEIPSD_00134]).


```json
"eventgroups" : [
    {
    "id" : 12074,
    "event_multicast_threshold" : 0
    }
]
```
每个event可以设置`event_multicast_threshold`


***

## VECTOR
In SOME/IP daemon an event is only allowed to be referenced by multiple eventgroups if all the referencing eventgroups use unicast communication (MulticastThreshold = 0).















