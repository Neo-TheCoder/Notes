# 简介
**APP**位于AUTOSAR CP架构的最上层，承载着业务逻辑：例如，VCU的正常控制策略、BMS的SOC估计、MCU的电机控制。以此，才能适配不同的车型（vehicle）、不同的供应商（ECU）、不同的硬件（ECU）的应用开发。

## AUTOSAR CP APP的开发
1. 基于设计工具，开发APP中相关的**SWC组件**，在SWC定义相关的Internal Behavior，以及跟其他SWC以及BSW交互的接口

2. 根据在开发工具中设计的SWC的ARXML文件，生成**该SWC的模板代码**进行Coding（或者将其导入到Simulink中进行模型搭建）



# APP中SWC元素介绍
## Software Component
即SWC，软件组件，定义了不同类型的软件单元的**行为和交互接口**。
一般一个`xml`文件来描述一个`SWC`。
对应到代码，就是一个C文件：包含了**函数、接口、变量**的相关定义。
在AutoSAR 中 SWC 一般包括: 
`Application`、
`Sensor Actuator`、
`I/0 Hardware Abstraction`、
`Complex Driver`、
`Non-Volatile  Component`、
`Service components`类型的 SWC 设计。


### SWC之间的通信

SWC成员之间不是直接通信的，它们之间的通信是通过`虚拟功能总线（VBF）`进行的。
该总线是意义上的片内外通信的结合体，具体分为下列两部分：
#### 片内通信：
在片内SWC成员之间就是通过RTE通信，`一个SWC`可以把它理解为`一个.c文件`，.c文件之间的通信显然就只能通过全局变量去实现了。所以可以将ECU内部SWC的通信想象成全局变量去理解。

#### 片外通信：
片外很显然，肯定是通过物理总线去实现，在汽车电子中，目前应用最多的是`CAN Bus`。
如下图所示：各个SWC之间通过蓝色的线去通信，这个线可以把它当作上述描述的连接器（Connect，也就是上述所说的全局变量），其中，蓝色线的每个出口和入口都应该遵循标准的AutoSAR接口（Ports）。

#### SWC的分配
把上述的5个SWC分配到两个ECU中（实际上汽车里面也是这么做的）。
将车灯开关、调光控制器和左右顶灯放到一个ECU中由`车身顶部的一个芯片控制`；
将左右车门开关和车门开关逻辑单元放到专用的`车门ECU芯片`中控制。

`两个ECU`即为`两个控制器`，分别位于 车身前部的车门控制器 和 位于车身顶部的顶灯控制器。
ECU内部的SWC是通过`RTE`的管理来通信的；
而跨ECU的通信就是通过`外部总线`（一般为CAN，就是车身上连接各ECU的CAN双绞线束）。



## Port Interface
一个Component有明确定义的Port，在SWC之间交互

## Runnable
函数实体，承载用户层C代码
在SWC设计中一般分为三种：
`Init Runnable`、
`Cyclic Runnable`、
`Trigger Runnable`

## Data Element
决定可以通过端口的`数据类型`以及`数据`

## Data Type
**Base Type**（Uint、int、float）

**IDT**（Implementation Data Type）
    对基于类型的特殊定义，即type define

**Application Data Type**
    用于实际的物理类型，具备了真实的物理数据含义

## CompuMethod
定义**数据元素内部数据的转换关系**，通常一些，通过 CM，可以实现数据元素的转换，例简单的线性运算 y = ax + b，就是CM中配置a(factor)和 b(offset)实现的。同时对于一些枚举元素的定义也可以在 CM 中设置。

## TypeMapping Set
建立数据类型的 Mapping 关系，通常在使用 ADT 和 DT 中，两者之间的映射关系就是在 Type Mapping Set 中实现的.

## DataMapping
在SWC中将`CAN、LIN、ETH`等Com层的**信号**与对应的SWC的Portlnterface中的**数据元素**建立关联，实现对系统信号的读写操作







# CP专家说过的话
## 关于驱动
由于在具体项目拿到手的芯片，采用哪些引脚等具体细节都是根据具体项目而变化的，于是必然有人要承担写驱动的职责。
由于CP的面向底层的特性，需要在BSW层进行一定程度的开发。
CP也是包含了多个模块，如超时保护等。
Runnable按一定频率周期性运行。



