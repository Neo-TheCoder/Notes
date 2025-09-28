# 异步IO
# Proactor模型
相比reactor模型，使用了`异步I/O`

# 设计理念
异步I/O的核心
是`任务调度器` + `事件循环`(event loop)

用户只管往`io_context`丢可调用对象、注册对网络/定时器事件的监听
```cpp
io_.post(handler);
```

# 总结
各种异步API的前提就是`io_context对象`，一开始要先让`io_context对象`run在另外的线程里，这样的run可以多线程去调，显然又会涉及到任务队列，又必然涉及到互斥操作，就会涉及到锁。

# asio demo(cpp11)
先看下asio的基本用法

## chat
### 协同工作的对象
`io_context`，`socket`

### 基本操作
```cpp
// 1. 构造io_context，再用io_context来构造socket

// 2. 单线程/多线程 执行run
  io_context.run();

// 3. async_connect()
async_connect(socket_, endpoints, []() {
  // ...
  do_read_header();
});

// 4.1 向io_context_提交写任务
// 4.2 写任务只做一件事：调async_write()
    // 在主线程中调以下函数（因为一次只提交一个任务，所以不会出现多线程并发访问资源的问题）
    boost::asio::post(io_context_,
        [this, msg]()
        {
          bool write_in_progress = !write_msgs_.empty();    // 这里的write_msgs_是std::deque，无需加锁
          write_msgs_.push_back(msg);
          if (!write_in_progress)
          {
            do_write();
          }
        });

    void do_write()
    {
      boost::asio::async_write(socket_,                                 // socket
          boost::asio::buffer(write_msgs_.front().data(),               // buffer
            write_msgs_.front().length()),                              // size
          [this](boost::system::error_code ec, std::size_t /*length*/)  // handler
          {
            if (!ec)
            {
              write_msgs_.pop_front();
              if (!write_msgs_.empty())
              {
                do_write();   // 确保串行发送
              }
            }
            else
            {
              socket_.close();
            }
          });
    }

```




# `class io_context`具体实现
为了平台兼容性，使用`impl模式`
```cpp
typedef scheduler io_context_impl;

class io_context
  : public execution_context
{
    typedef detail::io_context_impl impl_type;

    // ...
    template <typename Function, typename OtherAllocator>
    void post(BOOST_ASIO_MOVE_ARG(Function) f,
        const OtherAllocator& a) const;

    BOOST_ASIO_DECL count_type run();
    BOOST_ASIO_DECL count_type poll();
    void reset();

        private:
        impl_type& impl_;
};
```
所以linux上，实际实现都在`scheduler`中

















