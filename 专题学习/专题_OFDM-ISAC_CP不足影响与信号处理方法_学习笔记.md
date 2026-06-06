# OFDM-ISAC 中 CP 不足的影响、性能分析与信号处理方法总结

> 主题：OFDM-ISAC beyond-CP sensing  
> 目标：理解 CP 短为什么会影响感知、不同论文在性能分析上的假设差异，以及三类典型信号处理方法：相干补偿、Sliding Window、固定窗口 SIC-DFT/ESPRIT。

---

## 1. 问题背景：为什么 OFDM-ISAC 中 CP 会成为感知瓶颈？

### 1.1 通信中的 CP：主要覆盖“相对时延扩展”

在 OFDM 通信中，循环前缀（Cyclic Prefix, CP）的主要作用是把线性卷积转化为循环卷积，从而保持子载波正交性。设通信信道为

\[
h(t)=\sum_{\ell}h_\ell\delta(t-\tau_\ell).
\]

通信接收机通常会进行定时同步，将 FFT 窗口对齐到最早径、主径或某个合适参考径。因此，通信中常说的 CP 条件通常是

\[
\tau_{\max}-\tau_{\min}\leq T_{\rm cp}.
\]

也就是说，通信更关心**多径相对时延扩展**是否超过 CP，而不是某条路径的绝对传播时延是否超过 CP。

---

### 1.2 感知中的 CP：固定窗口下要覆盖“目标双程绝对时延”

在单站 OFDM-ISAC 感知中，基站发射 OFDM 信号，同时接收目标回波。第 \(q\) 个目标的双程时延为

\[
\tau_q=\frac{2R_q}{c},
\]

多普勒频移为

\[
f_{d,q}=\frac{2v_q f_c}{c}.
\]

如果感知接收机采用标准固定 FFT 窗口，即第 \(m\) 个 OFDM 符号的检测窗口从 \(mT_s\) 附近开始，那么为了不漏掉近距离目标，窗口通常相对于发射时刻固定，而不是针对某个远距离目标重新同步。

此时，无 ISI/ICI 的条件变为

\[
\tau_q\leq T_{\rm cp}.
\]

对应的 CP 保护距离为

\[
R_{\rm cp}=\frac{cT_{\rm cp}}{2}.
\]

例如 5G NR 毫米波、子载波间隔 \(\Delta f=120\text{ kHz}\) 时，normal CP 大约为 \(0.59\,\mu s\)，对应的 CP 无干扰感知距离只有约 \(90\text{ m}\)。这远小于许多室外感知需求。

---

### 1.3 这与通信中的 CP 条件并不矛盾

通信与感知的差别不是物理规律不同，而是**参考窗口不同**。

通信中：

\[
\text{CP 覆盖相对于同步参考径的时延扩展。}
\]

固定窗口感知中：

\[
\text{CP 覆盖目标回波相对于发射符号的双程时延。}
\]

如果允许感知接收机滑动 FFT 窗口，也可以让某个远距离目标重新落入局部 CP 保护范围。但多目标情况下，不同目标需要不同窗口，这会带来搜索、重构和干扰抵消问题。

---

## 2. 基本信号模型

### 2.1 CP 足够时的理想模型

理想情况下，第 \(q\) 个目标在第 \(n\) 个子载波、第 \(m\) 个 OFDM 符号上的频域回波可写为

\[
y_{n,m,q}
=
\alpha_q s_{n,m}
e^{-j2\pi n\Delta f\tau_q}
e^{j2\pi m f_{d,q}T_s}.
\]

其中：

- \(\alpha_q\)：目标复反射系数；
- \(s_{n,m}\)：已知 OFDM 数据符号；
- \(e^{-j2\pi n\Delta f\tau_q}\)：距离相关相位；
- \(e^{j2\pi m f_{d,q}T_s}\)：速度相关相位。

因为 \(s_{n,m}\) 已知，接收机可以做数据去除：

\[
y_{n,m,q}s_{n,m}^*
=
\alpha_q |s_{n,m}|^2
e^{-j2\pi n\Delta f\tau_q}
e^{j2\pi m f_{d,q}T_s}.
\]

若采用恒模星座，\(|s_{n,m}|^2=1\)，则可直接得到距离-多普勒二维相位结构。

常规 OFDM sensing 的距离-多普勒图（Range-Doppler Map, RDM）处理为

\[
\chi
=
F_N^H(Y\odot S^*)F_M.
\]

其中 \(F_N^H\) 在子载波维做 IDFT，用于距离估计；\(F_M\) 在慢时间维做 DFT，用于多普勒估计。

---

### 2.2 CP 不足时的模型

设目标离散时延为

\[
N_\tau=[\tau B],
\]

CP 长度为

\[
N_{\rm cp}=T_{\rm cp}B.
\]

当

\[
N_\tau>N_{\rm cp},
\]

固定 FFT 窗口中会出现两类问题：

1. **ISI**：窗口前部包含前一个 OFDM 符号的尾部；
2. **ICI**：当前 OFDM 符号没有被完整截取，子载波正交性被破坏。

定义归一化超出 CP 的时延比例：

\[
\rho=\frac{(N_\tau-N_{\rm cp})^+}{N}.
\]

对于 beyond-CP 目标，频域输出可抽象为

\[
\tilde y_{n,m,q}
=
\underbrace{
\alpha_q s_{n,m}
e^{-j2\pi n\Delta f\tau_q}
e^{j2\pi m f_{d,q}T_s}
}_{\text{理想主项}}
+
\underbrace{y^{\rm ISI}_{n,m,q}}_{\text{符号间干扰}}
-
\underbrace{y^{\rm ICI}_{n,m,q}}_{\text{子载波间干扰}}.
\]

矩阵形式为

\[
Y
=
Y_{\rm free}
+
Y_{\rm ISI}
-
Y_{\rm ICI}
+
Z.
\]

其中 \(Y_{\rm free}\) 保留理想距离-多普勒外积结构，而 \(Y_{\rm ISI}\) 与 \(Y_{\rm ICI}\) 会引入符号维和子载波维的耦合。

---

## 3. 性能分析：不同假设下的结论

目前关于 CP 不足的分析大致可以分成两类视角：

1. **结构化 ISI/ICI 视角**：认为 CP 不足产生的干扰具有确定耦合结构，会破坏 RDM，抬高旁瓣；
2. **随机数据 mask 视角**：认为通信数据的随机性会把 ISI/ICI 随机化，经过 IFFT/DFT 雷达处理后类似噪声，被处理增益显著削弱。

这两种分析并不完全矛盾，而是关注的性能指标和建模假设不同。

---

### 3.1 视角一：结构化 ISI/ICI 分析

#### 3.1.1 基本假设

该视角不把 ISI/ICI 简单近似为独立高斯噪声，而是显式保留其结构。典型模型为

\[
Y=Y_{\rm free}+Y_{\rm ISI}-Y_{\rm ICI}+Z.
\]

其中

\[
Y_{\rm free}
=
\sum_{q=1}^{Q}
\alpha_q
\left(
b(\tau_q)c^H(f_{d,q})\odot S
\right),
\]

\[
Y_{\rm ISI}
=
\sum_{q}
\alpha_q
\Phi_q
\left(
b(\tau_q-T_{\rm cp})c^H(f_{d,q})\odot S
\right)J_1,
\]

\[
Y_{\rm ICI}
=
\sum_{q}
\alpha_q
\Phi_q
\left(
b(\tau_q)c^H(f_{d,q})\odot S
\right).
\]

这里：

- \(\Phi_q\)：子载波泄露矩阵，描述 ICI；
- \(J_1\)：时间移位矩阵，描述相邻 OFDM 符号耦合，即 ISI；
- \(b(\tau_q)\)：距离维 steering vector；
- \(c(f_{d,q})\)：多普勒维 steering vector。

---

#### 3.1.2 SINR 结论

定义感知 SINR 为

\[
{\rm SINR}
=
\frac{
\mathbb{E}\{\|Y_{\rm free}\|_F^2\}
}{
\mathbb{E}\{\|Y_{\rm ISI}-Y_{\rm ICI}\|_F^2\}
+
\mathbb{E}\{\|Z\|_F^2\}
}.
\]

在结构化模型下，可得到

\[
{\rm SINR}
=
\frac{
MN\sum_{q=1}^{Q}\sigma_{\alpha,q}^2
}{
(2M-1)N
\sum_{q=\tilde Q+1}^{Q}
\rho_q\sigma_{\alpha,q}^2
+
MN\sigma^2
}.
\]

当 OFDM 符号数 \(M\gg 1\) 时，近似为

\[
{\rm SINR}
\approx
\frac{
\sum_{q=1}^{Q}\sigma_{\alpha,q}^2
}{
2\sum_{q=\tilde Q+1}^{Q}
\rho_q\sigma_{\alpha,q}^2
+
\sigma^2
}.
\]

核心结论是：

\[
\boxed{
\text{CP 超出比例 } \rho_q \text{ 越大，ISI/ICI 干扰越强，SINR 越低。}
}
\]

---

#### 3.1.3 RDM 二阶矩与旁瓣分析

RDM 定义为

\[
\chi=F_N^H(Y\odot S^*)F_M.
\]

结构化分析给出 RDM 二阶矩：

\[
\mathbb{E}\{|\chi(l,\nu)|^2\}
=
\sum_{q=1}^{Q}
\frac{\tilde\sigma_{\alpha,q}^2}{MN}
|D_N(l-\tilde l_q)|^2
|D_M(\nu-\tilde\nu_q)|^2
+
\sigma_{\rm SL}^2.
\]

其中，对于 beyond-CP 目标，

\[
\tilde\sigma_{\alpha,q}^2
=
(1-\rho_q)^2\sigma_{\alpha,q}^2.
\]

这说明 CP 不足会降低目标主瓣功率。旁瓣地板大致包含

\[
\sigma_{\rm SL}^2
=
(\mu_4-1)\sum_q\tilde\sigma_{\alpha,q}^2
+
\sum_{q=\tilde Q+1}^{Q}
\rho_q(2-\rho_q)\sigma_{\alpha,q}^2
+
\sigma^2.
\]

其中

\[
\rho_q(2-\rho_q)\sigma_{\alpha,q}^2
\]

是 CP 不足造成的残余 ISI/ICI 泄露项。

因此，CP 不足同时造成：

\[
\text{主瓣下降}
+
\text{旁瓣地板升高}
+
\text{PSLR 变差}.
\]

这一视角特别适合分析：

- RDM 旁瓣；
- 弱目标检测；
- 强目标泄露；
- 高精度距离-速度估计；
- 固定 FFT 窗口下的结构化失真。

---

### 3.2 视角二：随机数据 mask 分析

#### 3.2.1 基本假设

该视角强调 OFDM-ISAC 中通信数据 \(d_{k,m}\) 是随机的。CP 不足造成的 ICI/ISI 在数据去除后，会被随机数据“打散”。

CP 不足时，FFT 输出可写为

\[
\widetilde Y_m[i]
=
\underbrace{
(N-N_\tau+N_{\rm cp})d_{i,m}
e^{-j\frac{2\pi}{N}iN_\tau}
}_{\text{useful signal}}
+
I_c[i]
+
I_s[i].
\]

数据去除后，

\[
\widetilde F_m[i]
=
(N-N_\tau+N_{\rm cp})
e^{-j\frac{2\pi}{N}iN_\tau}
+
\frac{I_c[i]}{d_{i,m}}
+
\frac{I_s[i]}{d_{i,m}}.
\]

由于 \(d_{i,m}\) 随机，作者近似认为

\[
I_c'[i]=\frac{I_c[i]}{d_{i,m}}
\sim \mathcal{CN}(0,P_{\rm ICI}),
\]

\[
I_s'[i]=\frac{I_s[i]}{d_{i,m}}
\sim \mathcal{CN}(0,P_{\rm ISI}).
\]

再经过 IFFT 后，干扰方差被 \(N\) 个子载波平均：

\[
\widetilde I_c[p]\sim
\mathcal{CN}\left(0,\frac{P_{\rm ICI}}{N}\right),
\]

\[
\widetilde I_s[p]\sim
\mathcal{CN}\left(0,\frac{P_{\rm ISI}}{N}\right).
\]

---

#### 3.2.2 有用信号、ICI、ISI 功率

定义

\[
\rho=\frac{(N_\tau-N_{\rm cp})^+}{N}.
\]

在频域解调输出中：

\[
P_u=(1-\rho)^2,
\]

\[
P_{\rm ICI}=(1-\rho)\rho,
\]

\[
P_{\rm ISI}=\rho.
\]

注意：这些是 FFT 解调输出层面的功率。经过数据去除和 IFFT 后，ICI/ISI 进一步被 \(1/N\) 平均，而有用目标峰值会相干叠加。

---

#### 3.2.3 单目标 SINR

单目标情况下，range profile 中 SINR 可写为

\[
\gamma(d)
=
\frac{
M(1-\rho)^2
}{
\frac{1}{\gamma_1(d)}
+
\frac{\rho}{N}(2-\rho)
}.
\]

其中

\[
\gamma_1(d)=\frac{P_R(d,\kappa)}{N_0\Delta f},
\]

\(P_R(d,\kappa)\) 由单站雷达方程给出：

\[
P_R(d,\kappa)
=
\kappa
\frac{\lambda^2}{(4\pi)^3d^4}
G_TG_RP_T.
\]

该式说明：

- 主峰功率会被 \((1-\rho)^2\) 削弱；
- ISI/ICI 干扰项为 \(\frac{\rho}{N}(2-\rho)\)，当 \(N\) 很大时可能较小；
- 多符号积累提供 \(M\) 倍处理增益。

---

#### 3.2.4 多目标 SINR

多目标场景下，beyond-CP 目标的 ISI/ICI 会共同提升噪声地板。可写为

\[
I_t
=
\sum_l
\rho_l(2-\rho_l)P_R(d_l,\kappa_l).
\]

此时典型目标的 SINR 可表示为

\[
\gamma(d)
=
\frac{
M(1-\rho)^2
}{
\frac{1}{\gamma_1(d)}
+
\frac{I_t}{N}
}.
\]

当子载波数 \(N\) 足够大时，\(\frac{I_t}{N}\) 可能较小。此时最大感知距离主要受路径损耗、目标 RCS、处理增益和检测阈值影响，而不应简单等同于 \(cT_{\rm cp}/2\)。

---

### 3.3 两种性能分析的关系

两种观点的差异可以总结如下：

| 维度 | 结构化 ISI/ICI 分析 | 随机数据 mask 分析 |
|---|---|---|
| 核心假设 | ISI/ICI 具有确定结构，不能简单当高斯噪声 | 随机通信数据会将 ISI/ICI 随机化 |
| 主要指标 | SINR、RDM 二阶矩、PSLR、旁瓣地板、估计精度 | range profile SINR、最大可检测距离 |
| 关注问题 | RDM 结构破坏、强弱目标、旁瓣泄露 | CP-free range 是否等于最大 sensing range |
| 结论倾向 | CP 不足会近似线性降低 SINR、抬高旁瓣 | CP 不足不一定严重限制最大感知距离 |
| 适用场景 | 高精度估计、弱目标检测、强目标泄露 | 大 \(N\)、随机数据、平均检测性能分析 |

两者并不完全矛盾。更准确地说：

\[
\boxed{
\text{CP 长度限制的是“无 ISI/ICI 的固定窗口感知距离”，但不必然等于最大可检测距离。}
}
\]

同时：

\[
\boxed{
\text{即使目标可被检测，CP 不足仍可能造成 RDM 旁瓣抬升和参数估计退化。}
}
\]

---

## 4. 三种信号处理方法

下面总结三类典型 beyond-CP OFDM sensing 方法：

1. 相干补偿（Coherent Compensation）；
2. Sliding Window Sensing；
3. 固定窗口 SIC-DFT / SIC-ESPRIT。

---

## 4.1 方法一：相干补偿 Coherent Compensation

### 4.1.1 基本思想

相干补偿的核心是：

\[
\boxed{
\text{目标超过 CP 后，当前 OFDM 符号在 FFT 窗口中缺了一段；相干补偿尝试把后面的样点补到前面。}
}
\]

它不一定改变整体 FFT 窗口位置，而是对窗口内前若干个样点进行补偿。例如，设补偿长度为 \(N'\)，可以写成

\[
\tilde y[i]=
\begin{cases}
y[i]+y[i+N], & 0\le i<N',\\
y[i], & N'\le i<N.
\end{cases}
\]

直观上，\(y[i+N]\) 中包含当前 OFDM 符号后续未被原 FFT 窗口完整采到的部分。把它加回前面后，可以增强当前目标回波的相干能量。

---

### 4.1.2 数学功率分析

定义

\[
N_e=\frac{N_s-N_{\rm cp}}{N},
\]

表示超过 CP 的归一化长度；定义

\[
N_a=\frac{N'}{N},
\]

表示补偿长度比例。

补偿前，有用信号功率为

\[
P_u=(1-N_e)^2|\alpha|^2.
\]

补偿后，有用信号幅度变为

\[
1-N_e+N_a,
\]

因此

\[
P_u^{\rm cc}
=
(1-N_e+N_a)^2|\alpha|^2.
\]

补偿后的 ICI 功率可写为

\[
P_{\rm ICI}^{\rm cc}
=
|N_e-N_a|
\left(1-|N_e-N_a|\right)|\alpha|^2.
\]

ISI 通常不被该操作消除，因为 ISI 来自前一个 OFDM 符号：

\[
P_{\rm ISI}^{\rm cc}
=
N_e|\alpha|^2.
\]

噪声会增加，因为部分样点是两个噪声样点相加：

\[
P_w^{\rm cc}
=
(1+N_a)\sigma^2.
\]

于是补偿后 SINR 可写为

\[
\Upsilon_f^{\rm cc}
=
\frac{
(1-N_e+N_a)^2
}{
N_e
+
|N_e-N_a|\left(1-|N_e-N_a|\right)
+
\frac{1+N_a}{\gamma_0}
}.
\]

其中

\[
\gamma_0=\frac{|\alpha|^2}{\sigma^2}.
\]

---

### 4.1.3 最优补偿长度的直观理解

若取

\[
N'=N_s-N_{\rm cp},
\]

即

\[
N_a=N_e,
\]

则

\[
P_{\rm ICI}^{\rm cc}=0.
\]

这意味着刚好补齐缺失的当前 OFDM 符号，子载波正交性得到恢复。

但是补偿越多，噪声也越强；如果补偿过头，还可能重新引入 ICI。因此相干补偿本质上是在

\[
\text{有用信号增强}
\quad
\text{ICI 减小}
\quad
\text{噪声增加}
\]

之间做折中。

---

### 4.1.4 优缺点

优点：

- 复杂度较低；
- 不需要改变发射 OFDM 波形；
- 对单个远距离目标或目标稀疏场景有效；
- 可直接提升 beyond-CP 目标的主峰功率。

缺点：

- 需要知道或搜索合适补偿长度；
- 主要适合某个目标或某个距离段；
- 若存在近距离强目标，补偿会破坏近距离目标的正交性，使其产生严重 ISI/ICI；
- 不显式消除残余结构化 ISI/ICI；
- 可能伴随噪声放大。

适用场景：

\[
\boxed{
\text{目标较稀疏、远距离目标为主、近距离强目标不明显的场景。}
}
\]

---

## 4.2 方法二：Sliding Window Sensing

### 4.2.1 基本思想

Sliding Window 可以看作一种时域 SIC：

\[
\boxed{
\text{先检测近距离目标并消除，再把 FFT 检测窗口后移，检测下一段远距离目标。}
}
\]

第 \(v\) 个滑动窗口从

\[
t=mT_s+vT_{\rm cp}
\]

开始，主要检测满足

\[
vT_{\rm cp}\leq \tau <(v+1)T_{\rm cp}
\]

的目标。

它将整个距离范围按照 CP 长度分段：

\[
[0,T_{\rm cp}),
[T_{\rm cp},2T_{\rm cp}),
[2T_{\rm cp},3T_{\rm cp}),\ldots
\]

每一轮只检测当前局部 CP 范围内的目标。

---

### 4.2.2 为什么要先消除近距离目标？

如果直接后移窗口，远距离目标可以被更完整地截取，但近距离目标会在新窗口中变成不完整 OFDM 符号，从而产生强 ISI/ICI。

因此 Sliding Window 的关键是：

\[
\text{先消除已检测短距离强目标，再移动窗口。}
\]

这使其比简单相干补偿更适合近距离强目标与远距离弱目标共存的场景。

---

### 4.2.3 基本处理流程

第 \(v\) 轮处理中，将接收信号重新分组为

\[
y_m^{(v)}[n]
=
y[n+m(N+N_{\rm cp})+vN_{\rm cp}],
\quad
m=0,\ldots,M-1.
\]

然后执行常规 OFDM sensing：

\[
y_m^{(v)}[n]
\rightarrow
\text{FFT}
\rightarrow
Y_m^{(v)}[i]
\rightarrow
\text{data removal}
\rightarrow
F_m^{(v)}[i]
\rightarrow
\text{IFFT}
\rightarrow
R_m^{(v)}[p].
\]

之后沿慢时间维做 Doppler FFT 得到

\[
R^{(v)}[p,q].
\]

只在当前窗口对应的局部范围内检测峰值：

\[
p\in[0,N_{\rm cp}].
\]

检测到的局部时延需要加上窗口偏移：

\[
\hat \tau=\frac{\hat p+vN_{\rm cp}}{B}.
\]

对应距离为

\[
\hat d=
\frac{c(\hat p+vN_{\rm cp})}{2B}.
\]

---

### 4.2.4 近距离目标的 FIR 重构与消除

Sliding Window 不完全依赖显式参数

\[
\hat \alpha_l,\hat\tau_l
\]

来重构目标，因为 off-grid delay 会导致复增益相位估计误差。

它采用 FIR 信道重构。设当前窗口估计出的复杂 range response 为 \(R_m[p]\)，构造 FIR 系数：

\[
h[p]
=
\frac{1}{N}
R_m[p-N_{\rm lag}]_N
e^{j\pi p},
\quad
p=0,\ldots,N_{\rm cp}+N_{\rm lag}-1.
\]

其中 \(N_{\rm lag}\) 用于处理 off-grid delay 带来的非因果部分。

已知发送信号为

\[
x_m[n]
=
\sum_{k=0}^{N-1}
d_{k,m}
e^{j\frac{2\pi}{N}k(n+N_{\rm cp})}.
\]

通过 FIR 得到已检测目标回波：

\[
\tilde y_m^0[n]=x_m[n]*h[n],
\]

并取

\[
\hat y_m^0[n]=\tilde y_m^0[n+N_{\rm lag}].
\]

然后执行消除：

\[
y_m[n]\leftarrow y_m[n]-\hat y_m^0[n].
\]

---

### 4.2.5 SINR 改善

常规固定窗口下，远距离目标 SINR 中含有主峰功率损失项：

\[
\gamma(d)
=
\frac{
M(1-\rho)^2
}{
\frac{1}{\gamma_1(d)}
+
\frac{I_t}{N}
}.
\]

Sliding Window 通过移动窗口，使当前检测段内目标重新被完整截取，因此去掉 \((1-\rho)^2\) 衰减项：

\[
\gamma_{\rm sw}(d)
=
\frac{
M
}{
\frac{1}{\gamma_1(d)}
+
\frac{I_t}{N}
}.
\]

其本质是：

\[
\boxed{
\text{恢复远距离目标的相干主峰功率，而不是主要依靠降低 ISI/ICI。}
}
\]

---

### 4.2.6 复杂度

若最大检测距离为 \(d_{\max}\)，所需窗口数约为

\[
v_{\max}+1,
\]

其中

\[
v_{\max}
=
\left\lfloor
\frac{2d_{\max}}{cT_{\rm cp}}-1
\right\rfloor.
\]

因此整体复杂度大约是常规 OFDM sensing 的

\[
v_{\max}+1
\]

倍，外加每轮目标重构的 FIR 滤波开销。

---

### 4.2.7 优缺点

优点：

- 不改变发射波形；
- 不增加 CP 开销；
- 可以恢复远距离目标功率；
- 能处理近距离强目标与远距离弱目标共存的场景；
- FIR 重构比直接参数重构更工程化。

缺点：

- 需要多次滑动窗口处理，复杂度随最大距离线性增加；
- 依赖前序目标检测与消除效果；
- 前面窗口残差会影响后续窗口，存在 SIC 误差传播；
- 需要缓存较长时域接收信号；
- 更偏向距离检测和距离扩展，不是专门为超分辨二维参数估计设计。

适用场景：

\[
\boxed{
\text{需要扩展最大可检测距离，且存在近远目标共存的场景。}
}
\]

---

## 4.3 方法三：固定窗口 SIC-DFT / SIC-ESPRIT

### 4.3.1 基本思想

该方法保持标准固定 FFT 窗口不动，将 CP 不足引入的 ISI/ICI 显式建模为

\[
Y=Y_{\rm free}+Y_{\rm ISI}-Y_{\rm ICI}+Z.
\]

如果能估计目标参数

\[
\hat\tau_q,\quad \hat f_{d,q},\quad \hat\alpha_q,
\]

就可以重构

\[
\hat Y_{\rm ISI},\quad \hat Y_{\rm ICI},
\]

然后更新

\[
Y_{\rm free}^{k+1}
=
Y-\hat Y_{\rm ISI}^{k}+\hat Y_{\rm ICI}^{k}.
\]

这是频域/矩阵域 SIC：

\[
\boxed{
\text{估计目标参数}
\rightarrow
\text{重构结构化 ISI/ICI}
\rightarrow
\text{抵消干扰}
\rightarrow
\text{再次估计}.
}
\]

---

### 4.3.2 SIC-DFT

SIC-DFT 使用传统二维 DFT 生成 RDM。

初始化：

\[
Y_{\rm free}^0=Y.
\]

第 \(k\) 次迭代：

\[
\chi^k
=
F_N^H
\left(
Y_{\rm free}^{k}\odot S^*
\right)
F_M.
\]

然后用 CFAR 或峰值检测得到

\[
\hat\tau_q^k,\quad \hat f_{d,q}^k.
\]

反射系数用 LS 估计：

\[
\hat{\alpha}_q^k
=
\frac{
b^H(\hat{\tau}_q^k)
\left(
Y_{\rm free}^{k}\odot S^*
\right)
c(\hat f_{d,q}^k)
}{
\|S\|_F^2
}.
\]

随后重构 \(Y_{\rm ISI}^{k}\) 与 \(Y_{\rm ICI}^{k}\)，并更新：

\[
Y_{\rm free}^{k+1}
=
Y
-
Y_{\rm ISI}^{k}
+
Y_{\rm ICI}^{k}.
\]

停止条件通常为

\[
\frac{
\|Y_{\rm free}^{k+1}-Y_{\rm free}^{k}\|_F
}{
\|Y_{\rm free}^{k}\|_F
}
<
\delta_{\rm th}.
\]

优点：

- 实现简单；
- 复杂度相对较低；
- 易与传统 RDM/CFAR 流程结合。

缺点：

- 受 DFT 网格分辨率限制；
- off-grid 目标会产生谱泄露；
- 多目标接近时分辨能力有限。

---

### 4.3.3 SIC-ESPRIT

SIC-ESPRIT 将 SIC-DFT 中的 DFT 峰值估计替换为 ESPRIT 子空间估计。

构造数据辅助感知信道向量：

\[
\hat h
=
\operatorname{vec}
\left(
Y_{\rm free}^{k}\odot S^*
\right).
\]

在结构化 CP 不足模型下，其协方差可写为

\[
R
=
\mathbb{E}\{\hat h\hat h^H\}
=
A_Q\Sigma_\alpha A_Q^H
+
\sigma_{\rm SL}^2I.
\]

其中

\[
A_Q=[a_1,\ldots,a_Q],
\]

\[
a_q=c^*(f_{d,q})\otimes b(\tau_q).
\]

这说明：CP 不足会改变有效目标功率和噪声/旁瓣地板，但目标 steering vector 张成的信号子空间仍然存在。

ESPRIT 利用 steering vector 的移位不变性。距离维相邻子载波相位差为

\[
e^{-j2\pi\Delta f\tau_q},
\]

因此

\[
\hat\tau_q
=
-\frac{\angle\lambda_q^\tau}{2\pi\Delta f}.
\]

慢时间维相邻 OFDM 符号相位差为

\[
e^{j2\pi f_{d,q}T_s},
\]

因此

\[
\hat f_{d,q}
=
\frac{\angle\lambda_q^f}{2\pi T_s}.
\]

之后与 SIC-DFT 一样，估计 \(\alpha_q\)，重构 ISI/ICI，并更新 \(Y_{\rm free}\)。

优点：

- 超分辨；
- 减少 DFT 栅格误差；
- 适合高精度距离-速度估计；
- 对近距离目标或 off-grid 目标更有优势。

缺点：

- 复杂度高；
- 需要协方差估计、空间平滑和模型阶数估计；
- 对模型误差和参数估计误差敏感。

---

## 5. 三种方法的对比

| 方法 | 核心操作 | 处理对象 | 主要解决问题 | 优点 | 缺点 |
|---|---|---|---|---|---|
| 相干补偿 | 将后续样点补到当前 FFT 窗口前部 | 单目标或某距离段的缺失样点 | 增强 beyond-CP 目标主峰，降低 ICI | 简单、低复杂度、无需改波形 | 需要补偿长度；近目标存在时可能恶化干扰 |
| Sliding Window | 分段移动 FFT 窗口，先消近目标再检测远目标 | 已检测目标的时域回波 | 恢复远目标功率，扩展检测距离 | 适合近远目标共存；不增加 CP | 多轮处理；误差传播；缓存需求高 |
| SIC-DFT | 固定窗口下 DFT 估计目标，重构 ISI/ICI | 结构化 \(Y_{\rm ISI},Y_{\rm ICI}\) | 降低 RDM 旁瓣和结构化泄露 | 易实现，兼容 RDM/CFAR | DFT 栅格限制，off-grid 性能有限 |
| SIC-ESPRIT | 固定窗口下子空间估计目标，重构 ISI/ICI | 结构化 \(Y_{\rm ISI},Y_{\rm ICI}\) | 高精度参数估计与干扰抵消 | 超分辨，精度高 | 复杂度高，对模型和阶数估计敏感 |

---

## 6. 从信号处理角度的本质差异

### 6.1 相干补偿

相干补偿认为 beyond-CP 的主要问题是：

\[
\text{当前 OFDM 符号没收完整。}
\]

所以它直接补样点，使有用信号从

\[
1-N_e
\]

增强到

\[
1-N_e+N_a.
\]

它是**样点级能量恢复方法**。

---

### 6.2 Sliding Window

Sliding Window 认为 beyond-CP 的主要问题是：

\[
\text{固定窗口对远距离目标不对齐。}
\]

所以它通过移动窗口，让远距离目标在新的局部窗口中变成“近距离目标”。它是**时域窗口重定位 + 目标回波 SIC 方法**。

---

### 6.3 固定窗口 SIC-DFT/ESPRIT

固定窗口 SIC 认为 beyond-CP 的主要问题是：

\[
\text{固定窗口下 OFDM 正交性被破坏，形成结构化 ISI/ICI。}
\]

所以它保持窗口不动，显式重构并消除

\[
Y_{\rm ISI},\quad Y_{\rm ICI}.
\]

它是**频域/矩阵域结构化干扰抵消方法**。

---

## 7. 如何选择方法？

### 7.1 若目标是扩展最大检测距离

优先考虑：

\[
\text{Sliding Window}
\]

因为它直接恢复远距离目标的主峰功率。

---

### 7.2 若目标较稀疏，且已知大致目标距离

可考虑：

\[
\text{Coherent Compensation}
\]

它简单有效，但不适合复杂近远目标共存场景。

---

### 7.3 若目标是降低 RDM 旁瓣、改善弱目标检测

优先考虑：

\[
\text{SIC-DFT}
\]

它兼容传统 RDM/CFAR，复杂度适中。

---

### 7.4 若目标是高精度距离-速度估计

优先考虑：

\[
\text{SIC-ESPRIT}
\]

尤其适合 off-grid 目标、近距离多目标和超分辨估计任务。

---

## 8. 关键结论

1. CP 长度给出的

\[
R_{\rm cp}=\frac{cT_{\rm cp}}{2}
\]

只是固定窗口下的无 ISI/ICI 感知距离，不必然等于最大可检测距离。

2. CP 不足会导致两种效应：

\[
\text{目标主峰功率下降}
+
\text{ISI/ICI 泄露}.
\]

3. 在随机数据、大子载波数和平均 range profile 分析下，ISI/ICI 可能被随机化并显著削弱，最大感知距离可以远超 CP-free range。

4. 在固定窗口 RDM 质量、旁瓣和参数估计角度，ISI/ICI 仍然是结构化干扰，不能简单忽略。

5. 三类方法的核心区别是：

\[
\text{相干补偿：补样点。}
\]

\[
\text{Sliding Window：移窗口并消除已检测回波。}
\]

\[
\text{SIC-DFT/ESPRIT：固定窗口下重构并消除结构化 ISI/ICI。}
\]

6. 从工程上看，三类方法可以互补：Sliding Window 用于扩展远距离检测，SIC-DFT/ESPRIT 用于清洁 RDM 和提升估计精度，相干补偿则适合作为低复杂度的特定距离段增强方法。

---

## 参考文献说明

本笔记主要基于以下讨论对象整理：

1. *OFDM-ISAC Beyond CP Limit: Performance Analysis and Mitigation Algorithms*  
   重点：结构化 ISI/ICI 建模、SINR/RDM/PSLR 分析、SIC-DFT 与 SIC-ESPRIT。

2. *How Does CP Length Affect the Sensing Range for OFDM-ISAC?*  
   重点：随机数据 mask 分析、range profile SINR、Sliding Window sensing。

3. Coherent compensation 类方法  
   重点：通过样点补偿提升 beyond-CP 目标的有用信号功率。
