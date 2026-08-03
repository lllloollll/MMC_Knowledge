# JNI/NDK 第2课笔记：Java Native声明、SO生成与JNI注册机制

> 本节目标：
>
> 1. 理解 Java native 方法如何调用 C/C++
> 2. 理解 libxxx.so 的生成过程
> 3. 理解 Android Native 编译流程
> 4. 掌握 JNI 静态注册
> 5. 掌握 JNI 动态注册
> 6. 理解静态注册与动态注册的区别

---

# 一、JNI整体调用链

Android Java 调用 Native 的完整流程：

```
Java/Kotlin

    |
    |
    ↓

native 方法声明

    |
    |
    ↓

System.loadLibrary()

    |
    |
    ↓

加载 libxxx.so

    |
    |
    ↓

JNI 注册

    |
    |
    ↓

调用 C/C++ 方法

```

---

# 二、System.loadLibrary("hello") 原理


Java代码：

```java
static {

    System.loadLibrary("hello");

}
```

并不是寻找：

```
hello.so
```


Android 会自动转换：

```
hello

↓

libhello.so

```


规则：

```
lib + 参数 + .so
```


例如：

```java
System.loadLibrary("abc");
```


加载：

```
libabc.so
```

---

# 三、libhello.so 从哪里来？

libhello.so 不是手写文件。

它由：

```
C/C++源码
        |
        ↓
NDK编译
        |
        ↓
CMake构建
        |
        ↓
libhello.so
```


例如：

```
hello.cpp

        ↓

clang++

        ↓

libhello.so

```

---

# 四、Android JNI工程结构


典型结构：

```
app

 └── src

     └── main

         ├── java

         │    └── com.demo.jni

         │          └── HelloJNI.java


         ├── cpp

         │    ├── hello.cpp

         │    └── CMakeLists.txt

         │
         └── AndroidManifest.xml

```

---

# 五、JNI.h来源


C++代码：

```cpp
#include <jni.h>
```


jni.h 不需要自己创建。


来源：

```
Android NDK
```

路径类似：

```
Android/sdk/ndk/
    toolchains/
        llvm/
            sysroot/
                usr/include/jni.h
```


CMake会自动配置 include路径。


---

# 六、为什么现在不需要生成.h文件？


旧JNI流程：

```
Java文件

↓

javah

↓

生成xxx.h

↓

C++ include

```


例如：

```
com_demo_HelloJNI.h
```


现在 Android Studio + CMake：

通常：

直接：

```cpp
Java_com_demo_HelloJNI_method()
```

即可。


因此：

不需要：

```
javah
.h文件
```

---

# 七、JNI静态注册


## 1. Java代码


```java
package com.demo.jni;


public class HelloJNI {


    static {

        System.loadLibrary("hello");

    }


    public native int add(int a,int b);


}

```


---

## 2. C++实现


```cpp
#include <jni.h>


extern "C"

JNIEXPORT jint JNICALL

Java_com_demo_jni_HelloJNI_add(

        JNIEnv *env,

        jobject thiz,

        jint a,

        jint b

){

    return a+b;

}

```


---

# 八、静态注册原理


Java:

```java
native int add()
```


对应C函数：

```
Java_包名_类名_方法名
```


例如：

Java:

```
com.demo.jni.HelloJNI.add
```


转换：

```
Java_com_demo_jni_HelloJNI_add
```


调用流程：

```
Java add()

        ↓

ART根据规则查找

        ↓

Java_com_demo_jni_HelloJNI_add

        ↓

执行C++函数

```

---

# 九、extern "C"作用


C++具有名字修饰(Name Mangling)


例如：

C++:

```cpp
int add(int a,int b)
```


编译后可能：

```
_Z3addii
```


JNI需要：

```
Java_com_demo_xxx_add
```


因此：

```cpp
extern "C"
```


告诉编译器：

不要修改函数名。


---

# 十、CMake生成so


CMakeLists.txt：

```cmake
cmake_minimum_required(VERSION 3.22.1)


project("hello")


add_library(

        hello

        SHARED

        hello.cpp

)


find_library(

        log-lib

        log

)


target_link_libraries(

        hello

        ${log-lib}

)

```


---

# 十一、add_library含义


代码：

```cmake
add_library(
        hello
        SHARED
        hello.cpp
)
```


含义：

创建共享库。


生成：

```
libhello.so
```


CMake自动添加：

```
lib
.so

```


---

# 十二、Native编译流程


完整流程：

```
hello.cpp

    ↓

clang++

    ↓

目标文件 .o

    ↓

链接

    ↓

libhello.so

    ↓

APK

    ↓

lib/arm64-v8a/libhello.so

```

---

# 十三、APK中的so位置


APK结构：

```
APK

├── classes.dex

└── lib

    └── arm64-v8a

          └── libhello.so

```


---

# 十四、多个so文件的区别


构建过程中可能出现：

```
build/intermediates/cxx

build/intermediates/merged_native_libs

build/intermediates/stripped_native_libs

```


它们不是多个不同库。


---

## 1. cxx目录


例如：

```
intermediates/cxx/.../obj/arm64-v8a/libhello.so
```


表示：

CMake编译生成的原始so。


---

## 2. merged_native_libs


例如：

```
merged_native_libs/out/lib/arm64-v8a/libhello.so
```


表示：

Gradle合并所有Native库后的结果。


---

## 3. stripped_native_libs


例如：

```
stripped_native_libs/out/lib/arm64-v8a/libhello.so
```


表示：

去除调试符号后的版本。


最终进入APK。


---

# 十五、JNI动态注册


动态注册不依赖函数名。


Java：

```java
public native int add(int a,int b);
```


C++：

```cpp
jint native_add(
        JNIEnv *env,
        jobject thiz,
        jint a,
        jint b
){

    return a+b;

}

```


函数名字可以任意。

---

# 十六、JNINativeMethod


动态注册核心结构：

```cpp
typedef struct {

    char* name;

    char* signature;

    void* fnPtr;


} JNINativeMethod;

```


三个字段：

---

## 1. Java方法名


```cpp
"add"
```


对应：

```java
native int add()
```

---

## 2. JNI方法签名


```cpp
"(II)I"
```


表示：

```
输入：

int
int


输出：

int

```


常见：

|Java类型|JNI签名|
|-|-|
|int|I|
|long|J|
|boolean|Z|
|void|V|
|String|Ljava/lang/String;|

---

## 3. C函数地址


```cpp
(void*)native_add
```


表示：

调用：

```
native_add()
```

---

# 十七、JNI_OnLoad


动态注册入口：

```cpp
JNI_OnLoad()
```


执行时机：

```
System.loadLibrary()

        ↓

加载so

        ↓

调用JNI_OnLoad

        ↓

RegisterNatives

        ↓

完成绑定

```

---

# 十八、动态注册完整流程


```
libhello.so加载


        ↓


JNI_OnLoad()


        ↓


FindClass()


        ↓


RegisterNatives()


        ↓


建立映射


Java add()

        ↓


native_add()

```

---

# 十九、静态注册 VS 动态注册


|项目|静态注册|动态注册|
|-|-|-|
|函数名|Java_xxx|任意|
|绑定方式|名称匹配|RegisterNatives|
|入口|native方法调用时|JNI_OnLoad|
|逆向难度|低|高|
|大型项目|少用|常用|


---

# 二十、为什么商业项目喜欢动态注册？


## 1. 防止直接搜索JNI入口


静态：

搜索：

```
Java_
```


可以找到大量入口。


动态：

需要分析：

```
JNI_OnLoad

↓

RegisterNatives

```


---

## 2. 包名修改影响小


静态：

```
com.demo.app

↓

com.demo.newapp

```


所有：

```
Java_xxx

```


需要修改。


动态：

只修改：

```
FindClass()

```


---

## 3. 函数名更短


静态：

```
Java_com_company_project_LoginManager_login
```


动态：

```
login
```


---

# 二十一、验证JNI注册方式


查看so：

```bash
nm libhello.so
```


静态注册：

可以看到：

```
Java_com_demo_jni_HelloJNI_add
```


动态注册：

看到：

```
JNI_OnLoad
```


不会看到：

```
Java_xxx
```


---

# 二十二、本课重点


必须掌握：

1. native方法没有Java实现；
2. System.loadLibrary加载的是libxxx.so；
3. libxxx.so由NDK+CMake生成；
4. JNI.h由NDK提供；
5. 现代Android不需要javah生成.h；
6. 静态注册依靠函数名；
7. 动态注册依靠JNI_OnLoad；
8. RegisterNatives负责建立映射；
9. 商业项目大量使用动态注册。


---

# 下一课预告

## JNI第3课：libhello.so加载全过程


内容：

1. APK中的so结构
2. ELF文件格式
3. Android linker加载流程
4. System.loadLibrary内部流程
5. dlopen/dlsym
6. JNI_OnLoad调用时机
7. 静态注册和动态注册符号分析
8. 使用readelf分析真实so
