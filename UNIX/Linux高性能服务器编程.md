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


## 3.8 带外数据
有些`传输层协议`具有`带外（Out Of Band，OOB）数据`的概念，用于迅速通告对方本端发生的重要事件。
因此，**带外数据比普通数据（也称为带内数据）有`更高的优先级`，它应该`总是立即被发送`，而`不论 发送缓冲区中 是否有排队等待发送的普通数据`**。
带外数据的传输可以使用一条`独立的 传输层连接`，也可以映射到传输普通数据的连接中。
实际应用中，带外数据的使用很少见，已知的仅有telnet、ftp等远程非活跃程序。
UDP没有实现带外数据传输，TCP也没有真正的带外数据。
不过**TCP利用其头部中的`紧急指针标志`和`紧急指针`两个字段，给应用程序提供了一种紧急方式**。
TCP的紧急方式`利用 传输普通数据的 连接`来传输紧急数据。
这种紧急数据的含义和带外数据类似，因此后文也将TCP紧急数据称为带外数据。
我们先来介绍`TCP发送带外数据的过程`。
假设一个进程已经往某个TCP连接的发送缓冲区中写入了`N字节的 普通数据`，并等待其发送。
在数据`被发送前`，该进程又向这个连接写入了`3字节的带外数据“abc”`。
此时，`待发送的 TCP报文段的 头部`将被设置`URG标志`，并且`紧急指针`被设置为`指向最后一个带外数据的下一字节（进一步 减去 当前TCP报文段的序号值 得到其头部中的 紧急偏移值）`。
`发送端一次发送的多字节的带外数据`中只有`最后一字节`被当作带外数据（字母c），而其他数据（字母a和b）被当成了普通数据。（注意，这只是一种协议规定）
如果TCP模块以`多个TCP报文段`来发送图3-10所示TCP发送缓冲区中的内容，则`每个TCP报文段`都将设置`URG标志`，并且它们的`紧急指针`指向同一个位置（数据流中带外数据的下一个位置），但只有一个TCP报文段真正携带带外数据。

现在考虑`TCP接收带外数据的过程`。
TCP接收端只有在接收到紧急指针标志时才检查紧急指针，然后根据紧急指针所指的位置确定带外数据的位置，并将它读入一个特殊的缓存中。这个缓存只有`1字节`，称为`带外缓存`。
如果上层应用程序没有及时将带外数据从带外缓存中读出，则后续的带外数据（如果有的话）将覆盖它。

前面讨论的带外数据的接收过程是TCP模块接收带外数据的默认方式。
如果我们给TCP连接设置了`SO_OOBINLINE`选项，则带外数据将和普通数据一样被TCP模块存放在`TCP接收缓冲区`中。
此时应用程序需要像读取普通数据一样来读取带外数据。
那么这种情况下如何区分带外数据和普通数据呢？显然，紧急指针可以用来指出带外数据的位置，socket编程接口也提供了系统调用来识别带外数据（见第5章）。



## 5.2　创建socket
UNIX/Linux的一个哲学是：所有东西都是文件。
**`socket`也不例外，它就是`可读、可写、可控制、可关闭`的文件描述符。**下面的socket系统调用可创建一个socket：
```c
#include＜sys/types.h>
#include＜sys/socket.h>
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
创建socket时，我们给它指定了`地址族(PF_INET/PF_INET6/PF_UNIX)`，但是并未指定使用该地址族中的哪个`具体socket地址`。
`将一个 socket 与 socket地址 绑定` 称为 `给socket命名`。
**在`服务器程序`中，我们通常要`命名socket`，因为只有命名后，客户端才能知道该如何连接它。**
**`客户端`则通常不需要命名socket，而是采用`匿名方式`，即`使用操作系统自动分配的 socket地址`。**
`命名socket`的系统调用是`bind`，其定义如下：
```c
#include＜sys/types.h>
#include＜sys/socket.h>

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


## 5.4 监听socket
socket被命名之后，还不能马上接受客户连接，我们需要使用如下`系统调用`来创建一个`监听队列`以**存放`待处理`的`客户连接`**：
（如果有客户端连接请求到达，但服务端尚未调用accept来接受连接，这些连接请求就会被放入监听队列中等待处理）
```c
#include＜sys/socket.h>
int listen(int sockfd, int backlog);
```
`sockfd参数`
    指定`被监听的socket`。
`backlog`参数
    提示`内核监听队列的最大长度`。
    监听队列的长度如果超过`backlog`，服务器将不受理新的客户连接，客户端也将收到`ECONNREFUSED`错误信息。
    在内核版本2.2之前的Linux中，backlog参数是指所有处于`半连接状态（SYN_RCVD）`和`完全连接状态（ESTABLISHED）`的socket的上限。
    但自内核版本2.2之后，它只表示处于完全连接状态的socket的上限，处于半连接状态的socket的上限则由`/proc/sys/net/ipv4/tcp_max_syn_backlog内核参数`定义。
    backlog参数的典型值是5。
    listen成功时返回0，失败则返回-1并设置errno。

下面我们编写一个服务器程序，如代码清单5-3所示，以研究backlog参数对listen系统调用的实际影响。
```c
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<signal.h>
#include<unistd.h>
#include<stdlib.h>
#include<assert.h>
#include<stdio.h>
#include<string.h>

static bool stop = false;
/*SIGTERM信号的处理函数，触发时结束主程序中的循环*/
static void handle_term(int sig)
{
    stop = true;
}

int main(int argc, char*argv[])
{
    signal(SIGTERM, handle_term);
    if(argc＜= 3)
    {
        printf("usage:%s ip_address port_number backlog\n", basename(argv[0]));
        return 1;
    }
    const char*ip = argv[1];
    int port = atoi(argv[2]);
    int backlog = atoi(argv[3]);

    int sock = socket(PF_INET, SOCK_STREAM, 0); // TCP
    assert(sock >= 0);

    /*创建一个IPv4 socket地址*/
    struct sockaddr_in address;
    bzero(＆address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, ＆address.sin_addr);
    address.sin_port = htons(port);
    int ret = bind(sock, (struct sockaddr*)＆address, sizeof(address)); // 给socket命名
    assert(ret != -1);
    ret = listen(sock, backlog);
    assert(ret != -1);
    
    /*  ！！！循环等待连接，直到有SIGTERM信号将它中断 */
    while(!stop)
    {
        sleep(1);
    }
    /*关闭socket，见后文*/
    close(sock);
    return 0;
}
```
该服务器程序（名为testlisten）接收3个参数：`IP地址`、`端口号`和`backlog值`。
我们在Kongming20上运行该服务器程序，并在ernest-laptop上多次执行telnet命令来连接该服务器程序。
同时，每使用`telnet命令`建立一个连接，就执行一次`netstat命令`来查看服务器上连接的状态。
具体操作过程如下：
```sh
$./testlisten 192.168.1.109 12345 5     # 监听12345端口，给backlog传递典型值5
$telnet 192.168.1.109 12345     # 多次执行之    连接到<ip, port>的主机
$netstat-nt | grep 12345      # 多次执行之        查看当前系统中所有处于TCP连接状态的端口
```
代码清单5-4是`netstat命令`某次输出的内容，它显示了这一时刻listen监听队列的内容。
代码清单5-4　listen监听队列的内容
```sh
Proto Recv-Q Send-Q Local Address Foreign Address Statetcp
tcp 0 0 192.168.1.109:12345 192.168.1.108:2240 SYN_RECV
tcp 0 0 192.168.1.109:12345 192.168.1.108:2228 SYN_RECV [1]
tcp 0 0 192.168.1.109:12345 192.168.1.108:2230 SYN_RECV
tcp 0 0 192.168.1.109:12345 192.168.1.108:2238 SYN_RECV
tcp 0 0 192.168.1.109:12345 192.168.1.108:2236 SYN_RECV
tcp 0 0 192.168.1.109:12345 192.168.1.108:2217 ESTABLISHED
tcp 0 0 192.168.1.109:12345 192.168.1.108:2226 ESTABLISHED
tcp 0 0 192.168.1.109:12345 192.168.1.108:2224 ESTABLISHED
tcp 0 0 192.168.1.109:12345 192.168.1.108:2212 ESTABLISHED
tcp 0 0 192.168.1.109:12345 192.168.1.108:2220 ESTABLISHED
tcp 0 0 192.168.1.109:12345 192.168.1.108:2222 ESTABLISHED
```
可见，在监听队列中，处于`ESTABLISHED`状态的连接只有6个（backlog值加1），其他的连接都处于`SYN_RCVD`状态。
我们改变服务器程序的第3个参数并重新运行之，能发现同样的规律，即完整连接最多有（`backlog + 1`）个。在不同的系统上，运行结果会有些差别，不过监听队列中完整连接的上限通常比backlog值略大。


## 5.5　接受连接
下面的系统调用从listen监听队列中接受一个连接：
```c
#include＜sys/types.h>
#include＜sys/socket.h>
int accept(int sockfd, struct sockaddr*addr, socklen_t*addrlen);
```
`sockfd参数`
    是`执行过listen系统调用` 的 `监听socket` [1] 。
`addr参数`
    用来获取 `被接受连接`的 `远端socket地址`，该socket地址的长度由addrlen参数指出。
    accept成功时返回一个`新的连接socket`，该socket唯一地标识了被接受的这个连接，服务器可通过读写该socket来与被接受连接对应的客户端通信。
    accept失败时返回-1并设置errno。

现在考虑如下情况：如果监听队列中处于ESTABLISHED状态的连接对应的客户端出现网络异常（比如掉线），或者提前退出，那么服务器对这个连接执行的accept调用是否成功？
我们编写一个简单的服务器程序来测试之，如代码清单5-5所示。

代码清单5-5　接受一个异常的连接
```c
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<assert.h>
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<errno.h>
#include<string.h>

int main(int argc, char*argv[])
{
    if(argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char*ip = argv[1];
    int port = atoi(argv[2]);
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int sock = socket(PF_INET, SOCK_STREAM, 0);
    assert(sock >= 0);
    int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
    assert(ret != -1);
    ret = listen(sock, 5);
    assert(ret != -1);
    /*暂停20秒以等待客户端连接和相关操作（掉线或者退出）完成*/
    sleep(20);
    struct sockaddr_in client;
    socklen_t client_addrlength = sizeof(client);
    int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);
    if(connfd < 0)
    {
        printf("errno is:%d\n", errno);
    }
    else
    {
        /*接受连接成功则打印出客户端的IP地址和端口号*/
        char remote[INET_ADDRSTRLEN];
        printf("connected with ip:%s and port:%d\n", inet_ntop(AF_INET, &client.sin_addr, remote, INET_ADDRSTRLEN), ntohs(client.sin_port));
        close(connfd);
    }
    close(sock);
    return 0;
}
```
* 启动telnet客户端程序后，立即断开该客户端的网络连接（建立和断开连接的过程要在服务器启动后20秒内完成）。结果发现accept调用能够正常返回，
    连接socket的状态为：`ESTABLISHED`

* 在建立连接后立即退出客户端程序。这次accept调用同样正常返回
    连接socket的状态为：`CLOSE_WAIT`

由此可见，**`accept`只是从`监听队列`中取出连接，而不论连接处于何种状态（如上面的`ESTABLISHED`状态和`CLOSE_WAIT`状态），更不关心`任何网络状况的变化`**。
[1] 我们把执行过`listen调用`、处于`LISTEN状态`的socket称为`监听socket`，而所有处于`ESTABLISHED状态`的socket则称为`连接socket`。



## 5.6 发起连接
如果说服务器通过`listen`调用来被动接受连接，那么`客户端`需要通过如下`系统调用`来主动与服务器建立连接：
```c
#include＜sys/types.h＞
#include＜sys/socket.h＞
int connect(int sockfd, const struct sockaddr*serv_addr, socklen_t addrlen);
```
`sockfd参数`
    由socket系统调用返回一个socket。
`serv_addr参数`
    是`服务器监听的socket地址`，addrlen参数则指定这个地址的长度。
connect成功时返回0。
    一旦成功建立连接，`sockfd`就唯一地标识了这个连接，客户端就可以通过读写`sockfd`来与服务器通信。
connect失败则返回-1并设置errno。
    其中两种常见的errno是`ECONNREFUSED`和`ETIMEDOUT`，它们的含义如下：
    `ECONNREFUSED`，
        目标端口不存在，连接被拒绝。我们在3.5.1小节讨论过这种情况。
    `ETIMEDOUT`，
        连接超时。我们在3.3.3小节讨论过这种情况。



## 5.7 关闭连接
关闭一个连接实际上就是`关闭该连接对应的socket`，这可以通过如下关闭普通文件描述符的系统调用来完成：
```c
#include＜unistd.h＞
int close(int fd);
```
fd参数是`待关闭的socket`。
不过，**`close系统调用`并非总是立即关闭一个连接，而是将fd的`引用计数减1`。`只有当fd的引用计数为0时，才真正关闭连接`。多进程程序中，一次`fork系统调用`默认将使父进程中`打开的socket`的`引用计数加1`，因此我们必须在`父进程和子进程`中都对该socket执行`close`调用才能将连接关闭。**

如果无论如何都要立即终止连接（而不是将socket的引用计数减1），可以使用如下的shutdown系统调用（相对于close来说，它是专门为网络编程设计的）：
```c
#include＜sys/socket.h＞
int shutdown(int sockfd,int howto);
```
sockfd参数是待关闭的socket。`howto`参数决定了`shutdown的行为`，
shutdown能够`分别关闭socket上的读或写(接收或者读取数据的能力)，或者都关闭`。
而close在关闭连接时只能将socket上的读和写同时关闭。
shutdown成功时返回0，失败则返回-1并设置errno。

## 5.8 数据读写
### 5.8.1 TCP数据读写
对文件的读写操作`read`和`write`同样适用于socket。
但是socket编程接口提供了几个`专门用于socket数据读写`的系统调用，它们增加了对数据读写的控制。
其中用于TCP流数据读写的系统调用是：
```c
#include＜sys/types.h＞
#include＜sys/socket.h＞
ssize_t recv(int sockfd, void*buf, size_t len, int flags);
ssize_t send(int sockfd, const void*buf, size_t len, int flags);
```
`recv`读取`sockfd`上的数据，`buf`和`len`参数分别指定`读缓冲区的位置和大小`，flags参数的含义见后文，通常设置为0即可。
    recv成功时返回`实际读取到的数据的长度`，它可能小于我们期望的长度len。
    **因此我们可能要`多次调用recv`，才能读取到完整的数据**。
    recv可能返回0，这意味着通信对方已经关闭连接了。
    recv出错时返回-1并设置errno。
`send`往`sockfd`上写入数据，`buf`和`len`参数分别指定`写缓冲区的位置和大小`。
    send成功时返回实际写入的数据的长度，失败则返回-1并设置errno。
    flags参数为数据收发提供了额外的控制，它可以取表5-4所示选项中的一个或几个的逻辑或。

`MSG_OOB`选项给应用程序提供了发送和接收带外数据的方法
发送带外数据
```c

#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<assert.h>
#include<stdio.h>
#include<unistd.h>
#include<string.h>
#include<stdlib.h>

int main(int argc, char*argv[])
{
    if(argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char*ip = argv[1];
    int port = atoi(argv[2]);
    struct sockaddr_in server_address;
    bzero(&server_address, sizeof(server_address));
    server_address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &server_address.sin_addr);
    server_address.sin_port = htons(port);
    int sockfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(sockfd >= 0);
    if(connect(sockfd, (struct sockaddr*)&server_address, sizeof(server_address)) < 0)
    {
        printf("connection failed\n");
    }
    else
    {
        const char*oob_data = "abc";
        const char*normal_data = "123";
        send(sockfd, normal_data, strlen(normal_data), 0);  // 123
        send(sockfd, oob_data, strlen(oob_data), MSG_OOB);  // abc
        send(sockfd, normal_data, strlen(normal_data), 0);  // 123
    }
    close(sockfd);
    return 0;
}
```

接收带外数据
```c
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<assert.h>
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<errno.h>
#include<string.h>
#define BUF_SIZE 1024

int main(int argc, char*argv[])
{
    if(argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char*ip = argv[1];

    int port = atoi(argv[2]);
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);

    int sock = socket(PF_INET, SOCK_STREAM, 0);
    assert(sock >= 0);
    
    int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
    assert(ret != -1);
    
    ret = listen(sock, 5);  // 开始监听
    assert(ret != -1);

    struct sockaddr_in client;
    socklen_t client_addrlength = sizeof(client);
    int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);   // 阻塞调用
    if(connfd < 0)
    {
        printf("errno is:%d\n", errno);
    }
    else
    {   // 发送端发送了123 abc 123
        char buffer[BUF_SIZE];  // 1024
        memset(buffer, '\0', BUF_SIZE);
        ret = recv(connfd, buffer, BUF_SIZE - 1, 0);    // 接收1023个字符
        printf("got %d bytes of normal data'%s'\n", ret, buffer);   // 123ab
        memset(buffer, '\0', BUF_SIZE);
        ret = recv(connfd, buffer, BUF_SIZE - 1, MSG_OOB);
        printf("got %d bytes of oob data'%s'\n", ret, buffer);      // c
        memset(buffer, '\0', BUF_SIZE);
        ret = recv(connfd, buffer, BUF_SIZE - 1, 0);
        printf("got %d bytes of normal data'%s'\n", ret, buffer);   // 123
        close(connfd);
    }
    close(sock);
    return 0;
}
```
客户端发送给服务器的`3字节`的`带外数据`“`abc`”中，仅有最后一个字符“c”被服务器当成真正的带外数据接收（正如3.8节讨论的那样）。
并且，`服务器对正常数据的接收 将被 带外数据 截断`，即 前一部分正常数据“123ab” 和 后续的正常数据“123” 是不能被一个recv调用全部读出的。

值得一提的是，flags参数只对send和recv的当前调用生效，而后面我们将看到如何通过`setsockopt`系统调用永久性地修改socket的某些属性。



### 5.8.2 UDP数据读写
socket编程接口中用于UDP数据报读写的系统调用是：
```c
#include＜sys/types.h＞
#include＜sys/socket.h＞
ssize_t recvfrom(int sockfd, void*buf, size_t len, int flags, struct sockaddr*src_addr, socklen_t* addrlen);
ssize_t sendto(int sockfd, const void*buf, size_t len, int flags, const struct sockaddr* dest_addr, socklen_t addrlen);
```
`recvfrom`
    读取sockfd上的数据，`buf`和`len`参数分别指定读缓冲区的位置和大小。
    **因为UDP通信没有连接的概念，所以我们每次读取数据都需要获取`发送端的socket地址`，即参数src_addr所指的内容，addrlen参数则指定该地址的长度**。
`sendto`
    往sockfd上写入数据，`buf`和`len`参数分别指定写缓冲区的位置和大小。
    `dest_addr`参数指定`接收端的socket地址`，addrlen参数则指定该地址的长度。
    这两个系统调用的flags参数以及返回值的含义均与`send/recv`系统调用的flags参数及返回值相同。
    值得一提的是，`recvfrom/sendto`系统调用也可以用于`面向连接（STREAM）`的socket的数据读写，只需要把最后两个参数都设置为NULL以忽略发送端/接收端的socket地址（因为我们已经和对方建立了连接，所以已经知道其socket地址了）。



### 5.8.3 通用数据读写函数
socket编程接口还提供了一对`通用的 数据读写系统调用`。
它们不仅能用于TCP流数据，也能用于UDP数据报：
```c
#include＜sys/socket.h＞
ssize_t recvmsg(int sockfd, struct msghdr*msg, int flags);
ssize_t sendmsg(int sockfd, struct msghdr*msg, int flags);
```
sockfd参数指定被操作的`目标socket`。
msg参数是msghdr结构体类型的指针，msghdr结构体的定义如下：
```c
struct msghdr
{
    void *msg_name;              /*  socket地址  */
    socklen_t msg_namelen;      /*  socket地址的长度    */
    struct iovec*msg_iov;       /*  分散的内存块，见后文  */
    int msg_iovlen;             /*  分散内存块的数量    */
    void *msg_control;          /*  指向辅助数据的起始位置    */
    socklen_t msg_controllen;   /*  辅助数据的大小  */
    int msg_flags;              /*  复制函数中的flags参数，并在调用过程中更新   */
};
```
`msg_name`成员指向一个socket地址结构变量。
    它指定`通信对方的socket地址`。
        对于面向连接的TCP协议，该成员没有意义，必须被设置为NULL。
        这是因为对数据流socket而言，对方的地址已经知道。
`msg_namelen`成员则指定了msg_name所指socket地址的长度。
`msg_iov`成员是iovec结构体类型的指针，
    iovec结构体的定义如下：
```c
struct iovec
{
    void *iov_base;      /* 内存起始地址    */
    size_t iov_len;     /*  这块内存的长度    */
};
```

由上可见，`iovec结构体`封装了`一块内存的 起始位置 和 长度`。
`msg_iovlen`指定这样的iovec结构对象有多少个。
对于`recvmsg`而言，数据将被读取并存放在`msg_iovlen`块分散的内存中，这些内存的位置和长度则由msg_iov指向的数组指定，这称为`分散读（scatter read）`；
对于`sendmsg`而言，`msg_iovlen`块分散内存中的数据将被一并发送，这称为`集中写（gather write）`。
`msg_control`和`msg_controllen`成员用于辅助数据的传送。
我们不详细讨论它们，仅在第13章介绍如何使用它们来实现在进程间传递文件描述符。
`msg_flags`成员无须设定，它会复制`recvmsg/sendmsg`的flags参数的内容以影响数据读写过程。
`recvmsg`还会在调用结束前，将某些更新后的标志设置到`msg_flags`中。
`recvmsg/sendmsg`的flags参数以及返回值的含义均与send/recv的flags参数及返回值相同。
[1] 由于socket连接是`全双工的`，这里的“读端”是针对通信对方而言的。



## 5.9 带外标记
代码清单5-7演示了TCP带外数据的接收方法。
但在实际应用中，我们通常无法预期带外数据何时到来。
好在`Linux内核`检测到`TCP紧急标志`时，将`通知应用程序`有带外数据需要接收。
内核通知应用程序带外数据到达的两种常见方式是：
* `I/O复用`产生的`异常事件` 和 
* `SIGURG`信号。
但是，即使应用程序得到了有带外数据需要接收的通知，还需要知道带外数据在数据流中的具体位置，才能准确接收带外数据。
这一点可通过如下系统调用实现：
```c
#include＜sys/socket.h＞
int sockatmark(int sockfd);
```
sockatmark判断sockfd是否处于带外标记，即下一个被读取到的数据是否是带外数据。
如果是，sockatmark返回1，此时我们就可以利用带MSG_OOB标志的recv调用来接收带外数据。
如果不是，则sockatmark返回0。



## 5.10 地址信息函数
在某些情况下，我们想知道一个连接socket的本端socket地址，以及远端的socket地址。
下面这两个函数正是用于解决这个问题：
```c
#include＜sys/socket.h＞
int getsockname(int sockfd, struct sockaddr*address, socklen_t*address_len);
int getpeername(int sockfd, struct sockaddr*address, socklen_t*address_len);
```
`getsockname`获取sockfd对应的`本端socket地址`，并将其存储于address参数指定的内存中，该socket地址的长度则存储于address_len参数指向的变量中。
如果实际socket地址的长度大于address所指内存区的大小，那么该socket地址将被截断。
`getsockname`成功时返回0，失败返回-1并设置errno。
`getpeername`获取sockfd对应的`远端socket地址`，其参数及返回值的含义与getsockname的参数及返回值相同。



## 5.11 socket选项
如果说`fcntl系统调用`是控制`文件描述符`属性的`通用POSIX方法`，
那么下面两个系统调用则是`专门用来读取和设置socket文件描述符属性的方法`：
```c
#include＜sys/socket.h＞
int getsockopt(int sockfd, int level, int option_name, void*option_value, socklen_t*restrict option_len);
int setsockopt(int sockfd, int level, int option_name, const void*option_value, socklen_t option_len);
```
sockfd参数指定被操作的目标socket。
level参数指定要操作哪个协议的选项（即属性），比如IPv4、IPv6、TCP等。option_name参数则指定选项的名字。
我们在表5-5中列举了socket通信中几个比较常用的`socket选项`。
`option_value`和`option_len`参数分别是被操作选项的值和长度。
不同的选项具有不同类型的值，如表5-5中“数据类型”一列所示。
`getsockopt`和`setsockopt`这两个函数成功时返回0，失败时返回-1并设置errno。
值得指出的是，**对服务器而言，有部分`socket选项`只能在`调用listen系统调用前`针对`监听socket` [1] 设置才有效**。
这是因为`连接socket`只能由`accept`调用返回，而`accept`从`listen监听队列`中接受的连接至少已经完成了`TCP三次握手`的前两个步骤（因为listen监听队列中的连接至少已进入`SYN_RCVD`状态，参见图3-8和代码清单5-4），这说明服务器已经往被接受连接上发送出了`TCP同步报文段`。
但有的socket选项却应该在TCP同步报文段中设置，比如TCP最大报文段选项（回忆3.2.2小节，该选项只能由同步报文段来发送）。
对这种情况，Linux给开发人员提供的解决方案是：对监听socket设置这些socket选项，那么accept返回的连接socket将`自动继承`这些选项。
这些socket选项包括：`SO_DEBUG、SO_DONTROUTE、SO_KEEPALIVE、SO_LINGER、SO_OOBINLINE、SO_RCVBUF、SO_RCVLOWAT、SO_SNDBUF、SO_SNDLOWAT、TCP_MAXSEG和TCP_NODELAY`。
而**对`客户端`而言，这些socket选项则应该在调用connect函数之前设置，因为connect调用成功返回之后，TCP三次握手已完成**。
下面我们详细讨论部分重要的socket选项。



### 5.11.1 `SO_REUSEADDR`选项
我们在3.4.2小节讨论过TCP连接的`TIME_WAIT`状态，并提到服务器程序可以通过设置socket选项`SO_REUSEADDR`来强制使用被处于`TIME_WAIT`状态的连接占用的socket地址。
具体实现方法如代码清单5-9所示。















## 6.4 `sendfile`函数
`sendfile`函数在`两个文件描述符之间`直接传递数据（`完全在内核中`操作），从而**避免了`内核缓冲区 和 用户缓冲区之间`的数据拷贝**，效率很高，这被称为零拷贝。
sendfile函数的定义如下：
```c
#include＜sys/sendfile.h>
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



# 第7章 Linux服务器程序规范
除了网络通信外，服务器程序通常还必须考虑许多其他细节问题。
这些细节问题涉及面广且零碎，而且基本上是模板式的，所以我们称之为`服务器程序规范`。比如：
* Linux服务器程序一般以后台进程形式运行。
    后台进程又称`守护进程（daemon）`。
    它没有控制终端，因而也不会意外接收到用户输入。守护进程的父进程通常是`init进程（PID为1的进程）`。
* Linux服务器程序通常有一套`日志系统`，它至少能输出日志到文件，有的高级服务器还能输出日志到`专门的UDP服务器`。
    大部分后台进程都在`/var/log`目录下拥有自己的日志目录。
* Linux服务器程序一般以某个`专门的 非root身份运行`。
    比如`mysqld、httpd、syslogd`等后台进程，分别拥有自己的运行账户`mysql、apache和syslog`。
* Linux服务器程序通常是`可配置的`。
    服务器程序通常能处理很多`命令行选项`，如果一次运行的选项太多，则可以用配置文件来管理。
    绝大多数服务器程序都有配置文件，并存放在/etc目录下。
比如第4章讨论的squid服务器的配置文件是/`etc/squid3/squid.conf`。
* Linux服务器进程通常会在启动的时候生成一个`PID文件`并存入`/var/run`目录中，以记录该后台进程的PID。
    比如syslogd的PID文件是`/var/run/syslogd.pid`。
* Linux服务器程序通常需要考虑系统资源和限制，以预测自身能承受多大负荷，比如`进程可用文件描述符总数 和 内存总量等`。
在开始系统地学习网络编程之前，我们将用一章的篇幅来探讨服务器程序的一些主要的规范。
















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
它在`实现`和`使用`上与`select、poll`有很大差异。
首先，`epoll`使用`一组函数`来完成任务，而不是单个函数。
其次，`epoll`把用户关心的`文件描述符上的事件`放在`内核里的 一个事件表中`，从而无须像`select`和`poll`那样每次调用都要`重复传入文件描述符集 或 事件集`。
**但`epoll`需要使用一个`额外的文件描述符`，来唯一标识`内核中的这个事件表`**。
这个文件描述符使用如下`epoll_create`函数来创建：
```c
#include＜sys/epoll.h>
int epoll_create(int size);     // 用于标识事件表
```
size参数现在并不起作用，只是给内核一个提示，告诉它事件表需要多大。
该函数返回的文件描述符 将用作其他所有epoll系统调用的第一个参数，以指定要访问的`内核事件表`。

下面的函数用来**操作`epoll`的`内核事件表`**：
```c
#include＜sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);
```
`fd`参数是要操作的文件描述符，`op`参数则指定操作类型。
操作类型有如下3种：
* `EPOLL_CTL_ADD`，往事件表中`注册`fd上的事件。
* `EPOLL_CTL_MOD`，`修改`fd上的注册事件。
* `EPOLL_CTL_DEL`，`删除`fd上的注册事件。
`event`参数指定事件，它是`epoll_event`结构指针类型。

`epoll_event`的定义如下：
```c
struct epoll_event
{
    __uint32_t events;  /*  epoll事件   */
    epoll_data_t data;  /*  用户数据  */
};
```
其中`events`成员描述`事件类型`。
`epoll`支持的事件类型和`poll`基本相同。
表示`epoll`事件类型的宏是在poll对应的宏前加上“E”，比如`epoll`的`数据可读事件`是`EPOLLIN`。
但`epoll`有两个额外的事件类型————`EPOLLET`和`EPOLLONESHOT`。
它们对于epoll的高效运作非常关键，我们将在后面讨论它们。
`data`成员用于存储用户数据，其类型`epoll_data_t`的定义如下：
```c
typedef union epoll_data
{
    void* ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```
`epoll_data_t`是一个联合体，其4个成员中使用最多的是`fd`，它指定`事件所从属的 目标文件描述符`。
`ptr`成员可用来指定与fd相关的`用户数据`。
但由于`epoll_data_t`是一个联合体，我们不能同时使用其`ptr`成员和`fd`成员，因此，如果要将文件描述符和用户数据关联起来（正如8.5.2小节讨论的将句柄和事件处理器绑定一样），以实现快速的数据访问，只能使用其他手段，比如放弃使用`epoll_data_t`的fd成员，而在ptr指向的用户数据中包含fd。
`epoll_ctl`成功时返回0，失败则返回-1并设置errno。



### 9.3.2 `epoll_wait`函数
`epoll系列系统调用`的主要接口是`epoll_wait`函数。
它在一段`超时时间`内等待一组文件描述符上的事件，其原型如下：
```c
#include＜sys/epoll.h＞
int epoll_wait(int epfd,struct epoll_event*events, int maxevents, int timeout);
```
该函数成功时返回`就绪的文件描述符的个数`，失败时返回-1并设置errno。
关于该函数的参数，我们从后往前讨论。
timeout参数的含义与poll接口的timeout参数相同。
`maxevents`参数指定最多监听多少个事件，它必须大于0。
**`epoll_wait`函数如果检测到事件，就将`所有就绪的事件`从`内核事件表（由epfd参数指定）`中复制到它的第二个参数`events指向的数组`中**。
**这个`数组`只用于输出`epoll_wait`检测到的就绪事件，而不像`select`和`poll`的`数组参数`那样既用于`传入用户注册的事件`，又用于输出`内核检测到的就绪事件`。这就极大地提高了`应用程序 索引就绪文件描述符 的效率`**。
代码清单9-2体现了这个差别。

```c
/*如何索引poll返回的就绪文件描述符*/
int ret = poll(fds, MAX_EVENT_NUMBER, -1);
/*必须遍历所有已注册文件描述符并找到其中的就绪者（当然，可以利用ret来稍做优化）*/
for(int i = 0; i ＜ MAX_EVENT_NUMBER; ++i)
{
    if(fds[i].revents & POLLIN)/*判断第i个文件描述符是否就绪    POLLIN表示该socket可读*/
    {
        int sockfd = fds[i].fd;
        /*处理sockfd*/
    }
}

/*如何索引epoll返回的就绪文件描述符*/
int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
/*仅遍历就绪的ret个文件描述符   因为就绪的文件描述符和数目直接就返回了*/
for(int i = 0; i ＜ ret; i++)
{
    int sockfd = events[i].data.fd;
    /*sockfd肯定就绪，直接处理*/
}

```



### 9.3.3 `LT`和`ET`模式
epoll对文件描述符的操作有两种模式：
`LT（Level Trigger，电平触发）`模式和`ET（Edge Trigger，边沿触发）模式`。
`LT模式`是`默认的工作模式`，这种模式下epoll相当于一个效率较高的`poll`。
当往epoll内核事件表中注册一个文件描述符上的`EPOLLET`事件时，`epoll`将以`ET模式`来操作该文件描述符。`ET模式`是epoll的高效工作模式。
> 对于采用`LT工作模式`的文件描述符，当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，**应用程序可以不立即处理该事件**。
    这样，当应用程序下一次调用epoll_wait时，epoll_wait还会再次向应用程序通告此事件，直到该事件被处理。（内核如果发现文件描述符上有没有处理完的事件就会一直通知应用程序处理）
> 而对于采用`ET工作模式`的文件描述符，当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，**应用程序必须立即处理该事件**（不处理的话就忽略了），因为后续的`epoll_wait`调用将不再向应用程序通知这一事件。
    可见，ET模式在很大程度上降低了同一个epoll事件被重复触发的次数，因此效率要比LT模式高。

代码清单9-3体现了LT和ET在工作方式上的差异。
```c
#include<sys/types.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<assert.h>
#include<stdio.h>

#include<unistd.h>
#include<errno.h>
#include<string.h>
#include<fcntl.h>
#include<stdlib.h>
#include<sys/epoll.h>
#include<pthread.h>
#define MAX_EVENT_NUMBER 1024
#define BUFFER_SIZE 10

/*将文件描述符设置成非阻塞的*/
int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}

/*将文件描述符fd上的 EPOLLIN 注册到 epollfd指示的epoll内核事件表中，参数enable_et指定是否对fd启用ET模式*/
void addfd(int epollfd, int fd, bool enable_et)
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN;
    if(enable_et)
    {
        event.events |= EPOLLET;
    }
    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
    setnonblocking(fd);
}

/*  LT模式的工作流程    */
void lt(epoll_event*events, int number, int epollfd, int listenfd)
{
    char buf[BUFFER_SIZE];
    for(int i = 0; i < number; i++)
    {
        int sockfd = events[i].data.fd;
        if(sockfd == listenfd)
        {
            struct sockaddr_in client_address;
            socklen_t client_addrlength = sizeof(client_address);
            int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);   // 客户端产生一个连接请求就返回
            addfd(epollfd, connfd, false);  /*对connfd禁用ET模式*/
        }
        else if(events[i].events & EPOLLIN)
        {
            /*只要socket读缓存中还有未读出的数据，这段代码就被触发*/
            printf("event trigger once\n");
            memset(buf, '\0', BUFFER_SIZE);
            int ret = recv(sockfd, buf, BUFFER_SIZE - 1, 0);
            if(ret <= 0)
            {
                close(sockfd);
                continue;
            }
            printf("get%d bytes of content:%s\n", ret, buf);
        }
        else
        {
            printf("something else happened\n");
        }
    }
}

/*ET模式的工作流程*/
void et(epoll_event*events, int number, int epollfd, int listenfd)
{
    char buf[BUFFER_SIZE];
    for(int i = 0; i < number; i++)
    {
    int sockfd = events[i].data.fd;
    if(sockfd == listenfd)
    {
        struct sockaddr_in client_address;
        socklen_t client_addrlength = sizeof(client_address);
        int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
        addfd(epollfd, connfd, true);     /*对connfd开启ET模式*/
    }
    else if(events[i].events & EPOLLIN)
    {
        /*这段代码不会被重复触发，所以我们循环读取数据，以确保把socket读缓存中的所有数据读出*/
        printf("event trigger once\n");
        while(1)
        {
            memset(buf, '\0', BUFFER_SIZE);
            int ret = recv(sockfd, buf, BUFFER_SIZE - 1, 0);
            if(ret < 0)
            {
                /*对于非阻塞IO，下面的条件成立表示数据已经全部读取完毕。此后，epoll就能再次触发sockfd上的EPOLLIN事件，以驱动下一次读操作*/
                if((errno == EAGAIN) || (errno == EWOULDBLOCK))
                {
                    printf("read later\n");
                    break;
                }
                close(sockfd);
                break;
            }
            else if(ret == 0)
            {
                close(sockfd);
            }
            else
            {
                printf("get%d bytes of content:%s\n", ret, buf);
            }
        }
    }
    else
    {
        printf("something else happened\n");
    }
    }
}

int main(int argc,char*argv[])
{
    if(argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char*ip = argv[1];
    int port = atoi(argv[2]);
    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int listenfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);

    ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
    assert(ret != -1);

    ret = listen(listenfd, 5);
    assert(ret != -1);
    
    epoll_event events[MAX_EVENT_NUMBER];
    int epollfd = epoll_create(5);
    assert(epollfd != -1);

    addfd(epollfd, listenfd, true);
    while(1)
    {
        int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
        if(ret < 0)
        {
            printf("epoll failure\n");
            break;
        }
        lt(events, ret, epollfd, listenfd); /*使用LT模式*/
        // et(events, ret, epollfd, listenfd);  /*使用ET模式*/
    }
    close(listenfd);
    return 0;
}
```
telnet到这个服务器程序上并一次传输超过10字节（BUFFER_SIZE的大小）的数据，然后比较LT模式和ET模式的异同。
你会发现，正如我们预期的，ET模式下事件被触发的次数要比LT模式下少很多。
注意　每个使用ET模式的文件描述符都应该是`非阻塞的`。
如果文件描述符是阻塞的，那么读或写操作将会因为没有后续的事件而一直处于阻塞状态（饥渴状态）。



### 9.3.4 `EPOLLONESHOT`事件
即使我们使用`ET模式`，一个socket上的某个事件还是可能被触发多次。
这在**并发程序**中就会引起一个问题。
    比如一个线程（或进程，下同）在读取完某个`socket`上的数据后开始处理这些数据，而`在数据的处理过程中`该socket上又有`新数据可读`（EPOLLIN再次被触发），此时另外一个线程被唤醒来读取这些新的数据。于是就`出现了两个线程同时操作一个socket的局面`。这当然不是我们期望的。
我们期望的是一个socket连接在任一时刻都只被一个线程处理。这一点可以使用`epoll 的 EPOLLONESHOT事件`实现。
**对于注册了`EPOLLONESHOT事件`的文件描述符，操作系统`最多触发其上注册的一个可读、可写或者异常事件`，且`只触发一次`，除非我们使用`epoll_ctl`函数`重置`该文件描述符上注册的EPOLLONESHOT事件**。
这样，当一个线程在处理某个socket时，其他线程是不可能有机会操作该socket的。
但反过来思考，注册了`EPOLLONESHOT事件`的socket一旦被某个线程处理完毕，该线程就应该`立即重置`这个socket上的EPOLLONESHOT事件（因为是一次性的），以确保这个socket下一次可读时，其EPOLLIN事件能被触发，进而让其他工作线程有机会继续处理这个socket。

代码清单9-4　使用EPOLLONESHOT事件
```c
#include<sys/types.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<assert.h>
#include<stdio.h>
#include<unistd.h>
#include<errno.h>
#include<string.h>
#include<fcntl.h>
#include<stdlib.h>
#include<sys/epoll.h>
#include<pthread.h>

#define MAX_EVENT_NUMBER 1024
#define BUFFER_SIZE 1024

struct fds
{
    int epollfd;
    int sockfd;
};

int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}

/*将fd上的EPOLLIN和EPOLLET事件注册到epollfd指示的epoll内核事件表中，参数oneshot指定是否注册fd上的EPOLLONESHOT事件*/
void addfd(int epollfd, int fd, bool oneshot)
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN|EPOLLET; // 边缘触发 可读
    if(oneshot)
    {
        event.events |= EPOLLONESHOT;
    }
    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
    setnonblocking(fd);
}

/*重置fd上的事件。这样操作之后，尽管fd上的EPOLLONESHOT事件被注册，但是操作系统仍然会触发fd上的EPOLLIN事件，且只触发一次*/
void reset_oneshot(int epollfd, int fd)
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN|EPOLLET|EPOLLONESHOT;
    epoll_ctl(epollfd, EPOLL_CTL_MOD, fd, &event);
}

/*工作线程*/
void* worker(void* arg)
{
    int sockfd = ((fds*)arg)->sockfd;
    int epollfd = ((fds*)arg)->epollfd;
    printf("start new thread to receive data on fd:%d\n", sockfd);
    char buf[BUFFER_SIZE];
    memset(buf, '\0', BUFFER_SIZE);
    /*循环读取sockfd上的数据，直到遇到EAGAIN错误*/
    while(1)
    {
        int ret = recv(sockfd, buf, BUFFER_SIZE - 1, 0);
        if(ret == 0)
        {
            close(sockfd);
            printf("foreiner closed the connection\n");
            break;
        }
        else if(ret < 0)
        {
            if(errno == EAGAIN)
            {
                reset_oneshot(epollfd, sockfd);
                printf("read later\n");
                break;
            }
        }
        else
        {
            printf("get content:%s\n", buf);
            /*休眠5s，模拟数据处理过程*/
            sleep(5);
        }
    }
    printf("end thread receiving data on fd:%d\n", sockfd);
}

int main(int argc, char*argv[])
{
    if(argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }

    const char*ip = argv[1];
    int port = atoi(argv[2]);
    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);

    int listenfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);

    ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
    assert(ret != -1);

    ret = listen(listenfd, 5);
    assert(ret != -1);

    epoll_event events[MAX_EVENT_NUMBER];
    int epollfd = epoll_create(5);
    assert(epollfd != -1);
    /*  
        注意，监听socket listenfd上，是不能注册EPOLLONESHOT事件的，否则应用程序只能处理一个客户连接！
        因为后续的客户连接请求将不再触发listenfd上的EPOLLIN事件
    */
    addfd(epollfd, listenfd, false);
    while(1)
    {
        int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
        if(ret < 0)
        {
            printf("epoll failure\n");
            break;
        }
        for(int i = 0; i < ret; i++)
        {
            int sockfd = events[i].data.fd;
            if(sockfd == listenfd)
            {
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof(client_address);
                int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
                /*对每个非监听文件描述符都注册EPOLLONESHOT事件*/
                addfd(epollfd, connfd, true);
            }
            else if(events[i].events&EPOLLIN)
            {
                pthread_t thread;
                fds fds_for_new_worker;
                fds_for_new_worker.epollfd = epollfd;
                fds_for_new_worker.sockfd = sockfd;
                /*新启动一个工作线程为sockfd服务*/
                pthread_create(&thread, NULL, worker, (void*)&fds_for_new_worker);
            }
            else
            {
                printf("something else happened\n");
            }
        }
    }
    close(listenfd);
    return 0;
}
```
从工作线程函数worker来看，如果一个工作线程处理完某个socket上的一次请求（我们用休眠5 s来模拟这个过程）之后，又接收到该socket上新的客户请求，则该线程将继续为这个socket服务。
并且因为该socket上注册了EPOLLONESHOT事件，其他线程没有机会接触这个socket，如果工作线程等待5 s后仍然没收到该socket上的下一批客户数据，则它将放弃为该socket服务。
同时，它调用`reset_oneshot`函数来重置该socket上的注册事件，这将使epoll有机会再次检测到该socket上的EPOLLIN事件，进而使得其他线程有机会为该socket服务。
由此看来，尽管一个socket在不同时间可能被不同的线程处理，但同一时刻肯定只有一个线程在为它服务。这就保证了连接的完整性，从而避免了很多可能的竞态条件。



## 9.4 三组I/O复用函数的比较
前面我们讨论了`select、poll和epoll`三组I/O复用系统调用，这3组系统调用都能同时监听多个文件描述符。
它们将等待由`timeout`参数指定的`超时时间`，直到一个或者多个文件描述符上有事件发生时返回，返回值是就绪的文件描述符的数量。返回0表示没有事件发生。
现在我们从`事件集`、`最大支持文件描述符数`、`工作模式`和`具体实现`等四个方面进一步比较它们的异同，以明确在实际应用中应该选择使用哪个（或哪些）。
这3组函数都通过`某种结构体变量`来告诉内核`监听 哪些文件描述符上的 哪些事件`，并使用`该结构体类型的 参数`来获取`内核处理的结果`。

`select`的参数类型`fd_set`没有将`文件描述符`和`事件绑定`，它仅仅是一个`文件描述符集合`，因此select需要提供3个这种类型的参数来分别`传入 和 输出 可读、可写及异常 等事件`。
这一方面使得select不能处理更多类型的事件，另一方面由于内核对`fd_set集合`的在线修改（因为是传指针给内核以修改），应用程序下次调用select前不得不重置这3个fd_set集合。

`poll`的参数类型`pollfd`则多少“聪明”一些。
它把`文件描述符`和`事件`都定义其中，任何事件都被统一处理，从而使得编程接口简洁得多。
并且内核每次修改的是`pollfd结构体`的`revents成员`，而`events成员保持不变`，因此下次调用poll时应用程序无须重置pollfd类型的事件集参数。
由于每次`select`和`poll`调用都返回`整个用户注册的事件集合（其中包括就绪的和未就绪的）`，所以应用程序索引就绪文件描述符的时间复杂度为`O(n)`。

`epoll`则采用与select和poll完全不同的方式来管理用户注册的事件。
**它在`内核中`维护一个`事件表`，并提供了一个独立的系统调用`epoll_ctl`来控制往其中`添加、删除、修改事件`**。
**这样，每次`epoll_wait`调用都直接从该`内核事件表`中取得用户注册的事件，而无须反复从用户空间读入这些事件(减少状态切换)**。
`epoll_wait`系统调用的`events参数`仅用来返回就绪的事件，这使得应用程序索引就绪文件描述符的时间复杂度达到`O(1)`。

`poll`和`epoll_wait`分别用nfds和maxevents参数指定最多监听多少个文件描述符和事件。
这两个数值都能达到系统允许打开的最大文件描述符数目，即65535（cat/proc/sys/fs/file-max）。
而`select`允许监听的最大文件描述符数量通常有限制。虽然用户可以修改这个限制，但这可能导致不可预期的后果。
`select`和`poll`都只能工作在相对低效的`LT模式`，而epoll则可以工作在`ET高效模式`。
并且epoll还支持`EPOLLONESHOT事件`。该事件能进一步减少可读、可写和异常等事件被触发的次数。

从实现原理上来说，`select`和`poll`采用的都是轮询的方式，即每次调用都要`扫描整个注册文件描述符集合`，并将其中就绪的文件描述符返回给用户程序，因此它们检测就绪事件的算法的时间复杂度是`O(n)`。
`epoll_wait`则不同，它采用的是`回调`的方式。
内核检测到就绪的文件描述符时，将触发`回调函数`，回调函数就将该文件描述符上对应的事件插入`内核就绪事件队列`。内核最后在适当的时机`将 该就绪事件队列中的内容 拷贝到用户空间`。
因此`epoll_wait`无须轮询整个文件描述符集合来检测哪些事件已经就绪，其算法时间复杂度是O(1)。
**但是，当`活动连接比较多`的时候，epoll_wait的效率未必比select和poll高，因为此时回调函数被触发得过于频繁。所以epoll_wait适用于`连接数量多，但活动连接较少`的情况**。



## 9.5 I/O复用的高级应用一：`非阻塞`connect
这段话描述了connect出错时的一种errno值：EINPROGRESS。这种
错误发生在对非阻塞的socket调用connect，而连接又没有立即建立时。
根据man文档的解释，在这种情况下，我们可以调用select、poll等函数
来监听这个连接失败的socket上的可写事件。当select、poll等函数返回
后，再利用getsockopt来读取错误码并清除该socket上的错误。如果错误
码是0，表示连接成功建立，否则连接失败。
通过上面描述的非阻塞connect方式，我们就能同时发起多个连接并
一起等待。下面看看非阻塞connect的一种实现 [2] ，如代码清单9-5所
示。
代码清单9-5　非阻塞connect
```c

#include<sys/types.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<stdlib.h>
#include<assert.h>
#include<stdio.h>
#include<time.h>
#include<errno.h>
#include<fcntl.h>
#include<sys/ioctl.h>
#include<unistd.h>
#include<string.h>

#define BUFFER_SIZE 1023

int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}

/*超时连接函数，参数分别是服务器IP地址、端口号和超时时间（毫秒）。函数成功时返回已经处于连接状态的socket，失败则返回-1*/
int unblock_connect(const char*ip, int port, int time)
{
    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int sockfd = socket(PF_INET, SOCK_STREAM, 0);
    int fdopt = setnonblocking(sockfd);

    ret = connect(sockfd, (struct sockaddr*)&address, sizeof(address));
    if(ret == 0)
    {
        /*如果连接成功，则恢复sockfd的属性，并立即返回之*/
        printf("connect with server immediately\n");
        fcntl(sockfd, F_SETFL, fdopt);
        return sockfd;
    }
    else if(errno != EINPROGRESS)
    {
        /*如果连接没有立即建立，那么只有当errno是EINPROGRESS时才表示连接还在进行，否则出错返回*/
        printf("unblock connect not support\n");
        return -1;
    }

    // EINPROGRESS错误表示连接尚未建立
    fd_set readfds;
    fd_set writefds;
    struct timeval timeout;
    FD_ZERO(&readfds);
    FD_SET(sockfd, &writefds);
    timeout.tv_sec = time;
    timeout.tv_usec = 0;

    // 参数：最大文件描述符+1， 读描述符集合、写描述符集合、异常描述符集合、超时时间
    ret = select(sockfd + 1, NULL, &writefds, NULL, &timeout);
    if(ret <= 0)
    {
        /*select超时或者出错，立即返回*/
        printf("connection time out\n");
        close(sockfd);
        return -1;
    }
    if(!FD_ISSET(sockfd, &writefds))
    {
        printf("no events on sockfd found\n");
        close(sockfd);
        return -1;
    }

    int error = 0;
    socklen_t length = sizeof(error);

    /*调用getsockopt来获取并清除sockfd上的错误*/
    if(getsockopt(sockfd, SOL_SOCKET, SO_ERROR, &error, &length) < 0)
    {
        printf("get socket option failed\n");
        close(sockfd);
        return -1;
    }
    /*错误号不为0表示连接出错*/
    if(error != 0)
    {
        printf("connection failed after select with the error:%d\n", error);
        close(sockfd);
        return -1;
    }
    /*连接成功*/
    printf("connection ready after select with the socket:%d\n", sockfd);
    fcntl(sockfd, F_SETFL, fdopt);
    return sockfd;
}

int main(int argc, char*argv[])
{
    if(argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char*ip = argv[1];
    int port = atoi(argv[2]);
    int sockfd = unblock_connect(ip, port, 10);
    if(sockfd < 0)
    {
        return 1;
    }
    close(sockfd);
    return 0;
}
```
但遗憾的是，这种方法存在几处移植性问题。
首先，非阻塞的socket可能导致connect始终失败。
其次，select对处于EINPROGRESS状态下的socket可能不起作用。
最后，对于出错的socket，getsockopt在有些系统（比如Linux）上返回-1（正如代码清单9-5所期望的），而在有些系统（比如源自伯克利的UNIX）上则返回0。
这些问题没有一个统一的解决方法，感兴趣的读者可自行参考相关文献。



## 9.6 I/O复用的高级应用二：聊天室程序
像ssh这样的登录服务通常要同时处理`网络连接`和`用户输入`，这也可以使用`I/O复用`来实现。
本节我们以poll为例实现一个简单的聊天室程序，以阐述如何使用I/O复用技术来同时处理网络连接和用户输入。
该聊天室程序能让所有用户同时在线群聊，它分为客户端和服务器两个部分。
其中客户端程序有两个功能：
* 一是从标准输入终端读入用户数据，并将用户数据发送至服务器；
* 二是往标准输出终端打印服务器发送给它的数据。服务器的功能是接收客户数据，并把客户数据发送给每一个登录到该服务器上的客户端（数据发送者除外）。
下面我们依次给出客户端程序和服务器程序的代码。

### 9.6.1 客户端
客户端程序使用poll同时监听用户输入和网络连接，并利用`splice`函数将用户输入内容直接定向到网络连接上以发送之，从而实现数据零拷贝，提高了程序执行效率。
客户端程序如代码清单9-6所示。
```c
// #define_GNU_SOURCE 1    // 诉编译器包含GNU特定的扩展功能和库
#include<sys/types.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<assert.h>
#include<stdio.h>
#include<unistd.h>
#include<string.h>
#include<stdlib.h>
#include<poll.h>
#include<fcntl.h>

#define BUFFER_SIZE 64

int main(int argc, char*argv[])
{
    if(argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }

    const char*ip = argv[1];
    int port = atoi(argv[2]);
    struct sockaddr_in server_address;
    bzero(&server_address, sizeof(server_address));
    server_address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &server_address.sin_addr);
    server_address.sin_port = htons(port);

    int sockfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(sockfd >= 0);
    if(connect(sockfd, (struct sockaddr*)&server_address, sizeof(server_address)) < 0)
    {
        printf("connection failed\n");
        close(sockfd);
        return 1;
    }
    pollfd fds[2];
    /*  注册文件描述符0（标准输入）和文件描述符sockfd上的可读事件   */
    fds[0].fd = 0;
    fds[0].events = POLLIN;
    fds[0].revents = 0;
    fds[1].fd = sockfd;
    fds[1].events = POLLIN | POLLRDHUP;
    fds[1].revents = 0;
    char read_buf[BUFFER_SIZE];
    int pipefd[2];
    int ret = pipe(pipefd); // 创建管道，传入数组，0是读端，1是写端（管道就是这么使用的）
    assert(ret != -1);
    while(1)
    {
        ret = poll(fds, 2, -1);     // ！！！
        if(ret < 0)
        {
            printf("poll failure\n");
            break;
        }
        if(fds[1].revents & POLLRDHUP)  // server连接关闭
        {
            printf("server close the connection\n");
            break;
        }
        else if(fds[1].revents & POLLIN)    // 连接socket出现可读事件
        {
            memset(read_buf,'\0', BUFFER_SIZE);
            recv(fds[1].fd,read_buf, BUFFER_SIZE - 1, 0);   // 其他的客户端发了消息，一律都能收到
            printf("%s\n", read_buf);
        }
        if(fds[0].revents & POLLIN)     // 标注输入出现可读事件
        {
            //  splice：将数据从一个文件描述符直接传输到另一个文件描述符，而不需要经过用户空间的缓冲区
            /*
                splice 是一个系统调用，用于在两个文件描述符之间直接传输数据，而不需要经过用户空间。
                参数解释：
                    0：源文件描述符，这里是标准输入（通常是键盘输入）。
                    NULL：源文件描述符的偏移量指针，NULL 表示由内核自动管理偏移量。
                    pipefd[1]：目标文件描述符，这里是管道的写端。
                    NULL：目标文件描述符的偏移量指针，NULL 表示由内核自动管理偏移量。
                    32768：要传输的最大字节数。
                    SPLICE_F_MORE：指示还有更多数据将被发送。       ？？？内核会优化缓冲区管理和数据传输过程，以提高性能
                    SPLICE_F_MOVE：指示数据应尽可能地移动而不是复制。？？？内核具体是咋移动数据的？
            */
            ret = splice(0, NULL, pipefd[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);   // 将标准输入的数据传输到管道
            /*
                参数解释：
                    pipefd[0]：源文件描述符，这里是管道的读端。
                    NULL：源文件描述符的偏移量指针，NULL 表示由内核自动管理偏移量。
                    sockfd：目标文件描述符，这里是 socket。
                    NULL：目标文件描述符的偏移量指针，NULL 表示由内核自动管理偏移量。
                    32768：要传输的最大字节数。
                    SPLICE_F_MORE：指示还有更多数据将被发送。
                    SPLICE_F_MOVE：指示数据应尽可能地移动而不是复制。
            */
            ret = splice(pipefd[0], NULL, sockfd, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);  // 把数据从管道传输到socket
        }
    }
    close(sockfd);
    return 0;
}
```



### 9.6.2 服务器
服务器程序使用poll同时管理监听socket和连接socket，并且使用牺牲空间换取时间的策略来提高服务器性能，如代码清单9-7所示。
代码清单9-7　聊天室服务器程序
```c
// #define_GNU_SOURCE 1
#include<sys/types.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>

#include<assert.h>
#include<stdio.h>
#include<unistd.h>
#include<errno.h>
#include<string.h>
#include<fcntl.h>
#include<stdlib.h>
#include<poll.h>

#define USER_LIMIT 5    /*最大用户数量*/
#define BUFFER_SIZE 64  /*读缓冲区的大小*/

#define FD_LIMIT 65535  /*文件描述符数量限制*/

/*  客户数据：客户端socket地址、待写到客户端的数据的位置、从客户端读入的数据    */
struct client_data
{
    sockaddr_in address;
    char* write_buf;
    char buf[BUFFER_SIZE];
};

int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}

int main(int argc, char*argv[])
{
    if(argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }

    const char*ip = argv[1];
    int port = atoi(argv[2]);
    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);

    int listenfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);

    ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
    assert(ret != -1);
    ret = listen(listenfd, 5);
    assert(ret != -1);
    /*
        创建users数组，分配 FD_LIMIT 个client_data对象。可以预期：每个可能的socket连接都可以获得一个这样的对象，
        并且socket的值可以直接用来索引（作为数组的下标）socket连接对应的client_data对象，
        这是将socket和客户数据关联的简单而高效的方式
    */
    client_data* users = new client_data[FD_LIMIT];     // FD_LIMIT = 65535
    /*尽管我们分配了足够多的client_data对象，但为了提高poll的性能，仍然有必要限制用户的数量*/
    pollfd fds[USER_LIMIT + 1];
    int user_counter = 0;
    for(int i = 1; i <= USER_LIMIT; ++i)
    {
        fds[i].fd = -1;
        fds[i].events = 0;
    }
    fds[0].fd = listenfd;
    fds[0].events = POLLIN|POLLERR;
    fds[0].revents = 0;
    while(1)
    {
        ret = poll(fds, user_counter + 1, -1);  // pollfd类型，数组元素数量，超时时间
        if(ret < 0)
        {
            printf("poll failure\n");
            break;
        }
        for(int i = 0; i < user_counter + 1; ++i)
        {
            if( (fds[i].fd == listenfd) && (fds[i].revents & POLLIN) )  // 监听socket上，可读事件发生
            {
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof(client_address);
                int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
                if(connfd < 0)
                {
                    printf("errno is:%d\n", errno);
                    continue;
                }
                /*如果请求太多，则关闭新到的连接*/
                if(user_counter >= USER_LIMIT)
                {
                    const char* info = "too many users\n";
                    printf("%s", info);
                    send(connfd, info, strlen(info), 0);
                    close(connfd);
                    continue;
                }
                /*
                    对于新的连接，同时修改fds和users数组。
                    前文已经提到，users[connfd]对应于新连接文件描述符connfd的客户数据
                */
                user_counter++;

                users[connfd].address = client_address;
                setnonblocking(connfd);     // 设置新的连接socket为非阻塞模式
                fds[user_counter].fd = connfd;  // 因为传给poll的是数组指针，所以直接修改fds数组
                fds[user_counter].events = POLLIN | POLLRDHUP | POLLERR;
                fds[user_counter].revents = 0;
                printf("comes a new user, now have%d users\n", user_counter);
            }
            else if(fds[i].revents & POLLERR)   // 监听到错误
            {
                printf("get an error from%d\n", fds[i].fd);
                char errors[100];
                memset(errors, '\0', 100);
                socklen_t length = sizeof(errors);
                if(getsockopt(fds[i].fd, SOL_SOCKET, SO_ERROR, &errors, &length) < 0)   // 得到错误状态
                {
                    printf("get socket option failed\n");
                }
                continue;
            }
            else if(fds[i].revents & POLLRDHUP) // 连接socket发生连接关闭
            {
                /*如果客户端关闭连接，则服务器也关闭对应的连接，并将用户总数减1*/
                users[fds[i].fd] = users[fds[user_counter].fd];     // ？？？将最后的元素复制到当前这个关闭连接的元素的位置
                close(fds[i].fd);
                fds[i] = fds[user_counter];
                i--;
                user_counter--;
                printf("a client left\n");
            }
            else if(fds[i].revents & POLLIN)    // 连接socket发生可读事件
            {
                int connfd = fds[i].fd;
                memset(users[connfd].buf, '\0', BUFFER_SIZE);
                ret = recv(connfd, users[connfd].buf, BUFFER_SIZE - 1, 0);
                printf("get%d bytes of client data%s from%d\n", ret, users[connfd].buf, connfd);
                if(ret < 0)
                {
                    /*如果读操作出错，则关闭连接*/
                    if(errno != EAGAIN)
                    {
                        close(connfd);
                        users[fds[i].fd] = users[fds[user_counter].fd];
                        fds[i] = fds[user_counter];
                        i--;
                        user_counter--;
                    }
                }
                else if(ret == 0)
                {

                }
                else
                {
                    /*如果接收到客户数据，则通知其他socket连接准备写数据    ！！！因为需要实现客户端与客户端之间的通信*/
                    for(int j = 1; j <= user_counter; ++j)  // 其他的客户端发了消息，一律都要发送
                    {
                        if(fds[j].fd == connfd)
                        {
                            continue;
                        }
                        fds[j].events |= ~POLLIN;
                        fds[j].events |= POLLOUT;  // 告诉 poll() 函数在下一次调用时检查该文件描述符是否准备好写数据（当缓冲区有空间、连接已建立、非阻塞模式下就可以直接写了）
                        users[fds[j].fd].write_buf = users[connfd].buf;     // 写数据缓冲
                    }
                }
            }
            else if(fds[i].revents & POLLOUT)   // 连接socket发生可写事件
            {
                int connfd = fds[i].fd;
                if(!users[connfd].write_buf)
                {
                    continue;
                }
                ret = send(connfd,users[connfd].write_buf, strlen(users[connfd].write_buf), 0);
                users[connfd].write_buf = NULL;
                /*写完数据后需要重新注册fds[i]上的可读事件*/    // 因为写过一次了，等到下次读了数据再次监听可写事件
                fds[i].events |= ~POLLOUT;
                fds[i].events |= POLLIN;
            }
        }
    }
    delete[] users;
    close(listenfd);
    return 0;
}
```



## 9.7 I/O复用的高级应用三：同时处理TCP和UDP服务
至此，我们讨论过的服务器程序都只监听一个端口。
在实际应用中，有不少服务器程序能`同时监听多个端口`，比如超级服务inetd和android的调试服务adbd。
**从bind系统调用的参数来看，一个socket只能与一个socket地址绑定，即`一个socket`只能用来监听`一个端口`**。
因此，服务器如果要同时监听多个端口，就必须创建多个socket，并将它们分别绑定到各个端口上。
这样一来，服务器程序就需要同时管理多个监听socket，I/O复用技术就有了用武之地。
另外，即使是同一个端口，如果服务器要同时处理该端口上的TCP和UDP请求，则也需要创建两个不同的socket：一个是`流socket`，另一个是`数据报socket`，并将它们都绑定到该端口上。
比如代码清单9-8所示的回射服务器就能同时处理一个端口上的TCP和UDP请求。

代码清单9-8　同时处理TCP请求和UDP请求的回射服务器
```c
#include<sys/types.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<assert.h>

#include<stdio.h>
#include<unistd.h>
#include<errno.h>
#include<string.h>
#include<fcntl.h>
#include<stdlib.h>
#include<sys/epoll.h>
#include<pthread.h>
#define MAX_EVENT_NUMBER 1024
#define TCP_BUFFER_SIZE 512
#define UDP_BUFFER_SIZE 1024

int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}

void addfd(int epollfd, int fd)
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN | EPOLLET;   // 可读事件 / 边缘触发
    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
    setnonblocking(fd);
}

int main(int argc, char*argv[])
{
    if(argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }

    const char*ip = argv[1];
    int port = atoi(argv[2]);
    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    /*创建TCP socket，并将其绑定到端口port上*/

    int listenfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);

    ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
    assert(ret != -1);

    ret = listen(listenfd, 5);
    assert(ret != -1);

    /*创建UDP socket，并将其绑定到端口port上*/
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);

    int udpfd = socket(PF_INET, SOCK_DGRAM, 0);
    assert(udpfd >= 0);

    ret = bind(udpfd, (struct sockaddr*)&address, sizeof(address));
    assert(ret != -1);

    epoll_event events[MAX_EVENT_NUMBER];
    int epollfd = epoll_create(5);
    assert(epollfd != -1);
    /*注册TCP socket和UDP socket上的可读事件*/
    addfd(epollfd, listenfd);
    addfd(epollfd, udpfd);
    while(1)
    {
        int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
        if(number < 0)
        {
            printf("epoll failure\n");
            break;
        }
        for(int i = 0; i < number; i++)
        {
            int sockfd = events[i].data.fd;
            if(sockfd == listenfd)
            {
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof(client_address);
                int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
                addfd(epollfd, connfd);
            }
            else if(sockfd == udpfd)
            {
                char buf[UDP_BUFFER_SIZE];
                memset(buf, '\0', UDP_BUFFER_SIZE);
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof(client_address);
                ret = recvfrom(udpfd, buf, UDP_BUFFER_SIZE - 1, 0, (struct sockaddr*)&client_address, &client_addrlength);
                printf("UDP:    %s", buf);
                if(ret > 0)
                {
                    sendto(udpfd, buf, UDP_BUFFER_SIZE - 1, 0, (struct sockaddr*)&client_address, client_addrlength);
                }
            }
            else if(events[i].events & EPOLLIN)
            {
                char buf[TCP_BUFFER_SIZE];
                while(1)
                {
                    memset(buf, '\0', TCP_BUFFER_SIZE);
                    ret = recv(sockfd, buf, TCP_BUFFER_SIZE - 1, 0);
                    if(ret < 0)
                    {
                        if((errno == EAGAIN)||(errno == EWOULDBLOCK))
                        {
                            break;
                        }
                        close(sockfd);
                        break;
                    }
                    else if(ret == 0)
                    {
                        close(sockfd);
                    }
                    else
                    {
                        printf("TCP:    %s", buf);
                        send(sockfd, buf, ret, 0);
                    }
                }
            }
            else
            {
                printf("something else happened\n");
            }
        }
    }
    close(listenfd);
    return 0;
}
```















