---
title: Android 通过 ADB 使用 Mac 电脑网络：Gnirehtet 实战
tags:
  - Android
  - ADB
  - Gnirehtet
  - 网络调试
  - macOS
created: 2026-10-28
status: 已实测启动，网络连通性需单独验证
---

# Android 通过 ADB 使用 Mac 电脑网络：Gnirehtet 实战

> [!abstract] 一句话总结
> Android 手机通过 USB 连接 Mac，使用 **Gnirehtet + ADB + Android VPN（VpnService）**，让手机流量经 USB 转发到 Mac 的网络连接。这叫**反向网络共享（reverse tethering）**，不需要 Root。

## 1. 背景与适用场景

- Mac 能上网，但 Android 手机希望通过 USB 使用 Mac 的网络。
- 开发、测试时需要保持 USB ADB 连接。
- 适合 Android / OPPO 等可使用 ADB 的设备；**不是**鸿蒙 HDC/HAP 的替代工具。
- Gnirehtet 是开源工具：<https://github.com/Genymobile/gnirehtet>。

**注意：** 这是让**手机使用电脑网络**，与手机开热点给电脑上网的方向相反。

## 2. 工作原理

```text
Android App 网络请求
       ↓
Android VpnService 虚拟网络接口
       ↓
Gnirehtet Android 客户端
       ↓
USB / ADB reverse 通道
       ↓
Mac 上的 Gnirehtet relay
       ↓
Mac 的网络连接 → 互联网
```

- 不需要 Root，也不要求手机出现真实 USB 网卡。
- Android 会提示授权 VPN；此处 VPN 主要用于接管、转发手机流量，**不等于**连接远程商业 VPN 服务。
- 部分协议、局域网发现、ICMP ping 等行为可能不被支持；不能只用 `ping` 判断是否成功。

## 3. 安装准备（macOS）

### 3.1 安装 ADB

```bash
brew install android-platform-tools
adb version
```

手机打开开发者选项、USB 调试，用**支持数据传输**的 USB 线连接 Mac。手机弹出“允许 USB 调试”时选择允许。

```bash
adb devices
```

预期输出：

```text
List of devices attached
XXXXXXXXXXXX    device
```

如果显示 `unauthorized`，检查手机端授权弹窗；如果没有设备，检查数据线、USB 模式、驱动/授权和 ADB 服务。

### 3.2 下载 Gnirehtet

到官方 Releases 下载适用于 macOS 的 **Rust 版**发布包：

<https://github.com/Genymobile/gnirehtet/releases>

解压后进入目录（路径按实际情况修改）：

```bash
cd ~/Downloads/gnirehtet-rust-macos64
chmod +x gnirehtet
```

如果 macOS 阻止运行，先核对下载来源、架构与系统安全提示，再按系统允许的方式授权，不建议直接关闭 Gatekeeper。

## 4. 一键启动与停止

在 Gnirehtet 目录执行：

```bash
./gnirehtet run
```

然后在手机上确认 Gnirehtet 的 **VPN 连接授权**。保持终端中的 relay 进程运行，并保持 USB 连接。

停止：

```bash
./gnirehtet stop
```

如果要重新连接：

```bash
./gnirehtet stop
./gnirehtet run
```

> [!tip] 多设备场景
> 多台 Android 设备同时连接时，先通过 `adb devices` 确认设备；避免把命令作用到错误手机。具体设备选择参数以所下载版本的 `./gnirehtet --help` 为准。

## 5. 如何确认真正成功

1. Mac 自身能正常访问互联网。
2. `adb devices` 显示目标手机状态为 `device`。
3. `./gnirehtet run` 启动 relay，手机确认 VPN 授权。
4. 在手机系统 VPN 设置中确认 Gnirehtet 处于连接状态。
5. **关闭手机 Wi-Fi 和移动数据**（保持 USB 连接），用手机浏览器打开 HTTPS 网站，确认页面能加载。
6. 必要时检查：

```bash
adb shell dumpsys connectivity | grep -i -A 15 vpn
adb shell pidof com.genymobile.gnirehtet
```

上述检查命令依 Android 版本及系统权限而异；没有输出不一定等于失败。以手机实际 HTTPS 访问结果为准。

## 6. 我的真实操作记录（2026-10-23）

设备背景：Mac + OPPO Android 手机，通过 USB / ADB 使用 Gnirehtet。

执行命令：

```bash
./gnirehtet run
```

真实终端输出：

```text
2026-10-23 14:42:21.252 INFO Main: Starting relay server...
2026-10-23 14:42:21.252 INFO Main: Checking gnirehtet client...
2026-10-23 14:42:21.256 INFO Relay: Relay server started
2026-10-23 14:42:21.401 INFO Main: Starting client...
Broadcasting: Intent { act=com.genymobile.gnirehtet.START flg=0x400000 cmp=com.genymobile.gnirehtet/.GnirehtetControlReceiver }
Broadcast completed: result=0
```

**日志解读：**

| 日志 | 说明 |
|---|---|
| `Starting relay server` | Mac 端准备启动网络转发服务 |
| `Relay server started` | Mac 端 relay 已启动 |
| `Checking gnirehtet client` | 检查手机端客户端 |
| `Starting client` | 请求启动 Android 客户端 |
| `Broadcast completed: result=0` | Android 广播命令已返回；**不是网络连通性成功证明** |

**已确认的事实：** 以上命令和日志确实来自本人当时操作；之后还询问过“什么是 Gnirehtet VPN，如何开启”。**原始聊天未保留完整的手机 HTTPS 访问验证输出**，因此不能把该日志直接记作端到端联网成功截图。

## 7. 踩坑与排障笔记

### 坑 1：误以为需要开启 USB 网络共享

**原因：** 手机的“USB 网络共享”通常是把**手机网络分享给电脑**，方向相反。

**正确做法：** 使用 Gnirehtet 的 reverse tethering，无需打开手机“USB 网络共享”开关。

### 坑 2：看到 `result=0` 就认为联网成功

**现象：** 命令返回 `Broadcast completed: result=0`。

**原因：** 这只能说明广播执行返回，并不保证 VPN 授权、数据转发或 DNS 都正常。

**解决：** 检查手机 VPN 状态，关闭 Wi-Fi/移动数据，再访问 HTTPS 网站验证。

### 坑 3：不知道 Gnirehtet 为什么要使用 VPN

**原因：** Android 无 Root 条件下，Gnirehtet 利用系统 `VpnService` 捕获手机应用流量，再通过 ADB 转发给 Mac。

**注意：** 这是本地流量转发机制；手机可能出现 VPN 图标，不意味着流量直接进入某个外部 VPN 服务器。

### 坑 4：ADB 未授权或设备不在线（常见排障，不代表本人遇到）

```bash
adb devices
adb kill-server
adb start-server
adb devices
```

重新插拔数据线，在手机上确认 USB 调试授权。若多设备同时连接，先指定目标设备。

### 坑 5：手机已有其他 VPN（常见排障）

Android 通常不能同时让两个普通 VPN 服务接管流量。先断开手机上的其他 VPN，再启动 Gnirehtet。

### 坑 6：Mac 开了 Clash / Whistle，手机却没有自动走代理（常见排障）

Gnirehtet 使用 Mac 的网络出口，**不保证自动使用 Mac 上的 HTTP 代理、Whistle 规则或 Clash 系统代理**。实际取决于 Mac 路由、TUN 模式、代理监听与规则配置。要抓包或改写请求，需要单独验证网络路径。

### 坑 7：`ping` 不通就误判失败（常见排障）

Gnirehtet 并非完整的物理网络桥接。ICMP、局域网广播/发现等可能不符合普通网络预期。优先用浏览器 HTTPS 和实际 App 请求验证。

### 坑 8：电脑休眠、USB 断开后网络失效（常见排障）

检查 USB/ADB 状态，唤醒电脑，重新运行 Gnirehtet；使用期间保持 Mac relay 进程存活。

## 8. 发给同事的最简操作卡片

**前提：** Mac 可上网；Android 手机开启 USB 调试并允许 Mac 的 ADB 授权。

```bash
# 1. 安装 ADB
brew install android-platform-tools

# 2. 确认设备在线
adb devices

# 3. 从官方 Releases 下载并解压 gnirehtet-rust-macos64
# https://github.com/Genymobile/gnirehtet/releases
cd ~/Downloads/gnirehtet-rust-macos64
chmod +x gnirehtet

# 4. 启动 USB 反向网络共享
./gnirehtet run

# 5. 手机同意 Gnirehtet VPN 授权
# 6. 手机关闭 Wi-Fi、移动数据，浏览器验证 HTTPS

# 7. 不使用时停止
./gnirehtet stop
```

## 9. 和其他技术的区别

| 方案 | 流量方向 | 是否 Root | 典型用途 |
|---|---|---|---|
| 手机 USB 网络共享 | 手机 → 电脑 | 否 | 电脑借用手机流量 |
| Gnirehtet | 电脑 → Android 手机 | 否 | Android 开发测试、反向网络共享 |
| Android HTTP 代理 | 取决于代理配置 | 否 | HTTP(S) 调试/抓包；不保证所有 App 遵循代理 |
| ADB reverse | 端口级反向映射 | 否 | 手机访问电脑上的特定开发服务，并非全局联网 |

## 10. 参考链接

- 官方仓库：<https://github.com/Genymobile/gnirehtet>
- 官方发布包：<https://github.com/Genymobile/gnirehtet/releases>
- ADB 官方文档：<https://developer.android.com/tools/adb>

---

> [!success] 经验总结
> **先保证 ADB 在线 → 启动 Mac relay → 手机允许 VPN → 关闭其他手机网络验证 HTTPS。** 终端显示 `result=0` 只是过程信号，实际联网验证才是最终验收。
