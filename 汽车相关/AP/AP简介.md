



# What AP AUTOSAR
首先，***Adaptive Platform AUTOSAR***是一个中间件。

## 是中间件

### 没有中间件是怎么开发的？
没有中间件的话，自动驾驶或者域控制器的开发是**Application（Sensing、Perception、Decesion等）直接基于OS（Linux/ROS/RTOS）进行开发**。那么显然，Application直接基于OS，会导致和OS**高度耦合**。

#### 中间件干了什么？
中间件做的就是将Applicaiton与硬件分离
而AP AUTOSAR将Application与OS进行分离，准确来说，是**Runtime For Adaptive Application（ARA）**：
1. `Foundation`（包括了OS，AP AUTOSAR规定，OS需要使用符合POSIX OS标准的OS，例如：Linux、QNX）
2. `Service`


## 是软件平台
AP AUTOSAR的ARA由一系列的`Function Clusters`组成。
上文说到的`Foundation`和`Service`都包含各自的FC

## Service含有的FC：
1. ara::sm
2. ara::nm
3. ara::ucm

## Foundation包含的FC：
1. Operating System Interface（OSI）：
    规范操作系统接口（POSIX PSE51 / C++ STL）
2. ara::exec
    平台生命周期管理
    应用程序生命周期管理
3. ara::com
   负责APP之间（面向服务的）通信
4. ara::rest
    1. 一个framework
    2. 建立APP之间的通信路径
    3. 确保与非AUTOSAR的操作性（ara::rest可以与移动HTTP/JSON Client进行通信）
5. ara::diag
   1. 实现基于ISO 14229-1（UDS）和ISO13400-2（DoIP）的ISO 14229-5（UDSonIP）
    2. 配置基于CP中的DEXT（诊断提取模块）
6. ara::per
    1. 为APP和其他FC提供将信息存储在Adaptive Machine的非易失性存储器中的机制
    2. 提供了访问非易失存储器的标准接口
7. ara::tsyc
    提供时间同步功能（在CP是StbM）
8. ara::iam
    1. 为**Service Interface，AP Fundation FC以及Adaptive Application的请求**提供访问控制framework
    2. 在运行时强制执行对AUTOSAR资源的访问控制
9.  ara::crypto
    用于通用加密操作和安全密钥管理
10. ara::log
    负责记录AP AUTOSAR的日志
11. ara::phm
    对APP进行监督，并在发生故障时采取“补救措施”


## 是一个标准
1. 标准化了开发语言：C++
2. 标准化了软件开发发中使用到的接口（SWS文档）





# Why AP AUTOSAR
**传统的E/E架构**：没有域控制器，没有中央处理器，没有高性能处理器节点
那么结点**不会预留资源**供新增功能使用或消耗

## 硬件原因
由于智能网联汽车的研发过程中，必然涉及到处理视频、地图等大数据，传统的微控制器MCU算力不够用，那么就必须使用高性能的CPU，使用CPU开发域控制器，可以为新增功能预留资源。
此外，虚拟硬件技术也能降低开发成本。


## OS原因
使用高性能处理器的话，CP中的OS就没法适配了。但是使用POSIX OS的话（AP支持），比如Linux，是开源的，通信安全如何保证，诊断怎么做呢？
如果考虑使用`ASIL-D`的操作系统的话，是否得考虑经过功能安全认证的OS（MCOS、QNX）？

## SOA
是否基于SOA（面向服务的架构）开发？
不使用SOA开发的话，增加一个节点，就得更改路由表;增加一个功能的时候，得更改通信矩阵。毕竟SOA是基于更高层的抽象，具有灵活性，便于软件的动态部署。

是否使用面向服务的通信？
不使用的话，当前通信协议栈的带宽和通信效率是否足够？
如果使用DDS和RESTful的话，自己开发的成本有多大？

## 中间件
不使用中间件，则直接调用OS的接口，带来耦合性的问题，和操作系统绑定了，比如应用层的代码和算法就得完全推倒重来。

开发ADAS（***高级驾驶辅助系统：Advanced Driving Assistance System***），如果不使用AP AUTOSAR开发，
使用AP，可能达到一定的安全等级

## APP
运行时如何执行OTA呢？显然需要执行管理和状态管理
要考虑（多重）网络绑定的实现，以及切换

## 小结
AP既是中间件，又是面向服务的架构，增加功能和节点很方便，更换OS或硬件，也是比较方便。
此外，AP还支持多重网络绑定，动态部署，运行时更新，还考虑到了Safety和Security。



# How AP AUTOSAR






