# JNI基础


## 1. JNI是什么

JNI(Java Native Interface)

Java和Native代码通信的接口规范。


Java运行在ART/JVM。

C/C++运行在Native环境。

JNI负责二者桥接。


---

## 2. 为什么需要JNI


Java优势：

- 安全
- 开发效率高
- GC管理


Native优势：

- 高性能
- 操作硬件
- 调用底层库


常见场景：

- 音视频
- OpenGL
- 加密
- 游戏
- 算法库


---

## 3. JNI调用链


Java

↓

native方法

↓

ART

↓

JNI

↓

C/C++

↓

系统


---

## 4. System.loadLibrary


Java:

System.loadLibrary("hello")


加载：

libhello.so


规则：

xxx

↓

libxxx.so


---

## 5. native方法


Java:

public native String sayHello();


表示：

方法实现不在Java中。


---

## 6. C函数结构


JNI函数：

JNIEXPORT 返回类型 JNICALL
Java_包名_类名_方法名(
    JNIEnv* env,
    jobject obj
)


---

## 7. JNIEnv


JNI环境指针。

提供：

- 创建对象
- 调用Java方法
- 转换数据
- 异常处理


---

## 8. jobject


代表Java对象。

类似Java中的this。


---

## 下一课

Java native声明

↓

JNI函数命名规则

↓

.so符号查找

↓

为什么会出现：

java.lang.UnsatisfiedLinkError
