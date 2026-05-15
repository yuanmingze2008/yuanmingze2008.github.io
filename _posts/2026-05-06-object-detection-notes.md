---
title: "目标检测"
date: 2026-05-06 17:14:40 +08:00
permalink: /posts/object-detection-notes/
tags:
  - deep-learning
  - computer-vision
  - object-detection
  - notes
source_note: "local source file, not uploaded"
---

在目标检测的数据标注、图像处理乃至整个计算机图形学领域，一般都将原点设在左上角，向右引出 $x$ 轴正方向，向下引出 $y$ 轴正方向。不用笛卡尔坐标系的原因是为了契合矩阵形式的输入和计算，早期的阴极射线管显示器扫描轨迹就是从左上角开始从左至右从上至下，并且在计算机系统的角度看内存连续性好。

CNN 的独特之处就在于利用 kernel 做到了滑动窗口式提取局部特征从而学习全局信息。CNN 不只能做分类，也可以做回归，比如目标检测中的位置信息就是连续的回归问题。

### 目标检测

对于图像分类，我们要拟合的函数是固定的多对一映射：$f: \mathbb{R}^{H \times W \times 3} \rightarrow \mathbb{R}^{K}$

（输入是一张高 $H$、宽 $W$、3 通道的图像张量，输出是 $K$ 个类别的概率向量）

但目标检测的真实输出是一个**集合**，且集合的大小 $N$（画面中的物体数量）是未知的、动态的：$f: \mathbb{R}^{H \times W \times 3} \rightarrow \{(c_i, x_i, y_i, w_i, h_i)\}_{i=1}^N$

这里的 $c_i$ 是离散的分类变量，$(x_i, y_i, w_i, h_i)$ 是连续的回归变量，不过种类数是一定的。

关键矛盾在于**传统 CNN 的网络以及输入输出静态固定化，无法输出未定目标数目的信息，无法动态化。**

- Sliding Window

  滑动窗口非常暴力地枚举窗口大小、窗口位置，然后跑窗口个数次CNN。

  我们以分辨率为 $1920\times 1080$ 的图片为例，步长设定为较细的 $S=8$，窗口尺寸设定为 $5$ 种，则在一张图片上，需要抠出的窗口数量约为：$N=5\times\bigg(\frac{1920}{8}\bigg)\times\bigg(\frac{1080}{8}\bigg)=162000$。也就是，我们目前要做 $162000$ 次 CNN，这样有大量的重叠区域特征被重复计算了许多。

- R-CNN (2014)

  利用传统 CV 的 Selective Search 算法可以通过颜色、纹理、边缘，把图像里“看着像个完整物体”的色块圈出来。

  1. 先用 Selective Search 算法在原图上圈出约 $2000$ 个“候选框（Region Proposals）”。

     > 在 2014 年的 CPU 算力下，处理一张普通分辨率的图片（如 500×400），Selective Search 跑一遍大约需要 **2 到 10 秒**。

  2. 把这 2000 个大小不一的框抠出来，warp 成相同的 size（严重破坏物体的长宽比和原始特征）。

  3. 送进 CNN 提取特征，再接 SVM 分类器和边界框回归器。

     > 由于 CNN 参数太多而数据集太小，直接使用 CNN 容易使网络过拟合，只好使用对小样本和噪音更鲁棒的传统分类器 SVM 做最终判断。但这样出现的问题是需要处理好 CNN 和 SVM 的数据传输。
     >
     > 边界框回归器起到了微调边界框的作用，基于训练出来的边界框为初始值去打补丁。

- Fast R-CNN (2015)

  卷积操作具有空间平移不变性，我们只对原图做一次提取特征的 CNN（去除 head 保留 backbone），然后考虑输出的 feature map。

  引入 Region of Interest Pooling 技术：

  1. 整图做一次前向传播 CNN 得到 feature map；
  2. 将 SS 找出的 2000 个候选框坐标对应到 feature map 上（用 stride 换算）；
  3. 把 feature map 上的候选框抠出来池化成固定大小，再做分类（具体地：对于一个框展成的图片，展平成一维后做一层全连接控制输出维度，然后使用 Twin Head，一个 classification head 输出维度为种类数加一，一个 bounding box regression head 输出维度为四乘种类数的，这里回归头依然使用四乘种类数而非一个四维向量的原因是提供自由度，支持在不同种类下的框选拟合，使训练结果独立化，如果只输出一个四维向量，整个图都会对其影响）

  最后的 twin head 代替了传统 SVM 和微调边界框的作用，但优点是，R-CNN 中，CNN 和 SVM 是解耦的，SVM 无法通过训练反向指导 CNN 的卷积核更新。而在 Fast R-CNN 中，由于都是网络的组合，我们可以引入 Multi-task Loss：$L=L_{cls}(p,u)+\lambda[u\ge1]L_{reg}(t^u,v)$，即**总误差 = 分类没分对的误差 + 框画的不够精准的误差**。

  > Fast R-CNN 联系到了深度学习本质之处，将所有的参数通过损失函数连接起来，在其指挥下向同一个目标协同进化。CNN 学提取特征，不再是为了“盲目地提取”，而是为了“能让后面的分类头和回归头得分最高”去定制化地提取特征。

  至此，Fast R-CNN 把特征提取、分类、回归全部统一到了一个神经网络里，都在 GPU 上用张量计算完成。 唯一的瓶颈在于：**它还依赖 CPU 上的 Selective Search 算法来提供 2000 个初始框。**既然我们已经有了一张富含语义信息的 Feature Map，为什么不让神经网络自己去找框？

- Faster R-CNN (2015)

  Faster R-CNN 引入 Region Proposal Network (RPN) 接在 feature map 后面。

  假设 backbone 输出 $F\in \mathbb{R}^{C\times H_f\times W_f}$，如果每个位置预设 $k$ 个 anchors，则总 anchor 数为 $N_a=H_fW_fk$。

  例如 $50\times50$ 的 feature map，每个位置设置 $9$ 个 anchors，则共有 $22500$ 个 anchors。需要注意，anchor 不是网络学出来的框，而是人工预设的固定初始模板，坐标通常定义在原图坐标系下。

  RPN 先通过一个 $3\times3$ 卷积融合局部上下文，再分成两个 head：objectness head 输出每个 anchor 是 foreground/background 的分数，bbox regression head 输出每个 anchor 的偏移量 $(t_x,t_y,t_w,t_h)$。所以 RPN 不是凭空生成框，而是对固定 anchors 做两件事：判断它是否值得变成 proposal，以及预测它应该如何平移和缩放。

  RPN 的训练依赖 Anchor-GT 匹配。先计算所有 anchors 与 Ground Truth boxes 的 IoU，得到 $U=IoU(A,G)\in \mathbb{R}^{N_a\times M}$。

  再根据 IoU 给 anchors 打上 positive、negative 或 ignore 标签。由于背景 anchors 数量远多于前景 anchors，不能直接用全部 anchors 计算 loss，否则网络容易学成“全部预测背景”。因此训练时通常采样一小批 anchors，如 $256$ 个，并尽量控制正负比例。RPN 的 loss 为 $L_{rpn}=L_{rpn-cls}+L_{rpn-reg}$，其中分类 loss 对正负样本都计算，回归 loss 只对正样本计算。

  但采样 $256$ 个 anchors 只是为了计算 RPN loss，并不意味着 RPN 只输出 $256$ 个框。RPN 前向传播仍然对所有 anchors 输出 objectness 和 bbox deltas。生成 proposals 时，需要把 bbox deltas 解码到 anchors 上：
  $$
  B_i=decode(A_i,T_i)
  $$
  再经过 clip、remove small boxes、按 objectness score 排序、NMS 和 top-k，得到较少的候选框 $R\in \mathbb{R}^{N_p\times4}$，这些才是真正送入 RoI Pooling / RoI Align 的 proposals。

  之后的第二阶段基本继承 Fast R-CNN：根据 proposals 从共享 feature map 上截取对应区域，池化成固定大小，再送入 RoI Head 做具体类别分类和二次边界框回归。完整损失为
  $$
  L=L_{rpn-cls}+L_{rpn-reg}+L_{roi-cls}+L_{roi-reg}
  $$

  从本质上看，Faster R-CNN 是一种“两阶段的工程分工”：RPN 便宜而密集地处理大量 anchors，负责高召回地粗找哪里可能有物体；RoI Head 昂贵但稀疏地处理少量 proposals，负责精细判断具体类别并把框再次修准。