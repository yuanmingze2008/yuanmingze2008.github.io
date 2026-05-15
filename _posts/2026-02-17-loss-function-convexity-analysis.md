---
title: "学习笔记：损失函数凸性分析 (Convexity Analysis)"
date: 2026-02-17 18:00:46 +08:00
permalink: /posts/loss-function-convexity-analysis/
tags:
  - deep-learning
  - math
  - optimization
  - notes
source_note: "local source file, not uploaded"
---

# 学习笔记：损失函数凸性分析 (Convexity Analysis)

**主题**：Linear Regression vs. Logistic Regression (MSE vs. BCE)

**核心工具**：矩阵微积分、Hessian 矩阵、半正定性 (PSD)

---

## 0. 前置数学基石 (Mathematical Foundations)

### 0.1 布局约定 (Layout Convention)

本笔记严格遵循 **分母布局**：

- 标量 $y$ 对列向量 $\mathbf{x} \in \mathbb{R}^n$ 求导，结果为列向量 $\mathbb{R}^n$。
- 列向量 $\mathbf{y} \in \mathbb{R}^m$ 对列向量 $\mathbf{x} \in \mathbb{R}^n$ 求导，结果为矩阵 $\mathbb{R}^{n \times m}$（$\frac{\partial \mathbf{y}}{\partial \mathbf{x}}$ 的第 $(i,j)$ 元素为 $\frac{\partial y_j}{\partial x_i}$）。

### 0.2 核心矩阵求导公式 (The Toolbox)

基于标量展开法 (Scalar Expansion) 与 迹-微分法 (Trace-Differential) 的结论：

1. **线性形式**：

   $$
   \frac{\partial (\mathbf{A}\mathbf{x})}{\partial \mathbf{x}} = \mathbf{A}^T, \quad \frac{\partial (\mathbf{x}^T\mathbf{A})}{\partial \mathbf{x}} = \mathbf{A}
   $$

   $$
   \frac{\partial (\mathbf{b}^T\mathbf{x})}{\partial \mathbf{x}} = \mathbf{b}
   $$
2. **二次型**：

   $$
   \frac{\partial (\mathbf{x}^T\mathbf{A}\mathbf{x})}{\partial \mathbf{x}} = (\mathbf{A} + \mathbf{A}^T)\mathbf{x}
   $$

   - 特例：若 $\mathbf{A}$ 为对称矩阵（如 Hessian），则为 $2\mathbf{A}\mathbf{x}$。
3. **微分与迹的联系** (用于高阶推导)：

   $$
   df = \text{tr}\left( \left( \frac{\partial f}{\partial \mathbf{X}} \right)^T d\mathbf{X} \right)
   $$

   - 性质：$d(\mathbf{UV}) = (d\mathbf{U})\mathbf{V} + \mathbf{U}(d\mathbf{V})$ (顺序不可换)
   - 性质：$\text{tr}(\mathbf{A}) = \text{tr}(\mathbf{A}^T)$

### 0.3 凸性判定标准 (The Ruler)

利用 Taylor 展开分析函数局部几何性质：

$$
f(\mathbf{x}_0 + \mathbf{v}) \approx f(\mathbf{x}_0) + \nabla f(\mathbf{x}_0)^T \mathbf{v} + \frac{1}{2} \mathbf{v}^T \mathbf{H}(\mathbf{x}_0) \mathbf{v}
$$

- **Hessian Matrix ($\mathbf{H}$)**: 二阶偏导数矩阵 $\mathbf{H}_{ij} = \frac{\partial^2 f}{\partial x_i \partial x_j}$ (实对称)。
- **凸函数充要条件**：$\mathbf{H}$ 半正定 (PSD, $\mathbf{H} \succeq 0$)。
- **PSD 判定**：
  1. **定义法**：$\forall \mathbf{v} \in \mathbb{R}^n, \mathbf{v} \neq \mathbf{0} \implies \mathbf{v}^T \mathbf{H} \mathbf{v} \ge 0$ (即二次型非负)。
  2. **谱定理**：$\mathbf{H}$ 的所有特征值 $\lambda_i \ge 0$。

---

## 1. 线性回归 + MSE (Linear Regression)

**结论**：损失曲面是严格凸的 (Strictly Convex)，形如椭圆抛物面。

### 1.1 定义

- 预测模型：$\hat{\mathbf{y}} = \mathbf{X}\mathbf{w}$
- 损失函数 (MSE)：$L(\mathbf{w}) = \frac{1}{2N} \|\mathbf{X}\mathbf{w} - \mathbf{y}\|_2^2$

  *(忽略常数系数 $\frac{1}{2N}$，不改变凸性)*

### 1.2 推导过程

展开目标函数 $J(\mathbf{w})$：

$$
\begin{aligned} J(\mathbf{w}) &= (\mathbf{X}\mathbf{w} - \mathbf{y})^T (\mathbf{X}\mathbf{w} - \mathbf{y}) \\ &= \mathbf{w}^T\mathbf{X}^T\mathbf{X}\mathbf{w} - \mathbf{y}^T\mathbf{X}\mathbf{w} - \mathbf{w}^T\mathbf{X}^T\mathbf{y} + \mathbf{y}^T\mathbf{y} \end{aligned}
$$

*Trick*: 标量转置等于自身 $\mathbf{y}^T\mathbf{X}\mathbf{w} = (\mathbf{y}^T\mathbf{X}\mathbf{w})^T = \mathbf{w}^T\mathbf{X}^T\mathbf{y}$，合并中间项：

$$
J(\mathbf{w}) = \mathbf{w}^T(\mathbf{X}^T\mathbf{X})\mathbf{w} - 2\mathbf{X}^T\mathbf{y}^T \mathbf{w} + \text{const}
$$

**求梯度 ($\nabla J$)**:

$$
\nabla J(\mathbf{w}) = 2\mathbf{X}^T\mathbf{X}\mathbf{w} - 2\mathbf{X}^T\mathbf{y}
$$

**求 Hessian ($\mathbf{H}$)**:

$$
\mathbf{H} = \nabla^2 J(\mathbf{w}) = \frac{\partial (2\mathbf{X}^T\mathbf{X}\mathbf{w})}{\partial \mathbf{w}} = 2\mathbf{X}^T\mathbf{X}
$$

### 1.3 凸性证明 (PSD Check)

验证 $\mathbf{H} = 2\mathbf{X}^T\mathbf{X}$ 是否半正定：

$$
\forall \mathbf{v} \neq \mathbf{0}, \quad \mathbf{v}^T \mathbf{H} \mathbf{v} = 2 \mathbf{v}^T \mathbf{X}^T \mathbf{X} \mathbf{v} = 2 (\mathbf{X}\mathbf{v})^T (\mathbf{X}\mathbf{v}) = 2 \|\mathbf{X}\mathbf{v}\|^2
$$

由范数性质，$\|\cdot\|^2 \ge 0$ 恒成立。

$\therefore$ **Hessian 恒为半正定，Linear Regression + MSE 为凸函数。**

---

## 2. 逻辑回归 + MSE (Logistic Regression)

**结论**：**非凸 (Non-Convex)**。损失曲面存在波浪状起伏和局部极小值。

### 2.1 标量推导 (Scalar Analysis)

为了直观展示非凸来源，先考察单变量情况。

- 模型：$z = wx, \quad a = \sigma(z) = \frac{1}{1+e^{-z}}$
- 性质：$a' = a(1-a)$
- 损失：$L(w) = \frac{1}{2}(a-y)^2$

**梯度推导 (Chain Rule)**:

$$
\frac{\partial L}{\partial w} = (a-y) \cdot a(1-a) \cdot x
$$

**二阶导数推导 (Product Rule)**:

$$
\begin{aligned} L''(w) &= x \cdot \frac{\partial}{\partial w} [ (a-y) \cdot (a-a^2) ] \\ &= x \cdot \left[ \underbrace{a(1-a)x}_{\frac{\partial a}{\partial w}} \cdot (a-y) + a(1-a) \cdot \underbrace{a(1-a)x}_{\frac{\partial (a-y)}{\partial w}} \right] \quad \text{(这里稍微修正笔记中的合并顺序)} \\ \text{更直观的展开} &: \\ L''(w) &= x^2 a(1-a) [ \underbrace{a(1-a)}_{\text{Term 1}} + \underbrace{(a-y)(1-2a)}_{\text{Term 2}} ] \end{aligned}
$$

### 2.2 非凸成因分析

$$
K = a(1-a) + (a-y)(1-2a)
$$

- Term 1: $a(1-a)$ 恒 $>0$。
- **Term 2**: $(a-y)(1-2a)$ **符号不定**。

  - **Case (Prediction is confidently wrong)**:

    设 $y=1$ 但 $a \to 0$ (e.g., $a=0.01$)。

    Term 2 $\approx (0.01 - 1)(1 - 0.02) \approx (-0.99)(0.98) \approx -0.97$。

    Term 1 $\approx 0$。

    $\implies L''(w) < 0$ (Concave/开口向下)。

    $\therefore$ **存在二阶导数小于 0 的区域，故非凸。**

---

## 3. 逻辑回归 + BCE (The Solution)

**结论**：**严格凸 (Strictly Convex)**。交叉熵通过“数学消元”修复了 Sigmoid 的非凸性。

### 3.1 求解梯度 (Univariate Case)

- 损失 (BCE)：$L = -[y \ln a + (1-y) \ln (1-a)]$
- Loss 对 $a$ 的导数：$\frac{\partial L}{\partial a} = \frac{a-y}{a(1-a)}$

**梯度推导**:

$$
\begin{aligned} \frac{\partial L}{\partial w} &= \frac{\partial L}{\partial a} \cdot \frac{\partial a}{\partial z} \cdot \frac{\partial z}{\partial w} \\ &= \left[ \frac{a-y}{a(1-a)} \right] \cdot \left[ a(1-a) \right] \cdot x \\ &= (a-y)x \end{aligned}
$$

> **关键点**：BCE 的分母与 Sigmoid 的导数完美抵消。梯度形式回归线性。

### 3.2 Hessian 推导 (Vector Case)

基于梯度 $\nabla L(\mathbf{w}) = \mathbf{X}^T(\mathbf{a} - \mathbf{y})$：

$$
\mathbf{H} = \nabla^2 L(\mathbf{w}) = \frac{\partial (\mathbf{X}^T \mathbf{a})}{\partial \mathbf{w}} = \mathbf{X}^T \frac{\partial \mathbf{a}}{\partial \mathbf{z}} \frac{\partial \mathbf{z}}{\partial \mathbf{w}}
$$

其中 $\frac{\partial \mathbf{a}}{\partial \mathbf{z}}$ 是对角矩阵 $\mathbf{S} = \text{diag}(a_i(1-a_i))$。

$$
\mathbf{H} = \mathbf{X}^T \mathbf{S} \mathbf{X}
$$

### 3.3 凸性证明 (PSD Check)

验证 $\mathbf{H} = \mathbf{X}^T \mathbf{S} \mathbf{X}$：

$$
\forall \mathbf{v} \neq \mathbf{0}, \quad \mathbf{v}^T \mathbf{H} \mathbf{v} = (\mathbf{X}\mathbf{v})^T \mathbf{S} (\mathbf{X}\mathbf{v})
$$

令 $\mathbf{u} = \mathbf{X}\mathbf{v}$，则上式为加权平方和：

$$
\mathbf{u}^T \mathbf{S} \mathbf{u} = \sum_{i=1}^N \underbrace{a_i(1-a_i)}_{>0} u_i^2 \ge 0
$$

由于 $a \in (0,1)$，权重恒为正。

$\therefore$ **Hessian 恒为半正定，Logistic Regression + BCE 为凸函数。**
