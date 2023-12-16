# 简介
**APP**位于AUTOSAR CP架构的最上层，承载着业务逻辑：例如，VCU的正常控制策略、BMS的SOC估计、MCU的电机控制。以此，才能适配不同的车型（vehicle）、不同的供应商（ECU）、不同的硬件（ECU）的应用开发。

## AUTOSAR CP APP的开发
1. 基于设计工具，开发APP中相关的SWC组件，在SWC定义相关的Internal Behavior，以及跟其他SWC以及BSW交互的接口

2. 根据在开发工具中设计的SWC的ARXML文件，生成该SWC的模板代码进行Coding（或者将其导入到Simulink中进行模型搭建）







