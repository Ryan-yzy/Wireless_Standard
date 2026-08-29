---
title: "NTN 空口时频与协议适配学习笔记"
date: "2026-08-26"
updated: "2026-08-29"
sources:
  - "3GPP TR 38.811 V15.4.0"
  - "3GPP TR 38.821 V16.2.0"
  - "3GPP TS 38.211 V16.2.0 (Release 16)"
  - "3GPP TS 38.213 V16.3.0"
  - "3GPP TS 38.214 V16.7.0"
  - "3GPP TS 38.300 V16.17.0"
  - "3GPP TS 38.331 V16.18.0"
---

# NTN 空口时频与协议适配学习笔记

> 本笔记接收[轨道、时间与链路几何](./02_卫星轨道时间与链路几何_学习笔记.md)、[天线波束与覆盖组织](./03_NTN天线波束与覆盖组织_学习笔记.md)以及[传播损耗与信道模型](./04_NTN传播损耗与信道模型_学习笔记.md)输出的时延、多普勒、波束和局部信道状态，负责把它们映射为 NR 空口约束、反馈闭环和接入过程状态；节点架构、波束足迹和小区组织不在本篇重复展开。

| 主章节 | 核心对象 | 判断尺度 | 主要来源 |
|---|---|---|---|
| 1 系统级空口时频与波形约束 | TA、残余 CFO、SCS、DM-RS、PT-RS、CP、双工与波形 | 波形、符号、时隙和收发切换 | TR 38.811 Clauses 7.3.2、7.3.5-7.3.7，TR 38.821 Clause 6.1.2 |
| 2 测量、反馈与控制闭环 | TA/频偏维护、\(K_{\mathrm{offset}}\)、HARQ/RLC、CSI/CQI、MCS与功控 | 测量到执行的闭环时间 | TR 38.811 Clause 7.3.3、TR 38.821 Clauses 6.2-6.4 |
| 3 下行同步、系统信息与随机接入 | PSS/SSS、SSB/PBCH、辅助能力、PRACH/RAR/Msg3与联合空口状态 | 捕获、广播、接入消息和状态转换 | TR 38.811 Clauses 7.3.2.3、7.3.4，TR 38.821 Clauses 6.3、7.2.1.1 |

全文按以下知识链组织：

\[
\begin{aligned}
&\text{几何与信道输入}
\rightarrow
\text{系统级波形约束}
\rightarrow
\text{测量反馈闭环},\\
&\text{PSS/SSS与SSB捕获}
\rightarrow
\text{系统信息与RACH过程}
\rightarrow
\text{协议与仿真状态}.
\end{aligned}
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

#### 1.2.3 时间参考与馈电链路切换

TA 参考不是只由服务链路决定。透明转发架构中，公共部分还经过馈电链路；若馈电链路从一个 Gateway/卫星连接切换到另一个连接，公共传播时延、频率变换链和网络侧 DL/UL 时间关系都可能变化。UE 即使仍处于同一逻辑小区和地面覆盖区域，也可能需要更新 Common TA、公共频率预补偿和跨 DL-UL 逻辑偏移。

TR 38.821 没有为 feeder-link switch 给出一套收敛的 PHY 过程，只建议在规范阶段继续讨论其影响。因此本篇只把馈电链路身份和公共时间参考作为联合状态输入，不把某种切换流程写成既定规范；透明/再生载荷及 Gateway 架构详见[系统架构与部署场景](./01_NTN系统架构与部署场景_学习笔记.md)。

> **原文定位：**TR 38.821 Clauses 6.2.5、9.1。报告将 LEO 馈电链路切换对物理层过程的影响列为后续规范讨论项，未在研究阶段收敛具体机制。

#### 1.2.4 TA、SCS 与跨时隙调整

TR 38.811 以最大约 \(35\,\mu\mathrm{s/s}\) 的时延漂移评估 TA 更新。不同子载波间隔（Subcarrier Spacing，SCS）下，TA 量化步长和理论最大 TA step 不同；但报告同时指出，表中的最大 step 只在扩展 CP 下直接成立，正常 CP 场景的实际单次调整应受正常 CP 约束：

| SCS | 时隙长度 | TA量化步长 | Table 7.3.2.2.2-1 理论最大 step | 正常 CP 代表长度 | 最大漂移下的命令量级 |
|---:|---:|---:|---:|---:|---:|
| 15 kHz | 1 ms | 520.83 ns | 约16.6 μs | 约4.688 μs | 约10次/s |
| 30 kHz | 0.5 ms | 260.42 ns | 约8.3 μs | 约2.344 μs | 约15次/s（工程外推） |
| 60 kHz | 0.25 ms | 130.21 ns | 约4.15 μs | 约1.172 μs | 约40次/s |
| 120 kHz | 0.125 ms | 65.10 ns | 约2.1 μs | 约0.586 μs | 约80次/s |
| 240 kHz | 0.0625 ms | 32.55 ns | 约1 μs | 约0.293 μs | 约120次/s（工程外推） |

表中 15/60/120 kHz 的次数来自报告量级分析；30/240 kHz 只是按 \(35\,\mu\mathrm{s/s}\) 与代表性正常 CP 尺度做的工程外推，不是标准规定的发送周期。理论最大 step、CP 限制和命令更新率属于三个不同概念，不能把 16.6 μs 与“约 10 次/s”直接视为同一组条件。

SCS 增大后 TA 时间粒度更细，但单次调整量变小；同一物理传播时延还对应更多 slot。因此更大 SCS 不能解决大 TA，UE 的发送时间必须允许跨 slot/TTI 边界整体移动，逻辑调度还需由 \(K_{\mathrm{offset}}\) 保证因果性。

> **原文定位：**TR 38.811 Clause 7.3.2.2、Table 7.3.2.2.2-1 及其后关于 extended CP/normal CP 的说明。命令次数为报告在给定最大漂移下的量级分析，不是所有实现的固定发送周期。

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

| SCS | 正常 CP 代表长度 |
|---:|---:|
| 15 kHz | 4.688 μs |
| 30 kHz | 2.344 μs |
| 60 kHz | 1.172 μs |
| 120 kHz | 0.586 μs |
| 240 kHz | 0.293 μs |

这些数值按常用符号给出代表量级；正常 CP 并非每个 OFDM 符号都完全相同，slot 中首符号或周期性边界符号的 CP 会随 numerology 略长。判断 ISI 时应使用实际符号位置对应的 TS 38.211 参数，而不是把表中代表值当作所有符号的唯一 CP。

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

DFT-s-OFDM 的主要价值是较低 PAPR、较小回退和更高有效 EIRP，不表示其在相同 RE、调制阶数和编码率下天然具有较低频谱效率。TR 38.811 在影响研究阶段讨论了 CP-OFDM 的 PAPR 降低并指出上行 DFT-s-OFDM 可能有利；随后 TR 38.821 的 RAN1 最终建议明确，至少对 Rel-17 不需要规定 NTN 专用的下行 PAPR 优化。该结论只限定规范化必要性，不表示卫星功放回退、带内失真或带外泄漏可以忽略。TR 38.821 的数据 LLS 基线也未纳入 HPA 非线性，不能把基线排除项反推成硬件无影响。

> **原文定位：**TR 38.811 Clause 7.3.7.2、Figure 7.3.7.2.1-1；TR 38.821 Clause 6.1.2、Table 6.1.2-4、Clause 9.1。总损失示例来自 TR 38.811 引用的 ETSI TR 103 297。

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
\text{测量时刻 }t_{\mathrm{meas}}
\rightarrow
\text{报告时刻 }t_{\mathrm{report}}
\rightarrow
\text{决策时刻 }t_{\mathrm{decision}}
\rightarrow
\text{执行时刻 }t_{\mathrm{execute}}.
\]

滤波和预测可以位于测量、报告或决策侧，但判断信息是否过期时必须保留这四个时间戳。CSI 老化关注测量信道与数据执行信道的差异；TA/频率命令老化关注执行时的几何状态；HARQ 则必须等待某次传输的真实译码结果。它们共享长 RTT 背景，却不是同一种误差。

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

#### 2.2.1 几何预测与闭环残差

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

几何预测器使用 UE 和卫星位置、速度与视线方向。令 \(\mathbf u_{\mathrm{UE}\rightarrow\mathrm{sat}}\) 为 UE 指向卫星的单位向量，并约定斜距增大为正，则：

\[
\dot d
=
\mathbf u_{\mathrm{UE}\rightarrow\mathrm{sat}}^{T}
(\mathbf v_{\mathrm{sat}}-\mathbf v_{\mathrm{UE}}),
\qquad
\hat f_D
=
-\frac{f_c}{c}\dot d.
\]

在该符号约定下，卫星远离 UE 时 \(\dot d>0\)、多普勒为负；若实现使用“卫星指向 UE”的 LOS 向量，公式符号必须相应翻转。该约定与第二篇的 \(f_D=-f_c\dot d/c\) 保持一致。

同一组 \(\mathbf r_{\mathrm{sat}}(t)\)、\(\mathbf v_{\mathrm{sat}}(t)\) 同时产生斜距、传播时延、TA、斜距变化率和多普勒。网络测得的 UL 到达误差和残余 CFO 再修正位置误差、星历误差、网关群时延、晶振偏差和信息老化。

若位置误差对应的距离误差为 \(\Delta D\)，TA 误差量级为：

\[
\Delta T_{\mathrm{TA}}
=
\frac{2\Delta D}{c}.
\]

例如 \(\Delta D=100\,\mathrm{m}\) 时，\(\Delta T_{\mathrm{TA}}\approx0.67\,\mu\mathrm{s}\)。这一数值说明 GNSS 和星历可以提供粗对齐，但并不取消网络细化的必要性。

#### 2.2.2 时间同步的候选路径

TR 38.821 将上行定时维护归纳为两类研究方向。二者改变公共量和 UE 特定量的承担位置，但都需要处理估计误差和状态老化：

| 路径 | 公共部分 | UE 特定部分 | 网络侧作用 |
|---|---|---|---|
| UE autonomous TA | UE 根据位置、星历和必要的馈电链路信息计算 Full TA 或服务链路 TA | UE 计算差分量并持续外推 | 网络根据 UL 到达误差细化残差 |
| Network-indicated common TA | 网络按波束提供 Common TA | UE 使用 RAR/后续 TA 命令处理差分与剩余量 | 维护公共参考、扩展必要的 TA 范围并指示 timing drift |

单个波束使用一个参考点可简化公共广播与差分计算；多个参考点可能缩小局部差分范围，但其选择和 UE 关联在报告中仍未收敛。负方向 TA 修正、RAR TA 范围扩展以及 timing drift rate 的具体表示也属于研究阶段问题，不能直接写成所有 NTN Release 的统一配置。

#### 2.2.3 上行频率同步的候选路径

上行频率维护同样存在 UE 预测与网络闭环两条路径：UE 可以根据下行参考信号、位置和星历估计上行多普勒并预补偿；网络也可以根据 PRACH 或其他上行信号估计残余频偏，再向 UE 指示修正值。工程系统可以组合使用两者，即几何预测去除可预测公共量，网络估计校正晶振、位置、星历和馈电链路残差。

TR 38.821 认为可指示 UE 已补偿的频率值，但没有认为必须单独指示 Doppler drift rate。这里的“不需要漂移率指示”不表示漂移不存在，而是 UE 或网络可以通过周期更新、预测或频偏测量处理它。

#### 2.2.4 位置或星历不可用时的退化维护

位置与星历是 UE 自主预补偿的两个不同输入：只有卫星轨迹而不知道 UE 位置，不能唯一求出服务链路斜距和径向速度；只有 UE 位置而没有有效星历，也不能外推卫星状态。任一输入缺失或过期时，UE 可完成的几何预补偿都会下降，需要更多依赖网络广播的 Common TA、接收端时间/频率搜索和闭环修正。

本节负责连接建立后的持续维护；首次接入时如何选择补偿路径、PRACH 需要覆盖多大的残余不确定性，由第 3.2 节继续展开。

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

不同过程所需的附加偏移不一定相同，偏移也可能按波束或按小区维护。TR 38.821 在研究阶段没有收敛它应由广播信息推导还是由高层信令给出，也保留了扩展 \(K_1/K_2\) 范围的讨论空间。因此这里用统一的 \(K_{\mathrm{offset}}\) 表示共同原理，而不是断言所有过程共用一个规范字段和值。

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

TR 38.821 进一步讨论了两条总体路线：一条扩展 HARQ 进程和缓存以维持反馈、重传与软合并，另一条关闭或选择性关闭 HARQ 反馈并更多依靠 RLC ARQ。报告还记录了 DCI 指示、UCI disruption report、超过 8 slot 的聚合、时间交织聚合和新 MCS 表等候选，但这些方案没有收敛。RAN1 的后续建议仅要求继续讨论 HARQ 进程数、反馈、UE/gNB 缓存与 RLC ARQ 的联动，以及支持启用/禁用 HARQ 反馈，不能把任一候选写成 38.821 已确定的机制。

> **原文定位：**TR 38.811 Clauses 7.3.3.1-7.3.3.2，Figures 7.3.3.1-1、7.3.3.1.1-1，Table 7.3.3.1.1-1；TR 38.821 Clauses 6.4.1-6.4.2、9.1。表中进程数和 Clause 6.4 的方案均为研究评估或候选，不是所有 NTN 系统的强制配置。

### 2.5 CSI/CQI、预测与 MCS

自适应编码调制（Adaptive Coding and Modulation，ACM）使用信道状态信息（Channel State Information，CSI）和信道质量指示（Channel Quality Indicator，CQI）支持调度与调制编码方案（Modulation and Coding Scheme，MCS）选择。长 RTT 下真正使用的不是测量时刻信道，而是数据到达接收端时的信道。

| 场景 | 主要变化 | 闭环适用性 |
|---|---|---|
| GEO Ka | 雨衰通常较慢 | ACM可工作，但需滞回避免MCS振荡 |
| GEO S | 遮挡和局部多径可能较快 | 难以逐衰落跟踪，需要更大裕量 |
| LEO | 斜距引起的大尺度损耗可预测 | 可预测和跟踪大尺度变化，仍难跟踪快速多径 |
| 手持终端 | 姿态、阴影和局部遮挡 | 需要预测、滤波和保守MCS |

TR 38.821 的链路级结果说明，CSI aging 不是只由反馈时延决定。在 3 km/h、10 dB SNR 的参考评估中，NTN-TDL-C（LOS）把反馈时延从约 6 ms 增加到 40-46 ms 时，吞吐量或频谱效率损失约为 10%-12%；NTN-TDL-A（NLOS）的对应损失约为 28%-38%。另一个评估在 30 km/h 条件下没有观察到反馈时延从 6 ms 增加到 201 ms 的明显额外损失。不同信道、速度和实现得出的趋势并不单调一致，因此不能仅按 RTT 给 CSI 固定扣减。

报告评估的长期信道平均在 3 km/h、理想信道估计条件下没有改善吞吐量；预测式 CSI 在部分 NLOS 评估中约有 10% 增益，但预测可以仅作为实现算法还是需要新增 UE 报告参数没有共识。由此，Rel-15 CSI 框架至少可作为 LOS NTN 链路自适应基线，进一步优化仍未收敛。

> **原文定位：**TR 38.821 Clause 6.2.3。百分比是特定 NTN-TDL、SNR、速度和反馈时延下的来源结果，不是通用 NTN 性能保证。

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

> **原文定位：**TR 38.811 Clause 7.3.3.3；TR 38.821 Clause 6.2.3；MCS 和链路自适应基线参见 TS 38.214 Clauses 5.1.3、5.2.2 和 6.1.4。TR 没有规定上述工程分解的固定数值。

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

## 3 下行同步、系统信息与随机接入

本章按照 UE 建立接入认知和执行过程的先后关系组织：UE 首先接收同步信号块（Synchronization Signal Block，SSB），使用其中的主同步信号（Primary Synchronization Signal，PSS）和辅同步信号（Secondary Synchronization Signal，SSS）取得初始时频基准与物理小区标识，再解调物理广播信道（Physical Broadcast Channel，PBCH）并读取后续系统信息；获得 PRACH 资源及可用 NTN 辅助信息后，UE 才发送 Msg1 并进入随机接入。

\[
\text{PSS/SSS捕获}
\rightarrow
\text{PBCH与系统信息}
\rightarrow
\text{辅助能力与补偿路径}
\rightarrow
\text{PRACH}
\rightarrow
\text{RAR/Msg3}.
\]

PSS/SSS并不是在 SSB 之前单独发送的另一组信号；上述顺序表示 UE 对同一个 SSB 的处理层次，以及 SSB 输出如何成为 RACH 输入。

### 3.1 PSS/SSS、SSB与下行初始同步

SSB 包含 PSS、SSS、PBCH 及其解调参考信号。UE 先用 PSS/SSS 完成初始时间、频率和物理小区标识检测，再解调 PBCH 获得主信息块以及继续读取系统信息所需的基础配置。

#### 3.1.1 原始多普勒与同步捕获

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

> **原文定位：**TR 38.811 Clause 7.3.2.3；TR 38.821 Clause 6.3.2。报告中约 13,000 km 的高度判断来自特定 5 ppm 与最大几何多普勒比较，不是通用部署门限。

#### 3.1.2 SSB、PBCH与系统信息边界

SSB 的物理层任务是提供同步、物理小区标识、SSB index 相关信息和 PBCH 解调入口。星历、Common TA、timing drift rate 或其他 NTN 辅助量是否以及如何通过系统信息提供，属于更高层配置问题，不能笼统写成“SSB 本身携带全部 NTN 辅助信息”。

从 RACH 的输入角度，UE 在发送 Msg1 前可能需要：

| 输入 | 作用 | 获得层面 |
|---|---|---|
| 下行符号与帧时间 | 建立接收时间基准 | PSS/SSS/PBCH |
| 物理小区标识与SSB index | 关联候选波束和PRACH资源 | PSS/SSS、PBCH和配置 |
| 粗频偏估计 | 缩小残余 CFO | PSS/SSS及接收机估计 |
| PRACH资源关联 | 选择RO和前导资源 | 系统信息与SSB-to-RO配置 |
| 星历、Common TA或漂移辅助 | Msg1前TA/多普勒预补偿 | NTN系统信息或其他配置，取决于标准版本 |

这一分层避免把同步信号、广播信道和 RRC 系统信息混成同一对象。

#### 3.1.3 同步输出与后续消费者

SSB-RSRP 可进入波束测量、Layer 3 滤波和路径损耗估计。其消费者分属不同笔记和章节：波束候选与小区组织由[天线波束与覆盖组织](./03_NTN天线波束与覆盖组织_学习笔记.md)负责；RSRP 滤波和 PUSCH 功控由第 2.6 节负责；SSB-to-RO 关联、粗频偏和时间参考则进入第 3.2 节的随机接入。

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

### 3.2 NTN随机接入与初始时频对齐

随机接入把上游的传播时延、差分时延、残余 CFO 和时间参考具体化为 Msg1 检测、Msg2 监测、Msg3 调度和上行对齐问题。其核心不是把一个 TA 字段无限扩大，而是重组初始时延预测、网络细化和跨 DL-UL 调度。

#### 3.2.1 辅助能力与补偿路径

UE 能否在 Msg1 之前完成几何时频预补偿，取决于“UE 位置”和“卫星星历”两类信息是否同时可用且足够新鲜。系统信息还可能提供波束参考点、Common TA 或其他辅助量。四种基本状态为：

| UE位置 | 卫星星历 | Msg1前可用处理 | PRACH侧剩余压力 |
|---|---|---|---|
| 可用 | 可用 | UE可预测服务链路 TA 和上行多普勒，并结合网络公共量预补偿 | 主要覆盖位置、星历、晶振及馈电链路残差 |
| 可用 | 不可用或过期 | 不能可靠外推卫星距离与径向速度 | 需要网络提供更多公共量并扩大时频搜索 |
| 不可用 | 可用 | 知道轨道但不能唯一确定 UE 到卫星的斜距和径向速度 | Common TA只能去除公共部分，仍需覆盖 UE 特定差异 |
| 不可用 | 不可用 | 无法执行完整几何预补偿 | 主要依赖网络广播、接收端搜索及后续闭环修正 |

该表是基于输入可用性的机制分类，不是 TR 38.821 定义的 UE 能力组合。信息“存在”也不等于“有效”：位置误差、星历误差、参考时刻和信息老化必须共同决定预补偿残差。连接后的持续维护由第 2.2 节负责，本节只说明这些输入如何改变首次 PRACH 的条件。

#### 3.2.2 四步随机接入与初始TA

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

UE 自主计算时：

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

RAR 中的 TA 更接近细化量：

\[
\Delta T_{\mathrm{TA}}
=
T_{\mathrm{TA,true}}
-
\hat T_{\mathrm{TA,UE}}.
\]

由于 UE 可能低估或高估初始 TA，细化量在物理上可能向两个方向修正。

> **原文定位：**TR 38.811 Clauses 7.3.4.1-7.3.4.2；TR 38.821 Clauses 6.3.4、7.2.1.1.1.2，Figures 7.2.1.1.1.2-8 至 7.2.1.1.1.2-9。38.811 识别原机制范围问题，38.821 研究 Common TA、UE autonomous TA 和 network refinement；两者的标准状态不同。

#### 3.2.3 PRACH检测窗口与RO模糊

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

若所有 UE 使用同一个 RACH Occasion（RO），接收窗口至少需要覆盖：

\[
[\tau_{\min},\tau_{\max}].
\]

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

#### 3.2.4 无预补偿时的PRACH候选

TR 38.821 Clause 6.3.3 根据 UE 是否完成时频预补偿给出不同判断。UE 具有足够准确的位置和星历、能够在 Msg1 前预补偿时间与频率偏移的情况下，Rel-15 PRACH 格式和序列可以复用，是否通过重复或更大 SCS 改善覆盖可留到规范阶段讨论。若 UE 不进行预补偿，报告记录了四类增强候选：

| 候选 | 序列或波形方向 | 主要意图 | 研究状态 |
|---|---|---|---|
| 单个 Zadoff-Chu 序列 | 更大 SCS 和/或重复，CP 与 \(N_{\mathrm{CS}}\) 待确定 | 提高大频偏和低 SNR 下的检测能力 | 候选，未收敛 |
| 多个 Zadoff-Chu 序列 | 使用不同 root 的多根 ZC | 增加时频不确定性下的可观测结构 | 候选，未收敛 |
| Gold/m-sequence | 配合处理或变换预编码 | 改变大时频偏移下的相关与估计特性 | 候选，未收敛 |
| ZC加scrambling | 在 ZC 上增加扰码结构 | 改善检测或模糊区分能力 | 候选，未收敛 |

38.821 没有完成这些候选的 down-selection，也没有把其中任一种写成 NTN 统一 PRACH。因此本节只保留候选族及其问题背景；不同序列、root、循环移位、重复和接收机算法的波形级比较应在后续 RAN1 提案专题或[链路、系统与多星仿真](./06_NTN链路系统与多星仿真_学习笔记.md)中展开。

> **原文定位：**TR 38.821 Clause 6.3.3、Clause 9.1。RAN1 后续工作的焦点是 UE 不做时频预补偿时的 PRACH 序列和/或格式增强，报告本身没有完成候选选择。

#### 3.2.5 RAR监测与Msg3调度

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

> **原文定位：**TR 38.811 Clause 7.3.4.1.2；TR 38.821 Clauses 6.2.1、7.2.1.1.1.2，Figures 7.2.1.1.1.2-3、7.2.1.1.1.2-9。RAR window start、绝对 TA 知识和 Msg3 调度应分别核算。

#### 3.2.6 TA范围与波束尺寸

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

#### 3.2.7 两步随机接入的时间信息

两步随机接入把前导与上行数据合并为：

\[
\text{MsgA}
=
\text{PRACH}
+
\text{PUSCH payload}.
\]

从 TA 角度，它允许 UE 在更早阶段向网络报告已应用的初始 TA 或相关辅助信息，使网络更早建立对 UE 绝对 UL 时间基准的认知，减少四步随机接入中“Msg1 已对齐但网络仍不知道 UE 已应用多少 TA”的调度不确定性。

两步随机接入不会消除 MsgA 的传播时间，也不会自动解决残余 CFO、PRACH 窗口和竞争冲突；其价值主要在于更早交换时间状态。该内容属于 TR 38.821 对 NTN 随机接入候选方案的研究，不应写成所有 NTN UE 必须使用两步随机接入。

> **原文定位：**TR 38.821 Clause 7.2.1.1.2、Figures 7.2.1.1.2-1 至 7.2.1.1.2-2；相关 TA 框架见 Clause 6.3.4。具体两步随机接入字段和规范行为应以相应 TS 版本为准。

### 3.3 空口联合状态与下游接口

完整的波束足迹、逻辑小区、PCI、BWP、极化资源和波束管理流程由第三份笔记拥有。本篇只描述同步、接入和控制过程需要消费哪些联合空口状态：

\[
\mathbf s_{\mathrm{air}}
=
[
\mathrm{Cell},
\mathrm{Beam},
\mathrm{Satellite},
\mathrm{ReferencePoint},
T_{\mathrm{TA}},
f_D,
K_{\mathrm{offset}},
\mathrm{BWP},
\mathrm{Polarization},
\mathrm{FeederLink}
].
\]

这些状态不必同时改变。service-link switch 可以更换服务卫星或物理波束而保持逻辑小区，也可能连同参考点、Common TA、公共多普勒预补偿、SSB/CSI-RS 和 BWP 一起更新；feeder-link switch 即使不改变 UE 侧覆盖，也可能改变公共时延、频率参考和透明转发路径。切换不能简化成“选择新的最大 RSRP 波束”，还要保证目标状态生效时的时频连续性和调度因果性。

TR 38.821 以 Rel-15/16 波束管理和 BWP 操作为 NTN 基线，但频率复用条件下“一波束一 BWP”“一波束一 CC”及 DL/UL BWP 联动等候选没有收敛。报告还认为在部分 NTN 场景中指示 RHCP/LHCP 极化模式可能有益，但是否支持信令仍留待规范阶段讨论。因此本篇只记录 BWP 和极化变化会影响同步、资源与接入状态；具体空间/频率组织详见[天线波束与覆盖组织](./03_NTN天线波束与覆盖组织_学习笔记.md)。

> **原文定位：**TR 38.821 Clauses 6.2.4-6.2.5、9.1。波束/BWP映射、极化模式信令和 feeder-link switch PHY影响均属于基线之上的后续讨论项，不是报告已收敛的统一方案。

向[链路、系统与多星仿真](./06_NTN链路系统与多星仿真_学习笔记.md)输出时，应将物理到达时间、决策信息、接收机残差、反馈和队列状态分别建模，不能统一折算成固定 SINR 扣减：

| 状态组 | 下游仿真处理 |
|---|---|
| 时频对齐：Common/Differential/Residual TA、\(K_{\mathrm{offset}}\)、原始/公共/残余多普勒 | PRACH到达窗口、上行对齐、捕获范围、调度和残差映射 |
| 反馈控制：CSI/CQI/MCS、RSRP/功控、HARQ/RLC/队列 | 信息老化、链路自适应、发射功率、进程占用和时延统计 |
| 接入辅助：UE位置、星历、Common TA、系统信息、PRACH/HARQ反馈模式 | 选择预补偿或退化搜索路径、检测模型和反馈事件队列 |
| 服务资源：Beam/Satellite/Reference Point、BWP/Polarization/Feeder Link | 多速率状态迁移、资源、链路预算及公共时频状态更新 |
