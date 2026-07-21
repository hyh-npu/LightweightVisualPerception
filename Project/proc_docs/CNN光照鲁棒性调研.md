# CNN网络对光照变化的鲁棒性 — 调研分析

> 对应问题清单 1.2 (视觉前端方案) 和 1.3 (IALM光照自适应模块)

---

## 一、核心问题

UL-VIO的视觉前端是6层CNN。CNN本身对光照变化是否鲁棒？

**答案**: 普通CNN对光照变化**不鲁棒**。光照变化会导致:
- 激活值分布偏移 (CNN学习的是像素强度模式, 光照改变后强度分布改变)
- 特征响应不稳定 (同一点在不同光照下可能被检测到/被漏掉)
- 整体定位精度下降

但有一系列成熟方法可以解决这个问题。

---

## 二、5种光照鲁棒方案 (从易到难)

### 方案A: IBN-Net 归一化层替换 (推荐)
**核心论文**: IBN-Net (Pan et al., ECCV 2018) — arXiv:1807.09441

| 项目 | 详情 |
|:----|:------|
| 核心思想 | CNN中同时使用 **Instance Normalization (IN)** + **Batch Normalization (BN)** |
| IN的作用 | 消除外观差异(光照、颜色、风格), 保留内容结构 |
| BN的作用 | 加速训练, 保留判别性特征 |
| 计算开销 | **几乎不增加** (只是替换归一化层, 不增加参数量) |
| 代码 | https://github.com/XingangPan/IBN-Net (PyTorch) |
| 应用案例 | 分类/ReID/语义分割等任务上显著提升跨域泛化能力 |
| 是否可插入UL-VIO | ✅ **完全可行** — 直接把UL-VIO的6层CNN中的BN换成IBN块 |

**实现方式**: 在UL-VIO的VisualEncoder中, 把每层的 `BatchNorm2d` 替换为 `IBN` 块
```python
# 原来: nn.BatchNorm2d(channels)
# 替换为: IBN(channels)  # 内部自动分配IN和BN的比例
```

---

### 方案B: 训练数据光照增强
**不需要改网络结构, 只改训练数据**

| 方法 | 说明 | 实现难度 |
|:----|:-----|:--------|
| 随机亮度/对比度调整 | torchvision.transforms.ColorJitter | 一行代码 |
| 随机伽马变换 | torchvision.transforms.functional.adjust_gamma | 一行代码 |
| 随机阴影模拟 | 在图像上叠加随机椭圆遮罩 | 几行代码 |
| 随机曝光模拟 | 模拟过曝/欠曝效果 | 中等 |

**SuperPoint论文中公开了他们的数据增强策略**, 可以参考。
这种方法的局限: 只对训练数据分布内的光照变化有效, 极端光照可能仍不行。

---

### 方案C: 光照预处理 (输入网络前做归一化)
**在图像进入网络之前, 先做光照归一化**

| 方法 | 说明 | 计算开销 |
|:----|:-----|:--------|
| **CLAHE** (限制对比度直方图均衡) | OpenCV内置, 增强局部对比度 | 极低 (<0.5ms/帧) |
| **Retinex理论** (SSR/MSR) | 分离光照层和反射层 | 中等 |
| **Zero-DCE** (轻量CNN做光照增强) | 不需要配对数据, 自监督 | 轻量, 1-2ms/帧 |
| **直方图均衡化** | 最简单 | 极低 |

**优点**: 与VIO算法解耦, 可作为独立的预处理模块, 不影响VIO框架。
**缺点**: 增加前处理计算量; 某些场景可能过度增强噪声。

---

### 方案D: 更强光照鲁棒特征架构
**替换UL-VIO的CNN编码器为更鲁棒的深度特征架构**

| 方法 | 说明 | 参数量 |
|:----|:-----|:------|
| **SuperPoint** (CVPR 2018) | 自监督特征点+描述子, 训练含光照增强 | ~1.3M |
| **LightGlue** (2023) | SuperGlue加速版, 稀疏匹配 | ~5M |
| **AirVO (2023)** | CNN+GNN检测角点, 专为光照鲁棒设计 | 中等 |
| **LIR-LIVO (2025)** | SuperPoint+LightGlue, 光照鲁棒特征 | 适中 |

**参考工作**:
- AirVO (arXiv:2212.07595): 使用CNN+GNN在光照变化场景检测鲁棒角点, 低功耗平台实时运行
- LIR-LIVO (arXiv:2502.08676): SuperPoint+LightGlue做光照鲁棒特征, 在Hilti'22的弱光场景表现SOTA
- BEVIO (arXiv:2606.00709): 针对月球昼夜极端光照, 用BEV图像匹配, 0.25Hz低帧率仍可靠

---

### 方案E: 线特征辅助
**线特征对光照变化的鲁棒性高于点特征**

| 方法 | 说明 |
|:----|:-----|
| PL-EVIO (arXiv:2209.12160) | 事件相机+点线特征VIO |
| AirVO (arXiv:2212.07595) | 点+线特征VO, 光照鲁棒 |
| OTPL-VIO (arXiv:2603.09653) | 最优传输线关联VIO |

线特征在弱纹理/光照变化场景比点特征可靠, 但线特征的提取和匹配计算量较大。

---

## 三、对UL-VIO的具体改进建议

### 推荐方案组合

考虑到你们 **UL-VIO基座 + 轻量化** 的核心目标:

```
UL-VIO VisualEncoder 改进方案 (推荐组合)

第一步: IBN替换BN (方案A) ← 几乎零成本, 效果显著
第二步: 训练数据增加光照增强 (方案B) ← 一行代码
第三步 (可选): 输入加CLAHE预处理 (方案C) ← OpenCV自带

如果效果还不够:
第四步: 替换视觉前端为SuperPoint+LightGlue (方案D)
```

### 为什么不推荐其他方案

| 方案 | 不推荐理由 |
|:----|:----------|
| 线特征(方案E) | 线提取匹配计算量大, 与轻量化目标冲突 |
| Zero-DCE预处理 | 额外CNN推理, 增加2ms/帧延迟, 边缘设备可能扛不住 |
| 自建光照鲁棒CNN架构 | 研发周期太长, 本科团队时间不够 |

### 参考论文摘要

**IBN-Net (ECCV2018)** — arXiv:1807.09441
> "IBN-Net carefully integrates Instance Normalization (IN) and Batch Normalization (BN) as building blocks... IN learns features that are invariant to appearance changes, such as colors, styles, and virtuality/reality, while BN is essential for preserving content related information."
> → IN对光照/颜色/风格变化不变, BN保存内容信息. IBN-Net不增加计算量.

**AirVO (2023)** — arXiv:2212.07595
> CNN+GNN检测和匹配光照鲁棒角点, 点线特征融合, 可在低功耗嵌入式平台实时运行.

**LIR-LIVO (2025)** — arXiv:2502.08676
> SuperPoint+LightGlue光照鲁棒特征 + LiDAR辅助, 在Hilti'22弱光场景SOTA.

**BEVIO (2026)** — arXiv:2606.00709
> 月球昼夜导航, BEV图像匹配应对极端光照, 0.25Hz低帧率仍可靠.

---

## 四、对项目申报书的修正建议

申报书中提到的 **IALM光照自适应模块** 可以用 IBN-Net 替代:
- IBN-Net 作为成熟的归一化层方案, 比自研IALM更可靠
- 代码开源, 直接集成
- 不增加计算量, 符合轻量化要求

或者将 IALM 定义为 **IBN-Net + 数据光照增强** 的组合, 这样既有理论支撑, 实现起来也简单。
