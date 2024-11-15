# 相关类
------
day07
------
## `Server`

  `EventLoop`
  持有一个`Epoll`对象
    对应一个loop，其中执行如此的循环：
        先poll，然后挨个调用返回的若干`Channel`对象的`handleEvent`方法
    提供接口`updateChannel(Channel*)`，用于往`Epoll`对象上添加监听事件

  `Acceptor`
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
和连接socket强相关，封装监听socket的行为，以及提供接口访问其事件：
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
···





## 














