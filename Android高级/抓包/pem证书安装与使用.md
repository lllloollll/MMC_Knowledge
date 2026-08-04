# Android HTTPS 抓包完整笔记：Mac PEM证书安装、Whistle代理、Network Security Config、SSL Pinning排查

==举个例子，Android APP中访问了域名为`admin.casdon.com.cn`的接口，如果要对该域名的接口进行抓包，先要获取该域名的证书casdon.pem文件，在mac上双击安装，然后在“钥匙串访问”-->"系统根证书"找到该证书，双击后点击“信任”-->"使用此证书时"-->"始终信任"。同时，APP网络设置允许系统证书和用户证书。开启抓包后，APP的请求经过电脑端的代理，将使用该用户证书进行认证。 ==

## 一、背景

在 Android APP 开发过程中，经常需要抓取 HTTPS 请求，例如：

- 调试接口参数
    
- 分析网络请求
    
- 排查三方 SDK 网络问题
    
- 查看 SSE/WebSocket/HTTP 长连接数据
    

常用工具：

- Whistle
    
- Charles
    
- Burp Suite
    

本次环境：

```
Mac
 |
Whistle
 |
Android手机
 |
APP
 |
OkHttp / 三方SDK
```

遇到的问题：

```
CONNECT a1.tuyacn.com:443 HTTP/1.1

Host: a1.tuyacn.com:443

User-Agent: okhttp/4.12.0

aborted
```

说明：

- HTTP代理连接成功
    
- HTTPS TLS阶段失败
    
- APP主动断开连接
    

---

# 二、HTTPS抓包原理

## 1. 正常HTTPS流程

正常情况下：

```
APP

↓

服务器证书

↓

系统CA验证

↓

建立TLS连接

↓

HTTPS通信
```

验证链：

```
api.xxx.com证书

↓

可信CA

↓

系统信任
```

---

## 2. 代理抓HTTPS流程

Whistle抓HTTPS时：

```
APP

↓

Whistle生成伪造证书

↓

服务器
```

例如：

APP访问：

```
https://a1.tuyacn.com
```

实际收到：

```
a1.tuyacn.com

↓

Whistle签发证书

↓

Whistle CA
```

因此：

APP必须信任：

```
Whistle CA
```

否则：

```
SSLHandshakeException
```

---

# 三、Mac安装PEM证书

## 1. PEM证书格式

PEM通常：

```
-----BEGIN CERTIFICATE-----

xxxxxx

-----END CERTIFICATE-----
```

常见格式：

```
.pem
.cer
.crt
```

本质都是X509证书。

---

# 四、Mac通过钥匙串安装PEM

步骤：

```
双击 xxx.pem

↓

钥匙串访问(Keychain Access)

↓

选择存储位置

↓

系统(System)

↓

添加
```

安装后：

打开：

```
钥匙串访问

↓

系统

↓

证书
```

找到对应证书。

双击：

```
信任(Trust)

↓

使用此证书时

↓

始终信任(Always Trust)
```

输入Mac密码。

完成。

---

# 五、Mac命令行安装PEM

例如：

证书：

```
~/Downloads/whistle.pem
```

执行：

```bash
sudo security add-trusted-cert \
-d \
-r trustRoot \
-k /Library/Keychains/System.keychain \
~/Downloads/whistle.pem
```

查看：

```bash
security find-certificate \
-a \
/Library/Keychains/System.keychain
```

---

# 六、Android安装CA证书

Android：

```
设置

↓

安全

↓

更多安全设置

↓

从存储设备安装证书

↓

CA证书

↓

选择whistle.pem
```

安装后：

```
设置

↓

安全

↓

可信凭据

↓

用户
```

可以看到：

```
Whistle CA
```

---

# 七、Android CA信任机制

Android存在两类CA。

## 1. system CA

系统内置CA：

```
/system/etc/security/cacerts/
```

例如：

- DigiCert
    
- GlobalSign
    
- Let's Encrypt
    
- Google Trust Services
    

特点：

Android默认信任。

---

## 2. user CA

用户安装CA：

例如：

- Whistle CA
    
- Charles CA
    
- Burp CA
    
- 公司代理CA
    

来源：

```
设置 → 安全 → 安装证书
```

---

# 八、Android 7.0之后CA变化

Android 6及以前：

默认：

```
system CA

+

user CA
```

Android 7以后：

默认：

```
system CA
```

原因：

安全。

如果用户安装恶意CA：

```
恶意CA

↓

HTTPS中间人攻击

↓

窃取HTTPS数据
```

所以：

Android 7+ APP默认不信任用户CA。

---

# 九、Network Security Config

文件：

```
res/xml/network_security_config.xml
```

Manifest：

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config">
</application>
```

如果没有配置：

XML不会生效。

---

# 十、certificates src区别

## 1. system + user

配置：

```xml
<trust-anchors>

    <certificates src="system"/>

    <certificates src="user"/>

</trust-anchors>
```

含义：

APP信任：

```
系统CA

+

用户CA
```

可以抓：

```
Whistle HTTPS

Charles HTTPS

Burp HTTPS
```

适合：

```
Debug环境
```

---

## 2. 仅system

配置：

```xml
<trust-anchors>

    <certificates src="system"/>

</trust-anchors>
```

含义：

只信任：

```
系统CA
```

不信任：

```
Whistle CA

Charles CA
```

结果：

```
TLS握手失败
```

---

# 十一、Debug和Release配置建议

## Debug

目标：

```
支持抓包调试
```

配置：

```xml
<certificates src="system"/>
<certificates src="user"/>
```

目录：

```
src/debug/res/xml/network_security_config.xml
```

---

## Release

目标：

```
提高安全
```

配置：

```xml
<certificates src="system"/>
```

目录：

```
src/release/res/xml/network_security_config.xml
```

---

# 十二、Whistle HTTPS抓包排查流程

## 1. 确认代理

手机：

```
WiFi

↓

代理

↓

手动

↓

Mac IP

↓

Whistle端口
```

Whistle出现：

```
CONNECT xxx:443
```

说明：

代理成功。

---

## 2. 确认CA安装

检查：

Android：

```
用户证书
```

是否存在：

```
Whistle CA
```

---

## 3. 确认APP允许用户CA

检查：

```
network-security-config.xml
```

必须包含：

```xml
<certificates src="user"/>
```

---

# 十三、Logcat排查

过滤：

```
SSL

Handshake

Trust

Certificate

OkHttp
```

---

## 情况1：证书链失败

错误：

```
SSLHandshakeException
```

例如：

```
Trust anchor for certification path not found
```

原因：

APP不信任Whistle CA。

解决：

开启：

```xml
<certificates src="user"/>
```

---

## 情况2：SSL Pinning失败

错误：

```
SSLPeerUnverifiedException
```

例如：

```
Certificate pinning failure
```

原因：

APP进行了二次证书校验。

---

# 十四、SSL Pinning原理

普通HTTPS：

```
服务器证书

↓

系统CA验证

↓

通过
```

SSL Pinning：

```
服务器证书

↓

系统CA验证

↓

APP再次检查证书指纹

↓

通过
```

例如OkHttp：

```kotlin
CertificatePinner.Builder()
    .add(
        "api.xxx.com",
        "sha256/xxxx"
    )
    .build()
```

---

# 十五、为什么配置user CA仍然抓不到？

Network Security Config解决：

```
系统CA验证
```

SSL Pinning：

```
APP代码主动验证证书
```

流程：

```
Whistle替换证书

↓

系统CA验证

↓

成功

↓

SSL Pinning检查

↓

失败
```

所以：

```
<certificates src="user"/>
```

无法解决SSL Pinning。

---

# 十六、三方SDK抓包失败分析

例如：

```
Tuya SDK

OkHttp

a1.tuyacn.com
```

可能原因：

---

## 1. 没配置Network Security Config

概率：

20%

检查：

Manifest：

```xml
android:networkSecurityConfig
```

---

## 2. SDK自定义网络层

概率：

30%

例如：

```
SSLContext

TrustManager

HostnameVerifier
```

SDK绕过系统配置。

---

## 3. SSL Pinning

概率：

50%

表现：

```
CONNECT成功

HTTPS失败

aborted
```

进入：

```
Frida Hook

绕过CertificatePinner

绕过TrustManager
```

---

# 十七、本次问题定位结论

当前：

```
Android APP

+

Whistle

+

OkHttp

+

Tuya SDK
```

已经确认：

✅ 手机代理成功

```
CONNECT a1.tuyacn.com:443
```

下一步：

1. 检查Android是否安装Whistle CA
    
2. 检查network-security-config是否包含：
    

```xml
<certificates src="user"/>
```

3. 查看Logcat：
    

```
SSLHandshakeException

Certificate pinning failure
```

4. 如果确认Pinning：
    

进入：

```
Android SSL Pinning绕过

↓

Frida Hook

↓

三方SDK分析
```

---

# 十八、核心总结

一句话：

> system控制Android是否信任系统CA，user控制Android是否信任用户安装CA。Debug抓包必须开启user，Release通常只保留system。

完整排查链路：

```
Mac安装PEM证书

↓

Android安装CA

↓

手机设置代理

↓

APP允许user CA

↓

TLS握手成功

↓

HTTPS解密

↓

失败则检查SSL Pinning

↓

Frida绕过
```

---

# 十九、后续学习路线

```
HTTPS基础

↓

TLS握手流程

↓

OkHttp源码分析

↓

CertificatePinner原理

↓

TrustManager机制

↓

Frida Hook

↓

三方SDK网络分析
```