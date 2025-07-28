# mcap接口


## 数据格式


## 初始化
涉及到以下几个类：
### `Schema`
1. name
"smart_em_msgs/msg/LaneLines"

2. encoding
"ros2msg"

3. data，字节数组
存储着以上encoding

### `Channel`
1. topic
"sensor/smart_camera/dynamic_objects"

2. messageEncoding
"cdr"

3. schemaId

4. metadata，是key_value_map
4.1 key
offered_qos_profiles
4.2 
调用`QosToString(TOPIC_QOS_DEFAULT)`得到的string

### `Message`


### `McapWriter`
当前是唯一的

需要调用`addSchema`，`addChannel`
    
### `McapWriterOptions`
1. profile
"ros2"

2. compression
Lz4




# 落盘机制
## proxy_service.cpp
`receive handler`中，
每个service持有若干`DataQueue<data_type>`类型的对象。
触发`receive handler`时，调用`ConvertToIdl`，将收到数据转换为dds格式，往相应的`DataQueue<data_type>`对象`Push`数据，等待后续处理。

每个service根据event数量，有相应的callback。
每个service会对应一个运行的线程，someip_to_dds初始化时执行`StartDDSDataConsumers()`，其中（根据event个数）创建线程，轮询地调用上述的callback：
该callback调用`ConvertToMsg`，将该`DataQueue<data_type>`对象的数据存储到全局唯一的`DataQueue<mcap::Message>`类型的`mcap_msg_queue_`


```cpp
void SensorSmartcameraOutputServiceInterface::ReceiveHandlerServiceEventSmartCameraLaneLinesTopic() {
  service_proxy_->SmartCameraLaneLinesTopic.GetNewSamples(
    [this](ara::com::SamplePtr< const sensor_smartcamera_output_service::proxy::events::SmartCameraLaneLinesTopic::SampleType> ptr_sample) {
    	log_.LogDebug() << "[Event][SmartCameraLaneLinesTopic] call the FastDDS forwarding function";
        if(!service_->stop_record_){
           // convert smartcameralanelinestopic someip data to the dds data
           ::dds_types::data_type::LaneLines dds_data;
           type_convert::ConvertToIdl(*ptr_sample, dds_data); 
            // push the lanelines dds data to the queue
            if(smartcameralanelinestopic_events_.Push(dds_data)){
              log_.LogInfo() << "[Event][SmartCameraLaneLinesTopic] push success.";   
            } else {                                                                 
             log_.LogInfo() << "[Event][SmartCameraLaneLinesTopic] failed to push.";
          }
        }
    });
}
```


## someip_to_dds.cpp
在初始化时，单独创建线程，调用`WriteMcapMsgToDisk()`，将存储的mcap类型数据写入磁盘。

维护一个`DataQueue<mcap::Message>`类型的`mcap_msg_queue_`，把mcap类型的数据一个个地`Pop`出来

当前设计是，每一分钟存储一个mcap文件，文件中存储多种类型的数据。并且，如果RAM DISK满了，就删除最老的文件。

```cpp
void SomeipToDds::WriteMcapMsgToDisk() {
    auto participant = SomeipToDdsParticipant::GetInstance();
    std::string filename;
    std::string tmp_filename;
    std::mutex lock_mutex;
    while (!exit_requested_) {
        mcap::Message msg;
        mcap_msg_queue_.Pop(msg);

        // write msg to disk using mcap API
        filename = create_timestamp_filename();
        if (tmp_filename != filename && !tmp_filename.empty()) {
          {
            std::lock_guard<std::mutex> close_lock(lock_mutex);
            participant->GetWriter()->close();
          }
          log_.LogInfo() << "Close the previous file: " << filename;
          {
            std::lock_guard<std::mutex> close_lock(lock_mutex);
            participant->GetWriter()->addChannel(*(participant->GetSensorSmartCameraDynamicObjectsChannel()));
            participant->GetWriter()->addChannel(*(participant->GetSmartCameraLaneLinesChannel()));
          }
        }

        while(!CheckRamDiskSpace())
        {
          // 如果空间不足
          std::string ram_disk_oldest_mcap_file_name;
          if(GetRamDiskOldestMcapFileName(ram_disk_oldest_mcap_file_name))
          {
            if (std::remove(ram_disk_oldest_mcap_file_name.c_str()) == 0) {
                log_.LogInfo() << "ram_disk_oldest_mcap_file: " << ram_disk_oldest_mcap_file_name <<" was successfully deleted.\n";
            } else {
                log_.LogError() << "Failed to delete the file.\n";
            }
            // 删不了文件会一直循环
            // 更新文件内容
            UpdateRamDiskOldestFile();
          }
          else {
            UpdateRamDiskOldestFile(filename);
          }
        }

        struct stat ram_oldest_file_stat;
        if(access(ram_oldest_file_name, F_OK) != 0)
        {
          UpdateRamDiskOldestFile(filename);
        }

        auto s = participant->GetWriter()->open(filename,*(participant->GetOptions()));
        if (!s.ok()) {
          log_.LogError() << "Failed to open mcap writer: " << s.message;
        }

        if(msg.data) {
          auto ret = participant->GetWriter()->write(msg);
          if (!ret.ok()){
            log_.LogError() << "Write message faild: " << ret.message;
          } else {
            log_.LogInfo() << "Write message success.";
          }
        }
        else {
          log_.LogInfo() << "message is empty.";
        }
        

        tmp_filename = filename;
    }
    participant->GetWriter()->close();
    log_.LogInfo() << "Close the last file: " << filename;
}
```





# 线程安全队列
```cpp
#pragma once

#include <atomic>
#include <cassert>
#include <cstddef>
#include <condition_variable>
#include <cstddef>
#include <functional>
#include <mutex>
#include <queue>

namespace linearx {
namespace dq {
template <typename T>
  class DataQueue {
      std::mutex mutex_;
      std::condition_variable readerCv_;
      std::condition_variable writerCv_;
      std::condition_variable finishCv_;

      std::queue<T> queue_;
      bool done_;
      std::size_t maxSize_;

      // Must have lock to call this function
      bool Full() const {
        if (maxSize_ == 0) {
          return false;
        }
        return queue_.size() >= maxSize_;
      }

    public:
      DataQueue(std::size_t maxSize = 0) : done_(false), maxSize_(maxSize) {}

      bool Push(const T& item) {
        {
          std::unique_lock<std::mutex> lock(mutex_);
          while (Full() && !done_) {
            writerCv_.wait(lock); // 队列满，阻塞等待Pop中能够notify
          }
          if (done_) {
            return false;
          }
          queue_.push(std::move(item)); // 将数据存入队列
        }
        readerCv_.notify_one(); // 初始情况下，队列空，写完第一个数据的那一刻就可以读了
        return true;
      }

      bool Pop(T& item) {
        {
          std::unique_lock<std::mutex> lock(mutex_);
          while (queue_.empty() && !done_) {  // 队列空则无法读取
            readerCv_.wait(lock);
          }
          if (queue_.empty()) {
            assert(done_);
            return false;
          }
          item = std::move(queue_.front()); // 成功取出数据，是std::move
          queue_.pop();
        } // 线程自从调用wait被唤醒后，重新获取互斥量
        writerCv_.notify_one();
        return true;
      }

      void SetMaxSize(std::size_t maxSize) {  // 不SetMaxSize的话，队列容量最大为1
        {
          std::lock_guard<std::mutex> lock(mutex_);
          maxSize_ = maxSize;
        }
        writerCv_.notify_all();
      }

      void Finish() {
        {
          std::lock_guard<std::mutex> lock(mutex_);
          assert(!done_);
          done_ = true; // done_表示的是 这个队列是否可以关闭了
        }
        readerCv_.notify_all();
        writerCv_.notify_all();
        finishCv_.notify_all();
      }

      void WaitUntilFinished() {
        std::unique_lock<std::mutex> lock(mutex_);
        while (!done_) {
          finishCv_.wait(lock);
        }
      }
      bool TryPop(T& item) {  // TryPop：队列空就直接退出
        std::lock_guard<std::mutex> lock(mutex_);
        if (queue_.empty()){
          return false;
        }  
        item = std::move(queue_.front());
        queue_.pop();
        return true;
      }
  };
}
}
```




