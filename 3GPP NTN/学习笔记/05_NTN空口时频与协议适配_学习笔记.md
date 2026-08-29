---
title: "NTN 空口时频与协议适配学习笔记"
date: "2026-08-26"
updated: "2026-08-29"
sources:
  - "3GPP TR 38.811 V15.4.0"
  - "3GPP TR 38.821 V16.2.0"
  - "3GPP TS 38.211 Release 16"
  - "3GPP TS 38.213 V16.3.0"
  - "3GPP TS 38.214 V16.7.0"
  - "3GPP TS 38.300 V16.17.0"
  - "3GPP TS 38.331 V16.18.0"
---

# NTN 空口时频与协议适配学习笔记

> 本笔记接收[轨道、时间与链路几何](./02_卫星轨道时间与链路几何_学习笔记.md)、[天线波束与覆盖组织](./03_NTN天线波束与覆盖组织_学习笔记.md)以及[传播损耗与信道模型](./04_NTN传播损耗与信道模型_学习笔记.md)输出的时延、多普勒、波束和局部信道状态，负责把它们映射为 NR 空口约束、反馈闭环和接入过程状态；节点架构、波束足迹和小区组织不在本篇重复展开。

| 主章节 | 核心对象 | 判断尺度 | 主要来源 |
|---|---|---|---|
| 1 系统级空口时频与波形约束 | TA、残余 CFO、SCS、DM-RS、PT-RS、CP、双工与波形 | 波形、符号、时隙和收发切换 | TR 38.811 Clauses 7.3.2、7.3.5-7.3.7 |
| 2 测量、反馈与控制闭环 | TA/频偏维护、\(K_{\mathrm{offset}}\)、HARQ/RLC、CSI/CQI、MCS与功控 | 测量到执行的闭环时间 | TR 38.811 Clause 7.3.3、TR 38.821 Clause 6.2 |
| 3 初始接入、同步与状态迁移 | PRACH/RAR/Msg3、两步随机接入、PSS/SSS/PBCH与服务状态迁移 | 消息序列、监测窗口和状态转换 | TR 38.811 Clauses 7.3.2.3、7.3.4，TR 38.821 Clauses 6.3、7.2.1.1 |

全文按以下知识链组织：

\[
\text{几何与信道输入}
\rightarrow
\text{系统级波形约束}
\rightarrow
\text{测量反馈闭环}
\rightarrow
\text{RACH/SSB过程}
\rightarrow
\text{协议与仿真状态}.
\]

其中有三组量必须首先分开。对第 \(x\) 个 UE，定时提前量（Timing Advance，TA）可写成：

\[
T_{\mathrm{TA},x}(t)
=
T_{\mathrm{common}}(t)
+
T_{\mathrm{diff},x}(t)
+
T_{\mathrm{corr},x}(t),
\]

分别表示相对参考点的公共传播提前量、UE 相对参考点的差分提前量，以及位置、星历、网关时延和测量误差造成的剩余修正。TA 调整实际波形发射时刻；逻辑时序偏移 \(K_{\mathrm{offset}}\) 调整 DL 与 UL slot 的映射；传播往返时间（Round-Trip Time，RTT）则是信号必须真实经过的物理时间：

\[
\boxed{
T_{\mathrm{TA}}
\neq
K_{\mathrm{offset}}
\neq
T_{\mathrm{RTT}}.
}
\]

频率侧也要区分原始、公共、差分和残余量：

\[
f_{D,x}
=
f_{D,\mathrm{common}}
+
f_{D,\mathrm{diff},x},
\qquad
\delta f_x
=
f_{D,x}-\hat f_{D,x}.
\]

原始多普勒决定捕获和补偿范围，补偿后的残余频偏 \(\delta f_x\) 决定剩余载波间干扰（Inter-Carrier Interference，ICI），而局部多径的多普勒扩展决定一个公共频率旋转是否足够。

## 1 系统级空口时频与波形约束

本章讨论不依附于某一个消息流程、持续作用于波形或帧结构的约束。输入是几何笔记给出的绝对时延和多普勒、信道笔记给出的局部时延扩展，以及系统场景给出的星上射频条件；输出是 TA 参考、残余 CFO、参考信号配置边界、CP 适用性、双工选择和波形功率效率。

### 1.1 物理输入与机制边界

TR 38.811 Clause 7 的任务是识别 NTN 物理特征可能越过哪些 NR 基线能力，并不等于相应增强已经成为规范。判断过程可以写成：

\[
\text{NTN物理特征}
\rightarrow
\text{关键时空尺度}
\rightarrow
\text{NR基线容限}
\rightarrow
\text{失效条件}
\rightarrow
\text{候选适配}.
\]

| NTN输入 | 应比较的量 | 系统级消费者 | 不能直接推出的结论 |
|---|---|---|---|
| 大绝对时延 | 公共 TA、传播 RTT、slot 偏移 | 上行时间基准、TDD、跨 DL-UL 时序 | 不能据此把 CP 扩展到毫秒量级 |
| 波束内差分时延 | 差分 TA、PRACH 到达范围 | 多 UE 上行对齐、PRACH | 不能用卫星高度替代波束内差分距离 |
| 原始多普勒 | 捕获范围和公共预补偿能力 | PSS/SSS、粗频偏估计 | 不能直接等同于解调后的 ICI |
| 多普勒变化率 | \(\dot f_D T_{\mathrm{mech}}\) | DM-RS 时间跟踪、预测器 | 时隙内可忽略不表示长闭环内可忽略 |
| 局部超额时延 | 相干带宽、CP 覆盖范围 | DM-RS 频域采样、CP | 绝对传播时延不进入局部时延扩展 |
| 星上射频损伤 | CPE、ICI、PAPR、OBO | PT-RS、波形和有效 EIRP | 现有设计在参考场景可用不表示所有载荷均无影响 |

> **原文定位：**TR 38.811 Clauses 7.1-7.3.1，尤其是 Tables 7.1-1、7.2-1 和 7.3.1-1。

### 1.2 上行时间基准与 TA 架构

TA 的目的不是使 UE 和 gNB 的本地时钟相同，而是使 UE 的上行 OFDM 符号在 gNB 期望的接收时间窗内到达。设上下行传播近似对称，单程传播时延为 \(\tau_x\)。UE 接收下行时间基准时已经滞后约 \(\tau_x\)，上行发送后又经历约 \(\tau_x\)，因此用于理解的简化关系为：

\[
T_{\mathrm{TA},x}\approx 2\tau_x.
\]

#### 1.2.1 公共、差分与剩余 TA

在一个波束内选定参考点，记参考点到卫星的距离为 \(D_0\)，UE 到卫星的距离为 \(D_x\)。对星上 gNB 或再生载荷，可写成：

\[
T_{\mathrm{common}}
=
\frac{2D_0}{c},
\qquad
T_{\mathrm{diff},x}
=
\frac{2(D_x-D_0)}{c}.
\]

于是：

\[
T_{\mathrm{full},x}
=
T_{\mathrm{common}}
+
T_{\mathrm{diff},x}
=
\frac{2D_x}{c}.
\]

对于透明转发载荷，地面 gNB 位于网关之后，公共部分还要包含馈电链路。若 \(D_{01}\) 表示参考点到卫星的服务链路距离，\(D_{02}\) 表示卫星到网关的馈电链路距离，则研究模型可写成：

\[
T_{\mathrm{common}}
=
\frac{2(D_{01}+D_{02})}{c},
\qquad
T_{\mathrm{diff},x}
=
\frac{2(D_x-D_{01})}{c}.
\]

这里仅保留载荷部署对 TA 参考的接口；完整的透明/再生载荷和节点映射由[系统架构与部署场景](./01_NTN系统架构与部署场景_学习笔记.md)负责。

参考点选择决定差分 TA 的符号和范围。若参考点是波束内最短路径点，则通常有 \(D_x\ge D_0\)，差分量非负；若参考点位于其他位置，某些 UE 可能满足 \(D_x<D_0\)，从而需要负方向的剩余修正。因而参考点同时影响 TA 编码范围、是否需要有符号修正以及接收机搜索窗。

> **原文定位：**TR 38.821 Clause 6.3.4、Figure 6.3.4-1。Common TA、UE-specific TA 和参考点属于报告研究的 NTN 时间对齐框架，不能不加区分地写成所有版本 NR 的统一规范配置。

#### 1.2.2 Full TA 与 Differential TA 的承担位置

公共时延可以由 UE 侧吸收，也可以由网络时间架构吸收，两种方式的代价不同：

| 方案 | UE承担的提前量 | 网络侧时间关系 | 主要代价 |
|---|---|---|---|
| UE 应用 Full TA | Common + Differential + Correction | gNB DL/UL 帧可保持近似对齐 | UE 本地 UL timeline 相对 DL timeline 大幅提前 |
| UE 仅应用 Differential TA | Differential + Correction | 网络吸收 Common TA | gNB 需要维护 DL/UL frame timing shift |

无论公共时延放在哪一侧，传播路径都没有消失。TA 可以把上行波形的到达时刻对齐，却不能消除 RAR、HARQ 或 TDD 切换所经历的物理传播时间。

> **原文定位：**TR 38.821 Clause 6.2.1、Figures 6.2.1-1 和 6.2.1-2。两图讨论的是不同 DL/UL 时间架构及其调度影响。

#### 1.2.3 TA、SCS 与跨时隙调整

TR 38.811 以最大约 \(35\,\mu\mathrm{s/s}\) 的时延漂移评估 TA 更新。不同子载波间隔（Subcarrier Spacing，SCS）下，TA 量化步长和单次允许更新量不同：

| SCS | 时隙长度 | TA量化步长 | 报告估计的单次最大调整 | 跟踪最大漂移的命令量级 |
|---:|---:|---:|---:|---:|
| 15 kHz | 1 ms | 520.83 ns | 约16.6 μs | 约10次/s |
| 30 kHz | 0.5 ms | 260.42 ns | 约8.3 μs | 数十次/s量级 |
| 60 kHz | 0.25 ms | 130.21 ns | 约4.15 μs | 约40次/s |
| 120 kHz | 0.125 ms | 65.10 ns | 约2.1 μs | 约80次/s |
| 240 kHz | 0.0625 ms | 32.55 ns | 约1 μs | 更高更新频率 |

SCS 增大后 TA 时间粒度更细，但单次调整量变小；同一物理传播时延还对应更多 slot。因此更大 SCS 不能解决大 TA，UE 的发送时间必须允许跨 slot/TTI 边界整体移动，逻辑调度还需由 \(K_{\mathrm{offset}}\) 保证因果性。

> **原文定位：**TR 38.811 Clause 7.3.2.2、Table 7.3.2.2.2-1。命令次数为报告在给定最大漂移下的量级分析，不是所有实现的固定发送周期。

### 1.3 残余多普勒、SCS 与 OFDM 正交性

对单个 UE，若多普勒在一个 OFDM 符号内近似恒定，它首先表现为所有子载波的公共频移。第 \(k\) 个子载波在残余频偏 \(\delta f\) 下可写成：

\[
y_k(t)
=
X_k e^{j2\pi(k\Delta f+\delta f)t}.
\]

接收机在第 \(m\) 个子载波做 FFT，对应积分：

\[
\frac{1}{T_u}
\int_0^{T_u}
e^{j2\pi[(k-m)\Delta f+\delta f]t}
\,\mathrm{d}t,
\qquad
T_u=\frac{1}{\Delta f}.
\]

当 \(\delta f=0\) 时，\(k\ne m\) 的积分为零；当 \(\delta f\ne0\) 时，正交性被破坏并形成 ICI。真正决定剩余正交性损失的是：

\[
\varepsilon_{\mathrm{res}}
=
\frac{\delta f}{\Delta f}.
\]

例如 600 km LEO 在 2 GHz 下可能产生约 48 kHz 原始多普勒。若不补偿，对 15 kHz SCS 而言会跨越多个子载波；若星历、位置和频偏估计共同补偿 47.9 kHz，只剩 100 Hz，则：

\[
\varepsilon_{\mathrm{res}}
=
\frac{100}{15000}
\approx0.0067.
\]

此时剩余 ICI 已显著降低。若频偏恰好等于整数倍子载波间隔，理想条件下子载波间仍保持正交，但索引会整体错位，接收机仍须识别和校正整数频偏。

原始多普勒及其变化率需要分开比较。TR 给出的 600 km LEO 参考量级为：

| 载频 | 最大原始多普勒 | 最大变化率 | 1 ms内最大变化 |
|---:|---:|---:|---:|
| 2 GHz | 约±48 kHz | 544 Hz/s | 0.544 Hz |
| 20 GHz | 约±480 kHz | 5.44 kHz/s | 5.44 Hz |
| 30 GHz | - | 8.16 kHz/s | 8.16 Hz |

大原始多普勒决定 PSS/SSS 捕获和粗补偿范围；一个时隙内的变化量决定 DM-RS 时间跟踪；补偿信息经过较长闭环后的老化量则近似为：

\[
\delta f_{\mathrm{stale}}
\approx
\dot f_D T_{\mathrm{age}}.
\]

三者属于不同机制，不能用“多普勒很大”统一代替。

> **原文定位：**TR 38.811 Clauses 5.3.1.3、5.3.4.3-5.3.4.4、7.3.2.3-7.3.2.4，Table 7.3.2.4.1-1。公共、差分、残余和老化频偏的分层属于基于报告条件的工程解释。

### 1.4 局部信道、DM-RS 与循环前缀

卫星高度决定绝对传播时延，地面附近的反射与散射形成局部超额时延。解调参考信号（Demodulation Reference Signal，DM-RS）和循环前缀（Cyclic Prefix，CP）面对的是局部信道，而不是从地面到卫星的全部传播距离。

| 物理量关系 | 对应机制 | 判断目标 |
|---|---|---|
| \(\dot f_D T_{\mathrm{slot}}\) | DM-RS时间位置 | 一个时隙内能否跟踪符号间变化 |
| \(\Delta f_{\mathrm{pilot}}\) 与 \(B_c\) | DM-RS频域配置 | 导频之间能否可靠插值信道 |
| \(T_{\mathrm{CP}}\) 与 \(\tau_{\mathrm{excess,max}}+T_{\mathrm{timing,res}}\) | 循环前缀 | 是否避免 ISI 并维持循环卷积条件 |

#### 1.4.1 DM-RS 时间与频域采样

TR 将 1 ms 内的多普勒变化与频率误差量级比较，认为参考场景中的短时变化不足以要求修改 DM-RS 时间位置。该结论不包括尚未完成的粗多普勒补偿，也不包括百毫秒闭环中的信息老化。

频域采样需将导频间隔与相干带宽 \(B_c\) 比较。以一个 PRB 和一个 CDM 组为例：

| 配置 | 典型子载波位置 | 最大相邻采样间隔 | 特点 |
|---|---|---:|---|
| Type 1 | \(0,2,4,6,8,10\) | \(2\Delta f_{\mathrm{SCS}}\) | 梳状均匀采样，适合较强频率选择性 |
| Type 2 | \((0,1),(6,7)\) | \(5\Delta f_{\mathrm{SCS}}\) | 成对 RE，可组织更多 CDM 组与端口 |

Type 1/Type 2 是 DM-RS 频域配置类型，不是 SCS 类型。15 kHz SCS 下，两者的最大采样间隔分别为 30 kHz 和 75 kHz。报告用保守关系：

\[
B_c
=
\frac{1}{\alpha\tau_{\mathrm{DS}}},
\qquad
\alpha=50
\]

进行比较。例如 \(\tau_{\mathrm{DS}}=100\,\mathrm{ns}\) 时：

\[
B_c
=
\frac{1}{50\times100\times10^{-9}}
=200\,\mathrm{kHz}.
\]

| 场景 | 报告最大时延扩展 | 保守相干带宽 |
|---|---:|---:|
| D1 GEO Ka | 10 ns | 约2 MHz |
| D2 GEO S | 100 ns | 200 kHz |
| D3 LEO S | 100 ns | 200 kHz |
| D4 LEO Ka | 10 ns | 约2 MHz |
| D5 HAPS S | 150 ns | 133 kHz |

以 HAPS S 波段为例，30 kHz SCS 的 Type 1 最大间隔 60 kHz，较为安全；Type 2 最大间隔 150 kHz，略超出该保守判据。60 kHz SCS 下，Type 1 的 120 kHz 已接近边界，Type 2 的 300 kHz 则不满足该初步比较。因此报告的结论是可通过约束 SCS 和 DM-RS 配置适配，而不是所有配置天然等价。

> **原文定位：**TR 38.811 Clauses 7.3.2.4、7.3.5.1.1-7.3.5.1.3，Tables 7.3.5.1.1-1 至 7.3.5.1.1-3；精确 RE 和端口映射参见 TS 38.211 Clause 7.4.1.1.2。

#### 1.4.2 绝对时延、超额时延与 CP

信道可写成：

\[
h(t)
=
\sum_{\ell}
a_{\ell}
\delta\!\left(t-\tau_0-\Delta\tau_{\ell}\right),
\]

其中 \(\tau_0\) 是直达径的绝对传播时延，\(\Delta\tau_{\ell}\) 是第 \(\ell\) 条多径相对直达径的超额时延。同步和 TA 处理 \(\tau_0\)，CP 只需覆盖：

\[
T_{\mathrm{CP}}
\gtrsim
\max_{\ell}\Delta\tau_{\ell}
+
T_{\mathrm{timing,res}}.
\]

| SCS | 正常CP代表长度 |
|---:|---:|
| 15 kHz | 4.688 μs |
| 30 kHz | 2.344 μs |
| 60 kHz | 1.172 μs |
| 120 kHz | 0.586 μs |
| 240 kHz | 0.293 μs |

TR 给出的 2 GHz 卫星信道时延扩展约为 180-250 ns，正常 CP 在参考场景下具有充足覆盖能力。15 kHz SCS 的有效符号长度为：

\[
T_u
=
\frac{1}{15\,\mathrm{kHz}}
=66.67\,\mu\mathrm{s}.
\]

若实际只需约 0.25 μs，而正常 CP 为 4.688 μs，报告用：

\[
\frac{4.688-0.25}{66.67}
\approx6.7\%
\]

估计过度配置造成的额外开销。新增 NTN 专用短 CP 虽可降低少量开销，却会增加 numerology 和实现复杂度，因此报告不预期必须修改 CP 规范。

SCS 选择具有跨机制权衡：

\[
\boxed{
\begin{aligned}
\text{增大SCS}
&\Rightarrow
\text{归一化CFO减小、CP变短}\\
&\Rightarrow
\text{DM-RS绝对频率间隔增大}.
\end{aligned}
}
\]

> **原文定位：**TR 38.811 Clauses 7.3.5.2.1-7.3.5.2.3、Table 7.3.5.2.2-1。

### 1.5 相位噪声、PT-RS 与波形约束

星上本振和频率转换链会叠加相位噪声，有限电源与散热条件又推动功率放大器（Power Amplifier，PA）靠近饱和区工作。两类硬件问题分别作用于相位稳定性和功率效率。

#### 1.5.1 相位误差与参考信号分工

FFT 后第 \(l\) 个 OFDM 符号、第 \(k\) 个子载波可近似写成：

\[
Y_{k,l}
=
e^{j\phi_l}H_{k,l}X_{k,l}
+
I_{k,l}^{\mathrm{ICI}}
+
N_{k,l},
\]

其中 \(\phi_l\) 是符号级公共相位误差（Common Phase Error，CPE），\(I_{k,l}^{\mathrm{ICI}}\) 是符号内快速相位变化产生的 ICI。相位跟踪参考信号（Phase-Tracking Reference Signal，PT-RS）和 DM-RS 的职责为：

| 参考信号 | 主要任务 | 不能替代的处理 |
|---|---|---|
| DM-RS | 估计复信道的幅度、相位和空间层 | 大范围粗频偏捕获 |
| PT-RS | 跟踪符号级剩余 CPE | 严重残余 CFO 造成的宽带 ICI 校正 |

正确处理链为：

\[
\text{粗频偏估计与补偿}
\rightarrow
\text{FFT}
\rightarrow
\text{DM-RS信道估计}
\rightarrow
\text{PT-RS相位跟踪}.
\]

TR 对 DTH、VSAT 和典型弯管载荷的相位噪声模板进行评估。在载频不高于约 30 GHz、残余 CFO 不大、带宽不是特别宽且载荷相位噪声接近参考模板的条件下，现有 PT-RS 设计可能足以补偿。该结论不能推广到任意高载频、超大带宽或特殊星上本振。

> **原文定位：**TR 38.811 Clause 7.3.7.1、Annex B。

#### 1.5.2 PAPR、OBO 与波形选择

峰均功率比（Peak-to-Average Power Ratio，PAPR）定义为：

\[
\mathrm{PAPR}
=
\frac{\max_n|x[n]|^2}
{\mathbb E\{|x[n]|^2\}}.
\]

CP-OFDM 的多子载波叠加会产生较高瞬时峰值。PA 接近饱和时，非线性会造成星座压缩、带内失真、误差矢量幅度恶化、带外泄漏和多载波互调。为保持线性，需要输出功率回退（Output Back-Off，OBO）：

\[
\mathrm{OBO}
=
10\log_{10}
\frac{P_{\mathrm{sat}}}{P_{\mathrm{out}}}.
\]

TR 引用的典型总损失为：

| 波形 | QPSK | 16-QAM |
|---|---:|---:|
| CP-OFDM | 约6 dB | 约7.6 dB |
| DFT-s-OFDM | 约4 dB | 约6 dB |

差异约 1.6-2 dB。2 dB 输出功率差对应：

\[
10^{-2/10}\approx0.63,
\]

即相同功放下有效输出功率可能只剩约 63%。报告给出的容量影响为 20%-40% 范围，具体取决于系统是功率受限还是带宽受限，不能把功率下降直接等同于固定容量下降。

DFT-s-OFDM 的主要价值是较低 PAPR、较小回退和更高有效 EIRP，不表示其在相同 RE、调制阶数和编码率下天然具有较低频谱效率。TR 没有规定新的 NTN 下行波形，而是建议继续研究 CP-OFDM 的 PAPR 降低，并指出上行 DFT-s-OFDM 可能有利。

> **原文定位：**TR 38.811 Clause 7.3.7.2、Figure 7.3.7.2.1-1。总损失示例来自报告引用的 ETSI TR 103 297。

### 1.6 TDD 保护与双工选择

时分双工（Time Division Duplex，TDD）在同一频段切换上下行。gNB 发完最后一个下行符号后，该符号需要 \(\tau_{\mathrm{DL}}\) 才被 UE 收完；UE 完成收发切换并发出第一个上行符号后，还需 \(\tau_{\mathrm{UL}}\) 才到达 gNB。因此：

\[
T_{\mathrm{guard}}
\ge
\tau_{\mathrm{DL}}
+
\tau_{\mathrm{UL}}
+
T_{\mathrm{sw}}
+
M_{\mathrm{timing}}.
\]

忽略切换和同步余量、假设上下行近似对称时：

\[
T_{\mathrm{guard}}
\approx2\tau
=T_{\mathrm{RTT,prop}}.
\]

TA 不能消除这一保护时间，因为半双工 UE 不能在最后一个 DL 符号尚未接收完时发送 UL。多 UE 情况还需要按最坏传播路径配置。TR 使用最大单程传播时延给出保守量级：600 km LEO 约 \(2\times7\,\mathrm{ms}=14\,\mathrm{ms}\)，GEO 约 \(2\times270\,\mathrm{ms}=540\,\mathrm{ms}\)。

频分双工（Frequency Division Duplex，FDD）上下行使用不同频段，不需要为 DL/UL 切换预留同类传播保护时间，因此报告认为 FDD 是多数 NR NTN 接入的优选方式。FDD 仍需要成对频谱、双工器和射频隔离，并需分别处理不同上下行载频的多普勒。

> **原文定位：**TR 38.811 Clause 7.3.6.1。这里的 RTT 是纯传播往返时间，不包含检测、调度和编解码时间。

## 2 测量、反馈与控制闭环

本章拥有“测量何时生成、信息何时到达、决策何时执行”的闭环关系。长 RTT 不会让所有机制以同一种方式退化：HARQ 面对反馈等待，MCS 面对 CSI 老化，功控面对路径损耗估计和命令滞后，TA/频偏维护则可以利用可预测几何进行开环外推。

### 2.1 闭环时间与信息老化

多数自适应机制可写成：

\[
\text{测量}
\rightarrow
\text{滤波/预测}
\rightarrow
\text{反馈}
\rightarrow
\text{决策}
\rightarrow
\text{执行}.
\]

反馈有效性的基本条件是：

\[
T_{\mathrm{loop}}
\ll
T_{\mathrm{variation}},
\]

其中 \(T_{\mathrm{loop}}\) 包含传播、测量、处理、调度和命令执行，\(T_{\mathrm{variation}}\) 表示被跟踪量的变化时间。不同对象的可预测性不同：

| 对象 | 主要变化来源 | 适合的处理 |
|---|---|---|
| 斜距、公共 TA、公共多普勒 | 轨道与相对几何 | 星历和位置预测为主，闭环修残差 |
| 大尺度路径损耗 | 斜距、仰角、雨衰和阴影 | 预测、滤波、滞回和保守裕量 |
| 局部快衰落 | 多径和终端局部运动 | 参考信号跟踪；长 RTT 下难以逐衰落闭环 |
| HARQ成功/失败 | 一次传输块译码结果 | 必须等待真实 ACK/NACK，不能用几何预测代替 |

因此 NTN 的关键不是简单“所有反馈都放慢”，而是将可预测公共量、随机动态量和必须等待的协议事件分开。

> **原文定位：**TR 38.811 Clauses 7.3.2.2-7.3.3.3。预测/滤波/闭环三层拆分属于基于报告条件的工程组织。

### 2.2 TA 与频率补偿的持续维护

对非地球静止轨道（Non-Geostationary Orbit，NGSO）卫星：

\[
D=D(t),
\qquad
T_{\mathrm{TA}}(t)
=
\frac{2D(t)}{c},
\qquad
\dot T_{\mathrm{TA}}
=
\frac{2}{c}\dot D(t).
\]

若闭环在 \(t_0\) 测得误差，但 UE 到 \(t_0+T_{\mathrm{age}}\) 才执行命令，老化误差近似为：

\[
\Delta T_{\mathrm{stale}}
\approx
\dot T_{\mathrm{TA}}T_{\mathrm{age}}.
\]

频率补偿同理：

\[
\delta f_{\mathrm{stale}}
\approx
\dot f_D T_{\mathrm{age}}.
\]

因此适合 NTN 的结构为：

\[
\boxed{
\text{位置+星历的开环预测}
+
\text{网络测量的闭环细修}.
}
\]

几何预测器使用 UE 和卫星位置、速度与视线方向：

\[
\hat f_D
=
\frac{f_c}{c}
(\mathbf v_{\mathrm{sat}}-\mathbf v_{\mathrm{UE}})^T
\mathbf u_{\mathrm{LOS}}.
\]

同一组 \(\mathbf r_{\mathrm{sat}}(t)\)、\(\mathbf v_{\mathrm{sat}}(t)\) 同时产生斜距、传播时延、TA、斜距变化率和多普勒。网络测得的 UL 到达误差和残余 CFO 再修正位置误差、星历误差、网关群时延、晶振偏差和信息老化。

若位置误差对应的距离误差为 \(\Delta D\)，TA 误差量级为：

\[
\Delta T_{\mathrm{TA}}
=
\frac{2\Delta D}{c}.
\]

例如 \(\Delta D=100\,\mathrm{m}\) 时，\(\Delta T_{\mathrm{TA}}\approx0.67\,\mu\mathrm{s}\)。这一数值说明 GNSS 和星历可以提供粗对齐，但并不取消网络细化的必要性。

> **原文定位：**TR 38.811 Clause 7.3.2.2；TR 38.821 Clause 6.3.4。UE autonomous TA、network-indicated common TA 和 timing drift rate 均属于报告研究的候选框架，应与后续 TS 的规范机制分开表述。

### 2.3 跨 DL-UL 逻辑时序

大 TA 会使 UE 本地 UL timeline 相对 DL timeline 大幅提前。若 gNB 在 DL slot \(n\) 发送 DCI，要求 UE 在 \(n+K_2\) 的 UL slot 发 PUSCH，UE 收到命令时对应 UL slot 可能已经过去。物理到达对齐正确，但协议调度关系失去因果性。

TR 38.821 因此研究额外逻辑偏移 \(K_{\mathrm{offset}}\)：

\[
n_{\mathrm{PUSCH}}
=
n_{\mathrm{DCI}}
+K_2
+K_{\mathrm{offset}},
\]

\[
n_{\mathrm{ACK}}
=
n_{\mathrm{PDSCH}}
+K_1
+K_{\mathrm{offset}}.
\]

| 过程 | 基准关系或起点 | NTN修正方向 |
|---|---|---|
| DCI→PUSCH | DL 命令触发未来 UL | UL slot 加偏移 |
| RAR→Msg3 | Msg2 中 UL Grant | Msg3 PUSCH 加偏移 |
| PDSCH→HARQ-ACK | DL 数据触发 UL 反馈 | ACK/NACK slot 加偏移 |
| MAC CE生效 | ACK 后等待处理时间 | 延长并对齐生效时刻 |
| CSI on PUSCH | DL DCI 触发未来 UL 报告 | PUSCH slot 加偏移 |
| CSI reference resource | 从 UL 报告反推过去 DL 资源 | DL 参考 slot 减偏移 |
| 非周期SRS | DL DCI 触发未来 UL SRS | SRS slot 加偏移 |

纯下行 PDCCH→PDSCH 只在 DL timeline 上定义，不应无条件增加同类偏移。\(K_{\mathrm{offset}}\) 只保证调度动作发生在 UE 可执行的未来，不能缩短 ACK 的传播、网络处理和重传形成的 HARQ 闭环。

> **原文定位：**TR 38.821 Clause 6.2.1、Figures 6.2.1-1 至 6.2.1-2；相关 NR 基线参见 TS 38.214 Clauses 5.2、6.1.2.1.1、6.2.1 和 TS 38.213 Clause 9。TR 中的偏移讨论属于研究方案，不能直接写成所有 Release 16 配置的既定参数。

### 2.4 HARQ、RLC 与缓存定时

混合自动重传请求（Hybrid Automatic Repeat reQuest，HARQ）在 PHY/MAC 层使用 ACK/NACK、冗余版本和软合并快速修复误块。一个 HARQ 进程发送传输块后，在反馈返回前不能无条件复用同一进程状态。为保持流水，最小并行进程数可近似理解为：

\[
N_{\mathrm{HARQ,min}}
\approx
\left\lceil
\frac{T_{\mathrm{HARQ}}}{T_{\mathrm{slot}}}
\right\rceil.
\]

TR 在 15 kHz SCS、1 ms 时隙的参考条件下给出：

| 场景 | 参考HARQ周期 | 参考最小进程数 | 报告判断 |
|---|---:|---:|---|
| 地面NR | 16 ms | 16 | Rel-15参考能力可行 |
| LEO | 50 ms | 50 | 扩展HARQ后可行 |
| MEO | 180 ms | 180 | 需研究TBS/MCS和实现能力 |
| GEO/HEO | 600 ms | 600 | 简单线性扩展压力很大 |

SCS 增大只会让同一 RTT 内包含更多时隙，不会缩短物理反馈时间。进程数扩展还会增加软缓存、并行编解码、冗余版本和调度上下文。仅用于理解的原始在途数据下界为：

\[
B_{\mathrm{flight}}
\gtrsim
R T_{\mathrm{ACK}}.
\]

若 \(R=100\,\mathrm{Mbit/s}\)、\(T_{\mathrm{ACK}}=0.6\,\mathrm{s}\)，原始在途数据约为 \(60\,\mathrm{Mbit}=7.5\,\mathrm{MB}\)。实际 HARQ 软缓存还与编码块、量化精度、多用户并发和冗余版本有关。

| 机制 | 层次 | 接收端处理 | 长RTT下主要压力 |
|---|---|---|---|
| HARQ | PHY/MAC | 软合并和传输块反馈 | 进程数、软缓存、ACK/NACK关联 |
| RLC ARQ | RLC确认模式 | 状态报告和PDU重传 | 窗口、定时器、状态报告和缓存 |

限制或关闭部分 HARQ 反馈并不表示误块消失，而是将更多剩余错误交给 RLC ARQ、更强编码或应用容错。RLC 的恢复需要跨越更长的状态报告和重传闭环，因此不能认为它能够无代价替代 HARQ。

> **原文定位：**TR 38.811 Clauses 7.3.3.1-7.3.3.2，Figures 7.3.3.1-1、7.3.3.1.1-1，Table 7.3.3.1.1-1。表中进程数是研究评估参考，不是所有 NTN 系统的强制配置。

### 2.5 CSI/CQI、预测与 MCS

自适应编码调制（Adaptive Coding and Modulation，ACM）使用信道状态信息（Channel State Information，CSI）和信道质量指示（Channel Quality Indicator，CQI）支持调度与调制编码方案（Modulation and Coding Scheme，MCS）选择。长 RTT 下真正使用的不是测量时刻信道，而是数据到达接收端时的信道。

| 场景 | 主要变化 | 闭环适用性 |
|---|---|---|
| GEO Ka | 雨衰通常较慢 | ACM可工作，但需滞回避免MCS振荡 |
| GEO S | 遮挡和局部多径可能较快 | 难以逐衰落跟踪，需要更大裕量 |
| LEO | 斜距引起的大尺度损耗可预测 | 可预测和跟踪大尺度变化，仍难跟踪快速多径 |
| 手持终端 | 姿态、阴影和局部遮挡 | 需要预测、滤波和保守MCS |

工程实现可使用：

\[
\gamma_{\mathrm{design}}
=
\hat\gamma_{\mathrm{predicted}}
-M_{\mathrm{total}},
\]

\[
I_{\mathrm{MCS}}^{\star}
=
\max\left\{
i:
\gamma_{\mathrm{req}}(i,\mathrm{BLER}_{\mathrm{target}})
\le
\gamma_{\mathrm{design}}
\right\}.
\]

总裕量可以作为工程模型拆成：

\[
M_{\mathrm{total}}
=
M_{\mathrm{estimation}}
+
M_{\mathrm{feedback\ aging}}
+
M_{\mathrm{loop\ uncertainty}}.
\]

该分解不是 TR 或 TS 直接定义的协议参数。3GPP 规定 MCS 索引与调制阶数、目标码率及相应信令；下行 PDSCH 和通常的上行 PUSCH MCS 由 gNB 调度器选择，具体预测器、CQI 到 MCS 映射、滞回、外环链路自适应和 NTN 裕量属于实现算法。

> **原文定位：**TR 38.811 Clause 7.3.3.3；MCS 和链路自适应基线参见 TS 38.214 Clauses 5.1.3、5.2.2 和 6.1.4。TR 要求进一步研究闭环裕量，没有规定上述工程分解的固定数值。

### 2.6 RSRP 滤波与上行功率控制

UE 可根据同步信号块（Synchronization Signal Block，SSB）或信道状态信息参考信号（Channel State Information Reference Signal，CSI-RS）的参考信号接收功率（Reference Signal Received Power，RSRP）估计路径损耗：

\[
\widehat{PL}
=
P_{\mathrm{RS,tx}}
-
\mathrm{RSRP}_{\mathrm{filtered}}.
\]

PUSCH 功率控制可用简化结构表示：

\[
P_{\mathrm{PUSCH}}
=
\min\left\{
P_{\mathrm{CMAX}},
P_0
+10\log_{10}(2^{\mu}M_{\mathrm{RB}})
+\alpha\widehat{PL}
+\Delta_{\mathrm{TF}}
+f_{\mathrm{CL}}
\right\}.
\]

其中 \(P_0\) 是目标功率基准，\(M_{\mathrm{RB}}\) 是资源块数量，\(\alpha\) 是路径损耗补偿因子，\(\Delta_{\mathrm{TF}}\) 是传输格式修正，\(f_{\mathrm{CL}}\) 是闭环项。

TS 38.331 定义的一阶 Layer 3 指数滤波为：

\[
F_i
=(1-a)F_{i-1}+aM_i,
\qquad
a=\frac{1}{2^{k/4}},
\]

其中 \(M_i\) 是最新测量，\(F_i\) 是滤波输出，\(k\) 是 `filterCoefficient`。例如 \(F_{i-1}=-90\,\mathrm{dBm}\)、\(M_i=-98\,\mathrm{dBm}\)：\(k=4\) 时 \(a=1/2\)，得到 \(-94\,\mathrm{dBm}\)；\(k=8\) 时 \(a=1/4\)，得到 \(-92\,\mathrm{dBm}\)。后者更平滑，却会在真实功率快速下降时暂时低估路径损耗。

因此完整链条为：

\[
\text{SSB/CSI-RS}
\rightarrow
M_i
\rightarrow
F_i
\rightarrow
\widehat{PL}
\rightarrow
P_{\mathrm{PUSCH}}.
\]

TR 38.821 讨论过多个 Layer 3 滤波系数、按波束配置功控参数、利用星历预测功率和关闭部分闭环功控等方案，但未收敛为统一增强机制；研究结论仍以 NR Release 15 功控方案作为 NTN 基线。

> **原文定位：**TR 38.821 Clause 6.2.2；TS 38.213 Clause 7.1.1；TS 38.331 Clause 5.5.3.2。

## 3 初始接入、同步与状态迁移

本章按“NTN 影响集中程度”先讨论 RACH，再讨论 SSB，不代表实际空口时序中 RACH 早于 SSB。RACH 小节先把下行同步和系统信息提供的时间、频率及辅助信息视为输入；SSB 小节随后解释这些输入的获得边界。

### 3.1 NTN 随机接入与初始 TA

随机接入把上游的传播时延、差分时延、残余载波频率偏移（Carrier Frequency Offset，CFO）和时间参考具体化为 Msg1 检测、Msg2 监测、Msg3 调度和上行对齐问题。其核心不是把一个 TA 字段无限扩大，而是重组初始时延预测、网络细化和跨 DL-UL 调度。

从实现角度，应把随机接入看成两条同时收敛的估计链：

\[
\begin{aligned}
\text{时间链：}&\quad
\text{Common TA}
+\text{Differential TA}
+\text{Residual correction},\\
\text{频率链：}&\quad
\text{公共多普勒补偿}
+\text{UE-specific 多普勒补偿}
+\text{Residual CFO estimation}.
\end{aligned}
\]

只有两条链都进入 PRACH 接收机的捕获范围，相关峰、RO 标签和后续 Msg3 时间线才同时可信。

#### 3.1.1 四步随机接入与初始 TA

传统四步随机接入为：

\[
\text{Msg1 PRACH}
\rightarrow
\text{Msg2 RAR}
\rightarrow
\text{Msg3 PUSCH}
\rightarrow
\text{Msg4}.
\]

地面网络中，UE 可以先发送 Msg1，gNB 根据到达时间估计 TA，再在随机接入响应（Random Access Response，RAR）中发送 TA 命令，使 Msg3 对齐。NTN 中 Msg1 在首次网络 TA 命令到来之前已经需要跨越几十至数百毫秒的传播路径，传统 RAR TA 范围也无法承担完整卫星绝对时延。

因此初始接入需要在 Msg1 之前获得粗 TA。两条研究路线为：

| UE条件 | Msg1前粗补偿 | Msg2中的主要作用 |
|---|---|---|
| 有位置与星历能力 | UE 根据位置、星历及必要的网关信息自主计算 Full TA 或服务链路 TA | 修正 UE 估计的 residual error |
| 无位置能力 | 网络广播相对参考点的 Common TA 或时间偏移 | 给出 UE-specific differential/residual TA |

四步过程中的 UE 与网络知识状态并不相同：

| 阶段 | UE 已知或应用的量 | 网络从接收中可知的量 | 尚未闭合的问题 |
|---|---|---|---|
| Msg1 前 | 下行时频基准、广播辅助量、自主估计的 TA/CFO | 波束/小区级公共参考 | UE 估计误差尚未知 |
| Msg1 到达 | UE 已按自身估计提前发送 | 前导、到达误差、残余 CFO、候选 RO | 到达误差不等于 UE 已应用的绝对 TA |
| Msg2/RAR | 接收 TA correction 与 UL grant | 可通知检测结果和细化量 | Msg3 调度仍可能缺少 UE 绝对 UL timeline |
| Msg3 到达 | 已应用粗 TA 与 RAR correction | 可联合 Msg3 信息建立更完整 TA 状态 | 转入后续 TA/CFO 维护 |

UE 自主计算服务链路几何时：

\[
\hat D_x(t)
=
\left\|
\hat{\mathbf r}_{\mathrm{sat}}(t)
-
\hat{\mathbf r}_{\mathrm{UE}}(t)
\right\|,
\qquad
\hat T_{\mathrm{TA},x}
\approx
\frac{2\hat D_x(t)}{c}.
\]

这里 GNSS 只给出 UE 的位置和时间基准，并不会直接“输出 TA”。UE 还需要卫星星历、参考时刻以及透明转发场景所需的网关位置或馈电链路时延。对同一波束参考点 \(r\) 和 UE \(u\)，可写为：

\[
\tau_u(t)
=
\tau_r(t)
+
\Delta\tau_{\mathrm{geo},u}(t),
\]

\[
e_{\tau,u}
=
\left[
\tau_u-\tau_r
-
\widehat{\Delta\tau}_{\mathrm{geo},u}
\right]
+b_t,
\]

其中 \(b_t\) 汇总 UE 时钟、星历时效、位置误差、公共参考误差和未建模馈电时延。频率侧有对应分解：

\[
\delta f_u
=
\left(f_{D,u}-f_{D,r}\right)
-
\widehat{\Delta f}_{D,u}
+b_f.
\]

Common TA 是同一波束/小区内相对参考点共享的传播分量，不是“整条卫星 RTT 的另一个名字”。对再生载荷，可把参考点至卫星的分量作为公共参考；对透明载荷，还要明确馈电链路由 UE 估计、网络广播还是网络侧补偿。RAR 中的 TA 因而更接近残余细化量：

\[
\Delta T_{\mathrm{TA}}
=
T_{\mathrm{TA,true}}
-
\hat T_{\mathrm{TA,UE}}.
\]

由于 UE 可能低估或高估初始 TA，细化量在物理上可能向两个方向修正。

> **工程判断：**GNSS 失效并不只会造成“TA 变差”。它会同时放大 PRACH 接收窗、残余 CFO 搜索范围和跟踪环初始误差；若网络仍广播 Common TA，UE 可以退化为“公共分量 + 网络细化”，但可支持的波束尺寸、RO 密度和接入时延需要重新核算。

> **原文定位：**TR 38.811 Clauses 7.3.4.1-7.3.4.2；TR 38.821 Clauses 6.3.4、7.2.1.1.1.2，Figures 7.2.1.1.1.2-8 至 7.2.1.1.1.2-9。38.811 识别原机制范围问题，38.821 研究 Common TA、UE autonomous TA 和 network refinement；两者的标准状态不同。

#### 3.1.2 PRACH 序列、相关检测与保护区

NR PRACH 前导基于 Zadoff-Chu（ZC）根序列及其循环移位构造。为突出 NTN 中的物理含义，可用简化形式表示根序列：

\[
x_u[n]
=
\exp\!\left(
-j\frac{\pi u n(n+1)}{N_{\mathrm{ZC}}}
\right),
\qquad 0\le n<N_{\mathrm{ZC}},
\]

同一根 \(u\) 上第 \(v\) 个前导可写为：

\[
x_{u,v}[n]
=
x_u\!\left[
(n+C_v)\bmod N_{\mathrm{ZC}}
\right],
\qquad
C_v=vN_{\mathrm{CS}},
\]

其中 \(N_{\mathrm{CS}}\) 表示 ZC 序列索引域中的循环移位间隔，并不是采样率、CP 长度或 TA 步长。配置较大的 \(N_{\mathrm{CS}}\) 会扩大零相关区，但同一根可生成的可用前导数相应下降；若所需前导数量超过单根可提供的数量，就需要继续使用其他根序列。

在存在到达时延 \(n_\tau\) 和归一化 CFO \(\epsilon\) 时，接收序列可写为：

\[
r[n]
=
\alpha x_{u,v}[n-n_\tau]
\exp\!\left(j\frac{2\pi\epsilon n}{N_{\mathrm{ZC}}}\right)
+w[n].
\]

接收机对候选根、循环移位、时延和频偏进行联合或分级搜索，例如：

\[
R_{u,v}[m]
=
\left|
\sum_n r[n]x_{u,v}^{*}[n-m]
\right|^2.
\]

理想零 CFO 下，循环移位把不同前导的相关峰分隔开；残余 CFO 会使相关能量泄漏、峰值降低或出现根相关的错误候选。于是 NTN PRACH 检测至少有三种不同的失败模式：

| 失败模式 | 物理原因 | 典型后果 | 主要控制量 |
|---|---|---|---|
| 循环移位混淆 | 到达时延越过零相关区或落入另一前导的移位区域 | preamble index 误判 | \(N_{\mathrm{CS}}\)、restricted set、残余时延 |
| 频偏引起的相关失真 | 残余 CFO 破坏理想 ZC 相关特性 | 峰值损失、假峰、检测率下降 | 预补偿、频偏搜索、SCS、重复 |
| RO 标签混淆 | 相邻 RO 的接收窗口在绝对时间上重叠 | 同一前导无法唯一关联发送 RO | RO 间隔、前导分组、显式时频标签 |

受限集合（restricted set）从循环移位集合中选择对高速或多普勒条件更稳健的组合，它处理的是“同一根上的序列可分辨性”；它不会自动消除公共绝对时延，也不会替代 RO 标签设计。类似地，CP 保护的是一个 PRACH 格式内部的时间不确定性和多径扩展，RAR TA 调整的是后续上行发射时刻，二者都不能单独证明 RO 归属。

> **标准状态：**TR 38.821 Clause 6.3.3 在“UE 不做时频预补偿”的假设下研究了更大 SCS、重复、多 ZC 根、Gold/m 序列和附加 scrambling 等候选；报告要求在规范阶段继续筛选，不能把这些候选全部写成现网必选特性。ZC 生成、根序列和 restricted set 的规范定义见 TS 38.211 Clause 6.3.3.1。

#### 3.1.3 PRACH 接收窗口与 RO 模糊

对同一波束内第 \(i\) 个 UE：

\[
\tau_i
=
\tau_{\mathrm{common}}
+
\Delta\tau_i.
\]

公共时延可以由参考点和辅助信息处理，物理随机接入信道（Physical Random Access Channel，PRACH）接收机仍需覆盖差分时延、预补偿残差和残余 CFO。三个机制边界不能互相替代：

| 环节 | 解决的问题 | 主要约束 |
|---|---|---|
| PRACH前导与检测 | Msg1能否被无歧义检测 | 前导CP、相关峰、搜索窗和残余CFO |
| RAR监测窗口 | UE何时等待Msg2 | 上下行传播、处理调度和剩余不确定性 |
| RAR中的TA | 如何把到达误差通知UE | TA范围、粒度和公共参考 |

若所有 UE 使用同一个 RACH Occasion（RO），网络接收窗至少需要覆盖最早与最晚到达者。TR 38.821 使用往返对齐关系描述时，接收窗跨度与小区内最大单程差分时延的两倍相关：

\[
W_{\mathrm{rx}}
\gtrsim
2(\tau_{\max}-\tau_{\min})
+W_{\mathrm{res}},
\]

其中 \(W_{\mathrm{res}}\) 包含预补偿残差、接收机搜索余量和实现裕量。

当差分范围大于相邻 RO 间隔时：

\[
\mathrm{RxWindow}(RO_1)
\cap
\mathrm{RxWindow}(RO_2)
\ne\varnothing,
\]

gNB 检测到 Zadoff-Chu 前导后可能无法确定它属于哪个发送 RO，继而无法唯一恢复相对发送时间：

\[
\boxed{
\text{RO ambiguity}
\rightarrow
\text{TA ambiguity}.
}
\]

因此 beam footprint 不只是覆盖参数，它还通过差分距离约束 PRACH 搜索范围、RO 间隔和前导检测复杂度。

TR 38.821 给出的研究解法揭示了几种互不等价的资源权衡：

| 方法 | 如何增加 RO 可识别性 | 代价或边界 |
|---|---|---|
| 拉大相邻 RO 时间间隔 | 接收窗不再重叠 | 降低单位时间接入机会数 |
| 给邻近 RO 分配不同前导组 | 用 preamble group 携带 RO 标签 | 每个 RO 的有效前导数下降 |
| 在不同频带发送/跳频 | 用接收频带辅助判断 RO | 需要显式配置与接收机支持；研究候选并非自动能力 |
| MsgA 携带 SFN/时间辅助 | 数据部分显式说明发送时刻 | 依赖两步接入且 MsgA PUSCH 需先可解调 |

“频域复用多个 RO”与“一个前导做频率跳变”也要分开：前者直接增加接入机会数，后者提供多个频率观察或标签。跳频可能帮助区分 CFO、时延或相邻 RO，但不会凭空增加 ZC 正交签名数，也不能在没有映射规则时自动消除歧义。

TR 以 200 km 小区半径评估差分距离：

| 最低仰角 | 参考差分距离 |
|---:|---:|
| 10° | 390 km |
| 20° | 372 km |
| 40° | 303 km |
| 60° | 197 km |
| 80° | 67 km |

最低仰角越低，同样地面半径对应的斜距差越大。若位置辅助可把公共和大部分 UE-specific 几何时延预先消除，现有 PRACH 格式可能适用；若辅助不可用，前导格式、搜索窗或 RO 配置可能需要增强。

> **原文定位：**TR 38.811 Clause 7.3.4.1.1、Figure 7.3.4.1.1-1、Table 7.3.4.1.2-1；TR 38.821 Clause 7.2.1.1.1.2、Figures 7.2.1.1.1.2-1 至 7.2.1.1.1.2-2、Table 7.2.1.1.1.2-1。

#### 3.1.4 RAR 监测与 Msg3 调度

从 UE 发送 Msg1 到接收 Msg2 的时间为：

\[
\begin{aligned}
t_{\mathrm{RAR,rx}}-t_{\mathrm{PRACH,tx}}
={}&
\tau_{\mathrm{UL}}(t_0)
+T_{\mathrm{detect}}
+T_{\mathrm{schedule}}\\
&+
\tau_{\mathrm{DL}}(t_1),
\end{aligned}
\]

其中 \(t_1>t_0\)。LEO 在检测和调度期间继续运动，上下行还可能采用不同网关、馈电路径或载频，因此 \(\tau_{\mathrm{UL}}\) 与 \(\tau_{\mathrm{DL}}\) 在概念上应分别记账。

若 UE 从 Msg1 发送完毕后立即长时间盲监听 PDCCH，会增加功耗。更合理的候选方式是利用已知公共 RTT 延后 RAR window 的起点，再让窗口长度覆盖处理抖动和差分不确定性：

\[
t_{\mathrm{monitor,start}}
\approx
t_{\mathrm{Msg1}}
+
\hat T_{\mathrm{RTT}}.
\]

位置辅助四步随机接入还存在一个隐蔽问题：gNB 从 Msg1 到达误差可以知道 UE 还需修正多少，却未必知道 UE 在 Msg1 前已自主应用了多少绝对 TA：

\[
\text{arrival timing error}
\neq
\text{applied absolute TA}.
\]

因此网络在 Msg2 后调度 Msg3 时可能仍缺少 UE 完整 UL timeline，只能基于 cell 最大传播时延或最大差分时延进行保守调度。RAR→Msg3 的逻辑关系还需要使用第 2.3 节定义的 \(K_{\mathrm{offset}}\)：

\[
n_{\mathrm{Msg3}}
=
n_{\mathrm{RAR}}
+K_2
+\Delta
+K_{\mathrm{offset}},
\]

其中 \(\Delta\) 表示首次 Msg3 PUSCH 的附加处理时间。

把知识状态写成估计量更直观。若 UE 实际应用的初始 TA 为 \(T_{\mathrm{app}}\)，网络从 Msg1 相关峰只直接观测到到达误差 \(e_{\mathrm{arr}}\)：

\[
e_{\mathrm{arr}}
=
T_{\mathrm{true}}
-T_{\mathrm{app}}
+e_{\mathrm{det}}.
\]

网络可在 RAR 中发送 \(e_{\mathrm{arr}}\) 的量化修正，却不能仅凭该差值唯一反推出 \(T_{\mathrm{app}}\)。直到 UE 显式报告或 Msg3 让网络建立完整时间状态之前，Msg3 grant 需要保守覆盖剩余不确定性。这也是“RAR TA 可修正 Msg1 到达”与“网络知道 UE 绝对 TA”不可等同的原因。

> **原文定位：**TR 38.811 Clause 7.3.4.1.2；TR 38.821 Clauses 6.2.1、7.2.1.1.1.2，Figures 7.2.1.1.1.2-3、7.2.1.1.1.2-9。RAR window start、绝对 TA 知识和 Msg3 调度应分别核算。

#### 3.1.5 RACH 容量、碰撞与 SSB 分区

若每秒有 \(N_t\) 个时域 RO、每个时刻配置 \(N_f\) 个频域 RO，并且每个 RO 对某类 UE 实际可用的 contention-based 前导数为 \(M_{\mathrm{eff}}\)，则简化签名空间为：

\[
N_{\mathrm{opp}}=N_tN_f,
\qquad
S=N_{\mathrm{opp}}M_{\mathrm{eff}}.
\]

当 \(U\) 个 UE 独立、均匀选择这 \(S\) 个资源签名时，某个 UE 至少与另一个 UE 碰撞的概率为：

\[
P_{\mathrm{col}}
=
1-\left(1-\frac{1}{S}\right)^{U-1},
\]

单轮期望成功数近似为：

\[
\mathbb E[N_{\mathrm{succ}}]
=
U\left(1-\frac{1}{S}\right)^{U-1}.
\]

这里 \(M_{\mathrm{eff}}\) 通常小于“协议上最多 64 个前导”的口头值，因为还要扣除无竞争接入、SI request、不同 SSB 的前导分区、restricted set/root 设计及接收机可可靠区分的候选。多使用根序列可以凑足配置前导，不意味着在接收机处理预算、RO 数量都不变时容量必然等比例增长。

TS 38.331 的 `ssb-perRACH-OccasionAndCB-PreamblesPerSSB` 同时配置每个 RO 关联的 SSB 数，以及每个 SSB 的 contention-based preamble 数。其容量含义是：

- 一个 SSB 可映射到多个 RO：给该波束更多接入机会，但消耗更多时频资源；
- 多个 SSB 共享一个 RO：节省 RO，但需要按 SSB 切分前导，且热点波束更容易碰撞；
- 总有效容量必须同时看 RO 密度、频域复用、每 SSB 前导数和 RO 歧义保护，不能只看单个字段。

例如，将 RO 时间间隔拉大以消除 NTN 差分时延歧义，会直接降低 \(N_t\)；再把一个 RO 分给多个 SSB，又会降低每个 SSB 的 \(M_{\mathrm{eff}}\)。这解释了为什么大波束、无 GNSS 接入和高并发三者不能无代价兼得。

> **原文定位：**TR 38.821 Clauses 7.2.1.1.1.1-7.2.1.1.1.2；TS 38.331 RACH-ConfigCommon 的 totalNumberOfRA-Preambles、restrictedSetConfig 与 ssb-perRACH-OccasionAndCB-PreamblesPerSSB 字段。上式是便于设计核算的均匀选择近似，不代替 3GPP 的具体业务到达模型。

#### 3.1.6 TA 范围与波束尺寸

TR 38.811 给出的参考最大链路距离随 SCS 增大而缩小：

| SCS | 报告参考最大链路距离 |
|---:|---:|
| 15 kHz | 300 km |
| 30 kHz | 150 km |
| 60 kHz | 75 km |
| 120 kHz | 37.5 km |
| 240 kHz | 18.75 km |

若传统 RAR TA 直接承担全部卫星绝对时延，即使 15 kHz 对 LEO/GEO 也不足；公共传播时延另行处理后，RAR TA 才主要约束波束内差分距离。设最短、最长斜距为 \(D_{\min}\)、\(D_{\max}\)：

\[
\Delta T_{\mathrm{TA,max}}
=
\frac{2(D_{\max}-D_{\min})}{c}.
\]

在最低仰角 10°、15 kHz SCS、允许最大差分距离 300 km 的参考条件下，报告给出的最大波束直径约为 305 km，并且对 GEO、MEO 和不同高度 LEO 的数值十分接近。这说明卫星高度明显大于波束尺寸时，差分 TA 主要由波束直径和最低仰角决定，而不是由轨道高度单独决定。

> **原文定位：**TR 38.811 Clauses 7.3.4.2.1-7.3.4.2.3，Tables 7.3.4.2.1-1、7.3.4.2.2-1。表格是研究评估参考，后续版本 NTN TA 的规范机制应以对应 TS 为准。

#### 3.1.7 两步随机接入的时间信息

两步随机接入把前导与上行数据合并为：

\[
\text{MsgA}
=
\text{PRACH}
+
\text{PUSCH payload}.
\]

网络随后在 MsgB 中完成响应和竞争解决。从 TA 角度，MsgA 的 PUSCH payload 允许 UE 在更早阶段报告已应用的初始 TA 或相关辅助信息，使网络更早建立对 UE 绝对 UL 时间基准的认知，减少四步随机接入中“Msg1 已对齐但网络仍不知道 UE 已应用多少 TA”的调度不确定性。

但 MsgA 中的 PUSCH 比纯前导更依赖可用的初始时间和频率补偿：若 UE 的预补偿误差已经使 PUSCH 超出 CP、DM-RS 或接收机频偏容限，网络就无法读取“我应用了多少 TA”这条辅助信息。因此两步随机接入不会消除 MsgA 的传播时间，也不会自动解决残余 CFO、PRACH 窗口和竞争冲突；它减少消息往返并提前交换时间状态，却对 MsgA 前的初始同步质量提出更直接要求。

| 比较项 | 四步随机接入 | 两步随机接入 |
|---|---|---|
| 首次上行 | Msg1：PRACH 前导 | MsgA：PRACH + PUSCH |
| 网络首次获得 UE 数据 | Msg3 | MsgA |
| 已应用 TA 的报告时机 | Msg3 或后续状态建立 | 可在 MsgA PUSCH 中提前报告 |
| 初始预补偿要求 | PRACH 必须可检测 | PRACH 可检测且 PUSCH 可解调 |
| NTN 主要收益 | 基线流程成熟 | 减少往返并降低 Msg3 调度未知量 |

该内容属于 TR 38.821 对 NTN 随机接入候选方案的研究，不应写成所有 NTN UE 必须使用两步随机接入。

> **原文定位：**TR 38.821 Clause 7.2.1.1.2、Figures 7.2.1.1.2-1 至 7.2.1.1.2-2；相关 TA 框架见 Clause 6.3.4。具体两步随机接入字段和规范行为应以相应 TS 版本为准。

### 3.2 SSB 与下行初始同步

同步信号块（Synchronization Signal Block，SSB）包含主同步信号（Primary Synchronization Signal，PSS）、辅同步信号（Secondary Synchronization Signal，SSS）和物理广播信道（Physical Broadcast Channel，PBCH）。UE 通过 PSS/SSS 完成初始时频同步和物理小区标识检测，再解调 PBCH 获得主信息块及继续读取系统信息所需的基础配置。

#### 3.2.1 PSS、SSS、PBCH 与 SSB index

一次 SSB 候选检测不是单一相关操作，而是一条逐步缩小假设空间的链：

| 组成 | 主要任务 | 典型输出 | 不能单独提供 |
|---|---|---|---|
| PSS | 粗符号定时、粗频偏、\(N_{\mathrm{ID}}^{(2)}\) | 3 个扇区内身份候选之一 | 完整 PCI、系统信息 |
| SSS | 帧内同步细化、\(N_{\mathrm{ID}}^{(1)}\) | 与 PSS 合成 PCI | 星历、Common TA |
| PBCH DM-RS | PBCH 信道估计、细同步与 SSB index 相关判决 | PBCH 解调条件、SSB 候选识别 | 完整 NTN 接入配置 |
| PBCH/MIB | 最小系统信息与 SIB1 入口 | 帧结构基础量、CORESET#0/SearchSpace#0 入口 | 全部 PRACH/NTN 辅助参数 |

物理小区标识由 PSS/SSS 的两部分组成：

\[
N_{\mathrm{ID}}^{\mathrm{cell}}
=
3N_{\mathrm{ID}}^{(1)}
+N_{\mathrm{ID}}^{(2)}.
\]

SSB index 与 PCI 不是同一变量。一个 PCI 下可以有多个 SSB index，对应不同发送时刻和实现相关的波束方向；多个 SSB/波束可以共享同一 PCI。UE 检出 PCI 后仍需借助 PBCH DM-RS、PBCH 信息和配置确定候选 SSB 及其后续接入资源。

#### 3.2.2 原始多普勒、二维搜索与同步捕获

TR 使用地面 UE 约 5 ppm 初始频偏鲁棒性作为比较基线：

\[
5\,\mathrm{ppm}\times2\,\mathrm{GHz}
=10\,\mathrm{kHz},
\]

\[
5\,\mathrm{ppm}\times20\,\mathrm{GHz}
=100\,\mathrm{kHz}.
\]

600 km LEO 的参考最大原始多普勒在 2 GHz 和 20 GHz 分别约为 ±48 kHz 和 ±480 kHz，均高于对应数值。如果 UE 在完全未知多普勒条件下直接搜索 PSS/SSS，地面 NR 的单次捕获范围可能不足。

对一个候选同步序列 \(s[n]\)，接收机实际进行的是时间—频率二维搜索：

\[
C(\hat\tau,\hat f)
=
\left|
\sum_n
r[n]s^*[n-\hat\tau]
e^{-j2\pi\hat f nT_s}
\right|^2.
\]

搜索范围越大，候选栅格、计算量、虚警控制和捕获时间越高。NTN 的关键不是只提高某个相关器门限，而是先用几何与网络补偿缩小 \((\hat\tau,\hat f)\) 的先验范围。

但同步判据应使用预补偿后的残余量：

\[
\delta f_{\mathrm{sync}}
=
f_D
-
\hat f_{D,\mathrm{common}}
-
\hat f_{D,\mathrm{UE}}.
\]

波束中心公共预补偿、UE 位置、卫星星历和后续频偏估计都可以缩小搜索范围。因此大原始多普勒不必然要求修改同步信号；只有残余范围仍超出捕获能力时，才需要扩大搜索、增加辅助信息或研究同步增强。TR 38.821 的评估进一步观察到：GEO 以及采用波束级公共频移预补偿的 LEO 可以复用 Rel-15 SSB；LEO 若不做频移预补偿，UE 接收机需要增加搜索复杂度，但报告仍未识别出修改 SSB 波形的必要性。

完整捕获链可整理为：

\[
\begin{aligned}
\text{raw satellite Doppler}
&\rightarrow
\text{network/beam common pre-compensation}\\
&\rightarrow
\text{UE GNSS/ephemeris prediction}
\rightarrow
\text{PSS coarse CFO}\\
&\rightarrow
\text{SSS/PBCH DM-RS fine tracking}
\rightarrow
\text{PRACH residual CFO}.
\end{aligned}
\]

SSB 同步与 Msg1 发送之间还存在信息老化。若两者相隔 \(T_{\mathrm{age}}\)，一阶近似为：

\[
\delta f_{\mathrm{Msg1}}
\approx
\delta f_{\mathrm{sync}}
+\dot f_D T_{\mathrm{age}}
+b_f,
\]

\[
e_{\tau,\mathrm{Msg1}}
\approx
e_{\tau,\mathrm{sync}}
+\dot\tau T_{\mathrm{age}}
+b_t.
\]

所以“SSB 已捕获”并不等于“PRACH 一定在窗内”。实现仍要考虑 SIB1 读取时间、RO 等待时间、卫星运动和本振漂移，并在 Msg1 前更新 TA/CFO 预测。

> **原文定位：**TR 38.811 Clause 7.3.2.3；TR 38.821 Clause 6.3.2。报告中约 13,000 km 的高度判断来自特定 5 ppm 与最大几何多普勒比较，不是通用部署门限。

#### 3.2.3 波束扫描、SSB 选择与 SSB-to-RO 映射

在波束扫描部署中，一个 SSB burst set 可在不同 SSB index 上发送多个候选波束。UE 对可检测 SSB 测量 SSB-RSRP，并根据门限与选择规则确定候选 SSB；网络再通过 SSB-to-RO 映射把该选择带入随机接入。

TS 38.331 的 RACH-ConfigCommon 同时提供 rsrp-ThresholdSSB 和 ssb-perRACH-OccasionAndCB-PreamblesPerSSB。后一个字段包含两个维度：

1. 一个 RO 对应多少个 SSB，或一个 SSB 分散到多少个 RO；
2. 每个 SSB 分配多少个 contention-based preamble。

因此 gNB 可以从“在哪个 RO、检测到哪个前导集合”反推出 UE 选择的 SSB/候选波束，而不必在 Msg1 中显式发送完整波束编号。该映射也形成资源—碰撞权衡：

| 映射方式 | 波束识别 | 资源效率 | 碰撞/热点影响 |
|---|---|---|---|
| 一个 SSB 对多个 RO | 识别直接 | 占用更多 RO | 单波束机会增加 |
| 一个 SSB 对一个 RO | 关系简单 | 中等 | 取决于每 SSB 前导数 |
| 多个 SSB 共享一个 RO | 依靠前导分区识别 | 节省 RO | 每 SSB 可用前导减少，热点更敏感 |

NTN 中还要叠加第 3.1.3 节的 RO 接收窗重叠问题。即使 SSB-to-RO 配置在逻辑上唯一，若长差分时延使相邻 RO 在网络侧不可分，物理到达仍会破坏该映射。因此 SSB/RO 配置、波束足迹和 Common/Differential TA 需要联合设计。

> **跨笔记边界：**SSB 波束扫描方式、卫星波束与 PCI/小区组织由[天线波束与覆盖组织](./03_NTN天线波束与覆盖组织_学习笔记.md)展开；本节只保留 SSB 选择进入 PRACH 资源映射的接口。

#### 3.2.4 SSB、PBCH 与系统信息边界

SSB 的物理层任务是提供同步、物理小区标识、SSB index 相关信息和 PBCH 解调入口。星历、Common TA、timing drift rate 或其他 NTN 辅助量是否以及如何通过系统信息提供，属于更高层配置问题，不能笼统写成“SSB 本身携带全部 NTN 辅助信息”。

从 RACH 的输入角度，UE 在发送 Msg1 前可能需要：

| 输入 | 作用 | 获得层面 |
|---|---|---|
| 下行符号与帧时间 | 建立接收时间基准 | PSS/SSS/PBCH |
| PCI | 识别物理小区 | PSS/SSS |
| SSB index 与 PBCH 解调条件 | 识别候选 SSB/波束并读取 MIB | PBCH DM-RS、PBCH |
| 粗频偏估计 | 缩小残余 CFO | PSS/SSS及接收机估计 |
| SIB1 读取入口 | 获取初始接入所需配置 | PBCH/MIB 指向 CORESET#0/SearchSpace#0 |
| PRACH资源关联 | 选择RO和前导资源 | SIB1/RRC中的RACH与SSB-to-RO配置 |
| 星历、Common TA或漂移辅助 | Msg1前TA/多普勒预补偿 | NTN系统信息或其他配置，取决于标准版本 |
| UE位置与本地时间 | 形成几何预测输入 | UE GNSS/其他定位能力，不属于 SSB |

这一分层避免把同步信号、广播信道和 RRC 系统信息混成同一对象。

#### 3.2.5 从 SSB 到 Msg1 的状态传递与失败定位

SSB 到 RACH 的物理顺序可压缩为：

\[
\begin{aligned}
\text{PSS/SSS detection}
&\rightarrow
\text{PCI + coarse time/frequency}
\rightarrow
\text{PBCH/MIB}\\
&\rightarrow
\text{SIB1/RACH configuration}
\rightarrow
\text{SSB/RO/preamble selection}\\
&\rightarrow
\text{TA/CFO prediction update}
\rightarrow
\text{Msg1}.
\end{aligned}
\]

用失败现象反推环节时，可按下表排查：

| 现象 | 优先怀疑 | 不应直接归因于 |
|---|---|---|
| PSS 峰始终不稳定 | 原始/残余 CFO 超范围、搜索栅格、低 SNR | PRACH TA 命令 |
| PCI 可得但 PBCH 失败 | 精频偏、信道估计、SSB index 假设 | SSB-to-RO 碰撞 |
| SIB1 可读但 Msg1 无检测 | TA/CFO 老化、PRACH 功率、RO 选择、前导检测 | “SSB 波形必然要改” |
| Msg1 有峰但 RO 不确定 | 差分时延造成接收窗重叠 | 单纯扩大 CP |
| Msg2 可收但 Msg3 调度困难 | 网络缺少 UE 已应用的绝对 TA | PSS/SSS PCI 检测 |

#### 3.2.6 测量与后续消费者

SSB-RSRP 可进入波束测量、Layer 3 滤波和路径损耗估计。其消费者分属不同笔记和章节：波束候选与小区组织由[天线波束与覆盖组织](./03_NTN天线波束与覆盖组织_学习笔记.md)负责；RSRP 滤波和 PUSCH 功控由第 2.6 节负责；SSB-to-RO 关联、粗频偏和时间参考则进入第 3.1 节的随机接入。

因此 SSB 在本篇的稳定输出为：

\[
\boxed{
\text{下行时间基准}
+
\text{粗频率基准}
+
\text{小区/SSB身份}
+
\text{接入配置入口}.
}
\]

### 3.3 服务状态迁移与仿真接口

完整的波束足迹、逻辑小区、PCI、BWP 和波束管理流程由第三份笔记拥有。本篇只描述服务波束或卫星变化时需要共同迁移的空口状态：

\[
\mathbf s_{\mathrm{air}}
=
[
\mathrm{Beam},
\mathrm{Satellite},
\mathrm{ReferencePoint},
T_{\mathrm{TA}},
f_D,
K_{\mathrm{offset}}
].
\]

切换并不只是选择新的最大 RSRP 波束。不同服务状态可能对应新的 TA 参考点、公共多普勒预补偿、频率资源、SSB/CSI-RS 配置和逻辑时序偏移。几何预测可提前给出候选卫星、预计切换时刻和参考量；反馈老化决定测量报告到达时是否仍然有效；协议过程负责在目标状态生效前保持调度因果性。

向[链路、系统与多星仿真](./06_NTN链路系统与多星仿真_学习笔记.md)输出的状态为：

| 状态 | 更新依据 | 仿真处理 |
|---|---|---|
| Common/Differential/Residual TA | 几何、参考点、预补偿与测量 | PRACH到达窗口和上行对齐 |
| \(K_{\mathrm{offset}}\) | 时间架构、配置与过程触发 | 调度、反馈和参考资源时间轴 |
| 原始/公共/残余多普勒 | 星历、位置、估计与补偿时效 | 捕获范围、残差模型和BLER映射 |
| CSI/CQI/MCS状态 | 参考信号、反馈RTT与调度器 | 信息老化、预测和链路自适应 |
| RSRP/功控状态 | 测量、L3滤波和功控命令 | 发射功率和功率受限事件 |
| HARQ/RLC/队列状态 | ACK/NACK、定时器和缓存 | 进程占用、重传与时延统计 |
| Beam/Satellite/Reference Point | 几何预测与服务状态机 | 多速率切换和联合状态迁移 |

系统级抽象不能把 RTT、CSI 老化、TA 残差和 HARQ 进程占用统一写成固定 SINR 扣减。它们分别作用于物理到达时间、决策信息、接收机残差和队列状态；只有明确映射假设并使用链路级结果校准后，才可把其中一部分折算为等效性能损失。
