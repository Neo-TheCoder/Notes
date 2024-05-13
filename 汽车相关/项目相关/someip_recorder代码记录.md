
# 关键类
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
    void Init(SomeipRecorder* someip_to_dds_ptr, ara::log::Logger* log_ptr);
    ~StorageMcap();
    void OverWriteFile(const std::string& file_name, const std::string& content);
    std::string ReadFile(const std::string& filename);
    std::tm string_to_tm(const std::string &input);
};
```


## ``
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


















