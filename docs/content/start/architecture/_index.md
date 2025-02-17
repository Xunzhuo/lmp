---
title: LMP 架构设计
weight: 10
disableToc: true
---

## 逻辑层类图注解
## PluginInfo

主要作用：主要用于记录所有的插件（包括BCC、Shell、纯C等）的共有信息。

主要成员及其作用：

 1. `ExecPath`：该成员用于记录对应插件的执行文件的路径信息；
 2. `Runtime`：当有一个执行对应插件的命令下发后，该成员用于记录本次其插件执行的时间；
 3. `PluginState`：用于记录所属的插件的一个状态；
   目前插件状态主要以下三类：
   - `PluginSleeping`：代表所属插件可以接收执行命令；
   - `PluginRunning`：代表所属插件正在被执行，此时无法接收再次执行该插件的命令；
   - `PluginInvalid`：代表该插件在初始化或执行时发生错误，此时无法接收执行该插件的命令。
 4. `LinuxVersion`：用于记录该插件适用的内核版本信息（后续迭代中实现）；
 5. `DescPath`：用于记录该插件对应的注解文档的路径信息，可以用于前端拓展展示插件具体信息的功能实现（后续爹地啊中实现）；

主要方法：

 1. `EnterRun()`：对应插件执行之前，负责进行状态检查以及运行时成员信息的设置；
 2. `ExitRun()`：对应插件执行完成之后，负责设置该插件状态为可用。

## BccPlugin、CbpfPlugin、ShellPlugin

主要作用：充当`PluginInfo`的子类，继承`PluginInfo`适用于所有插件的成员信息以及`EnterRun()`和`ExitRun()`两种方法；并且单独实现自己独立的`Run()`方法（不同插件可能执行的方式存在差异）。

 > 在Go语言中，采用内嵌匿名结构体的方式，继承父类的成员与方法。

 1. `BccPlugin`：对应于每个BCC插件；
 2. `CbpfPlugin`：对应于每个纯C编写的eBPF插件；
 3. `ShellPlugin`：对应于每个Shell编写的脚本插件；

无论是`BccPlugin`、`CbpfPlugin`、`ShellPlugin`，还是后续要添加的其他插件类型，都要实现自己的`Run()`方法。

主要方法：

 `Run()`：在确定该插件可以执行之后，主要负责具体调用该插件的执行文件的不同方式。 

 ## Plugin

 主要作用：抽象出来的插件总类，便于统一管理。

 > 在Go语言中，`Plugin`使用`interface{}`实现，主要方法就是`Run()`。
 > 在对应的具体插件类型中，可以使用**指针接收**的方式实现`Run()`方法。
 > 好处：
 > 1. 实现了泛化；
 > 2. 可以让存储单位是指针，而不是值。

 ## PluginStorage

 主要作用：存储所有插件的指针，具体实现统一管理。

 主要成员：

  1. `Mp`：以指标的名称为key，value为该指标对应的插件的指针。

主要方法：

 1. `GetPlugin()`：在`Mp`中根据指标名称，获得对应插件的指针，方便进行后续的操作；
 2. `RunPlugines()`：根据controller层传递过来的信息，调用对应的插件的`Run()`方法。
 3. `InitPlugines()`：用于初始化成员`Mp`。

 > 如果想要实现不同的插件拥有不同的执行时间，可以让`RunPlugines()`的参数为一个map，key为要执行的指标名称，value为对应指标的执行时间；
 > 如果不需要此功能，可以让`RunPlugines()`的参数为一个字符串切片 和 一个数字，字符串切片用于记录要执行的指标名称，数字为要执行的时间。

 > 当前可以遍历文件目录的方式初始化；
 > 后续迭代可以采用去数据库查表的方式初始化。