---
title: H.265/HEVC 编码结构 笔记
date: 2021-03-31 11:34:13+0800
tags:
  - 视频编码
categories:
  - 无情笔记
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 名词一览

* GOP (Group of Pictures) - 图像组
* IDR (Instantaneous Decoding Refresh) - 即时解码刷新
* Slice - 片
* SS (Slice Segment) - 片段
* CTU (Coding Tree Unit) - 树形结构单元
* CTB (Coding Tree Block) - 树形编码块
* CU (Coding Unit) - 编码单元
* SPS (Sequence Parameter Set) - 序列参数集
* PPS (Picture Parameter Set) - 图像参数集
* CVS (Coded Video Sequence) - 一个 GOP 编码后生成的压缩数据
* VPS (Video Parameter Set) - 视频参数集

## 编码结构概述

###  编码结构

**视频序列**分隔为若干个图像组 (**GOP**)。

存在两种 GOP 类型：

* 封闭式 GOP

  每一个 GOP 以 IDR 图像开始，各个 GOP 之间独立编解码。

* 开放式 GOP

  第一个 GOP 中的第一个帧内编码图像为 IDR 图像，后续 GOP 中的第一个帧内编码图像为 non-IDR 图像。后面 GOP 中的帧间编码图像可以使用前一个 GOP 的已编码图像作为参考图像。

![](https://raw.githubusercontent.com/bygonexf/Blog-Images/master/20210331120920.png)

每个 GOP 又被分为多个片 (**Slice**)，片与片之间独立编解码。主要目的之一是在数据丢失的情况下进行重新同步。

每个片由一个或多个片段 (**SS**, Slice Segment) 组成。

树形结构单元 (CTU) 类似传统的宏块。每个 CTU 包括一个亮度 CTB 和两个色差 CTB。

一个 SS 在编码时，先被分割为相同大小的 **CTU**，每一个 CTU 按照四叉树分割方式被划分为不同类型的 **CU**。

![](https://raw.githubusercontent.com/bygonexf/Blog-Images/master/20210331120838.png)

以上即为编码时的分层处理架构。

### 码流结构

码流结构上，H.265/HEVC 压缩数据采用了类似于 H.264/AVC 的分层结构。

将属于 GOP 层、Slice 层中共用的大部分语法元素游离出来，组成 SPS 和 PPS。

**SPS** 中包含了一个 CVS 中所有图像共用的信息。SPS 中大致包括解码相关信息，如档次级别、分辨率、某档次中编码工具开关标识和涉及的参数、时域可分级信息等。

**PPS** 中包含一副图像所用的公共参数，大致包括初始图像控制信息，如初始量化参数、分块信息等。一副图像中所有 SS 引用同一个 PPS。

此外，为了兼容在其他应用上的扩展，H.265/HEVC 的语法架构中增加了 **VPS**，其内容大致包括多个子层共享的语法元素，其他不属于 SPS 的特定信息等。

对于一个 SS，通过引用它所使用的 PPS，该 PPS 又引用对应的 SPS，该 SPS 又引用对应的 VPS，最终得到 SS 的公用信息。

![](https://raw.githubusercontent.com/bygonexf/Blog-Images/master/20210331120942.png)

## 片段层

一副图像可以被分割为一个或多个 Slice，每个 Slice 的压缩数据都是独立的，Slice 头信息无法通过前一个 Slice 的头信息推断得到。这就要求 Slice 不能跨过它的边界来进行帧内或帧间预测。

根据编码类型不同，Slice 可分为以下几部分：

1. I Slice

   该 Slice 中所有 CU 的编码过程都使用帧内预测

2. P Slice

   在 I Slice 的基础上，该 Slice 中的 CU 还可以使用帧间预测，每个 PB（预测块）使用至多一个运动补偿预测信息。P Slice 只使用图像参考列表 list 0。

3. B Slice

   在 P Slice 的基础上，该 Slice 中的 CU也可以使用帧间预测，每个 PB（预测块）可以使用至多两个运动补偿预测信息。B Slice 可以使用图像参考列表 list 0 和 list 1。

一个独立的 Slice 可以被进一步划分为若干个 SS，包括一个独立 SS 和若干个依赖 SS，并且以独立 SS 作为该 Slice 的开始。

一个 SS 包含整数个 CTU（至少一个），并且这些 CTU 分布在同一个 NAL 单元中。SS 可以作为一个分组来传送视频编码数据。

## Tile 单元

### Tile 单元描述

一副图像不仅可以划分为若干个 Slice，也可以划分为若干个 Tile。即从水平和垂直方向将一个图像分割为若干个矩形区域，一个矩形区域就是一个 Tile。每个 Tile 包含整数个 CTU。

Tile 提供比 CTB 更大程度的并行，在使用时无须进行复杂的线程同步。

在同一幅图像中，可以存在某些 Slice 中包含多个 Tile 和某些 Tile 包含多个 Slice 的情况。

### Slice 和 Tile

Tile 形装基本上为矩形，Slice 为条带状。

Slice 由一系列 SS 组成，一个 SS 由一系列 CTU 组成。Tile 则直接由一系列 CTU 组成。

每个 Slice/SS 和 Tile 至少要满足以下两个条件之一：

1. 一个 Slice/SS 中的所有 CTU 属于同一个 Tile
2. 一个 Tile 中的所有 CTU 属于同一个 Slice/SS

## 树形编码块

传统的视频编码基于宏块实现。考虑到高清视频 / 超高清视频的自身特性，H.265/HEVC 标准中引入了树形编码单元 CTU，其尺寸由编码器指定，且可大于宏块尺寸。

同一位置处的一个亮度 CTB 和两个色度 CTB，再加上相应的语法元素形成一个 CTU。在高分辨率视频的编码中，使用较大的 CTB 可以获得更好的压缩性能。

H.265/HEVC 为图像划分定义了一套全新的语法单元，包括编码单元 (CU)、预测单元 (PU)、变换单元  (TU)。

* CU 是进行预测、变换、量化和熵处理等处理的基本单元
* PU 是进行帧内/帧间预测的基本单元
* TU 是进行变换和量化的基本单元

### 编码单元

大尺寸图像的一个特点是平缓区域的面积更大，用较大的块编码能极大提升编码效率。在 H.264/AVC 中，编码块的大小是固定的。而在 H.265/HEVC 中，一个 CTB 可以直接作为一个 CB，也可以进一步以四叉树的形式划分为多个小的 CB。大的 CB 可以使得平缓区域的编码效率提高，小 CB 能很好地处理图像局部的细节。

编码单元是否继续划分取决于分割标志位 Split Flag。

### 预测单元

预测单元规定了编码单元的所有预测模式。帧内预测的方向、帧间预测的分割方式、运动矢量预测、帧间预测参考图像索引号都属于预测单元的范畴。

![](https://raw.githubusercontent.com/bygonexf/Blog-Images/master/20210331153821.png)

### 变换单元

TU 的大小依赖于 CU 模式，在一个 CU 内，允许 TU 跨越多个 PU，以四叉树的形式递归划分。对于一个 2N × 2N 的 CU，有一个标志位决定其是否划分为 4 个 N × N 的 TU，是否可以进一步划分由 SPS 中的 TU 最大划分深度决定。

## 档次、层和级别

在 H.264 中就有对档次 (Profile) 和级别 (Level) 的划分，它们规定了比特流必须遵守的一些限制要求。而 H.265/HEVC 中在此基础上又新定义了一个概念：层 (Tile)。

* Profile 主要规定编码器可采用哪些编码工具或算法
* Level 是指根据解码端的负载和存储空间情况对关键参数加以限制
* 有些 Level 定义了两个 Tile: 主层 (Main Tile) 和高层 (High Tile)，主层用于大多数应用，高层用于那些最苛刻的应用

满足某一 Level 和 Tile 的解码器应当可以解码当前以及比当前更低的 Level 和 Tile 的所有码流。

满足某一 Profile 的解码器必须支持该 Profile 中的所有特性。编码器不必实现 Profile 中的所有特性，但生成的码流必须遵守标准规定。