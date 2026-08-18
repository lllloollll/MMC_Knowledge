# Android Framework 高级开发：SystemServer、AMS、Zygote 与 App 启动流程

> 本篇目标：建立 Android Framework 的**进程视角**，理解 `system_server`、`SystemServer`、AMS、ATMS、Zygote、ActivityThread、ServiceManager、Binder 之间的关系，并梳理一个未运行 App 从桌面点击到 Activity 创建的整体流程。

---

# 一、Android Framework 整体架构

Android 系统启动后，可以先建立如下整体模型：

```text
Linux Kernel
    ↓
init
    ↓
Zygote
    ↓ fork
system_server
    ↓
SystemServer.main()
    ↓
启动 Framework 系统服务
    ↓
AMS / ATMS / PMS / WMS / ...
    ↓
启动 Launcher
    ↓
启动普通 App
```

其中两个非常重要的 Linux 进程是：

```text
Zygote
system_server
```

普通 App 又分别运行在自己的 Linux 进程中。

例如：

```text
system_server
com.android.launcher
com.example.app
com.tencent.mm
```

这些都是不同的 Linux 进程，拥有不同的虚拟地址空间。

---

# 二、system_server 和 SystemServer 的区别

这是非常容易混淆的一组概念。

## 1. system_server

`system_server` 是一个 **Linux 进程**。

Android Framework 中大量核心系统服务都运行在这个进程中。

例如：

```text
system_server
    │
    ├── ActivityManagerService
    ├── ActivityTaskManagerService
    ├── WindowManagerService
    ├── PackageManagerService
    ├── PowerManagerService
    ├── InputManagerService
    └── ...
```

所以：

```text
system_server = 进程
```

---

## 2. SystemServer

`SystemServer` 是一个 **Java 类**，是 system_server 进程进入 Java Framework 世界后的重要入口。

整体关系可以理解为：

```text
Zygote
    ↓
fork
    ↓
system_server 进程
    ↓
SystemServer.main()
    ↓
初始化 Android Framework
    ↓
启动各种系统服务
```

因此不要把它们混为一谈：

```text
system_server
    ↓
Linux 进程

SystemServer
    ↓
Java 类 / Framework 初始化入口
```

---

# 三、SystemServer 的作用

SystemServer 最核心的职责之一就是：

> 启动 Android Framework 中大量系统服务。

可以粗略分为：

```text
SystemServer
    │
    ├── Bootstrap Services
    │
    ├── Core Services
    │
    └── Other Services
```

常见的重要系统服务：

|服务|主要职责|
|---|---|
|AMS|进程、Service、Broadcast、Provider 等管理|
|ATMS|Activity、Task、Activity 启动等管理|
|PMS|Package、APK、权限、应用信息管理|
|WMS|Window 管理|
|InputManagerService|输入事件管理|
|PowerManagerService|电源管理|
|NotificationManagerService|通知管理|

---

# 四、AMS 到底是什么

AMS 全称：

```text
ActivityManagerService
```

一个非常重要的错误认知是：

```text
AMS = Linux 进程
```

这是错误的。

正确关系：

```text
system_server
    ↓
Linux 进程

ActivityManagerService
    ↓
运行在 system_server 中的 Java 系统服务
```

可以粗略理解为：

```java
class ActivityManagerService {
    // Android Framework 系统服务
}
```

所以：

```text
system_server = 进程

AMS = system_server 中的对象 / 系统服务
```

类似的：

```text
system_server
    │
    ├── AMS
    ├── ATMS
    ├── PMS
    ├── WMS
    └── ...
```

---

# 五、AMS 不只是管理 Activity

早期学习 Android Framework 时，经常会简单理解：

```text
AMS
    ↓
管理 Activity 生命周期
管理 Activity 栈
```

这个理解现在已经不够准确。

现代 Android 中，Activity / Task 相关职责大量由：

```text
ActivityTaskManagerService
```

即：

```text
ATMS
```

负责。

可以粗略理解为：

```text
AMS
├── Process
├── Service
├── Broadcast
├── ContentProvider
├── OOM Adj
└── 应用生命周期协调等

ATMS
├── Activity
├── Task
├── Activity 启动
└── Activity / Task 管理
```

因此以后分析 Activity 启动流程时：

```text
AMS
```

和：

```text
ATMS
```

都会频繁出现。

---

# 六、为什么 App 不能直接 new 一个 AMS 使用

假设普通 App 中真的能够执行：

```java
new ActivityManagerService(...)
```

即使暂时忽略隐藏 API、构造参数、权限等问题，这个 AMS 也不是系统真正使用的 AMS。

原因是：

> Linux 进程之间存在地址空间隔离。

例如：

```text
App Process

虚拟地址空间 A

new ActivityManagerService()
```

真正的 AMS 在：

```text
system_server

虚拟地址空间 B

真正的 ActivityManagerService
```

两个对象位于：

```text
不同 Linux 进程
不同虚拟地址空间
```

普通 App 无法直接访问 system_server 中真正的 AMS Java 对象。

因此：

```text
App
    X
AMS
```

不能直接进行普通 Java 对象调用。

必须通过：

```text
Binder IPC
```

完成跨进程通信。

---

# 七、Binder 在这里解决什么问题

假设：

```text
App Process
```

需要调用：

```text
system_server
    ↓
AMS
```

由于两个进程地址空间隔离，因此 Android 使用 Binder。

整体模型：

```text
App Process
    ↓
Binder Proxy
    ↓
Binder Driver
====================
进程边界
====================
    ↓
system_server
    ↓
Binder Stub
    ↓
真正的 AMS
```

因此 App 看起来可能只是：

```java
service.xxx();
```

实际上背后可能发生了：

```text
普通 Java 调用
    ↓
Binder Proxy
    ↓
transact()
    ↓
Binder Driver
    ↓
目标进程 Binder 线程
    ↓
onTransact()
    ↓
Stub
    ↓
真正系统服务
```

这就是 Android Framework 学习中非常重要的思想：

> 看到一个方法调用，首先判断调用双方是否处于同一个进程。

---

# 八、ServiceManager 是什么

SystemServer 和 ServiceManager 不是一个东西。

可以粗略理解为：

```text
SystemServer
    ↓
创建系统服务

AMS
PMS
WMS
...
    ↓
注册 Binder 服务
    ↓
ServiceManager
```

ServiceManager 可以理解成：

> Binder 服务注册中心 / 查询中心。

概念上：

```text
ServiceManager.addService(...)
```

负责注册服务。

而客户端可以：

```text
ServiceManager.getService(...)
```

查询服务。

因此几个角色可以这样区分：

```text
SystemServer
    ↓
负责启动 / 创建大量 Framework 系统服务

ServiceManager
    ↓
负责 Binder 服务注册与查询

Binder
    ↓
负责跨进程通信

AMS / PMS / WMS
    ↓
真正执行系统业务逻辑
```

---

# 九、Zygote 是什么

Zygote 是 Android 非常重要的进程。

它可以粗略理解为：

> Android Java 应用进程的孵化器。

Android 启动：

```text
init
    ↓
Zygote
```

之后很多 Java 进程都由 Zygote fork 出来。

例如：

```text
Zygote
    │
    ├── fork → system_server
    │
    ├── fork → App A
    │
    ├── fork → App B
    │
    └── fork → App C
```

因此：

```text
Zygote
```

和：

```text
AMS
```

职责完全不同。

---

# 十、Zygote 和 AMS 的职责区别

可以简单记：

```text
AMS
    ↓
决定 / 调度进程应该什么时候启动

Zygote
    ↓
真正执行 fork 创建 Linux 进程
```

例如：

```text
AMS
 ↓
发现目标 App 进程不存在
 ↓
请求启动进程
 ↓
Zygote
 ↓
fork()
 ↓
App Process
```

所以不能理解为：

```text
AMS 自己 fork App
```

而应该理解：

```text
AMS：管理和调度

Zygote：孵化进程
```

---

# 十一、ActivityThread 是什么

每个普通 Android App 进程中都有一个非常重要的类：

```text
ActivityThread
```

它可以粗略理解为：

> App 进程 Java Framework 层的核心管理者之一。

例如：

```text
App Process
    ↓
ActivityThread
    │
    ├── Application
    ├── Activity
    ├── Service
    └── Provider
```

注意：

```text
ActivityThread
```

虽然名字里面有：

```text
Thread
```

但它本身并不是 `Thread` 的子类。

不要简单理解成：

```text
ActivityThread = Android 主线程
```

更加准确的理解是：

> ActivityThread 是 App 进程 Framework 层的重要管理类，它负责与 system_server 协作，并在 App 进程中完成 Application、Activity、Service 等组件的调度。

---

# 十二、ActivityThread.main()

普通 App 进程启动后，会进入：

```java
ActivityThread.main()
```

可以先粗略理解：

```text
Zygote
    ↓
fork
    ↓
App Process
    ↓
进入 App Runtime
    ↓
ActivityThread.main()
```

因此不要理解成：

```text
AMS
    ↓
直接调用
    ↓
ActivityThread.main()
```

原因非常简单：

```text
AMS
    ↓
system_server

ActivityThread
    ↓
App Process
```

双方处于不同 Linux 进程。

普通 Java 方法不能直接跨进程调用。

---

# 十三、ActivityThread.main() 大致做什么

可以先建立如下模型：

```text
ActivityThread.main()
    ↓
准备主线程 Looper
    ↓
创建 ActivityThread
    ↓
ActivityThread.attach()
    ↓
Binder
    ↓
AMS
```

也就是说：

> App 进程创建完成以后，还需要向 system_server “报到”。

可以粗略理解：

```text
App Process
    ↓
ActivityThread
    ↓
attach()
    ↓
Binder IPC
====================
    ↓
system_server
    ↓
AMS
```

AMS 从而知道：

```text
目标 App 进程已经成功启动
```

---

# 十四、system_server 怎么反过来调用 App

这里开始出现 Android Framework 非常重要的“双向 Binder”结构。

App 调用系统服务：

```text
App
 ↓
Binder
 ↓
system_server
```

但是 system_server 有时候也需要通知 App：

```text
system_server
 ↓
Binder
 ↓
App
```

例如：

```text
现在创建 Activity

现在执行 Activity Resume

现在创建 Service

现在处理 Broadcast
```

因此 App 进程需要向 system_server 提供一个 Binder 接口。

这里有一个非常重要的角色：

```text
ApplicationThread
```

可以先记：

> ApplicationThread 是 system_server 调用 App 进程的重要 Binder 接口。

于是形成：

```text
App → system_server

ActivityManager / ActivityTaskManager
        ↓
Binder
        ↓
AMS / ATMS
```

以及：

```text
system_server → App

AMS / ATMS
        ↓
ApplicationThread
        ↓
Binder
        ↓
ActivityThread
```

这就是 Android Framework 非常重要的：

```text
双向 Binder 通信
```

---

# 十五、点击桌面图标启动 App 的完整流程

假设：

```text
目标 App 当前没有运行
```

用户点击 Launcher 中的 App 图标。

---

## 第一步：Launcher 请求启动 Activity

Launcher 本身也是一个普通 Android App。

```text
用户点击图标
    ↓
Launcher
    ↓
startActivity()
```

Launcher 无法自己创建目标 App 的 Activity。

它必须请求系统：

```text
Launcher
    ↓
Activity.startActivity()
    ↓
Framework API
    ↓
ActivityTaskManager
    ↓
Binder
    ↓
system_server
    ↓
ATMS
```

---

## 第二步：ATMS 分析 Activity 启动

ATMS 会处理 Activity / Task 相关逻辑。

例如：

```text
Intent 指向哪个 Activity？

目标 Activity 是否存在？

应该放到哪个 Task？

启动模式是什么？

目标进程是否存在？
```

如果目标 App 进程不存在：

```text
ATMS / AMS
    ↓
需要启动目标进程
```

---

# 十六、AMS 请求 Zygote 创建 App 进程

AMS 本身不会直接：

```text
fork()
```

而是请求 Zygote。

```text
system_server
    ↓
AMS
    ↓
请求启动目标 App
    ↓
Zygote
```

Zygote：

```text
Zygote
    ↓
fork()
    ↓
App Process
```

至此：

```text
新的 Linux App 进程
```

正式出现。

---

# 十七、新 App 进入 ActivityThread.main()

App Process 创建后：

```text
App Process
    ↓
ActivityThread.main()
```

然后：

```text
ActivityThread.main()
    ↓
准备 Main Looper
    ↓
创建 ActivityThread
    ↓
ActivityThread.attach()
```

接下来 App 向 AMS 注册：

```text
ActivityThread.attach()
    ↓
Binder
    ↓
AMS
```

告诉系统：

```text
我的 App 进程已经启动完成，可以进行后续组件调度。
```

---

# 十八、AMS / ATMS 调度 App 创建 Activity

system_server 接下来需要通知 App：

```text
创建 Application

创建 Activity

执行生命周期
```

但：

```text
system_server
```

不能直接调用 App 进程里的普通 Java 对象。

因此再次通过 Binder：

```text
system_server
    ↓
AMS / ATMS
    ↓
ApplicationThread
    ↓
Binder
====================
    ↓
App Process
    ↓
ActivityThread
```

ActivityThread 收到调度之后，在 App 主线程中继续处理。

---

# 十九、Application 与 Activity 创建

后续可以粗略理解为：

```text
ActivityThread
    ↓
创建 Application
    ↓
Application.attach()
    ↓
Application.onCreate()
    ↓
创建 Activity
    ↓
Activity.attach()
    ↓
Activity.onCreate()
    ↓
Activity.onStart()
    ↓
Activity.onResume()
```

现代 Android 实际流程中还会涉及：

```text
ClientTransaction
LaunchActivityItem
ResumeActivityItem
TransactionExecutor
```

这些会在 Activity 启动源码专题中继续深入。

---

# 二十、完整 App 冷启动流程

把目前所有知识串起来：

```text
用户点击桌面图标
        ↓
Launcher
        ↓
startActivity()
        ↓
ActivityTaskManager
        ↓
Binder IPC
        ↓
system_server
        ↓
ATMS / AMS
        ↓
发现目标进程不存在
        ↓
请求 Zygote
        ↓
Zygote fork()
        ↓
创建 App Process
        ↓
ActivityThread.main()
        ↓
创建 ActivityThread
        ↓
ActivityThread.attach()
        ↓
Binder IPC
        ↓
AMS
        ↓
App 进程注册完成
        ↓
AMS / ATMS 调度客户端
        ↓
ApplicationThread
        ↓
Binder IPC
        ↓
ActivityThread
        ↓
创建 Application
        ↓
Application.onCreate()
        ↓
创建 Activity
        ↓
Activity.onCreate()
        ↓
Activity.onStart()
        ↓
Activity.onResume()
```

这是目前阶段最重要的一张流程图。

---

# 二十一、Android Framework 的进程视角

以后分析 Framework 源码时，不要只看：

```text
A.method()
    ↓
B.method()
    ↓
C.method()
```

一定要先问：

```text
A 在哪个进程？

B 在哪个进程？

A 和 B 是否属于同一个进程？

如果不是，中间通过什么 IPC？
```

例如：

```text
App
 ↓
AMS
```

看到这里应该立刻意识到：

```text
App Process
 ↓
Binder Proxy
 ↓
Binder Driver
 ↓
system_server
 ↓
Binder Stub
 ↓
AMS
```

而看到：

```text
AMS
 ↓
ActivityThread
```

也应该立刻意识到：

```text
system_server
 ↓
Binder
 ↓
App Process
```

而不是简单理解成：

```java
ams.xxx();
activityThread.xxx();
```

这种思维方式是从普通 Android 应用开发进入 Framework 开发的重要转变。

---

# 二十二、几个核心角色最终总结

## Zygote

```text
类型：Linux 进程

职责：
孵化 Java App 进程和 system_server 等进程。
```

核心关键词：

```text
fork
```

---

## system_server

```text
类型：Linux 进程

职责：
承载 Android Framework 大量核心系统服务。
```

内部存在：

```text
AMS
ATMS
PMS
WMS
InputManagerService
PowerManagerService
...
```

---

## SystemServer

```text
类型：Java 类

职责：
system_server 进入 Java Framework 世界后的核心入口，
负责初始化和启动大量系统服务。
```

---

## AMS

```text
ActivityManagerService

类型：
Framework 系统服务

运行位置：
system_server

主要职责：
Process
Service
Broadcast
Provider
应用生命周期协调
OOM Adj
...
```

---

## ATMS

```text
ActivityTaskManagerService

类型：
Framework 系统服务

运行位置：
system_server

主要职责：
Activity
Task
Activity 启动
任务栈管理
...
```

---

## ServiceManager

```text
类型：
Binder 基础设施中的核心角色

职责：
Binder 服务注册与查询。
```

可以粗略理解为：

```text
Binder 服务注册中心
```

---

## ActivityThread

```text
类型：
Java Framework 类

运行位置：
App Process

职责：
App 进程 Framework 层核心管理者之一，
负责 Application、Activity、Service 等组件的客户端调度。
```

重要入口：

```java
ActivityThread.main()
```

---

## ApplicationThread

```text
运行位置：
App Process

职责：
作为 Binder 接口，让 system_server 能够向 App 进程发送组件调度命令。
```

可以先记：

```text
ApplicationThread
        ↓
system_server → App 的重要 Binder 通道
```

---

# 二十三、最容易混淆的概念

## 错误 1：AMS 是一个 Linux 进程

错误：

```text
AMS = Linux Process
```

正确：

```text
system_server = Linux Process

AMS = system_server 中运行的系统服务
```

---

## 错误 2：system_server 创建 SystemServer 进程

错误。

正确关系：

```text
Zygote
 ↓ fork
system_server
 ↓
执行 SystemServer.main()
```

---

## 错误 3：AMS 直接调用 ActivityThread.main()

错误：

```text
AMS
 ↓
ActivityThread.main()
```

正确：

```text
AMS
 ↓
请求 Zygote 创建进程
 ↓
Zygote fork
 ↓
App Process
 ↓
进入 ActivityThread.main()
```

---

## 错误 4：AMS 直接 new Activity

错误：

```text
AMS
 ↓
new Activity()
```

Activity 最终是在：

```text
App Process
```

中创建。

整体模型：

```text
system_server
 ↓
AMS / ATMS
 ↓
Binder
 ↓
ApplicationThread
 ↓
ActivityThread
 ↓
创建 Activity
```

---

## 错误 5：ActivityThread 就是主线程

不准确。

ActivityThread 是：

```text
App Framework 核心管理类
```

它运行和调度大量逻辑到 App 主线程，但它本身并不是：

```java
Thread
```

的子类。

---

# 二十四、最终核心架构图

```text
                         Android
                            │
                           init
                            │
                         Zygote
                        /      \
                       /        \
                  fork           fork
                    ↓             ↓
            system_server     App Process
                    │             │
            SystemServer.main()   │
                    │        ActivityThread.main()
                    │             │
        ┌───────────┼──────┐      │
        ↓           ↓      ↓      ↓
       AMS         ATMS    WMS  ActivityThread
        │           │             │
        │           │        ApplicationThread
        │           │             ↑
        └───────────┼─────────────┘
                    │
                 Binder
                    │
              跨进程通信
```

进一步理解：

```text
App → System

App
 ↓
Proxy
 ↓
Binder Driver
 ↓
Stub
 ↓
AMS / ATMS / PMS / WMS
```

反方向：

```text
System → App

AMS / ATMS
 ↓
ApplicationThread
 ↓
Binder
 ↓
ActivityThread
 ↓
Application / Activity / Service
```

---

# 二十五、本阶段必须形成的思维方式

以后看到 Android Framework 调用链：

```text
A
 ↓
B
 ↓
C
```

第一反应不要只是：

```text
A 调用了 B
B 调用了 C
```

而应该先问：

```text
A 在哪个进程？

B 在哪个进程？

C 在哪个进程？

是否跨进程？

如果跨进程：

Proxy 在哪里？

Stub 在哪里？

Binder Driver 在哪里？

真正执行代码的是哪个线程？

最终业务对象是谁？
```

这就是 Android Framework 的：

# 进程视角

---

# 二十六、当前知识链

目前已经逐渐建立：

```text
Linux 进程隔离
        ↓
UID / 权限
        ↓
Zygote
        ↓
system_server
        ↓
SystemServer
        ↓
AMS / ATMS / PMS / WMS
        ↓
ServiceManager
        ↓
Binder IPC
        ↓
App Process
        ↓
ActivityThread
        ↓
Application
        ↓
Activity
```

下一步最重要的就是把中间这个：

```text
Binder IPC
```

彻底拆开。

---

# 下一课：Binder 深入

下一阶段重点研究：

```text
App
 ↓
Proxy
 ↓
transact()
 ↓
Binder Driver
 ↓
system_server Binder Thread
 ↓
onTransact()
 ↓
Stub
 ↓
AMS
```

需要解决以下问题：

1. Binder 到底是什么？
    
2. 为什么 Android 不直接使用 Socket？
    
3. IBinder 是什么？
    
4. Binder 是什么？
    
5. BinderProxy 是什么？
    
6. Stub 是什么？
    
7. Proxy 是什么？
    
8. AIDL 到底生成了什么代码？
    
9. transact() 做了什么？
    
10. onTransact() 做了什么？
    
11. Binder Driver 在内核中承担什么职责？
    
12. ServiceManager 为什么能找到 AMS？
    
13. Binder 如何知道目标进程是谁？
    
14. Binder 线程池是什么？
    
15. Binder 调用发生在哪个线程？
    
16. system_server 如何反向调用 App？
    
17. ApplicationThread 在 Binder 体系中到底是什么？
    
18. 一次完整 Binder IPC 在 Java → Native → Kernel → Native → Java 之间如何流转？
    

学完 Binder 后，再继续：

```text
Binder
 ↓
Activity 启动源码
 ↓
AMS / ATMS
 ↓
Zygote
 ↓
ActivityThread
 ↓
ClientTransaction
 ↓
Application / Activity
 ↓
WMS
 ↓
ViewRootImpl
 ↓
Choreographer / VSync
 ↓
SurfaceFlinger
```

最终把 Android App 从：

```text
点击桌面图标
```

一直追踪到：

```text
第一帧真正显示到屏幕
```

形成完整的 Android Framework 知识链。