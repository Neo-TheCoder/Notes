# 1
语速要慢，忘记了可以多想一会儿、慢条斯理
把简历先整一整，可能需要黑体加粗一些关键字

## 自我介绍
比较熟悉C++开发这块，熟悉一些常用的特性

### 项目介绍
#### 简明扼要地介绍下AP各个模块，`拎出关键字`
EM
进程启动关闭、资源管理（CPU、内存、核）
COM（基于service，多重绑定）
SM
PER
PHM（两种模式，一种是基于频率，一种是deadline）
LOG（syslog，console / file / network）


## method是如何实现的？
client端发送请求，server端把请求存储下来（`<id, request数据>`），可以单线程/多线程地取出请求，调用method的回调，拿到结果之后把结果发回
AP设置了三种模式，同步地处理method回调（可能是为了避免上下文切换带来的开销） / 单线程处理 / 多线程处理

## fastdds如何实现rpc机制？

## arxml和idl怎么映射的？
event和topic映射
method和service映射（当然最后还是映射到fastdds里的topic了）
    也就是说，ROS2的请求应答模式还是通过fastdds topic来实现的，一个method映射成两个fastdds topic（一个写请求、一个是读应答）



## 遇到的问题
**要强调`有价值的`开发问题**
总是需要去阅读源码（ROS2，mcap插件的源码）


## 有些东西忘记了，还要仔细看下，熟悉一下
比如实际上按照每分钟/每秒钟所做的mcap文件生成是生成到内存中（`/tmp`），真正的落盘操作是trigger来了之后进行的


## 想办法增加一些闪光点
### 可能要看下fastdds的实现、以及RTPS协议






