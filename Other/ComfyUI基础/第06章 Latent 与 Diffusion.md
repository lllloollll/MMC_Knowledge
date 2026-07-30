---

title: 第06章 Latent 与 Diffusion  
book: AI 内容创作技术体系（程序员版）  
module: ComfyUI  
version: v1.0  
difficulty: ★★★☆☆  
estimated_time: 2 小时  
prerequisite:

- 第03章 Workflow 工作流
    
- 第05章 Checkpoint、CLIP、MODEL、VAE
    

---

# 第06章 Latent 与 Diffusion

## 本章学习目标

完成本章学习后，你将能够：

- 理解为什么 Stable Diffusion 不直接处理图片，而是处理 Latent。
    
- 理解 Empty Latent Image 的真正作用。
    
- 理解 VAE Encode 与 VAE Decode 的职责。
    
- 理解文生图与图生图的根本区别。
    
- 理解 KSampler 的真正职责。
    
- 理解 Positive / Negative Prompt 的作用。
    
- 理解 CFG 的本质。
    
- 理解 Sampling、Sampler、Steps 的设计思想。
    

本章结束后，你将能够独立分析官方 Text-to-Image Workflow。

---

# 6.1 Stable Diffusion 为什么不直接处理图片？

很多初学者都会认为：

```text
Prompt
↓

AI

↓

Image
```

事实上，Stable Diffusion 并不是这样工作的。

真正的流程是：

```text
Prompt

↓

Latent（随机噪声）

↓

Diffusion（逐步去噪）

↓

Latent

↓

VAE Decode

↓

Image
```

整个 Diffusion 过程中，AI 从来没有直接处理图片。

它一直处理的是 **Latent**。

---

# 6.2 什么是 Latent？

Latent 可以理解为：

> **AI 内部使用的数据表达方式。**

它不是：

- 图片
    
- 缩略图
    
- 压缩图片
    
- 隐藏图片
    

它是一组高维数据。

AI 能够理解 Latent。

而人类只能理解 Image。

因此：

```text
Image
↓

VAE Encode

↓

Latent

↓

Diffusion

↓

Latent

↓

VAE Decode

↓

Image
```

图片只存在于 Workflow 的开始和结束。

整个生成过程都发生在 Latent 空间。

---

# 6.3 为什么需要 Latent？

假设直接处理一张：

1024 × 1024

RGB 图片。

需要处理的数据约为：

```text
1024 × 1024 × 3
≈ 314 万个数值
```

如果每一步扩散都处理如此庞大的数据，计算成本会非常高。

因此 Stable Diffusion 采用 VAE：

先将图片压缩到 Latent Space，再进行扩散。

这种设计极大降低了计算量。

---

# 6.4 Empty Latent Image 的作用

很多教程会说：

> Empty Latent Image 创建一张空白图片。

这是错误的。

准确来说：

> **Empty Latent Image 创建的是一块指定尺寸的 Latent 初始状态。**

它并没有创建图片。

它只是为 AI 创建了一块可以工作的 Latent。

因此：

Width 与 Height 表示的是：

> **未来生成图片对应的 Latent 尺寸。**

而不是当前图片尺寸。

Batch Size 表示：

一次创建多少份 Latent。

而不是一次保存多少张图片。

---

# 6.5 文生图与图生图的区别

## 文生图（Text to Image）

由于没有输入图片，因此需要：

```text
Empty Latent Image

↓

Latent
```

作为扩散的起点。

完整流程：

```text
Prompt

↓

CLIP

↓

Conditioning

↓

KSampler

↓

VAE Decode

↓

Image
```

---

## 图生图（Image to Image）

图生图已经拥有图片。

因此：

```text
Image

↓

VAE Encode

↓

Latent

↓

KSampler

↓

VAE Decode

↓

Image
```

图生图不会使用 Empty Latent Image。

而是直接把已有图片转换成 Latent。

---

# 6.6 KSampler 到底做了什么？

KSampler 是整个 Workflow 的核心。

它真正负责：

> **根据 Prompt，不断修改 Latent。**

它并不是：

- 绘图工具
    
- 图片编辑器
    

它只是：

不断预测：

> 下一步应该如何修改 Latent。

直到最终收敛。

---

## KSampler 的输入

KSampler 主要接收四类数据：

### 1. MODEL

来源：

Checkpoint Loader。

作用：

预测噪声。

---

### 2. Latent

来源：

- Empty Latent Image
    
- VAE Encode
    

作用：

当前需要处理的数据。

KSampler 不关心：

Latent 是如何产生的。

它只要求：

> 输入必须是合法的 Latent。

---

### 3. Conditioning

来源：

- CLIP Text Encode
    
- ControlNet（后续章节）
    
- IPAdapter（后续章节）
    

作用：

告诉模型应该生成什么。

Conditioning 是一种高维向量。

不是字符串。

更不是关键字。

---

### 4. Sampling 参数

包括：

- Seed
    
- CFG
    
- Steps
    
- Sampler
    
- Scheduler
    

共同决定扩散过程。

---

# 6.7 Positive 与 Negative Prompt

Positive Prompt：

表示：

> 希望生成什么。

例如：

```text
1girl
long hair
red dress
```

Negative Prompt：

表示：

> 希望远离什么。

例如：

```text
low quality
blurry
bad anatomy
extra fingers
```

Negative Prompt 并不是可有可无。

它和 Positive Prompt 共同决定优化方向。

可以理解为：

- Positive：目标。
    
- Negative：约束。
    

---

# 6.8 CFG（Classifier-Free Guidance）

CFG 决定：

> **Prompt 对模型决策的影响权重。**

不是：

Prompt 是否生效。

而是：

Prompt 影响模型的程度。

例如：

CFG 较低：

模型更多依赖自身经验。

CFG 较高：

模型更加严格执行 Prompt。

但是：

CFG 并不是越高越好。

过高的 CFG 容易导致：

- 画面过拟合
    
- 颜色不自然
    
- 面部崩坏
    
- 光影失真
    

因此：

大多数模型推荐：

```text
CFG ≈ 6～8
```

这只是经验值。

并不是固定规则。

---

# 6.9 Sampling 与 Sampler

很多教程把 Sampling 翻译成：

采样。

实际上，更容易理解为：

> **每一步如何选择下一步的方向。**

Model 负责预测：

> 哪个方向更合理。

Sampler 负责决定：

> 最终采用哪一种策略。

因此：

不同 Sampler：

例如：

- Euler
    
- DPM++
    
- UniPC
    

并不是不同模型。

而是：

不同的扩散策略。

---

# 6.10 为什么不同 Sampler 会生成不同图片？

假设：

所有参数都完全一致：

- Prompt
    
- Seed
    
- CFG
    
- Steps
    
- LoRA
    

唯一不同：

Sampler。

由于：

每一种 Sampler：

走向最终结果的策略不同。

因此：

最终图片通常：

> 略有区别。

而不是完全一致。

---

# 6.11 整个 Workflow 的数据流

官方文生图 Workflow：

```text
                Checkpoint Loader
                /        |        \
               /         |         \
          MODEL       CLIP        VAE
             |           |          |
             |     CLIP Text Encode |
             |       /         \     |
             | Positive   Negative   |
             |         \      /      |
Empty Latent Image ---> KSampler ----|
                    |
                 Latent
                    |
              VAE Decode
                    |
                 Image
                    |
               Save Image
```

请注意：

真正流动的是：

> **数据。**

而不是节点。

理解数据流，比记住节点名称更重要。

---

# 本章总结

本章建立了 Stable Diffusion 最核心的思维模型：

- AI 处理的是 Latent，而不是图片。
    
- Empty Latent Image 创建的是 Latent 初始状态。
    
- VAE Encode 负责进入 AI 世界。
    
- VAE Decode 负责返回人类世界。
    
- KSampler 是整个 Workflow 的核心。
    
- Positive 提供目标，Negative 提供约束。
    
- CFG 控制 Prompt 的影响程度。
    
- Sampler 决定每一步如何走向最终结果。
    

当你理解这些内容后，你已经不再是"会使用 ComfyUI"。

而是开始理解：

> **为什么 ComfyUI 会这样设计。**

---

# 程序员视角

如果把 Stable Diffusion 看成一个软件系统：

|Stable Diffusion|软件工程类比|
|---|---|
|Checkpoint|程序运行时环境|
|MODEL|核心算法|
|CLIP|文本编码器|
|VAE|编码/解码器|
|Latent|内部数据结构|
|KSampler|核心业务逻辑|
|Conditioning|输入参数|
|Image|最终输出结果|

理解 Workflow，本质上就是理解：

> **数据如何在系统中流动。**

---

# 本章练习

请独立回答以下问题：

1. 为什么 Stable Diffusion 不直接处理图片？
    
2. Empty Latent Image 为什么不是创建一张图片？
    
3. 为什么图生图不需要 Empty Latent Image？
    
4. KSampler 为什么不关心 Latent 的来源？
    
5. Positive Prompt 与 Negative Prompt 分别承担什么职责？
    
6. CFG 为什么不是越高越好？
    
7. 为什么更换 Sampler 后，图片会发生变化？
    

如果能够完整回答以上问题，说明你已经掌握本章内容，可以进入下一章《ControlNet》。