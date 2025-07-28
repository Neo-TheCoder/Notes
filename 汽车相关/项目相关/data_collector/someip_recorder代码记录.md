

# `Init()`

## 关键类
```cpp
void SomeipRecorder::Init() {
  linearx::yaml::YamlParser::Instance()->Init("./etc/config.yaml");
  storage_mcap_.Init(this, &(this->log_));
  state_mgr_.Init(this, &(this->log_));
  record_command_subscriber_->Init();
  record_status_publisher_->Init();
  record_debug_publisher_->Init();
  // InitIOWatcher();
  LoadOfflineFilesUnderProduct();   // 加载离线配置文件
  LoadOfflineFilesUnderDebug();
}
```
`StorageMcap`对象、`StateMgr`对象、以及一些特定的fastdds控制消息的topic

## `StorageMcap`
```cpp
class StorageMcap
{
private:
    SomeipRecorder* someip_to_dds_ptr_;
    ara::log::Logger* log_ptr_;
public:
    StorageMcap();
    void Init(SomeipRecorder* someip_to_dds_ptr, ara::log::Logger* log_ptr);    // 传入SomeipRecorder指针，log指针
    ~StorageMcap();
    void OverWriteFile(const std::string& file_name, const std::string& content);
    std::string ReadFile(const std::string& filename);
    std::tm string_to_tm(const std::string &input);     // 解析时间戳
};
```

相关接口实现：
```cpp
void StorageMcap::OverWriteFile(const std::string& file_name, const std::string& content)
{
    // 打开文件，如果文件存在则截断其内容
    std::ofstream file(file_name, std::ios::out | std::ios::trunc); // 以输出模式打开，并清空已有内容

    if (!file.is_open()) {
        log_ptr_->LogError() << "Failed to open ram_oldest_file_name file for writing.";
    }

    // 写入新内容
    file << content;

    // 关闭文件流确保内容被正确写入磁盘
    file.close();
}
```
其中`close()`操作释放资源：文件描述符、缓冲区，`close()`操作会自动调用`flush()`：
将文件流中的输出缓冲区的内容立即写入到与之关联的文件中，而不必等待缓冲区满或者流被关闭。
这样可以确保缓冲区中的数据被及时写入文件，即使程序还没有结束或者文件流还没有被关闭。
也就是调用一次`flush()`就会实现一次同步。
随后，文件流和文件就不再关联。

## `RecorderWriter`
`SomeipRecorder`的成员对象之一为`std::shared_ptr<RecorderWriter>`，初始化时会构造对象
`init()中`，有`AddChannel`


## `StateMgr`
```cpp
class StateMgr
{

using RecordClock = std::chrono::system_clock;

private:
    SomeipRecorder* someip_to_dds_ptr_;
    ara::log::Logger* log_ptr_;

    StorageInformation storage_information_;
    RecorderStatus recorder_status_;
    CurrentRecordingStatus current_recording_status_;
    RecordClock recorder_clock_;
    std::mutex status_information_lock_mutex_; // 数据修改数据的锁

    
    
public:
    StateMgr();
    ~StateMgr();
    void InitCurrentRecordingStatus(std::vector<std::string> &topic_name_list); // 初始化TopicName
    void StartRecord(); // 开始计时，记录开始录制时间，设置录制的Enum值
    void StopRecord(); // 停止计时，设置录制的Enum值
    void StartSice(); // 开始切片
    void StopSlice(); // 停止切片
    void AddMcapFile(std::string file_name); // 增加分包文件记录
    void DeleteMcapFile(std::string file_name); // 删除分包文件记录
    void UpdateTopicMessageParameter();// 频率 计数 时间戳
    void UpdateStorageState(); // 调用接口，读取文件
    void SetError(RecorderError error);
    void Init(SomeipRecorder* someip_to_dds_ptr, ara::log::Logger* log_ptr);

    RecorderStatus GetRecorderStatus();
    CurrentRecordingStatus GetCurrentRecordingStatus(); 
    StorageInformation GetStorageInformation();
};
```



# `Run()`
```cpp
// ...
  service_control_cmd_2icc_service_interface_->StartDDSDataConsumers();
  service_driving_control_output_2icc_service_interface_->StartDDSDataConsumers();
  service_ego_motion_service_interface_->StartDDSDataConsumers();
  service_acc_debug_service_interface_->StartDDSDataConsumers();
  service_prediction_leadingobjects_service_interface_->StartDDSDataConsumers();
  service_parameter_service_interface_->StartDDSDataConsumers();
  service_driving_modemanager_2icc_service_interface_->StartDDSDataConsumers();
  service_hmi_driving_function_service_interface_->StartDDSDataConsumers();
  service_vehicle_data_output_a4soc_service_interface_->StartDDSDataConsumers();

  // StartIOWatcher();
  StartThreadPool();

  thread_pool_->AddTask([this]() { this->WriteMcapMsgUnderProduct(); });

// ...
```

## `WriteMcapMsgUnderProduct()`实现
以`秒`为单位，落入文件
```cpp
void SomeipRecorder::WriteMcapMsgUnderProduct() {
  auto recorder_writer = GetRecorderWriter();
  std::string filename;
  std::string latest_filename;
  mcap::Message msg;
  while (!mcap_msg_queue_.Done() && recorder_mode_) {
    if (!mcap_msg_queue_.Pop(msg)) {    // 只有当队列为空，才退出循环
      continue;
    }

    auto now = linearx::time::GetNowTimepoint();
    std::string filename = 
      linearx::time::FormatTimepointToStringV2(now).append(std::string(".mcap"));   // 得到当前时间戳文件名

    // 20240428_171413.mcap
    if (latest_filename.substr(0, 15) != filename.substr(0, 15) && 
        !latest_filename.empty()) {     // 文件名不相等，也就是自从上一个文件生成，已经过去了1秒钟，上一个文件完成1秒内的任务了
      recorder_writer->GetWriter()->close();    // 写MCAP文件尾，调用flush，
      recorder_writer->AddChannel();    // 调用若干topic的addChannel接口，由于会移除之前添加的channel，所以要再次AddChannel
    }

    while(CheckFileCountOverflow()) {   // 检查ram disk实际文件数目 是否 大于 yaml中的 2*(m + n)配置
      std::string& storage_path = 
        linearx::yaml::YamlParser::Instance()->GetProductMode().path;
      {
        std::lock_guard<std::mutex> locker(g_mcap_file_mutex);
        if (offline_mcap_files_product_.size() > 0) {
          auto first_it = offline_mcap_files_product_.begin();
          env::posix::DeleteFile(first_it->second);
          log_.LogInfo() << "Delete file:[" 
                         << first_it->second << "] under product mode";
          offline_mcap_files_product_.erase(first_it);  // 维护一个map，对应ram disk下的文件
        }
      }
    }

    if (latest_filename != filename) {  // 说明此刻要新建文件
      std::string file_path = 
        linearx::yaml::YamlParser::Instance()->GetProductMode().path + "/" + filename;;
      auto s = recorder_writer->GetWriter()->open(file_path, *(recorder_writer->GetOptions())); // open，以待write
      if (!s.ok()) {
        log_.LogError() << "Failed to open mcap file: " << s.message;
        continue;
      }
      std::lock_guard<std::mutex> locker(g_mcap_file_mutex);
      offline_mcap_files_product_[filename] = file_path;;
    }

    if(msg.data) {
      auto ret = recorder_writer->GetWriter()->write(msg);
      if (!ret.ok()){
        log_.LogError() << "Write message faild: " << ret.message;
      } 
    }
    latest_filename = filename;     // ！！！
  }
  recorder_writer->GetWriter()->close();    // 调用close()以落盘，会移除之前添加的channel
}
```

如果此时trigger触发了，就合并前10后5个文件，合并好了，落盘之后，通过`发出trigger status为“trigger finish”的通知`的方式来通知data collector


## `WriteMcapMsgUnderDebug()`
以`分钟`为单位
```cpp
void SomeipRecorder::WriteMcapMsgUnderDebug() {
  auto recorder_writer = GetRecorderWriter();
  std::string filename;
  std::string latest_filename;
  mcap::Message msg;
  while (!mcap_msg_queue_.Done() 
         && !stop_debug_record_ 
         && !recorder_mode_) {
    if (!mcap_msg_queue_.Pop(msg)) {
      continue;
    }

    auto now = linearx::time::GetNowTimepoint();
    std::string filename = 
      linearx::time::FormatTimepointToString(now).append(std::string(".mcap"));

    if (latest_filename.substr(0, 12) != filename.substr(0, 12) && 
        !latest_filename.empty()) {   // 只比较到 分钟
      recorder_writer->GetWriter()->close();
      recorder_writer->AddChannel();
    }

    while(!CheckRamDiskSpace()) {
      std::string& storage_path = 
        linearx::yaml::YamlParser::Instance()->GetDebugMode().path;
      {
        std::lock_guard<std::mutex> locker(g_mcap_file_mutex);
        if (offline_mcap_files_debug_.size() > 0) {
          auto first_it = offline_mcap_files_debug_.begin();
          env::posix::DeleteFile(first_it->second);
          log_.LogInfo() << "Delete file:[" 
                         << first_it->second << "] under debug mode";
          offline_mcap_files_debug_.erase(first_it);
        }
      }
    }

    if (latest_filename.substr(0, 12) != filename.substr(0, 12)) {
      std::string file_path = 
        linearx::yaml::YamlParser::Instance()->GetDebugMode().path + "/" + filename;;
      auto s = recorder_writer->GetWriter()->open(file_path, *(recorder_writer->GetOptions()));
      if (!s.ok()) {
        log_.LogError() << "Failed to open mcap file: " << s.message;
        continue;
      }
      std::lock_guard<std::mutex> locker(g_mcap_file_mutex);
      offline_mcap_files_debug_[filename.substr(0, 12)] = file_path;
    }

    if(msg.data) {
      auto ret = recorder_writer->GetWriter()->write(msg);
      if (!ret.ok()){
        log_.LogError() << "Write message faild: " << ret.message;
      } 
    }
    latest_filename = filename;
  }
  recorder_writer->GetWriter()->close();
}
```


## 每个service interface执行的`StartDDSDataConsumers()`
```cpp
void StartDDSDataConsumers() {
    dds_data_consumer_threads_.emplace_back([this] {
    ::dds_types::smart_func_debug_msgs::AccDebug data;
    while (!exit_requested_) {
      if( adaptivecruisecontroldebugtopic_events_.Pop(data)) {
        AdaptiveCruiseControlDebugTopicDataCallback(data);
        }
      }
    });

    dds_data_consumer_threads_.emplace_back([this] {
    ::dds_types::smart_func_debug_msgs::AccVisual data;
    while (!exit_requested_) {
      if( adaptivecruisecontrolvisualtopic_events_.Pop(data)) {
        AdaptiveCruiseControlVisualTopicDataCallback(data);
        }
      }
    });

}
```
每个service interface使用event数量的线程，持续地将接收到的SOME/IP数据转为mcap数据，存入唯一的`mcap_msg_queue_`


# 其他线程：自建线程的操作

## 每个service interface的event执行的`GetNewSamples()`
把dds数据类型push进队列（每个service interface维护的一个数据队列）
```cpp
  service_proxy_->AdaptiveCruiseControlDebugTopic.GetNewSamples(
    [this](ara::com::SamplePtr< const acc_debug_service::proxy::events::AdaptiveCruiseControlDebugTopic::SampleType> ptr_sample) {
    	log_.LogDebug() << "[Event][AdaptiveCruiseControlDebugTopic] call the FastDDS forwarding function";
        if(!service_->stop_record_){
           // convert adaptivecruisecontroldebugtopic someip data to the dds data
           ::dds_types::smart_func_debug_msgs::AccDebug dds_data;
           adaptivecruisecontroldebugtopic_window_.Add();
           adaptivecruisecontroldebugtopic_count_++;
           type_convert::ConvertToIdl(*ptr_sample, dds_data); 
            // push the accdebug dds data to the queue
            if(adaptivecruisecontroldebugtopic_events_.Push(dds_data)){
              log_.LogInfo() << "[Event][AdaptiveCruiseControlDebugTopic] push success.";   
            } else {                                                                 
             log_.LogInfo() << "[Event][AdaptiveCruiseControlDebugTopic] failed to push.";
          }
        }
    });
```
注意到：
```cpp
adaptivecruisecontroldebugtopic_window_.Add();
adaptivecruisecontroldebugtopic_count_++;
```

注意到：
```cpp
class Window {
  public:
    Window(int window_size);
    ~Window();
    void Add();
    double GetFrameRate();
    std::chrono::high_resolution_clock::time_point GetLastTimeStamp();
  private:
    std::deque<std::chrono::high_resolution_clock::time_point> time_queue_;
    double frame_rate_;
    int window_size_;
    std::chrono::high_resolution_clock clock_;
    static std::mutex s_mtx;
};
```

`Add()`
```cpp
void Window::Add() {
  std::chrono::high_resolution_clock::time_point new_data_time = clock_.now();
  std::lock_guard<std::mutex> locker(Window::s_mtx);
  if(time_queue_.size() >= window_size_) {
    time_queue_.push_back(new_data_time);
    time_queue_.pop_front();
    frame_rate_ = (float)(window_size_ - 1) * 1000 * 1000 * 1000 / 
      std::chrono::duration_cast<std::chrono::nanoseconds>(new_data_time - time_queue_.front()).count();
  } else if(time_queue_.size() != 0) {
    time_queue_.push_back(new_data_time);
    frame_rate_ = (float)(time_queue_.size() - 1) * 1000 * 1000 * 1000 / 
      std::chrono::duration_cast<std::chrono::nanoseconds>(new_data_time - time_queue_.front()).count();
  } else if(time_queue_.size() == 0) {
    time_queue_.push_back(new_data_time);
  }
}
```
`Add()`：
将“收到数据的时刻”入队，移除队首元素，计算`frame_rate_`

此处是使用`滑窗平均算法`，计算每个event收到的帧率

- `滑窗平均算法`
注意：滑窗平均算法计算消息的频率，收到第一帧数据后记下timestamp，依次记下后边的每帧数据的timestamp，依次计算帧率并且按照实际个数取平均值，当达到50个数据后，保持按照50来算帧率的平均值，新加一帧，删除最旧一帧；
各个状态的广播周期需要客户定义，广播的时候只广播当前的状态信息。



# fastdds线程的操作
`监听DDS控制消息`
```cpp
void RecorderCommandEventSubscriber::SubListener::on_data_available(DataReader* reader) {
  smart_platform_msgs::msg::RecorderCommand record_command;
  SampleInfo info;
  if (reader->take_next_sample(&record_command, &info) == ReturnCode_t::RETCODE_OK) {
    if (info.valid_data) {
      uint32_t cmd = record_command.command();
      service_->GetLogger().LogInfo() << "RecorderCommandEventSubscriber got record_command data, cmd no:" << cmd;
      service_->HandleRecorderCommandMsg(record_command);
    }
  }
}
```
根据其中的字段判断，收到的是什么控制消息
1. `COMMAND_DEBUG_RECORD_STOP`
```cpp
UpdateDebugRecordStatus(true);
```
使得`WriteMcapMsgUnderDebug`中的`while`退出，函数从而也退出

2. `COMMAND_DEBUG_RECORD_START`
```cpp
UpdateDebugRecordStatus(false);
thread_pool_->AddTask([this]() { WriteMcapMsgUnderDebug();});
```

以上两个函数操作`stop_debug_record_`标志位

3. `COMMAND_SWITCH_TO_DEBUG_MODE`
```cpp
UpdateRecorderMode(false);
```

`recorder_mode_`置为false
为`true`时，表示`product`，trigger才有意义
为`false`时，表示`debug`

4. `COMMAND_SWITCH_TO_PRODUCT_MODE`
```cpp
UpdateRecorderMode(true);
thread_pool_->AddTask([this]() { WriteMcapMsgUnderProduct();});
```

5. `COMMAND_TRIGGER`
```cpp
HandleTriggerEvent(recorder_command);
```

`HandleTriggerEvent`
```cpp
void SomeipRecorder::HandleTriggerEvent(smart_platform_msgs::msg::RecorderCommand& cmd) {
  std::string trigger_id = std::to_string(static_cast<unsigned>(cmd.trigger_id()));

  auto local_trigger_time = linearx::time::GetNowTimepoint();
  auto remote_trigger_time = linearx::time::UTCTimestampToTimepoint(cmd.trigger_timestamp_ns());
  auto trigger_time = local_trigger_time;

  log_.LogInfo() << "local trigger time: " << linearx::time::FormatTimepointToStringV2(local_trigger_time)
                 << " remote trigger time: " << linearx::time::FormatTimepointToStringV2(remote_trigger_time)
                 << " trigger_time: " << linearx::time::FormatTimepointToStringV2(trigger_time)
                 << " trigger_id: " << trigger_id;

  uint32_t before_secs = linearx::yaml::YamlParser::Instance()->GetProductMode().dump_before_trigger_in_sec;
  uint32_t after_secs  = linearx::yaml::YamlParser::Instance()->GetProductMode().dump_after_trigger_in_sec;

  std::chrono::system_clock::time_point dump_start_time;
  std::chrono::system_clock::time_point dump_end_time;

  if (linearx::time::FormatTimepointToStringV2(trigger_time) <= 
      linearx::time::FormatTimepointToStringV2(last_slice_endtime_)) {
    int fd = env::posix::NewAppendWritableFile(last_slice_dir_ + "/trigger_list.txt");
    std::string ttime_string = linearx::time::FormatTimepointToStringV2(trigger_time);
    env::posix::Write(fd, ttime_string.data(), ttime_string.size());
    env::posix::Write(fd, ",", 1);
    env::posix::Write(fd, trigger_id.data(), trigger_id.size());
    env::posix::Write(fd, "\n", 1);
    return;
  }

  dump_start_time = trigger_time - std::chrono::seconds(before_secs);
  dump_end_time = trigger_time + std::chrono::seconds(after_secs);
  last_slice_endtime_ = dump_end_time;

  log_.LogInfo() << "SomeipRecorder::HandleTriggerEvent, trigger time: " 
                 << linearx::time::FormatTimepointToStringV2(trigger_time)
                 << " dump_start_time: " << linearx::time::FormatTimepointToStringV2(dump_start_time)
                 << " dump_end_time: "   << linearx::time::FormatTimepointToStringV2(dump_end_time);

  const std::vector<std::string>& expect_files = 
    GetExpectFileNameByTimePoint(dump_start_time, dump_end_time); 
  thread_pool_->AddTask([=]() {
      std::vector<std::string> dump_files;
      std::vector<std::string> pending_dump_files;
      {

        // 3 4 5 6 7 8 
        std::lock_guard<std::mutex> locker(g_mcap_file_mutex);
        for (auto file : expect_files) {
          auto it = offline_mcap_files_product_.find(file);
          if (it != offline_mcap_files_product_.end() &&
              it->first != linearx::time::FormatTimepointToStringV2(trigger_time).append(".mcap")) {
              dump_files.emplace_back(it->second);
          } else {
              pending_dump_files.emplace_back(file);
          }
        }
      }

      std::string base_dir = linearx::yaml::YamlParser::Instance()->GetProductMode().dump_path;
      std::string sub_dir = std::string("trigger_") + trigger_id + std::string("_") + 
                            linearx::time::FormatTimepointToStringV2(trigger_time);
      std::string slice_dir = base_dir + "/" + sub_dir;
      last_slice_dir_ = slice_dir;
      log_.LogInfo() << "Creating dir: " << slice_dir;
      if (!env::posix::CreateDir(slice_dir)) {
        log_.LogError() << "SomeipRecorder::HandleDumpEvent, failed to CreateDir. ";
        return;
      }

      for (auto file : dump_files) {
        if (env::posix::FileExists(file)) {
          std::string dest_path = slice_dir + std::string("/") + 
                                  std::string(basename(const_cast<char*>(file.c_str())));
          recorder_merger_->Add(dest_path);
          env::posix::CopyFileWithRange(file, dest_path);
          log_.LogInfo() << "Copying from " << file << " to " << dest_path;
        }   
      }

      std::this_thread::sleep_for(std::chrono::seconds(after_secs + 1));
      for (auto file : pending_dump_files) {
        auto it = offline_mcap_files_product_.find(file);
        if (it != offline_mcap_files_product_.end()) {
          if (env::posix::FileExists(it->second)) {
            std::string dest_path =
              slice_dir + std::string("/") + std::string(basename(const_cast<char*>(it->second.c_str())));
            recorder_merger_->Add(dest_path);
            env::posix::CopyFileWithRange(it->second, dest_path);
            log_.LogInfo() << "Copying  pending_dump_files from "
                           << it->second << "to " << dest_path;
          }
        }
      }

      std::string output_file = slice_dir + std::string("/") + sub_dir.append(".mcap");
      recorder_merger_->SetOutputFile(output_file);
      log_.LogInfo() << "RecoderMerger Will merge files to " << output_file;
      recorder_merger_->Merge();
  });
}
```

6. `COMMAND_CLEAR_DEBUG_STORAGE_POOL`
...


7. `COMMAND_CLEAR_PRODUCT_STORAGE_POOL`
...




