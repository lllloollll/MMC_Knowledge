
> 场景：更换开发电脑后，新电脑生成的 Debug APK 无法覆盖安装旧电脑生成的 Debug APK，提示 `signatures do not match previously installed version`。由于旧 App 中存在重要业务数据，不能直接卸载，因此需要理解 Debug 签名机制，并通过 `run-as` 完成 App 私有数据的备份与恢复。

---

## 一、问题背景

旧电脑安装的 Debug APK：

```text
packageName = com.xxx.app
签名 = Debug Key A
```

换电脑以后重新生成 Debug APK：

```text
packageName = com.xxx.app
签名 = Debug Key B
```

执行覆盖安装：

```bash
adb install -r app-debug.apk
```

报错：

```text
INSTALL_FAILED_UPDATE_INCOMPATIBLE

Package com.xxx.app signatures do not match
previously installed version
```

本质原因：

```text
旧电脑
    ↓
debug.keystore A
    ↓
Debug APK A
    ↓
安装到手机
    ↓
产生 App 数据


新电脑
    ↓
debug.keystore B
    ↓
Debug APK B
    ↓
尝试覆盖 APK A
    ↓
签名不一致
    ↓
Android 拒绝安装
```

---

# 二、Debug APK 也有签名

一个容易产生的误区：

> Debug 包没有签名，只有 Release 包才需要签名。

这是错误的。

Android APK 必须经过签名才能正常安装。

区别只是：

```text
Debug APK
    ↓
Android Gradle Plugin 自动完成签名
    ↓
开发者平时基本感知不到


Release APK
    ↓
通常由开发者显式配置正式签名
```

所以：

```text
Debug / Release
       ≠
有签名 / 没签名
```

而是：

```text
Debug APK
    └── 通常自动使用 debug.keystore

Release APK
    └── 通常使用 release.jks / release.keystore
```

---

# 三、默认 Debug Keystore

## 3.1 Windows

默认位置：

```text
C:\Users\<用户名>\.android\debug.keystore
```

例如：

```text
C:\Users\zhangsan\.android\debug.keystore
```

---

## 3.2 macOS / Linux

默认位置：

```text
~/.android/debug.keystore
```

例如：

```text
/Users/zhangsan/.android/debug.keystore
```

---

## 3.3 默认参数

Android 默认生成的 Debug Keystore 通常使用：

```text
storePassword = android
keyAlias      = androiddebugkey
keyPassword   = android
```

即：

```gradle
storePassword "android"
keyAlias "androiddebugkey"
keyPassword "android"
```

注意：

如果项目自己配置了 Debug 签名，则不一定使用这些默认值。

---

# 四、为什么换电脑以后 Debug 签名会变化

例如旧 Windows：

```text
C:\Users\xxx\.android\debug.keystore
                ↓
              Key A
                ↓
          app-debug.apk
```

新 Mac：

```text
~/.android/debug.keystore
          ↓
        Key B
          ↓
    app-debug.apk
```

即使：

```text
packageName 一样
代码一样
versionCode 一样
buildType 都是 debug
```

Android 看到的仍然是：

```text
旧 APK → 签名 A
新 APK → 签名 B
```

因此：

```text
相同 packageName
        +
不同签名
        ↓
不能覆盖安装
```

---

# 五、如何查看 Debug 包到底用了哪个签名

## 5.1 Gradle signingReport

执行：

```bash
./gradlew signingReport
```

Windows：

```cmd
gradlew signingReport
```

可能得到：

```text
Variant: debug
Config: debug
Store: C:\Users\xxx\.android\debug.keystore
Alias: AndroidDebugKey
SHA1: ...
SHA-256: ...
```

重点关注：

```text
Store
Alias
SHA-256
```

其中：

```text
Store
```

就是当前 Debug APK 使用的 keystore。

---

# 六、查看 APK 签名

执行：

```bash
apksigner verify --print-certs app-debug.apk
```

重点查看：

```text
Signer #1 certificate SHA-256 digest
```

例如：

```text
Signer #1 certificate SHA-256 digest:
AA:BB:CC:DD:...
```

SHA-256 可以用于判断两个 APK 是否使用同一份签名证书。

---

# 七、从手机提取旧 APK 检查签名

首先获取 APK 路径：

```bash
adb shell pm path com.xxx.app
```

可能返回：

```text
package:/data/app/~~xxx/com.xxx.app-xxx/base.apk
```

然后：

```bash
adb pull /data/app/~~xxx/com.xxx.app-xxx/base.apk old.apk
```

检查旧 APK：

```bash
apksigner verify --print-certs old.apk
```

得到：

```text
旧 APK SHA-256 = XXXXX
```

检查新 APK：

```bash
apksigner verify --print-certs app-debug.apk
```

如果：

```text
旧 APK SHA-256
        =
新 APK SHA-256
```

说明：

```text
签名一致
```

可以正常覆盖。

如果：

```text
旧 APK SHA-256
        ≠
新 APK SHA-256
```

说明：

```text
签名不一致
```

不能直接覆盖。

---

# 八、检查候选 debug.keystore

如果从旧电脑、备份、同事电脑找到一个：

```text
debug.keystore
```

可以执行：

```bash
keytool -list -v \
  -keystore debug.keystore \
  -alias androiddebugkey \
  -storepass android
```

Windows：

```cmd
keytool -list -v -keystore debug.keystore -alias androiddebugkey -storepass android
```

查看：

```text
SHA256
```

然后比较：

```text
旧 APK SHA-256
        =
候选 debug.keystore SHA-256
```

如果一致：

```text
找到原来的 Debug Keystore
```

这是最理想的情况。

---

# 九、build.gradle 指定 Debug Keystore

假设：

```text
Project/
├── app/
│   └── build.gradle
│
└── keystore/
    └── debug.keystore
```

可以：

```gradle
android {

    signingConfigs {
        debug {
            storeFile file("../keystore/debug.keystore")
            storePassword "android"
            keyAlias "androiddebugkey"
            keyPassword "android"
        }
    }

    buildTypes {

        debug {
            signingConfig signingConfigs.debug
        }

        release {
            // Release 配置
        }
    }
}
```

如果找到旧电脑的 `debug.keystore`：

```text
旧 debug.keystore
        ↓
新电脑 Gradle 指定使用
        ↓
生成新 Debug APK
        ↓
签名与旧 APK 一致
        ↓
adb install -r
        ↓
覆盖成功
        ↓
App 数据原样保留
```

这是最优方案。

---

# 十、adb install -r 是什么

执行：

```bash
adb install -r app-debug.apk
```

其中：

```text
-r = replace
```

作用：

```text
相同 packageName 已存在
        ↓
尝试覆盖安装
        ↓
保留 App 数据
```

但：

```text
-r
```

不能绕过签名校验。

必须满足：

```text
旧 APK 签名
      =
新 APK 签名
```

如果签名不同：

```text
adb install -r
      ↓
INSTALL_FAILED_UPDATE_INCOMPATIBLE
```

---

# 十一、允许版本降级

如果新 APK：

```text
versionCode < 已安装 APK
```

可以：

```bash
adb install -r -d app-debug.apk
```

其中：

```text
-r = 覆盖安装
-d = 允许 versionCode 降级
```

但是：

```text
-d
```

同样不能绕过签名校验。

---

# 十二、adb uninstall -k

执行：

```bash
adb uninstall -k com.xxx.app
```

其中：

```text
-k = keep data and cache
```

也就是：

```text
卸载 APK
    ↓
请求保留 App data/cache
```

但是非常重要：

```text
adb uninstall -k
```

不能用于绕过签名问题。

例如：

```text
旧 APK
package = com.xxx.app
签名 = A
    ↓
adb uninstall -k
    ↓
APK 被卸载
旧数据保留
    ↓
安装
package = com.xxx.app
签名 = B
    ↓
Android 仍可能进行签名身份校验
    ↓
INSTALL_FAILED_UPDATE_INCOMPATIBLE
```

这是 Android 的安全机制。

否则攻击者就可以：

```text
卸载目标 App
    ↓
保留目标 App 数据
    ↓
安装相同 packageName 的恶意 APK
    ↓
读取目标 App 数据
```

所以：

> `adb uninstall -k` 不是“换签名同时保留数据”的解决方案。

---

# 十三、找不到旧 debug.keystore 怎么办

如果：

```text
旧电脑无法访问
旧硬盘没有备份
Time Machine 没有
CI 没有
同事电脑没有
```

那么：

```text
无法从 APK 反推出原来的私钥
```

因为 APK 中包含：

```text
证书
公钥
签名信息
```

但不包含：

```text
私钥
```

所以不能：

```text
old.apk
   ↓
反推出
   ↓
debug.keystore
```

这是数字签名安全性的基础。

---

# 十四、旧 App 有重要数据怎么办

如果：

```text
旧 Debug App
      +
重要业务数据
      +
旧 Keystore 丢失
```

可以考虑：

```text
旧 Debug App
    ↓
run-as
    ↓
导出 App 私有数据
    ↓
验证备份
    ↓
彻底卸载旧 App
    ↓
安装新签名 Debug APK
    ↓
run-as
    ↓
恢复旧数据
```

---

# 十五、什么是 run-as

`run-as` 可以理解为：

> 让 adb shell 临时以指定 App 的 Linux UID 身份执行命令。

Android 每个 App 都有自己的 UID。

例如：

```text
adb shell
UID = 2000(shell)

com.xxx.app
UID = 10234
```

App 私有目录：

```text
/data/user/0/com.xxx.app/
```

通常只有：

```text
root
或
App 自己的 UID
```

能够访问。

普通：

```bash
adb shell ls /data/user/0/com.xxx.app
```

可能：

```text
Permission denied
```

但是 Debug App 可以：

```bash
adb shell run-as com.xxx.app ls
```

其过程：

```text
adb shell
    ↓
shell UID
    ↓
run-as com.xxx.app
    ↓
切换成 App UID
    ↓
访问 App 私有目录
```

---

# 十六、run-as 后面的 id 是什么

执行：

```bash
adb shell run-as com.xxx.app id
```

这里：

```text
id
```

不是让开发者填写一个 ID。

它是 Linux 的 `id` 命令。

作用：

```text
显示当前进程 UID / GID
```

例如：

```text
uid=10234(u0_a234)
gid=10234(u0_a234)
```

说明：

```text
run-as 成功
```

---

# 十七、run-as 的前提

通常目标 App 必须：

```text
debuggable = true
```

Debug APK 默认通常满足。

执行：

```bash
adb shell run-as com.xxx.app id
```

如果：

```text
uid=xxxxx
```

说明可以使用。

如果：

```text
run-as: package not debuggable
```

说明这套方案不能直接使用。

---

# 十八、run-as 常用命令

查看 UID：

```bash
adb shell run-as com.xxx.app id
```

查看当前目录：

```bash
adb shell run-as com.xxx.app pwd
```

查看 App 数据：

```bash
adb shell run-as com.xxx.app ls -la
```

查看所有文件：

```bash
adb shell run-as com.xxx.app find . -type f
```

查看 SharedPreferences：

```bash
adb shell run-as com.xxx.app ls -la shared_prefs
```

查看数据库：

```bash
adb shell run-as com.xxx.app ls -la databases
```

查看 files：

```bash
adb shell run-as com.xxx.app ls -la files
```

查看数据大小：

```bash
adb shell run-as com.xxx.app du -sh .
```

---

# 十九、App 私有数据主要有哪些

通常：

```text
/data/user/0/com.xxx.app/
```

包含：

```text
databases/
shared_prefs/
files/
cache/
code_cache/
no_backup/
```

主要价值：

```text
shared_prefs/
    ↓
配置、Token、用户状态等

databases/
    ↓
SQLite / Room / LitePal 数据

files/
    ↓
App 自己保存的文件

no_backup/
    ↓
App 明确不希望参与系统自动备份的数据

cache/
    ↓
缓存，一般不重要

code_cache/
    ↓
运行/编译缓存，一般不重要
```

---

# 二十、完整数据备份流程

## 20.1 确认 run-as

```bash
adb shell run-as com.xxx.app id
```

必须成功。

---

## 20.2 强制停止 App

```bash
adb shell am force-stop com.xxx.app
```

非常重要。

避免：

```text
SQLite 正在写
MMKV 正在写
SharedPreferences 正在修改
SDK 正在写文件
```

造成备份状态不一致。

---

## 20.3 查看数据

```bash
adb shell run-as com.xxx.app ls -la
```

查看大小：

```bash
adb shell run-as com.xxx.app du -sh .
```

查看文件：

```bash
adb shell run-as com.xxx.app find . -type f
```

---

## 20.4 完整备份

执行：

```bash
adb exec-out run-as com.xxx.app tar -cf - . > app_data_backup.tar
```

原理：

```text
/data/user/0/com.xxx.app/
        ↓
run-as 获得 App UID
        ↓
tar 打包
        ↓
输出到 stdout
        ↓
adb exec-out
        ↓
传到电脑
        ↓
app_data_backup.tar
```

---

# 二十一、备份完成后必须验证

千万不要：

```text
看到 app_data_backup.tar
        ↓
马上卸载 App
```

必须先验证。

查看文件大小：

```bash
ls -lh app_data_backup.tar
```

不能是：

```text
0B
```

查看内容：

```bash
tar -tf app_data_backup.tar
```

应该看到：

```text
./
./shared_prefs/
./shared_prefs/xxx.xml
./databases/
./databases/app.db
./databases/app.db-wal
./databases/app.db-shm
./files/
...
```

---

# 二十二、实际解压验证

建议：

```bash
mkdir verify
```

然后：

```bash
tar -xf app_data_backup.tar -C verify
```

检查：

```bash
ls -la verify
```

应该：

```text
verify/
├── databases/
├── files/
├── shared_prefs/
├── no_backup/
└── ...
```

确认重要文件真实存在。

---

# 二十三、建议复制第二份备份

如果数据非常重要：

```bash
cp app_data_backup.tar app_data_backup_copy.tar
```

甚至保存到另外一个磁盘。

最终：

```text
app_data_backup.tar

app_data_backup_copy.tar
```

只有确认备份有效以后，才能继续卸载旧 App。

---

# 二十四、可以额外单独备份关键目录

完整备份之外，还可以：

## SharedPreferences

```bash
adb exec-out run-as com.xxx.app tar -cf - shared_prefs > shared_prefs.tar
```

## Database

```bash
adb exec-out run-as com.xxx.app tar -cf - databases > databases.tar
```

## Files

```bash
adb exec-out run-as com.xxx.app tar -cf - files > files.tar
```

最终：

```text
app_data_backup.tar

shared_prefs.tar
databases.tar
files.tar
```

形成双保险。

---

# 二十五、卸载旧 App

确认备份有效以后：

```bash
adb uninstall com.xxx.app
```

这里使用正常卸载。

不要依赖：

```bash
adb uninstall -k
```

因为新旧 APK 签名不同。

流程：

```text
旧 App 数据
    ↓
已经人工备份
    ↓
彻底卸载旧 App
    ↓
清除旧 package/signature 状态
    ↓
安装新签名 APK
```

---

# 二十六、安装新 Debug APK

执行：

```bash
adb install app-debug.apk
```

正常：

```text
Success
```

---

# 二十七、第一次启动新 App

安装完成后：

```text
先启动一次新 App
```

目的：

```text
创建 Application
创建 data 目录
初始化数据库
初始化 SharedPreferences
初始化 SDK
```

然后：

```bash
adb shell am force-stop com.xxx.app
```

---

# 二十八、确认新 App 支持 run-as

执行：

```bash
adb shell run-as com.xxx.app id
```

必须成功。

查看：

```bash
adb shell run-as com.xxx.app ls -la
```

---

# 二十九、建议备份新 App 初始数据

恢复旧数据之前：

```bash
adb exec-out run-as com.xxx.app tar -cf - . > new_app_initial_data.tar
```

现在：

```text
app_data_backup.tar
    ↓
旧 App 数据

new_app_initial_data.tar
    ↓
新 App 初始数据
```

如果恢复失败，还有回退参考。

---

# 三十、恢复旧数据

恢复之前：

```bash
adb shell am force-stop com.xxx.app
```

然后：

```bash
cat app_data_backup.tar | adb shell "run-as com.xxx.app tar -xf -"
```

过程：

```text
电脑

app_data_backup.tar
        ↓
       adb
        ↓
run-as com.xxx.app
        ↓
tar 解压
        ↓
/data/user/0/com.xxx.app/
```

---

# 三十一、为什么使用 run-as 恢复

不要简单理解成：

```text
adb push
```

`run-as` 有一个重要优势：

```text
恢复文件
    ↓
由 App UID 执行
    ↓
文件属于 App 自己
```

避免：

```text
文件 owner = shell/root

App UID = u0_a123
```

导致：

```text
App 无权限访问恢复的数据
```

---

# 三十二、恢复后检查

先不要启动 App。

检查：

```bash
adb shell run-as com.xxx.app ls -la
```

SharedPreferences：

```bash
adb shell run-as com.xxx.app ls -la shared_prefs
```

数据库：

```bash
adb shell run-as com.xxx.app ls -la databases
```

files：

```bash
adb shell run-as com.xxx.app ls -la files
```

查看 UID：

```bash
adb shell run-as com.xxx.app id
```

查看文件 owner：

```bash
adb shell run-as com.xxx.app ls -ln databases
```

确认数据恢复正常后再启动 App。

---

# 三十三、启动后重点验证

需要验证：

```text
1. 登录状态

2. 用户配置

3. SharedPreferences

4. SQLite / Room / LitePal

5. MMKV

6. 本地文件

7. 设备列表

8. 第三方 SDK 状态

9. Token / Session

10. 核心业务数据
```

---

# 三十四、哪些数据通常可以直接迁移

一般：

| 数据 | 默认依赖 AndroidKeyStore | run-as 迁移 |
|---|---|---|
| 普通 SharedPreferences | 否 | 可以 |
| SQLite | 否 | 可以 |
| Room | 否 | 可以 |
| LitePal | 否 | 可以 |
| 普通 files | 否 | 可以 |
| JSON | 否 | 可以 |
| 普通 MMKV | 否 | 通常可以 |
| Hive | 否 | 通常可以 |
| Cache | 否 | 可以但通常没必要 |
| EncryptedSharedPreferences | 是 | 有风险 |
| EncryptedFile | 是 | 有风险 |
| SQLCipher | 不一定 | 有风险 |
| AndroidKeyStore 自定义加密 | 是 | 有风险 |

---

# 三十五、AndroidKeyStore 为什么是特殊情况

例如：

```text
AndroidKeyStore
      ↓
保存 AES Key
      ↓
加密 Token
      ↓
密文保存到 SharedPreferences
```

`run-as` 可以备份：

```text
SharedPreferences 中的密文
```

但不能简单备份：

```text
AndroidKeyStore 中不可导出的 Key
```

卸载 App 后：

```text
旧 App
    ↓
AndroidKeyStore Key
    ↓
可能被删除
```

重新安装：

```text
新 App
    ↓
没有旧 Key
    ↓
恢复旧 SharedPreferences
    ↓
只有密文
    ↓
无法解密
```

所以：

```text
文件恢复成功
    ≠
所有数据一定可用
```

---

# 三十六、如何检查项目是否使用 AndroidKeyStore

全局搜索：

```text
AndroidKeyStore

KeyStore.getInstance

MasterKey

MasterKeys

EncryptedSharedPreferences

EncryptedFile

KeyGenerator

KeyGenParameterSpec

SQLCipher

net.sqlcipher
```

最关键的是：

```text
AndroidKeyStore
```

例如：

```kotlin
KeyStore.getInstance("AndroidKeyStore")
```

如果存在，需要继续追踪：

```text
这个 Key
    ↓
到底用于加密什么数据
```

---

# 三十七、第三方 SDK 也可能使用 AndroidKeyStore

即使自己项目没有：

```text
AndroidKeyStore
```

也不能完全确定没有依赖。

因为：

```text
第三方 AAR
第三方 JAR
第三方 SO
```

内部可能使用：

```text
AndroidKeyStore
```

尤其：

```text
账号 SDK
IoT SDK
支付 SDK
安全 SDK
推送 SDK
身份认证 SDK
```

可能保存：

```text
Session
Token
设备身份
认证信息
SDK 登录凭证
```

所以第三方 SDK 的本地状态恢复后需要实际验证。

---

# 三十八、数据库迁移注意事项

假设：

```text
旧 APK = v2.2

新 APK = v2.3
```

恢复：

```text
v2.2 数据库
    ↓
v2.3 App
```

如果使用 Room：

```text
旧 Schema
    ↓
Migration
    ↓
新 Schema
```

如果 Migration 正确：

```text
正常
```

如果新旧版本跨度很大：

```text
数据库 Schema
SharedPreferences 格式
SDK 数据格式
```

都可能变化。

所以最稳妥的是：

```text
旧 APK
    ↓
找到对应 Git Commit
    ↓
使用新电脑的新签名
    ↓
生成代码完全相同的 APK
    ↓
恢复数据
    ↓
确认运行正常
    ↓
再升级代码版本
```

即：

```text
先解决签名和数据迁移

再解决版本升级
```

不要同时引入两个变量。

---

# 三十九、SQLite 特别注意 WAL

SQLite / Room 可能存在：

```text
app.db
app.db-wal
app.db-shm
```

不要只复制：

```text
app.db
```

否则最新数据可能还在：

```text
app.db-wal
```

因此备份前必须：

```bash
adb shell am force-stop com.xxx.app
```

然后直接备份整个：

```text
databases/
```

这是比单独复制 `.db` 更稳妥的方式。

---

# 四十、run-as 不是进入 App 进程

例如：

```bash
adb shell run-as com.xxx.app ls
```

不是：

```text
进入 com.xxx.app 正在运行的进程
```

而是：

```text
启动一个 ls 命令
    ↓
让这个命令使用
    ↓
com.xxx.app 的 Linux UID
```

所以：

```bash
adb shell run-as com.xxx.app id
```

表示：

```text
以 App UID 执行 id
```

而：

```bash
adb shell run-as com.xxx.app cat shared_prefs/config.xml
```

表示：

```text
以 App UID 执行 cat
```

---

# 四十一、可以进入交互式 run-as Shell

执行：

```bash
adb shell
```

然后：

```bash
run-as com.xxx.app
```

再：

```bash
pwd
```

可能：

```text
/data/user/0/com.xxx.app
```

然后：

```bash
ls
```

可能：

```text
cache
code_cache
databases
files
shared_prefs
```

可以继续：

```bash
cd databases
ls -la
```

非常适合调试 App 数据。

---

# 四十二、外部存储不在这套备份范围内

上面的：

```bash
adb exec-out run-as com.xxx.app tar -cf - .
```

主要针对：

```text
/data/user/0/com.xxx.app/
```

如果 App 还在：

```text
/storage/emulated/0/Android/data/com.xxx.app/
```

保存重要数据，需要另外检查。

所以：

```text
App 完整数据
    ≠
只有 /data/user/0/
```

需要根据项目实际存储方式判断。

---

# 四十三、如果恢复后 App 崩溃

执行：

```bash
adb logcat
```

重点搜索：

```text
SecurityException

BadPaddingException

AEADBadTagException

KeyStoreException

SQLiteException

database disk image is malformed

ClassCastException

JsonSyntaxException
```

如果：

```text
AEADBadTagException
KeyStoreException
key not found
Failed to decrypt
```

高度怀疑：

```text
旧数据使用 AndroidKeyStore 加密
    ↓
卸载导致旧 Key 丢失
    ↓
密文虽然恢复
    ↓
新 App 无法解密
```

---

# 四十四、完整安全操作流程

最终建议按照下面流程执行：

```text
旧 Debug App
      │
      ↓
① adb shell run-as com.xxx.app id
      │
      ↓
确认 run-as 可用
      │
      ↓
② adb shell am force-stop com.xxx.app
      │
      ↓
停止所有数据写入
      │
      ↓
③ 查看数据目录
      │
      ↓
④ 完整 tar 备份
      │
      ↓
app_data_backup.tar
      │
      ↓
⑤ tar -tf 验证
      │
      ↓
⑥ 实际解压验证
      │
      ↓
⑦ 复制第二份备份
      │
      ↓
★★★★★
确认备份真的有效
★★★★★
      │
      ↓
⑧ adb uninstall com.xxx.app
      │
      ↓
彻底卸载旧签名 App
      │
      ↓
⑨ adb install app-debug.apk
      │
      ↓
安装新签名 App
      │
      ↓
⑩ 启动一次新 App
      │
      ↓
创建新数据目录
      │
      ↓
⑪ force-stop
      │
      ↓
⑫ run-as 确认新 App 可调试
      │
      ↓
⑬ 备份新 App 初始数据
      │
      ↓
new_app_initial_data.tar
      │
      ↓
⑭ force-stop
      │
      ↓
⑮ 恢复旧 app_data_backup.tar
      │
      ↓
⑯ 检查文件 / UID / 数据库
      │
      ↓
⑰ 启动 App
      │
      ↓
⑱ 验证业务数据
```

---

# 四十五、核心命令速查

## 查看 Android 版本

```bash
adb shell getprop ro.build.version.release
```

查看 API Level：

```bash
adb shell getprop ro.build.version.sdk
```

---

## 查看 APK 路径

```bash
adb shell pm path com.xxx.app
```

---

## 查看 APK 签名

```bash
apksigner verify --print-certs app-debug.apk
```

---

## 查看当前 Debug 签名配置

```bash
./gradlew signingReport
```

---

## 测试 run-as

```bash
adb shell run-as com.xxx.app id
```

---

## 查看 App 私有目录

```bash
adb shell run-as com.xxx.app ls -la
```

---

## 停止 App

```bash
adb shell am force-stop com.xxx.app
```

---

## 完整备份

```bash
adb exec-out run-as com.xxx.app tar -cf - . > app_data_backup.tar
```

---

## 查看备份内容

```bash
tar -tf app_data_backup.tar
```

---

## 完整卸载

```bash
adb uninstall com.xxx.app
```

---

## 安装新 APK

```bash
adb install app-debug.apk
```

---

## 相同签名覆盖安装

```bash
adb install -r app-debug.apk
```

---

## 相同签名且允许版本降级

```bash
adb install -r -d app-debug.apk
```

---

## 恢复数据

macOS / Linux：

```bash
cat app_data_backup.tar | adb shell "run-as com.xxx.app tar -xf -"
```

---

# 四十六、最终结论

## 情况一：能找到旧 debug.keystore

这是最优方案：

```text
找到旧 debug.keystore
        ↓
新电脑使用旧签名
        ↓
重新生成 Debug APK
        ↓
确认 SHA-256 一致
        ↓
adb install -r
        ↓
直接覆盖
        ↓
App 数据完整保留
```

优先级：

```text
★★★★★
```

---

## 情况二：旧 debug.keystore 永久丢失

使用：

```text
旧 Debug App
    ↓
run-as
    ↓
备份 /data/user/0/<package>
    ↓
确认备份
    ↓
彻底卸载
    ↓
安装新签名 APK
    ↓
run-as
    ↓
恢复普通文件数据
```

但要注意：

```text
AndroidKeyStore
第三方 SDK 安全密钥
硬件密钥
加密数据库
EncryptedSharedPreferences
```

可能无法通过普通文件备份完整迁移。

---

# 四十七、一句话记忆

Debug APK 不是没有签名，而是 Android Gradle Plugin 默认使用 `~/.android/debug.keystore` 自动完成了签名。

换电脑后，如果 Debug Keystore 发生变化：

```text
相同 packageName + 不同签名
        ↓
不能覆盖安装
```

找得到旧 Keystore：

```text
旧签名 + adb install -r
        ↓
直接保留数据
```

找不到旧 Keystore，但旧包是 Debug 包：

```text
run-as
   ↓
备份 App 私有数据
   ↓
卸载
   ↓
安装新包
   ↓
恢复数据
```

而 `run-as` 的核心可以记成：

> **`run-as` = 在不 Root 手机的情况下，以 Debug App 自己的 Linux UID 执行命令，从而合法访问该 App 的私有数据。**