滤波算法是**信号处理、自动控制、导航定位、计算机视觉与数据同化中的基础工具**，其核心目标是在噪声、建模误差与观测不完整的条件下，从观测数据中恢复更可靠的信号、图像或系统状态。

## 1. 概述

“滤波“本质上是<u>**从受扰动的数据中提取更有价值的信息**</u>。

在线性时不变信号处理中，滤波通常表示**对频谱成分进行增强或抑制**；在图像处理中，滤波则是**针对像素邻域进行某种空间或统计操

滤波算法大致可以分为*三类*：

**第一类**是面向离线数据平滑的局部窗口方法，例如 Savitzky-Golay 滤波，它重点解决**实验数据噪声抑制**与**形状保持的平衡问题**；

**第二类**是面向动态系统的状态空间递推估计方法，包括 Kalman 滤波、扩展 Kalman 滤波、无迹 Kalman 滤波和粒子滤波等，其主要思想是**将系统演化与观测过程分别建模，并在时间轴上连续修正估计**；

**第三类**是面向图像的**非线性局部滤波方法**，如中值滤波与双边滤波，目标是在保持边缘等结构信息的同时尽可能去除噪声。

## 2. 滤波算法分类

### 2.1 按处理对象分类

滤波算法以下三类：

第一类是**信号平滑滤波**。其*输入通常是一维离散序列*，如实验测量曲线、光谱数据、传感器输出等。此时研究者最关心的是如何在抑制高频噪声的同时**尽量保留信号的峰值位置、局部趋势及导数信息**。Savitzky-Golay 滤波正是这一类方法的代表。

第二类是**状态估计滤波**。此类问题中，系统状态本身往往不可直接准确测量，只能通过动态模型与带噪观测进行间接估计。例如目标跟踪中的速度与位置估计、惯性导航中的姿态与偏置估计、机器人定位中的位姿估计等。Kalman 滤波及其非线性扩展方法构成了这一领域的主干。

第三类是**图像空间滤波**。图像去噪的关键难点在于，边缘与细节在局部区域内通常也表现为剧烈变化，因此简单平均会同时削弱噪声和有用结构。中值滤波通过次序统计增强对脉冲噪声的鲁棒性，双边滤波则通过“位置接近”和“灰度相近”的双重约束实现保边平滑。

### 2.2 按模型假设分类

如果从模型假设角度划分，经典滤波方法又可分为<u>线性高斯、非线性近似和样本近似</u>三种。

线性高斯路线以 **Kalman 滤波为代表**。它假定系统状态转移和观测方程是线性的，过程噪声与观测噪声满足零均值高斯分布。在这些前提下，后验分布可以完全由均值和协方差描述，滤波过程可递推完成，并具有最小均方误差意义下的最优性。

非线性近似路线主要包括**扩展 Kalman 滤波与无迹 Kalman 滤波**。扩展 Kalman 滤波通过在当前工作点对非线性函数做一阶泰勒展开，将问题近似为局部线性问题；无迹 Kalman 滤波则放弃显式线性化，而是用一组精心选择的样本点传播均值和协方差，以更准确地逼近非线性变换后的统计量。

样本近似路线以**粒子滤波**为代表。它不再假定后验分布可以被单个高斯分布近似，而是直接用一组带权粒子表示后验。这样一来，算法可以处理明显非线性、非高斯甚至多峰分布问题。

## 3. 论文信息整合

### 3.1 Savitzky-Golay 滤波

#### 3.1.1 参考文献

Savitzky, A., and Golay, M. J. E. 1964. *Smoothing and Differentiation of Data by Simplified Least Squares Procedures*.

#### 3.1.2 研究背景

在**化学分析、光谱测量和实验数据处理**中，原始数据往往含有随机波动。简单滑动平均虽然可以降低噪声，但会导致*谱峰变宽、峰值降低、拐点模糊，从而影响后续定量分析*。作者观察到，许多实验曲线在局部窗口内可以用**低阶多项式很好地逼**近，因此提出用**局部最小二乘拟合代替简单平均**，以实现既平滑又尽量不破坏形状特征的处理方式。

#### 3.1.3 核心思想

对于长度为 $2m+1$ 的滑动窗口，设窗口中心为 $x_0$，邻域采样值为

$$
y_{-m}, y_{-m+1}, \dots, y_0, \dots, y_{m}.
$$

在该窗口内，用 $p$ 阶多项式

$$
f(t)=a_0+a_1 t+a_2 t^2+\cdots+a_p t^p
$$

对数据做最小二乘拟合。由于最终只关心中心点的平滑值或其导数值，因此可以预先推导出一组固定卷积系数，使得每次计算只需做加权求和。也就是说，Savitzky-Golay 滤波**本质上是“局部多项式拟合”的快速卷积实现**。

如果记窗口数据向量为 $\mathbf{y}$，设计矩阵为 $\mathbf{A}$，则最小二乘解为

$$
\hat{\mathbf{a}} = (\mathbf{A}^T\mathbf{A})^{-1}\mathbf{A}^T\mathbf{y}.
$$

中心点的平滑值即为 $\hat a_0$，而一阶、二阶导数也可以从拟合多项式系数直接得到。因此，该方法不仅可用于平滑，还可以稳定地进行数值微分。

### 3.2 Kalman 线性滤波

#### 3.2.1 参考文献

Kalman, R. E. 1960. *A New Approach to Linear Filtering and Prediction Problems*.

#### 3.2.2 研究背景

20 世纪中期，研究者已经在随机过程、维纳滤波和控制理论领域取得了大量成果，但针对**动态系统在线估计**的统一框架尚不成熟。传统方法在处理非平稳系统和实时递推时存在困难。

#### 3.2.3 状态空间模型

Kalman 滤波建立在线性离散状态空间模型之上：

$$
\mathbf{x}_k = \mathbf{F}_k \mathbf{x}_{k-1} + \mathbf{B}_k \mathbf{u}_k + \mathbf{w}_k,
$$

$$
\mathbf{z}_k = \mathbf{H}_k \mathbf{x}_k + \mathbf{v}_k,
$$

其中，$\mathbf{x}_k$ 为系统状态，$\mathbf{z}_k$ 为观测量，$\mathbf{w}_k$ 和 $\mathbf{v}_k$ 分别表示过程噪声与测量噪声。通常假设它们满足零均值高斯分布，协方差分别为 $\mathbf{Q}_k$ 和 $\mathbf{R}_k$。

#### 3.2.4 核心递推

Kalman 滤波由“预测”和“更新”两个阶段组成。**预测阶段用系统模型将上一时刻状态传递到当前时刻：**

$$
\hat{\mathbf{x}}_{k|k-1} = \mathbf{F}_k \hat{\mathbf{x}}_{k-1|k-1} + \mathbf{B}_k \mathbf{u}_k,
$$

$$
\mathbf{P}_{k|k-1} = \mathbf{F}_k \mathbf{P}_{k-1|k-1}\mathbf{F}_k^T + \mathbf{Q}_k.
$$

更新阶段将当前观测引入：

$$
\mathbf{K}_k = \mathbf{P}_{k|k-1}\mathbf{H}_k^T(\mathbf{H}_k\mathbf{P}_{k|k-1}\mathbf{H}_k^T+\mathbf{R}_k)^{-1},
$$

$$
\hat{\mathbf{x}}_{k|k} = \hat{\mathbf{x}}_{k|k-1} + \mathbf{K}_k(\mathbf{z}_k-\mathbf{H}_k\hat{\mathbf{x}}_{k|k-1}),
$$

$$
\mathbf{P}_{k|k} = (\mathbf{I}-\mathbf{K}_k\mathbf{H}_k)\mathbf{P}_{k|k-1}.
$$

其中，$\mathbf{K}_k$ 为 Kalman 增益，它决定了模型预测与测量校正之间的权衡。若测量噪声较小，则增益偏大，估计更依赖观测；若模型更可信，则增益偏小，估计更依赖预测。

#### 3.2.5 适用场景

Kalman 滤波广泛应用于雷达跟踪、导航制导、金融时序估计、工业过程监测和多传感器信息融合等领域。它不仅是一个具体算法，更是一种统一的动态估计思想。

### 3.3 无迹 Kalman 滤波

#### 3.3.1 参考文献

Julier, S. J., and Uhlmann, J. K. 2004. *Unscented Filtering and Nonlinear Estimation*.

早期提出稿 *A New Extension of the Kalman Filter to Nonlinear Systems*。

#### 3.3.2 研究背景

扩展 Kalman 滤波通过**一阶泰勒展开**近似非线性函数。

#### 3.3.3 Unscented Transform 的基本思想

无迹变换的核心思想是：与其近似非线性函数，不如更准确地近似分布。具体地，若一个 $n$ 维随机变量的均值为 $\hat{\mathbf{x}}$，协方差为 $\mathbf{P}$，则选择 $2n+1$ 个 sigma 点：

$$
\chi_0 = \hat{\mathbf{x}},
$$

$$
\chi_i = \hat{\mathbf{x}} + \left(\sqrt{(n+\lambda)\mathbf{P}}\right)_i,\quad i=1,\dots,n,
$$

$$
\chi_{i+n} = \hat{\mathbf{x}} - \left(\sqrt{(n+\lambda)\mathbf{P}}\right)_i,\quad i=1,\dots,n.
$$

这些点按一定权重分布在均值周围，能够刻画原分布的前两阶矩。然后将每个 sigma 点通过非线性函数传播，再对传播结果加权求和，以估计输出的均值与协方差。与 EKF 不同，UKF 不需要显式求导。

#### 3.3.4 UKF 的递推流程

对状态方程和观测方程分别进行 sigma 点传播，可得到预测状态均值、预测协方差、预测观测均值及交叉协方差，进而构造 Kalman 增益并更新状态。其结构与 Kalman 滤波类似，*但内部对非线性的处理从“线性化函数”变为“传播样本点”*。



### 3.4 中值滤波

#### 3.4.1 文献

Huang, T. S., Yang, G. J., and Tang, G. Y. 1979. *A Fast Two-Dimensional Median Filtering Algorithm*.

#### 3.4.2 研究背景

中值滤波是图像**去除脉冲噪声**的工具。与均值滤波相比，它不容易模糊边缘。然而，早期二维中值滤波实现往往需要*对每个窗口内像素排序*，这在图像尺寸增大或窗口滑动频繁时计算开销极大。

该文要解决的问题是如何在保持中值滤波效果的同时大幅降低二维实现的计算复杂度.

#### 3.4.3 基本原理

中值滤波的输出定义为窗口内像素集合的中位数，即次序统计量中的中间值。对于窗口 $\Omega$ 内像素集合 $\{I(q)\mid q\in \Omega\}$，输出可记为

$$
\hat I(p)=\operatorname{median}\{I(q)\mid q\in \Omega(p)\}.
$$

若对每个位置都重新排序，复杂度很高。作者的关键改进是利用滑动窗口相邻位置之间的重叠关系，通过维护灰度直方图并进行增量更新来求中值。窗口每向右移动一列，只需从直方图中删除最左侧离开的像素灰度计数，并加入最右侧新进入的像素灰度计数，然后在更新后的直方图中寻找累计计数达到一半位置的灰度值即可。

### 3.5 双边滤波

#### 3.5.1 文献

Tomasi, C., and Manduchi, R. 1998. *Bilateral Filtering for Gray and Color Images*.

#### 3.5.2 研究背景与问题

传统低通滤波器的核心问题在于，它们*只利用像素之间的空间接近性，而不考虑灰度或颜色是否相似*，因此在平滑图像的同时也会跨边缘做平均，造成轮廓模糊。对彩色图像逐通道单独平滑时，这个问题更严重，还可能*引入伪色和颜色漂移*。本文作者的工作正是针对“**如何在去噪的同时保持边缘和色彩结构**”这一问题提出了解法。

#### 3.5.3 公式

双边滤波对像素 $p$ 的输出定义为

$$
\hat I(p)=\frac{1}{W_p}\sum_{q\in \Omega}
G_{\sigma_s}(\|p-q\|)\,G_{\sigma_r}(|I(p)-I(q)|)\,I(q),
$$

其中归一化因子为

$$
W_p=\sum_{q\in \Omega}
G_{\sigma_s}(\|p-q\|)\,G_{\sigma_r}(|I(p)-I(q)|).
$$

上式中，$G_{\sigma_s}$ 是空间域高斯核，用于度量像素位置接近程度；**$G_{\sigma_r}$ 是值域高斯核，用于度量灰度或颜色相似程度**。只有同时“离得近”和“像得像”的像素才会对中心像素产生较大影响。正因为此，位于边缘两侧但灰度差异显著的像素不会被强行平均，从而实现保边平滑。



## 参考文献

[1] Savitzky A, Golay M J E. Smoothing and Differentiation of Data by Simplified Least Squares Procedures[J]. Analytical Chemistry, 1964, 36(8): 1627-1639.

[2] Kalman R E. A New Approach to Linear Filtering and Prediction Problems[J]. Journal of Basic Engineering, 1960, 82(1): 35-45. 

[3] Schmidt S F. Application of State-Space Methods to Navigation Problems[A]. In: Leondes C T, ed. Advances in Control Systems. New York: Academic Press, 1966: 293-340.

[4] Julier S J, Uhlmann J K. Unscented Filtering and Nonlinear Estimation[J]. Proceedings of the IEEE, 2004, 92(3): 401-422. 

[5] Julier S J, Uhlmann J K. A New Extension of the Kalman Filter to Nonlinear Systems[C]. Proceedings of AeroSense: The 11th International Symposium on Aerospace/Defense Sensing, Simulation and Controls, 1997.

[6] Gordon N J, Salmond D J, Smith A F M. Novel Approach to Nonlinear/Non-Gaussian Bayesian State Estimation[J]. IEE Proceedings F - Radar and Signal Processing, 1993, 140(2): 107-113. 

[7] Arulampalam M S, Maskell S, Gordon N, Clapp T. A Tutorial on Particle Filters for Online Nonlinear/Non-Gaussian Bayesian Tracking[J]. IEEE Transactions on Signal Processing, 2002, 50(2): 174-188.

[8] Huang T S, Yang G J, Tang G Y. A Fast Two-Dimensional Median Filtering Algorithm[J]. IEEE Transactions on Acoustics, Speech, and Signal Processing, 1979, 27(1): 13-18.

[9] Tomasi C, Manduchi R. Bilateral Filtering for Gray and Color Images[C]. Proceedings of the 1998 IEEE International Conference on Computer Vision, 1998: 839-846. 