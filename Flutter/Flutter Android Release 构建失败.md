# Flutter Android Release 构建失败：GeneratedPluginRegistrant 找不到 Plugin Class 排查笔记

## 一、问题背景

当前 Flutter Android 项目：

```text
项目：casdon_smart
包名：com.casdon.smart
Flutter：3.41.3
AGP：8.9.1
Gradle：8.14
JDK：21.0.10
```

Android App 使用 Kotlin DSL：

```text
android/app/build.gradle.kts
android/build.gradle.kts
```

Release 配置：

```kotlin
buildTypes {
    getByName("release") {
        isMinifyEnabled = true
        isShrinkResources = true
        signingConfig = signingConfigs.getByName("release")
    }
}
```

项目存在多个 Push Vendor Flavor：

```kotlin
flavorDimensions += "pushVendor"

productFlavors {
    allVendors
    universal
    huawei
    xiaomi
    oppo
    vivo
    honor
    meizu
}
```

因此：

```text
allVendorsDebug
allVendorsRelease
```

对应 Gradle Task：

```bash
./gradlew assembleAllVendorsDebug
./gradlew assembleAllVendorsRelease
```

---

# 二、实际遇到的问题

Debug 构建成功：

```bash
./gradlew assembleAllVendorsDebug
```

结果：

```text
BUILD SUCCESSFUL
```

Release 构建失败：

```bash
./gradlew assembleAllVendorsRelease
```

错误：

```text
Task :app:compileAllVendorsReleaseJavaWithJavac FAILED

GeneratedPluginRegistrant.java:104:
错误: 找不到符号

flutterEngine.getPlugins().add(
    new dev.steenbakker.mobile_scanner.MobileScannerPlugin()
);

符号:
类 MobileScannerPlugin

位置:
程序包 dev.steenbakker.mobile_scanner
```

---

# 三、第一判断：这不是 R8 / ProGuard 问题

虽然 Release 配置：

```kotlin
isMinifyEnabled = true
isShrinkResources = true
```

但是当前失败任务是：

```text
compileAllVendorsReleaseJavaWithJavac
```

也就是：

```text
Java/Kotlin 编译阶段
```

而 R8 位于后面的处理阶段。

大致流程：

```text
Kotlin 编译
    ↓
Java 编译
    ↓
资源处理
    ↓
DEX
    ↓
R8 / Minify
    ↓
Resource Shrinking
    ↓
APK
```

因此：

```text
compile...JavaWithJavac
```

阶段失败时，不应该首先排查：

- ProGuard
    
- R8
    
- isMinifyEnabled
    
- isShrinkResources
    
- 签名
    
- APK 混淆
    

### 通用原则

看到：

```text
compileXXXJavaWithJavac
```

优先排查：

```text
源码
依赖
Variant
Kotlin 编译产物
Java Compile Classpath
Gradle Task 顺序
```

而不是 R8。

---

# 四、确认 GeneratedPluginRegistrant

检查：

```bash
grep -n -A5 -B5 "mobile_scanner" \
app/src/main/java/io/flutter/plugins/GeneratedPluginRegistrant.java
```

得到：

```java
flutterEngine.getPlugins().add(
    new dev.steenbakker.mobile_scanner.MobileScannerPlugin()
);
```

说明 Flutter 的插件自动注册代码确实认为：

```text
mobile_scanner
```

应该存在。

因此不能简单删除这段代码。

否则即使 Java 编译通过，运行时也可能出现：

```text
MissingPluginException
```

---

# 五、检查 mobile_scanner 是否进入 Debug / Release Classpath

Debug：

```bash
./gradlew :app:dependencies \
  --configuration allVendorsDebugCompileClasspath \
  | grep -i -A3 -B3 "mobile_scanner"
```

得到：

```text
+--- project :mobile_scanner
|    +--- org.jetbrains.kotlin:kotlin-stdlib:2.2.20 (*)
|    \--- io.flutter:flutter_embedding_debug:...
```

Release：

```bash
./gradlew :app:dependencies \
  --configuration allVendorsReleaseCompileClasspath \
  | grep -i -A3 -B3 "mobile_scanner"
```

得到：

```text
+--- project :mobile_scanner
|    +--- org.jetbrains.kotlin:kotlin-stdlib:2.2.20 (*)
|    \--- io.flutter:flutter_embedding_release:...
```

因此可以排除：

> Release 完全没有引入 mobile_scanner。

Debug 和 Release 都存在：

```text
project :mobile_scanner
```

---

# 六、找到 mobile_scanner 的真实位置

执行：

```bash
./gradlew projects | grep -A5 -B2 mobile_scanner
```

发现：

```text
+--- Project ':mobile_scanner'
```

然后：

```bash
./gradlew :mobile_scanner:properties \
  | grep -E "^(path|projectDir|buildDir):"
```

得到：

```text
buildDir:
/Users/casdon/projects/casdon_smart/build/mobile_scanner

path:
:mobile_scanner

projectDir:
/Users/casdon/.pub-cache/hosted/pub.dev/mobile_scanner-7.4.2/android
```

这个命令非常有价值。

### 通用技巧

当 Gradle 显示：

```text
project :xxx
```

不要猜它在哪里。

直接：

```bash
./gradlew :xxx:properties \
  | grep -E "^(path|projectDir|buildDir):"
```

就能找到真实目录。

---

# 七、发现实际使用的 mobile_scanner 版本

虽然 `pubspec.yaml`：

```yaml
mobile_scanner: ^7.0.1
```

但：

```bash
grep -A10 -B2 "^  mobile_scanner:" ../pubspec.lock
```

显示：

```yaml
mobile_scanner:
  dependency: "direct main"
  description:
    name: mobile_scanner
    sha256: ...
    url: "https://pub.dev"
  source: hosted
  version: "7.4.2"
```

原因是：

```text
^7.0.1
```

允许：

```text
>=7.0.1 <8.0.0
```

因此 Pub 解析到了：

```text
7.4.2
```

### 通用原则

不要只看：

```yaml
pubspec.yaml
```

真正参与当前构建的插件版本应该同时检查：

```text
pubspec.yaml
pubspec.lock
```

尤其是排查第三方 Flutter Plugin 构建问题时。

---

# 八、确认 Plugin 源码真实存在

检查：

```bash
grep -R "class MobileScannerPlugin" \
/Users/casdon/.pub-cache/hosted/pub.dev/mobile_scanner-7.4.2/android
```

得到：

```text
.../MobileScannerPlugin.kt:
class MobileScannerPlugin : FlutterPlugin, ActivityAware {
```

同时：

```bash
grep -R "MobileScannerPlugin" \
/Users/casdon/.pub-cache/hosted/pub.dev/mobile_scanner-7.4.2/android
```

确认源码存在。

源码目录：

```text
android/src/main/kotlin/dev/steenbakker/mobile_scanner/
```

其中包含：

```text
MobileScannerPlugin.kt
BarcodeHandler.kt
MobileScanner.kt
MobileScannerCallbacks.kt
...
```

因此可以排除：

```text
插件源码缺失
```

也可以确认 package：

```text
dev.steenbakker.mobile_scanner
```

与 GeneratedPluginRegistrant 一致。

---

# 九、检查 mobile_scanner 的 Gradle 配置

`mobile_scanner-7.4.2` 使用：

```text
android/build.gradle.kts
```

不是：

```text
android/build.gradle
```

其核心配置：

```kotlin
plugins {
    id("com.android.library")
}
```

并根据 AGP 版本决定是否应用 Kotlin：

```kotlin
val agpMajor =
    Version.ANDROID_GRADLE_PLUGIN_VERSION
        .substringBefore('.')
        .toInt()

val builtInKotlin =
    agpMajor >= 9 &&
    (findProperty("android.builtInKotlin")
        ?.toString() ?: "true")
        .toBoolean()

if (!builtInKotlin) {
    apply(plugin = "kotlin-android")
}
```

当前项目：

```text
AGP 8.9.1
```

因此：

```text
agpMajor = 8
```

会进入：

```kotlin
apply(plugin = "kotlin-android")
```

同时：

```kotlin
jvmTarget = JvmTarget.JVM_17
```

这和当前项目 Java/Kotlin 配置：

```text
Java 17 target
Kotlin JVM 17
```

总体上是匹配的。

---

# 十、最关键的发现：Debug 有 class，Release 没有 class

执行：

```bash
./gradlew :mobile_scanner:compileDebugKotlin
```

结果：

```text
BUILD SUCCESSFUL
10 actionable tasks: 10 up-to-date
```

执行：

```bash
./gradlew :mobile_scanner:compileReleaseKotlin
```

结果：

```text
BUILD SUCCESSFUL
10 actionable tasks: 10 up-to-date
```

但是：

```bash
find /Users/casdon/projects/casdon_smart/build/mobile_scanner \
  -type f -name "MobileScannerPlugin.class"
```

只发现：

```text
build/mobile_scanner/intermediates/runtime_library_classes_dir/debug/bundleLibRuntimeToDirDebug/dev/steenbakker/mobile_scanner/MobileScannerPlugin.class

build/mobile_scanner/tmp/kotlin-classes/debug/dev/steenbakker/mobile_scanner/MobileScannerPlugin.class
```

没有：

```text
tmp/kotlin-classes/release/...
```

这说明：

```text
Debug:
Kotlin 编译产物存在

Release:
MobileScannerPlugin.class 不存在
```

这是当前问题最重要的证据。

---

# 十一、为什么“compileReleaseKotlin 成功”反而值得怀疑？

这里出现了一个非常异常的组合：

```text
compileReleaseKotlin
        ↓
BUILD SUCCESSFUL
        ↓
10 actionable tasks: 10 up-to-date
```

但是：

```text
release/MobileScannerPlugin.class
```

不存在。

也就是说 Gradle 认为：

```text
Release Kotlin 编译结果是最新的
```

但实际产物不存在。

这提示：

```text
Gradle/Kotlin 增量编译缓存
```

或者：

```text
任务的输出状态
```

可能存在异常。

---

# 十二、当前最合理的下一步

不要马上修改：

```text
GeneratedPluginRegistrant.java
```

也不要修改：

```text
ProGuard
R8
isMinifyEnabled
isShrinkResources
```

先强制重新执行 Release Kotlin 编译：

```bash
./gradlew :mobile_scanner:compileReleaseKotlin --rerun-tasks
```

然后：

```bash
find /Users/casdon/projects/casdon_smart/build/mobile_scanner \
  -type f -name "MobileScannerPlugin.class"
```

如果出现：

```text
tmp/kotlin-classes/release/dev/steenbakker/mobile_scanner/MobileScannerPlugin.class
```

则说明之前很可能是：

```text
Gradle/Kotlin 增量编译状态异常
```

然后重新：

```bash
./gradlew assembleAllVendorsRelease
```

---

# 十三、如果 --rerun-tasks 后仍然没有 Release class

进一步：

```bash
./gradlew :mobile_scanner:clean
```

然后：

```bash
./gradlew :mobile_scanner:compileReleaseKotlin
```

再检查：

```bash
find /Users/casdon/projects/casdon_smart/build/mobile_scanner \
  -type f -name "MobileScannerPlugin.class"
```

如果仍然没有 Release class，再继续检查：

```text
mobile_scanner Release Variant
        ↓
compileReleaseKotlin
        ↓
Kotlin source set
        ↓
Kotlin output directory
        ↓
app Release JavaCompile classpath
```

这时才需要深入 Gradle Task wiring / Variant / Classpath。

---

# 十四、以后遇到类似问题的通用排查方法

以后如果看到：

```text
GeneratedPluginRegistrant.java
找不到符号
XXXPlugin
```

并且：

```text
Debug 正常
Release 失败
```

推荐按照下面的顺序排查。

## 第 1 步：确定失败阶段

先看：

```text
Task :app:xxxx
```

例如：

```text
compileReleaseJavaWithJavac
```

说明是编译阶段。

如果是：

```text
minifyReleaseWithR8
```

才开始考虑：

```text
R8 / ProGuard / keep
```

如果是：

```text
shrinkReleaseRes
```

才考虑资源压缩。

---

## 第 2 步：确认 GeneratedPluginRegistrant 注册了什么

```bash
grep -n -A5 -B5 "插件名" \
app/src/main/java/io/flutter/plugins/GeneratedPluginRegistrant.java
```

例如：

```java
new dev.steenbakker.mobile_scanner.MobileScannerPlugin()
```

---

## 第 3 步：确认 Plugin 是否进入 Release Classpath

```bash
./gradlew :app:dependencies \
  --configuration xxxReleaseCompileClasspath \
  | grep -i -A3 -B3 "插件名"
```

如果没有：

```text
project :xxx
```

优先解决依赖/Variant 问题。

如果有：

```text
project :xxx
```

继续往下。

---

## 第 4 步：找到插件真实目录

```bash
./gradlew :插件名:properties \
  | grep -E "^(path|projectDir|buildDir):"
```

不要猜 Pub Cache 路径。

---

## 第 5 步：确认源码是否存在

```bash
grep -R "class XXXPlugin" \
实际插件目录
```

同时确认 package：

```text
package dev.xxx
```

是否和：

```text
GeneratedPluginRegistrant
```

一致。

---

## 第 6 步：分别编译 Debug / Release Plugin

```bash
./gradlew :插件名:compileDebugKotlin
./gradlew :插件名:compileReleaseKotlin
```

如果插件是 Java，则查看：

```bash
./gradlew :插件名:compileDebugJavaWithJavac
./gradlew :插件名:compileReleaseJavaWithJavac
```

---

## 第 7 步：直接检查 Class 文件

这是非常重要的一步。

不要只相信：

```text
BUILD SUCCESSFUL
```

直接找：

```bash
find build/插件名 \
  -type f -name "XXXPlugin.class"
```

判断：

```text
Debug class 有
Release class 无
```

那么问题已经从：

```text
Flutter 插件问题
```

缩小成：

```text
Release 编译产物问题
```

---

## 第 8 步：强制重新编译

优先：

```bash
./gradlew :插件名:compileReleaseKotlin --rerun-tasks
```

然后再次检查 `.class`。

---

## 第 9 步：必要时 clean 插件模块

```bash
./gradlew :插件名:clean
./gradlew :插件名:compileReleaseKotlin
```

再检查 class。

---

## 第 10 步：最后才深入 Variant / Task / Classpath

如果仍然失败，再检查：

```bash
./gradlew :app:compileXXXReleaseJavaWithJavac --dry-run
```

观察 Release JavaCompile 前是否执行了：

```text
:plugin:compileReleaseKotlin
```

进一步可以使用：

```bash
./gradlew :app:compileXXXReleaseJavaWithJavac --info
```

查看 JavaCompile 实际使用的 classpath。

---

# 十五、一个非常实用的判断表

|现象|优先怀疑|
|---|---|
|`GeneratedPluginRegistrant` 找不到 Plugin Class|插件编译/Classpath|
|Debug 正常，Release 失败|Release Variant|
|Debug/Release Classpath 都有插件|不是简单依赖缺失|
|Plugin 源码不存在|Pub Cache / 依赖解析|
|Plugin 源码存在|继续检查编译|
|Debug `.class` 有，Release `.class` 无|Release 编译产物/缓存|
|`compileReleaseKotlin` FAILED|Kotlin/源码/依赖|
|`compileReleaseKotlin` SUCCESS，但 `.class` 不存在|增量缓存/Task Output/Variant wiring|
|JavaCompile FAILED|不要先查 R8|
|`minifyReleaseWithR8` FAILED|R8/ProGuard|
|`shrinkReleaseRes` FAILED|Resource Shrinking|
|APK 已生成但运行 `MissingPluginException`|Plugin 注册/平台实现|

---

# 十六、本次问题的当前状态

目前已经确定：

```text
mobile_scanner 版本：7.4.2
        ↓
源码存在
        ↓
MobileScannerPlugin.kt 存在
        ↓
package 正确
        ↓
Debug Class 存在
        ↓
Release Class 不存在
        ↓
compileReleaseKotlin 却显示 UP-TO-DATE
        ↓
优先怀疑 Gradle/Kotlin Release 增量编译产物状态
```

当前下一条命令：

```bash
./gradlew :mobile_scanner:compileReleaseKotlin --rerun-tasks
```

然后：

```bash
find /Users/casdon/projects/casdon_smart/build/mobile_scanner \
  -type f -name "MobileScannerPlugin.class"
```

这是目前最关键的验证。

---

# 十七、核心经验

以后排查 Flutter Android 构建问题，可以始终遵循一个原则：

> **不要根据“看起来像什么问题”修改配置，而是沿着失败 Task → Dependency → Source → Compile Task → Class Output → Classpath 一层层缩小范围。**

对于本次问题：

```text
Release APK 构建失败
        ↓
compileJava 失败
        ↓
GeneratedPluginRegistrant 找不到 Plugin
        ↓
Release CompileClasspath 有 Plugin
        ↓
找到 Plugin 真实目录
        ↓
源码存在
        ↓
Debug class 存在
        ↓
Release class 不存在
        ↓
锁定 Release 编译产物
```

这种排查方式比直接：

```text
flutter clean
rm -rf .gradle
删 Pub Cache
改 ProGuard
删除 GeneratedPluginRegistrant
```

更加可靠，因为每一步都有明确证据。