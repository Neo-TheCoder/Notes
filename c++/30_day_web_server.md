# 相关类
------
day07, day08
------

## `Server`
在构造的时候，新建一个`Acceptor`对象（视作`监听socket`，负责其行为，而一个`Channel`）

  ### `EventLoop`
  持有一个`Epoll`对象
    对应一个loop，其中执行如此的循环：
        先poll，然后挨个调用返回的若干`Channel`对象的`handleEvent`方法
        提供接口`updateChannel(Channel*)`，用于往`Epoll`对象上添加监听事件
```cpp
class EventLoop
{
private:
    Epoll *ep;
    bool quit;
public:
    EventLoop();
    ~EventLoop();

    void loop();
    void updateChannel(Channel*);
};
```
  ### `Epoll`
```cpp
class Epoll
{
private:
    int epfd;
    struct epoll_event *events;
public:
    Epoll();
    ~Epoll();

    // void addFd(int fd, uint32_t op);
    void updateChannel(Channel*);
    // std::vector<epoll_event> poll(int timeout = -1);
    std::vector<Channel*> poll(int timeout = -1);
};
```

  ### `Acceptor`
```cpp
class Acceptor
{
private:
    EventLoop *loop;
    Socket *sock;
    Channel *acceptChannel;
    std::function<void(Socket*)> newConnectionCallback;
public:
    Acceptor(EventLoop *_loop);
    ~Acceptor();
    void acceptConnection();
    void setNewConnectionCallback(std::function<void(Socket*)>);
};
```
  可理解成一个组件，负责干活：监听新的连接请求，并在接收到请求后创建一个新的连接对象。
    **每个对应一个`监听socket`**
    持有`EventLoop`指针
    `Socket`对象、`InetAddress`对象、`Channel`对象
    存储一个回调：`void acceptConnection();`，用于传递给`Channel`对象

  `Server`定义了两个函数：
  ```cpp
    /*
        监听到连接socket上的可读事件时触发
        被注册给Channel对象
    */
    void handleReadEvent(int);

    /*
        监听socket上发生可读事件时触发
        被注册给Acceptor对象，注册之后，再被注册给Channel对象
    */
    void newConnection(Socket *serv_sock);
  ```

## `Channel`
一个文件描述符封装成一个`Channel`类，一个Channel类自始至终只负责`一个文件描述符`，对不同的服务、不同的事件类型，都可以在类中进行不同的处理，而不是仅仅拿到一个`int`类型的文件描述符。
一个`Channel`和一个`epoll`以及`fd`绑定
```cpp
class Channel
{
private:
    EventLoop *loop;
    int fd;
    uint32_t events;
    uint32_t revents;
    bool inEpoll;
    std::function<void()> callback;
public:
    Channel(EventLoop *_loop, int _fd);
    ~Channel();

    void handleEvent();
    void enableReading();

    int getFd();
    uint32_t getEvents();
    uint32_t getRevents();
    bool getInEpoll();
    void setInEpoll();

    // void setEvents(uint32_t);
    void setRevents(uint32_t);
    void setCallback(std::function<void()>);
};
```

提供接口访问fd上的事件：
```cpp
    void handleEvent();
    void enableReading();

    int getFd();
    uint32_t getEvents();
    uint32_t getRevents();
    bool getInEpoll();
    void setInEpoll();

    // void setEvents(uint32_t);
    void setRevents(uint32_t);
```

## `Connection`
把`TCP连接`进行抽象
```cpp
class Connection
{
private:
    EventLoop *loop;
    Socket *sock;
    Channel *channel;   // 其实是组合关系，把业务函数：echo，注册给Channel，在handleEvent时调用
    std::function<void(Socket*)> deleteConnectionCallback;
public:
    Connection(EventLoop *_loop, Socket *_sock);
    ~Connection();

    void echo(int sockfd);
    void setDeleteConnectionCallback(std::function<void(Socket*)>);
};
```
相比day07，在构造`Connection`对象时，新建一个`Channel`对象
而`Acceptor`::`acceptConnection`直接负责建立连接的操作：（day07是`Server`定义建立连接函数，注册到`Acceptor`）而`Server`仍然注册`newConnection`函数给`Acceptor`，用于`new`一个`Connection`对象，并且注册`deleteConnection`函数
day08的`Acceptor`构造时负责：
创建监听socket对象、绑定、监听、新建`Channel`、设置监听socket发生可读事件时的回调：事件的回调：（由`Server`注册）新建`Connection`对象，添加到`Server`维护的map中，同时给`Connection`注册移除callback，负责从map中移除

```cpp
void Acceptor::acceptConnection() {
    InetAddress *clnt_addr = new InetAddress();
    Socket *clnt_sock = new Socket(sock->accept(clnt_addr));
    printf("new client fd %d! IP: %s Port: %d\n", clnt_sock->getFd(), inet_ntoa(clnt_addr->getAddr().sin_addr), ntohs(clnt_addr->getAddr().sin_port));
    clnt_sock->setnonblocking();
    newConnectionCallback(clnt_sock);
    delete clnt_addr;
}
```
而`Connection`类主要负责：
  新建`Channel`对象（该对象不直接和`Epoll`交互，而是和`EventLoop`产生关联），设置连接socket可读时的回调函数（其实这些回调最终都是注册到`Channel`上的），也就是业务逻辑函数，此处是回显（其中不仅打印收到的数据，还负责在断开连接时，从`Server`所维护的map中移除，`Connection`析构时，同样析构`Channel`和`Socket`（此处的socket显然是连接socket，它在Acceptor所注册的监听socket发生可读事件时，触发回调：`acceptConnection`：创建连接socket以传入）

# day09
在`Connection`，增加读写缓冲区，在`echo`时，输入、读取、清空
```cpp
class Connection
{
private:
    EventLoop *loop;
    Socket *sock;
    Channel *channel;
    std::function<void(Socket*)> deleteConnectionCallback;
    std::string *inBuffer;
    Buffer *readBuffer;
public:
    Connection(EventLoop *_loop, Socket *_sock);
    ~Connection();
    
    void echo(int sockfd);
    void setDeleteConnectionCallback(std::function<void(Socket*)>);
};
```

# day10
当EventLoop线程监听到事件发生，应该把相应事件分发到线程池处理
而之前都是在主线程进行处理：
```cpp
void EventLoop::loop() {
    while(!quit) {
    std::vector<Channel*> chs;
        chs = ep->poll();
        for(auto it = chs.begin(); it != chs.end(); ++it) {
            (*it)->handleEvent();
        }
    }
}
```

`每一个Reactor只应该负责事件分发而不应该负责事件处理`
在`EventLoop`中添加成员变量线程池，并且对外提供`addThread()`（以供`Channel`对象调用）
```cpp
void EventLoop::loop() {
    while(!quit) {
    std::vector<Channel*> chs;
        chs = ep->poll();
        for(auto it = chs.begin(); it != chs.end(); ++it) {
            (*it)->handleEvent();
        }
    }
}

void EventLoop::addThread(std::function<void()> func) {
    threadPool->add(func);
}
```

# day11
改造前：
```cpp
void ThreadPool::add(std::function<void()> func) {
    {
        std::unique_lock<std::mutex> lock(tasks_mtx);
        if(stop)
            throw std::runtime_error("ThreadPool already stop, can't add task any more");
        tasks.emplace(func);
    }
    cv.notify_one();
}
```

改造后：
```cpp
template<class F, class... Args>
auto ThreadPool::add(F&& f, Args&&... args) -> std::future<typename std::result_of<F(Args...)>::type> {
    using return_type = typename std::result_of<F(Args...)>::type;  //返回值类型

    auto task = std::make_shared< std::packaged_task<return_type()> >(  //使用智能指针
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)  //完美转发参数
        );  
        
    std::future<return_type> res = task->get_future();  // 使用期约
    {   //队列锁作用域
        std::unique_lock<std::mutex> lock(tasks_mtx);   //加锁

        if(stop)
            throw std::runtime_error("enqueue on stopped ThreadPool");

        tasks.emplace([task](){ (*task)(); });  //将任务添加到任务队列
    }
    cv.notify_one();    //通知一次条件变量
    return res;     //返回一个期约
}
```
对于`Acceptor`而言，处理连接是很快的，单线程也是可以的，不一定要塞到线程池处理；并且，也不必使用epoll ET模式（`Acceptor`没有显式调用`channel->useET();`，则是LT模式）



# day12 主从`Reactor`
在上一天的教程，我们实现了一种最容易想到的多线程Reactor模式，即`将每一个Channel的任务分配给一个线程执行`。
这种模式有很多缺点，逻辑上也有不合理的地方。
比如当前版本线程池对象被`EventLoop`所持有，这显然是不合理的，线程池显然应该由服务器类来管理，不应该和事件驱动产生任何关系。
如果强行将线程池放进`Server`类中，由于`Channel`类只有`EventLoop`对象成员，使用线程池则需要注册回调函数，十分麻烦。
今天我们将采用`主从Reactor多线程模式`，也是大多数高性能服务器采用的模式，即陈硕《Linux多线程服务器编程》书中的one loop per thread模式。

此模式的特点为：
1. 服务器一般只有一个`main Reactor`，有很多个`sub Reactor`。
2. 服务器管理一个线程池，每一个sub Reactor由一个线程来负责`Connection`上的事件循环，事件执行也在这个线程中完成。
3. main Reactor只负责`Acceptor`建立新连接，然后`将这个连接分配给一个sub Reactor`。
```cpp
class Server {
private:
    EventLoop *mainReactor;     //只负责接受连接，然后分发给一个subReactor。这是通过把Server定义的回调newConnection，注册进Acceptor对象，然后Acceptor再注册给Channel
    Acceptor *acceptor;                     //连接接受器
    std::map<int, Connection*> connections; //TCP连接
    std::vector<EventLoop*> subReactors;    //负责处理事件循环！！！
    ThreadPool *thpool;     //线程池
};
```
在构造服务器时：
```cpp
Server::Server(EventLoop *_loop) : mainReactor(_loop), acceptor(nullptr) {
    acceptor = new Acceptor(mainReactor);   //Acceptor 由且只由mainReactor负责
    std::function<void(Socket*)> cb = std::bind(&Server::newConnection, this, std::placeholders::_1);
    acceptor->setNewConnectionCallback(cb);

    int size = std::thread::hardware_concurrency();     //线程数量，也是subReactor数量
    thpool = new ThreadPool(size);      //新建线程池
    for(int i = 0; i < size; ++i) {
        subReactors.push_back(new EventLoop());     //每一个线程中执行一个EventLoop
    }

    for(int i = 0; i < size; ++i) {
        std::function<void()> sub_loop = std::bind(&EventLoop::loop, subReactors[i]);   // 相当于创建函数：subReactors[i]->loop();
        thpool->add(sub_loop);      //开启所有线程的事件循环
    }
}
```
每个从reactor线程（数量和核心数保持一致）负责什么？
--> 每个从reactor线程都执行以下逻辑：
    对epoll返回的各个channel事件调用回调：`handleEvent()`
```cpp
void EventLoop::loop() {
    while(!quit) {
    std::vector<Channel*> chs;
        chs = ep->poll();
        for(auto it = chs.begin(); it != chs.end(); ++it) {
            (*it)->handleEvent();
        }
    }
}
```
主reactor是如何进行分发的？
--> 监听socket每发生一个可读事件/建立一个连接，
```cpp
void Server::newConnection(Socket *sock) {
    if(sock->getFd() != -1) {
        int random = sock->getFd() % subReactors.size();    // 选择要分配的从reactor线程
        Connection *conn = new Connection(subReactors[random], sock);   // Connection -- EventLoop，是聚合关系，构造Connection需要传入EventLoop*，用于构造Channel对象
        std::function<void(int)> cb = std::bind(&Server::deleteConnection, this, std::placeholders::_1);
        conn->setDeleteConnectionCallback(cb);
        connections[sock->getFd()] = conn;
    }
}
```

当连接socket发生可读事件时，触发注册到channel上的业务逻辑回调（`echo`）
每有一个连接socket，就新建一个Channel对象，注册echo回调
```cpp
Connection::Connection(EventLoop *_loop, Socket *_sock) : loop(_loop), sock(_sock), channel(nullptr), inBuffer(new std::string()), readBuffer(nullptr) {
    channel = new Channel(loop, sock->getFd());
    channel->enableRead();
    channel->useET();
    std::function<void()> cb = std::bind(&Connection::echo, this, sock->getFd());
    channel->setReadCallback(cb);
    channel->setUseThreadPool(true);
    readBuffer = new Buffer();
}
```

# day14
`Server`新定义了一个函数：`OnConnect()`
```cpp
void Server::OnConnect(std::function<void(Connection *)> fn) { on_connect_callback_ = std::move(fn); }

void Server::NewConnection(Socket *sock) {
  ErrorIf(sock->GetFd() == -1, "new connection error");
  uint64_t random = sock->GetFd() % sub_reactors_.size();
  Connection *conn = new Connection(sub_reactors_[random], sock);
  std::function<void(Socket *)> cb = std::bind(&Server::DeleteConnection, this, std::placeholders::_1);
  conn->SetDeleteConnectionCallback(cb);
  conn->SetOnConnectCallback(on_connect_callback_);
  connections_[sock->GetFd()] = conn;
}

void Connection::SetOnConnectCallback(std::function<void(Connection *)> const &callback) {
  on_connect_callback_ = callback;
  channel_->SetReadCallback([this]() { on_connect_callback_(this); });
}
```
注意到，`Connection`新增接口：`SetReadCallback()`

如此，`echo服务器`理论上只需要这几行代码：
```cpp
int main() {
  EventLoop *loop = new EventLoop();
  Server *server = new Server(loop);
  server->OnConnect([](Connection *conn) {  // 业务逻辑
    conn->Read();
    std::cout << "Message from client " << conn->GetSocket()->GetFd() << ": " << conn->ReadBuffer() << std::endl;
    if (conn->GetState() == Connection::State::Closed) {
      conn->Close();
      return;
    }
    conn->SetSendBuffer(conn->ReadBuffer());    // 把read_buffer的数据拷贝到send_buffer
    conn->Write();  // 调用write，借助socket，把send_buffer的数据发送出去
  });
  loop->Loop(); // 开始事件循环
  delete server;
  delete loop;
  return 0;
}
```



# day15
## echo服务器
```cpp
// Server类
  server->OnMessage([](Connection *conn) {
    std::cout << "Message from client " << conn->ReadBuffer() << std::endl;
    if (conn->GetState() == Connection::State::Connected) {
      conn->Send(conn->ReadBuffer());
    }
  });


// Connection类
  conn->SetOnMessageCallback(on_message_callback_);

  void Connection::Business() {
    Read();
    on_message_callback_(this);
  }

void Connection::SetOnMessageCallback(std::function<void(Connection *)> const &callback) {
  on_message_callback_ = callback;
  std::function<void()> bus = std::bind(&Connection::Business, this);
  channel_->SetReadCallback(bus);
}
```

### `Server`注册回调
```cpp
void Server::NewConnection(Socket *sock) {
  if (sock->GetFd() == -1) {
    throw Exception(ExceptionType::INVALID_SOCKET, "New Connection error, invalid client socket!");
  }

  // ErrorIf(sock->GetFd() == -1, "new connection error");
  uint64_t random = sock->GetFd() % sub_reactors_.size();
  Connection *conn = new Connection(sub_reactors_[random], sock);

  std::function<void(Socket *)> cb = std::bind(&Server::DeleteConnection, this, std::placeholders::_1);
  conn->SetDeleteConnectionCallback(cb);

  // conn->SetOnConnectCallback(on_connect_callback_);
  conn->SetOnMessageCallback(on_message_callback_);     // 外部调用OnMessage()接口传入回调函数
  connections_[sock->GetFd()] = conn;

  if (new_connect_callback_)
    new_connect_callback_(conn);
}

Server::Server(EventLoop *loop) : main_reactor_(loop), acceptor_(nullptr), thread_pool_(nullptr) {
// ...
  acceptor_ = new Acceptor(main_reactor_);
  std::function<void(Socket *)> cb = std::bind(&Server::NewConnection, this, std::placeholders::_1);
  acceptor_->SetNewConnectionCallback(cb);
// ...
}

/*
    以下接口的调用，是在主线程main执行的，注册完回调才调用：loop->Loop();    才开始epoll_wait，也就是说，回调尚不会触发
    Server构造时，即一路把NewConnectionCallback函数传入acceptor、传入channel，而SetNewConnectionCallback注册的函数在实际触发时，实际上是会读取Server::new_connect_callback_这个std::function<void(Connection *)>类型的成员变量的
*/

// 新建立连接时触发的函数：
server->NewConnect(
    [](Connection *conn) { std::cout << "New connection fd: " << conn->GetSocket()->GetFd() << std::endl; });

// 收到client数据时触发的函数：对外提供OnMessage()接口：
// 注意，这里的lambda，不需要捕获任何变量
server->OnMessage([](Connection *conn) {
std::cout << "Message from client " << conn->ReadBuffer() << std::endl;
if (conn->GetState() == Connection::State::Connected) {
    conn->Send(conn->ReadBuffer());
}
});

```

### `Channel`发生了什么变化？
```cpp
class Channel {
 public:
  Channel(EventLoop *loop, Socket *socket);
  ~Channel();

  DISALLOW_COPY_AND_MOVE(Channel);  // copy, move = delete

  void HandleEvent();   // 分别触发可读回调，可写回调
  void EnableRead();    // 
  void EnableWrite();    

  Socket *GetSocket();
  int GetListenEvents();
  int GetReadyEvents(); // ！
  bool GetExist();      // ！设置Channel是否开通/关闭
  void SetExist(bool in = true);    // ！
  void UseET(); // 进行运算：   |= ET

  void SetReadyEvents(int ev);  // epoll返回时，set
  void SetReadCallback(std::function<void()> const &callback);  // set可读回调
  void SetWriteCallback(std::function<void()> const &callback); // set可写回调

  static const int READ_EVENT;   // NOLINT
  static const int WRITE_EVENT;  // NOLINT
  static const int ET;           // NOLINT

 private:
  EventLoop *loop_;
  Socket *socket_;
  int listen_events_{0};
  int ready_events_{0};
  bool exist_{false};
  std::function<void()> read_callback_;
  std::function<void()> write_callback_;
};
```



# day16
```cpp
class TcpServer {
 public:
    // ......

 private:
  std::unique_ptr<EventLoop> main_reactor_; // 外部new
  std::unique_ptr<Acceptor> acceptor_;  // 由Server自己管理

  std::unordered_map<int, std::unique_ptr<Connection>> connections_;
  std::vector<std::unique_ptr<EventLoop>> sub_reactors_;

  std::unique_ptr<ThreadPool> thread_pool_; // 由Server自己管理

  std::function<void(Connection *)> on_connect_;
  std::function<void(Connection *)> on_recv_;
};

class Channel {
 public:

    // ......

 private:
  int fd_;
  EventLoop *loop_; // 指针，表示仅访问、不持有（不new，不delete）
  short listen_events_;
  short ready_events_;
  bool exist_;
  std::function<void()> read_callback_;
  std::function<void()> write_callback_;
};
```





































