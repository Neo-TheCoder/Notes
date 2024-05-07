# UDP port
UdpPort 配置，在 IP-Unicast 情况下用于方法和事件通信。
## 在 SOME/IP 服务发现期间： 在 SD-Offer 消息中发送给客户端（SD-find 应答）或客户端（SD-offer）的端口号。
方法： 这是服务器接受方法调用消息（来自客户端）的目的端口。这是服务器发送方法响应信息（给客户端）的源端口。
事件： 在 IP-Unicast 情况下，这是服务器向订阅客户端发送事件消息的事件源端口。

# TCP port
tcp connection的两端，server和client都可以指定端口，只不过client可以不显式指定port，而是让操作系统（在AP里就是让someipd）分配。












