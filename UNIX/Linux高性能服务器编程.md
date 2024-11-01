## 1.1　TCP/IP协议族体系结构以及主要协议
`TCP/IP协议族`是一个四层协议系统，自底而上分别是`数据链路层、网络层、传输层和应用层`。
每一层完成不同的功能，且通过若干协议来实现，上层协议使用下层协议提供的服务。


## 1.7 socket和TCP/IP协议族的关系
前文提到，`数据链路层`、`网络层`、`传输层`协议是在`内核`中实现的。
因此操作系统需要实现一组`系统调用`，`使得应用程序 能够访问 这些协议提供的服务`。
实现这组系统调用的API（Application Programming Interface，应用程序编程接口）主要有两套：`socket`和XTI。
由socket定义的这一组API提供如下两点功能：
* 一是**将`应用程序数据` 从 `用户缓冲区`中 复制到 T`CP/UDP内核发送缓冲区` ，以交付`内核`来`发送数据`（比如图1-5所示的send函数）**，或者是**从 `内核TCP/UDP接收缓冲区` 中 复制数据到 `用户缓冲区`，以`读取数据`**；
* 二是**应用程序可以通过它们来 修改内核中 `各层协议的`某些`头部信息` 或 `其他数据结构`，从而精细地控制`底层通信`的行为**。
    比如可以通过`setsockopt`函数来设置`IP数据报 在网络上的存活时间`。我们将在第5章详细讨论这一组API。
值得一提的是，socket是一套`通用网络编程接口`，它不但可以访问内核中`TCP/IP`协议栈，而且可以访问`其他网络协议栈（比如X.25协议栈、UNIX本地域协议栈等）`。



## 5.2　创建socket
UNIX/Linux的一个哲学是：所有东西都是文件。**`socket`也不例外，它就是`可读、可写、可控制、可关闭`的文件描述符。**下面的socket系统调用可创建一个socket：
```c
#include＜sys/types.h＞
#include＜sys/socket.h＞
int socket(int domain, int type, int protocol);
```
`domain`参数告诉系统使用哪个底层协议族。
    对TCP/IP协议族而言，该参数应该设置为`PF_INET（Protocol Family of Internet，用于IPv4）` 或 `PF_INET6（用于IPv6）`；
    对于`UNIX本地域协议族`而言，该参数应该设置为`PF_UNIX`。
    关于socket系统调用支持的所有协议族，请读者自己参考其man手册。
`type`参数指定服务类型。
    服务类型主要有`SOCK_STREAM服务（流服务）`和`SOCK_UGRAM（数据报）服务`。
    对TCP/IP协议族而言，其值取SOCK_STREAM表示传输层使用TCP协议，取SOCK_DGRAM表示传输层使用UDP协议。
    值得指出的是，自Linux内核版本2.6.17起，type参数可以接受上述服务类型与下面两个重要的标志`相与`的值：`SOCK_NONBLOCK`和`SOCK_CLOEXEC`。
    它们分别表示将新创建的socket设为`非阻塞的`，以及`用fork调用创建子进程时 在子进程中关闭该socket`。
    在内核版本2.6.17之前的Linux中，文件描述符的这两个属性都需要使用`额外的系统调用`（比如`fcntl`）来设置。
`protocol`参数是在前两个参数构成的协议集合下，再选择一个具体的协议。
    不过这个值通常都是唯一的（前两个参数已经完全决定了它的值）。几乎在所有情况下，我们都应该把它设置为0，表示使用`默认协议`。
    socket系统调用成功时返回一个socket文件描述符，失败则返回-1并设置errno。

## 5.3 命名socket
创建socket时，我们给它指定了`地址族`(PF_INET/PF_INET6/PF_UNIX)，但是并未指定使用该地址族中的哪个`具体socket地址`。
`将一个 socket 与 socket地址 绑定` 称为 `给socket命名`。
**在`服务器程序`中，我们通常要`命名socket`，因为只有命名后客户端才能知道该如何连接它。**
**`客户端`则通常不需要命名socket，而是采用`匿名方式`，即`使用操作系统自动分配的 socket地址`。**
`命名socket`的系统调用是`bind`，其定义如下：
```c
#include＜sys/types.h＞
#include＜sys/socket.h＞
int bind(int sockfd, const struct sockaddr*my_addr, socklen_t addrlen);
```
`bind`将`my_addr`所指的`socket地址`分配给`未命名的 sockfd文件描述符`，`addrlen`参数指出`该socket地址的长度`。
`bind`成功时返回0，失败则返回-1并设置errno。
其中两种常见的errno是EACCES和EADDRINUSE，它们的含义分别是：
* `EACCES`，
    被绑定的地址是受保护的地址，仅`超级用户`能够访问。
    比如普通用户将socket绑定到知名服务端口（端口号为0～1023）上时，bind将返回EACCES错误。
* `EADDRINUSE`，
    被绑定的地址`正在使用中`。
    比如将socket绑定到一个处于`TIME_WAIT`状态的socket地址。



## 6.4 `sendfile`函数
`sendfile`函数在`两个文件描述符之间`直接传递数据（`完全在内核中`操作），从而**避免了`内核缓冲区 和 用户缓冲区之间`的数据拷贝**，效率很高，这被称为零拷贝。
sendfile函数的定义如下：
```c
#include＜sys/sendfile.h＞
ssize_t sendfile(int out_fd, int in_fd, off_t*offset, size_t count);
```
`in_fd`参数是待读出内容的文件描述符，
`out_fd`参数是待写入内容的文件描述符。
`offset`参数指定从`读入文件流`的哪个位置开始读，如果为空，则使用读入文件流默认的起始位置。
`count`参数指定在文件描述符`in_fd`和`out_fd`之间传输的字节数。
sendfile成功时返回传输的字节数，失败则返回-1并设置errno。
该函数的man手册明确指出，`in_fd`必须是一个支持类似`mmap`函数的文件描述符，即它必须指向真实的文件，不能是socket和管道；而out_fd则必须是一个socket。
由此可见，sendfile几乎是专门为`在网络上`传输文件而设计的。

该函数没有为目标文件分配任何用户空间的缓存，也没有执行读取文件的操作，但相比普通的文件传输，同样实现了文件的发送，其效率显然要高得多。


# 第9章 I/O复用
I/O复用使得程序能**同时监听多个文件描述符**，这对提高程序的性能至关重要。
通常，网络程序在下列情况下需要使用I/O复用技术：
* `客户端`程序要`同时处理多个socket`。比如本章将要讨论的非阻塞connect技术。
* `客户端`程序要同时处理`用户输入和网络连接`。比如本章将要讨论的聊天室程序。
* `TCP服务器`要同时处理`监听socket和连接socket`。
    这是`I/O复用使用最多的场合`。后续章节将展示很多这方面的例子。
    服务器要同时处理`TCP请求`和`UDP请求`。比如本章将要讨论的回射服务器。
* `服务器`要同时`监听多个端口`，或者`处理多种服务`。比如本章将要讨论的xinetd服务器。
需要指出的是，I/O复用虽然能同时监听多个文件描述符，但它本身是阻塞的。并且当多个文件描述符同时就绪时，如果不采取额外的措施，程序就只能按顺序依次处理其中的每一个文件描述符，这使得服务器程序看起来像是串行工作的。如果要实现并发，只能使用多进程或多线程等编程手段。
Linux下实现I/O复用的系统调用主要有select、poll和epoll，本章将依次讨论之，然后介绍使用它们的几个实例。

## 9.3　epoll系列系统调用
### 9.3.1　内核事件表
`epoll`是Linux特有的`I/O复用函数`。
它在`实现`和`使用`上与select、poll有很大差异。
首先，epoll使用`一组函数`来完成任务，而不是单个函数。
其次，epoll把用户关心的`文件描述符上的事件`放在`内核里的 一个事件表中`，从而无须像`select`和`poll`那样每次调用都要`重复传入文件描述符集 或 事件集`。
但epoll需要使用一个`额外的文件描述符`，来唯一标识`内核中的这个事件表`。
这个文件描述符使用如下`epoll_create`函数来创建：
```c
#include＜sys/epoll.h＞
int epoll_create(int size);
```
size参数现在并不起作用，只是给内核一个提示，告诉它事件表需要多大。
该函数返回的文件描述符 将用作其他所有epoll系统调用的第一个参数，以指定要访问的`内核事件表`。

下面的函数用来**操作`epoll`的`内核事件表`**：
```c
#include＜sys/epoll.h＞
int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);
```
fd参数是要操作的文件描述符，op参数则指定操作类型。
操作类型有如下3种：
* `EPOLL_CTL_ADD`，往事件表中注册fd上的事件。
* `EPOLL_CTL_MOD`，修改fd上的注册事件。
* `EPOLL_CTL_DEL`，删除fd上的注册事件。
event参数指定事件，它是`epoll_event`结构指针类型。
`epoll_event`的定义如下：
```c
struct epoll_event
{
    __uint32_t events;  /*epoll事件*/
    epoll_data_t data;  /*用户数据*/
};
```

















