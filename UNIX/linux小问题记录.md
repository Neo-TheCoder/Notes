
# malloc分配方式
在C/C++中，`malloc` 函数用于动态分配内存空间。具体分配内存的方式是在堆（Heap）中找到一块足够大小的连续空闲内存块，然后将该内存块标记为已使用，并返回该内存块的起始地址给调用者。`malloc` 函数的原型为：

```c
void* malloc(size_t size);
```

`malloc` 函数接受一个参数 `size`，表示需要分配的内存空间大小（以字节为单位）。它会在堆中找到一块大小不小于 `size` 的空闲内存块，如果找到合适的内存块，则返回该内存块的起始地址；如果找不到足够大的内存块，则返回 `NULL`。

`malloc` 函数的一些缺点包括：
1. **内存泄漏**：使用 `malloc` 分配内存后，需要手动调用 `free` 函数来释放内存。如果忘记释放内存，就会导致内存泄漏，使得程序占用的内存不断增加。
2. **无法自动调整内存大小**：`malloc` 分配的内存大小是固定的，无法动态调整。如果需要调整内存大小，需要重新分配内存并拷贝数据，比较繁琐。
3. **没有类型检查**：`malloc` 返回的是 `void*` 类型的指针，需要手动进行类型转换，容易出现类型错误。

## 传一个指针进来，它如何确定具体要清理多少空间？
当传入一个指针进来时，程序无法确定具体要清理多少空间。通常情况下，需要在调用 `malloc` 分配内存的同时记录下分配的大小，以便在释放内存时使用。一种常见的做法是在分配内存时多分配一些空间用于存储分配的大小，然后在释放内存时先读取这个大小，再释放相应大小的内存。这样可以确保正确地释放分配的内存空间，避免内存泄漏和越界访问等问题。




# linux如何从用户态转到内核态？
在Linux系统中，从用户态（User Mode）转换到内核态（Kernel Mode）通常涉及系统调用（System Call）或者异常（Exception）的处理。以下是一般情况下的用户态到内核态的转换方式：

1. **系统调用（System Call）**：
   - 用户程序通过`调用库函数`（如C标准库中的函数）发起系统调用，比如：打开文件、读写文件等操作。
   - `库函数`会将`系统调用号`和`参数`传递给内核，然后触发`软中断`（Software Interrupt）。
   - `操作系统内核`会根据`系统调用号`找到对应的`处理函数`，并在**内核态**执行该函数。
   - 在`内核态`执行完`系统调用`后，将结果返回给用户程序。

2. **异常（Exception）**：
   - 当用户程序执行一些特殊的指令或者发生一些特定的情况时（如除零错误、访问非法内存等），会触发异常。
   - 异常会导致`处理器`从`用户态`切换到`内核态`，并跳转到`内核`中预先定义好的`异常处理程序`。
   - 内核态的异常处理程序会根据异常类型进行相应的处理，可能会终止用户程序、发送信号等。

总的来说，无论是系统调用还是异常，都会导致**处理器**从**用户态切换到内核态**，这涉及到特权级别的切换和硬件的支持。
在内核态中，操作系统具有更高的权限和更广泛的访问权限，可以执行一些用户态无法执行的操作，如访问硬件设备、管理内存等。


# 一次I/O操作的完整流程
一次I/O（Input/Output）操作的完整流程涉及用户程序、操作系统内核和硬件设备之间的交互。以下是一次I/O 操作的典型流程：

1. **用户程序发起系统调用**：
   - 用户程序通过调用相应的系统调用接口（如`read()`或`write()`）请求进行I/O 操作。
   - 系统调用会将请求传递给操作系统内核。

2. **内核准备I/O 操作**：
   - 操作系统内核接收到用户程序的I/O 请求后，会进行相应的准备工作，包括确定要操作的设备、数据传输方向等。

3. **内核发起设备控制命令**：
   - 内核根据用户程序的请求向相应的设备控制器发送命令，要求设备执行读取或写入操作。

4. **设备执行I/O 操作**：
   - 设备控制器接收到内核发送的命令后，开始执行实际的 I/O 操作，包括从设备读取数据或向设备写入数据。

5. **数据传输**：
   - 如果是读取操作，设备会将数据读取到设备缓冲区中；如果是写入操作，设备会将数据从设备缓冲区写入到设备中。

6. **中断通知**：
   - 当设备完成数据传输后，会向设备控制器发送中断信号，通知操作系统内核数据已准备好。

7. **内核处理中断**：
   - 操作系统内核接收到设备的中断信号后，会进行中断处理程序的调用，处理设备完成的I/O 操作。

8. **数据传递给用户空间**：
   - 如果是读取操作，内核会将数据从设备缓冲区复制到用户程序的内存空间；如果是写入操作，内核会将用户程序提供的数据复制到设备缓冲区。

9. **系统调用返回**：
   - 内核将操作结果返回给用户程序，系统调用完成，用户程序继续执行后续操作。

整个I/O 操作的流程涉及用户程序、操作系统内核和硬件设备之间的协同工作，通过系统调用、中断处理等机制实现数据的传输和交互。



# DMA
DMA（Direct Memory Access，直接内存访问）是计算机系统中一种数据传输方式，允许外部设备直接访问计算机内存，而无需经过中央处理器（CPU）的介入。以下是关于DMA的详细说明：

1. **工作原理**：
   - 在传统的I/O操作中，数据传输通常需要通过CPU来控制，即CPU从外部设备读取数据或向外部设备写入数据。而使用DMA时，外部设备可以直接与内存进行数据传输，而不需要CPU的直接参与。
   - DMA控制器是负责管理数据传输的硬件设备，它可以独立地访问系统总线，直接和内存进行数据交换。

2. **优点**：
   - 提高系统性能：通过使用DMA，可以减少CPU的负担，释放CPU用于执行其他任务，从而提高系统的整体性能。
   - 高效的数据传输：DMA能够实现高速、连续的数据传输，提高数据传输的效率和速度。

3. **应用**：
   - DMA常用于大数据量的数据传输，比如网络数据包的接收和发送、磁盘数据的读写等。
   - 在图形处理、音频处理等需要高速数据传输的应用中，DMA也发挥着重要作用。

4. **工作流程**：
   - 外部设备向DMA控制器发送数据传输请求。
   - DMA控制器获取总线控制权，直接访问内存，将数据传输到指定内存地址或从指定内存地址读取数据。
   - 数据传输完成后，DMA控制器会向外部设备发送传输完成的信号。

总的来说，DMA是一种通过硬件实现的数据传输方式，能够提高系统性能，减少CPU的负担，适用于需要高效数据传输的场景，如大数据量的数据传输和高速数据处理。



# CGroup
linux中，有将进程分组的概念以及需求，经常需要追踪一组进程的内存和IO的使用情况。

CGroup的功能：**对一组进程进行分组，以及对进程进行统一的资源监控和限制**
用户层看来，cgroup就是把系统中的进程组织成若干树，**树的每个节点是一个进程组**，每棵树和一个或多个`subsystem`关联，树的作用是将进程进行分组，而`subsystem`的作用就是对这些组进行操作。


## 组成部分
### subsystem
一个subsystem就是一个**内核模块**，他被关联到一颗cgroup树之后，就会在树的每个节点(进程组) 上做具体的操作。
subsystem经常被称作"resource controller”，因为它主要被用来调度或者限制每个进程组的资源，但是这个说法不完全准确，因为有时我们将进程分组只是为了做一些监控，观察一下他们的状态，比如：perf event subsystem。


### hierarchy
一个hierarchy可以理解为一棵cgroup树，树的每个节点就是一个进程组，每棵树都会与零到多个subsystem关联。
系统中可以有很多颗cgroup树，每棵树都和不同的subsystem关联，一个进程可以属于多颗树，即**一个进程可以属于多个进程组，只是这些进程组和不同的subsystem关联**。

目前Linux支持12种subsystem，如果不考虑不与任何subsystem关联的情况 (systemd就属于这种情况)，Linux里面最多可以建12颗cgroup树，每棵树关联一个subsystem，当然也可以只建一棵树，然后让这棵树关联所有的subsystem。
当一颗cgroup树不和任何subsystem关联的时候，意味着这棵树只是将进程进行分组，至于要在分组的基础上做些什么，将由应用程序自己决定，systemd就是一个这样的例子。


# `epoll`

在这里，`"事件"`指的是：**发生在文件描述符上的某种状态变化或操作**，例如数据可读、数据可写、连接建立等。
在使用 I/O 复用模型（如 epoll）时，程序会监听多个文件描述符，当某个文件描述符上发生了特定的事件时，操作系统会通知程序，程序可以通过处理这些事件来进行相应的操作。

事件和文件描述符的关系可以通过一个简单的例子来说明：
假设有一个服务器程序，使用 epoll 来同时监听多个客户端的连接。
每个客户端连接都会有一个对应的文件描述符，服务器程序会将这些文件描述符添加到 epoll 实例中进行监听。
当客户端发送数据到服务器时，服务器程序会收到数据可读的事件，然后通过相应的文件描述符来处理这些数据。
具体来说，假设有两个客户端连接，分别对应文件描述符 fd1 和 fd2。服务器程序使用 epoll 监听这两个文件描述符，当 fd1 上有数据可读时，操作系统会将数据可读的事件通知给服务器程序，服务器程序通过 fd1 来接收和处理客户端1发送的数据。同理，当 fd2 上有数据可读时，操作系统会将数据可读的事件通知给服务器程序，服务器程序通过 fd2 来接收和处理客户端2发送的数据。
因此，事件和文件描述符之间的关系是：事件是发生在特定文件描述符上的状态变化或操作，程序通过监听文件描述符上的事件来实现对不同连接或操作的处理。

注意：只要有`任何一个事件`发生，`epoll_wait`就会立即返回。
```cpp
#include <iostream>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <cstring>

# ifdef ABC    // epoll.h中的结构体定义
typedef union epoll_data
{
  void *ptr;   // 指向用户自定义的、任意类型的数据
  int fd;   // 表示文件描述符
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;

struct epoll_event
{
  uint32_t events;	/* Epoll events */   // 定义事件类型
  epoll_data_t data;	/* User data variable */
} __EPOLL_PACKED;

# endif

struct Data {
    int id;
    char message[256];
};

void* threadB(void* arg) {
    int* sockfd = (int*)arg;
    struct epoll_event event;   // epoll_event类型用于描述事件：事件类型 / 事件数据
    struct epoll_event events[1];
    int epollfd = epoll_create(1);  // 创建epoll实例，监听的文件描述符数量设置为1
    event.events = EPOLLIN; // EPOLLIN表示关注可读事件,文件描述符上有数据可读
    event.data.fd = *sockfd;    // 设置了event结构体的data字段，存储了要监听的文件描述符，即socket
    epoll_ctl(epollfd, EPOLL_CTL_ADD, *sockfd, &event);     // 这行代码将线程B的socket文件描述符添加到epoll实例中进行监听
    // epoll_ctl函数用于控制epoll实例，EPOLL_CTL_ADD表示添加一个新的文件描述符到epoll实例中，*sockfd是要添加的文件描述符，&event是描述事件的结构体。

    while (true) {
        int nfds = epoll_wait(epollfd, events, 1, -1);  // -1 表示不设置超时时间，nfds存储了发生事件的文件描述符的数量（既然在epoll_ctl里设置了要监听的文件描述符，为什么还要判断：因为实际情况中，会同时监听多个文件描述符）
        for (int i = 0; i < nfds; ++i) {
            if (events[i].data.fd == *sockfd) {     // 遍历发生事件的文件描述符，判断是否是线程B的socket文件描述符
                Data data;
                recv(*sockfd, &data, sizeof(data), 0);
                std::cout << "Received data from A - ID: " << data.id << ", Message: " << data.message << std::endl;
            }
        }
    }
}

int main() {
    int sv[2];
    socketpair(AF_UNIX, SOCK_STREAM, 0, sv);    // 使用socketpair函数创建了一个全双工的通信管道

    pthread_t tid;
    pthread_create(&tid, NULL, threadB, &sv[1]);

    Data data = {123, "Hello from A"};
    send(sv[0], &data, sizeof(data), 0);

    pthread_join(tid, NULL);
    close(sv[0]);
    close(sv[1]);

    return 0;
}

```


## socket pair && epoll
```cpp
#include<vector>
#include <iostream>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <cstring>

# ifdef ABC
typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;

struct epoll_event
{
  uint32_t events;	/* Epoll events */
  epoll_data_t data;	/* User data variable */
} __EPOLL_PACKED;

# endif

struct Data {
    int id;
    char message[256];
};

void* threadB(void* arg) {
    int* sockfd = (int*)arg;
    struct epoll_event event;   // epoll_event类型用于描述事件：事件类型 / 事件数据
    struct epoll_event events[1];
    int epollfd = epoll_create(1);  // 创建epoll实例，监听的文件描述符数量设置为1
    event.events = EPOLLIN; // EPOLLIN表示关注可读事件,文件描述符上有数据可读
    event.data.fd = *sockfd;    // 设置了event结构体的data字段，存储了要监听的文件描述符，即socket
    epoll_ctl(epollfd, EPOLL_CTL_ADD, *sockfd, &event);     // 这行代码将线程B的socket文件描述符添加到epoll实例中进行监听
    // epoll_ctl函数用于控制epoll实例，EPOLL_CTL_ADD表示添加一个新的文件描述符到epoll实例中，*sockfd是要添加的文件描述符，&event是描述事件的结构体。

    while (true) {
        int nfds = epoll_wait(epollfd, events, 1, -1);  // -1 表示不设置超时时间，nfds存储了发生事件的文件描述符的数量（既然在epoll_ctl里设置了要监听的文件描述符，为什么还要判断：因为实际情况中，会同时监听多个文件描述符）
        for (int i = 0; i < nfds; ++i) {
            if (events[i].data.fd == *sockfd) {     // 遍历发生事件的文件描述符，判断是否是线程B的socket文件描述符
                Data data;
                recv(*sockfd, &data, sizeof(data), 0);
                std::cout << "Received data from A - ID: " << data.id << ", Message: " << data.message << std::endl;
            }
        }
    }
}

int main() {
    int sv[2];
    socketpair(AF_UNIX, SOCK_STREAM, 0, sv);    // 使用socketpair函数创建了一个全双工的通信管道

    pthread_t tid;
    pthread_create(&tid, NULL, threadB, &sv[1]);

    Data data = {123, "Hello from A"};
    send(sv[0], &data, sizeof(data), 0);

    pthread_join(tid, NULL);
    close(sv[0]);
    close(sv[1]);

    return 0;
}
```


# 关于调度和优先级
## 概览
> 从**用户空间**来看，进程优先级就是`nice value`和`scheduling priority`；
对应到**内核**，有`静态优先级`、`realtime优先级`、`归一化优先级`和`动态优先级`等概念
### 用户空间的视角
在用户空间，进程优先级有两种含义：`nice value`和`scheduling priority`。
对于`普通进程`而言，进程优先级就是`nice value`，从`-20`（优先级最高）～`19`（优先级最低），通过修改`nice value`可以改变普通进程获取`cpu资源`的比例。
随着实时需求的提出，进程又被赋予了另外一种属性`scheduling priority`，而这些进程被称为`实时进程`。
实时进程的优先级的范围可以通过`sched_get_priority_min`和`sched_get_priority_max`，对于linux而言，实时进程的`scheduling priority`的范围是`1`（优先级最低）～`99`（优先级最高）。
当然，普通进程也有`scheduling priority`，被设定为`0`。

### 内核中的实现
内核中，task struct中有若干和进程优先级有关的成员，如下：
```c
struct task_struct {
// ......
    int prio, static_prio, normal_prio;
    unsigned int rt_priority;
// ......
    unsigned int policy;
// ......
};
```
#### 静态优先级
`task_struct`中的`static_prio`成员。
我们称之静态优先级，其特点如下：
1. 值越小，进程优先级越高
2. `0 – 99`用于`实时进程`，`100 – 139`用于`普通进程`
3. 缺省值是 120
4. `用户空间`可以通过`nice()`或者`setpriority`对该值进行修改。通过`getpriority`可以获取该值。
5. 新创建的进程会`继承父进程`的`static priority`。
`静态优先级`是所有相关优先级的计算的起点，要么继承自父进程，要么用户空间自行设定。
一旦修改了静态优先级，那么`normal priority`和`动态优先级`都需要重新计算。

#### 实时优先级
task struct中的`rt_priority`成员表示该线程的`实时优先级`，也就是从用户空间的视角来看的`scheduling priority`。
`0`是`普通进程`，`1～99`是`实时进程`，99的优先级最高。

#### 归一化优先级
`task struct`中的`normal_prio`成员。
我们称之归一化优先级（normalized priority），它是根据`静态优先级、scheduling priority和调度策略`来计算得到，代码如下：
```cpp
static inline int normal_prio(struct task_struct *p)
{
    int prio;

    if (task_has_dl_policy(p))
        prio = MAX_DL_PRIO - 1;
    else if (task_has_rt_policy(p))
        prio = MAX_RT_PRIO -1 - p->rt_priority;
    else
        prio = __normal_prio(p);
    return prio;
}
```
对于这里的优先级，调度器需要综合考虑各种因素，例如`调度策略，nice value、scheduling priority`等，把这些factor全部考虑进来，归一化成一个数轴上的number，以此来表示其优先级，这就是normalized priority。
对于一个线程，其`normalized priority`的number`越小`，其`优先级越大`。
调度策略是`deadline`的进程比`RT进程`和`normal进程`的优先级还要高，因此它的归一化优先级是负数：`-1`。
如果采用`实时调度策略`，那么该线程的`normalized priority`和`rt_priority`相关。
task struct中的`rt_priority`成员是`用户空间视角的``实时优先级（scheduling priority）`，`MAX_RT_PRIO-1`是`99`，`MAX_RT_PRIO-1 - p->rt_priority`则翻转了实时进程的`scheduling priority`，最高优先级是0，最低是98。顺便说一句，normalized priority是99的情况是没有意义的。
**对于`普通进程`，`normalized priority`就是其静态优先级。**

#### 动态优先级
`task struct`中的`prio`成员表示了该线程的`动态优先级`，也就是`调度器在进行调度时候使用的那个优先级`。
动态优先级在运行时可以被修改，例如在处理`优先级翻转问题`的时候，系统可能会临时调升一个普通进程的优先级。
一般设定动态优先级的代码是这样的：`p->prio = effective_prio(p)`，具体计算动态优先级的代码如下：
```c
static int effective_prio(struct task_struct *p)
{
    p->normal_prio = normal_prio(p);
    if (!rt_prio(p->prio))
        return p->normal_prio;
    return p->prio;
}
```
`rt_prio`是一个根据`当前优先级`来确定是否是`实时进程`的函数，包括两种情况：
* 一种情况是该进程是`实时进程`，
* 调度策略是`SCHED_FIFO`或者`SCHED_RR`。
另外一种情况是人为的将该进程提升到`RT priority的区域`（例如在使用优先级继承的方法解决系统中优先级翻转问题的时候）。
在这两种情况下，我们都不改变其动态优先级，即`effective_prio`返回当前动态优先级`p->prio`。
其他情况，进程的`动态优先级`跟随`归一化的优先级`。


#### 典型数据流程分析
##### 用户空间设定`nice value`
用户空间设定nice value的操作，在内核中主要是`set_user_nice`函数实现的，无论是`sys_nice`或者`sys_setpriority`，在参数检查和权限检查之后都会调用`set_user_nice`函数，完成具体的设定。
代码如下：
```cpp
void set_user_nice(struct task_struct *p, long nice)
{
    int old_prio, delta, queued;
    unsigned long flags;
    struct rq *rq; 
    rq = task_rq_lock(p, &flags);
    if (task_has_dl_policy(p) || task_has_rt_policy(p)) { // －－－－－－－－－－－（1）
        p->static_prio = NICE_TO_PRIO(nice);
        goto out_unlock;
    }
    queued = task_on_rq_queued(p); // －－－－－－－－－－－－－－－－－－－（2）
    if (queued)
        dequeue_task(rq, p, DEQUEUE_SAVE);

    p->static_prio = NICE_TO_PRIO(nice); // －－－－－－－－－－－－－－－－（3）
    set_load_weight(p);
    old_prio = p->prio;
    p->prio = effective_prio(p);
    delta = p->prio - old_prio;

    if (queued) {
        enqueue_task(rq, p, ENQUEUE_RESTORE); // －－－－－－－－－－－－（2）
        if (delta < 0 || (delta > 0 && task_running(rq, p))) // －－－－－－－－－－－－（4）
            resched_curr(rq);
    }
out_unlock:
    task_rq_unlock(rq, p, &flags);
}
```
1. 如果是`实时进程或者deadline类型的进程`，那么`nice value`其实是没有什么实际意义的，不过我们还是设定其静态优先级，当然，这样的设定其实不会起到什么作用的，也不会实际改变调度器行为，因此直接返回，没有dequeue和enqueue的动作。
2. 在1中已经处理了调度策略是RT类和DEADLINE类的进程，因此，执行到这里，只可能是普通进程了，使用CFS算法。
  如果该task在run queue上（queued等于true），那么由于我们修改了nice value，调度器需要重新审视当前runqueue中的task。
  因此，我们需要将该task从rq中摘下，在重新计算优先级之后，再次插入该runqueue对应的runable task的红黑树中。
3. 最核心的代码就是`p->static_prio = NICE_TO_PRIO(nice);`这一句了，其他的都是side effect。
  比如说`load weight`。当cpu一刻不停的运算的时候，其load是100％，没有机会调度到`idle进程`休息一下。
  当系统中没有实时进程或者deadline进程的时候，所有的runnable的进程一起来瓜分cpu资源，以此不同的进程分享一个`特定比例的 cpu资源`，我们称之load weight。
  **`不同的nice value`对应`不同的cpu load weight`**，因此，当更改nice value的时候，也必须通过`set_load_weight`来更新该进程的cpu load weight。
  除了load weight，该线程的`动态优先级`也需要更新，这是通过`p->prio = effective_prio(p);`来完成的。
4. delta 记录了`新旧线程的动态优先级的差值`，当调试了该线程的优先级（delta < 0），那么有可能产生一个`调度点`（可能会引起`线程切换`），因此，调用resched_curr，给当前正在运行的task做一个标记，以便在返回用户空间的时候进行调度。此外，如果修改当前running状态的task的动态优先级，那么调降（delta > 0）意味着该进程有可能需要让出cpu，因此也需要`resched_curr`标记当前running状态的task需要reschedule。

##### 进程`缺省的`调度策略和调度参数
我们先思考这样的一个问题：在用户空间设定调度策略和调度参数之前，一个线程的`default scheduling policy`是什么呢？
这需要追溯到`fork`的时候（具体代码在`sched_fork`函数中），这个和task struct中`sched_reset_on_fork`设定相关。
如果没有设定这个flag，那么说明在fork的时候，子进程跟随父进程的调度策略，
如果设定了这个flag，则说明子进程的调度策略和调度参数不能继承自父进程，而是需要设定为`default`。
代码片段如下：
```c
int sched_fork(unsigned long clone_flags, struct task_struct *p)
{
// ……
    p->prio = current->normal_prio; //－－－－－－－－－－－－－－－－－－－（1）
    if (unlikely(p->sched_reset_on_fork)) {
        if (task_has_dl_policy(p) || task_has_rt_policy(p)) {//－－－－－－－－－－（2）
            p->policy = SCHED_NORMAL;
            p->static_prio = NICE_TO_PRIO(0);
            p->rt_priority = 0;
        } else if (PRIO_TO_NICE(p->static_prio) < 0)
            p->static_prio = NICE_TO_PRIO(0);

        p->prio = p->normal_prio = __normal_prio(p); //－－－－－－－－－－－－（3）
        set_load_weight(p); 
        p->sched_reset_on_fork = 0;
    }
// ……
}
```
1. `sched_fork`只是fork过程中的一个片段，在fork一开始，`dup_task_struct`已经复制了一个和父进程完全一个的`进程描述符（task struct）`，因此，如果没有步骤2中的重置，那么子进程是跟随父进程的调度策略和调度参数（各种优先级），当然，有时候为了解决PI问题而临时调升父进程的动态优先级，在fork的时候不宜传递到子进程中，因此这里重置了动态优先级。
2. 缺省的调度策略是`SCHED_NORMAL`，`静态优先级`等于`120（也就是说nice value等于0）`，`rt priority`等于`0（普通进程）`。
不管父进程如何，即便是deadline的进程，其fork的子进程也需要恢复到缺省参数。
3. **既然调度策略和静态优先级已经修改了，那么也需要更新`动态优先级`和`归一化优先级`**。
  此外，`load weight`也需要更新。
  一旦子进程中恢复到了缺省的调度策略和优先级，那么`sched_reset_on_fork`这个flag已经完成了历史使命，可以clear掉了。
OK，至此，我们了解了在fork过程中对调度策略和调度参数的处理，这里还是要追加一个问题：为何不一切继承父进程的调度策略和参数呢？为何要在fork的时候reset to default呢？
  **在linux中，对于每一个进程，我们都会进行`资源限制`**。
  例如对于那些`实时进程`，如果它持续消耗cpu资源而没有发起一次可以引起阻塞的系统调用，那么我们猜测这个realtime进程跑飞了，从而锁住了系统。
  对于这种情况，我们要进行干预，因此引入了`RLIMIT_RTTIME`这个per-process的资源限制项。
  但是，如果用户空间的`realtime进程`通过fork其实也可以绕开`RLIMIT_RTTIME`这个限制，从而肆意的攫取cpu资源。
  然而，机智的内核开发人员早已经看穿了这一切，为了防止实时进程“泄露”到其子进程中，`sched_reset_on_fork`这个flag被提出来。

##### 用户空间`设定`调度策略和调度参数
通过`sched_setparam`接口函数可以修改`rt priority`的调度参数，而通过`sched_setscheduler`功能会更强一些，不但可以设定`rt priority`，还可以设定`调度策略`。
而`sched_setattr`是一个集大成之接口，可以设定一个线程的调度策略以及该调度策略下的调度参数。
当然，对于内核，这些接口都通过`__sched_setscheduler`这个内核函数来完成对指定线程调度策略和调度参数的修改。
`__sched_setscheduler`分成两个部分：
* 首先进行安全性检查和参数检查，
* 其次进行具体的设定。
我们先看看安全性检查。
如果用户空间可以自由的修改调度策略和调度优先级，那么世界就乱套了，每个进程可能都想把自己的调度策略和优先级提升上去，从而获取足够的CPU 资源。
因此用户空间设定调度策略和调度参数要遵守一定的规则：如果没有CAP_SYS_NICE的能力，那么基本上该线程能被允许的操作只是降级而已。
例如从`SCHED_FIFO`修改成`SCHED_NORMAL`，或不修改`scheduling policy`，而是降低`静态优先级（nice value）`或者`实时优先级（scheduling priority）`。
这里例外的是SCHED_DEADLINE的设定，按理说如果进程本身的调度策略就是SCHED_DEADLINE，那么应该允许“优先级”降低的操作（这里用优先级不是那么合适，其实就是减小run time，或者加大period，这样可以放松对cpu资源的获取），但是目前的4.4.6内核不允许（也许以后版本的内核会允许）。
此外，如果没有`CAP_SYS_NICE`的能力，那么设定调度策略和调度参数的操作只能是限于属于同一个登录用户的线程。
如果拥有`CAP_SYS_NICE`的能力，那么就没有那么多限制了，可以从普通进程提升成实时进程（修改policy），也可以提升静态优先级或者实时优先级。
具体的修改比较简单，是通过`__setscheduler_params`函数完成，其实也就是是根据sched_attr中的参数设定到task struct相关成员中，大家可以自行阅读代码进行理解。
















`RR`和`FIFO`都只用于实时任务。
## `RR（Round Robin）`调度算法：
RR调度算法是一种循环调度算法。
任务按照到达的顺序依次执行，（在`相同的进程优先级`的情况下，如果优先级高的话可以抢占）每个任务执行一个时间片后，会被放到`就绪队列的末尾`等待下一次调度（放在队尾体现了公平）。
在实时任务中，RR调度算法可以确保每个任务都有机会被执行，并且避免了某个任务长时间占用CPU的情况。
RR调度算法适用于需要`公平分配CPU时间`的实时任务，但可能无法满足对任务响应时间有严格要求的场景。

## `FIFO（First In, First Out）`调度算法：
先到先得，一旦占用cpu则一直运行，直到有更高优先级任务到达或自己放弃。
（在`相同的进程优先级`的情况下，如果优先级高的话可以抢占）任务按照到达的顺序依次执行，**直到任务完成或者被阻塞**。
在实时任务中，FIFO调度算法可以`确保任务按照到达的顺序执行`，不会被其他任务打断，适用于对任务执行顺序有严格要求的场景。
由于FIFO调度算法是`非抢占式`的，因此如果某个任务长时间运行或者阻塞，可能会导致其他任务无法及时执行，影响系统的实时性能。

## `SCHED_OTHER`调度算法：
`SCHED_OTHER`是Linux系统中的一种`默认的`普通进程调度策略，也称为`CFS（Completely Fair Scheduler，完全公平调度器）`，是一种分时调度策略。
在`SCHED_OTHER`调度策略下，系统会根据`进程的优先级（nice值，范围是：-20~19）`和其他因素动态地调整进程的执行顺序，以实现对CPU资源的公平分配。
具体来说，SCHED_OTHER调度策略采用了`时间片轮转`的方式，每个进程都会被分配一个时间片，当`时间片用完`或者`进程主动让出CPU`时，调度器会选择下一个就绪队列中的进程来执行。
`优先级较高`的进程会获得更多的CPU时间，但并不会完全抢占CPU资源，而是通过`动态调整时间片长度`来实现公平调度。
`SCHED_OTHER`调度策略适用于大多数普通进程，它能够在保证系统整体性能的同时，尽可能公平地分配CPU资源给各个进程。
相比于实时调度策略（如SCHED_FIFO和SCHED_RR），SCHED_OTHER更注重系统的整体性能和公平性，适用于一般的后台任务和交互式应用程序。


## RT优先级 和 normal优先级
### 用户空间下
- RT 优先级：
1~99 （值越大，优先级越高）

- normal 优先级：
nice 值从 -20(优先级最高) ~ 19（优先级最低） ，nice值是越小优先级越高，默认为0

#### 内核空间中
（会对normal和rt线程优先级nice值做**归一化**，对内核来讲都是一个个的`task_struct`，`归一化`有利于后面调度策略的计算和选择）
从用户空间的rt优先级值和normal线程的nice值看到，它们的定义似乎是相反的，一个越大代表优先级越高，一个越小优先级越高，怎么归一化呢，实际上内核中是用99 - rt 值进行了反转

normal_priority RT线程优先级：0~99 （注意这里的0~99 和用户空间真实设定的值是相反的关系，用户空间设置rt优先级99，这里就是0）
normal_priority normal线程优先级：100~139 （nice值为 -20，代表这里的100）


# 可执行文件如何被加载到内存执行
(具体要看程序员的自我修养)
在Linux系统中，可执行文件被加载到内存并执行的过程涉及到几个关键步骤，其中包括将文件内容划分为若干个页。
以下是这一过程的概述：

### 1. 可执行文件的格式
Linux下的可执行文件通常是`ELF（Executable and Linkable Format）`格式。
ELF文件包含了程序的代码、数据以及一些元数据，这些元数据定义了程序的内存布局和加载方式。

### 2. 程序加载
当一个程序被执行时，操作系统会进行以下步骤：

#### a. 加载ELF文件
操作系统首先将ELF文件加载到虚拟内存中。这通常涉及到读取文件的头部信息，以确定程序的入口点、程序头表（Program Header Table）等。

#### b. 创建进程控制块
操作系统为新程序创建一个新的`进程控制块（Process Control Block, PCB）`，它包含了：进程的状态、寄存器集合、调度信息等。

#### c. 分配内存
操作系统会为新进程分配虚拟内存空间。这包括为代码段、数据段、堆、栈等分配内存。

### 3. 页面划分
页面的划分通常在以下阶段进行：

#### a. 虚拟内存管理
操作系统的`内存管理单元（Memory Management Unit, MMU）`负责将虚拟地址空间映射到物理内存。
**在加载ELF文件时，操作系统会根据文件的程序头表创建相应的内存页映射。**

#### b. 懒加载（Lazy Loading）
操作系统可能不会立即将整个ELF文件加载到物理内存中。
相反，它可能`只加载必要的部分`，如代码段和初始数据段。
其他部分（如未初始化的数据段）会在需要时才加载，这个过程称为懒加载。

#### c. 页面置换
如果物理内存不足以容纳所有页面，操作系统会使用页面置换算法来决定哪些页面应该被换出到虚拟内存（如磁盘）。

### 4. 页表的更新
当页面被加载到物理内存时，操作系统会更新页表以反映新的虚拟到物理地址的映射。页表通常由操作系统在进程创建时初始化，并在加载和页面置换时动态更新。

### 5. 执行
一旦所有必要的页面都被加载到内存中，CPU就可以开始执行程序的指令了。当CPU访问尚未加载到物理内存中的页面时，会触发缺页异常（Page Fault），操作系统会暂停执行，将缺失的页面加载到内存中，然后恢复执行。

### 结论
页面的划分和内存的加载是一个动态的过程，通常在程序加载和执行期间进行。
操作系统负责管理这一过程，确保程序的内存需求得到满足，同时优化内存的使用。
这个过程涉及到虚拟内存管理、懒加载、页面置换等多个方面。

## 具体的流程
### 编译阶段
当程序源代码被编译成可执行文件时，编译器会生成目标代码，并将其组织成多个段（Segment），如文本段（代码段）、数据段、BSS段等。

### 加载阶段：
当运行程序时，shell会调用`execve()系统调用`，请求内核启动一个新的程序。
操作系统会启动加载器（Loader），**加载器负责将可执行文件中的段映射到内存中的页**。
加载器将可执行目标文件中的`代码`和`数据`从磁盘复制到内存中，然后通过跳转到程序的`第一条指令`或`入口点`来运行该程序。这个将程序复制到内存并运行的过程叫做加载。

这个过程通常具体涉及到以下步骤：
读取头部信息：加载器首先读取可执行文件的头部信息，这些信息包含了程序的各种属性，如程序入口地址、段表等。
地址空间分配：操作系统为程序分配虚拟地址空间。内核将进程的虚拟地址空间划分为大小相等的页（通常是4KB）。每个页在页表中都有一个对应的条目。
映射页表：
内核根据可执行文件的段信息，将虚拟地址空间中的相应页映射到可执行文件的段。
例如，.text段会被映射到只读的页，.data和.bss段会被映射到可读写的页。
将程序的各个段映射到虚拟地址空间的相应页中。操作系统会为这些页分配物理内存，并更新页表。

### 执行阶段：
当程序开始执行时，CPU通过虚拟地址访问内存。如果访问的页尚未加载到物理内存中（发生了缺页中断），操作系统会介入，将磁盘上的页加载到物理内存中，并更新页表。
**一个线程对应一个函数调用栈，函数的嵌套调用是通过不断把当前函数的信息压栈做到的（底层是通过不断移动栈底和栈顶寄存器来实现的）**，
每调用一个函数，就会生成一个新的栈帧。
PS：栈帧（存储着局部变量，和返回地址）


### 加载器的具体工作
Linux 系统中的每个程序都运行在一个进程上下文中，有自己的虚拟地址空间。
当 shell 运行一个程序时，父 shell 进程生成一个子进程，它是父进程的一个复制。
子进程通过`execve系统调用`启动加载器。
加载器`删除子进程现有的虚拟内存段，并创建一组新的代码、数据、堆和栈段`。
新的栈和堆段被初始化为零。
通过**将虚拟地址空间中的页映射到可执行文件的页大小的片**（chunk），新的代码和数据段被初始化为可执行文件的内容。
最后，加载器跳转到`_start`地址，它最终会调用应用程序的`main`函数。
**除了一些头部信息，在加载过程中没有任何从磁盘到内存的数据复制**。
**直到 CPU 引用一个被映射的虚拟页时才会进行复制，此时，操作系统利用它的页面调度机制自动将页面从磁盘传送到内存**。



每个 Linux 程序都有一个`运行时内存映像`(用户栈[由栈指针来指向栈底]、运行时堆、读/写段、只读代码段)，其中的读/写段和只读代码段需要从可执行文件中加载
在 Linux 86-64 系统中，代码段总是从地址`0x400000`处开始，后面是数据段。
运行时堆在数据段之后，通过调用 malloc 库往上增长。
堆后面的区域是为共享模块保留的。
用户栈总是从最大的合法用户地址开始，向较小内存地址增长。
栈上的区域，从地址开始，是为内核（kernel）中的代码和数据保留的，所谓内核就是操作系统驻留在内存的部分。



## 各类函数调用的过程
### 普通函数、类成员函数、类虚函数的调用过程
`普通函数调用流程`
- 开辟栈帧空间
- 函数参数从右至左进行压栈
- 函数返回地址进行压栈
- 函数局部变量进行压栈

`普通成员函数调用流程（大体）`
- 由于函数地址在编译期间已确定，所以直接找到该函数地址
- this指针，作为隐含参数传入该函数
- 之后的调用和普通函数调用方式一致
- 注意：如果该函数中，使用了未实例化的成员变量，由于this指针为null，程序会报错。

`虚函数调用流程（大体）`
- 查找this指针（也就是实例）的地址
- 根据this指针，查找虚函数表（函数指针数组）的地址
- 从虚函数表中，取出相应的函数地址
（也就是说，对于虚函数，在运行时还要对于选择哪一个版本的函数进行选择）


`线程切换的底层实现`
在多线程(任务)环境，栈顶指针指向的存储器区域就是当前使用的堆栈。
**切换线程的一个重要工作，就是将栈顶指针设为当前线程的堆栈栈顶地址。**



# 文件操作接口
```cpp
// 递归地查找某一路径下所有后缀名为xxx的文件，塞进set中

static bool GetOrderedFilesWithExtension(const std::string& base_path, const std::string& extension, std::set<std::string>& ordered_files)
{
  char path[512];
  struct dirent *dp;
  struct stat st; 
  DIR* dir = opendir(base_path.c_str());

  if (dir == nullptr) {
    return false;
  }
  while ((dp = readdir(dir)) != nullptr) {   // readdir返回指向struct dirent的指针
    if (strcmp(dp->d_name, ".") != 0 && strcmp(dp->d_name, "..") != 0) {
      sprintf(path, "%s/%s", base_path.c_str(), dp->d_name);
      ::stat(path, &st);
      if (S_ISDIR(st.st_mode)) {
        GetOrderedFilesWithExtension(path, extension, ordered_files);
      } else {
        int len = strlen(dp->d_name);
        int ext_len = strlen(extension.c_str());
        if (len > ext_len && strcmp(dp->d_name + len - ext_len, extension.c_str()) == 0) {
          ordered_files.insert(path);
        }   
      }   
    }   
  }
  ::closedir(dir);
  return true;
}
```





# 关于 linux系统中，进程的时间片轮转这一调度策略，是如何实现的
在Linux系统中，进程的时间片轮转调度策略是通过`内核中的调度器`来实现的。
具体来说，Linux内核会为每个进程分配一个时间片（通常为几十毫秒），当一个进程的时间片用尽时，调度器会将其放入就绪队列的末尾，然后选择下一个就绪进程来运行。
时钟中断在这个过程中起到关键作用。
**Linux内核会设置一个定时器，当定时器计时器到达时，会触发`时钟中断`。**
在时钟中断处理程序中，调度器会检查当前运行的进程是否已经用尽时间片，如果是，则进行进程切换。

总的来说，时钟中断在Linux系统中起到了调度器实现时间片轮转调度策略的关键作用，但调度器本身负责具体的进程调度逻辑。



# linux系统，线程A正在不停地写入文件，线程B同时执行拷贝线程A正在写入的文件，为什么线程B能够正确完整地拷贝其内容？
`文件系统存在锁机制`
当线程A写入文件时，实际上是在修改文件的`Inode`和`数据块`.
线程B在拷贝文件时，会读取文件的`Inode`和`数据块`，
线程B在拷贝线程A正在写入的文件时，会等待相应的锁被释放后再进行读取操作，从而能够正确地完成文件的拷贝。
因此可以正确完整地拷贝线程A正在写入的文件内容。



# 便捷命令记录
```sh
ls -1 | awk -F. '{print length($NF)}' | sort | uniq -c

# ls -1：列出当前目录下的所有文件名，每个文件名占一行（注意这里是数字1，不是字母l）。
# awk -F.：使用awk命令处理输出，其中-F.指定了字段分隔符为.。
# '{print length($NF)}'：对于每一行输入，$NF表示最后一个字段（即最后一个.后面的那部分），length($NF)则计算这部分的长度，并打印出来。
# 这样，你就能得到每个文件名中最后一个.后面字符的个数。

find -name "*.log" -type f  | awk -F. '{print length($NF)}'  | sort | uniq -c
# -type f   只针对文件
find -name "*log*" -type f  | awk -F. '{print length($5)}'  | sort | uniq -c
```




# `fg`和`bg`命令
进程的`前后台调度命令`，
即将指定号码（非进程号）的命令进程放到前台或后台运行。
比如一个需要长时间运行的命令，我们就希望把它放入后台，这样就不会阻塞当前的操作；
而一些服务型的命令进程我们则希望能把它们长期运行于后台。


```sh
# 在前台结束进程
Ctrl + C
 
# 在前台暂停进程
Ctrl + Z
 
# 查看后台执行的进程 查看进程号码
jobs
 
# 运行命令时，在命令末尾加上&可让命令在后台执行
&
 
# 将命令进程号码为N的命令进程放到前台执行
fg processNum
# 或者
%processNum
 
# 将命令进程号码为N的命令进程放到后台执行
bg processNum
```









# 轻量级线程
`Linux内核`中不存在`线程`的概念，
`每个线程`都被视为一个特殊的`单独的进程`，
所以并不存在内核级线程的概念。
Linux支持用户空间的线程，但是Linux内核本身是不知道线程的存在。
也就是说一个`多线程程序`对于`内核`来说也只是一个`进程`而已，
这就会导致这样的一个多线程程序一旦有一个线程阻塞整个程序都将停止。
为了解决这个问题，Linux将`用户级线程`交给对应的`LWP（Lightweight Process，轻量级进程）`来管理，LWP为什么轻量呢？
因LWP间共享相同的虚拟地址空间、文件描述表、堆、data段等资源，而一般的进程的资源是独享的。
简单来说，我们可以认为`线程`就是`LWP`，
`线程`用于`用户空间`，而`LWP`用于`内核空间`。
**`用户空间线程库`如`POSIX`通过调用`pthread_create`函数创建线程，该函数进一步使用`系统调用clone`去`内核`中创建对应的`LWP`**。

用户空间通过`pthread_self()`以及`this_thread::get_id()`获取到的都是`用户级线程ID`（后者封装的前者），这个ID只保证在`当前进程`内唯一而不能保证全局唯一。要获得该`线程全局唯一ID`需要通过`系统调用syscall(SYS_gettid)`获取该线程对应的`LWP的内核ID`，该ID全局唯一。
（
进程`task_struct`中`pid`存储的是内核对该进程的唯一标示, 即对进程则标示进程号, 对线程来说就是其线程号, 那么对于线程来说一个线程组所有线程与领头线程具有相同的进程号，存入tgid字段
因此`getpid()`返回`当前进程的进程号`，返回的应该是tgid值而不是pid的值, 对于用户空间来说同组的线程拥有相同进程号即tpid, 而对于内核来说, 某种程度上来说不存在线程的概念, 那么`pid`就是内核唯一区分每个进程的标示。

正是linux下`组管理`, `写时复制`等这些巧妙的实现方式
* linux下进程或者线程的创建开销很小
* 既然不管是`线程`或者`进程`内核都是不加区分的，一组共享地址空间或者资源的线程可以组成一个线程组, 那么其他进程即使不共享资源也可以组成进程组, 甚至来说一组进程组也可以组成会话组, 进程组可以简化向所有组内进程发送信号的操作, 一组会话也更能适应多道程序环境
）

虽然我们在区分Linux进程类别, 但是我还是想说Linux下只有一种类型的进程，那就是`task_struct`，
当然我也想说linux其实也没有线程的概念, 只是将那些`与其他进程共享资源的进程`称之为`线程`。
一个进程由于其`运行空间`的不同, 从而有`内核线程`和`用户进程`的区分, `内核线程`运行在`内核空间`, 之所以称之为`线程`是因为它没有虚拟地址空间, 只能访问`内核的代码和数据`, 而`用户进程`则运行在`用户空间`, 但是可以通过`中断`, `系统调用`等方式 从 用户态 陷入 内核态。
`用户进程`运行在用户空间上, 而一些通过`共享资源`实现的`一组进程`我们称之为`线程组`, Linux下内核其实本质上没有线程的概念, **Linux下线程 其实上是与其他进程共享某些资源的进程而已**。
但是我们习惯上还是称他们为线程或者轻量级进程
因此, Linux上进程分3种，内核线程（或者叫核心进程）、用户进程、用户线程, 当然如果更严谨的，你也可以认为用户进程和用户线程都是用户进程。

总而言之, `Linux`中线程与专门线程支持系统是完全不同的
`Unix System V`和`Sun Solaris`将`用户线程`称作为`轻量级进程`(LWP-Light-weight process), 相比较重量级进程, 线程被抽象成一种耗费较少资源, 运行迅速的执行单元。
而对于linux来说, `用户线程`只是一种进程间共享资源的手段, 相比较其他系统的进程来说, linux系统的进程本身已经很轻量级了
举个例子来说, 假如我们有一个包括了四个线程的进程,
- 在`提供专门线程支持的系统`中, 通常会有 一个 包含 指向 `四个不同线程` 的指针 的 `进程描述符`。该描述符复制描述像地址空间, 打开的文件这样的共享资源。
  线程本身再去描述它独占的资源。
- 相反, `Linux`仅仅创建了`四个进程`, 并分配四个普通的`task_struct`结构, 然后建立这四个进程时指定他们共享某些资源。


# `task_struct`
linux系统中的`每个进程`都有一个名为`task_struct`的数据结构，它相当于“进程控制块（`PCB`）”

内核在为每个进程分配`Task_struct`结构的内存空间时，
实际上一次性分配`两个连续`的`内存页面`（共8KB），
其底部约`1KB`空间存放`Task_struct`结构，
上面的7KB空间存放`进程系统空间堆栈`。 

`task_struct`
主要包括：
* 任务ID
* 亲缘关系
* 任务状态
* 任务权限
* 运行统计
* 进程调度
* 信号处理
* 内存管理
* 文件与文件系统
* 内核栈

## 任务ID
```c
pid_t pid;  // process id
pid_t tgid; // thread group id
struct task_struct *group_leader;
```

任何一个进程，如果只有`主线程`，那`pid`和`tgid`相同，group_leader 指向自己。
但是，如果一个进程创建了其他线程，那就会有所变化了。
`线程`有自己的`pid`，`tgid 就是进程的主线程的 pid`，`group_leader`指向的进程的`主线程`。
因此 **根据`pid`和`tgid``是否相等` 我们可以判断该任务是`进程`还是`线程`**。


## 亲缘关系
除了0号进程以外，其他进程都是有`父进程`的。
全部进程其实就是一颗`进程树`，相关成员变量如下所示:
```c
struct task_struct __rcu *real_parent;
/* real parent process */

struct task_struct __rcu *parent;
/* recipient of SIGCHLD, wait4() reports */

struct list_head children;
/* list of my children */

struct list_head sibling;
/* linkage in my parent's children list */

/*
PS:   
在Linux内核中，__rcu是一种用于读-复制更新（RCU，Read-Copy Update）机制的修饰符。这种机制用于确保多线程环境下对数据结构的安全访问。因此，
*/
```

* parent 指向其父进程。
   当它终止时，必须向它的父进程发送信号。
* children 指向子进程链表的头部。
   链表中的所有元素都是它的子进程。
* sibling 用于把当前进程插入到兄弟链表中。

通常情况下，`real_parent`和`parent`是一样的，但是也会有另外的情况存在。
例如，`bash`创建一个`进程`，那进程的`parent`和`real_parent`就都是 bash。
如果在`bash`上使用`GDB`来 debug 一个进程，这个时候`GDB`是 parent(GDB实际上会接管被调试进程的执行，成为直接父进程)，bash 是这个进程的`real_parent`。
两个parent是`真实父进程`和`直接父进程`的关系

## 任务状态
```c
volatile long state;    /* -1 unrunnable, 0 runnable, >0 stopped */
int exit_state;
unsigned int flags;
```

```c
// include/linux/sched.h

/* Used in tsk->state: */
#define TASK_RUNNING                    0
#define TASK_INTERRUPTIBLE              1
#define TASK_UNINTERRUPTIBLE            2
#define __TASK_STOPPED                  4
#define __TASK_TRACED                   8
/* Used in tsk->exit_state: */
#define EXIT_DEAD                       16
#define EXIT_ZOMBIE                     32
#define EXIT_TRACE                      (EXIT_ZOMBIE | EXIT_DEAD)
/* Used in tsk->state again: */
#define TASK_DEAD                       64
#define TASK_WAKEKILL                   128
#define TASK_WAKING                     256
#define TASK_PARKED                     512
#define TASK_NOLOAD                     1024
#define TASK_NEW                        2048
#define TASK_STATE_MAX                  4096

#define TASK_KILLABLE           (TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
```


`TASK_RUNNING`并不是说进程正在运行，而是表示`进程在时刻准备运行`的状态。
当处于这个状态的进程`获得时间片`的时候，就是在运行中；
如果没有获得时间片，就说明它被其他进程抢占了，在等待再次分配时间片。
在运行中的进程，一旦要进行一些`I/O`操作，需要等待`I/O`完毕，这个时候会释放 CPU，进入睡眠状态。

在Linux中有两种睡眠状态:
* 一种是`TASK_INTERRUPTIBLE`，`可中断`的睡眠状态。
这是一种浅睡眠的状态，也就是说，虽然在睡眠，等待 I/O 完成，但是这个时候一个信号来的时候，进程还是要被唤醒。
只不过唤醒后，不是继续刚才的操作，而是进行`信号处理`。
当然程序员可以根据自己的意愿，来写信号处理函数，例如收到某些信号，就放弃等待这个 I/O 操作完成，直接退出；
或者收到某些信息，继续等待。
* 另一种睡眠是`TASK_UNINTERRUPTIBLE`，不可中断的睡眠状态。这是一种深度睡眠状态，不可被信号唤醒，只能死等 I/O 操作完成。
一旦 I/O 操作因为特殊原因不能完成，这个时候，谁也叫不醒这个进程了。你可能会说，我 kill 它呢？别忘了，kill 本身也是一个信号，既然这个状态不可被信号唤醒，kill 信号也被忽略了。
除非重启电脑，没有其他办法。
因此，这其实是一个比较危险的事情，除非程序员极其有把握，不然还是不要设置成 TASK_UNINTERRUPTIBLE。
* 于是，我们就有了一种新的进程睡眠状态，`TASK_KILLABLE`，可以终止的新睡眠状态。进程处于这种状态中，它的运行原理类似TASK_UNINTERRUPTIBLE，只不过`可以响应致命信号`。
由于`TASK_WAKEKILL`用于在接收到致命信号时唤醒进程，因此TASK_KILLABLE即在TASK_UNINTERUPTIBLE的基础上增加一个TASK_WAKEKILL标记位即可。

`TASK_STOPPED`是在进程接收到`SIGSTOP、SIGTTIN、SIGTSTP或者 SIGTTOU`信号之后进入该状态。

`TASK_TRACED`表示进程被 debugger 等进程监视，进程执行被调试程序所停止。
当一个进程被另外的进程所监视，每一个信号都会让进程进入该状态。

一旦一个进程要结束，先进入的是`EXIT_ZOMBIE`状态，但是这个时候它的父进程还没有使用`wait()`等系统调用来获知它的终止信息，此时进程就成了`僵尸进程`。
`EXIT_DEAD`是进程的最终状态。`EXIT_ZOMBIE`和`EXIT_DEAD`也可以用于`exit_state`。

上面的`进程状态`和进程的`运行`、`调度`有关系，还有其他的一些状态，我们称为`标志`。
放在 flags字段中，这些字段都被定义成为宏，以 PF 开头。
```c
#define PF_EXITING    0x00000004
#define PF_VCPU      0x00000010
#define PF_FORKNOEXEC    0x00000040
```

`PF_EXITING`表示正在退出。
当有这个 flag 的时候，在函数`find_alive_thread()`中，找活着的线程，遇到有这个 flag 的，就直接跳过。

`PF_VCPU`表示进程运行在虚拟 CPU 上。
在函数 account_system_time中，统计进程的系统运行时间，如果有这个 flag，就调用 account_guest_time，按照客户机的时间进行统计。

`PF_FORKNOEXEC`表示 fork 完了，还没有 exec。
在`_do_fork ()`函数里面调用`copy_process()`，这个时候把 flag 设置为 PF_FORKNOEXEC()。
当 exec 中调用了`load_elf_binary()`的时候，又把这个 flag 去掉。


## 任务权限
`real_cred`是指可以操作本任务的对象，
而`red`是指本任务可以操作的对象。

```c
/* Objective and real subjective task credentials (COW): */
const struct cred __rcu         *real_cred;
/* Effective (overridable) subjective task credentials (COW): */
const struct cred __rcu         *cred;
```
red定义如下所示：
```c
struct cred {
// ......
    kuid_t          uid;            /* real UID of the task */
    kgid_t          gid;            /* real GID of the task */
    kuid_t          suid;           /* saved UID of the task */
    kgid_t          sgid;           /* saved GID of the task */
    kuid_t          euid;           /* effective UID of the task */
    kgid_t          egid;           /* effective GID of the task */
    kuid_t          fsuid;          /* UID for VFS ops */
    kgid_t          fsgid;          /* GID for VFS ops */
// ......
    kernel_cap_t    cap_inheritable; /* caps our children can inherit */
    kernel_cap_t    cap_permitted;  /* caps we're permitted */
    kernel_cap_t    cap_effective;  /* caps we can actually use */
    kernel_cap_t    cap_bset;       /* capability bounding set */
    kernel_cap_t    cap_ambient;    /* Ambient capability set */
// ......
} __randomize_layout;
```
大部分是关于用户和用户所属的用户组信息。

* `uid`和`gid`，注释是 real user/group id。一般情况下，谁启动的进程，就是谁的 ID。但是权限审核的时候，往往不比较这两个，也就是说不大起作用。
* `euid` 和`egid`，注释是 effective user/group id。一看这个名字，就知道这个是起“作用”的。
   当这个进程要操作消息队列、共享内存、信号量等对象的时候，其实就是在比较这个用户和组是否有权限。
* `fsuid`和`fsgid`，也就是 filesystem user/group id。
   这个是对文件操作会审核的权限。

除了以`用户`和`用户组`控制权限，Linux 还有另一个机制就是 `capabilities`。
原来控制进程的权限，要么是高权限的 root 用户，要么是一般权限的普通用户，这时候的问题是，root 用户权限太大，而普通用户权限太小。
有时候一个普通用户想做一点高权限的事情，必须给他整个 root 的权限。这个太不安全了。
于是，我们引入新的机制 capabilities，用位图表示权限，在capability.h可以找到定义的权限。我这里列举几个。
```c
#define CAP_CHOWN            0
#define CAP_KILL             5
#define CAP_NET_BIND_SERVICE 10
#define CAP_NET_RAW          13
#define CAP_SYS_MODULE       16
#define CAP_SYS_RAWIO        17
#define CAP_SYS_BOOT         22
#define CAP_SYS_TIME         25
#define CAP_AUDIT_READ          37
#define CAP_LAST_CAP         CAP_AUDIT_READ
```
对于普通用户运行的进程，当有这个权限的时候，就能做这些操作；
没有的时候，就不能做，这样粒度要小很多。


## 运行统计
运行统计从宏观来说也是一种`状态变量`，但是和任务状态不同，其存储的主要是`运行时间`相关的成员变量，具体如下所示
```c
u64        utime;    //用户态消耗的CPU时间
u64        stime;    //内核态消耗的CPU时间
unsigned long      nvcsw;     //自愿(voluntary)上下文切换计数
unsigned long      nivcsw;    //非自愿(involuntary)上下文切换计数
u64        start_time;     //进程启动时间，不包含睡眠时间
u64        real_start_time;      //进程启动时间，包含睡眠时间
```

## 进程调度
```c
//是否在运行队列上
int        on_rq;
//优先级
int        prio;
int        static_prio;
int        normal_prio;
unsigned int      rt_priority;
//调度器类
const struct sched_class  *sched_class;
//调度实体
struct sched_entity    se;
struct sched_rt_entity    rt;
struct sched_dl_entity    dl;
//调度策略
unsigned int      policy;
//可以使用哪些CPU
int        nr_cpus_allowed;
cpumask_t      cpus_allowed;
struct sched_info    sched_info;
```

## 信号处理
```c
/* Signal handlers: */
struct signal_struct    *signal;
struct sighand_struct    *sighand;
sigset_t      blocked;
sigset_t      real_blocked;
sigset_t      saved_sigmask;
struct sigpending    pending;
unsigned long      sas_ss_sp;
size_t        sas_ss_size;
unsigned int      sas_ss_flags;
```
这里将信号分为三类:
* `阻塞暂不处理`的信号（blocked）
* `等待处理`的信号（pending）
* `正在通过信号处理函数处理`的信号（sighand）
信号处理函数默认使用用户态的函数栈，当然也可以开辟新的栈专门用于信号处理，这就是 sas_ss_xxx 这三个变量的作用。


## 内存管理
```c
struct mm_struct                *mm;
struct mm_struct                *active_mm;
```

## 文件与文件系统
```c
/* Filesystem information: */
struct fs_struct                *fs;
/* Open file information: */
struct files_struct             *files;
```

## 内核栈
内核栈相关的成员变量如下所示。
为了介绍清楚其作用，我们需要从为什么需要内核栈开始逐步讨论。
```c
struct thread_info    thread_info;
void  *stack;
```
当进程产生`系统调用`时，会利用`中断`陷入`内核态`。
而`内核态`中也存在着各种函数的调用，因此我们需要有`内核态函数栈`。Linux 给每个 task 都分配了`内核栈`。
- 在 32 位系统上`arch/x86/include/asm/page_32_types.h`，是这样定义的：一个`PAGE_SIZE`是 4K，左移一位就是乘以 2，也就是 8K。
```c
#define THREAD_SIZE_ORDER  1
#define THREAD_SIZE    (PAGE_SIZE << THREAD_SIZE_ORDER)
```

- 内核栈在 64 位系统上`arch/x86/include/asm/page_64_types.h`，是这样定义的：在 PAGE_SIZE 的基础上左移两位，也即 16K，并且要求起始地址必须是 8192 的整数倍。
```c
#ifdef CONFIG_KASAN
#define KASAN_STACK_ORDER 1
#else
#define KASAN_STACK_ORDER 0
#endif

#define THREAD_SIZE_ORDER  (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```


```c
union thread_union {
#ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK
    struct task_struct task;
#endif
#ifndef CONFIG_THREAD_INFO_IN_TASK
    struct thread_info thread_info;
#endif
    unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```
这个结构是对`task_struct`结构的补充。
因为 task_struct 结构庞大但是通用，不同的体系结构就需要保存不同的东西，所以**往往与`体系结构`有关的，都放在`thread_info里面**。
在内核代码里面采用一个union将thread_info和stack放在一起，在 `include/linux/sched.h`中定义用以表示内核栈。
由代码可见，这里根据架构不同可能采用旧版的task_struct直接放在内核栈，而新版的均采用thread_info，以节约空间。

另一个结构 pt_regs，定义如下。
其中，32 位和 64 位的定义不一样
```c
#ifdef __i386__
struct pt_regs {
  unsigned long bx;
  unsigned long cx;
  unsigned long dx;
  unsigned long si;
  unsigned long di;
  unsigned long bp;
  unsigned long ax;
  unsigned long ds;
  unsigned long es;
  unsigned long fs;
  unsigned long gs;
  unsigned long orig_ax;
  unsigned long ip;
  unsigned long cs;
  unsigned long flags;
  unsigned long sp;
  unsigned long ss;
};
#else 
struct pt_regs {
  unsigned long r15;
  unsigned long r14;
  unsigned long r13;
  unsigned long r12;
  unsigned long bp;
  unsigned long bx;
  unsigned long r11;
  unsigned long r10;
  unsigned long r9;
  unsigned long r8;
  unsigned long ax;
  unsigned long cx;
  unsigned long dx;
  unsigned long si;
  unsigned long di;
  unsigned long orig_ax;
  unsigned long ip;
  unsigned long cs;
  unsigned long flags;
  unsigned long sp;
  unsigned long ss;
/* top of stack page */
};
#endif
```
内核栈和task_struct是可以互相查找的，而这里就需要用到task_struct中的两个内核栈相关成员变量了。

### 通过task_struct查找内核栈
如果有一个 task_struct 的 stack 指针在手，
即可通过下面的函数找到这个线程内核栈：
```c
static inline void *task_stack_page(const struct task_struct *task)
{
    return task->stack;
}
```
从 task_struct 如何得到相应的`pt_regs`呢？
我们可以通过下面的函数，先从 task_struct找到内核栈的开始位置。
然后这个位置加上`THREAD_SIZE`就到了最后的位置，然后转换为 struct pt_regs，再减一，就相当于减少了一个 pt_regs 的位置，就到了这个结构的首地址。
```c
/*
 * TOP_OF_KERNEL_STACK_PADDING reserves 8 bytes on top of the ring0 stack.
 * This is necessary to guarantee that the entire "struct pt_regs"
 * is accessible even if the CPU haven't stored the SS/ESP registers
 * on the stack (interrupt gate does not save these registers
 * when switching to the same priv ring).
 * Therefore beware: accessing the ss/esp fields of the
 * "struct pt_regs" is possible, but they may contain the
 * completely wrong values.
 */
#define task_pt_regs(task) \
({                  \
  unsigned long __ptr = (unsigned long)task_stack_page(task);  \
  __ptr += THREAD_SIZE - TOP_OF_KERNEL_STACK_PADDING;    \
  ((struct pt_regs *)__ptr) - 1;          \
})



// TOP_OF_KERNEL_STACK_PADDING
#ifdef CONFIG_X86_32
# ifdef CONFIG_VM86
#  define TOP_OF_KERNEL_STACK_PADDING 16
# else
#  define TOP_OF_KERNEL_STACK_PADDING 8
# endif
#else
# define TOP_OF_KERNEL_STACK_PADDING 0
#endif
```
也就是说，32 位机器上是 8，其他是 0。
这是为什么呢？因为压栈 pt_regs 有两种情况。
我们知道，CPU 用 ring 来区分权限，从而 Linux 可以区分内核态和用户态。
因此，第一种情况，我们拿涉及从用户态到内核态的变化的系统调用来说。因为涉及`权限的改变`，会`压栈保存 SS、ESP 寄存器`的，这两个寄存器共占用 8 个 byte。
另一种情况是，`不涉及权限的变化`，就不会压栈这 8 个 byte。
这样就会使得两种情况不兼容。如果没有压栈还访问，就会报错，所以还不如预留在这里，保证安全。在 64 位上，修改了这个问题，变成了定长的。

### 通过内核栈找task_struct
`thread_info`
```c
struct thread_info {
    struct task_struct  *task;    /* main task structure */
    __u32      flags;    /* low level flags */
    __u32      status;    /* thread synchronous flags */
    __u32      cpu;    /* current CPU */
    mm_segment_t    addr_limit;
    unsigned int    sig_on_uaccess_error:1;
    unsigned int    uaccess_err:1;  /* uaccess failed */
};
struct thread_info {
    unsigned long flags;          /* low level flags */
    unsigned long status;    /* thread synchronous flags */    
};
```
老版中采取current_thread_info()->task 来获取task_struct。thread_info 的位置就是内核栈的最高位置，减去 THREAD_SIZE，就到了 thread_info 的起始地址。
```c
static inline struct thread_info *current_thread_info(void)
{
    return (struct thread_info *)(current_top_of_stack() - THREAD_SIZE);
}
```
  而新版本则采用了另一种current_thread_info

```c
#include <asm/current.h>
#define current_thread_info() ((struct thread_info *)current)
#endif
```
那 current 又是什么呢？在 arch/x86/include/asm/current.h 中定义了。
```c
struct task_struct;

DECLARE_PER_CPU(struct task_struct *, current_task);

static __always_inline struct task_struct *get_current(void)
{
    return this_cpu_read_stable(current_task);
}
```

新的机制里面，每个 CPU 运行的 task_struct 不通过thread_info 获取了，而是直接放在`Per CPU 变量`里面了。
多核情况下，CPU 是同时运行的，但是它们共同使用其他的硬件资源的时候，我们需要解决多个 CPU 之间的同步问题。
Per CPU 变量是内核中一种重要的同步机制。
顾名思义，Per CPU 变量就是为每个 CPU 构造一个变量的副本，这样多个 CPU 各自操作自己的副本，互不干涉。
比如，当前进程的变量 current_task 就被声明为 Per CPU 变量。
要使用 Per CPU 变量，首先要声明这个变量，在 arch/x86/include/asm/current.h 中有：
```c
DECLARE_PER_CPU(struct task_struct *, current_task);
```
然后是定义这个变量，在 arch/x86/kernel/cpu/common.c 中有：
```c
DEFINE_PER_CPU(struct task_struct *, current_task) = &init_task;
```
也就是说，系统刚刚初始化的时候，
current_task 都指向`init_task`。
当某个 CPU 上的进程进行切换的时候，current_task 被修改为将要切换到的目标进程。
例如，进程切换函数__switch_to 就会改变 current_task。



# `strings`命令
`打印文件中可打印的字符`。
这个文件可以是文本文件（test.c）, 可执行文件(test),  动态链接库(test.o), 静态链接库(test.a)
在对象文件或二进制文件中查找可打印的字符串。








