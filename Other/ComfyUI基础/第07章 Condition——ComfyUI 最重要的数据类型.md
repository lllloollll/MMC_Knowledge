---

title: 第07章 Condition——ComfyUI 最重要的数据类型  
book: AI 内容创作技术体系（程序员版）  
module: ComfyUI  
version: v1.0  
difficulty: ★★★☆☆  
estimated_time: 1.5 小时  
prerequisite:

- 第05章 Checkpoint、CLIP、MODEL、VAE
    
- 第06章 Latent 与 Diffusion
    

---

# 第07章 Condition——ComfyUI 最重要的数据类型

## 本章学习目标

完成本章学习后，你将能够：

- 理解 Prompt 与 Conditioning 的区别。
    
- 理解 CLIP Text Encode 的真正职责。
    
- 理解 Positive Conditioning 与 Negative Conditioning 的来源。
    
- 理解为什么 KSampler 不接受普通文本。
    
- 理解 ComfyUI 为什么采用"数据类型"而不是"节点名称"进行连接。
    
- 为后续学习 ControlNet、IPAdapter 等控制类技术建立统一思维模型。
    

---

# 7.1 Prompt 不是 Condition

这是初学者最容易混淆的概念。

例如：

```text
1girl, red dress, long hair
```

这是 Prompt。

Prompt 的数据类型是：

```text
STRING
```

它只是人类能够阅读的自然语言。

AI 并不能直接理解这段文字。

---

# 7.2 CLIP Text Encode 的真正职责

CLIP Text Encode 并不是：

- 翻译器
    
- 解码器
    
- 关键字提取器
    

它真正完成的是：

> **把自然语言编码成模型能够理解的数据。**

流程如下：

```text
Prompt（String）

↓

CLIP Text Encode

↓

Conditioning
```

因此：

CLIP Text Encode 的职责可以抽象为：

```kotlin
fun encode(prompt: String): Conditioning
```

输入：

```text
STRING
```

输出：

```text
CONDITIONING
```

---

# 7.3 什么是 Conditioning？

Conditioning 可以理解为：

> **模型生成图片时所依据的条件数据。**

请注意：

Conditioning 不是：

- 字符串
    
- 图片
    
- Latent
    

它是一种模型内部能够理解的高维数据。

它通常由：

- CLIP
    
- ControlNet
    
- 其他条件模块
    

生成。

KSampler 能够直接处理的是：

Conditioning。

而不是 Prompt。

---

# 7.4 Positive 与 Negative 的真正关系

默认 Workflow 中：

实际上存在两份 Prompt。

第一份：

```text
Positive Prompt

1girl
red dress
long hair
```

第二份：

```text
Negative Prompt

low quality
blurry
bad anatomy
```

它们分别经过：

CLIP Text Encode。

得到：

```text
Positive Prompt
        │
        ▼
CLIP Text Encode
        │
        ▼
Positive Conditioning
```

以及：

```text
Negative Prompt
        │
        ▼
CLIP Text Encode
        │
        ▼
Negative Conditioning
```

请注意：

它们彼此完全独立。

CLIP Text Encode 在编码 Positive Prompt 时：

不知道 Negative Prompt 的内容。

同样：

编码 Negative Prompt 时：

也不知道 Positive Prompt 的内容。

因此：

Positive Conditioning 并不是：

> 去掉 Negative 后得到的数据。

它只是：

Positive Prompt 的编码结果。

Negative Conditioning 同理。

---

# 7.5 KSampler 为什么不能直接接受 Prompt？

KSampler 的输入包括：

- MODEL
    
- Latent
    
- Positive Conditioning
    
- Negative Conditioning
    

它没有：

```text
STRING
```

类型输入接口。

因此：

下面这种连接：

```text
Prompt

↓

KSampler
```

在 ComfyUI 中是不允许的。

原因有两个：

第一：

数据类型不匹配。

Prompt：

属于：

```text
STRING
```

KSampler：

要求：

```text
CONDITIONING
```

第二：

Prompt 只是自然语言。

AI 无法直接利用自然语言完成扩散计算。

必须经过：

CLIP Text Encode。

转换成：

Conditioning。

---

# 7.6 KSampler 为什么只关心 Conditioning？

KSampler 的设计思想非常重要。

它并不关心：

Conditioning 来自哪里。

例如：

未来：

ControlNet：

可以提供：

Condition。

IPAdapter：

也可以参与生成条件。

Region Prompt：

同样可以影响生成。

对于 KSampler 来说：

它只关心：

> 输入是不是合法的 Conditioning。

而不关心：

是谁生成的。

这是典型的软件工程思想：

> **面向数据类型，而不是面向具体实现。**

---

# 7.7 ComfyUI 的节点连接思想

很多初学者会认为：

节点之间因为：

功能相同。

所以能够连接。

实际上。

ComfyUI 判断是否能够连接：

依据的是：

> **数据类型（Data Type）。**

例如：

CLIP Text Encode：

输出：

```text
CONDITIONING
```

因此：

可以连接：

```text
KSampler Positive
```

或者：

```text
KSampler Negative
```

如果某个节点输出：

```text
IMAGE
```

那么：

无论节点名称叫什么。

都无法连接到：

```text
Positive Conditioning
```

因此：

理解数据类型。

远比记住节点名称更加重要。

---

# 7.8 程序员视角

如果使用 Android 开发进行类比：

```kotlin
String
    ↓
Parser
    ↓
Object
```

类似于：

```kotlin
Prompt
    ↓
CLIP Text Encode
    ↓
Conditioning
```

KSampler 更像：

```kotlin
fun sample(
    model: Model,
    latent: Latent,
    positive: Conditioning,
    negative: Conditioning
)
```

它要求：

参数类型必须正确。

至于：

Conditioning：

是由哪个模块产生的。

并不重要。

---

# 7.9 常见误区

## 误区一

Prompt 就是 Conditioning。

错误。

Prompt 是：

```text
STRING
```

Conditioning 是：

模型内部数据。

两者不是同一种数据类型。

---

## 误区二

Positive Conditioning 是从 Prompt 中去掉 Negative 得到的。

错误。

Positive Prompt 与 Negative Prompt：

分别独立编码。

互不影响。

---

## 误区三

CLIP Text Encode 分为：

Positive 节点。

Negative 节点。

错误。

实际上：

官方 Workflow 使用的是：

同一个节点。

只是：

输入内容不同。

输出连接位置不同。

因此：

"正向"与"负向"并不是节点属性。

而是：

KSampler 输入端口赋予它们的职责。

---

# 本章总结

本章建立了 ComfyUI 最重要的数据类型之一：

Conditioning。

需要牢记：

- Prompt 是 String。
    
- Conditioning 是模型能够理解的条件数据。
    
- CLIP Text Encode 的职责是把 Prompt 编码为 Conditioning。
    
- Positive Prompt 与 Negative Prompt 分别生成各自的 Conditioning。
    
- KSampler 只接受 Conditioning，不接受 String。
    
- ComfyUI 节点连接依据的是数据类型，而不是节点名称。
    

理解了 Conditioning，后续学习：

- ControlNet
    
- IPAdapter
    
- Regional Prompt
    
- AI 漫剧角色一致性
    

都会变得非常自然。

---

# 本章练习

请独立回答以下问题：

1. Prompt 与 Conditioning 有什么区别？
    
2. 为什么 KSampler 不接受 String？
    
3. Positive Conditioning 是如何产生的？
    
4. Negative Conditioning 是如何产生的？
    
5. 为什么 CLIP Text Encode 并不知道自己是在处理 Positive 还是 Negative？
    
6. 为什么 ComfyUI 更关注数据类型，而不是节点名称？
    
7. 如果你设计一个新的 Prompt 节点，希望能够连接到 KSampler，它最终应该输出什么数据类型？为什么？
    

完成以上问题，说明你已经掌握了本章内容，可以进入下一章《为什么 Prompt 不够——ControlNet 的诞生》。