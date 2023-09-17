***https://cmake.org/cmake/help/latest/guide/tutorial/A%20Basic%20Starting%20Point.html***
# EX1 构建基本工程
最简单的cmake工程就是一个源文件最终编译成一个可执行文件。
cmake命令支持大小写，小写命令更易读。
## 三条基本命令：
1. 工程的顶级CMakeLists.txt必须使用cmake_minimum_required()指定CMake版本。
2. 创建工程：project()，紧跟在指定版本之后
3. add_executable()，根据指定的源文件创建可执行文件

见cmake官方示例（注释版本）


