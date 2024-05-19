# 数据回传方案
## 场景概览
Someip_recorder将兼容当前方案并扩展出适合量产车的方案，默认的情况下运行Product逻辑代码，当收到PC端的DDS消息需要切换到Debug模式后，切换到Debug模式，此时停止响应Product模式下的任何指令，直到PC端发送的DDS消息切换到Product模式
data_collector仅在Product场景
当someip_recorder收到trigger触发消息，合并mcap文件，通知data collector上云

## data_collector三大阶段
1. 启动阶段
一些初始化操作，解析配置，建立通信连接，监听someip_recorder状态

2. 运行阶段
解析someip_recorder过来的消息，**并行**收集数据：收集一些必要的数据，收集完了打包--压缩--上传（显然都是串行），和云端进行通信

3. 退出阶段
延迟下电（why？）

## someip recorder模块细节
### 概览
- param_mgr
参数解析、存储管理

- someip_converter

- dds_control_mgr
DDS控制消息接收、处理

- someip_record_mgr
DDS消息落盘管理

- slice_mgr
切片数据管理

- state_mgr
模块状态消息广播

所有功能模块主要包含两个线程，someip_recorder_thread、dds_control_report_thread
1. someip_recorder_thread
主要负责`dds数据收集`、`数据落盘`、`trigger处理`等

2. dds_control_report_thread
主要负责`control消息接收`、`状态汇报`等


### 参数解析、存储管理模块param_mgr
#### param_mgr debug模式
- someip_recorder.json
    获取默认需要录制的someip的topic
    注意，该json文件可能在运行时被动态地修改

- recorder_parameter.yaml
存储池配置信息、切片功能配置信息、动态录制的topic列表清单信息、trigger列表清单以及trigger时间参数等

#### param_mgr product模式
增加对Product方案新增的数据的解析及相关的接口

### SOMEIP消息接收、转换为DDS数据模块someip_converter
#### someip_converter debug模式
someip消息都是通过someip client提供的回调函数获取，在获取到someip消息之后，首先判断是否存在动态配置的topic list，如果不存在则判断是否在默认的topic list中，如果不是，则不处理，如果存在动态配置topic list则判断是否在动态的topic list中存在；

如果someip消息存在于动态或者默认topic list中，则将someip消息转化为fastdds消息，然后通过线程间通讯将dds数据转发给someip_recorder_thread线程，来统一进行落盘处理；
？？？默认topic list是什么

#### someip_converter product模式
 本模块复用Debug方案中的处理逻辑，someip数据收到后，转换为DDS数据，然后存储为mcap文件。

### DDS消息落盘管理模块someip_record_mgr
本模块负责someip消息转换后的fastdds消息进行落盘处理

#### 3.3.1 someip_record_mgr debug模式
当收到`fastdds录制开始消息`后，在ram disk中写入（不存在则创建）当前时间的mcap文件，该文件按照”yyyymmhhmm.mcap”的格式来定义文件名，文件的打开、创建根据当前的时间，精确到分钟；

文件落盘需要启动一个子线程，在子线程中首先判断disk是否已满（或者留有一定的余量即剩余空闲空间是否低于10M）：
- 如果已满，则需要删除最老的mcap文件，在进行关闭存盘；
- 如果未满，则直接关闭当前文件存储到磁盘，落盘子线程退出；
落盘压缩格式，默认使用zstd压缩格式；
注意：落盘的文件在规定的时间内保持打开，等规定的时间到了单独启动一个线程进行文件关闭存盘，防止数据接收因为存盘（同步地等待落盘很慢）被block住；另外Schema id，channel id可以参考示例或者每个mcap文件使用随机码来填充

#### 3.3.2 someip_record_mgr product模式
进程启动默认进入Product模式，
接收到someip消息就开始在ram disk中写入（不存在则创建）当前时间的mcap文件，即1秒一个mcap文件。该文件按照”yyyymmdd_hhmmss.mcap”的格式来定义文件名，文件的打开、创建根据当前的时间，**精确到秒**；
增加一个`磁盘管理的线程`，实时监控磁盘的使用情况，实时落入的mcap文件将以（M+N）*2个文件作为界限（或者磁盘空间满了），达到上限后落入新文件的同时删除最老的文件，不断重复此操作。

### DDS控制消息接收、处理模块dds_control_mgr
本模块负责同时接收处理两种传输方式的控制信息：
- 一种是基于dds（包括ros2上位机或远程trigger）
- 另一种是基于someip的IPC的控制信息（包括自动Trigger功能）。

该模块作为client端需要订阅所有的控制信息，获取到控制信息后解析相关的信号，控制信号主要用来处理：
1)	清除磁盘除当前正在使用的mcap外的其他文件；
2)	Trigger触发功能(扩展)；
3)	开始、停止录制功能；
4)	触发切片功能；

```c
    struct RecorderCommand {
      std_msgs::msg::Header header;
	uint8 trigger_id;
      uint8 command;
      sequence<string> record_topic_list;
};
```

### 3.5 切片数据管理模块slice_mgr
#### slice_mgr product模式
product模式独有
Trigger源产生时，复用3.4定义的结构体，此时的trigger_id字段，根据分配好的trigger id来进行填充。
不用填充record_topic_list，通过配置来录制topic，默认配置全量topic。
Someip_recorder对接收到的trigger进行基本的有效性判断，当前状态在Product模式时，才能响应相关的trigger信息，并且根据配置文件读取的trigger列表判断是否是有效的trigger源，无效则忽视。
收到trigger指令后，解析协议自带的时间戳字段，以该时间戳作为trigger产生的时间，同时根据此时间戳即trigger id创建trigger目录，执行trigger逻辑（见第5章节），等待锁定期(5秒)结束。

后5个文件处理结束后，开始合并前10后5即15个mcap文件为一个新的mcap文件，并根据trigger时间戳及trigger_id来命名该文件，新的mcap文件合并结束后，**发DDS消息到Data collector进程**。
若5秒锁定期内还有trigger产生，则单独将trigger信息记录在同目录下的文件中。最后将此目录拷贝到ufs分区中。


### 3.6 模块状态消息广播模块state_mgr
本模块负责状态信息广播，支持两种消息的广播，一种是someip消息，一种是dds消息，共用相同的消息体。
1. Someip消息：
    需要建模工具首先建好模型，模型使用的service interface数据应该跟dds使用的数据一致，作为服务端offer提供的服务的event信息，并且按照指定的时间间隔广播状态信息到订阅者。
2. DDS消息：
    需要定义状态信息的idl文件，转化为dds代码之后，作为服务端offer提供的服务的event信息，并且按照指定的时间间隔广播状态信息到订阅者：
1)	广播当前录制状态；
2)	广播someip_recorder的整体运行状态；
3)	广播存储池的状态信息；

注意：滑窗平均算法计算消息的频率，收到第一帧数据后记下timestamp，依次记下后边的每帧数据的timestamp，依次计算帧率并且按照实际个数取平均值，当达到50个数据后，保持按照50来算帧率的平均值，新加一帧，删除最旧一帧；
各个状态的广播周期需要客户定义，广播的时候只广播当前的状态信息。


## data collector模块细节
DataCollector作为单独的进程，需要提供根据该模型开发的数据采集及上云功能的源代码，跟someip recorder的区别是，**不再使用脚本加jinja2模板的方式提供源代码**。

`管理模块dc_mgr`负责初始化和反初始化配置解析模块、监听模块、收集模块，创建数据收集、数据处理、数据上传线程，创建线程间交互的数据结构，进程退出时销毁线程，关闭打开的文件描述符；

`配置解析模块`负责解析yaml文件中模块配置信息，供管理模块使用，解析yaml文件中的收集文件的详细信息，供收集模块使用；

`监听模块`负责监听来自recorder的收集通知，**新起线程执行数据收集任务**，同时监听recorder的完成通知，通过交互通知收集模块通信数据已收集；

`收集模块`负责：
根据解析模块解析出的参数、监听模块的输入、管理模块创建的线程与数据结构，实现数据的收集功能，同时依赖压缩模块，实现数据的处理；

### 数据收集管理模块 dc_mgr
负责其他模块的生命周期管理。
如内部通信模块的建立，外部通信模块的建立等各个模块的初始化过程以及接收退出信号对各个模块的反初始化处理。
内部通信资源是与someip_recorder进行的someip通信，用于接收someip_recorder的状态通知；
外部通信资源是与云端通信的链路，将利用车云通信的sdk接口；

#### 启动流程
##### 1. 初始化配置解析模块

##### 2. 初始化内部通信模块

##### 3. 初始化云端通信模块

##### 4. 初始化数据收集模块
某一文件类型 -- 对应的任务类

#### 退出流程
在下电阶段，将执行程序的退出动作。需要做自身资源的释放动作，内容包括关闭线程池、退出线程、关闭文件句柄、释放动态分配的内存、系统调用对象的析构函数。

### someip_recorder状态的监听模块 state_listener
负责监听someip_recorder状态，用于触发其它非通信数据的收集流程，构成一个完整整包来上云。

处理流程：
1. 收到Someip Recorder整体运行场景Scenario为1–Debug场景，不做任何处理；
2. 收到Someip Recorder整体运行场景Scenario为2–Product场景，将做如下处理；
    - 当收到trigger状态为“trigger working”时，将数据收集任务提交到线程池进行执行；
    - 当收到trigger状态为“trigger finish”时，需要将main_trigger_id、触发时间戳信息入队通信数据已收集列表（该列表用来查询状态），表示someip_recorder已经将trigger_[main_trigger_id]_yyyymmdd_hhmmss目录拷贝过来； 


### dc配置解析模块 param_mgr
负责本模块配置的解析及参数管理。如trigger列表，trigger绑定的数据收集列表，车云通信参数等。
由dc_mgr初始化之后，对配置文件进行解析，同时内部进行参数化管理，供其它子模块使用。

dc配置解析模块 param_mgr主要有两个功能：
1. dc_mapping.yaml解析
2. dc_cfg.yaml解析

### 数据压缩模块 compress
负责进行整体数据的打包及压缩。
内部文件的压缩根据实际的受益来选择压缩或不压缩，camera数据采用H264格式压缩。
外层将进行tar打包以及gz的压缩处理

### 数据收集模块 collector
负责收集someip_recorder数据及其它数据，如版本文件，设备信息文件，标定文件，camera数据，日志文件等。
！！！数据包括`通信数据`和`非通信数据`。
- `通信数据`是`someip_recorder录制的数据`；
- `非通信数据`是`版本文件，设备信息文件，标定文件，camera数据，日志文件`；

在确认数据收集结束后转发内部信号，触发：数据打包、数据压缩、数据上云、原始数据删除等处理。
在trigger来源方面，`auto_trigger`与`manual_trigger`不分优先级，**由recorder模块将trigger统一处理**。
**DC模块无法感知trigger的差异。**
数据的收集和数据的处理分为两个线程处理。
对于`日志文件`，需要各模块对日志内容的质量、日志的刷写频度、大小做一定程度的要求，通过采集的日志结合数据回放来定位问题。
收集的日志可以根据trigger的时间戳收集当前日志及上一份压缩日志。

#### 流程
PS：量产限定
执行步骤如下：
1. 数据收集（配备线程池）
    a. `数据收集任务类`根据`触发时间戳`在ufs_disk中创建指定目录，如在`/smart_data/col/`下创建`data_bag_yyyymmdd_hhmmss目录`（精确到秒）。

    b. 先并行收集`非通信文件`
        i. 根据`main_trigger_id`查询`DcMapping对象`，返回需要回传的数据类型，向线程池提交需要执行的任务，等待所有任务执行完毕。
        （其中`DcMapping`类对象是param_mgr模块持有的，**一个trigger会绑定数据收集列表，也就是若干topic**）
        它解析`dc_mapping.yaml`文件，将triggerId与对应数据类型存储到unordered map中去，并提供查询接口能通过triggerId返回需要收集的数据类型
        而triggerID和某一场景相关（如急加速、司机紧急制动等，这些属于`auto trigger`；而`上位机手动录制trigger`属于`pc_trigger`；而`云端手动录制trigger`，属于`cloud_trigger`），而该场景对应若干数据类型
        由此可见，trigger是用于`通知recorder，要进行数据回传了`的信号
        ii.	从全局变量中获取VIN码，写入到`/smart_data/col/data_bag_yyyymmdd_hhmmss/device`文件中；

    c. 再查询`通信文件`
        i. 由于recorder需要**在收到trigger之后还需要采集5s的数据**，recorder在落盘完成之后，会发出`trigger status`为`trigger finish`的通知给到dc模块， 此时，通知`数据收集模块collector`，通信数据收集完毕；
        dc判断其他非通信数据是否已经拷贝完毕，如果已经拷贝完毕，可以将拷贝过来后将执行整体的打包。

        ii. 循环查询通信数据已收集并发队列，
        如果查询成功，则将已收集完毕的数据入队待压缩并发队列，退出循环；
        否则等待1秒之后再查询，直到查询成功；

    d. 收集视频数据
        i. camera处理雷同，也会在此状态下，拷贝了需要收集的数据在指定目录下（`/smart_data/col/data_bag_yyyymmdd_hhmmss/camera_h264`）

2. 数据处理
    a. `数据处理线程`收到触发事件，从待压缩并发队列中出队需`压缩文件夹`，将对应的`执行打包`和`压缩动作`添加到线程池并行执行； 
    b. 将`已压缩文件`插入待上云并发队列，并发送通知给到数据上传线程；

3. 压缩文件上云
    a. `数据上传线程`收到通知，从待上云并发队列出队；
    b. 判断文件大小，小于等于10M，调用`云端通信模块v2c_sdk`的`小文件上传接口`，大于10M，调用`分片传送上传接口`进行文件上传操作；

4. 确认数据已经上云后，将压缩包拷贝到`/smart_data/col/uploaded`目录下；（也就是一个简单的备份操作）
    统计uploaded下面的文件数量，大于预设的最大值，将最老的压缩包删除；

5. 在`/smart_data/col/`目录下创建一个操作日志：
    记录数据收集、包括非通信数据、通信数据的收集情况、数据打包、数据压缩、数据上传的整个pipeline情况；
    操作日志不上传，主要是给自身模块debug用。
    目的是：记录上次下电的时候，dc执行到哪一步了。
    dc本次上电后，会根据解析的结果，继续执行上一次的操作。


### 云端通信模块v2c_sdk
负责与云端交互，从云端下载`配置文件`，若下载失败默认使用本地的配置文件；
将`收集的数据`进行上云。该模块将考虑数据的`安全通信`，与云端的`身份认证`内容。
车云通信协议对数据上云采用`https协议`。
数据回传走`vlan 11`链路上云。

`云端通信模块`提供的接口中所有涉及时间戳（timestamp）的填充时，比如：`小文件上传接口`和`开始分片上传接口`中`Headers`的填充时，需要根据当前系统的时区采取相应的动作。
如果当前时区是东八区，则直接填充系统时间到Headers的timestamp字段中，如果是零时区，需要在当前系统时间加上八小时填充到Headers的timestamp字段中。

`断点续传`目前考虑处理好`上电周期内`的续传：
在上传过程中，突然网络中断，需要重传几次（超时时间和重传次数从yaml配置文件获取），如果信号恢复，能够继续上传。
重试几次失败后，本次对于下电场景，如果没传完，在下次上电时，读取操作日志，发现下次上电有上传动作没有执行完，就重新请求上传。

## 相关配置文件解读
```yaml
dc_mapping.yaml
mapping:
- triggerId: xxx
      Task: uploadCalibration
- triggerId: yyy
      Task: uploadAllLog
- triggerId: zzz
      Task: allFileMergeAndUpload
```

```yaml
dc_cfg.yaml
dc_config:
  version: 1.0
  maxThreadNum: 15
  collections:
    versionCollector:   # 版本信息
      searchFileList:
        - "/smart_plt/VERSION"
        - "/etc/version"
    calibrationFilesCollector:  # 标定文件
      searchFolderPath:
        - "/smart_para/parameter/calibration_eol"
    logCollector:   # 日志文件
      searchSubPath: false
      searchFolderPath:
        - "/smart_data/logs/app"
        - "/smart_data/logs/mcu/zlog"
        - "/smart_data/logs/os"
        - "/smart_data/logs/om"
        - "/smart_data/logs/security"
      searchFileName:
        - "*.log"
        - "*.log.gz"
      logNum: 2 # 每种log的收集数量
    cameraEncodeCollector:  # camera数据
      - camera_1
      - camera_2
      - camera_3
      - camera_4
      - camera_5
      - camera_6
      - camera_7
      - camera_8
      - camera_9
      - camera_10
      - camera_11
  processors:
    compress:
      outputFileName: "data_bag_%Y%m%d_%H%M%S.tar.gz"   # 输出压缩包，以精确到秒的时刻来命名
      outputFolderPath: "/smart_data/col"   # 压缩包输出路径
      delay_time: 10 # ms，分时压缩，降低cpu占有率
    monitor:
      maxfiles: 10 # 若已上传文件大于10个，则进行清理逻辑
      maxDays: 7 # 若存在超过7天的已上传文件，则进行清理逻辑
  upload:
    timerOffSet: 10000 # ms，启动初始化后，延迟进行车云请求
    deleteAfterUpload: true
url: "" # 服务器地址
timeout: 5000   # ms
    retryCount: 3
    retryInterval: 500 # ms
  remoteTasks:
    -taskid: 1
     task: uploadAllFileEvery10Min
    #-taskid: 2
    # task: uploadAllFileEvery30Min
    #-taskid: 3
    # task: uploadAllFileEvery60Min
    #-taskid: 4
    # task: uploadAllLogAfter5Min
    #-taskid: 5
    # task: uploadCalibrationImmediately
```







# 后记
## RAM Disk && UFS Disk
RAM Disk（RAM盘）和 UFS Disk（通用闪存存储盘）是两种不同的存储技术，它们在计算机和移动设备中扮演着重要的角色。

### `RAM Disk`：
- RAM Disk是利用计算机内存（RAM）的一部分来模拟一个磁盘驱动器。因为内存的读写速度远快于任何传统硬盘或固态硬盘，所以RAM Disk可以提供极高的数据读写速度。
- 它通常用于临时存储，可以显著提高某些应用程序的运行速度，比如数据库管理系统、游戏、图像编辑软件等。
- 但是，RAM Disk中的数据是易失性的，这意味着一旦断电，其中的数据就会丢失。因此，它不适合用于永久存储数据。

### `UFS Disk`：
- UFS（Universal Flash Storage）是一种存储规范，专为带有嵌入式存储的移动设备设计，如智能手机和平板电脑。UFS提供了比传统eMMC更快的读写速度，尤其是在处理大量数据时。
- UFS支持双通道，允许数据同时读写，这大大提高了其性能，尤其是在需要高数据吞吐量的应用场景中。
- UFS存储器是非易失性的，即使在电源断开的情况下，数据也能被保留，适合用于长期存储。

总结来说，RAM Disk提供极快的速度，但数据在断电后会丢失，适合用作临时存储来提升性能；
而UFS Disk则是一种快速、稳定的存储解决方案，适用于移动设备中的长期数据存储。