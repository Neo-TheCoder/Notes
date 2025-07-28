


# 第二章 主机规划与磁盘分区
### 2.1.3 各硬件设备在Linux中的文件名
在Linux系统中，每个设备都被当成一个文件来对待
例如：SATA接口的硬盘的文件名称即为/dev/sd\[a-d]
虚拟机内使用/dev/vd\[a-p]

## 2.2 磁盘分区
### 2.2.1 磁盘连接的方式与设备文件名的关系
Linux是使用**侦测到的顺序**来决定设备文件名，并非与实际插槽代号有关

一个硬盘有多个盘片（受体积和成本限制，一般在5片以内），每个盘片包含两个**盘面**，每个盘面对应一个读/写磁头。
盘片的编号自下向上从0开始，如最下边的盘片有0面和1面，再上一个盘片就编号为2面和3面。
一个盘面上，**磁道**是一圈圈的同心圆（从最外层开始编号为0），每个磁道上的一段弧，称为一个**扇区**。扇区是磁盘的最小组成单元（每个扇区通常是4096Byte）
具有相同编号的磁道形成一个圆柱（类似于卷筒纸芯的形状），称之为**柱面**，整个磁盘的柱面数和一个盘面上的磁道数自然是相等的
并且每个盘面都有自己的磁头，因此盘面数等于总的磁头数
磁盘容量计算：存储容量 ＝ 磁头数 × 磁道(柱面)数 × 每道扇区数 × 每扇区字节数
PS：新的硬盘数据的密度都一致，这样磁道的周长越长，扇区就越多，存储的数据量就越大。

**块**是操作系统中最小的逻辑存储单位。操作系统与磁盘打交道的最小单位是磁盘块。通俗的来讲，在Windows下如NTFS等文件系统中叫做簇；在Linux下如Ext4等文件系统中叫做块（block）。
每个簇或者块可以包括2、4、8、16、32、64…2的n次方个扇区。
操作系统与内存操作，是虚拟一个页的概念来作为最小单位。

扇区 <= 块/簇 <= page

整个磁盘的第一个扇区最为重要，因为记录了整个磁盘的重要信息（称为**MBR格式**：Master Boot Record）
但是由于近年来磁盘的容量不断扩大，造成读写上的一些困扰， 甚至有些大于2TB以上的磁盘分区已经让某些操作系统无法存取。因此后来又多了一个新的磁盘分区格式，称为**GPT（GUID partition table）**
硬盘分区后才能有效使用



# 第五章 Linux的文件权限与目录配置
Linux的特色就是**多用户多任务环境**
Linux一般将文件可存取的身份分为三个类别：
1. owner
2. group
3. others（非本群组外的其他人）
三个类别各自具有`read/write/execute`权限

## 5.1 使用者与群组
### 1 文件拥有者

### 2 群组
使得文件能够某些群组的成员访问，其余人不能访问，而一个成员可以同时属于多个群组
多个用户虽然在同一群组内，但是我们可以设置“权限”， 好让某些使用者个人的信息不被群组的拥有者查询，实现“私人空间”。 而设置群组共享，则可使得资源共享。
而root什么都能访问

Linux中，系统上的账号、root的相关信息，都记录在`/etc/passwd`，Linux所有的群组名称都记录在`/etc/group`，个人密码：`/etc/shadow`

## 5.2 Linux文件权限概念
### 5.2.1 Linux文件属性
```sh
drwxrwxrwx  4 test test 4096 Dec 26 10:09 .
drwxrwxrwx 10 test test 4096 Dec 26 17:51 ..
drwxrwxrwx  3 test test 4096 Dec 26 17:43 apd
drwxrwxrwx  2 test test 4096 Dec 26 16:28 data
-rwxrwxrwx  1 test test  692 Sep 27 13:45 someip_event_method_field.xml
```
#### 第一栏
文件属性：
`- rwx rwx ---`
1. 第一个字符表示文件的类型：
    d：目录
    -：文件
    l：链接文件
    b：设备文件中的可随机存取设备
    c：设备文件中的一次性读取设备（键盘、鼠标）

2. 第一组rwx表示：**文件拥有者**可具备的权限
3. 第二组rwx表示：加入此群组的账号的权限
4. 第三组rwx表示：非本人、且没有加入本群组的账号的权限

#### 第二栏
表示有多少文件名链接到此节点（i-node）
每个文件都会将他的权限与属性记录到文件系统的i-node中，不过，我们使用的目录树却是使用文件名来记录， 因此每个文件名就会链接到一个i-node。这个属性记录的，就是有多少不同的文件名链接到相同的一个i-node号码。 

#### 第三栏
表示这个文件（或目录）的拥有者账号

#### 第四栏
表示这个文件的所属群组

#### 第五、六栏
大小，单位为Byte
文件创建日期/最近修改日期

PS：对于目录这种文件而言，没有x（执行权限）就无法进入，因为无法`cd`


### 5.2.2 如何改变文件属性与权限
`chgrp`：change group
改变文件所属群组

`chown`
改变文件拥有者
```sh
sudo chown bin initial-setup-ks.cfg # 将文件拥有者改为bin账号
sudo chown root:root initial-setup-ks.cfg # 修改文件拥有者、群组，中间用冒号隔开
```
**使用场景**
`cp`文件给别人时：
```sh
[root@study ~]# cp .bashrc .bashrc_test
[root@study ~]# ls -al .bashrc*
-rw-r--r--. 1 root root 176 Dec 29 2013 .bashrc
-rw-r--r--. 1 root root 176 Jun 3 00:04 .bashrc_test # 复制时，新文件的属性没变
```

`chmod`
改变文件的权限（SUID、SGID、SBIT等等）
```sh
[root@study ~]# chmod [-R] xyz 文件或目录
选项与参数：
xyz : 就是刚刚提到的数字类型的权限属性，为 rwx 属性数值的相加。
-R : 进行递回（recursive）的持续变更，亦即连同次目录下的所有文件都会变更

# 数字方式
chmod 644 .bashrc

#字符方式
chmod u=rwx,go=rx .bashrc

# 去掉所有人的可执行权限
chmod a-x .bashrc
```

### 5.2.3 目录与文件之权限意义
#### 对于文件
文件是实际含有数据的地方，包括一般文本文件、数据库内容档、二进制可可执行文件（binary program）等等。 因此，权限对于文件来说，他的意义是这样的：
1. r （read）：
    可读取此一文件的实际内容，如读取文本文件的文字内容等；
2. w （write）：
    可以编辑、新增或者是修改该文件的内容（但**不含删除**该文件）；
3. x （eXecute）：
该文件具有可以被系统执行的权限。

#### 对于目录
目录主要的内容在记录文件名清单
1. r （read contents in directory）：
    表示具有读取目录结构清单的权限，所以当你具有读取（r）一个目录的权限时，表示你可以查询该目录下的文件名数据。
2. w （modify contents of directory）：
    表示你具有修改该目录结构清单的权限：
        创建新的文件与目录；
        删除已经存在的文件与目录（不论该文件的权限为何！）
        将已存在的文件或目录进行更名；
        搬移该目录内的文件、目录位置。
3. x （access directory）：
    **目录的x**代表的是使用者**能否进入该目录成为工作目录**（即当前目录）的用途

对一般文件而言，rwx主要是针对“文件的内容”来设计权限；
对目录来说，rwx则是针对“目录内的文件名列表”来设计权限。


### 5.2.4 Linux文件种类与扩展名
任何设备在Linux下面都是**文件**

#### 正规文件（regular file）
`ls -al`所显示出来的属性：第一个字符为`-`

##### 纯文本文件（ASCII）：
这是Linux系统中最多的一种文件类型，称为纯文本文件
举例来说，你可以下达`cat ~/.bashrc`就可以看到该文件的内容。

##### 二进制档（binary）：
系统其实仅认识且可以执行二进制文件（binary file）。
Linux中的可执行文件（scripts, 文字体批处理文件不算）就是这种格式。
举例来说，刚刚下达的指令`cat`就是一个binary file。

##### 数据格式文件（data）：
有些程序在运行的过程当中会读取某些特定格式的文件，
那些**特定格式**的文件可以被称为数据文件（data file）。
举例来说，Linux在使用者登陆时，都会将登录的数据记录在`/var/log/wtmp`那个文件内，该文件是一个data file，他能够通过`last`这个指令读出来！ 但是使用cat时，会读出乱码。因为他是属于一种特殊格式的文件。


#### 目录（directory）
`d`

#### 链接文件（link）
类似Windows系统下面的**快捷方式**
第一个属性为`l`

#### 设备与设备文件（device）
与系统周边及储存等相关的一些文件， 通常都集中在`/dev`这个目录之下。通常又分为两种：
1. 区块（block）设备文件
就是一些储存数据， 以提供系统随机存取的周边设备（如：硬盘与软盘）
你可以随机的在硬盘的不同区块读写，这种设备就是区块设备

2. 字符（character）设备文件
一些序列埠的周边设备（如键盘、鼠标等等）
特色就是**一次性读取**的，不能够截断输出
举例来说，你不可能让鼠标“跳到”另一个画面，而是“连续性滑动”到另一个地方啊！

#### 数据接口文件（sockets）
这种类型的文件通常被用在网络上的数据承接
我们可以启动一个程序来监听用户端的要求， 而用户端就可以通过这个socket来进行数据的沟通。
常在`/run`或`/tmp`这些个目录中看到这种文件类型

#### 数据输送档（FIFO, pipe）
FIFO也是一种特殊的文件类型，他主要的目的在解决多个程序同时存取一个文件所造成的错误问题

PS：一个Linux文件能不能被执行，与他的第一栏的十个属性有关， 与文件名根本一点关系也没有。并且，具有“可执行的权限”以及“具有可执行的程序码”是两回事。


# 第十七章 认识系统服务（daemons）
## 17.1 什么是`daemon`与服务(`service`)
服务：常驻的程序（`daemon`），可提供一些系统或网络功能（`service`）

一般来说，当我们以文字模式或图形模式 （非单人维护模式） 完整开机进入 Linux 主机后，系统已经提供我们很多的服务：包括打印服务、工作调度服务、邮件管理服务等等。

### 17.1.1 早期`System V`（最初由AT&T开发并发布的Unix操作系统的版本）的`init`管理行为中 daemon 的主要分类
对于SystemV这一版本而言，我们启动系统服务的管理方式被称为`SysV的init脚本程序`的处理方式。操作系统内核调用的第一个程序是`init`。然后 init 去唤起所有的系统所需要的服务，包括本机服务或网络服务。

####  init 的管理机制的特色
1. 服务的启动、关闭与观察等方式：
所有的服务启动脚本通通放置于`/etc/init.d/`下面，基本上都是使用`bash shell script`所写成的脚本程序。
当需要`启动、关闭、重新启动、观察状态`时， 可以通过如下的方式来处理：
  - 启动：/etc/init.d/daemon start
  - 关闭：/etc/init.d/daemon stop
  - 重新启动：/etc/init.d/daemon restart
  - 状态观察：/etc/init.d/daemon status

2. 服务启动的分类
  - 独立启动模式（stand alone）
服务独立启动，该服务直接`常驻于内存`中。
它提供本机或用户的服务行为，`反应速度快`。

  - 总管程序（super daemon）
由特殊的`xinetd`或`inetd`这两个总管程序提供`socket对应`或`port对应`的管理。
当没有用户要求某socket或port时，所需要的服务是不会被启动的。
若有用户要求时， `xinetd总管`才会去唤醒相对应的服务程序。
当该要求结束时，这个服务也会被结束掉。也就是**按需启动**。
因为通过`xinetd`所总管，因此这个家伙就被称为`super daemon`。
好处是可以通过`super daemon`这一程序来进行服务的时程、连线需求等的控制。
缺点是唤醒服务需要一点`时间的延迟`。

3. 服务的相依性问题
服务是可能会有相依性的。
例如，你要启动网络服务，但是系统没有网络， 那怎么可能可以唤醒网络服务呢？
如果你需要连线到外部取得认证服务器的连线，但该连线需要另一个A服务的需求，问题是，A服务没有启动， 因此，你的认证服务就不可能会成功启动的！
这就是所谓的服务相依性问题。init 在管理员自己**手动**处理这些服务时，是**没有办法**协助相依服务的唤醒的！

4. 执行等级的分类
init 是开机后内核主动调用的，然后 init 可以根据`使用者自订`的`执行等级`（runlevel） 来唤醒不同的服务，以进入不同的操作界面。
基本上 Linux提供 7 个执行等级，其中比较重要的是：
1）单人维护模式
3）纯文本模式
5）文字加图形界面
而各个执行等级的启动脚本是通过`/etc/rc.d/rc[0-6]/SXXdaemon`链接到`/etc/init.d/daemon`。
链接文件名 （SXXdaemon） 的功能为： S为启动该服务，XX是数字，为启动的顺序。由于有 SXX 的设置，因此在开机时可以`“依序执行”`所有需要的服务， 同时也能解决相依服务的问题。这点与管理员自己手动处理不太一样就是了。

5. 制定执行等级默认要启动的服务
若要创建如上提到的 SXXdaemon 的话，不需要管理员手动创建链接文件，通过如下的指令可以来处理默认启动、默认不启动、观察默认启动否的行为：
  - 默认要启动： chkconfig daemon on
  - 默认不启动： chkconfig daemon off
  - 观察默认为启动否： chkconfig --list daemon

6. 执行等级的切换行为
当你要从纯命令行（runlevel 3）切换到图形界面（runlevel 5）， 不需要手动启动、关闭该执行等级的相关服务，只要“ init 5 ”即可切换，init 这小子
会主动去分析`/etc/rc.d/rc[3 & 5].d/`这两个目录内的脚本， 然后启动转换 runlevel 中需要的服务，从而就完成整体的 runlevel 切换。

#### 总结
重要的指令包括 daemon 本身自己的脚本（`/etc/init.d/daemon`） 、`xinetd`这个特殊的总管程序 （`super daemon`）、设置默认开机启动的`chkconfig`， 以及会影响到执行等级的`init N`等。虽然 CentOS 7 已经不使用 init 来管理服务了，不过因为考虑到某些脚本没有办法直接塞入 systemd 的处理，因此这些脚本还是被保留下来，


### 17.1.2 `systemd`使用的`unit分类`
从 CentOS 7.x 以后，Red Hat 系列的 distribution 放弃沿用多年的 System V 开机启动服务的流程，就是前一小节提到的 init 启动脚本的方法，改用`systemd`这个启动服务管理机制。

#### 使用`systemd`的优点
1. 平行处理所有服务 加速开机流程
旧的 init 启动脚本是串行，即便是**不存在依赖关系的服务**也是需要等待前面的完成才能进行。
由于目前我们的硬件主机系统与操作系统几乎都支持**多核心架构**了，可以支持没有依赖关系的服务同时启动。

2. 一经要求就回应的`on-demand`启动方式
systemd全部就是仅有一只systemd服务搭配`systemctl`指令来处理，无须其他额外的指令来支持。
不像 systemV 还要 init, chkconfig, service... 等等指令。
此外，systemd由于**常驻内存**，因此任何要求（on-demand）都可以立即处理后续的 daemon 启动的任务，响应迅速。

3. 服务相依性的自我检查
由于 systemd 可以自订服务相依性的检查，因此如果 B 服务是架构在 A 服务上面启动的，那当你在没有启动 A 服务的情况下仅手动启动 B 服务时，systemd 会**自动**帮你启动 A 服务！
这样就可以免去管理员得要一项一项服务去分析的麻烦（例如：当你没有启动网络， 但却启动NIS/NFS时，那个开机时的 timeout 甚至可达到 10~30 分钟...）

4. 依 daemon 功能分类
systemd 旗下管理的服务非常多。
首先 systemd 先定义所有的服务为一个`服务单位 （unit）`，并将该unit归类到不同的服务类型（type）去。 
旧的 init 仅分为`stand alone`与`super daemon`实在不够看，systemd 将服务单位 （unit） 区分为：
`service`, `socket`, `target`, `path`, `snapshot`, `timer`等多种不同的类型（type）， 方便管理员的分类与记忆。

5. 将多个 daemons 集合成为一个群组
如同 systemV 的 init 里头有个runlevel的特色，systemd亦将许多的功能集合成为一个所谓的target项目，这个项目主要在设计操作环境的创建，所以是集合了许多的 daemons，亦即是**执行某个 target 就是执行好多个daemon**的意思！

PS："runlevel"（运行级别）是一个重要的概念，用于表示系统的不同状态或模式。
每个运行级别都对应着一组特定的系统服务和功能，系统可以根据需要切换不同的运行级别来实现不同的功能和行为。
SystemV的init系统中通常定义了多个运行级别，比如0到6级别，每个级别都有特定的含义和对应的服务。例如，`运行级别3`可能表示`多用户模式`，包含了`网络服务和图形界面`；而`运行级别1`可能表示`单用户模式`，只包含`最基本的系统服务`。
通过在SystemV的init系统中设置不同的运行级别，可以实现系统在不同场景下的灵活切换和配置。比如在维护模式下可以选择单用户模式（runlevel 1），在正常使用时可以选择多用户模式（runlevel 3或5）。每个运行级别都定义了系统启动时需要启动或关闭的服务，从而实现了系统在不同状态下的管理和控制。

6. 向下相容旧有的 init 服务脚本
基本上，systemd是可以相容于init的启动脚本的，因此，旧的 init 启动脚本也能够通过 systemd 来管理，只是更进阶的 systemd 功能就没有办法支持就是了。

#### 不过`systemd`也是有些地方无法完全取代`init`的
1. 在 runlevel 的对应上，大概仅有 runlevel 1, 3, 5 有对应到 systemd 的某些 target 类型而已，没有全部对应
2. 全部的 systemd 都用 systemctl 这个管理程序管理，而 systemctl 支持的语法有限制，不像 /etc/init.d/daemon 就是纯脚本可以自订参数，systemctl 不可自订参数。灵活性降低。
3. 如果某个服务启动是管理员自己手动执行启动，而不是使用 systemctl 去启动的 （例如你自己手动输入 crond 以启动 crond 服务），那么 systemd 将无法侦测到该服务，而无法进一步管理。
4. systemd 启动过程中，无法与管理员通过 standard input 传入讯息！因此，自行撰写systemd 的启动设置时，务必要取消互动机制～（连通过启动时传进的标准输入讯息也要避免！）

不过，光是`同步启动服务脚本`这个功能就可以节省你很多开机的时间。同时 systemd 还有很多特殊的服务类型 （type） 可以提供更多有趣的功能


#### systemd管理的`unit`
1. systemd 的配置文件放置目录
基本上， systemd 将过去所谓的 daemon 执行脚本通通称为一个服务单位 （unit），而每种服务单位依据功能来区分时，就分类为不同的类型 （type）。
基本的类型有包括`系统服务`、`数据监听与交换的插槽档服务` （socket）、`储存系统状态的快照类型`、`提供不同类似执行等级分类的操作环境（target）`等等
相关配置文件都在以下目录：
/usr/lib/systemd/system/
每个服务最主要的`启动脚本设置`，有点类似以前的`/etc/init.d`下面的文件

/run/systemd/system/
系统执行过程中所产生的服务脚本，这些脚本的`优先序`要 比`/usr/lib/systemd/system/`高

/etc/systemd/system/
管理员依据`主机系统的需求`所创建的执行脚本，其实这个目录有点像以前`/etc/rc.d/rc5.d/Sxx`之类的功能
执行优先序又比`/run/systemd/system/`高

也就是说，到底系统开机会不会执行某些服务其实是看`/etc/systemd/system/`下面的设置，所以该目录下面就是一大堆链接文件。
而**实际执行的 systemd 启动脚本配置文件**， 其实都是放置在`/usr/lib/systemd/system/`下面的。
因此如果你想要修改某个服务启动的设置，应该要去`/usr/lib/systemd/system/`下面修改才对！
`/etc/systemd/system/`仅是链接到正确的执行脚本配置文件而已。所以想要看执行脚本设置，应该就得要到 /usr/lib/systemd/system/ 下面去查阅才对

PS：/etc/systemd/system/目录中包含的是针对系统本地特定配置的符号链接。这些链接文件实际上指向/usr/lib/systemd/system/目录下的真实服务配置文件。
所以，这个目录下的设置决定了系统开机是否执行某些服务，它提供了本地系统管理者修改和定制服务启动行为的入口。
而实际的 systemd 启动脚本配置文件则位于/usr/lib/systemd/system/目录下。这些文件是系统所使用的默认配置，由软件包管理器提供和维护。一般情况下，**推荐在/etc/systemd/system/目录下创建或修改相应的符号链接，而不建议直接修改这些文件，因为它们可能会在软件包更新时被覆盖**。


2. systemd 的 unit 类型分类说明
`/usr/lib/systemd/system/`以下的数据如何区分上述所谓的不同的类型（type）呢？
--> 答案是根据扩展名（.sevice、.target）来判断

| 文件扩展名 | 主要服务功能 |
|  ----  | ----  |
| `.service`  | 一般服务类型 （service unit）：主要是系统服务，包括服务器本身所需要的本机服务以及网络服务都是！比较经常被使用到的服务大多是这种类型！|
| `.socket`  | 内部程序数据交换的插槽服务 （socket unit）：主要是 IPC （Inter-process communication） 的传输讯息插槽档 （socket file） 功能。 这种类型的服务通常在监控讯息传递的插槽档，当有通过此插槽档传递讯息来说要链接服务时，就依据当时的状态将该用户的要求传送到对应的daemon， 若 daemon 尚未启动，则启动该 daemon 后再传送用户的要求。使用 socket 类型的服务一般是比较不会被用到的服务，因此在开机时通常会稍微延迟启动的时间 （因为比较没有这么常用嘛！）。一般用于本机服务比较多，例如我们的图形界面很多的软件都是通过 socket 来进行本机程序数据交换的行为。 （这与早期的 xinetd 这个 super daemon 有部份的相似喔！） |
| `.target`  | 执行环境类型 （target unit）：其实是一群 unit 的集合，例如上面表格中谈到的 multi-user.target 其实就是一堆服务的集合～也就是说， 选择执行multi-user.target 就是执行一堆其他 .service 或/及 .socket 之类的服务就是了！ |
| `.mount` `.automount`  | 文件系统挂载相关的服务 （automount unit / mount unit）：例如来自网络的自动挂载、NFS 文件系统挂载等与文件系统相关性较高的程序管理。 |
| `.path`  | 侦测特定文件或目录类型 （path unit）：某些服务需要侦测某些特定的目录来提供伫列服务，例如最常见的打印服务，就是通过侦测打印伫列目录来启动打印功能！ 这时就得要.path 的服务类型支持了！ |
| `.timer` | 循环执行的服务 （timer unit）：这个东西有点类似 anacrontab 喔！不过是由 systemd 主动提供的，比 anacrontab 更加有弹性！ |

## 17.2 通过`systemctl`管理服务
基本上， systemd 这个启动服务的机制，主要是通过一只名为`systemctl`的指令来处理的！
跟以前systemV需要 service / chkconfig / setup / init等指令来协助不同，systemd只有一个`systemctl`命令

### 17.2.1 通过 systemctl 管理单一服务 （service unit） 的启动/开机启动与观察状态
一般来说，服务的启动有两个阶段：
1. `开机的时候`设置要不要启动这个服务
2. 你`现在`要不要启动这个服务

```sh
[root@study ~]# systemctl [command] [unit]
command 主要有：
start ：立刻启动后面接的 unit
stop ：立刻关闭后面接的 unit
restart ：立刻关闭后启动后面接的 unit，亦即执行 stop 再 start 的意思
reload ：不关闭后面接的 unit 的情况下，重新载入配置文件，让设置生效
enable ：设置下次开机时，后面接的 unit 会被启动
disable ：设置下次开机时，后面接的 unit 不会被启动
status ：目前后面接的这个 unit 的状态，会列出有没有正在执行、开机默认执行否、登录等信息等！
is-active ：目前有没有正在运行中
is-enable ：开机时有没有默认要启用这个 unit


PS：systemctl status命令是systemd系统管理器的一个命令，用于查看系统中某个服务的状态信息。通过执行systemctl status命令，可以获取服务的当前状态、最近一次状态变化的时间、服务所在的进程ID、服务的主要配置文件路径、服务所在的CGroup等信息。此外，该命令还可以显示服务的日志信息，以便管理员进行故障排查和问题分析。通常，管理员可以在执行systemctl status命令时，指定服务的名称或者使用通配符来查看所有服务的状态信息。总之，systemctl status命令是一个非常有用的工具，可以帮助管理员快速了解系统中各个服务的状态，从而更好地管理和维护系统。


# 范例一：看看目前 atd 这个服务的状态为何？
[root@study ~]# systemctl status atd.service
atd.service - Job spooling tools
Loaded: loaded （/usr/lib/systemd/system/atd.service; enabled）
Active: active （running） since Mon 2015-08-10 19:17:09 CST; 5h 42min ago
Main PID: 1350 （atd）
CGroup: /system.slice/atd.service
└─1350 /usr/sbin/atd -f
Aug 10 19:17:09 study.centos.vbird systemd[1]: Started Job spooling tools.
# 重点在第二、三行喔～
# Loaded：这行在说明，开机的时候这个 unit 会不会启动，enabled 为开机启动，disabled 开机不会启动
# Active：现在这个 unit 的状态是正在执行 （running） 或没有执行 （dead）
# 后面几行则是说明这个 unit 程序的 PID 状态以及最后一行显示这个服务的登录文件信息！
# 登录文件信息格式为：“时间” “讯息发送主机” “哪一个服务的讯息” “实际讯息内容”
# 所以上面的显示讯息是：这个 atd 默认开机就启动，而且现在正在运行的意思！

# 范例二：正常关闭这个 atd 服务
[root@study ~]# systemctl stop atd.service
[root@study ~]# systemctl status atd.service
atd.service - Job spooling tools
Loaded: loaded （/usr/lib/systemd/system/atd.service; enabled）
Active: inactive （dead） since Tue 2015-08-11 01:04:55 CST; 4s ago
Process: 1350 ExecStart=/usr/sbin/atd -f $OPTS （code=exited, status=0/SUCCESS）
Main PID: 1350 （code=exited, status=0/SUCCESS）
Aug 10 19:17:09 study.centos.vbird systemd[1]: Started Job spooling tools.
Aug 11 01:04:55 study.centos.vbird systemd[1]: Stopping Job spooling tools...
Aug 11 01:04:55 study.centos.vbird systemd[1]: Stopped Job spooling tools.
# 目前这个 unit 下次开机还是会启动，但是现在是没在运行的状态中！同时，
# 最后两行为新增加的登录讯息，告诉我们目前的系统状态喔！
```

不应该使用 kill 的方式来关掉一个正常的服务喔！否则 systemctl 会无法继续监控该服务的！ 那就比较麻烦。


将 cups 服务整个关闭：
```sh
[root@study ~]# systemctl stop cups.service
[root@study ~]# systemctl disable cups.service
rm '/etc/systemd/system/multi-user.target.wants/cups.path'
rm '/etc/systemd/system/sockets.target.wants/cups.socket'
rm '/etc/systemd/system/printer.target.wants/cups.service'
# 说明三个链接文件之间存在依赖关系
```

在关闭了cups.service的情况下，启动cups.socket，cups.service也会一并唤醒


### 17.2.2 通过 systemctl 观察系统上所有的服务











# 第十九章 开机流程、模块管理与Loader
核心得要**侦测硬件**并**载入适当的驱动程序**后，接下来则必须要调用程序来准备好系统运行的环境，以让使用者能够顺利的操作整部主机系统。

## 19.1 Linux的开机流程分析
### 19.1.1 开机流程一览
目前各大Linux发行版的主流开机管理程序（Boot Loader）是grub2。
按下电源键后，计算机硬件会主动地**读取BIOS（Basic Input Output System）或UEFI BIOS来载入硬件信息**及进行**硬件系统的自我测试**，然后系统就会主动**读取第一个可开机的设备**（由BIOS设置），然后就可以**读入开机管理程序**了。
**boot loader**：可以**指定使用哪个核心文件来开机，并实际载入核心到内存中解压缩和执行**。核心运行时会**侦测所有硬件信息与载入适当的驱动程序使得整部主机开始运行**，然后操作系统就在PC上跑起来了。此时，Linux才会调用外部程序开始准备软件执行的环境，并实际载入所有系统运行所需的软件程序。最后系统就会等待用户的登录与操作。

#### 系统的开机过程
1. 载入 BIOS 的硬件信息与进行自我测试，并依据设置取得第一个可开机的设备；
2. 读取并执行第一个开机设备内`MBR`（Master Boot Record, 主要开机记录区）的`boot Loader`（亦即是 grub2, spfdisk 等程序）；
3. 依据boot loader的设置载入`Kernel`，Kernel会开始侦测硬件与载入驱动程序；
4. 在硬件驱动成功后，Kernel会主动调用 `systemd` 程序，并以`default.target`流程开机；
    systemd 执行 `sysinit.target` 初始化系统及`basic.target`准备操作系统；
    systemd 启动 `multi-user.target` 下的本机与服务器服务；
    systemd 执行 `multi-user.target` 下的 `/etc/rc.d/rc.local` 文件；
    systemd 执行 `multi-user.target` 下的 getty.target 及登陆服务；
    systemd 执行 graphical 需要的服务

PS：`MBR`是磁盘最前面可安装boot loader的部分

### 19.1.2 BIOS,boot loader与kernel载入
在个人计算机架构下，你想要启动整部系统首先就得要让系统去载入`BIOS`（Basic Input Output System），并**通过BIOS程序去载入CMOS的信息**，并且借由CMOS内的设置值取得主机的各项硬件设置，例如：CPU与周边设备的沟通频率啊、开机设备的搜寻顺序啊、硬盘的大小与类型啊、系统时间啊、各周边总线的是否启动Plug and Play（PnP, 随插即用设备）啊、各周边设备的I/O位址啊、以及与CPU沟通的IRQ岔断等等的信息。

