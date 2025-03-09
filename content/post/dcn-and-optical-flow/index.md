---
title: "可变形卷积与光流"
date: 2022-02-20 17:04:37+0800
tags:
  - 视频编码
categories:
  - 有情笔记
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 可变形卷积代码篇

一个调用 `from mmcv.ops import ModulatedDeformConv2d` 的例子：

```python
def forward(self, input):
        x = self.offset_mask_conv(input)
        o1, o2, mask = torch.chunk(x, 3, dim=1)
        offset = torch.cat((o1, o2), dim=1)
        mask = torch.sigmoid(mask)
        output = self.dcnv2(input, offset, mask)
        return output
```

总而言之 `offset` 的 size 就是 `2*kernel[0]*kernel[1]` ，想一想，原来我们求偏移的时候，会把 `B*H*W*C` 的图像送入普通卷积得到 `B*H*W*2C` 得到偏移，也就是每个通道每个位置点都有 x 和 y 两个方向的偏移量。

对 DCN 来说，每个通道都做一样的处理，也就是只需要对每个位置点存卷积核每个点的 x 和 y 的偏移，所以就是 `B*H*W*(2*kernel_size)` 。

关于 mask:：置信 mask，并非必需，不作展开了

所以 offset 和 mask 一起就是 `B*H*W*(3*kernel_size)`

关于 deform_groups：本来是所有通道公用，也可以改成划成几组，组数就是 deform_groups，那这样就是 `B*H*W*(group_num*3*kernel_size)`

## 可变形卷积与光流

端到端视频压缩模型里，运动估计运动补偿环节，常用到光流和可变形卷积。本质上都是预测偏移，只不过 DCN 可以利用多个偏移，这样在预测难度较大的位置就有多个 offset 互作补充，所以比只用单一的光流更有优势一些。

推荐一下这篇文章：[Understanding Deformable Alignment in Video Super-Resolution](https://ckkelvinchan.github.io/projects/DCN/)，讲得比较透彻了。

### 可变形卷积涉及多少个不同的 offset map

1. 首先，与 kernel size 有关

   ![image-20230225152456053](https://raw.githubusercontent.com/bygonexf/Blog-Images/master/2023/02/upgit_20230225_1677309897.png)

   假设特征图大小为 `W * H`，每个点的感受野都对应了 `kernel_size * kernel_size` 个偏移

2. 其次，与 group num 有关

   ![image-20230225152648179](https://raw.githubusercontent.com/bygonexf/Blog-Images/master/2023/02/upgit_20230225_1677310198.png)

   每 `C(通道数) / group_num` 个通道共用一套 offset

   比如假设特征有 8 个通道，group_num 为 4,则 每 2 个通道共用一套偏移参数

综合来说，一共会有 `(kernel_size * kernel_size) * group_num` 个 `W * H` 大小的偏移图。

### DCN 对齐和光流对齐的本质差异

![image-20230225152953943](https://raw.githubusercontent.com/bygonexf/Blog-Images/master/2023/02/upgit_20230225_1677310203.png)

`kernel size 为 n * n 的DCN = n^2 个空间warping + 1 * 1 * n^2 的3D卷积`

也就是说，如果 n=1，group_num=1，DCN对齐基本就等同于光流对齐，DCN 和光流的本质区别就在于 offset diversity。
