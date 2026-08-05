---
title: "TR 38.811 NTN 天线方向图与信道模型学习笔记 v3"
author: ""
date: "2026-08-04"
---

# TR 38.811 NTN 天线方向图与信道模型学习笔记

> 本笔记基于 3GPP TR 38.811 V15.4.0（Release 15）Clauses 6.3-6.10 整理，并以 TR 38.901 的空间信道生成框架作为必要补充。正文按“链路几何 → 参考天线 → 大尺度传播 → 模型选择 → 局部快衰落 → 模型装配”的知识链展开；CDL → TDL → 平坦衰落的简化关系在进入参数细节前统一说明，直接规定与原文位置分布在相应小节中。

涉及的 ITU-R 方法按 TR 38.811 所引用的建模入口说明，不把后续新版 ITU-R 参数反向写成 3GPP 原文。

一个较完整的 NTN 基带信道可以概括为：

\[
\boxed{
\text{链路几何}
+\text{参考天线}
+\text{大尺度传播}
+\text{局部快衰落}
}
\]

四层分别回答不同问题：链路有多远、从什么方向传播；天线向该方向提供多少增益；传播途中损失多少平均功率；接收信号在局部时延、角度和时间上如何起伏。

## 1 链路几何

### 1.1 坐标系、仰角与天顶角

TR 38.811 使用地心地固坐标系（Earth-Centred Earth-Fixed，ECEF）描述 UE、卫星和 HAPS 的位置，并把地球近似为半径 \(R_E=6371\ \mathrm{km}\) 的球体。ECEF 坐标原点位于地心，\(x-y\) 平面位于赤道平面，\(z\) 轴指向地理北极。

对于地面 UE，更容易使用局部坐标系理解传播方向：

- 局部 \(+z\) 轴指向天顶；
- 仰角 \(\alpha_{\mathrm{elev}}\) 从当地水平面向上测量；
- 天顶角从局部 \(+z\) 轴向下测量；
- 方位角描述水平方向。

接收端的天顶到达角（Zenith Angle of Arrival，ZOA）描述信号从哪个俯仰方向到达。若采用“接收机指向信号源”的方向定义，则地面 UE 处有：

\[
\theta_{\mathrm{ZOA}}
=90^\circ-\alpha_{\mathrm{elev}}.
\]

例如，UE 看到卫星的仰角为 \(30^\circ\)，则 ZOA 为 \(60^\circ\)；卫星位于正上方时，仰角为 \(90^\circ\)，ZOA 为 \(0^\circ\)。

| 角度 | 英文与缩写 | 定义端点 |
|---|---|---|
| 方位到达角 | Azimuth Angle of Arrival，AOA | 接收端 |
| 天顶到达角 | Zenith Angle of Arrival，ZOA | 接收端 |
| 方位出发角 | Azimuth Angle of Departure，AOD | 发射端 |
| 天顶出发角 | Zenith Angle of Departure，ZOD | 发射端 |

在卫星端使用 ZOA/ZOD 时，必须先确认卫星局部坐标系的 \(+z\) 轴朝向地球还是远离地球，并确认角度表示“指向信号源”还是“电磁波传播方向”。不同定义之间可能需要姿态坐标变换，不能脱离坐标系直接比较数值。

> **关键原文定位：**TR 38.811 Clause 6.3、Figure 6.3-1 定义 ECEF 坐标系；TR 38.901 Clause 7.1-7.2 给出阵列与角度建模所采用的坐标系和场分量约定。

### 1.2 斜距与几何传播时延

设平台高度为 \(h_0\)，地球半径为 \(R_E\)，地面终端仰角为 \(\alpha\)，则 UE 到卫星或 HAPS 的斜距为：

\[
d
=
\sqrt{
R_E^2\sin^2\alpha
+h_0^2
+2h_0R_E
}
-R_E\sin\alpha.
\]

一程几何传播时延为：

\[
\tau_{\mathrm{geo}}=\frac{d}{c}.
\]

斜距不是平台高度。只有卫星位于 UE 正上方时，\(d\approx h_0\)；仰角降低后，传播路径显著增长。链路预算、绝对定时、HARQ 与随机接入时序关心的是这一毫秒至百毫秒量级的几何传播时延。

局部多径超额时延则来自反射或散射路径相对参考路径的路程差。完整到达时延应写成：

\[
\tau_p^{\mathrm{total}}
=\tau_{\mathrm{geo}}
+\tau_{p,\mathrm{excess}}.
\]

例如 \(\tau_{\mathrm{geo}}=4\ \mathrm{ms}\)、局部超额时延为 \(250\ \mathrm{ns}\)，该径的绝对到达时延约为 \(4\ \mathrm{ms}+250\ \mathrm{ns}\)。接收机完成公共定时补偿后，CDL/TDL 主要描述后面的纳秒至微秒级相对时延结构。

> **关键原文定位：**TR 38.811 Clause 6.6.2、Figure 6.6.2-1 和式 (6.6-3) 给出斜距与仰角、高度的关系；CDL/TDL 中的抽头时延不包含这段公共几何传播时延。

### 1.3 公共多普勒与局部差分多普勒

平台与 UE 的径向相对速度使斜距随时间变化。载频为 \(f_c\) 时，几何多普勒可写为：

\[
\nu_{\mathrm{geo}}(t)
=-\frac{f_c}{c}\frac{\mathrm d d(t)}{\mathrm dt}.
\]

多普勒可以进一步分层：

\[
\nu_{n,m}
=\nu_{\mathrm{platform}}
+\nu_{\mathrm{UE},n,m}
+\nu_{\mathrm{scatter},n,m}.
\]

- \(\nu_{\mathrm{platform}}\) 是平台运动造成的大公共频移，决定初始捕获范围与预补偿要求；
- \(\nu_{\mathrm{UE},n,m}\) 与 \(\nu_{\mathrm{scatter},n,m}\) 随局部路径不同，决定补偿后的时间选择性和差分多普勒；
- 星历、GNSS 和频偏估计可以消除大部分公共频移，但不能自动消除所有路径或所有 UE 的残余频偏。

因此，大多普勒本身并不必然破坏 OFDM 正交性。若它对某个 UE 近似为公共频偏且能被准确预补偿，真正形成 ICI 的主要因素是残余频偏、符号内频偏变化、差分多普勒以及补偿更新不及时。

> **关键原文定位：**TR 38.811 Clause 6.2 说明 NTN 与地面蜂窝信道在平台运动和大多普勒上的差异；Clause 6.7.2 的快衰落参数用于描述补偿后的局部时间变化，而非替代轨道几何产生的公共多普勒。

### 1.4 几何量与局部信道的分层基带表达

把几何时延、公共多普勒、大尺度损耗和局部快衰落组合起来，可写为：

\[
y(t)
=
\sqrt{G_{\mathrm{LS}}(t)}
e^{j2\pi\nu_{\mathrm{geo}}t}
\sum_p h_p(t)
x\!\left(t-\tau_{\mathrm{geo}}-\tau_p\right)
+n(t),
\]

其中：

- \(G_{\mathrm{LS}}\) 表示路径损耗、阴影、杂波和附加传播衰减共同形成的线性功率增益；
- \(\tau_{\mathrm{geo}}\) 和 \(\nu_{\mathrm{geo}}\) 由链路几何给出；
- \(h_p(t)\) 与 \(\tau_p\) 由 CDL、TDL 或单抽头快衰落模型给出。

对第 \(q\) 个 OFDM 符号、第 \(k\) 个子载波，可写成：

\[
Y[k,q]
=
\sqrt{G_{\mathrm{LS}}[q]}
e^{-j2\pi f_k\tau_{\mathrm{geo}}[q]}
e^{j2\pi\nu_{\mathrm{geo}}t_q}
H_{\mathrm{fast}}[k,q]
X[k,q]+N[k,q].
\]

若研究残余频偏或 ICI，必须在时域采样级实现符号内相位变化；只在每个 OFDM 符号上乘一个静态相位不能描述符号内正交性破坏。

> **关键原文定位：**TR 38.811 Clause 6 将坐标系、天线、大尺度传播和快衰落分别建模。上述分层基带式是对 Clauses 6.3-6.7 的统一表达，用来避免把几何传播量和局部多径参数混为一谈。

## 2 参考天线

### 2.1 卫星与 HAPS 的参考方向图

TR 38.811 Clause 6.4.1 采用典型圆形孔径反射面天线的归一化增益方向图描述卫星参考波束。直径为 \(D=2a\) 的均匀圆形孔径，其归一化功率方向图为：

\[
P_{\mathrm{circ}}(\theta)
=
\left|
\frac{2J_1(u)}{u}
\right|^2,
\qquad
u=ka\sin\theta
=\frac{\pi D}{\lambda}\sin\theta,
\]

其中 \(\theta\) 为相对波束轴 boresight 的偏离角，\(J_1(\cdot)\) 为第一类一阶贝塞尔函数。第一零点和半功率波束宽度近似为：

\[
\theta_{\mathrm{null}}
\approx1.22\frac{\lambda}{D},
\qquad
\mathrm{HPBW}
\approx1.03\frac{\lambda}{D}\ \mathrm{rad}.
\]

峰值增益近似为：

\[
G_{\max}
\approx
\eta\left(\frac{\pi D}{\lambda}\right)^2,
\]

其中 \(\eta\) 为孔径效率。孔径越大或频率越高，主瓣通常越窄、峰值增益越高，同时对姿态、机械误差和指向误差更敏感。“归一化方向图峰值为 1”只描述相对形状，不表示绝对增益为 0 dBi。

HAPS 可以采用两种参考形式：上述圆形孔径贝塞尔方向图，或 TR 38.901 Clause 7.3 的双线极化矩形面板阵列模型。

> **关键原文定位：**TR 38.811 Clause 6.4.1、Figure 6.4.1-1 给出圆形孔径方向图；HAPS 还可采用 TR 38.901 Clause 7.3 的基站面板阵列。

### 2.2 UE 参考天线与适用场景

TR 38.811 Clause 6.4.2 为快衰落建模采用三类 UE 参考方向图：

| UE 参考天线 | 极化 | 典型空间性质 |
|---|---|---|
| 准全向天线 | 线极化 | 一个平面内近似全向，接收较多方向的局部散射 |
| 同相阵列 | 双线极化 | 具有阵列增益和方向选择性 |
| VSAT 型天线 | 圆极化，固定或跟踪 | 高增益、窄波束，面向卫星方向 |

报告只在满足平坦衰落条件的部署中采用 VSAT 型参考方向图。这是该参考模型的适用限定，不表示物理 VSAT 只能工作在平坦衰落信道。宽带 VSAT、低仰角 VSAT 或处于强反射环境中的 VSAT 仍可能需要 TDL/CDL。

> **关键原文定位：**TR 38.811 Clause 6.4.2 列出 Quasi Isotropic、Co-phased array 和 “VSAT type - circular polarization: fixed or tracking” 三类参考 UE 天线，并限定 VSAT 参考模式用于平坦衰落部署。

### 2.3 圆形孔径、端口、波束与数据流

“圆形孔径”描述物理口径及其方向图，不直接规定馈源数、射频链数、天线端口数或数据流数。

| 概念 | 所在层次 | 与数据流的关系 |
|---|---|---|
| 圆形孔径 | 物理辐射口径 | 不直接决定流数 |
| 馈源或阵元 | 激励孔径的物理单元 | 可以形成一个或多个端口 |
| 天线端口 | 基带可独立控制或观测的维度 | 限制可用空间维度 |
| 射频链 | 独立上/下变频通道 | 限制同时处理的流数 |
| 波束 | 对空间方向的加权响应 | 与端口、流数不一一对应 |
| 数据流或层 | 相互独立的数据序列 | 受端口、射频链和信道秩共同限制 |

同一个反射面可以采用单馈源、双极化馈源、多馈源或相控阵馈源。可支持的数据流数满足：

\[
N_{\mathrm{stream}}
\leq
\min\!\left(
N_{\mathrm{RF,t}},
N_{\mathrm{RF,r}},
\operatorname{rank}(\mathbf H)
\right).
\]

如果仿真仅把圆形孔径公式实现为一个标量增益 \(G(\theta)\)，该具体实现等效于标量端口。这是建模选择造成的单流限制，不是圆形孔径本身的物理限制。

方向也要分层理解：方向图相对于自身 boresight 固定；反射面若固定安装，boresight 在卫星本体坐标系中通常固定；但卫星姿态、多馈源切换、电子扫描和数字波束赋形均可改变它在地球坐标系中的覆盖方向。LEO 波束若相对卫星本体固定，会随平台运动扫过地面；对地固定波束则需要持续指向控制或波束切换。

> **关键原文定位：**TR 38.811 Clause 6.4.1 只规定归一化功率方向图与 boresight 偏离角，并未规定馈源、端口、射频链或空间层数；波束和数据流能力需由额外阵列架构确定。

### 2.4 接收波束的导向矢量表示

对离散阵列，令到达方向为 \(\boldsymbol{\Omega}=(\phi,\theta)\)，其中 \(\phi\) 为 AOA，\(\theta\) 为 ZOA。单位方向向量可写为：

\[
\mathbf u(\phi,\theta)
=
\begin{bmatrix}
\sin\theta\cos\phi\\
\sin\theta\sin\phi\\
\cos\theta
\end{bmatrix}.
\]

若第 \(m\) 个接收阵元位置为 \(\mathbf p_m\)，窄带导向矢量为：

\[
\mathbf a_r(\phi,\theta)
=
\frac{1}{\sqrt M}
\begin{bmatrix}
e^{-j\frac{2\pi}{\lambda}\mathbf p_1^T\mathbf u}\\
\vdots\\
e^{-j\frac{2\pi}{\lambda}\mathbf p_M^T\mathbf u}
\end{bmatrix}.
\]

第 \(b\) 个接收波束由合并向量 \(\mathbf v_b\) 表示：

\[
z_b=\mathbf v_b^H\mathbf y,
\qquad
G_b^{\mathrm{rx}}(\phi,\theta)
=
\left|
\mathbf v_b^H\mathbf a_r(\phi,\theta)
\right|^2.
\]

对于连续圆形孔径，导向矢量可推广为孔径导向函数：

\[
A(\boldsymbol{\Omega})
=
\int_{\mathcal A}
w^*(\boldsymbol{\rho})
e^{-jk\boldsymbol{\rho}^T\mathbf u(\boldsymbol{\Omega})}
\,\mathrm d\boldsymbol{\rho}.
\]

对均匀圆形孔径积分后可得到贝塞尔函数形式的远场方向图。但 TR 38.811 给出的 \(G(\theta)\) 只有标量功率响应，缺少阵元位置、端口复相位、馈电网络和极化端口结构，因此不能仅凭该公式唯一反推出多端口导向矢量。

> **关键原文定位：**TR 38.811 Clause 6.4.1 提供标量参考方向图；离散阵列的球坐标、阵元场响应和阵列组合参见 TR 38.901 Clauses 7.1-7.3。

## 3 大尺度传播

### 3.1 LOS 概率与环境状态

大尺度模型首先决定链路处于视距（Line-of-Sight，LOS）还是非视距（Non-Line-of-Sight，NLOS）。TR 38.811 直接按 UE 环境和地面终端仰角给出 LOS 概率：

| 参考仰角 | 密集城市 | 城市 | 郊区与农村 |
|---:|---:|---:|---:|
| \(10^\circ\) | 28.2% | 24.6% | 78.2% |
| \(20^\circ\) | 33.1% | 38.6% | 86.9% |
| \(30^\circ\) | 39.8% | 49.3% | 91.9% |
| \(40^\circ\) | 46.8% | 61.3% | 92.9% |
| \(50^\circ\) | 53.7% | 72.6% | 93.5% |
| \(60^\circ\) | 61.2% | 80.5% | 94.0% |
| \(70^\circ\) | 73.8% | 91.9% | 94.9% |
| \(80^\circ\) | 82.0% | 96.8% | 95.2% |
| \(90^\circ\) | 98.1% | 99.2% | 99.8% |

一次 drop 的状态生成步骤为：

1. 把实际仰角 \(\alpha\) 映射到最近的 \(10^\circ,20^\circ,\ldots,90^\circ\) 参考角，超出表格范围时先限制到表格边界；
2. 根据环境和参考角查得 \(P_{\mathrm{LOS}}\)；
3. 生成 \(U_{\mathrm{LOS}}\sim\mathcal U(0,1)\)；
4. 若 \(U_{\mathrm{LOS}}\leq P_{\mathrm{LOS}}\)，令链路为 LOS，否则为 NLOS。

例如，城市环境、仰角 \(34^\circ\) 使用 \(30^\circ\) 表项，因此

\[
P_{\mathrm{LOS}}=0.493.
\]

若一次抽样得到 \(U_{\mathrm{LOS}}=0.62\)，则该 drop 为 NLOS。LOS/NLOS 状态随后决定杂波损耗、阴影衰落标准差、Ricean K 因子以及快衰落参数。

这一抽样是 drop-based 的空间统计实现。若 UE 连续移动，不能在每个时间采样点独立重抽 LOS/NLOS，否则状态会产生不连续跳变；此时需要为状态设置空间一致性或使用连续状态模型。

> **关键原文定位：**TR 38.811 Clause 6.6.1、Table 6.6.1-1 给出不同环境和仰角的 LOS 概率；实际角度使用最近的参考仰角值。

### 3.2 基本路径损耗、杂波与阴影衰落

TR 38.811 把基本路径损耗写为：

\[
PL_b
=FSPL(d,f_c)
+SF
+CL(\alpha,f_c),
\]

其中自由空间路径损耗为：

\[
FSPL(d,f_c)
=32.45
+20\log_{10}(f_c)
+20\log_{10}(d),
\]

式中 \(f_c\) 以 GHz、\(d\) 以 m 计。为避免单位误用，等价形式可以写为：

\[
\begin{aligned}
FSPL
&=32.45+20\log_{10}f_{\mathrm{GHz}}
       +20\log_{10}d_{\mathrm{m}},\\
&=92.45+20\log_{10}f_{\mathrm{GHz}}
       +20\log_{10}d_{\mathrm{km}},\\
&=32.45+20\log_{10}f_{\mathrm{MHz}}
       +20\log_{10}d_{\mathrm{km}}.
\end{aligned}
\]

这三种写法不能交叉混用；最稳妥的校验式为：

\[
FSPL
=20\log_{10}\left(\frac{4\pi d f_c}{c}\right),
\]

其中 \(d\) 与 \(c\) 使用相同长度单位，\(f_c\) 使用 Hz。

- \(FSPL\) 由斜距和载频决定；
- 阴影衰落 \(SF\) 在 dB 域中采用零均值高斯随机变量；
- 杂波损耗 \(CL\) 描述地面建筑和物体造成的平均附加衰减，依赖环境、频率和仰角；
- LOS 条件下，基线模型把杂波损耗置为 0 dB。

#### 3.2.1 阴影衰落与杂波损耗查表

表中 \(\sigma_{\mathrm{SF}}\) 是阴影衰落标准差，\(CL\) 只在 NLOS 分支使用。

密集城市参数为：

| 仰角 | S-LOS \(\sigma_{\mathrm{SF}}\) | S-NLOS \(\sigma_{\mathrm{SF}}\) | S-NLOS \(CL\) | Ka-LOS \(\sigma_{\mathrm{SF}}\) | Ka-NLOS \(\sigma_{\mathrm{SF}}\) | Ka-NLOS \(CL\) |
|---:|---:|---:|---:|---:|---:|---:|
| 10° | 3.5 | 15.5 | 34.3 | 2.9 | 17.1 | 44.3 |
| 20° | 3.4 | 13.9 | 30.9 | 2.4 | 17.1 | 39.9 |
| 30° | 2.9 | 12.4 | 29.0 | 2.7 | 15.6 | 37.5 |
| 40° | 3.0 | 11.7 | 27.7 | 2.4 | 14.6 | 35.8 |
| 50° | 3.1 | 10.6 | 26.8 | 2.4 | 14.2 | 34.6 |
| 60° | 2.7 | 10.5 | 26.2 | 2.7 | 12.6 | 33.8 |
| 70° | 2.5 | 10.1 | 25.8 | 2.6 | 12.1 | 33.3 |
| 80° | 2.3 | 9.2 | 25.5 | 2.8 | 12.3 | 33.0 |
| 90° | 1.2 | 9.2 | 25.5 | 0.6 | 12.3 | 32.9 |

城市参数为：

| 仰角 | S-LOS \(\sigma_{\mathrm{SF}}\) | S-NLOS \(\sigma_{\mathrm{SF}}\) | S-NLOS \(CL\) | Ka-LOS \(\sigma_{\mathrm{SF}}\) | Ka-NLOS \(\sigma_{\mathrm{SF}}\) | Ka-NLOS \(CL\) |
|---:|---:|---:|---:|---:|---:|---:|
| 10° | 4 | 6 | 34.3 | 4 | 6 | 44.3 |
| 20° | 4 | 6 | 30.9 | 4 | 6 | 39.9 |
| 30° | 4 | 6 | 29.0 | 4 | 6 | 37.5 |
| 40° | 4 | 6 | 27.7 | 4 | 6 | 35.8 |
| 50° | 4 | 6 | 26.8 | 4 | 6 | 34.6 |
| 60° | 4 | 6 | 26.2 | 4 | 6 | 33.8 |
| 70° | 4 | 6 | 25.8 | 4 | 6 | 33.3 |
| 80° | 4 | 6 | 25.5 | 4 | 6 | 33.0 |
| 90° | 4 | 6 | 25.5 | 4 | 6 | 32.9 |

郊区和农村参数为：

| 仰角 | S-LOS \(\sigma_{\mathrm{SF}}\) | S-NLOS \(\sigma_{\mathrm{SF}}\) | S-NLOS \(CL\) | Ka-LOS \(\sigma_{\mathrm{SF}}\) | Ka-NLOS \(\sigma_{\mathrm{SF}}\) | Ka-NLOS \(CL\) |
|---:|---:|---:|---:|---:|---:|---:|
| 10° | 1.79 | 8.93 | 19.52 | 1.9 | 10.7 | 29.5 |
| 20° | 1.14 | 9.08 | 18.17 | 1.6 | 10.0 | 24.6 |
| 30° | 1.14 | 8.78 | 18.42 | 1.9 | 11.2 | 21.9 |
| 40° | 0.92 | 10.25 | 18.28 | 2.3 | 11.6 | 20.0 |
| 50° | 1.42 | 10.56 | 18.63 | 2.7 | 11.8 | 18.7 |
| 60° | 1.56 | 10.74 | 17.68 | 3.1 | 10.8 | 17.8 |
| 70° | 0.85 | 10.17 | 16.50 | 3.0 | 10.8 | 17.2 |
| 80° | 0.72 | 11.52 | 16.30 | 3.6 | 10.8 | 16.9 |
| 90° | 0.72 | 11.52 | 16.30 | 0.4 | 10.8 | 16.8 |

所有数值单位均为 dB。具体计算步骤为：

1. 使用第 3.1 节得到的 LOS/NLOS 状态；
2. 将实际仰角映射到最近的参考仰角；
3. 按环境、频段、状态和参考仰角查得 \(\sigma_{\mathrm{SF}}\)；NLOS 同时查得 \(CL\)，LOS 令 \(CL=0\)；
4. 生成 \(Z_{\mathrm{SF}}\sim\mathcal N(0,1)\)，定义附加损耗符号下

\[
SF_{\mathrm{loss}}
=\sigma_{\mathrm{SF}}Z_{\mathrm{SF}};
\]

5. 计算

\[
PL_b
=FSPL+CL+SF_{\mathrm{loss}}.
\]

\(SF_{\mathrm{loss}}\) 可以为负，表示该位置比中位路径损耗更有利。TR 38.901 的相关大尺度参数生成过程常使用“正 \(SF\) 表示接收功率增加”的增益符号；若与第 5.2 节的相关矩阵共同实现，应令

\[
SF_{\mathrm{gain}}=-SF_{\mathrm{loss}},
\]

并在整个程序中只保留一种符号约定。

#### 3.2.2 基本路径损耗例子

设 LEO 高度 \(h_0=600\ \mathrm{km}\)、仰角 \(30^\circ\)、载频 \(2\ \mathrm{GHz}\)、城市环境。由第 1.2 节得到：

\[
d=1075.09\ \mathrm{km},
\qquad
\tau_{\mathrm{geo}}=3.586\ \mathrm{ms}.
\]

自由空间路径损耗为：

\[
\begin{aligned}
FSPL
&=92.45+20\log_{10}2
+20\log_{10}(1075.09)\\
&=159.10\ \mathrm{dB}.
\end{aligned}
\]

若第 3.1 节抽到 NLOS，则城市 S 波段 \(30^\circ\) 表项给出：

\[
CL=29.0\ \mathrm{dB},
\qquad
\sigma_{\mathrm{SF}}=6\ \mathrm{dB}.
\]

若本次 \(Z_{\mathrm{SF}}=0.7\)，则：

\[
SF_{\mathrm{loss}}=4.2\ \mathrm{dB},
\qquad
PL_b=159.10+29.0+4.2=192.30\ \mathrm{dB}.
\]

需要区分杂波损耗和局部快衰落：前者是大尺度平均附加衰减，后者描述多径相干叠加造成的快速幅相变化。

> **关键原文定位：**TR 38.811 Clause 6.6.2、式 (6.6-2) 至 (6.6-4)；Tables 6.6.2-1 至 6.6.2-3 给出不同环境、频段、LOS/NLOS 与参考仰角下的参数。

### 3.3 建筑进入损耗

建筑进入损耗（Building Entry Loss，BEL）描述室外传播路径进入建筑后产生的附加损耗。TR 38.811 的卫星基准场景只考虑室外 UE；Table 6.10.1-1 中 D1～D4 的 O2I 均为 No，只有 D5 HAPS 场景为 Possible。因此，BEL 是卫星/HAPS 共用框架中的可选组件，不是每条卫星链路都要加入。

给定建筑类型、频率 \(f\)、建筑立面处仰角 \(\theta\) 和不超过概率 \(P\)，BEL 分位数为：

\[
L_{\mathrm{BEL}}(P)
=10\log_{10}
\left(
10^{0.1A(P)}
+10^{0.1B(P)}
+10^{0.1C}
\right),
\]

\[
\begin{aligned}
A(P)&=F^{-1}(P)\sigma_1+\mu_1,\\
B(P)&=F^{-1}(P)\sigma_2+\mu_2,\\
C&=-3,\\
\mu_1&=L_h+L_e,\\
L_h&=r+s\log_{10}f+t(\log_{10}f)^2,\\
L_e&=0.212|\theta|,\\
\sigma_1&=u+v\log_{10}f,\\
\mu_2&=w+x\log_{10}f,\\
\sigma_2&=y+z\log_{10}f.
\end{aligned}
\]

其中 \(f\) 的单位为 GHz，\(\theta\) 的单位为度，\(F^{-1}\) 是标准正态分布的逆 CDF。系数为：

| 建筑类型 | \(r\) | \(s\) | \(t\) | \(u\) | \(v\) | \(w\) | \(x\) | \(y\) | \(z\) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 传统建筑 | 12.64 | 3.72 | 0.96 | 9.6 | 2.0 | 9.1 | -3.0 | 4.5 | -2.0 |
| 高热效率建筑 | 28.19 | -3.00 | 8.48 | 13.5 | 3.8 | 27.8 | -2.9 | 9.4 | -2.1 |

\(P\) 表示“损耗不超过 \(L_{\mathrm{BEL}}(P)\) 的概率”，不是 UE 在室内的概率、LOS 概率或通信成功概率。典型使用方式为：

- 中位损耗：固定 \(P=0.5\)；
- 保守链路预算：根据目标覆盖分位数取 \(P=0.9\) 或 \(0.95\)；
- drop-based Monte Carlo：对每个室内 UE 生成 \(U\sim\mathcal U(0,1)\)，再令 \(P=U\)；
- 连续移动：不能在每个时间采样点独立重抽 \(P\)，需要额外的空间相关模型。

例如 \(f=2\ \mathrm{GHz}\)、\(\theta=30^\circ\)、\(P=0.5\) 时，\(F^{-1}(0.5)=0\)。传统建筑得到：

\[
\begin{aligned}
L_h&=13.85\ \mathrm{dB},&
L_e&=6.36\ \mathrm{dB},\\
A&=20.21\ \mathrm{dB},&
B&=8.20\ \mathrm{dB},
\end{aligned}
\]

\[
L_{\mathrm{BEL},50\%}=20.49\ \mathrm{dB}.
\]

相同频率和仰角下，高热效率建筑得到：

\[
L_{\mathrm{BEL},50\%}=35.13\ \mathrm{dB}.
\]

\(A\)、\(B\)、\(C\) 不能直接在 dB 域相加；式 (6.6-5) 要求先转到线性功率域求和。

> **关键原文定位：**TR 38.811 Clause 6.6.3、式 (6.6-5) 至 (6.6-7)、Table 6.6.3-1；Table 6.10.1-1 规定 D1～D4 不使用 O2I，D5 HAPS 可使用。

### 3.4 大气气体吸收

大气气体吸收主要由氧气和水汽引起，依赖频率、仰角、海拔、气压、温度和水汽密度。TR 38.811 的判断准则为：

- \(f<10\ \mathrm{GHz}\) 时通常可忽略；
- 若仰角低于 \(10^\circ\)，则 \(f>1\ \mathrm{GHz}\) 时也建议计算；
- 系统级基线采用 ITU-R P.676 Annex 2 的近似方法；
- UE 海拔高于 10 km，或载频距离吸收谱线中心小于 0.5 GHz 时，不使用该简化基线，应采用更完整的方法。

系统级参考大气取海平面和年平均全球大气：

\[
T=288.15\ \mathrm K,\qquad
p=1013.25\ \mathrm{hPa},\qquad
\rho=7.5\ \mathrm{g/m^3},
\]

对应水汽分压约为：

\[
e\approx 9.98\ \mathrm{hPa}.
\]

先由 ITU-R P.676 得到目标频率的天顶衰减 \(A_{\mathrm{zenith}}(f)\)，再按倾斜路径近似：

\[
PL_g(\alpha,f)
=\frac{A_{\mathrm{zenith}}(f)}{\sin\alpha}.
\]

例如，若某频率处天顶衰减为 \(0.5\ \mathrm{dB}\)，则 \(30^\circ\) 仰角下：

\[
PL_g=\frac{0.5}{\sin30^\circ}=1.0\ \mathrm{dB}.
\]

该 \(1/\sin\alpha\) 关系解释了低仰角为何更敏感，但它不能替代吸收谱线附近的完整大气积分。

> **关键原文定位：**TR 38.811 Clause 6.6.4、式 (6.6-8)；天顶衰减与完整/近似算法来自 ITU-R P.676。

### 3.5 雨衰与云衰

TR 38.811 认为 \(6\ \mathrm{GHz}\) 以下的雨云衰减可忽略。系统级默认采用晴空条件，即：

\[
A_{\mathrm{rain}}=A_{\mathrm{cloud}}=0.
\]

若研究恶劣天气、链路可用率或 Ka 波段覆盖，则：

1. 根据 UE 地理位置、频率、仰角、极化和目标时间百分比，分别由 ITU-R P.618 与 ITU-R P.840 得到雨衰和云衰的 CDF；
2. 对每个 drop 和每个 UE 生成 \(U\sim\mathcal U(0,1)\)；
3. 通过逆 CDF 得到该 UE 的 \(A_{\mathrm{rain}}\) 与 \(A_{\mathrm{cloud}}\)；
4. TR 38.811 的简化 drop 流程不考虑不同 UE 之间的雨云空间相关性。

雨衰和云衰如果已经作为可用率余量计入系统尺寸设计或链路预算，就不应在信道实现中再次叠加。

> **关键原文定位：**TR 38.811 Clause 6.6.5；Table 6.10.1-1 Note 3 指定雨衰使用 ITU-R P.618、云衰使用 ITU-R P.840，并要求避免与系统尺寸设计重复计算。

### 3.6 电离层与对流层闪烁

闪烁是接收信号幅度和相位的快速波动。TR 38.811 在系统级路径损耗中主要把它简化为一个额外幅度衰减 \(PL_s\)，并未给出完整的复数时域闪烁生成器。

#### 3.6.1 闪烁指标与频率缩放

幅度闪烁指数为：

\[
S_4
=\frac{\sqrt{\mathbb E[I^2]-\mathbb E[I]^2}}
{\mathbb E[I]},
\]

其中 \(I\) 是接收强度。相位闪烁指数为：

\[
\sigma_\phi
=\sqrt{\mathbb E[\phi^2]-\mathbb E[\phi]^2}.
\]

两者通常在约 60 s 的窗口内统计。弱、中、强闪烁的参考范围为：

| 闪烁等级 | 幅度指标 | 相位指标 |
|---|---|---|
| 弱 | \(S_4<0.3\) | \(\sigma_\phi<0.25\sim0.3\ \mathrm{rad}\) |
| 中 | \(0.3\leq S_4\leq0.6\) | \(0.25\sim0.3\leq\sigma_\phi\leq0.5\sim0.7\ \mathrm{rad}\) |
| 强 | \(S_4>0.6\) | \(\sigma_\phi>0.5\sim0.7\ \mathrm{rad}\) |

从参考频率 \(f_1\) 缩放到 \(f_2\) 时：

\[
S_{4,f_2}
=S_{4,f_1}
\left(\frac{f_2}{f_1}\right)^{-n},
\qquad n=1.5\ \text{（L 波段推荐）},
\]

\[
\sigma_{\phi,f_2}
=\sigma_{\phi,f_1}
\left(\frac{f_2}{f_1}\right)^{-n},
\qquad n=1\ \text{（L 波段推荐）}.
\]

该幂律主要适用于弱散射、较高仰角和 \(S_4<0.6\) 的弱到中等闪烁；强闪烁饱和时 \(n\rightarrow0\)。

#### 3.6.2 电离层闪烁损耗

电离层传播在系统级基线中用于 \(f<6\ \mathrm{GHz}\)。Gigahertz 闪烁模型的计算流程为：

1. 从 4 GHz、相应季节和太阳活动曲线读取峰峰值波动 \(P_{\mathrm{fluc},4}\)；
2. 缩放到目标频率：

\[
P_{\mathrm{fluc}}(f)
=P_{\mathrm{fluc},4}
\left(\frac f4\right)^{-1.5};
\]

3. 将峰峰值换成链路预算衰减：

\[
A_{\mathrm{IS}}
=\frac{P_{\mathrm{fluc}}(f)}{\sqrt{2}}.
\]

按地磁纬度绝对值 \(|\lambda_m|\) 选择：

\[
PL_s=
\begin{cases}
A_{\mathrm{IS}}, & |\lambda_m|\leq20^\circ,\\
0, & 20^\circ<|\lambda_m|\leq60^\circ,\\
0\ \text{（仅指幅度损耗基线）}, & |\lambda_m|>60^\circ.
\end{cases}
\]

\(|\lambda_m|\) 是地磁纬度绝对值，不是波长或特征值。高纬度置 \(PL_s=0\) 只表示该基线忽略附加幅度损耗；高纬地区仍可能存在明显相位闪烁。

系统级默认在 \(|\lambda_m|\leq20^\circ\) 的区域采用 P3 曲线 99% 时间点。若 4 GHz 曲线上读得 \(1.1\ \mathrm{dB}\)，缩放到 2 GHz：

\[
A_{\mathrm{IS}}
=1.1\left(\frac24\right)^{-1.5}\frac1{\sqrt2}
=2.2\ \mathrm{dB}.
\]

#### 3.6.3 对流层闪烁损耗

对流层闪烁在系统级基线中用于 \(f>6\ \mathrm{GHz}\)，尤其在 10 GHz 以上和低仰角时更显著。TR 38.811 给出的 20 GHz、Toulouse、圆极化 99% 参考衰减为：

| 仰角 | \(PL_s\) |
|---:|---:|
| 10° | 1.08 dB |
| 20° | 0.48 dB |
| 30° | 0.30 dB |
| 40° | 0.22 dB |
| 50° | 0.17 dB |
| 60° | 0.13 dB |
| 70° | 0.12 dB |
| 80° | 0.12 dB |
| 90° | 0.12 dB |

系统级可固定使用对应仰角的 99% 值，也可以按 Figure 6.6.6.2.1-1 的 CDF 为每个 UE 抽样；基线假设不同 UE 的闪烁相互独立。

#### 3.6.4 复信道实现边界

若要研究闪烁对跟踪环、信道估计或相干合并的影响，应使用：

\[
h_{\mathrm{sci}}(t)
=a_{\mathrm{sci}}(t)e^{j\phi_{\mathrm{sci}}(t)},
\qquad
a_{\mathrm{sci}}(t)
=10^{-L_{\mathrm{sci}}(t)/20}.
\]

幅度项会造成瞬时 SNR 波动、深衰落和中断；相位项会造成星座旋转与扩散、信道估计老化和相干合并损失。但 TR 38.811 只给出 \(S_4\)、\(\sigma_\phi\)、频率缩放和幅度余量，没有给出相位过程的功率谱、相关时间或完整时序生成算法。因此仅按报告基线实现时，应把 \(PL_s\) 解释为闪烁衰落余量，而不是完整的时变相位噪声。

> **关键原文定位：**TR 38.811 Clauses 6.6.6.1.1-6.6.6.1.4、式 (6.6-9) 至 (6.6-14)，以及 Clause 6.6.6.2.1、Table 6.6.6.2.1-1。

### 3.7 法拉第旋转

法拉第旋转描述线极化电磁波穿过地磁场中的电离介质后，极化方向发生旋转。TR 38.811 在每条子径的双极化耦合中加入：

\[
\mathbf F_r(\psi)
=
\begin{bmatrix}
\cos\psi & -\sin\psi\\
\sin\psi & \cos\psi
\end{bmatrix},
\qquad
\psi=\frac{108}{f^2}\ \mathrm{degree},
\]

其中 \(f\) 的单位为 GHz。若发射信号原为第一种线极化：

\[
\mathbf e_{\mathrm{tx}}
=
\begin{bmatrix}1\\0\end{bmatrix},
\qquad
\mathbf e_{\mathrm{rx}}
=
\begin{bmatrix}\cos\psi\\\sin\psi\end{bmatrix}.
\]

原同极化端口获得 \(\cos^2\psi\) 的功率，正交极化端口获得 \(\sin^2\psi\) 的功率。旋转矩阵本身不耗散能量；损失来自接收机只接收一个固定线极化端口。

在 2 GHz：

\[
\psi=27^\circ,
\qquad
L_{\mathrm{pol}}
=-10\log_{10}(\cos^2 27^\circ)
\approx1.0\ \mathrm{dB}.
\]

约 20.6% 的功率进入正交极化端口。在 20 GHz 时 \(\psi=0.27^\circ\)，该效应通常很小。

仿真实现分为两类：

- 双极化 CDL：把 \(\mathbf F_r\) 右乘到每条子径的极化耦合矩阵，再与收发天线场响应组合；
- 单线极化标量模型：近似使用 \(h_{\mathrm{eff}}=h\cos\psi\)，或加入 \(L_{\mathrm{pol}}\)，但无法观察正交端口泄漏。

双极化接收机若同时估计两个端口，通常可以恢复大部分旋转后的能量；圆极化是法拉第介质的本征极化，主要表现为附加相位，因此卫星系统常用圆极化降低法拉第旋转和姿态误差的敏感性。

> **关键原文定位：**TR 38.811 Clause 6.8.2、式 (6.8-1) 至 (6.8-2)；报告要求在 TR 38.901 的极化信道系数生成中右乘法拉第旋转矩阵。

### 3.8 总路径损耗与重复计数边界

对于使用 Clause 6.6 宽带大尺度模型的链路，可按场景写成：

\[
PL
=PL_b
+PL_g
+PL_s
+PL_e
+A_{\mathrm{rain}}
+A_{\mathrm{cloud}}.
\]

其中：

- \(PL_b=FSPL+CL+SF_{\mathrm{loss}}\)；
- \(PL_e=L_{\mathrm{BEL}}\)，仅室内场景启用；
- \(PL_g\) 为气体吸收；
- \(PL_s\) 为电离层或对流层闪烁幅度余量；
- \(A_{\mathrm{rain}}\) 与 \(A_{\mathrm{cloud}}\) 按天气和可用率选择。

接收功率为：

\[
P_{\mathrm{rx}}
=P_{\mathrm{tx}}
+G_{\mathrm{tx}}(\Omega)
+G_{\mathrm{rx}}(\Omega)
-PL
-L_{\mathrm{point}}
-L_{\mathrm{impl}},
\]

其中 \(L_{\mathrm{point}}\) 是指向误差损耗，\(L_{\mathrm{impl}}\) 是馈线、极化失配和实现损耗等未在传播模型中包含的项。

不同快衰落模型的叠加规则为：

| 所用模型 | 基本路径损耗 | 是否另加 \(CL+SF\) | 局部衰落 |
|---|---|---|---|
| Clause 6.6 + CDL/TDL | \(FSPL+CL+SF\) | 已包含，不再另加 | CDL/TDL |
| Clause 6.6 + 单抽头 Rice | \(FSPL+CL+SF\) | 已包含，不再另加 | Rice |
| Clause 6.7.1 ITU 两状态 Loo | 仅 \(FSPL\) | 否；Loo 已含杂波与阴影 | GOOD/BAD Loo |
| 纯链路预算、无随机快衰落 | 按所需分位数装配 | 每项只计一次 | 无或固定余量 |

雨云衰减若已经在系统尺寸设计中计入，同样不再进入 \(PL\)。法拉第旋转若已用双极化矩阵实现，也不应再额外加入同一极化失配损耗。

继续使用第 3.2.2 节的 600 km LEO、2 GHz、城市 NLOS 例子。若 UE 为室外、中纬度、晴空，且按报告准则忽略该频率和仰角下的气体吸收，则：

\[
PL_e=PL_g=PL_s=A_{\mathrm{rain}}=A_{\mathrm{cloud}}=0,
\]

\[
PL=PL_b=192.30\ \mathrm{dB}.
\]

若同一链路位于地磁赤道附近，并采用前述 \(PL_s=2.2\ \mathrm{dB}\) 的闪烁基线，则：

\[
PL=194.50\ \mathrm{dB}.
\]

> **关键原文定位：**TR 38.811 Clause 6.6、式 (6.6-1)；Clause 6.7.1、式 (6.7-1) 规定 ITU 两状态模型只与 FSPL 组合；Table 6.10.1-1 Note 3 规定雨云损耗的重复计数边界。

### 3.9 仰角作为几何与传播的共同控制量

仰角并不是只影响斜距。它同时作用于多个建模层次：

| 仰角变化 | 几何后果 | 传播后果 |
|---|---|---|
| 仰角降低 | 斜距增大，几何时延增大 | FSPL 增大 |
| 仰角降低 | 穿越近地环境的路径更长 | LOS 概率下降、杂波和阴影通常增强 |
| 仰角降低 | 穿越大气的倾斜路径更长 | 气体、雨云和对流层效应增强 |
| 仰角变化 | 径向速度投影改变 | 多普勒及其变化率改变 |
| 仰角变化 | UE 周围可见散射结构改变 | DS、ASA、ZSA 和 K 因子可能改变 |

因此，仰角是把“链路几何”“大尺度传播”和“局部快衰落”连接起来的核心变量。仿真中不应只根据仰角更新天线增益，而保持 LOS 概率、路径损耗和快衰落参数不变。

> **关键原文定位：**TR 38.811 Clause 6.6 的 LOS、杂波、阴影和闪烁参数均显式依赖或按参考仰角查表；Clause 6.7.2 的宽带快衰落参数也按环境、频率与仰角组织。

## 4 模型简化与选择

### 4.1 系统级与链路级建模方法

TR 38.811 Clause 6.5 先区分两类使用方式：

- 系统级采用类似 TR 38.901 的 drop-based 基线流程生成信道系数；若所有 UE 都满足平坦衰落条件，可使用单抽头简化模型；
- 链路级针对给定环境和仰角使用参考 CDL/TDL；其参数来自系统级基线模型的一个代表性实例；
- 平坦衰落或 AWGN 条件下，链路级无需再生成多抽头模型。

模型复杂度应由研究问题决定，而不是默认 CDL 一定“更准确”。若研究链路预算，显式生成簇内子径没有收益；若研究波束、阵列或 MIMO 秩，固定 TDL 又会丢失必要的空间结构。

> **关键原文定位：**TR 38.811 Clauses 6.5.1-6.5.2、Figures 6.5.1-1 和 6.5.1-2；完整基线来自 TR 38.901，简化模型只在所有 UE 满足平坦衰落标准时使用。

### 4.2 统一宽带 MIMO 基带模型

连续时间宽带 MIMO 模型为：

\[
\mathbf y(t)
=
\int
\mathbf H(t,\tau)
\mathbf x(t-\tau)
\,\mathrm d\tau
+\mathbf n(t).
\]

CDL、TDL 和平坦衰落并不是彼此无关的三套模型，而是对 \(\mathbf H(t,\tau)\) 保留不同层次的信息：

| 模型 | 保留的信息 | 主要丢失的信息 |
|---|---|---|
| CDL | 时延、功率、角度、阵列、极化、子径和多普勒联合结构 | - |
| TDL | 抽头时延、功率、时间统计和频率选择性 | 显式角度、极化来源和簇内子径 |
| 平坦衰落 | 单抽头标量或矩阵复衰落 | 可分辨时延和带内频率选择性 |
| AWGN | 噪声 | 所有随机信道衰落 |

> **关键原文定位：**TR 38.811 Clause 6.5 用完整基线、CDL/TDL 和单抽头分别服务不同层级仿真；TR 38.901 Clause 7.7 给出 CDL/TDL 的通用生成框架。

### 4.3 CDL 到 TDL 的空间滤波与合并

CDL 信道可写为：

\[
\mathbf H_{\mathrm{CDL}}(t,\tau)
=
\sum_n\sum_m
\alpha_{n,m}(t)
\mathbf a_r(\Omega^r_{n,m})
\mathbf a_t^H(\Omega^t_{n,m})
\delta(\tau-\tau_n).
\]

给定发射波束 \(\mathbf w\) 和接收波束 \(\mathbf v\)，有效冲激响应为：

\[
h_{\mathrm{eff}}(t,\tau)
=
\mathbf v^H
\mathbf H_{\mathrm{CDL}}(t,\tau)
\mathbf w.
\]

将同一簇或相近时延内的子径合并：

\[
g_n(t)
=
\sum_m
\alpha_{n,m}(t)
\mathbf v^H\mathbf a_r(\Omega^r_{n,m})
\mathbf a_t^H(\Omega^t_{n,m})\mathbf w,
\]

即可得到 TDL：

\[
h_{\mathrm{TDL}}(t,\tau)
=
\sum_n g_n(t)\delta(\tau-\tau_n).
\]

这个过程固定了天线与波束，并删除显式角度、阵列几何、极化和簇内子径，只保留空间滤波后的抽头时延、功率和时间变化：

\[
\boxed{
\mathrm{CDL}
\xrightarrow{\text{天线与波束空间滤波}}
\text{波束相关的等效 TDL}
}
\]

TDL 到 CDL 的逆过程通常不唯一，因为合并后已无法恢复每个抽头由哪些角度与散射体构成。

> **关键原文定位：**TR 38.811 Clause 6.9.2 说明 NTN-TDL 从 NTN-CDL 经空间滤波得到，基线使用各向同性 UE 天线；通用空间滤波方法参见 TR 38.901 Clauses 7.7.2-7.7.4。

### 4.4 TDL 到平坦衰落的时延合并

TDL 接收模型为：

\[
y(t)
=
\sum_n g_n(t)x(t-\tau_n)+n(t).
\]

若信号带宽远小于有效相干带宽：

\[
B_{\mathrm{sig}}\ll B_c,
\]

或近似满足：

\[
B_{\mathrm{sig}}\sigma_{\tau,\mathrm{eff}}\ll1,
\]

则信号无法分辨抽头间相对时延，\(x(t-\tau_n)\approx x(t-\tau_0)\)。所有抽头合并为：

\[
h_{\mathrm{flat}}(t)=\sum_n g_n(t),
\]

从而：

\[
y(t)
=h_{\mathrm{flat}}(t)x(t-\tau_0)+n(t).
\]

完成公共定时补偿后：

\[
y(t)=h_{\mathrm{flat}}(t)x(t)+n(t).
\]

平坦衰落只表示业务带宽内无法分辨相对时延，不表示没有多径或信道恒为 1。保留多天线端口时，它仍可为矩阵模型：

\[
\mathbf y(t)
=\mathbf H_{\mathrm{flat}}(t)\mathbf x(t)
+\mathbf n(t).
\]

> **关键原文定位：**TR 38.811 Clauses 6.5、6.7 规定只有在满足平坦衰落标准时才能使用单抽头简化；相干带宽取决于环境、天线方向图和仰角。

### 4.5 OFDM 下的三个尺度条件

TDL 采样后为多抽头卷积：

\[
y[q]
=
\sum_{\ell=0}^{L_h-1}
h[q,\ell]x[q-\ell]+n[q].
\]

若循环前缀覆盖有效多径时延，且信道在一个 OFDM 符号内近似不变，则：

\[
Y[k]=H[k]X[k]+N[k],
\]

\[
H[k]
=
\sum_\ell
h[\ell]e^{-j2\pi k\ell/N}.
\]

需要区分三个条件：

| 条件 | 数学判断 | 含义 |
|---|---|---|
| 整体信号平坦 | \(B_{\mathrm{sig}}\ll B_c\) | 全业务带宽近似同一复系数 |
| 子载波内平坦 | \(\Delta f\ll B_c\) | 整体可频率选择性，但每个子载波可单抽头均衡 |
| CP 足够长 | \(T_{\mathrm{CP}}>\tau_{\max}\) | 维持循环卷积并抑制相邻符号干扰 |

例如 20 MHz 信号可能整体频率选择性，但每个 15 kHz 子载波内部仍近似平坦。子载波内单抽头均衡不等于把整个系统简化成平坦衰落。

> **关键原文定位：**TR 38.811 Clause 6.7 的“UE 带宽低于相干带宽”针对整体平坦衰落；OFDM 子载波平坦和 CP 条件属于接收机波形层面的进一步判断。

### 4.6 研究目标与模型选择

| 研究目标 | 推荐快衰落模型 | 主要原因 |
|---|---|---|
| 链路预算、覆盖与可用率 | 单抽头或仅大尺度模型 | 重点是斜距、天线增益、FSPL、阴影和雨衰 |
| OFDM CP、均衡与导频 | TDL | 直接保留时延抽头和频率选择性 |
| 编码、交织与频率分集 | TDL | 易控制 RMS 时延扩展和抽头时间统计 |
| 波束赋形与波束管理 | CDL | 需要显式角度和阵列响应 |
| MIMO 秩与空间相关性 | CDL | 需要簇内角度、极化和空间结构 |
| 固定波束后的接收机测试 | 波束滤波后的等效 TDL | 先从 CDL 得到指定波束下的抽头功率 |
| 经验证的窄带屋顶 VSAT | Ricean 或 ITU 两状态单抽头 | 方向图抑制多径且业务带宽低于相干带宽 |
| 宽带定位或感知 | CDL 或几何宽带模型 | 需要保留时延和角度结构 |

模型选择应遵循“只保留研究问题需要的自由度”。大尺度模型、CDL、TDL 和单抽头可以组合，但不能把描述同一效应的两个模型重复叠加。

> **关键原文定位：**TR 38.811 Clause 6.5 给出系统级完整/简化模型与链路级 CDL/TDL 的分工；Clause 6.7 给出平坦衰落简化边界；Clause 6.9 给出 NTN-CDL/TDL 参考剖面。

## 5 局部快衰落

### 5.1 平坦衰落与 ITU 两状态模型

TR 38.811 Clause 6.7 允许窄带 SISO 在平坦条件成立时采用简化模型，并使用 95% RMS 时延扩展估计相干带宽：

\[
B_c\approx\frac{1}{10\tau_{\mathrm{rms},95\%}}.
\]

报告引用的测量结果表明，对准全向 UE，ITU 两状态模型至少可在以下条件同时满足时作为简化备选：

- S 波段；
- 最低仰角不低于 \(20^\circ\)；
- 准 LOS，衰落余量约不超过 5 dB；
- 信道带宽不超过 5 MHz；
- 农村、郊区或城市环境。

ITU 两状态模型用 GOOD 状态表示 LOS 或轻微遮挡，用 BAD 状态表示严重遮挡；状态持续距离采用半马尔可夫过程。与第 3.1 节的宽带模型不同，选择两状态平坦模型后，不再使用 Table 6.6.1-1 的 LOS 概率，而是由该模型自身的 GOOD/BAD 状态概率和持续距离控制传播状态。

每个状态内部采用 Loo 分布：

\[
h(t)
=A(t)e^{j\phi_D(t)}+g(t),
\]

其中：

- \(A(t)e^{j\phi_D(t)}\) 为直达或准直达分量；
- \(20\log_{10}A\) 在 dB 域服从正态分布，因此 \(A\) 在线性域为对数正态；
- \(g(t)=g_I(t)+jg_Q(t)\) 为局部弥散多径，通常令

\[
g_I,g_Q\sim\mathcal N(0,\sigma_g^2),
\qquad
\mathbb E[|g|^2]=2\sigma_g^2.
\]

Loo 模型由三个量决定：

| 参数 | 物理含义 | 主要作用 |
|---|---|---|
| \(M_A\) | 一个状态事件内直达分量的平均 dB 电平 | 决定直达分量典型强度 |
| \(\Sigma_A\) | 直达分量 dB 电平的标准差 | 描述树木、建筑等对直达径的慢阴影 |
| \(MP\) | 弥散多径平均功率，dB | 决定随机多径背景强度 |

在一个足够短的局部时间内固定 \(A=a\) 后：

\[
h(t)\mid A=a
=ae^{j\phi_D(t)}+g(t)
\]

是普通 Rice 信道，其瞬时 K 因子为：

\[
K=\frac{a^2}{2\sigma_g^2},
\qquad
K_{\mathrm{dB}}
=P_{D,\mathrm{dB}}-MP_{\mathrm{dB}}.
\]

更长时间观察时，\(A\) 又受对数正态阴影调制，因此：

\[
p_R(r)
=\int_0^\infty
p_{\mathrm{Rice}}(r\mid A=a)
p_{\mathrm{LN}}(a)\,\mathrm da.
\]

因此 Loo 可以理解为“直达分量由对数正态过程调制的 Rice 信道”。GOOD 与 BAD 状态使用不同的 \(M_A,\Sigma_A,MP\)：GOOD 通常直达分量更强、K 因子更大；BAD 中直达分量被严重遮挡，信道可能接近 Rayleigh。

drop-based 短时生成步骤为：

1. 设定载频、LMS 环境、最近可用仰角、UE 位置、姿态和速度；
2. 从 ITU-R P.681 参数表读取 GOOD/BAD 的状态持续距离参数、\(M_A\) 分布参数以及线性关系系数；
3. 在选定状态 \(i\in\{G,B\}\) 内抽取 \(M_{A,i}\)；
4. 计算

\[
\Sigma_{A,i}=g_{1,i}M_{A,i}+g_{2,i},
\qquad
MP_i=h_{1,i}M_{A,i}+h_{2,i};
\]

5. 再抽取直达分量电平 \(P_D\sim\mathcal N(M_{A,i},\Sigma_{A,i}^2)\)，并计算

\[
K_{\mathrm{dB}}=P_D-MP_i;
\]

6. 用该 K 因子生成单抽头 Rice 快衰落。

这一短时流程只适用于持续几个 TTI、状态不切换且 K 因子近似不变的仿真。长时间演化见第 5.8 节。

该模型已经包含杂波损耗和阴影衰落，因此基础路径损耗只使用：

\[
PL_b=FSPL(d,f_c).
\]

若再叠加 Clause 6.6.2 的 \(CL+SF\)，会重复计数。

> **关键原文定位：**TR 38.811 Clauses 6.7、6.7.1，式 (6.7-1)、(6.7-6) 至 (6.7-8)；两状态参数来源为 ITU-R P.681。平坦条件取决于环境、天线方向图和仰角，不是只由带宽决定。

### 5.2 相关大尺度参数与 NTN-CDL 空间结构

TR 38.811 Clause 6.7.2 的“大尺度参数”（Large-Scale Parameters，LSP）与第 3 节的总路径损耗相关，但作用不同：

| 参数 | 决定的信道性质 |
|---|---|
| \(SF\) | 局部平均接收功率相对中位路径损耗的偏移 |
| \(DS\) | 局部功率时延谱的 RMS 宽度 |
| \(ASA,ASD\) | 到达/出发方位角扩展 |
| \(ZSA,ZSD\) | 到达/出发天顶角扩展 |
| \(K\) | LOS 确定性功率与弥散功率之比 |

这些量不是每条子径的参数，而是先生成簇和子径所需的一组相关统计尺度。参数表按环境、LOS/NLOS、S/Ka 频段和 \(10^\circ\) 步进仰角给出：

\[
\mu_{\lg X}=\mathbb E[\log_{10}X],
\qquad
\sigma_{\lg X}
=\operatorname{std}[\log_{10}X].
\]

#### 5.2.1 相关 LSP 抽样

设表中需要联合生成的高斯变量向量为：

\[
\mathbf z
=
\begin{bmatrix}
z_{\mathrm{SF}}&
z_K&
z_{\mathrm{DS}}&
z_{\mathrm{ASD}}&
z_{\mathrm{ASA}}&
z_{\mathrm{ZSD}}&
z_{\mathrm{ZSA}}
\end{bmatrix}^{T}.
\]

按对应表格中的交叉相关系数组成相关矩阵 \(\mathbf C\)，求分解：

\[
\mathbf C=\mathbf L\mathbf L^T.
\]

先生成独立标准高斯向量：

\[
\mathbf u\sim\mathcal N(\mathbf 0,\mathbf I),
\]

再令：

\[
\mathbf z=\mathbf L\mathbf u.
\]

这样才能保留例如 DS 与 K、ASA 与 SF 之间的统计相关性。随后计算：

\[
\begin{aligned}
DS&=10^{\mu_{\lg DS}+\sigma_{\lg DS}z_{\mathrm{DS}}}\ \mathrm s,\\
ASA&=10^{\mu_{\lg ASA}+\sigma_{\lg ASA}z_{\mathrm{ASA}}}\ \mathrm{degree},\\
ZSA&=10^{\mu_{\lg ZSA}+\sigma_{\lg ZSA}z_{\mathrm{ZSA}}}\ \mathrm{degree},\\
K_{\mathrm{dB}}&=\mu_K+\sigma_K z_K,\\
SF_{\mathrm{gain}}&=\sigma_{\mathrm{SF}}z_{\mathrm{SF}}.
\end{aligned}
\]

ASD 和 ZSD 在需要时同样由对数正态式生成。生成后应按 TR 38.901 的上限约束：

\[
\mathrm{ASA}=\min(\mathrm{ASA},104^\circ),\quad
\mathrm{ASD}=\min(\mathrm{ASD},104^\circ),
\]

\[
\mathrm{ZSA}=\min(\mathrm{ZSA},52^\circ),\quad
\mathrm{ZSD}=\min(\mathrm{ZSD},52^\circ),
\]

随后再按角度域规则截断或折返。

若第 3.2 节已经独立生成了 \(SF_{\mathrm{loss}}\)，这里不能再生成第二个 SF。完整 GSCM 应在本步骤一次性生成相关 LSP，并使用：

\[
SF_{\mathrm{loss}}=-SF_{\mathrm{gain}}
\]

进入路径损耗。只研究固定链路级 CDL/TDL 时，也可以直接指定 \(DS,ASA,ZSA,K\)，不再随机抽取。

以城市、LOS、S 波段、\(30^\circ\) 为例，Table 6.7.2-3a 给出：

\[
\begin{aligned}
\mu_{\lg DS}&=-8.21,& \sigma_{\lg DS}&=0.68,\\
\mu_{\lg ASA}&=0.41,& \sigma_{\lg ASA}&=1.30,\\
\mu_{\lg ZSA}&=0.54,& \sigma_{\lg ZSA}&=1.10,\\
\mu_K&=10.49\ \mathrm{dB},&
\sigma_K&=10.42\ \mathrm{dB}.
\end{aligned}
\]

若只观察各变量的中位值，即令相应 \(z=0\)，则：

\[
DS_{\mathrm{median}}
=10^{-8.21}\ \mathrm s
=6.17\ \mathrm{ns},
\]

\[
ASA_{\mathrm{median}}=10^{0.41}=2.57^\circ,
\qquad
ZSA_{\mathrm{median}}=10^{0.54}=3.47^\circ,
\]

\[
K_{\mathrm{median}}=10.49\ \mathrm{dB}.
\]

若 \(z_{\mathrm{DS}}=0.5\)，则本次：

\[
DS=10^{-8.21+0.68\times0.5}\ \mathrm s
=13.49\ \mathrm{ns}.
\]

这个例子只演示从表格参数到实际 LSP 的变换；正式联合抽样必须使用相关矩阵，而不能让所有 \(z\) 独立。

#### 5.2.2 卫星侧与 UE 侧角度结构

TR 38.811 Clause 6.9.1 在 TR 38.901 的 CDL 生成方法上给出 NTN 专用剖面。卫星到 UE 的距离远大于 UE 周围局部散射区尺寸，因此从卫星侧观察，各簇方向非常接近。对 GEO/LEO 卫星，报告把方位出发角扩展 ASD 和天顶出发角扩展 ZSD 置为零；UE 一侧仍保留 AOA、ZOA、ASA、ZSA 与 DS。HAPS 更接近地面，若采用 TR 38.901 面板及相应参数，则不能自动套用“所有平台侧角扩展均为零”的卫星简化。

这种非对称性可概括为：

\[
\boxed{
\text{平台侧方向集中}
\quad+\quad
\text{UE侧局部散射展开}
}
\]

CDL 基带信道为：

\[
\mathbf H_{\mathrm{CDL}}(t,\tau)
=
\sum_{n=1}^{N_{\mathrm{cl}}}
\sum_{m=1}^{M_n}
\alpha_{n,m}(t)
\mathbf a_r(\Omega^r_{n,m})
\mathbf a_t^H(\Omega^t_{n,m})
\delta(\tau-\tau_n).
\]

同一子径同时携带时延、出发/到达角、多普勒、极化和随机相位，所以 CDL 保留这些参数的联合结构，适合研究波束、MIMO 秩、空间相关性和宽带阵列信道估计。

> **关键原文定位：**TR 38.811 Clause 6.7.2 的 Tables 6.7.2-1 至 6.7.2-8 给出 LSP 分布、交叉相关、簇数和相关距离；Clauses 6.9.1 给出 NTN-CDL-A 至 D；具体簇系数生成继承 TR 38.901 Clause 7.5 与 7.7.1。

### 5.3 簇与子径的物理含义

一个簇表示一组传播时延、到达角、出发角和多普勒相近的有效路径。它们经常来自同一片或相邻散射区域，例如一栋建筑的立面、屋顶边缘、窗口及附近附属结构。

建筑遮挡本身更接近 LOS/NLOS 状态：它可能削弱或移除 LOS；建筑立面和边缘才进一步产生反射、绕射与散射路径。一个建筑可能形成一个簇，也可能形成多个可分辨簇；一个统计簇也可能汇总多个无法逐一辨认的物理散射过程。

子径是簇内不同的有效散射分量，可能对应不同散射点，也可能是大量微小散射的等效射线。簇内子径经常共用簇代表时延 \(\tau_n\) 和簇总平均功率 \(P_n\)，再在子径间分配功率并赋予不同角度、相位、极化和多普勒。其依据是：

\[
\Delta\tau_{\mathrm{intra-cluster}}
\ll
\Delta\tau_{\mathrm{inter-cluster}},
\]

且簇内差异小于模型或系统的时延分辨率。“共用”是模型分辨率下的簇级简化，不表示所有物理子径具有完全相同的真实时延与瞬时功率。

> **关键原文定位：**TR 38.901 Clause 7.7.1 的 CDL 生成过程使用 cluster 与 ray 的两级结构；TR 38.811 Clause 6.9.1 在此基础上给出 NTN 专用角度和时延剖面。

### 5.4 NTN-TDL 与抽头模型

TR 38.811 Clause 6.9.2 给出 NTN-TDL-A 至 NTN-TDL-D。它们由 NTN-CDL 经空间滤波得到，基线采用各向同性 UE 天线，省略显式角度和簇内子径，只保留抽头时延、平均功率、衰落分布和时间统计。

SISO TDL 为：

\[
h_{\mathrm{TDL}}(t,\tau)
=
\sum_{p=1}^{P}
g_p(t)\delta(\tau-\tau_p),
\]

\[
y(t)
=
\sum_{p=1}^{P}
g_p(t)x(t-\tau_p)+n(t).
\]

频域响应为：

\[
H_{\mathrm{TDL}}(t,f)
=
\sum_{p=1}^{P}
g_p(t)e^{-j2\pi f\tau_p}.
\]

只要存在多个可分辨抽头，TDL 就会形成频率选择性。MIMO-TDL 还可为每个抽头指定空间相关矩阵，但不能唯一解释相关性来自哪些物理角度和散射体。

若实际 UE 使用方向性天线或改变波束，固定 NTN-TDL 的抽头功率不会自动变化。此时应先在 CDL 或几何路径模型中应用实际方向图，再生成新的等效 TDL。

> **关键原文定位：**TR 38.811 Clause 6.9.2 给出 NTN-TDL-A 至 D，并说明其空间滤波来源；TR 38.901 Clauses 7.7.2-7.7.4 给出通用 TDL 和空间滤波方法。

### 5.5 归一化时延与实际超额时延

CDL/TDL 表格中的归一化时延 \(\widehat\tau_p\) 不是纳秒、微秒或毫秒，而是功率时延谱的无量纲形状模板。令归一化功率为：

\[
p_p=\frac{P_p}{\sum_iP_i},
\]

功率加权平均归一化时延为：

\[
\overline{\widehat\tau}
=\sum_p p_p\widehat\tau_p.
\]

若剖面归一化为单位 RMS 时延扩展，则：

\[
\sqrt{
\sum_p p_p
\left(
\widehat\tau_p-\overline{\widehat\tau}
\right)^2
}=1.
\]

选择目标 RMS 时延扩展 \(DS_{\mathrm{target}}\) 后，实际局部超额时延为：

\[
\tau_{p,\mathrm{excess}}
=\widehat\tau_p DS_{\mathrm{target}}.
\]

例如 \(\widehat\tau_p=2.5\)、\(DS_{\mathrm{target}}=100\ \mathrm{ns}\)，则该抽头局部超额时延为 \(250\ \mathrm{ns}\)。这仍不包含第 1.2 节中的毫秒级几何传播时延。

> **关键原文定位：**TR 38.811 Clauses 6.9.1-6.9.2 给出归一化 NTN-CDL/TDL 剖面；TR 38.901 Clause 7.7.3 规定按目标 RMS delay spread 缩放归一化抽头时延。

### 5.6 NTN-A～D 参考剖面

“剖面”表示一组标准化的平均功率—时延结构；同字母 CDL 与 TDL 共享相应的簇/抽头功率—时延骨架，CDL 还保留角度、极化和簇内子径。它不是地形剖面，也不是一次瞬时信道实现。

| 剖面 | 状态 | 归一化时延与平均功率 | 衰落分布 |
|---|---|---|---|
| NTN-TDL-A | NLOS | \((0,0)\)、\((1.0811,-4.675)\)、\((2.8416,-6.482)\) | 3 个 Rayleigh 抽头 |
| NTN-TDL-B | NLOS | \((0,0)\)、\((0.7249,-1.973)\)、\((0.7410,-4.332)\)、\((5.7392,-11.914)\) | 4 个 Rayleigh 抽头 |
| NTN-TDL-C | LOS | 首抽头总功率 0 dB，\(K_1=10.224\ \mathrm{dB}\)；尾径 \((14.8124,-23.373)\) | Rice 首抽头 + 1 个 Rayleigh 尾径 |
| NTN-TDL-D | LOS | 首抽头总功率 0 dB，\(K_1=11.707\ \mathrm{dB}\)；后径 \((0.5596,-9.887)\)、\((7.3340,-16.771)\) | Rice 首抽头 + 2 个 Rayleigh 后径 |

二元组表示：

\[
(\text{归一化时延},\ \text{平均功率/dB}).
\]

LOS 首抽头在表中进一步拆为同一时延上的镜面分量和弥散分量：

| 剖面 | 镜面分量 | 同时延弥散分量 | 合成 K 因子 |
|---|---:|---:|---:|
| C | -0.394 dB | -10.618 dB | 10.224 dB |
| D | -0.284 dB | -11.991 dB | 11.707 dB |

例如 C 的第一抽头总功率为：

\[
10\log_{10}
\left(
10^{-0.394/10}
+10^{-10.618/10}
\right)
\approx0\ \mathrm{dB}.
\]

因此不能把“镜面功率 -0.394 dB”和“首抽头平均功率 0 dB”理解为两个不相干的抽头；它们位于同一时延并共同构成 Rice 首抽头。

A/B 是两种不同形状的 NLOS 剖面，不表示 A 优于 B；C/D 是两种不同形状的 LOS 剖面，也没有固定的“城市/农村”一一映射。TR 38.811 的 NTN-A～D 也不能按字母直接等同于 TR 38.901 的通用 CDL/TDL-A～E。

> **关键原文定位：**TR 38.811 Clauses 6.9.1-6.9.2，Tables 6.9.1-1 至 6.9.1-4、Tables 6.9.2-1 至 6.9.2-4。

### 5.7 参考剖面的链路适配

NTN-CDL-A～D 的角度表以 \(50^\circ\) 仰角为参考。把参考剖面用于目标链路时，需要依次适配时延、角度、K 因子和多普勒。

#### 5.7.1 时延缩放

按第 5.5 节：

\[
\tau_{p,\mathrm{excess}}
=\widehat\tau_p DS_{\mathrm{target}}.
\]

例如 \(DS_{\mathrm{target}}=100\ \mathrm{ns}\) 时，NTN-TDL-A 的三个实际超额时延为：

\[
0,\qquad108.11\ \mathrm{ns},\qquad284.16\ \mathrm{ns}.
\]

#### 5.7.2 出发角与到达角适配

目标仰角为 \(\alpha_{\mathrm{desired}}\) 时，卫星侧各簇 ZOD 设为：

\[
\theta_{n,\mathrm{ZOD,desired}}
=90^\circ+\alpha_{\mathrm{desired}}.
\]

参考剖面 \(\alpha_{\mathrm{model}}=50^\circ\)，因此参考 ZOD 为 \(140^\circ\)。UE 侧 ZOA 按目标 ZSA 和平均方向缩放：

\[
\theta_{n,\mathrm{ZOA,scaled}}
=
\frac{ZSA_{\mathrm{desired}}}{ZSA_{\mathrm{model}}}
\left(
\theta_{n,\mathrm{ZOA,model}}
-\mu_{\mathrm{ZOA,model}}
\right)
+\mu_{\mathrm{ZOA,desired}}
-\Delta\alpha,
\]

\[
\Delta\alpha
=\alpha_{\mathrm{desired}}-\alpha_{\mathrm{model}}.
\]

缩放结果若超出天顶角定义域 \([0^\circ,180^\circ]\)，应按 TR 38.901 的折返规则映射回定义域。AOA 可用同类原则缩放到目标 ASA。

#### 5.7.3 K 因子与多普勒适配

LOS 的 NTN-CDL/TDL-C、D 可以按 TR 38.901 Clause 7.7.6 调整到目标 K 因子。公共卫星多普勒必须作用于所有簇或所有抽头：

\[
g_p(t)
\rightarrow
g_p(t)
\exp\left(
j2\pi\int_{t_0}^{t}
\nu_{\mathrm{platform}}(\tilde t)\,\mathrm d\tilde t
\right).
\]

局部 UE 运动和散射产生的多普勒扩展仍保留在每个 \(g_p(t)\) 内。因而最终频谱是“公共卫星频移 + 其周围的局部多普勒扩展”，不是只给 LOS 抽头加一个频偏。

> **关键原文定位：**TR 38.811 Clause 6.9.1、式 (6.9-1) 至 (6.9-2) 给出目标仰角和 ZOA/ZOD 适配；Clause 6.9.2 规定时延、K 因子缩放并把卫星附加多普勒施加到所有 TDL 抽头。

### 5.8 短时与长时信道演化

TR 38.811 用“几个 TTI”描述可以冻结状态、速度和仰角的短时范围，但没有给出一个固定毫秒数。传输时间间隔（Transmission Time Interval，TTI）在 NR 中可对应 slot 或 mini-slot。普通循环前缀下，一个 14 符号 slot 的典型时长为：

| 子载波间隔 | 参数 \(\mu\) | slot 时长 |
|---:|---:|---:|
| 15 kHz | 0 | 1 ms |
| 30 kHz | 1 | 0.5 ms |
| 60 kHz | 2 | 0.25 ms |
| 120 kHz | 3 | 0.125 ms |

\[
T_{\mathrm{slot}}=\frac{1\ \mathrm{ms}}{2^\mu}.
\]

mini-slot 还可以只占 2、4 或 7 个 OFDM 符号。因此，“几个 TTI”应理解为亚毫秒到几毫秒量级的局部仿真假设，而不是协议规定的统一时间阈值。真正的判断是：这段时间内状态、K 因子、仰角、速度投影和大尺度参数能否近似不变。

#### 5.8.1 短时抽头过程

在经典各向同性散射假设下，Rayleigh 抽头的时间自相关函数常写为：

\[
R_g(\Delta t)
=J_0(2\pi f_D\Delta t).
\]

Ricean LOS 抽头可以表示为：

\[
g_0(t)
=
\sqrt{\frac{K}{K+1}}
e^{j\phi_{\mathrm{LOS}}(t)}
+
\sqrt{\frac{1}{K+1}}
g_{\mathrm{diff}}(t).
\]

短时仿真可以固定 \(K\)、仰角和多普勒值，但快衰落样本和多普勒相位仍随时间变化。“几何参数固定”不等于“信道相位固定”。

#### 5.8.2 长时参数更新

| 信道量 | 几个 TTI 的短时仿真 | 长时间仿真 |
|---|---|---|
| GOOD/BAD 或 LOS/NLOS 状态 | 固定 | 按持续距离和状态过程切换 |
| Loo 直达分量与 K | 固定局部抽样 | 相关变化，K 不再视为常数 |
| 卫星仰角 | 常数 | 随轨道位置更新 |
| 斜距与几何时延 | 常数 | 由 \(d(t)\) 持续更新 |
| 公共多普勒 | 固定频移 | 随径向速度变化，可能过零 |
| 多普勒相位 | 线性旋转 | 对瞬时多普勒积分 |
| 路径损耗与大气损耗 | 通常固定 | 随距离、仰角和天气按需更新 |
| \(DS,ASA,ZSA,SF,K\) | 一个 drop 内固定 | 按相关距离与空间一致性更新 |
| 局部快衰落 | 按多普勒相关变化 | 与上述慢变量连续联合演化 |

Loo 状态持续参数以距离为单位。若某状态持续距离为 \(D_{\mathrm{state}}\)，UE 速度为 \(v_{\mathrm{UE}}\)，则状态持续时间为：

\[
T_{\mathrm{state}}
=\frac{D_{\mathrm{state}}}{v_{\mathrm{UE}}}.
\]

长时间仿真中，不能在每个采样点独立重抽 \(K\)、SF 或状态；应按各参数的相关距离生成空间一致的随机过程，并在状态转换区间保持连续。

#### 5.8.3 时变多普勒相位

几何时延与多普勒满足：

\[
\tau_{\mathrm{geo}}(t)=\frac{d(t)}{c},
\qquad
\nu_{\mathrm{geo}}(t)
=-f_c\frac{\mathrm d\tau_{\mathrm{geo}}(t)}{\mathrm dt}.
\]

固定多普勒时：

\[
\phi(t)=2\pi\nu_D(t-t_0).
\]

长时间情况下必须使用：

\[
\phi(t)
=2\pi\int_{t_0}^{t}
\nu_D(\tilde t)\,\mathrm d\tilde t.
\]

若：

\[
\nu_D(t)=\nu_0+\dot\nu_Dt,
\]

则：

\[
\phi(t)
=2\pi
\left(
\nu_0t+\frac12\dot\nu_Dt^2
\right).
\]

不能直接写成 \(2\pi\nu_D(t)t\)，否则会把多普勒变化率对应的二次相位项放大一倍。

因此，长时间 NTN 信道演化的核心为：

\[
\boxed{
\text{状态与阴影演化}
+
\text{几何量演化}
+
\text{相关 LSP 演化}
+
\text{多普勒累积相位}
}
\]

> **关键原文定位：**TR 38.811 Clause 6.7.1 规定短时两状态流程仅适用于几个 TTI、长仿真中 K 不能视为常数；Clause 6.8.1 给出时变多普勒相位积分；Clauses 6.9.1-6.9.2 允许几个 TTI 内固定速度和仰角。NR slot 时长关系来自 TS 38.211 的 numerology。

## 6 VSAT 性质集合与建模结论

### 6.1 天线与部署性质

VSAT 不是一种独立的信道分布，而是一类终端、天线和部署条件的组合。TR 38.811 中与其建模直接相关的性质可以归纳为：

| 性质 | 物理含义 | 对信道模型的影响 |
|---|---|---|
| 高增益、强方向性 | 能量集中到卫星方向 | 提高链路预算，抑制偏轴散射径 |
| 圆极化 | 降低终端姿态和法拉第旋转带来的极化敏感性 | 仍需考虑轴比与极化失配 |
| 固定或跟踪波束 | 固定站可机械对准，移动平台需跟踪 | 指向误差直接形成增益损失 |
| 窄波束 | 角度选择性强 | 改变有效 PDP、DS 和相干带宽 |
| 屋顶或开阔地部署 | 周围散射通常较少 | 更容易满足准 LOS 和平坦衰落条件 |
| 常用于宽带或回传 | 业务带宽可能很大 | 高方向性不保证整个带宽平坦 |

高增益和窄波束通常同时出现：孔径越大，峰值增益越高，主瓣越窄，但指向误差与跟踪误差代价也越大。因此 VSAT 的优势和约束来自同一个孔径尺度。

> **关键原文定位：**TR 38.811 Clause 6.4.2 把 VSAT 参考天线定义为圆极化、固定或跟踪；Clause 6.7 NOTE 1 指出，高方向性天线位于屋顶或开阔地时更容易达到平坦条件，但仍必须检查平坦判据。

### 6.2 角度滤波与有效功率时延谱

经过收发方向图和波束后，第 \(l\) 条路径的有效复系数为：

\[
\alpha_l^{\mathrm{eff}}
=
\alpha_l
\mathbf v^H\mathbf a_r(\Omega_l^r)
\mathbf a_t^H(\Omega_l^t)\mathbf w.
\]

有效功率时延谱为：

\[
P_{\mathrm{eff}}(\tau)
=
\sum_l
P_l
\left|\mathbf v^H\mathbf a_r(\Omega_l^r)\right|^2
\left|\mathbf a_t^H(\Omega_l^t)\mathbf w\right|^2
\delta(\tau-\tau_l).
\]

令空间加权后的路径功率为 \(P_l^{\mathrm{eff}}\)，则：

\[
\overline\tau_{\mathrm{eff}}
=
\frac{\sum_lP_l^{\mathrm{eff}}\tau_l}
{\sum_lP_l^{\mathrm{eff}}},
\]

\[
\sigma_{\tau,\mathrm{eff}}
=
\sqrt{
\frac{
\sum_lP_l^{\mathrm{eff}}
(\tau_l-\overline\tau_{\mathrm{eff}})^2
}{
\sum_lP_l^{\mathrm{eff}}
}
}.
\]

典型 VSAT 下行中，LOS 位于主瓣，而地面或建筑反射径偏离主瓣并受抑制，因此经常出现：

\[
\sigma_{\tau,\mathrm{eff}}\downarrow
\quad\Longrightarrow\quad
B_c\uparrow.
\]

这不是“天线消除了多径”，而是天线按角度重加权多径。若波束错误抑制早到强径而保留迟到反射径，有效时延扩展也可能增加。

> **关键原文定位：**TR 38.811 Clause 6.7 明确指出相干带宽取决于环境、天线方向图和仰角；Clause 6.9.2 的 NTN-TDL 则是特定空间滤波条件下的结果。

### 6.3 两径例子

假设直达径相对时延为 0、到达角为 \(0^\circ\)，等功率反射径相对时延为 \(1\ \mu\mathrm{s}\)、到达角为 \(40^\circ\)。全向接收时：

\[
H(f)
=1+e^{-j2\pi f(1\ \mu\mathrm{s})}.
\]

第一处深零点约为：

\[
f_{\mathrm{null}}
=\frac{1}{2\Delta\tau}
=500\ \mathrm{kHz}.
\]

两径的 RMS 时延扩展为 \(0.5\ \mu\mathrm{s}\)。若 VSAT 窄波束把反射径功率压低 20 dB，则反射径幅度变为 0.1：

\[
H_{\mathrm{eff}}(f)
=1+0.1e^{-j2\pi f(1\ \mu\mathrm{s})}.
\]

此时幅频响应只在约 0.9 至 1.1 之间变化，有效 RMS 时延扩展约降至 \(0.099\ \mu\mathrm{s}\)。这个例子说明 VSAT 方向图如何通过角度滤波削弱频率选择性。

> **关键原文定位：**该例是对 TR 38.811 Clause 6.7“天线方向图影响相干带宽”结论的基带说明；方向图本身采用 Clause 6.4，时延路径结构对应 Clauses 6.7.2、6.9。

### 6.4 平坦衰落边界与业务带宽

VSAT 更容易满足平坦衰落，但不能仅凭“VSAT”三个字选用单抽头。最终判断仍是：

\[
B_{\mathrm{sig}}\ll B_c
\approx
\frac{1}{10\sigma_{\tau,\mathrm{eff}}}.
\]

假设同一屋顶 VSAT 经方向图加权后得到 \(B_c=8\ \mathrm{MHz}\)：

- 对 1 MHz 业务，\(B_{\mathrm{sig}}/B_c=0.125\)，单抽头可作为合理基准；
- 对 5 MHz 业务，虽然小于 \(B_c\)，但“远小于”的裕量有限，应按可接受的频域相关误差判断；
- 对 100 MHz 业务，整体平坦衰落明显不成立，应使用 TDL 或 CDL。

因此，同一副 VSAT 天线在不同业务带宽、仰角和环境下可以对应不同快衰落模型。方向性天线、平坦衰落和窄带业务不是固定的一一映射。

> **关键原文定位：**TR 38.811 Clause 6.7 给出的 S 波段、仰角、准 LOS、5 MHz 和环境条件基于准全向 UE 测量；NOTE 1 只说明方向性 VSAT 更容易满足条件，并未免除相干带宽检查。

### 6.5 VSAT 仿真建模清单

| 仿真目标 | VSAT 建模建议 | 需要验证的条件 |
|---|---|---|
| 覆盖和链路预算 | 圆形孔径绝对增益 + 大尺度损耗 | 口径效率、指向误差、雨衰、可用率 |
| 窄带准 LOS 基准 | Ricean 或 ITU 两状态单抽头 | \(B_{\mathrm{sig}}\ll B_c\)，避免重复加入 SF/CL |
| 宽带均衡和 CP | 等效 TDL | 有效 PDP、RMS DS、最大抽头时延 |
| 波束宽度或指向误差研究 | CDL/几何路径 + 实际方向图 | 不使用固定 TDL 替代可变空间滤波 |
| 阵列、多流或波束间干扰 | 明确馈源/阵元、端口和射频链 | 标量圆形孔径方向图本身不够 |

若波束从 \(5^\circ\) 收窄到 \(1^\circ\)，不同反射径增益会改变：

\[
P_p\rightarrow P_p^{\mathrm{eff}},
\qquad
\sigma_\tau\rightarrow\sigma_{\tau,\mathrm{eff}}.
\]

固定 TDL 的抽头功率已经预先给定，不能自动反映新的波束宽度。研究 VSAT 波束时，应在 CDL 或几何路径模型中先施加新方向图，再生成相应等效 TDL。

> **关键原文定位：**TR 38.811 Clauses 6.4.2、6.7、6.9.2 共同限定 VSAT 天线、平坦判据和等效 TDL 的使用方式；不存在脱离带宽、环境和波束配置的统一“VSAT 信道模型”。

## 7 完整模型的装配关系

### 7.1 部署场景的基线开关

TR 38.811 Table 6.10.1-1 可整理为：

| 部署 | 平台与频段 | O2I | 气体吸收 | 雨云 | 闪烁 | 链路级快衰落 |
|---|---|---|---|---|---|---|
| D1 | GEO Ka | No | Mandatory | 若系统尺寸设计未计入则使用 ITU 模型 | 对流层 | 平坦 |
| D2 | GEO S | No | Negligible | Negligible | 电离层 | 平坦或 CDL/TDL |
| D3 | 非 GEO S | No | Negligible | Negligible | 电离层 | CDL/TDL |
| D4 | 非 GEO Ka | No | Mandatory | 若系统尺寸设计未计入则使用 ITU 模型 | 对流层 | 平坦 |
| D5 | HAPS、低于 6 GHz | Possible | Negligible | Negligible | Negligible | CDL/TDL |

D1/D4 即使最大系统带宽很大，校准表仍采用平坦衰落，主要对应 VSAT 高方向性天线和低散射的特定评估假设。它不能推出“所有 Ka 宽带卫星链路天然平坦”。

### 7.2 单个 drop 的计算顺序

#### 7.2.1 输入参数

至少需要明确：

| 参数组 | 输入 |
|---|---|
| 几何 | UE/平台 ECEF 位置、轨道或速度、姿态、仿真时刻 |
| 载波与波形 | \(f_c\)、业务带宽、SCS、CP、采样率 |
| 场景 | D1～D5、密集城市/城市/郊区/农村、室内外 |
| 天线 | 口径或阵列、波束轴、极化、端口、指向误差 |
| 传播 | 气象位置、目标可用率、地磁纬度、是否采用晴空基线 |
| 快衰落 | 平坦/Loo/CDL/TDL、目标 DS/ASA/ZSA/K、UE 速度 |
| 时间尺度 | 单个 drop、几个 TTI，或连续轨迹 |

#### 7.2.2 几何与天线计算

1. 由 UE 和平台位置得到斜距 \(d\)、仰角 \(\alpha\)、AOA/ZOA/AOD/ZOD；
2. 计算

\[
\tau_{\mathrm{geo}}=\frac dc,
\qquad
\nu_{\mathrm{geo}}
=-\frac{f_c}{c}\frac{\mathrm dd}{\mathrm dt};
\]

3. 把传播方向变换到天线本体坐标系；
4. 计算收发方向增益、极化响应和指向误差损耗。

#### 7.2.3 大尺度传播计算

若采用 Clause 6.6 + CDL/TDL 路径：

1. 根据环境和最近参考仰角抽取 LOS/NLOS；
2. 查 \(\sigma_{\mathrm{SF}}\) 和 \(CL\)；
3. 生成一次 SF；若后续联合生成相关 LSP，此处复用同一个 SF；
4. 计算 \(FSPL\) 和 \(PL_b\)；
5. 仅在室内场景计算 BEL；
6. 按频率与仰角开关计算气体吸收；
7. 按晴空基线或目标可用率处理雨云衰减；
8. 按频率和地磁纬度选择电离层/对流层闪烁；
9. 得到总路径损耗 \(PL\)。

若采用 Clause 6.7.1 ITU 两状态平坦模型：

1. 不使用 Table 6.6.1-1 的 LOS 概率；
2. 只计算 \(PL_b=FSPL\)；
3. 由 GOOD/BAD Loo 模型产生遮挡、阴影和单抽头快衰落；
4. 再按场景加入气体、雨云、闪烁等不属于 Loo 的附加效应。

#### 7.2.4 局部快衰落计算

对于完整 GSCM：

1. 按环境、频段、状态和仰角读取 LSP 均值、标准差、相关矩阵和相关距离；
2. 联合抽取 \(DS,ASA,ZSA,SF,K\)；卫星侧 ASD/ZSD 置零；
3. 生成簇时延、簇功率、AOA/ZOA/AOD/ZOD、XPR 和子径；
4. 加入收发阵列、极化矩阵和法拉第旋转；
5. 生成初相位与局部多普勒；
6. 对所有路径加入公共卫星多普勒。

对于链路级参考模型：

1. 选择 A/B（NLOS）或 C/D（LOS）；
2. 用目标 DS 缩放归一化时延；
3. CDL 进一步缩放 ASA、ZSA 和平均角度；
4. LOS 时按目标 K 调整首簇或首抽头；
5. 应用实际天线和波束；若改用 TDL，确认其空间滤波条件与实际天线一致。

#### 7.2.5 信道装配

最终连续时间模型可以写为：

\[
\mathbf y(t)
=
10^{-PL(t)/20}
e^{j\phi_{\mathrm{geo}}(t)}
\int
\mathbf H_{\mathrm{local}}(t,\tau)
\mathbf x\!\left(t-\tau_{\mathrm{geo}}(t)-\tau\right)
\,\mathrm d\tau
+\mathbf n(t),
\]

\[
\phi_{\mathrm{geo}}(t)
=2\pi\int_{t_0}^{t}
\nu_{\mathrm{geo}}(\tilde t)\,\mathrm d\tilde t.
\]

这里 \(10^{-PL/20}\) 是复幅度缩放；若在功率域工作，则对应 \(10^{-PL/10}\)。天线绝对增益可以并入链路预算，也可以保留在 \(\mathbf H_{\mathrm{local}}\) 的阵列响应中，但不能两处重复加入。

### 7.3 时间更新粒度

不同量不需要以同一采样率更新：

| 量 | 典型更新触发 |
|---|---|
| OFDM 内快衰落相位 | 基带采样或 OFDM 符号 |
| 局部多普勒相关过程 | 按最大多普勒与目标相关精度 |
| 几何时延与公共多普勒 | 当相位、时延或频偏误差超过容限 |
| 波束方向与天线增益 | 当指向误差或波束切换条件变化 |
| LSP | UE 移动距离达到相关距离的一小部分 |
| LOS/NLOS 或 GOOD/BAD | 状态持续距离结束或几何遮挡改变 |
| 雨云与气体 | 按天气、位置和仰角变化时间尺度 |

长时间轨迹中，最容易出现的错误是低频更新 \(\nu_D\) 后直接重置相位。正确做法是保持相位累计连续，并在更新点使用积分或数值累加：

\[
\phi[q+1]
=\phi[q]
+2\pi\nu_D[q]\Delta t.
\]

### 7.4 仿真输出与一致性检查

每个 drop 或时间点至少保存：

- 几何量：\(\alpha,d,\tau_{\mathrm{geo}},\nu_{\mathrm{geo}}\)；
- 天线量：收发方向增益、波束编号、极化和指向损耗；
- 大尺度量：状态、\(FSPL,CL,SF,PL_e,PL_g,PL_s,A_{\mathrm{rain}},A_{\mathrm{cloud}},PL\)；
- LSP：\(DS,ASA,ZSA,K\) 及使用的随机种子；
- 快衰落量：模型名称、抽头/簇时延功率、局部多普勒、公共频移；
- 模型开关：是否已在链路预算计入雨云、是否使用 Loo、是否启用法拉第矩阵。

交付或复现实验前应检查：

1. \(FSPL\) 的距离和频率单位组合是否一致；
2. 实际仰角映射到了哪个参考表项；
3. LOS 时是否误加了 NLOS 杂波损耗；
4. SF 是否被独立抽样两次；
5. Loo 与 \(CL+SF\) 是否重复；
6. 雨云余量是否与系统尺寸设计重复；
7. BEL 是否错误加入 D1～D4 室外卫星基线；
8. \(\tau_{\mathrm{geo}}\) 是否与局部超额时延混合；
9. 公共多普勒是否已作用于所有路径；
10. 法拉第旋转与标量极化损耗是否重复；
11. TDL 是否仍与实际方向图和波束条件一致；
12. 长时间多普勒相位是否连续累计。

四层不能互相替代：CDL/TDL 不负责提供完整卫星几何时延，大尺度路径损耗不描述相干多径，参考方向图也不自动给出端口数与空间流数。完整模型的关键不是把所有效应全部打开，而是在明确研究目标后，让每个物理效应只由一个模型组件负责。

> **关键原文定位：**TR 38.811 Clauses 6.3-6.10；系统级生成主流程见 Figure 6.5.1-1，简化平坦流程见 Figure 6.5.1-2，部署开关见 Table 6.10.1-1。
