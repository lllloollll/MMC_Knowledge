# Clash Verge TUN 模式、Rule规则、Umeng绕过配置笔记

## 一、问题背景

开发环境：

- macOS
    
- Clash Verge
    
- Android 开发
    
- Git / Gradle / Android Studio / Flutter
    
- Whistle 抓包调试
    

遇到问题：

> 开启 Clash Verge 后无法访问 Umeng（友盟），需要配置绕过。

同时发现：

- 开启 TUN 模式可以正常访问外网
    
- 关闭 TUN，仅开启系统代理 + Rule 模式时无法访问外网
    
- 需要理解 TUN、系统代理、Rule规则之间的关系
    

---

# 二、Clash Verge 三个核心概念

## 1. 系统代理（System Proxy）

系统代理本质是：

设置 HTTP/HTTPS Proxy：

```
127.0.0.1:7890
```

请求链路：

```
浏览器
  |
  ↓
系统代理
  |
  ↓
Clash Verge
  |
  ↓
代理节点
  |
  ↓
目标网站
```

优点：

- 配置简单
    
- 浏览器访问方便
    

缺点：

不是所有程序都会使用系统代理。

可能不走系统代理的软件：

- Git
    
- Gradle
    
- Maven
    
- Android Studio部分请求
    
- Flutter
    
- Java程序
    
- Docker
    
- 部分命令行工具
    

---

## 2. TUN模式

TUN = Tunnel。

Clash 创建虚拟网卡，接管系统网络流量。

链路：

```
应用程序
 |
 ↓
TUN虚拟网卡
 |
 ↓
Clash规则匹配
 |
 ├── DIRECT
 |
 └── Proxy
```

重点：

> TUN接管流量 ≠ 所有流量走代理。

是否消耗代理流量，取决于 Rule：

```
请求进入Clash
        |
        ↓
规则判断
        |
        ├── DIRECT
        |       ↓
        |    直连，不消耗代理流量
        |
        └── Proxy
                ↓
             消耗代理流量
```

---

## 3. Rule模式

Rule模式根据规则决定：

- DIRECT
    
- Proxy
    
- REJECT
    

例如：

```
百度
 |
 ↓
DIRECT


Github
 |
 ↓
Proxy
```

---

# 三、为什么推荐 TUN + Rule？

Android开发环境：

需要访问：

- GitHub
    
- Gradle依赖
    
- Maven仓库
    
- Flutter依赖
    
- HuggingFace
    
- Docker
    

这些工具很多不会读取系统代理。

如果只使用系统代理：

需要分别配置：

```
Git代理
Gradle代理
Android Studio代理
Flutter代理
npm代理
pip代理
```

非常麻烦。

推荐：

```
TUN        开启
系统代理    开启
模式        Rule
```

效果：

```
浏览器
   |
系统代理


Git
Gradle
Flutter
Android Studio
   |
TUN


        ↓

Clash Rule分流
```

---

# 四、macOS系统代理配置

路径：

```
系统设置
 ↓
网络
 ↓
WiFi/以太网
 ↓
详细信息
 ↓
代理
```

Clash Verge默认：

## 网页代理 HTTP

开启：

```
服务器：
127.0.0.1

端口：
7890
```

## 网页安全代理 HTTPS

开启：

```
服务器：
127.0.0.1

端口：
7890
```

不要配置：

```
自动代理配置
```

除非使用PAC。

一般不用：

```
SOCKS代理
```

除非专门给Git、SSH等配置。

---

# 五、Clash Rule规则原理

规则按照顺序匹配：

```
第一条匹配成功
立即停止
```

例如：

```yaml
rules:

- DOMAIN-SUFFIX,umeng.com,DIRECT

- GEOIP,CN,DIRECT

- MATCH,Proxy
```

访问：

```
api.umeng.com
```

匹配：

```
DOMAIN-SUFFIX,umeng.com
```

直接：

```
DIRECT
```

不会继续匹配下面规则。

---

# 六、Umeng绕过配置

## 原问题

开启代理后：

```
Umeng无法访问
```

原因：

Umeng请求被送入代理节点。

解决：

增加DIRECT规则。

配置：

```yaml
rules:

# Umeng直连
- DOMAIN-SUFFIX,umeng.com,DIRECT
- DOMAIN-SUFFIX,umengcloud.com,DIRECT
- DOMAIN-SUFFIX,umsns.com,DIRECT

# 国内域名直连
- DOMAIN-SUFFIX,cn,DIRECT

# 国内IP直连
- GEOIP,CN,DIRECT

# 其他走代理
- MATCH,🚀节点选择
```

注意：

Umeng规则必须放在：

```
GEOIP
MATCH
```

之前。

---

# 七、当前 Clash 配置分析

当前规则：

```yaml
- DOMAIN-SUFFIX,cn,DIRECT
- GEOIP,cn,DIRECT,resolve
- MATCH,🐟漏网之鱼
```

说明：

流程：

```
.cn域名
 ↓
DIRECT


中国IP
 ↓
DIRECT


其他
 ↓
🐟漏网之鱼
```

---

# 八、发现的问题

当前：

```
MATCH,🐟漏网之鱼
```

而：

```
🐟漏网之鱼
```

配置：

```yaml
type: select

proxies:

- 🚀节点选择
- 日本节点
- 美国节点
- DIRECT
```

意味着：

最终出口由当前选择决定。

如果：

```
🐟漏网之鱼 = DIRECT
```

那么：

国外访问：

```
Google
Github

↓

DIRECT

↓

失败
```

---

# 九、优化建议

开发环境推荐：

将：

```yaml
- MATCH,🐟漏网之鱼
```

修改为：

```yaml
- MATCH,🚀节点选择
```

原因：

避免中间选择组影响。

最终：

```yaml
rules:

# 本地规则

...

# Umeng
- DOMAIN-SUFFIX,umeng.com,DIRECT
- DOMAIN-SUFFIX,umengcloud.com,DIRECT
- DOMAIN-SUFFIX,umsns.com,DIRECT


# 国内
- DOMAIN-SUFFIX,cn,DIRECT
- GEOIP,CN,DIRECT


# 其他
- MATCH,🚀节点选择
```

---

# 十、开发环境推荐配置

## 日常开发

```
Clash Verge:

TUN:
开启

系统代理:
开启

模式:
Rule
```

适合：

- Android Studio
    
- Git
    
- Gradle
    
- Flutter
    
- GitHub
    
- AI工具
    

---

## Whistle抓包

不要同时：

```
手机代理
 ↓
Whistle
 ↓
Clash TUN
 ↓
代理节点
```

容易产生：

- HTTPS异常
    
- CONNECT失败
    
- 证书问题
    

抓包时：

关闭：

```
TUN
系统代理
```

开启：

```
Whistle
```

手机：

```
WiFi代理
 ↓
Mac IP:8899
```

---

# 十一、排查流程总结

## 1. 外网打不开

检查：

```
Clash模式
```

确认：

```
Rule
```

---

## 2. 检查MATCH出口

查看：

```
proxy-groups
```

确认：

```
MATCH
```

最终进入：

```
代理节点
```

而不是：

```
DIRECT
```

---

## 3. 查看连接日志

访问：

```
github.com
google.com
```

检查：

正确：

```
github.com
 ↓
🚀节点选择
```

错误：

```
github.com
 ↓
DIRECT
```

---

# 十二、最终结论

Clash Verge最佳开发配置：

```
TUN + Rule + 系统代理
```

TUN负责：

```
接管所有程序流量
```

Rule负责：

```
决定哪些直连
哪些代理
```

不要担心TUN浪费代理流量：

真正消耗流量的是：

```
Proxy规则
```

不是：

```
TUN模式
```

Android开发环境：

推荐：

```
国内服务:
DIRECT


国外开发资源:
Proxy
```

这样可以兼顾：

- 开发效率
    
- 代理流量控制
    
- Git/Gradle稳定访问
    
- Whistle抓包调试