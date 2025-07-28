***https://cmake.org/cmake/help/latest/guide/tutorial/A%20Basic%20Starting%20Point.html***
# EX1 构建基本工程
最简单的cmake工程就是一个源文件最终编译成一个可执行文件。
cmake命令支持大小写，小写命令更易读。
## 三条基本命令：
1. 工程的顶级CMakeLists.txt必须使用cmake_minimum_required()指定CMake版本。
2. 创建工程：project()，紧跟在指定版本之后
3. add_executable()，根据指定的源文件创建可执行文件

见cmake官方示例（注释版本）



# 理解cmake 中`PRIVATE`、`PUBLIC`、`INTERFACE`的含义和用法
在 cmake 中，`PRIVATE`、`PUBLIC`、`INTERFACE`是用来指定目标之间的依赖关系的关键字，它们可以用在`target_link_libraries`、`target_include_directories`、`target_compile_definitions`等指令中，用来控制依赖项的传递和范围

## `target_link_libraries`示例
### `PRIVATE`
表示`依赖项`只对当前目标有效，`不会传递`给其他依赖于当前目标的目标。
例如，如果你有一个库 libA，它依赖于另一个库 libB，
并且只在 libA 的源文件中使用了 libB 的功能，那么你可以这样写：
```sh
target_link_libraries(libA PRIVATE
                        libB)
```
这样，libA 就可以链接到 libB。

但是如果有一个可执行文件 app，它依赖于 libA，那么它不会自动链接到libB，
也不能使用 libB 的功能。
如果你想让 app 也链接到 libB，你需要显式地写出：
```sh
target_link_libraries(app libA libB)
```

### `INTERFACE`
表示`依赖项`只对`其他依赖于当前目标的目标`有效，不会影响当前目标本身。
例如，如果你有一个库 libA，它依赖于另一个库 libB，
并且只在 libA 的头文件中使用了 libB 的功能，那么你可以这样写：
```sh
target_link_libraries(libA INTERFACE
                        libB)
```
这样，**libA 就不会链接到 libB**，
但是如果有一个可执行文件 app，它依赖于 libA，那么它会`自动链接到 libB，并且可以使用 libB 的功能`。
这种情况通常发生在当前目标只是作为一个接口或者包装器，将另一个目标的功能暴露给其他目标。

### `PUBLIC`
表示依赖项既对当前目标有效，也会传递给其他依赖于当前目标的目标。
例如，如果你有一个库 libA，它依赖于另一个库 libB，并且在 libA 的源文件和头文件中都使用了 libB 的功能，那么你可以这样写：
```sh
target_link_libraries(libA PUBLIC libB)
```
这样，libA 就会链接到 libB，并且如果有一个可执行文件 app，它依赖于 libA，那么它也会自动链接到 libB，并且可以使用 libB 的功能。这种情况通常发生在当前目标既使用了另一个目标的功能，也将其暴露给其他目标。



## `target_include_directories`示例
用于指定目标包含的头文件路径。
`PRIVATE`、`PUBLIC`和`INTERFACE`，分别表示不同的作用域和传递方式。

### `PRIVATE`
表示头文件只能被目标本身使用，不能被其他依赖目标使用。
例如，如果你有一个库`libhello-world.so`，
它只在`自己的源文件中`包含了`hello.h`，而`不在对外的头文件`hello_world.h中包含，那么你可以写：
```sh
target_include_directories(hello-world PRIVATE hello) 
```
这样，其他依赖 libhello-world.so的目标（比如一个可执行文件）就不会自动包含hello.h。

### `PUBLIC`
表示头文件既能被目标本身使用，也能被其他依赖目标使用。
例如，如果你有一个库 libhello-world.so，它在自己的源文件和对外的头文件中都包含了 hello.h，那么你可以写：
```sh
target_include_directories(hello-world PUBLIC hello) 
```
这样，其他依赖 libhello-world.so 的目标就会自动包含 hello.h。

### `INTERFACE`
表示头文件不能被目标本身使用，只能被其他依赖目标使用。
例如，如果你有一个库 libhello-world.so，它只在对外的头文件中包含了 hello.h，而不在自己的源文件中包含，那么你可以写：
```sh
target_include_directories(hello-world INTERFACE hello) 
```
这样，其他依赖 libhello-world.so 的目标就会自动包含 hello.h，但是 libhello-world.so 本身不会。

一些示例：
如果你有一个`静态库或者动态库`，它需要`暴露一些头文件`给其他目标使用，
    那么你可以使用 PUBLIC 或者 INTERFACE 关键字。
如果你有一个`可执行文件或者测试程序`，它需要包含一些内部的头文件，而不需要暴露给其他目标使用，
    那么你可以使用 PRIVATE 关键字。
如果你有一个`接口库`（INTERFACE library），它只定义了一些接口或者抽象类，并没有实现任何功能，
    那么你可以使用`INTERFACE`关键字









