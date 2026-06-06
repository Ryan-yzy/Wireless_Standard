# 控制 PAPR 的方法：专题学习笔记

> 主题：正交频分复用（Orthogonal Frequency Division Multiplexing, OFDM）/ NR 波形中的峰均功率比（Peak-to-Average Power Ratio, PAPR）问题及其控制方法。  
> 重点：PAPR 的数学来源、功放非线性影响、有失真方法、无失真方法、DFT-s-OFDM，以及导频/感知波形中的 PAPR 控制。

---

## 1. 为什么需要控制 PAPR？

### 1.1 OFDM 时域信号的基本形式

一个 $N$ 点 OFDM 符号的时域采样可以写为

$$
x[n]=\frac{1}{\sqrt{N}}\sum_{k=0}^{N-1}X_k e^{j2\pi kn/N},\quad n=0,1,\ldots,N-1,
$$

其中：

- $X_k$：第 $k$ 个子载波上的频域调制符号；
- $x[n]$：IFFT 后的时域采样；
- $N$：IFFT/FFT 点数。

OFDM 的时域信号是多个子载波信号的叠加。当很多子载波在某些采样点上相位接近时，会形成很大的瞬时峰值。

---

### 1.2 PAPR 的定义

PAPR 定义为

$$
\mathrm{PAPR}
=\frac{\max_n |x[n]|^2}{\mathbb E\{|x[n]|^2\}}.
$$

对于一个离散 OFDM 符号，也常写成

$$
\mathrm{PAPR}(\mathbf x)
=\frac{\|\mathbf x\|_\infty^2}{\frac{1}{N}\|\mathbf x\|_2^2}
=\frac{\max_n |x[n]|^2}{\frac{1}{N}\sum_{n=0}^{N-1}|x[n]|^2}.
$$

其中：

- 分子表示最大瞬时功率；
- 分母表示平均功率。

工程上常用 dB 表示：

$$
\mathrm{PAPR}_{\mathrm{dB}}
=10\log_{10}(\mathrm{PAPR}).
$$

---

### 1.3 OFDM 为什么容易出现高 PAPR？

如果 $X_k$ 近似独立，且子载波数 $N$ 较大，则根据中心极限定理，$x[n]$ 近似服从复高斯分布：

$$
x[n]\sim \mathcal{CN}(0,\sigma_x^2).
$$

于是 $|x[n]|^2$ 近似服从指数分布。若近似认为不同采样点独立，则 PAPR 超过门限 $z$ 的概率可近似为

$$
\Pr(\mathrm{PAPR}>z)
\approx
1-(1-e^{-z})^N.
$$

这说明：

$$
N\uparrow \quad \Rightarrow \quad \Pr(\mathrm{PAPR}>z)\uparrow.
$$

也就是说，子载波数越多，越容易出现高峰值。

---

## 2. PAPR 高对射频功放的影响

### 2.1 功率放大器的非线性模型

功率放大器（Power Amplifier, PA）不是理想线性器件。一个常见的基带等效多项式模型为

$$
y(t)=\alpha_1 x(t)+\alpha_3 x(t)|x(t)|^2+\alpha_5 x(t)|x(t)|^4+\cdots.
$$

其中：

- 一阶项 $\alpha_1x(t)$ 是理想线性放大；
- 三阶、五阶等高阶项代表非线性失真。

当输入信号峰值较大时，功放进入非线性区，会产生：

1. **带内失真**：星座点偏移，误差矢量幅度（Error Vector Magnitude, EVM）增大；
2. **带外泄漏**：非线性频谱再生，导致邻道泄漏比（Adjacent Channel Leakage Ratio, ACLR）恶化。

---

### 2.2 功放回退与功率效率

为避免功放进入非线性区，需要降低平均输入功率，使信号峰值远离饱和点。这称为功率回退。

输入回退（Input Back-Off, IBO）可定义为

$$
\mathrm{IBO}
=\frac{P_{\mathrm{sat}}}{P_{\mathrm{avg}}},
$$

其中：

- $P_{\mathrm{sat}}$：功放饱和功率；
- $P_{\mathrm{avg}}$：平均输入功率。

若 PAPR 较高，则需要更大的回退：

$$
P_{\mathrm{avg}}=\frac{P_{\mathrm{sat}}}{\mathrm{IBO}}.
$$

因此，PAPR 高会导致平均发射功率下降。对上行链路来说，这会直接影响覆盖能力：

$$
\mathrm{SNR}
=\frac{P_{\mathrm{tx}}G_{\mathrm{tx}}G_{\mathrm{rx}}}{L_{\mathrm{path}}N_0B}.
$$

当 $P_{\mathrm{tx}}$ 因功放回退而降低时，上行链路 SNR 和覆盖范围都会下降。

---

### 2.3 工程直觉

OFDM 的高 PAPR 可以理解为：多个子载波偶尔在时域“同相叠加”，形成很高的瞬时峰值。功放为了不被这些峰值推入非线性区，只能降低平均输出功率。

因此，高 PAPR 的真正代价不是“峰值高”本身，而是：

$$
\boxed{\text{为了容纳峰值，必须牺牲平均发射功率和功率效率。}}
$$

这也是 NR 上行除了支持 CP-OFDM 之外，还支持 DFT-s-OFDM 的重要原因。

---

## 3. PAPR 控制方法的总体分类

PAPR 控制方法大致可以分为三类：

1. **有失真方法**：直接修改时域信号，降低峰值，但会引入失真；
2. **无失真方法**：不直接破坏数据符号，而是利用相位、冗余、码字、保留子载波等自由度降低峰值；
3. **波形级方法**：从波形结构本身降低 PAPR，例如 DFT-s-OFDM 或低 PAPR 导频/感知波形设计。

可以统一写成优化问题：

$$
\min_{\widetilde{\mathbf X}\in \mathcal E(\mathbf X)}
\mathrm{PAPR}(\mathbf F^H\widetilde{\mathbf X}),
$$

其中：

- $\mathbf F^H$：IFFT 矩阵；
- $\mathbf X$：原始频域符号；
- $\widetilde{\mathbf X}$：经过 PAPR 控制后的频域符号；
- $\mathcal E(\mathbf X)$：允许的等价或近似等价波形集合。

不同方法的本质区别在于如何定义 $\mathcal E(\mathbf X)$。

---

# 第一部分：有失真 PAPR 控制方法

---

## 4. 限幅 Clipping

### 4.1 基本思想

限幅是最直接的 PAPR 降低方法：当时域信号幅度超过门限 $A$ 时，将其截断到 $A$。

$$
\tilde{x}[n]
=\begin{cases}
x[n], & |x[n]|\le A,\\
A e^{j\angle x[n]}, & |x[n]|>A.
\end{cases}
$$

限幅后的信号可以写成

$$
\tilde{x}[n]=x[n]+d[n],
$$

其中 $d[n]$ 是限幅噪声。

---

### 4.2 频域影响

对限幅信号做 FFT：

$$
\tilde{X}_k=X_k+D_k.
$$

其中 $D_k$ 是限幅噪声在第 $k$ 个子载波上的分量。

因此，限幅会造成：

1. **带内失真**：$D_k$ 污染数据子载波；
2. **带外泄漏**：非线性截断导致频谱再生。

---

### 4.3 限幅率

常用限幅率（Clipping Ratio, CR）定义为

$$
\mathrm{CR}=\frac{A}{\sqrt{\mathbb E\{|x[n]|^2\}}}.
$$

CR 越小，削峰越强，PAPR 越低，但失真越大。

---

### 4.4 工程评价

限幅的优点是：

- 简单；
- 不需要边信息；
- 可以直接降低峰值。

缺点是：

- 带内 EVM 增大；
- 带外泄漏增大；
- 可能需要滤波；
- 滤波后可能产生峰值再生，需要迭代 clipping + filtering。

工程上，限幅适合对复杂度要求低、允许一定失真的场景；但对于高阶调制、严格频谱掩膜或感知波形，不宜过强使用。

---

## 5. 峰值加窗 Peak Windowing

### 5.1 基本思想

峰值加窗不是直接截断峰值，而是在峰值附近乘一个平滑窗函数，使峰值逐渐衰减。

若 $n_0$ 是峰值位置，则可写成

$$
\tilde{x}[n]=x[n]w[n-n_0],
$$

其中 $w[n]$ 是局部衰减窗。

---

### 5.2 工程直觉

与 clipping 相比，windowing 更平滑，因此带外泄漏通常更容易控制。但它仍然改变了原始时域波形，因此仍属于有失真方法。

---

## 6. 压扩 Companding

### 6.1 基本思想

压扩方法通过非线性幅度变换压缩大幅度样本、扩展小幅度样本，使幅度分布更均匀。

例如 $\mu$-law 压扩可写为

$$
\tilde{x}[n]
=\frac{\ln(1+\mu |x[n]|)}{\ln(1+\mu)}e^{j\angle x[n]}.
$$

### 6.2 优缺点

优点：

- PAPR 降低效果明显；
- 可控制幅度分布。

缺点：

- 引入非线性失真；
- 可能需要接收端反压扩；
- 对噪声和信道估计可能更敏感。

---

# 第二部分：无失真 PAPR 控制方法

---

## 7. 无失真方法的本质

所谓“无失真”不是完全没有代价，而是指不直接像 clipping 那样加入削峰噪声。其目标是保持数据符号可恢复：

$$
\widetilde{X}_k=X_k
\quad\text{或}\quad
\widetilde{X}_k\text{ 可等价判决为 }X_k.
$$

这些方法通常利用以下自由度：

- 相位自由度；
- 子块组合自由度；
- 保留子载波；
- 星座扩展空间；
- 低 PAPR 码字；
- DFT 扩展结构。

---

## 8. 选择映射 SLM

### 8.1 基本思想

选择映射（Selected Mapping, SLM）的思想是：为同一组数据生成多个等价候选 OFDM 信号，然后选择 PAPR 最低的一个发送。

给定原始频域符号

$$
\mathbf X=[X_0,X_1,\ldots,X_{N-1}]^T,
$$

构造第 $u$ 个相位旋转序列

$$
\mathbf P^{(u)}=[P_0^{(u)},P_1^{(u)},\ldots,P_{N-1}^{(u)}]^T,
$$

其中

$$
|P_k^{(u)}|=1.
$$

候选频域符号为

$$
\mathbf X^{(u)}=\mathbf X\odot \mathbf P^{(u)}.
$$

对应时域信号为

$$
\mathbf x^{(u)}=\mathbf F^H\mathbf X^{(u)}.
$$

选择

$$
u^\star=\arg\min_u \mathrm{PAPR}(\mathbf x^{(u)}).
$$

---

### 8.2 SLM 的 PAPR 分布直觉

若单个候选超过门限 $z$ 的概率为

$$
\Pr(\mathrm{PAPR}>z),
$$

则 $U$ 个候选都超过 $z$ 的概率近似为

$$
\Pr(\mathrm{PAPR}_{\min}>z)
\approx
\left[\Pr(\mathrm{PAPR}>z)\right]^U.
$$

因此候选数 $U$ 越大，找到低 PAPR 波形的概率越高。

---

### 8.3 边信息

接收端必须知道发射端选择了哪个 $u^\star$。边信息比特数为

$$
B_{\mathrm{SI}}=\log_2 U.
$$

若边信息出错，接收端无法正确反旋转，整块数据可能解调失败。

---

### 8.4 优缺点

优点：

- 不引入 clipping 噪声；
- PAPR 降低效果随候选数增加而增强。

缺点：

- 需要多次 IFFT；
- 需要可靠边信息；
- 与导频、MIMO、信道估计结合复杂。

---

## 9. 部分传输序列 PTS

### 9.1 基本思想

部分传输序列（Partial Transmit Sequence, PTS）把频域符号分成多个子块，然后给每个子块乘一个相位因子，使时域峰值尽量互相抵消。

将频域符号分解为

$$
\mathbf X=\sum_{v=1}^{V}\mathbf X_v,
$$

其中 $\mathbf X_v$ 是第 $v$ 个子块。

每个子块分别 IFFT：

$$
\mathbf x_v=\mathbf F^H\mathbf X_v.
$$

组合信号为

$$
\mathbf x=\sum_{v=1}^{V}b_v\mathbf x_v,
$$

其中

$$
b_v\in\mathcal B=\left\{e^{j2\pi q/W},\ q=0,1,\ldots,W-1\right\}.
$$

PTS 优化问题为

$$
\{b_v^\star\}
=\arg\min_{\{b_v\}}
\mathrm{PAPR}\left(\sum_{v=1}^{V}b_v\mathbf x_v\right).
$$

因为公共相位不影响 PAPR，通常固定

$$
b_1=1.
$$

---

### 9.2 PTS 需要的边信息

真正需要告诉接收端的是

$$
b_2^\star,b_3^\star,\ldots,b_V^\star.
$$

若每个 $b_v$ 有 $W$ 种选择，则边信息比特数为

$$
B_{\mathrm{SI}}=(V-1)\log_2 W.
$$

例如：

- $V=4, W=4$，则 $B_{\mathrm{SI}}=6$ bits；
- $V=8, W=4$，则 $B_{\mathrm{SI}}=14$ bits。

---

### 9.3 接收端检测

设第 $v$ 个子块对应子载波集合 $\mathcal K_v$，则发送端相当于对这些子载波施加相位旋转：

$$
\widetilde X_k=b_vX_k,\quad k\in\mathcal K_v.
$$

经过信道后：

$$
Y_k=H_kb_vX_k+N_k.
$$

如果接收端知道 $b_v$，则均衡和反旋转为

$$
\widehat X_k=b_v^*\frac{Y_k}{\widehat H_k}.
$$

MMSE 形式为

$$
\widehat X_k
=b_v^*
\frac{\widehat H_k^*}{|\widehat H_k|^2+\sigma^2/P_X}Y_k.
$$

之后进行普通星座判决。

标准检测流程为

$$
\boxed{
\text{FFT}
\rightarrow
\text{信道估计}
\rightarrow
\text{均衡}
\rightarrow
\text{读取边信息}
\rightarrow
\text{子块反旋转}
\rightarrow
\text{星座判决}
}
$$

---

### 9.4 边信息错误的影响

如果实际使用 $b_v$，但接收端误认为 $\hat b_v$，则反旋转后变成

$$
\widehat X_k=\hat b_v^*b_vX_k+\text{noise}.
$$

当 $\hat b_v\ne b_v$ 时，整个子块星座会被错误旋转，尤其对 16QAM、64QAM、256QAM 等高阶调制非常严重。

因此 PTS 的边信息必须强保护，例如：

- 使用控制信道发送；
- 加 CRC；
- 使用更强纠错编码；
- 嵌入保留子载波；
- 使用导频辅助估计；
- 使用盲检测。

---

### 9.5 不显式发送边信息的可能方式

#### 方式一：盲检测

接收端枚举所有可能的相位组合，选择使均衡后符号最接近星座点的组合：

$$
\widehat{\mathbf b}
=\arg\min_{\mathbf b\in\mathcal B^{V-1}}
\sum_k
\left|
\tilde X_k(\mathbf b)-\mathcal Q(\tilde X_k(\mathbf b))
\right|^2,
$$

其中 $\mathcal Q(\cdot)$ 是星座判决函数。

缺点是复杂度约为

$$
W^{V-1}.
$$

#### 方式二：导频辅助估计

若每个子块中包含导频，且导频也乘以相同 $b_v$，则

$$
Y_p=H_pb_vP_p+N_p.
$$

若已知 $\widehat H_p$，可估计

$$
\widehat b_v
=\frac{
\sum_{p\in\mathcal P_v}Y_p\widehat H_p^*P_p^*
}{
\left|
\sum_{p\in\mathcal P_v}Y_p\widehat H_p^*P_p^*
\right|
}.
$$

然后量化到 $\mathcal B$ 中。

需要注意：在标准化系统中，导频通常有严格生成规则，不能随意旋转，否则会影响信道估计、端口定义和参考信号一致性。

---

## 10. 音调保留 TR

### 10.1 基本思想

音调保留（Tone Reservation, TR）预留一部分子载波不传数据，而是专门用于生成削峰信号。

设数据子载波集合为 $\mathcal D$，保留子载波集合为 $\mathcal R$：

$$
X_k=
\begin{cases}
D_k, & k\in\mathcal D,\\
C_k, & k\in\mathcal R.
\end{cases}
$$

时域信号为

$$
\mathbf x=\mathbf x_{\mathcal D}+\mathbf x_{\mathcal R}
=\mathbf F_{\mathcal D}^H\mathbf D+
\mathbf F_{\mathcal R}^H\mathbf C.
$$

优化目标为

$$
\min_{\mathbf C}
\left\|
\mathbf x_{\mathcal D}+\mathbf F_{\mathcal R}^H\mathbf C
\right\|_\infty.
$$

也可以写为凸优化形式：

$$
\min_{\mathbf C,t}\quad t
$$

$$
\text{s.t.}\quad
\left|
x_{\mathcal D}[n]+(\mathbf F_{\mathcal R}^H\mathbf C)[n]
\right|
\le t,
\quad n=0,1,\ldots,N-1.
$$

---

### 10.2 为什么 TR 不算数据失真？

因为数据子载波上的符号未被改变：

$$
\widetilde X_k=D_k,\quad k\in\mathcal D.
$$

接收端只解调数据子载波即可，保留子载波可以忽略。

---

### 10.3 优缺点

优点：

- 不需要边信息；
- 不污染数据子载波；
- 接收端处理简单。

缺点：

- 保留子载波不能传数据，降低频谱效率；
- 发射端需要优化 $\mathbf C$；
- 保留子载波本身也会消耗功率；
- 保留子载波过少时削峰能力有限。

---

## 11. 音调注入 TI

### 11.1 基本思想

音调注入（Tone Injection, TI）利用 QAM 星座的等价点。把原始星座点移动到更远的等价星座位置，使接收端仍能判决为原始数据，但时域峰值降低。

可写为

$$
\widetilde X_k=X_k+\Delta_k,
$$

其中 $\Delta_k$ 是满足等价判决条件的星座平移。

例如可令

$$
\Delta_k=2D(p_k+jq_k),
$$

其中 $p_k,q_k$ 是整数，$D$ 与 QAM 星座边界有关。

优化问题为

$$
\min_{\{\Delta_k\}}
\mathrm{PAPR}\left(\mathbf F^H(\mathbf X+\boldsymbol\Delta)\right).
$$

---

### 11.2 工程评价

优点：

- 不需要保留子载波；
- 不需要显式边信息；
- 不产生 clipping 噪声。

缺点：

- 平均功率可能增加；
- 接收端判决需要考虑等价星座；
- 对高阶 QAM 更适合；
- 优化复杂。

本质是：

$$
\boxed{\text{用平均功率增加换取峰值功率降低。}}
$$

---

## 12. 主动星座扩展 ACE

### 12.1 基本思想

主动星座扩展（Active Constellation Extension, ACE）允许外圈星座点沿着不减小判决距离的方向移动。

设

$$
\widetilde X_k=X_k+\Delta_k,
$$

要求

$$
\widetilde X_k\in\mathcal A_k,
$$

其中 $\mathcal A_k$ 是该星座点的允许扩展区域。

优化为

$$
\min_{\{\Delta_k\}}
\left\|
\mathbf F^H(\mathbf X+\boldsymbol\Delta)
\right\|_\infty.
$$

---

### 12.2 与 TI 的区别

- TI：利用等价星座点，可能移动较远；
- ACE：只沿不会降低最小判决距离的“安全方向”移动。

ACE 更温和，但也可能增加平均功率和 EVM。

---

## 13. 编码法

### 13.1 基本思想

编码法从源头上禁止高 PAPR 码字，只允许发送低 PAPR 的频域符号组合。

设低 PAPR 码本为

$$
\mathcal C=\{\mathbf X^{(1)},\mathbf X^{(2)},\ldots,\mathbf X^{(M_c)}\},
$$

并满足

$$
\mathrm{PAPR}(\mathbf F^H\mathbf X^{(i)})\le \gamma,
\quad i=1,2,\ldots,M_c.
$$

每个 OFDM 符号可承载

$$
\log_2 M_c
$$

比特。

相对未编码系统的速率为

$$
R=\frac{\log_2 M_c}{N\log_2 M},
$$

其中 $M$ 是调制阶数。

---

### 13.2 工程评价

优点：

- 理论上干净；
- 可保证 PAPR 上界；
- 不需要削峰。

缺点：

- 码率损失；
- 码本设计复杂；
- 大规模 OFDM 中实现困难；
- 与现有信道编码、调制、HARQ 结合复杂。

---

# 第三部分：波形级低 PAPR 方法

---

## 14. DFT-s-OFDM

### 14.1 基本结构

DFT-s-OFDM 的流程为

$$
\mathbf d
\rightarrow
\mathrm{DFT}
\rightarrow
\text{子载波映射}
\rightarrow
\mathrm{IFFT}
\rightarrow
\text{加 CP}.
$$

先对 $M$ 个数据符号做 DFT：

$$
D[k]=\frac{1}{\sqrt M}\sum_{m=0}^{M-1}d[m]e^{-j2\pi mk/M}.
$$

然后将 $D[k]$ 映射到 $N$ 个 OFDM 子载波中的一部分，再做 IFFT：

$$
x[n]
=\frac{1}{\sqrt N}\sum_{k\in\mathcal A}D[k]e^{j2\pi kn/N}.
$$

---

### 14.2 为什么 DFT-s-OFDM 的 PAPR 较低？

若 $M=N$ 且连续映射，则 DFT 和 IFFT 近似抵消：

$$
x[n]\approx d[n].
$$

因此时域信号更接近单载波序列。

若 $M<N$ 且局部连续映射，则可理解为

$$
x[n]=\sum_{m=0}^{M-1}d[m]g[n-mN/M],
$$

其中 $g[\cdot]$ 是由频域映射产生的插值滤波器。

因此，DFT-s-OFDM 的时域包络比 CP-OFDM 更平滑，PAPR 更低。

---

### 14.3 工程意义

DFT-s-OFDM 对上行尤其重要，因为 UE 更受功放效率、电池容量和散热限制。降低 PAPR 可以减少功放回退，提高平均发射功率，从而改善覆盖。

其代价包括：

- 频域调度灵活性弱于 CP-OFDM；
- MIMO 多流支持不如 CP-OFDM 灵活；
- 接收机处理和资源分配约束更多。

---

## 15. 单载波与低 PAPR 脉冲成形

除了 DFT-s-OFDM，还可以采用更一般的单载波调制和脉冲成形方法。

发射信号可写为

$$
x(t)=\sum_m d[m]p(t-mT),
$$

其中 $p(t)$ 是成形滤波器。

若 $d[m]$ 为恒模调制，如 PSK，且 $p(t)$ 设计合理，则时域包络相对平滑。

但单载波系统面对频率选择性信道时，均衡复杂度可能更高，因此 DFT-s-OFDM 是一种折中：

$$
\boxed{\text{保留 OFDM 的频域均衡便利性，同时获得近似单载波的低 PAPR 特性。}}
$$

---

# 第四部分：导频/感知波形中的 PAPR 控制

---

## 16. 导频符号是否也存在 PAPR 高的问题？

答案是：存在。

只要导频最终通过多子载波 OFDM 发射，其时域信号仍是

$$
x[n]=\frac{1}{\sqrt N}\sum_{k\in\mathcal A}X_k e^{j2\pi kn/N}.
$$

即使 $X_k$ 是已知导频而不是随机数据，也可能出现多个子载波同相叠加，导致高 PAPR。

因此：

$$
\boxed{\text{导频已知} \ne \text{PAPR 一定低。}}
$$

---

## 17. 感知导频 PAPR 高的影响

如果导频用于雷达感知或 ISAC 感知，PAPR 高会带来额外问题：

1. 功放回退降低平均发射功率，导致回波 SNR 下降；
2. 功放非线性会破坏已知导频结构；
3. 匹配滤波输出旁瓣升高；
4. 虚警概率可能上升；
5. 距离、多普勒、角度估计可能出现偏差；
6. 带外泄漏影响频谱合规。

因此，感知场景中控制 PAPR 不只是为了通信误码率，更是为了保持雷达波形的相关性和参数估计性能。

---

## 18. 低 PAPR 感知导频设计的基本优化问题

若导频频域符号为 $\mathbf X$，可以设计

$$
\min_{\mathbf X}
\mathrm{PAPR}(\mathbf F^H\mathbf X)
+\lambda\cdot \mathrm{PSL}(\mathbf F^H\mathbf X),
$$

其中 PSL 是峰值旁瓣电平（Peak Sidelobe Level）：

$$
\mathrm{PSL}
=\max_{\tau\ne 0}
\left|
\sum_n x[n]x^*[n-\tau]
\right|.
$$

常见约束包括：

$$
|X_k|=1,\quad k\in\mathcal A,
$$

或

$$
X_k\in\mathcal QPSK,
$$

以及

$$
X_k=0,\quad k\notin\mathcal A.
$$

设计目标通常是多目标折中：

$$
\boxed{
\text{低 PAPR}
+\text{低相关旁瓣}
+\text{频谱合规}
+\text{通信可兼容}
}
$$

---

## 19. ZC/CAZAC 序列

### 19.1 基本性质

Zadoff-Chu（ZC）序列是一类恒包络零自相关序列（Constant Amplitude Zero Auto-Correlation, CAZAC）。其典型形式为

$$
x[n]=e^{-j\pi q n(n+1)/N},
\quad n=0,1,\ldots,N-1,
$$

其中 $q$ 与 $N$ 互素。

它满足恒模性质：

$$
|x[n]|=1.
$$

因此理想 PAPR 为

$$
\mathrm{PAPR}=1=0\ \mathrm{dB}.
$$

同时，其周期自相关满足

$$
\sum_n x[n]x^*[n-\tau]=0,
\quad \tau\ne 0.
$$

---

### 19.2 工程意义

ZC/CAZAC 序列适合同步、随机接入、信道探测和感知导频，因为它兼具：

- 低 PAPR；
- 好相关峰；
- 好多用户循环移位复用能力。

但要注意：当 ZC 序列被映射到 OFDM 子载波、插零、截断、梳状映射或多天线预编码之后，最终时域 PAPR 未必仍严格为 0 dB，需要结合具体映射分析。

---

## 20. Newman/Schroeder 相位设计

### 20.1 基本思想

若导频只要求每个子载波恒模，即

$$
X_k=e^{j\theta_k},
$$

则可以通过设计 $\theta_k$ 来降低 IFFT 后的时域峰值。

典型思路是让不同子载波在时域上尽量不要大规模同相叠加。

例如一种二次相位结构为

$$
\theta_k=\pi\frac{k(k-1)}{N}.
$$

这样可以使时域能量更加均匀分布。

---

### 20.2 优点与限制

优点：

- 不改变子载波幅度；
- 适合导频和感知波形；
- 不需要接收端知道额外边信息，只要序列预定义即可。

限制：

- 与具体资源映射有关；
- 需要同时考虑相关旁瓣；
- 对多天线、多端口、多用户复用需要额外设计。

---

## 21. Golay 互补序列

### 21.1 基本性质

Golay 互补序列对 $(a[n],b[n])$ 满足

$$
R_a[\tau]+R_b[\tau]=0,\quad \tau\ne 0,
$$

其中 $R_a[\tau]$ 和 $R_b[\tau]$ 分别是两个序列的非周期自相关。

因此，若连续发送两段互补序列并将相关结果相加，则旁瓣可以互相抵消：

$$
R_{\mathrm{sum}}[\tau]
=R_a[\tau]+R_b[\tau].
$$

---

### 21.2 工程意义

Golay 序列适合需要低旁瓣的同步和感知场景，也可用于构造低 PAPR OFDM 符号。

但其限制是：

- 通常需要成对发送；
- 对多普勒敏感性需要额外考虑；
- 在高速目标感知中，两个互补符号之间的相位变化可能破坏互补性。

---

## 22. 感知场景中的 PTS/SLM 使用

如果基站发射已知感知波形并由自己接收回波，即单站感知，则 PTS/SLM 的“边信息问题”相对较小，因为基站知道自己最终发送的实际波形。

若实际发射波形为

$$
\tilde{x}[n]=\mathrm{IFFT}\{\tilde X_k\},
$$

感知匹配滤波应使用实际波形：

$$
r[\tau]=\sum_n y[n]\tilde{x}^*[n-\tau].
$$

这点非常重要：如果发射端为了降低 PAPR 改变了导频相位，而接收端仍用原始导频做匹配，会造成相关失配。

因此：

$$
\boxed{
\text{感知端必须使用“实际发射波形”进行匹配处理。}
}
$$

---

# 第五部分：方法对比与工程选择

---

## 23. 各类 PAPR 控制方法对比

| 方法 | 是否引入数据失真 | 是否需要边信息 | 主要代价 | 适用场景 |
|---|---|---|---|---|
| Clipping | 是 | 否 | EVM、带外泄漏 | 简单系统、低复杂度场景 |
| Peak Windowing | 是 | 否 | 局部失真、旁瓣变化 | 温和削峰 |
| Companding | 是 | 可能需要 | 非线性失真、接收复杂度 | 对幅度分布可控的系统 |
| SLM | 否 | 是 | 多次 IFFT、边信息 | 理论研究、可承受复杂度系统 |
| PTS | 否 | 是 | 组合搜索、边信息 | PAPR 降低效果较好但复杂 |
| TR | 否 | 否 | 牺牲子载波、优化复杂 | 不想发送边信息的系统 |
| TI | 否 | 否/少量 | 平均功率增加 | QAM 系统 |
| ACE | 否 | 否 | EVM、平均功率增加 | 高阶 QAM、外圈星座可扩展 |
| 编码法 | 否 | 否 | 码率损失、码本复杂 | 小规模或理论设计 |
| DFT-s-OFDM | 波形级低 PAPR | 否 | 调度/MIMO 灵活性下降 | NR 上行、覆盖受限 UE |
| ZC/CAZAC | 低 PAPR 序列 | 否 | 映射受限 | 导频、同步、感知 |
| Golay | 低旁瓣/低 PAPR | 否 | 需成对设计、多普勒敏感 | 感知、同步、短帧 |

---

## 24. 选择方法时应考虑的指标

### 24.1 PAPR 降低能力

通常用互补累积分布函数（Complementary Cumulative Distribution Function, CCDF）评价：

$$
\mathrm{CCDF}(z)=\Pr(\mathrm{PAPR}>z).
$$

例如比较不同方法在 $\mathrm{CCDF}=10^{-3}$ 处的 PAPR 降低量。

---

### 24.2 误码率与 EVM

有失真方法会增加 EVM：

$$
\mathrm{EVM}
=\sqrt{
\frac{\mathbb E\{|\tilde X_k-X_k|^2\}}{\mathbb E\{|X_k|^2\}}
}.
$$

对高阶调制，EVM 约束通常更严格。

---

### 24.3 带外泄漏

非线性削峰和功放失真会增加带外发射：

$$
\mathrm{ACLR}
=\frac{P_{\mathrm{in-band}}}{P_{\mathrm{adjacent}}}.
$$

若 ACLR 不满足频谱掩膜，需要更强滤波或更大功放回退。

---

### 24.4 复杂度

- SLM：约需要 $U$ 次 IFFT；
- PTS：搜索复杂度约 $W^{V-1}$；
- TR/ACE/TI：涉及优化；
- DFT-s-OFDM：结构性复杂度较低，适合标准化实现。

---

### 24.5 对 NR/ISAC 的兼容性

在 NR 或 ISAC 系统中，还要考虑：

- DM-RS/CSI-RS/SRS/PTRS 是否允许被修改；
- 信道估计是否会受到影响；
- 多天线预编码是否兼容；
- 感知匹配滤波是否知道实际发射波形；
- 是否满足频谱掩膜和 EVM 要求；
- 是否增加控制开销或边信息可靠性问题。

---

## 25. 面向 NR 上行的建议

对于 NR 上行链路，最现实、最重要的低 PAPR 方法是：

$$
\boxed{\text{DFT-s-OFDM}}
$$

原因是：

1. UE 功放效率最敏感；
2. DFT-s-OFDM 不需要额外边信息；
3. 标准支持程度高；
4. 可以显著降低 PAPR；
5. 对覆盖受限场景有直接收益。

其他复杂 PAPR 算法虽然理论上有效，但在标准化链路中会受到导频、控制信道、MIMO、调度和边信息可靠性等限制。

---

## 26. 面向感知导频/ISAC 的建议

如果波形主要用于感知或通信感知融合，不建议简单使用强 clipping。更推荐从波形设计本身控制 PAPR：

1. 使用 ZC/CAZAC 等恒模序列；
2. 使用 DFT-s-OFDM 形式的导频；
3. 设计恒模子载波相位，如 Newman/Schroeder 相位；
4. 使用 Golay 互补序列控制相关旁瓣；
5. 使用 TR 保留部分子载波削峰；
6. 若使用 SLM/PTS，感知接收端必须使用实际发射波形进行匹配。

感知波形设计应同时关注：

$$
\boxed{
\text{PAPR}
+\text{相关旁瓣}
+\text{多普勒鲁棒性}
+\text{频谱约束}
+\text{通信兼容性}
}
$$

---

# 27. 总结

PAPR 控制的核心矛盾是：

$$
\boxed{
\text{降低峰值}
\quad\text{vs.}\quad
\text{保持数据、频谱、复杂度和标准兼容性}
}
$$

有失真方法简单直接，但会引入削峰噪声和带外泄漏；无失真方法更优雅，但通常需要边信息、冗余资源或较高复杂度；波形级方法如 DFT-s-OFDM 和低 PAPR 导频设计，则更适合工程系统。

对通信系统：

$$
\boxed{
PAPR 控制的最终目标是减少功放回退，提高平均发射功率和能效。
}
$$

对感知/ISAC 系统：

$$
\boxed{
PAPR 控制还会影响匹配滤波旁瓣、目标检测、参数估计和频谱合规。
}
$$

因此，在 NR/ISAC 中，PAPR 控制不应被看作单独的“削峰算法”，而应作为波形设计、射频硬件、参考信号设计和接收机处理共同优化的问题。

---

## 28. 一句话记忆

> OFDM 的高 PAPR 来自多子载波同相叠加；控制 PAPR 的方法要么削掉峰值，要么改变叠加方式，要么从波形结构上避免峰值。工程上，UE 上行最实用的是 DFT-s-OFDM；感知导频更适合直接设计低 PAPR、低旁瓣的已知序列。
