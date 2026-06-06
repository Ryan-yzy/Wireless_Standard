# OFDM 变式与滤波器组波形学习笔记

> 主题：W-OFDM、F-OFDM、UF-OFDM、FBMC/FBMC-OQAM 的数学原理与性能比较  
> 关键词：带外泄漏（OOB）、频域局部化、滤波器拖尾、正交性、OQAM、功率效率、时变信道、相位噪声

---

## 1. 从统一多载波模型理解几种波形

多载波波形可以统一写成

$$
x[n]=\sum_m\sum_{k=0}^{N-1} d_{k,m}\, p_k[n-mM] e^{j2\pi kn/N},
$$

其中：

- $d_{k,m}$：第 $m$ 个时间位置、第 $k$ 个子载波上的数据符号；
- $N$：IFFT/FFT 点数或子载波数；
- $M$：相邻多载波符号的时间间隔；
- $p_k[n]$：第 $k$ 个子载波对应的脉冲成形滤波器；
- $e^{j2\pi kn/N}$：把基带脉冲调制到第 $k$ 个子载波频率上。

几类波形的核心区别，本质上就是 **脉冲/滤波器 $p_k[n]$ 的设计方式不同**：

| 波形 | 核心操作 | 滤波粒度 |
|---|---|---|
| CP-OFDM | 矩形窗 + CP | 整个 OFDM 符号 |
| W-OFDM | 对 OFDM 符号边缘加平滑窗 | 符号级 |
| F-OFDM | 对 OFDM 子带或整带信号滤波 | 子带级/整带级 |
| UF-OFDM | 对一组相邻子载波滤波 | 子带组级 |
| FBMC | 每个子载波由频移后的原型滤波器形成 | 子载波级 |

可以记为：

$$
\text{CP-OFDM}
\rightarrow
\text{W-OFDM}
\rightarrow
\text{F-OFDM}
\rightarrow
\text{UF-OFDM}
\rightarrow
\text{FBMC},
$$

对应的是滤波逐渐精细、频域泄漏逐渐减小，但复杂度和时域扩展逐渐增加。

---

## 2. CP-OFDM：基准波形

普通 OFDM 的一个时域符号为

$$
s_l[n]=\frac{1}{\sqrt N}\sum_{k=0}^{N-1}X_{k,l}e^{j2\pi kn/N},\quad 0\le n<N.
$$

加入循环前缀（Cyclic Prefix, CP）后：

$$
x_l[n]=s_l[(n-N_{\mathrm{CP}})\bmod N],\quad 0\le n<N+N_{\mathrm{CP}}.
$$

CP 的作用是把信道线性卷积近似转化为循环卷积。若信道长度不超过 CP，则频域上有

$$
Y_{k,l}=H_kX_{k,l}+W_{k,l}.
$$

因此接收端只需要一拍均衡：

$$
\widehat X_{k,l}=\frac{Y_{k,l}}{H_k}.
$$

### CP-OFDM 的问题

CP-OFDM 的符号时间窗接近矩形窗：

$$
p_{\mathrm{rect}}(t)=
\begin{cases}
1, & 0\le t<T,\\
0, & \text{otherwise}.
\end{cases}
$$

矩形窗的频域响应是 sinc 形：

$$
P_{\mathrm{rect}}(f)\propto \frac{\sin(\pi fT)}{\pi fT}.
$$

sinc 旁瓣衰减较慢，因此 CP-OFDM 的带外泄漏（Out-of-Band Emission, OOB）较高。

---

## 3. W-OFDM：Windowed OFDM

### 3.1 基本思想

W-OFDM 的核心是：**把 OFDM 符号边缘从硬截断变成平滑过渡**。

普通 CP-OFDM 可以理解为

$$
x[n]=s^{\mathrm{CP}}[n]w_{\mathrm{rect}}[n].
$$

W-OFDM 改为

$$
x^{\mathrm{W}}[n]=s^{\mathrm{CP}}[n]w[n],
$$

其中 $w[n]$ 是平滑窗，例如升余弦窗。

一个典型升余弦窗可以写成

$$
w[n]=
\begin{cases}
\frac{1}{2}\left[1-\cos\left(\frac{\pi n}{N_{\mathrm{roll}}}\right)\right],
&0\le n<N_{\mathrm{roll}},\\[4pt]
1,&N_{\mathrm{roll}}\le n<N_{\mathrm{sym}}-N_{\mathrm{roll}},\\[4pt]
\frac{1}{2}\left[1-\cos\left(\frac{\pi (N_{\mathrm{sym}}-n)}{N_{\mathrm{roll}}}\right)\right],
&N_{\mathrm{sym}}-N_{\mathrm{roll}}\le n<N_{\mathrm{sym}}.
\end{cases}
$$

### 3.2 为什么加窗可以降低 OOB？

时域乘窗对应频域卷积：

$$
X^{\mathrm{W}}(f)=X^{\mathrm{OFDM}}(f)*W(f).
$$

平滑窗比矩形窗边缘连续性更好，因此频域旁瓣更低。直观上：

- CP-OFDM 是“硬切断”；
- W-OFDM 是“渐入渐出”；
- 边缘越平滑，高频泄漏越少。

### 3.3 代价

窗函数滚降区会占用保护时间。若滚降长度为 $N_{\mathrm{roll}}$，有效 CP 大致变成

$$
N_{\mathrm{CP},eff}=N_{\mathrm{CP}}-N_{\mathrm{roll}}.
$$

为了避免严重符号间干扰（Inter-Symbol Interference, ISI），需要

$$
N_{\mathrm{CP},eff}\ge L_h-1,
$$

其中 $L_h$ 是信道长度。因此 W-OFDM 的折中是：

$$
\text{更低 OOB}\quad \Longleftrightarrow\quad \text{更短有效 CP 或更长保护开销}.
$$

---

## 4. F-OFDM：Filtered OFDM

### 4.1 基本思想

F-OFDM 的核心是：**先生成 OFDM 信号，再对整个子带或整带信号进行滤波**。

若 CP-OFDM 信号为 $x^{\mathrm{CP}}[n]$，则 F-OFDM 为

$$
x^{\mathrm{F}}[n]=x^{\mathrm{CP}}[n]*g[n],
$$

其中 $g[n]$ 是 FIR 滤波器。频域上为

$$
X^{\mathrm{F}}(f)=X^{\mathrm{CP}}(f)G(f).
$$

如果普通 OFDM 的频域信号写成

$$
X_{\mathrm{OFDM}}(f)=\sum_{k\in\mathcal K}X_kQ(f-k\Delta f),
$$

其中 $Q(f)$ 是矩形窗导致的 sinc 频谱，那么 F-OFDM 后变为

$$
X_{\mathrm{F}}(f)=G(f)\sum_{k\in\mathcal K}X_kQ(f-k\Delta f).
$$

从单个子载波看，好像是

$$
X_k Q(f-k\Delta f)G(f),
$$

即 sinc 频谱又乘了一个滤波器频响。但要注意：**F-OFDM 是对整个子带乘同一个 $G(f)$，不是给每个子载波单独设计一个 $G(f-k\Delta f)$。**

### 4.2 F-OFDM 会不会破坏 OFDM 子载波正交性？

理想情况下不会产生 FBMC-OQAM 那种结构性的非复正交问题。

原因是 F-OFDM 仍然保留 OFDM 的 FFT/CP/QAM 结构。若滤波器可以被 CP 或保护间隔吸收，滤波近似为循环卷积，则 DFT 子载波仍然是滤波器的特征向量。

令 $u_k[n]=\frac{1}{\sqrt N}e^{j2\pi kn/N}$，循环卷积滤波矩阵为 $C_g$，则

$$
C_g u_k=G[k]u_k.
$$

于是

$$
\langle C_g u_k,C_g u_l\rangle
=G[k]G^*[l]\langle u_k,u_l\rangle
=|G[k]|^2\delta_{k,l}.
$$

因此滤波只会表现为每个子载波上的等效增益：

$$
Y_k=H_kG_kX_k+W_k.
$$

接收端可以将 $G_k$ 并入等效信道：

$$
\widehat X_k=\frac{Y_k}{H_kG_k}.
$$

### 4.3 F-OFDM 的实际问题：滤波器拖尾

若滤波器长度为 $L_g$，输入信号长度为 $N$，线性卷积输出长度为

$$
N+L_g-1.
$$

多出来的 $L_g-1$ 个采样就是 **滤波器拖尾**。拖尾可能进入后续 OFDM 符号，造成 ISI；如果接收 FFT 窗口不能把滤波等效为循环卷积，也可能造成子载波间干扰（Inter-Carrier Interference, ICI）。

F-OFDM 的核心折中是：

$$
L_g\uparrow \Rightarrow \text{频域更干净，但时域拖尾更长};
$$

$$
L_g\downarrow \Rightarrow \text{拖尾更短，但 OOB 抑制较弱}.
$$

---

## 5. UF-OFDM：Universal Filtered OFDM / UFMC

### 5.1 子带组的概念

“子带组”不是说这些子载波发送同一个符号，而是说 **多个相邻子载波作为一组，共用一个滤波器**。

例如

$$
\mathcal K_b=\{24,25,\ldots,35\}.
$$

这 12 个子载波上的 QAM 符号可以分别为

$$
X_{24},X_{25},\ldots,X_{35},
$$

它们一般是不同的数据符号。子带组只是滤波粒度，不是调制符号粒度。

### 5.2 发射信号为什么是子带相加？

第 $b$ 个子带组的频域向量为

$$
\tilde X_b[k]=
\begin{cases}
X_k,&k\in\mathcal K_b,\\
0,&k\notin\mathcal K_b.
\end{cases}
$$

对它做 IFFT：

$$
s_b[n]=\frac{1}{\sqrt N}\sum_{k=0}^{N-1}\tilde X_b[k]e^{j2\pi kn/N}.
$$

对子带组滤波：

$$
x_b^{\mathrm{F}}[n]=s_b[n]*g_b[n].
$$

最终所有子带组通过同一个基带链路和射频前端同时发射，因此时域叠加：

$$
x[n]=\sum_{b=1}^{B}x_b^{\mathrm{F}}[n].
$$

这来自线性叠加原理：多个频带上的信号同时发射，最终在时域上就是相加。

### 5.3 UF-OFDM 为什么通常不使用传统 CP？

经典 UF-OFDM/UFMC 通常不使用传统 CP，而是依靠：

1. 子带滤波降低 OOB；
2. 滤波器长度控制时域扩展；
3. 接收端使用更长 FFT，例如 $2N$ 点 FFT，处理滤波器拖尾。

它可以粗略理解为更接近零填充（Zero Padding, ZP）或保护间隔思想，但不是简单地“用 ZP 替代 CP”。UF-OFDM 的关键在于子带组滤波，ZP/保护区只是为了容纳滤波器拖尾。

若滤波输出长度为

$$
N+L_g-1,
$$

接收端可补零到 $2N$ 点做 FFT：

$$
Y[m]=\sum_{n=0}^{2N-1}y[n]e^{-j2\pi mn/(2N)}.
$$

数据子载波通常对应于 $2N$ 点 FFT 中的偶数频点，形式上可写为

$$
\widehat X_k\propto Y[2k].
$$

---

## 6. FBMC：Filter Bank Multicarrier

### 6.1 “调制到不同子载波上”是什么意思？

这里的“调制”不是 QAM 星座调制，也不是射频上变频，而是 **子载波调制/频移调制**。

先设计一个低通原型滤波器 $p[n]$，其频域响应为 $P(f)$。把它乘以复指数：

$$
g_k[n]=p[n]e^{j2\pi kn/N},
$$

频域上等价于

$$
G_k(f)=P(f-k\Delta f).
$$

这就是“把原型滤波器调制到第 $k$ 个子载波上”。

OFDM 其实也可以看成这样，只不过 OFDM 的原型滤波器是矩形窗：

$$
g_k^{\mathrm{OFDM}}[n]=p_{\mathrm{rect}}[n]e^{j2\pi kn/N}.
$$

FBMC 的区别在于：它用一个更长、更平滑、频域局部化更好的 $p[n]$ 替代矩形窗。

### 6.2 FBMC 发射信号

FBMC 可写为

$$
x[n]=\sum_{k=0}^{N-1}\sum_m d_{k,m}p[n-mM]e^{j2\pi kn/N}.
$$

其中 $p[n]$ 通常是长滤波器，长度为

$$
L_p=KN,
$$

$K$ 为重叠因子。例如 $K=4$ 表示滤波器长度约为 4 个 OFDM 符号长度。

### 6.3 为什么叫滤波器组？

因为每个子载波都是一个滤波器分支。

发射端是综合滤波器组（Synthesis Filter Bank, SFB）：

$$
x[n]=\sum_{k=0}^{N-1}x_k[n],
$$

其中

$$
x_k[n]=\sum_m d_{k,m}p[n-mM]e^{j2\pi kn/N}.
$$

接收端是分析滤波器组（Analysis Filter Bank, AFB）：

$$
z_{k,m}=\sum_n r[n]g_{k,m}^*[n].
$$

---

## 7. FBMC-OQAM：实数域正交

### 7.1 为什么需要 OQAM？

如果 FBMC 直接传输复 QAM 符号，并希望满足复数域严格正交：

$$
\langle g_{k,m},g_{k',m'}\rangle=\delta_{k,k'}\delta_{m,m'},
$$

同时又希望滤波器有良好的时频局部化，会非常困难。

FBMC-OQAM 的思路是：不要求复数域正交，只要求实数域正交：

$$
\operatorname{Re}\{\langle g_{k,m},g_{k',m'}\rangle\}
=\delta_{k,k'}\delta_{m,m'}.
$$

### 7.2 FBMC-OQAM 信号形式

FBMC-OQAM 的发送信号常写成

$$
x[n]=\sum_{k=0}^{N-1}\sum_m a_{k,m}
p[n-mN/2]
e^{j2\pi kn/N}
e^{j\phi_{k,m}},
$$

其中：

- $a_{k,m}\in\mathbb R$；
- 相邻 OQAM 符号间隔为 $N/2$；
- 常见相位旋转为

$$
\phi_{k,m}=\frac{\pi}{2}(k+m),\quad e^{j\phi_{k,m}}=j^{k+m}.
$$

该相位旋转的作用是让邻近时频点干扰主要落到虚部。

接收端得到

$$
z_{k,m}=a_{k,m}+jI_{k,m}+w_{k,m},
$$

其中 $jI_{k,m}$ 是内禀干扰。由于有用符号是实数，接收端取实部：

$$
\widehat a_{k,m}=\operatorname{Re}\{z_{k,m}\}.
$$

### 7.3 FBMC-OQAM 会不会因为只传实数而速率减半？

不会天然减半。

普通 QAM/OFDM 可以理解为每 $T$ 秒发送一个复数符号：

$$
X_{k,l}=I_{k,l}+jQ_{k,l},
$$

也就是每 $T$ 秒发送两个实数维度。

FBMC-OQAM 是每 $T/2$ 秒发送一个实数符号：

$$
\frac{1\ \text{real symbol}}{T/2}=\frac{2\ \text{real symbols}}{T}.
$$

所以它的实数维度速率与每 $T$ 秒发送一个复 QAM 符号相同。

可以理解为：

$$
a_{k,2l}=I_{k,l},\quad a_{k,2l+1}=Q_{k,l},
$$

即复 QAM 的实部和虚部被拆开，错半个符号周期发送。

### 7.4 FBMC-OQAM 的真实代价

它的代价不是天然速率减半，而是：

1. 导频和信道估计复杂；
2. MIMO 预编码和均衡复杂；
3. 对相位噪声和时变信道更敏感；
4. 短包传输时长滤波器的首尾开销较明显。

---

## 8. F-OFDM 与 FBMC 的本质差别

F-OFDM：

$$
X_{\mathrm{F}}(f)=G(f)\sum_k X_kQ(f-k\Delta f).
$$

即：先生成 OFDM 信号，再整体/子带滤波。

FBMC：

$$
X_{\mathrm{FBMC}}(f)=\sum_k d_kP(f-k\Delta f).
$$

即：每个子载波本身就是一个频移后的滤波器分支。

| 对比项 | F-OFDM | FBMC |
|---|---|---|
| 滤波对象 | 整个 OFDM 子带信号 | 每个子载波分支 |
| 子载波波形 | 仍以 OFDM 复指数为基础 | 频移后的原型滤波器 |
| 正交性 | 理想循环卷积下仍保持复正交 | 通常只保持实数域正交 |
| 数据符号 | 复 QAM | OQAM 实数符号，错半拍 |
| 主要问题 | 滤波拖尾、ISI/ICI 近似误差 | 内禀干扰、导频/MIMO复杂 |
| 优势 | 兼容 OFDM 框架 | 频谱局部化最好 |

---

## 9. OOB 与频域局部化

### 9.1 OOB 是什么？

OOB 指信号泄漏到被分配频带之外的能量，而不是子载波间频谱重叠。

若分配频带为

$$
\mathcal B_{\mathrm{alloc}}=[f_1,f_2],
$$

则带外泄漏功率可写为

$$
P_{\mathrm{OOB}}=\int_{f\notin\mathcal B_{\mathrm{alloc}}}S_x(f)df.
$$

归一化 OOB 可写为

$$
\eta_{\mathrm{OOB}}
=\frac{\int_{f\notin\mathcal B_{\mathrm{alloc}}}S_x(f)df}
{\int_{-\infty}^{+\infty}S_x(f)df}.
$$

OFDM 子载波之间本来就是 sinc 频谱重叠的，只要正交条件成立，频谱重叠不等于 ICI。

### 9.2 频域局部化是什么意思？

频域局部化指一个波形的能量是否集中在它应该占用的频率附近。

对第 $k$ 个子载波，若频谱为 $G_k(f)$，则希望 $|G_k(f)|^2$ 主要集中在 $f\approx k\Delta f$ 附近，远离中心频率时快速衰减。

可用泄漏指标描述：

$$
\eta_{k,\mathrm{leak}}
=\frac{\int_{f\notin\mathcal B_k}|G_k(f)|^2df}
{\int_{-\infty}^{+\infty}|G_k(f)|^2df}.
$$

OOB 是系统级/子带级指标；频域局部化更偏向单个子载波或子带的频谱集中程度。

### 9.3 OOB 与频域局部化排序

带外泄漏严重程度大致为：

$$
\text{CP-OFDM}>
\text{W-OFDM}>
\text{F-OFDM}\approx\text{UF-OFDM}>
\text{FBMC-OQAM}.
$$

其中 $>$ 表示 OOB 更严重。

频域局部化能力大致为：

$$
\text{FBMC-OQAM}>
\text{F-OFDM/UF-OFDM}>
\text{W-OFDM}>
\text{CP-OFDM}.
$$

---

## 10. 功率效率比较

“功率效率”需要区分两个含义。

### 10.1 功放效率 / PAPR

多载波信号峰均功率比较高：

$$
\mathrm{PAPR}=\frac{\max_n|x[n]|^2}{\mathbb E[|x[n]|^2]}.
$$

PAPR 越高，功放越需要回退，功放效率越低。

几类多载波波形在 PAPR 上总体差异不如与 DFT-s-OFDM 的差异明显：

$$
\text{CP-OFDM}\approx
\text{W-OFDM}\approx
\text{F-OFDM}\approx
\text{UF-OFDM}\approx
\text{FBMC}.
$$

它们的主要目标不是降低 PAPR，而是改善频谱泄漏和频域局部化。

### 10.2 资源效率 / CP 开销

CP-OFDM 的时间利用率为

$$
\eta_{\mathrm{CP}}=\frac{N}{N+N_{\mathrm{CP}}}.
$$

FBMC 通常不需要 CP，因此长连续传输中资源效率更高：

$$
\eta_{\mathrm{FBMC}}\approx 1.
$$

但 FBMC 的滤波器很长，短包传输有启动和结束滤波器过渡开销，因此短包场景下未必占优。

---

## 11. 时变信道鲁棒性

时变信道可写成

$$
h[n,l].
$$

如果信道在一个符号期间变化明显，频域上会出现子载波耦合：

$$
Y_k=H_{k,k}X_k+\sum_{l\ne k}H_{k,l}X_l+W_k.
$$

其中第二项是 ICI。

归一化多普勒可写为

$$
\nu=f_DT_{\mathrm{sym}}.
$$

$\nu$ 越大，一个符号内信道变化越明显，正交性越容易被破坏。

几类波形的趋势：

- **CP-OFDM**：符号结构规整，导频/均衡成熟，对时变信道处理最方便；
- **W-OFDM**：接近 CP-OFDM，但窗长过长会增加时间扩展；
- **F-OFDM**：滤波器越长，等效时间扩展越大，高速场景下更敏感；
- **UF-OFDM**：无传统 CP 时，多径和时变信道下均衡更复杂；
- **FBMC-OQAM**：长滤波器导致时间局部化较差，时变信道会破坏实数域正交，较敏感。

时变信道鲁棒性大致为：

$$
\text{CP-OFDM}\gtrsim
\text{W-OFDM}\gtrsim
\text{F-OFDM}\gtrsim
\text{UF-OFDM}>
\text{FBMC-OQAM}.
$$

---

## 12. 基带复杂度

### 12.1 CP-OFDM

发射端：IFFT + CP 插入。  
接收端：去 CP + FFT + 一拍均衡。  
复杂度约为

$$
\mathcal O(N\log N).
$$

### 12.2 W-OFDM

在 CP-OFDM 基础上增加加窗和符号边缘重叠处理，复杂度略高，但仍较低。

### 12.3 F-OFDM

增加 FIR 滤波：

$$
x^{\mathrm{F}}[n]=x[n]*g[n].
$$

若直接卷积，复杂度约为

$$
\mathcal O(NL_g).
$$

可用快速卷积降低复杂度，但仍高于 CP/W-OFDM。

### 12.4 UF-OFDM

需要多个子带组分别处理：

$$
x^{\mathrm{UF}}[n]=\sum_{b=1}^{B}s_b[n]*g_b[n].
$$

复杂度来自多个子带分支、子带滤波、接收端长 FFT 和更复杂均衡。

### 12.5 FBMC

FBMC 需要多相网络（Polyphase Network, PPN）、长原型滤波器、OQAM 预处理/后处理、复杂导频和 MIMO 处理，复杂度最高。

总体排序：

$$
\text{CP-OFDM}<
\text{W-OFDM}<
\text{F-OFDM}\lesssim
\text{UF-OFDM}<
\text{FBMC-OQAM}.
$$

---

## 13. 相位噪声鲁棒性

相位噪声模型为

$$
r[n]=x[n]e^{j\theta[n]}+w[n].
$$

若 $\theta[n]$ 在一个符号内近似常数：

$$
\theta[n]\approx \theta_0,
$$

主要表现为公共相位误差（Common Phase Error, CPE）：

$$
Y_k=e^{j\theta_0}X_k+W_k.
$$

若 $\theta[n]$ 在符号内快速变化，会破坏子载波正交性，产生 ICI：

$$
Y_k=C_0X_k+\sum_{l\ne k}C_{k-l}X_l+W_k.
$$

### 13.1 OFDM 类波形

CP-OFDM、W-OFDM、F-OFDM、UF-OFDM 都保留了较多 OFDM 结构，相位噪声机理相似。CPE 可用导频估计，ICI 可通过更大子载波间隔、PT-RS 或补偿算法减轻。

### 13.2 FBMC-OQAM 为什么更敏感？

FBMC-OQAM 理想接收为

$$
z_{k,m}=a_{k,m}+jI_{k,m}.
$$

取实部恢复：

$$
\widehat a_{k,m}=\operatorname{Re}\{z_{k,m}\}.
$$

若存在相位误差 $\theta$：

$$
z'_{k,m}=e^{j\theta}z_{k,m}.
$$

展开其实部：

$$
\operatorname{Re}\{z'_{k,m}\}
=a_{k,m}\cos\theta-I_{k,m}\sin\theta.
$$

当 $\theta$ 较小时：

$$
\operatorname{Re}\{z'_{k,m}\}\approx a_{k,m}-\theta I_{k,m}.
$$

这说明原本位于虚部的内禀干扰会被相位旋转到实部，直接污染有用符号。因此 FBMC-OQAM 对相位噪声更敏感。

鲁棒性大致为：

$$
\text{CP-OFDM}\approx
\text{W-OFDM}\approx
\text{F-OFDM}\approx
\text{UF-OFDM}>
\text{FBMC-OQAM}.
$$

---

## 14. 综合性能对比表

| 指标 | CP-OFDM | W-OFDM | F-OFDM | UF-OFDM | FBMC-OQAM |
|---|---|---|---|---|---|
| OOB | 高 | 中等 | 较低 | 较低 | 最低 |
| 频域局部化 | 差 | 略好 | 好，子带级 | 好，子带组级 | 最好，子载波级 |
| PA 功率效率/PAPR | 高 PAPR，较差 | 类似 CP-OFDM | 类似或略差 | 类似或略差 | 高 PAPR，通常不优 |
| 资源效率 | 有 CP 开销 | CP/窗开销 | CP + 滤波拖尾 | 通常无传统 CP，但有滤波拖尾 | 长包高，短包受滤波器首尾影响 |
| 时变信道 | 处理成熟 | 接近 CP | 滤波器长时略敏感 | 无 CP 时均衡更复杂 | 长滤波器，较敏感 |
| 基带复杂度 | 最低 | 低 | 中等 | 中高 | 高 |
| 相位噪声鲁棒性 | 中等，补偿成熟 | 接近 CP | 接近 CP | 接近 CP | 较差 |
| MIMO/导频友好性 | 最好 | 很好 | 较好 | 中等 | 较差 |
| 典型优势 | 简单成熟 | 小改动降 OOB | 子带隔离能力强 | 异步子带场景 | 极低 OOB，频谱局部化最好 |
| 主要代价 | OOB 高、CP 开销 | 有效 CP 变短 | 滤波拖尾 | 长 FFT/均衡复杂 | 导频、MIMO、相位噪声复杂 |

---

## 15. 核心结论

1. **OOB 是分配频带之外的泄漏，不是子载波间频谱重叠。**  
   子载波 sinc 频谱重叠是 OFDM 的正常现象，只要正交性成立，不等于 ICI。

2. **频域局部化越好，OOB 通常越低。**  
   FBMC 每个子载波都由精心设计的原型滤波器形成，因此频域局部化最好；CP-OFDM 采用矩形窗，因此频域局部化最差。

3. **F-OFDM 不等于 FBMC。**  
   F-OFDM 是先生成 OFDM 子带信号再整体滤波；FBMC 是每个子载波本身就是一个滤波器分支。

4. **F-OFDM 通常不会产生 FBMC-OQAM 那种结构性非复正交问题。**  
   只要滤波拖尾能被 CP/保护间隔吸收，它仍然可以近似保持 OFDM 的复正交结构。

5. **UF-OFDM 的子带组不是共享同一个符号，而是共享同一个滤波器。**  
   子带组内部每个子载波仍然承载不同的 QAM 符号。

6. **FBMC-OQAM 只在实数域正交，但不天然损失一半速率。**  
   因为它每半个符号周期发送一个实数符号，等效上仍然是每个完整符号周期发送两个实数维度。

7. **滤波越精细，频谱越干净，但时域越扩展。**  
   这会带来更高复杂度、更复杂导频/均衡、更差的短包效率，以及对时变信道和相位噪声更敏感。

最终可用一句话概括：

> CP-OFDM 简单成熟但 OOB 高；W-OFDM 小改动降低 OOB；F-OFDM 和 UF-OFDM 通过子带/子带组滤波改善频谱；FBMC 频谱最干净，但以导频、MIMO、时变信道和相位噪声处理复杂为代价。

