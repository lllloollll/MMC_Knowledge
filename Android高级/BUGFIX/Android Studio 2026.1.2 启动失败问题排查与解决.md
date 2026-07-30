# Android Studio 2026.1.2 启动失败问题排查与解决

## 问题描述

Android Studio 2026.1.2 启动时报错：

```
Error occurred during initialization of VM
java.lang.Error: A command line option has attempted to allow or enable the Security Manager.
Enabling a Security Manager is not supported.
at java.lang.System.initPhase3(java.base@25.0.1/System.java:1969)
```

## 根因分析

### 1. JDK 版本冲突

- 系统通过 Homebrew 安装了 JDK 25（`/opt/homebrew/opt/openjdk` → `/opt/homebrew/Cellar/openjdk/25.0.1`）
- 用户将 `JAVA_HOME` 设置为 JDK 25 后，IDE 自动将 JDK 25 路径缓存到了 `studio.jdk` 文件

### 2. studio.jdk 文件覆盖捆绑 JBR

- 文件路径：`~/Library/Application Support/Google/AndroidStudio2026.1.2/studio.jdk`
- 文件内容：`/opt/homebrew/opt/java/libexec/openjdk.jdk`（指向 Homebrew JDK 25）
- JetBrains native launcher 的 JDK 搜索顺序：
  1. product-specific 环境变量（如 `STUDIO_JDK`）
  2. **用户配置文件（`studio.jdk`）** ← 命中了 JDK 25
  3. 捆绑的 JBR（JDK 21）
  4. `JDK_HOME`
  5. `JAVA_HOME`
- 由于步骤 2 优先于步骤 3，IDE 始终使用 JDK 25 而非捆绑的 JBR 21

### 3. Security Manager 不兼容

- IDE 安装目录默认 `bin/studio.vmoptions` 包含 `-Djava.security.manager=allow`
- JDK 24 起（JEP 486）Security Manager 被永久移除
- JDK 25 遇到 `-Djava.security.manager=allow` 直接崩溃

## 解决方案

### 核心修复

删除 `studio.jdk` 文件，让 launcher 回退到捆绑的 JBR 21：

```bash
rm ~/Library/Application\ Support/Google/AndroidStudio2026.1.2/studio.jdk
```

### 辅助修改

将 `~/.zshrc` 中的 `JAVA_HOME` 改回 JDK 21：

```bash
# 修改前（导致问题的配置）
export JAVA_HOME="/opt/homebrew/opt/openjdk/libexec/openjdk.jdk/Contents/Home"

# 修改后
export JAVA_HOME="/Users/casdon/Library/Java/JetBrainsRuntime/jbr_jcef-21.0.8-osx-aarch64-b895.146/Contents/Home"
```

## 验证结果

启动日志确认使用捆绑 JBR 21：

```
JRE: 21.0.10+-117844308-b1163.108, aarch64 (JetBrains s.r.o.)
boot library path: /Applications/Android Studio.app/Contents/jbr/Contents/Home/lib
```

## 经验总结

| 项目     | 说明                                                       |
| ------ | -------------------------------------------------------- |
| 问题触发条件 | 将 `JAVA_HOME` 设为 JDK 25 后启动 Android Studio               |
| 根本原因   | `studio.jdk` 缓存文件覆盖了捆绑 JBR                               |
| 崩溃原因   | JDK 25 不支持 Security Manager，与 IDE 默认配置冲突                 |
| 修复方式   | 删除 `studio.jdk` 缓存文件                                     |
| 注意事项   | 切换 `JAVA_HOME` 后，IDE 可能自动缓存新的 JDK 路径到 `studio.jdk`，需注意清理 |
