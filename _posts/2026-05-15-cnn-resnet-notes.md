---
title: "5.2（CNN+ResNet）"
date: 2026-05-15 15:09:09 +08:00
permalink: /posts/cnn-resnet-notes/
tags:
  - deep-learning
  - cnn
  - resnet
  - notes
source_note: "local source file, not uploaded"
---

- 由于针对数据结构的不同，深度神经网络有了许多分支，如卷积神经网络 CNN（处理对象：网格状数据，利用 kernel 提取空间特征和跨通道融合），大模型主干 Transformer 架构（处理对象：序列化数据，利用 self-attention 处理文本序列），但是由于动力核心均为“用梯度下降来更新参数”，所以针对神经网络的方法是互通的，比如 Adam。

- ResNet 相关：

  - 由恒等映射引入，反向传播的规则使得局部网络无法独立于全局去做到纯粹的恒等映射。所以我们增加一条 shortcut，尝试将“恒等映射”（单位矩阵）问题转化为“向零拟合”问题。神经网络输出的向零拟合由于 Weight Decay、L2 正则化的存在，会很容易。

  - ResNet 使用的是 Residual Block 去组合成 Net，比如 BasicBlock 和 Bottleneck。浅层 block 的 shortcut 期望还是要加上一些东西的，还是要往 target 去前进的。而深层的很大概率会多余，为了使 loss 不降，就会期望全走 shortcut。

    反向传播的时候，监督信号既走正常层也走 shortcut，很符合梯度分发的规则。残差的出现对梯度加了一个 1，减轻了梯度消失的情况。

  - BasicBlock 只使用了两个 $3\times 3$ 卷积层，我们假设 in_channels 是 $256$，两层参数量均为 $3\times 3\times 256\times 256$（一个 filter 里面有 $256$ 个 $3\times 3$ kernel 并且产生一个 out_channel，一层有 out_channel=256 个 filter）约为 $59$ 万，共计 $118$ 万。

    Bottleneck 的三层结构先使用 $1\times 1$ kernel 降维，再用 $3\times 3$ kernel 处理压缩后的核心特征，最后用 $1\times 1$ kernel 升维，共计约 $7$ 万。

    两者相比，前者优势在于特征提取无损，但是参数量过多，在浅层网络如 ResNet-18/34 使用性价比较高；而后者压缩了参数量，使得信息微损，但是能支持深层网络如 ResNet-50/101/152 提取更优秀的特征。

  - 细节一点，ResNet 一般是由 channel 相同的 Residual blocks 组成的 stages 构成，在 stage 内 shortcut 无需多操心直接引入加法即可，但在 channel 改变时，一般长宽 size 也变了，stage 中的第一个 block 内的 shortcut 就要承担起 tensor 的 unification，这里通过在 shortcut 上加一个 $1\times 1$ kernel 的卷积层去匹配 channel，并且还需要对该卷积层的 stride 进行调整以适配 size。其实就是起到 downsample 下采样层的作用，一旦 block 发现输入输出维度不一样，就会激活 downsample 去修饰 shortcut。

- 回望 CNN，为什么标准卷积要让一个 filter 对应一个 out_channel 并在内部强行进行通道相加呢？为什么不直接让单个 kernel 独立对应一个 out_channel 跑到底呢？由于最终任务势必只需要一个综合结果，考虑那种晚期融合的设想：通道越独立，特征越鲜明，推迟融合也不会像正常的每层融合那样有出现“多对一”的可能性，可以防止不同维度的原始特征在早期被忽略或抹去，并且最重要的一点是省参数。但是每层融合的优势在于其全局性，每层都可以得出较为全局的结论，有利于在全局特征上进行深化，当然自由度高对应了算力需求大的缺点。

  事实上，2017 年 Google 提出的 MobileNet 就基于节省资源的想法使用了独立通道的卷积。而对于每层融合可能出现的多对一映射，现在采取的方法是使用数量庞大的输出通道，通过多组权重不同的 filter 对同一输入进行高维展开，多个 filter 学习出的内容甚至可以做到类似哈希的效果，利用多种逻辑组合去保留特征。

  2017 年 Kaiming 提出的 ResNeXt 也是该思考的产出，引入 Group  Convolution 去抵抗逐层融合带来的过拟合和算力问题，强迫网络先在独立通道上专心提取特征，再融合，起到类似于正则化的效果。

- ![1048.png](/images/notes/cnn-resnet-notes/1048-1777730297349-3.png)

  2015 年 Kaiming 训练 ResNet 的时候，使用的 optimizer 还是 SGD+Momentum，并且使用 LR Scheduler 去按次数调小了学习率，所以会出现断崖的情况，此时 Adam 还没普及。现在似乎使用了更为强大的 AdamW + Warmup + Cosine Decay，带 Weight Decay 的 Adam，辅以起步的 Warmup，再加上 Decay 时依照 cos 的速率下降。
