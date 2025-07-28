# Service通信方式

1. 生成service代码
```sh
colcon build --packages-select service_interface2
source ~/.bashrc
```

2. 生成应用层代码
```sh
colcon build --packages-select cpp_srvcli2
```

PS：注意生成文件的位置


## 命令行启动ros2 client发起service request
```sh
ros2 service call /service_name interface_package/srv/ServiceType "{request_data}"

ros2 service call /Method2 service_interface2/srv/Method2 "1"
/Method1
/Method2
```


/service_name是server端提供的service的名称。
interface_package是包含service接口定义的包名。
ServiceType是service的类型。
"{request_data}"是要发送给service的请求数据，根据实际情况填写。



ros2 service list


