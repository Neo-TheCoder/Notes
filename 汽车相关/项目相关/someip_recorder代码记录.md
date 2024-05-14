

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

`WriteMcapMsgUnderProduct()`实现
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





