---
title: 矩阵的逆
date: 2024-08-27
description: 
categories:
    - 线性代数
---

## 待定系数法
例如$A=
 \left(
    \begin{matrix}
        1 & 1\\
        2 & 3
    \end{matrix}
\right)
$，设$A^T=
 \left(
    \begin{matrix}
        a & b\\
        c & d
    \end{matrix}
  \right)
$，则$$A·A^T=
\left(
  \begin{matrix}
        a+2c & b+2d\\
        a+3c & b+3d
  \end{matrix}
\right) =
\left(
  \begin{matrix}
        1 & 0\\
        0 & 1
  \end{matrix}
\right)$$
## 初等行变换/初等列变换
行变换和列变换不能同时使用
## 伴随矩阵
$A^{-1} = \frac{A^*}{|A|}$
伴随矩阵就是矩阵各个元素代数余子式组成的矩阵
