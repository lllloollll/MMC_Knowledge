# Whistle Android 抓包与接口调试完全笔记

> 用途：
>
> - Android App HTTP/HTTPS 抓包
> - API接口分析
> - SSE流式接口分析
> - Mock接口
> - 弱网测试
> - 请求/响应修改
> - APK逆向前期网络分析


---

# 一、Whistle简介


## 1.1 什么是Whistle

Whistle是一个基于Node.js的HTTP代理调试工具。

核心能力：

- HTTP抓包
- HTTPS抓包
- 修改Request
- 修改Response
- Mock接口
- 请求重定向
- 弱网模拟
- SSE流式数据查看


工作流程：

```
Android App

      |
      | HTTP/HTTPS

      ↓

Whistle Proxy

      |

      ↓

服务器
```


HTTPS流程：

```
Android App

      |
      | 加密请求

      ↓

Whistle

      |
      | 解密

      ↓

服务器
```


---

# 二、安装Whistle


## 2.1 安装Node.js


检查：

```bash
node -v

npm -v
```


Mac安装：

```bash
brew install node
```


---

## 2.2 安装Whistle


```bash
npm install -g whistle
```


检查：

```bash
w2 -V
```


---

# 三、启动Whistle


启动：

```bash
w2 start
```


默认地址：

```
http://127.0.0.1:8899
```


打开：

```
http://127.0.0.1:8899/
```


---

## 查看状态


```bash
w2 status
```


示例：

```
Whistle@2.10.8 is running


http://127.0.0.1:8899/

http://172.xxx.xxx.xxx:8899/
```


---

# 四、Android连接Whistle


## 4.1 获取Mac局域网IP


查看：

```bash
ifconfig
```


或者：

```bash
ipconfig getifaddr en0
```


注意：

Whistle可能显示多个IP：

例如：

```
127.0.0.1
172.16.231.25
172.16.232.32
198.18.0.1
```


不要盲目使用。


测试：

手机浏览器访问：

```
http://IP:8899
```


哪个能打开Whistle页面，就使用哪个。


实际环境：

```
172.16.232.32:8899
```

可以正常访问。


---

## 4.2 Android设置代理


路径：

```
设置

↓

WiFi

↓

当前网络

↓

修改网络

↓

高级选项

↓

代理
```


选择：

```
手动
```


填写：

```
代理主机：

172.16.232.32


代理端口：

8899
```


---

# 五、代理异常排查


## 5.1 手机能访问Whistle，但是开启代理后无法上网


排查顺序：


---

## ① 检查Whistle监听


```bash
lsof -i :8899
```


正常：

```
node TCP *:8899 LISTEN
```


说明：

Whistle监听正常。


---

## ② 测试Whistle代理能力


```bash
curl -x http://172.16.232.32:8899 http://example.com -I
```


正常：

```
HTTP/1.1 200 OK
```


说明：

Whistle代理能力正常。


---

## ③ 检查手机代理配置


正确：

```
代理类型：

手动


代理：

172.16.232.32


端口：

8899
```


错误：

```
http://172.16.232.32

172.16.232.32:8899
```


---

## ④ Clash Verge影响


如果电脑开启：

- Clash Verge
- TUN模式
- VPN


可能产生多个虚拟网卡。


例如：

```
198.18.x.x

172.16.x.x
```


这些可能不是手机可访问地址。


建议：

抓包时：

- 关闭TUN模式
- 开启Allow LAN


---

# 六、HTTPS抓包


## 6.1 开启HTTPS


Whistle页面：

```
HTTPS

↓

Enable HTTPS
```


---

## 6.2 安装CA证书


手机访问：

```
http://rootca.pro/
```


下载：

```
rootCA.cer
```


安装：【手机设置直接搜索“证书”，找到下载的证书安装。】

```
设置

↓

安全

↓

加密与凭据

↓

安装证书

↓

CA证书
```


---

# 七、HTTPS抓包失败排查


## 情况1：只能看到CONNECT


例如：

```
CONNECT api.xxx.com:443
```


没有：

```
GET
POST
Response
```


原因：

HTTPS解密失败。


检查：

- HTTPS是否开启
- CA证书是否安装
- SSL Pinning


---

## 情况2：浏览器可以抓，App不能抓


原因：

Android 7以后：

```
系统证书

√


用户证书

×
```


或者：

App开启：

```
SSL Pinning
```


需要：

- network_security_config
- Frida Hook
- 修改APK


---

# 八、Android App抓包流程


步骤：


```
手机设置代理

↓

安装Whistle证书

↓

打开App

↓

执行操作

↓

Whistle Network查看请求
```


查看：

```
Network

↓

请求

↓

Request

↓

Response
```


可以分析：

- URL
- Header
- Token
- Cookie
- 参数
- 返回数据


---

# 九、SSE抓包


## 9.1 SSE简介


SSE：

Server Sent Events


特点：

服务器持续向客户端推送数据。


普通HTTP：

```
请求

↓

完整Response

↓

结束
```


SSE：

```
请求

↓

200 OK

↓

chunk1

↓

chunk2

↓

chunk3

↓

持续连接
```


---

# 十、Whistle抓SSE


## 10.1 SSE响应特点


例如：

```
HTTP/1.1 200 OK


Content-Type:
text/event-stream


Transfer-Encoding:
chunked


Connection:
keep-alive
```


说明：

这是SSE。


---

## 10.2 Response看不到内容


原因：

SSE没有完整Response Body。


因为：

```
连接不会结束

数据持续发送
```


---

## 10.3 查看SSE数据


Whistle：

```
Network

↓

Frames
```


可以看到：

```
chunk
```


例如：

```
event:message

data:{"content":"你好"}


event:done

```


---

# 十一、SSE协议分析


常见格式：


## OpenAI风格


```
data:
{
 "delta":{
   "content":"你好"
 }
}
```


---

## 自定义事件


```
event:message

data:{}


event:done
```


---

# 十二、SSE失败模拟


## 12.1 模拟HTTP错误


Rule：

```
https://api.xxx.com/chat statusCode://500
```


测试：

- HTTP失败
- error处理


---

## 12.2 Mock SSE响应


创建：

```
mock_sse.txt
```


内容：

```
event:message

data:{"content":"hello"}


event:error

data:{"code":500}

```


Rule：

```
https://api.xxx.com/chat file:///path/mock_sse.txt
```


---

# 十三、Whistle常用Rule


## 13.1 修改响应


服务器：

```json
{
 "vip":false
}
```


Rule：

```
https://api.xxx.com/user resBody://{"vip":true}
```


效果：

App收到：

```json
{
 "vip":true
}
```


用途：

- 测试会员
- 权限


---

## 13.2 修改请求


修改：

```
uid=100
```


为：

```
uid=999
```


---

## 13.3 请求重定向


线上：

```
api.xxx.com
```


转：

```
test.xxx.com
```


用途：

- 不重新打包切环境


---

## 13.4 延迟接口


模拟弱网：

```
api.xxx.com reqDelay://3000
```


效果：

接口延迟3秒。


用途：

测试：

- Loading
- 超时
- 重试


---

## 13.5 本地Mock


准备：

```
user.json
```


Rule：

```
api.xxx.com/user file:///path/user.json
```


用途：

前后端分离开发。


---

# 十四、Android开发中的Whistle应用


## 接口调试

快速查看：

- 请求地址
- 参数
- Header
- Token


---

## 异常测试


模拟：

- 500
- 空数据
- 错误字段
- 超时


---

## 弱网测试


模拟：

- 慢响应
- 网络延迟
- 请求失败


---

## AI助手开发


非常适合分析：

- Chat接口
- SSE流式输出
- Token
- Prompt请求
- 对话协议


---

# 十五、Android高级开发推荐工具链


## 网络分析


### Whistle

用途：

开发调试、Mock、弱网。


### Charles

用途：

企业常用抓包。


### mitmproxy

用途：

自动化抓包、逆向。


---

## 性能分析


### Android Studio Profiler

分析：

- CPU
- Memory
- Network


---

### Perfetto

分析：

- 卡顿
- ANR
- Binder
- VSync


---

## UI调试


### Layout Inspector


分析：

- View层级
- 属性
- 实际尺寸


---

## 内存分析


### LeakCanary


分析：

- Activity泄漏
- Fragment泄漏


---

## 逆向分析


工具链：

```
Whistle

↓

jadx

↓

Frida

↓

so分析

↓

Native逆向
```


---

# 十六、个人工具链总结


Android高级开发：

```
Android Studio

+

ADB

+

Whistle

+

Profiler

+

Layout Inspector

+

LeakCanary

+

Perfetto
```


Android逆向：

```
Whistle

+

jadx

+

apktool

+

Frida

+

Ghidra/IDA
```


---

# 十七、学习路线


当前：

```
Whistle抓包

↓

HTTPS分析

↓

SSE分析

↓

SSL Pinning

↓

Frida Hook

↓

APK逆向

↓

Native分析
```


Whistle是Android开发和APK逆向之间的重要桥梁。