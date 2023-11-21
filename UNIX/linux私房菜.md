


# 第十九章 开机流程、模块管理与Loader
核心得要**侦测硬件**并**载入适当的驱动程序**后，接下来则必须要调用程序来准备好系统运行的环境，以让使用者能够顺利的操作整部主机系统。

## 19.1 Linux的开机流程分析
### 19.1.1 开机流程一览
目前各大Linux发行版的主流开机管理程序（Boot Loader）是grub2。
按下电源键后，计算机硬件会主动地**读取BIOS（Basic Input Output System）或UEFI BIOS来载入硬件信息**及进行**硬件系统的自我测试**，然后系统就会主动**读取第一个可开机的设备**（由BIOS设置），然后就可以**读入开机管理程序**了。
**boot loader**：可以**指定使用哪个核心文件来开机，并实际载入核心到内存中解压缩和执行**。核心运行时会**侦测所有硬件信息与载入适当的驱动程序使得整部主机开始运行**，然后操作系统就在PC上跑起来了。此时，Linux才会调用外部程序开始准备软件执行的环境，并实际载入所有系统运行所需的软件程序。最后系统就会等待用户的登录与操作。

#### 系统的开机过程
1. 载入 BIOS 的硬件信息与进行自我测试，并依据设置取得第一个可开机的设备；
2. 读取并执行第一个开机设备内MBR（Master Boot Record, 主要开机记录区）的 boot Loader （亦即是 grub2, spfdisk 等程序）；
3. 依据boot loader的设置载入Kernel，Kernel会开始侦测硬件与载入驱动程序；
4. 在硬件驱动成功后，Kernel会主动调用 **systemd** 程序，并以 default.target 流程开机；
    systemd 执行 sysinit.target 初始化系统及 basic.target 准备操作系统；
    systemd 启动 multi-user.target 下的本机与服务器服务；
    systemd 执行 multi-user.target 下的 /etc/rc.d/rc.local 文件；
    systemd 执行 multi-user.target 下的 getty.target 及登陆服务；
    systemd 执行 graphical 需要的服务

PS：MBR是磁盘最前面可安装boot loader的部分

### 19.1.2 BIOS,boot loader与kernel载入
在个人计算机架构下，你想要启动整部系统首先就得要让系统去载入BIOS（Basic Input Output System），并**通过BIOS程序去载入CMOS的信息**，并且借由CMOS内的设置值取得主机的各项硬件设置，例如：CPU与
周边设备的沟通频率啊、开机设备的搜寻顺序啊、硬盘的大小与类型啊、系统时间啊、各周边总线的是否启动Plug and Play（PnP, 随插即用设备）啊、各周边设备的I/O位址啊、以及与CPU沟通的IRQ岔断等等的信息。








