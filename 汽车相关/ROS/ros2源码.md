# ROS2的整体架构

从上往下：
## User Application

## rclcpp  (C++ API)
即：`ROS2 C++ Client Library`
是对外接口：`订阅、发布topic、调用服务等`

## rcl  (C API  / C++ Implementaion)
即：`ROS2 Common Libraries`
通用C库，提供节点管理、参数管理、日志记录、时钟管理等

## rmw  (C API)
即`ROS Middleware Interface`
定义了ROS2中不同中间件（FastRTPS、RTI Connext、RTI Connext Dynameic）与ROS2系统其他部分（rclcpp，rcl）之间的通信接口标准



