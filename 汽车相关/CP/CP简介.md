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
Application、 Sensor Actuator、I/0 Hardware Abstraction、 Complex Driver、 Non-Volatile  Component、Service components 类型的 SWC 设计。



## Port Interface
一个Component有明确定义的Port，在SWC之间交互

## Runnable
函数实体，承载用户层C代码
在SWC设计中一般分为三种：
Init Runnable、Cyclic Runnable、Trigger Runnable

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
rnbl按一定频率周期性运行。



