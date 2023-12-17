# 简介
**APP**位于AUTOSAR CP架构的最上层，承载着业务逻辑：例如，VCU的正常控制策略、BMS的SOC估计、MCU的电机控制。以此，才能适配不同的车型（vehicle）、不同的供应商（ECU）、不同的硬件（ECU）的应用开发。

## AUTOSAR CP APP的开发
1. 基于设计工具，开发APP中相关的**SWC组件**，在SWC定义相关的Internal Behavior，以及跟其他SWC以及BSW交互的接口

2. 根据在开发工具中设计的SWC的ARXML文件，生成**该SWC的模板代码**进行Coding（或者将其导入到Simulink中进行模型搭建）



# APP中SWC元素介绍
## Software Component
即SWC，软件组件，定义了不同类型的软件单元的**行为和交互接口**。
一般一个xml文件来描述一个SWC。
对应到代码，就是一个C文件，包含了**函数、接口、变量**的相关定义。







## Port Interface





