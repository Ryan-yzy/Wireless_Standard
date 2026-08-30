---
title: "NTN 空口时频与协议适配学习笔记"
date: "2026-08-26"
updated: "2026-08-30"
sources:
  - "3GPP TR 38.811 V15.4.0"
  - "3GPP TR 38.821 V16.2.0"
  - "3GPP TS 38.211 V16.2.0 (Release 16)"
  - "3GPP TS 38.213 V17.13.0"
  - "3GPP TS 38.214 V17.13.0"
  - "3GPP TS 38.215 V17.13.0"
  - "3GPP TS 38.304 V17.9.0"
  - "3GPP TS 38.300 V17.8.0"
  - "3GPP TS 38.321 V17.13.0"
  - "3GPP TS 38.331 V17.13.0"
  - "3GPP TS 38.331 V18.6.0 (Release 18对比)"
---

# NTN 空口时频与协议适配学习笔记

> 本笔记面向非地面网络（Non-Terrestrial Network，NTN）的新空口（New Radio，NR）机制，接收[轨道、时间与链路几何](./02_卫星轨道时间与链路几何_学习笔记.md)、[天线波束与覆盖组织](./03_NTN天线波束与覆盖组织_学习笔记.md)以及[传播损耗与信道模型](./04_NTN传播损耗与信道模型_学习笔记.md)输出的时延、多普勒、波束和局部信道状态，负责把它们映射为空口约束、反馈闭环、接入过程和无线资源管理状态；节点架构、波束足迹和小区组织不在本篇重复展开。

文中把 3GPP 技术报告（Technical Report，TR）作为问题识别、候选方案和评估结果的来源，把技术规范（Technical Specification，TS）作为规范化行为与字段的依据。读到“TR 认为可行”时，不应自动理解为“TS 已要求所有设备实现”；每个关键结论都会在相邻位置标明其研究或规范状态。

| 主章节 | 核心对象 | 判断尺度 | 主要来源 |
|---|---|---|---|
| 1 系统级空口时频与波形约束 | 上行定时、残余频偏、子载波间隔、参考信号、循环前缀、双工与波形 | 波形、符号、时隙和收发切换 | TR 38.811 Clauses 7.3.2、7.3.5-7.3.7，TR 38.821 Clause 6.1.2 |
| 2 测量、反馈与控制闭环 | 时频维护、跨上下行时序、反馈重传、链路自适应与功控 | 测量到执行的闭环时间 | TR 38.821 Clauses 6.2-6.4，TS 38.300/38.321/38.331 Rel-17 NTN机制 |
| 3 下行同步、系统信息与随机接入 | 同步捕获、广播信息、NTN辅助、随机接入与联合空口状态 | 捕获、广播、接入消息和状态转换 | TR 38.821 Clauses 6.3、7.2.1.1，TS 38.300 Clause 16.14，TS 38.331 Rel-17系统信息 |
| 4 无线资源管理测量、预测与移动性执行 | 测量配置、无线/几何事件、条件切换、重选、跟踪区与寻呼 | 测量有效期、几何可用期和目标驻留时间 | TR 38.821 Clause 7.3，TS 38.304 Rel-17，TS 38.331 Rel-17/18 |

全文按以下知识链组织：

\[
\begin{aligned}
&\text{几何与信道输入}
\rightarrow
\text{波形与时序约束}
\rightarrow
\text{测量反馈闭环}
\rightarrow
\text{同步、广播与接入},\\
&\text{带有效期的空口状态}
\rightarrow
\text{无线资源管理测量与预测}
\rightarrow
\text{移动性执行与仿真状态}.
\end{aligned}
\]

为了让四章不是彼此割裂的机制清单，全文采用一个贯穿性的参考场景：一颗非地球静止轨道（Non-Geostationary Orbit，NGSO）卫星相对地面运动，地面用户设备（User Equipment，UE）可以静止；网络为波束定义时间/频率参考点，UE 具有全球导航卫星系统（Global Navigation Satellite System，GNSS）定位能力，并能获得带时间标签的卫星星历和公共辅助信息。该场景只用于串联机制，不限定透明或再生载荷；涉及馈电链路时再单独加入相应状态。

沿着这一场景，可以把全文理解为同一组状态在四个阶段中的不同用途：

| 阶段 | 读者应跟踪的核心变化 | 对应章节 |
|---|---|---|
| 卫星运动改变斜距和径向速度 | 公共/差分时延、原始/残余多普勒怎样影响波形 | 第1章 |
| 网络持续测量并下发控制 | 哪些量可预测、哪些量会老化、哪些结果必须等待 | 第2章 |
| UE从完全未知小区到首次上行 | 怎样先捕获同步，再读取辅助信息，最后发送随机接入前导 | 第3章 |
| 覆盖关系继续随时间变化 | 怎样测量邻区、提前准备目标并执行移动性 | 第4章 |

阅读具体机制前，时间侧的四个量必须分开。它们可能都以微秒或毫秒表示，却回答不同问题：

| 量 | 回答的问题 | 主要消费者 |
|---|---|---|
| 传播时延与传播往返时间 | 信号在空间中真实飞行多久 | 反馈等待、双工保护、端到端时延 |
| 定时提前量（Timing Advance，TA） | UE应把上行波形提前多久发出 | 上行到达对齐、随机接入 |
| 逻辑时序偏移 \(K_{\mathrm{offset}}\) | 下行命令对应哪个未来上行时隙 | 上行数据、重传反馈、信道状态报告与探测参考信号调度 |
| 局部超额时延 | 多径之间相差多久 | 循环前缀（Cyclic Prefix，CP）和频域信道采样 |

对第 \(x\) 个 UE，TA 可以进一步写成：

\[
T_{\mathrm{TA},x}(t)
=
T_{\mathrm{common}}(t)
+
T_{\mathrm{diff},x}(t)
+
T_{\mathrm{corr},x}(t),
\]

三项分别表示相对参考点的公共传播提前量、UE 相对参考点的差分提前量，以及位置、星历、网关时延和测量误差造成的剩余修正。TA 调整实际波形发射时刻；逻辑时序偏移 \(K_{\mathrm{offset}}\) 调整下行（Downlink，DL）与上行（Uplink，UL）时隙的映射；传播往返时间（Round-Trip Time，RTT）则是信号必须真实经过的物理时间：

\[
\boxed{
T_{\mathrm{TA}}
\neq
K_{\mathrm{offset}}
\neq
T_{\mathrm{RTT}}.
}
\]

频率侧采用相同的“物理总量 → 公共预测 → UE差分 → 接收机残差”记账方式：

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

原始多普勒决定初始捕获和补偿范围，公共预补偿去除参考点处的大部分可预测量，UE 相对参考点的差分量决定波束内剩余范围，补偿后的残余频偏 \(\delta f_x\) 决定剩余载波间干扰（Inter-Carrier Interference，ICI）。局部多径的多普勒扩展则是另一种量：它描述不同传播分量的频移差异，决定一个公共频率旋转是否足够。

## 1 系统级空口时频与波形约束

本章讨论不依附于某一个消息流程、持续作用于波形或帧结构的约束。输入是几何笔记给出的绝对时延和多普勒、信道笔记给出的局部时延扩展，以及系统场景给出的星上射频条件；输出是 TA 参考、残余载波频率偏移、参考信号配置边界、CP 适用性、双工选择和波形功率效率。

阅读顺序是从“信号在空间中发生了什么”逐步走向“NR 哪个尺度可能失效”：绝对传播时延先进入上行时间基准，原始多普勒经过补偿后才进入正交频分复用的子载波正交性判断，局部超额时延再决定解调参考信号的频域采样和循环前缀，星上振荡器与功放条件分别进入相位跟踪和波形功率效率，最后用传播 RTT 检查同频上下行的收发保护。这样可以避免把所有 NTN 影响都笼统归因于“大时延、大多普勒”。

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

这条链的关键是比较“处理后的剩余量”与“真正消费它的 NR 容限”。例如，初始同步捕获器面对的是公共预补偿后的剩余搜索范围，循环前缀面对的是相对最早到达径的超额时延，而不是把卫星高度直接换算成一个统一的协议改动。

| NTN输入 | 应比较的量 | 系统级消费者 | 不能直接推出的结论 |
|---|---|---|---|
| 大绝对时延 | 公共 TA、传播 RTT、时隙偏移 | 上行时间基准、同频上下行保护、跨 DL-UL 时序 | 不能据此把 CP 扩展到毫秒量级 |
| 波束内差分时延 | 差分 TA、随机接入前导到达范围 | 多 UE 上行对齐、前导检测 | 不能用卫星高度替代波束内差分距离 |
| 原始多普勒 | 捕获范围和公共预补偿能力 | 下行同步捕获、粗频偏估计 | 不能直接等同于解调后的 ICI |
| 多普勒变化率 | \(\dot f_D T_{\mathrm{mech}}\) | 解调参考信号时间跟踪、预测器 | 时隙内可忽略不表示长闭环内可忽略 |
| 局部超额时延 | 相干带宽、CP 覆盖范围 | 参考信号频域采样、CP | 绝对传播时延不进入局部时延扩展 |
| 星上射频损伤 | 公共相位误差、ICI、峰均功率比与功率回退 | 相位跟踪、波形和有效等效全向辐射功率 | 现有设计在参考场景可用不表示所有载荷均无影响 |

> **原文定位：**TR 38.811 Clauses 7.1-7.3.1，尤其是 Tables 7.1-1、7.2-1 和 7.3.1-1。

### 1.2 上行时间基准与 TA 架构

TA 的目的不是使 UE 和下一代基站（next generation NodeB，gNB）的本地时钟相同，而是使 UE 的上行正交频分复用（Orthogonal Frequency Division Multiplexing，OFDM）符号在 gNB 期望的接收时间窗内到达。设上下行传播近似对称，单程传播时延为 \(\tau_x\)。UE 接收下行时间基准时已经滞后约 \(\tau_x\)，上行发送后又经历约 \(\tau_x\)，因此用于理解的简化关系为：

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

一个仅用于量级理解的例子是：若参考点斜距 \(D_0=1000\,\mathrm{km}\)，某 UE 比参考点远 \(10\,\mathrm{km}\)，在上下行对称且只考虑服务链路时：

\[
T_{\mathrm{common}}
\approx
6.67\,\mathrm{ms},
\qquad
T_{\mathrm{diff},x}
\approx
66.7\,\mu\mathrm{s}.
\]

公共量与差分量相差两个数量级。卫星总体距离主要决定公共 TA，波束内 UE 与参考点的几何差异主要决定接收搜索范围和 UE-specific 修正；二者不能用同一个“卫星链路很长”概括。

对于透明转发载荷，地面 gNB 位于网关（Gateway）之后，公共部分还要包含馈电链路。若 \(D_{01}\) 表示参考点到卫星的服务链路距离，\(D_{02}\) 表示卫星到网关的馈电链路距离，则研究模型可写成：

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
| UE 应用完整 TA（Full TA） | 公共量 + 差分量 + 剩余修正 | gNB 的 DL/UL 帧可保持近似对齐 | UE 本地 UL 时间线相对 DL 时间线大幅提前 |
| UE 仅应用差分 TA（Differential TA） | 差分量 + 剩余修正 | 网络吸收公共 TA | gNB 需要维护 DL/UL 帧时序偏移 |

无论公共时延放在哪一侧，传播路径都没有消失。TA 可以把上行波形的到达时刻对齐，却不能消除随机接入响应、重传反馈或时分双工切换所经历的物理传播时间。

> **原文定位：**TR 38.821 Clause 6.2.1、Figures 6.2.1-1 和 6.2.1-2。两图讨论的是不同 DL/UL 时间架构及其调度影响。

#### 1.2.3 时间参考与馈电链路切换

TA 参考不是只由服务链路决定。透明转发架构中，公共部分还经过馈电链路；若馈电链路从一个 Gateway/卫星连接切换到另一个连接，公共传播时延、频率变换链和网络侧 DL/UL 时间关系都可能变化。UE 即使仍处于同一逻辑小区和地面覆盖区域，也可能需要更新 Common TA、公共频率预补偿和跨 DL-UL 逻辑偏移。

TR 38.821 没有为 feeder-link switch 给出一套收敛的物理层（Physical Layer，PHY）过程，只建议在规范阶段继续讨论其影响。因此本篇只把馈电链路身份和公共时间参考作为联合状态输入，不把某种切换流程写成既定规范；透明/再生载荷及 Gateway 架构详见[系统架构与部署场景](./01_NTN系统架构与部署场景_学习笔记.md)。

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

也可以从信息老化方向理解该量级：若 TA 变化率达到 \(35\,\mu\mathrm{s/s}\)，控制状态经过 \(100\,\mathrm{ms}\) 才生效，仅漂移就会产生约

\[
35\,\mu\mathrm{s/s}\times0.1\,\mathrm{s}
=
3.5\,\mu\mathrm{s}
\]

的定时误差。这个结果已经接近部分正常 CP 的时间尺度，说明更新设计真正关心的是“执行时的剩余误差”，而不是不断重发完整的毫秒级公共 TA。

SCS 增大后 TA 时间粒度更细，但单次调整量变小；同一物理传播时延还对应更多时隙。因此更大 SCS 不能解决大 TA，UE 的发送时间必须允许跨时隙或传输时间间隔边界整体移动，逻辑调度还需由 \(K_{\mathrm{offset}}\) 保证因果性。

> **原文定位：**TR 38.811 Clause 7.3.2.2、Table 7.3.2.2.2-1 及其后关于 extended CP/normal CP 的说明。命令次数为报告在给定最大漂移下的量级分析，不是所有实现的固定发送周期。

### 1.3 残余多普勒、SCS 与 OFDM 正交性

对单个 UE，若多普勒在一个 OFDM 符号内近似恒定，它首先表现为所有子载波的公共载波频率偏移（Carrier Frequency Offset，CFO）。接收机通常先在捕获阶段估计大范围原始频移，再由位置、星历或参考信号去除可预测部分，最后才把残余频偏 \(\delta f\) 交给 OFDM 解调器。第 \(k\) 个子载波在残余频偏下可写成：

\[
y_k(t)
=
X_k e^{j2\pi(k\Delta f+\delta f)t}.
\]

接收机在第 \(m\) 个子载波执行快速傅里叶变换（Fast Fourier Transform，FFT），对应积分：

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

例如 600 km 低地球轨道（Low Earth Orbit，LEO）卫星在 2 GHz 下可能产生约 48 kHz 原始多普勒。若不补偿，对 15 kHz SCS 而言会跨越多个子载波；若星历、位置和频偏估计共同补偿 47.9 kHz，只剩 100 Hz，则：

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

大原始多普勒决定初始同步信号的捕获和粗补偿范围；一个时隙内的变化量决定解调参考信号的时间跟踪；补偿信息经过较长闭环后的老化量则近似为：

\[
\delta f_{\mathrm{stale}}
\approx
\dot f_D T_{\mathrm{age}}.
\]

三者属于不同机制，不能用“多普勒很大”统一代替。

> **原文定位：**TR 38.811 Clauses 5.3.1.3、5.3.4.3-5.3.4.4、7.3.2.3-7.3.2.4，Table 7.3.2.4.1-1。公共、差分、残余和老化频偏的分层属于基于报告条件的工程解释。

### 1.4 局部信道、解调参考信号与循环前缀

卫星高度决定绝对传播时延，地面附近的反射与散射形成局部超额时延。解调参考信号（Demodulation Reference Signal，DM-RS）和循环前缀（Cyclic Prefix，CP）面对的是局部信道，而不是从地面到卫星的全部传播距离。

例如，假设最早路径在 \(100\,\mathrm{ms}\) 后到达，另一条反射路径在 \(100.0002\,\mathrm{ms}\) 后到达。同步与 TA 负责处理共同的约 \(100\,\mathrm{ms}\) 时间原点；CP 需要覆盖的只是两条路径之间的 \(0.2\,\mu\mathrm{s}\) 差值。这个例子直接解释了为何百毫秒卫星传播时延并不要求百毫秒 CP。

| 物理量关系 | 对应机制 | 判断目标 |
|---|---|---|
| \(\dot f_D T_{\mathrm{slot}}\) | DM-RS时间位置 | 一个时隙内能否跟踪符号间变化 |
| \(\Delta f_{\mathrm{pilot}}\) 与 \(B_c\) | DM-RS频域配置 | 导频之间能否可靠插值信道 |
| \(T_{\mathrm{CP}}\) 与 \(\tau_{\mathrm{excess,max}}+T_{\mathrm{timing,res}}\) | 循环前缀 | 是否避免符号间干扰并维持循环卷积条件 |

#### 1.4.1 DM-RS 时间与频域采样

TR 将 1 ms 内的多普勒变化与频率误差量级比较，认为参考场景中的短时变化不足以要求修改 DM-RS 时间位置。该结论不包括尚未完成的粗多普勒补偿，也不包括百毫秒闭环中的信息老化。

频域采样需将导频间隔与相干带宽 \(B_c\) 比较。以一个物理资源块（Physical Resource Block，PRB）和一个码分复用（Code Division Multiplexing，CDM）组为例；表中的资源元素（Resource Element，RE）是时频资源的最小网格单元：

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

表中的 GEO 表示地球静止轨道（Geostationary Earth Orbit），HAPS 表示高空平台系统（High Altitude Platform Station）。对应的报告场景为：

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

这些数值按常用符号给出代表量级；正常 CP 并非每个 OFDM 符号都完全相同，时隙中首符号或周期性边界符号的 CP 会随参数集（numerology）略长。判断符号间干扰时应使用实际符号位置对应的 TS 38.211 参数，而不是把表中代表值当作所有符号的唯一 CP。

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

估计过度配置造成的额外开销。新增 NTN 专用短 CP 虽可降低少量开销，却会增加参数集和实现复杂度，因此报告不预期必须修改 CP 规范。

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

### 1.5 相位噪声、相位跟踪与波形约束

星上本振和频率转换链会叠加相位噪声，有限电源与散热条件又推动功率放大器（Power Amplifier，PA）靠近饱和区工作。两类硬件问题分别作用于相位稳定性和功率效率。

#### 1.5.1 相位误差与参考信号分工

粗残余载波频偏、符号级公共相位旋转和符号内快速相位扰动是三个连续但不同的处理层次。粗频偏应在 FFT 前后由频偏估计与补偿处理；补偿后近似作用于整符号的相位旋转表现为公共相位误差；符号内更快的相位变化才扩散成子载波间干扰。相位跟踪参考信号主要跟踪第二层，不能替代第一层的大范围多普勒捕获。

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

TR 对卫星直播到户（Direct-to-Home，DTH）、甚小口径终端（Very Small Aperture Terminal，VSAT）和典型弯管载荷的相位噪声模板进行评估。在载频不高于约 30 GHz、残余 CFO 不大、带宽不是特别宽且载荷相位噪声接近参考模板的条件下，现有 PT-RS 设计可能足以补偿。该结论不能推广到任意高载频、超大带宽或特殊星上本振。

> **原文定位：**TR 38.811 Clause 7.3.7.1、Annex B。

#### 1.5.2 峰均功率比、功率回退与波形选择

峰均功率比（Peak-to-Average Power Ratio，PAPR）定义为：

\[
\mathrm{PAPR}
=
\frac{\max_n|x[n]|^2}
{\mathbb E\{|x[n]|^2\}}.
\]

循环前缀正交频分复用（CP-OFDM）的多子载波叠加会产生较高瞬时峰值。离散傅里叶变换扩频正交频分复用（DFT-s-OFDM）通过扩频处理降低波形包络起伏。PA 接近饱和时，非线性会造成星座压缩、带内失真、误差矢量幅度恶化、带外泄漏和多载波互调。为保持线性，需要输出功率回退（Output Back-Off，OBO）：

\[
\mathrm{OBO}
=
10\log_{10}
\frac{P_{\mathrm{sat}}}{P_{\mathrm{out}}}.
\]

TR 引用的典型总损失按正交相移键控（Quadrature Phase Shift Keying，QPSK）和 16 阶正交幅度调制（16-Quadrature Amplitude Modulation，16-QAM）分别为：

| 波形 | QPSK | 16-QAM |
|---|---:|---:|
| CP-OFDM | 约6 dB | 约7.6 dB |
| DFT-s-OFDM | 约4 dB | 约6 dB |

差异约 1.6-2 dB。2 dB 输出功率差对应：

\[
10^{-2/10}\approx0.63,
\]

即相同功放下有效输出功率可能只剩约 63%。报告给出的容量影响为 20%-40% 范围，具体取决于系统是功率受限还是带宽受限，不能把功率下降直接等同于固定容量下降。

DFT-s-OFDM 的主要价值是较低 PAPR、较小回退和更高有效等效全向辐射功率（Equivalent Isotropically Radiated Power，EIRP），不表示其在相同 RE、调制阶数和编码率下天然具有较低频谱效率。TR 38.811 在影响研究阶段讨论了 CP-OFDM 的 PAPR 降低并指出上行 DFT-s-OFDM 可能有利；随后 TR 38.821 的 3GPP 无线接入网第一工作组（Radio Access Network Working Group 1，RAN1）最终建议明确，至少对 Release 17 不需要规定 NTN 专用的下行 PAPR 优化。该结论只限定规范化必要性，不表示卫星功放回退、带内失真或带外泄漏可以忽略。TR 38.821 的链路级仿真（Link-Level Simulation，LLS）基线也未纳入高功率放大器（High-Power Amplifier，HPA）非线性，不能把基线排除项反推成硬件无影响。

> **原文定位：**TR 38.811 Clause 7.3.7.2、Figure 7.3.7.2.1-1；TR 38.821 Clause 6.1.2、Table 6.1.2-4、Clause 9.1。总损失示例来自 TR 38.811 引用的 ETSI TR 103 297。

### 1.6 时分双工保护与双工选择

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

因此，FDD 的优势不是缩短了卫星传播 RTT，而是避免把近似 RTT 长度的保护区直接放进同频上下行切换周期。反馈、重传和调度仍然经历真实传播时间，只是不再以同一种方式消耗空口收发保护资源。

> **原文定位：**TR 38.811 Clause 7.3.6.1。这里的 RTT 是纯传播往返时间，不包含检测、调度和编解码时间。

### 1.7 系统级约束接口

本章没有直接决定一次接入或切换是否成功，而是给后续过程提供不能被违反的物理边界：

| 本章输出 | 后续消费者 | 进入后续机制的方式 |
|---|---|---|
| 公共/差分/剩余 TA | 第2.2节维护、第3.2节随机接入 | 决定预测值、到达误差与搜索窗 |
| 原始/公共/残余多普勒及变化率 | 第2.2节维护、第3.1节同步 | 决定捕获范围、预补偿和残余跟踪 |
| SCS、DM-RS与CP适用边界 | 解调与链路配置 | 比较归一化频偏、相干带宽和超额时延 |
| 公共相位误差、ICI、PAPR 与 OBO 状态 | 相位跟踪、波形与功率预算 | 区分接收机残差与发射机有效功率 |
| 传播RTT与双工保护需求 | 第2章反馈与调度 | 决定反馈等待、进程占用和TDD资源损失 |

后续章节可以调整预测、窗口、进程和事件，却不能通过协议配置消除真实传播路径。这是理解第2章“哪些量能预测、哪些结果必须等待”的物理起点。

## 2 测量、反馈与控制闭环

本章拥有“测量何时生成、信息何时到达、决策何时执行”的闭环关系。长 RTT 不会让所有机制以同一种方式退化：传输块重传面对反馈等待，链路自适应面对信道质量老化，功控面对路径损耗估计和命令滞后，TA/频偏维护则可以利用可预测几何进行开环外推。

阅读本章时，可以先对每个控制对象连续问三件事：它能否由位置和星历预测，它在闭环时间内是否会明显变化，以及网络是否必须等待一次真实接收结果。斜距、公共 TA 和公共多普勒大多可预测；局部遮挡和快衰落只能部分预测；肯定确认/否定确认（Acknowledgement/Negative Acknowledgement，ACK/NACK）则是一次传输块的真实译码结果，不能由轨道几何替代。后续机制的差异都来自这三项判断。

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

从测量到真正使用该信息的总年龄为：

\[
T_{\mathrm{age}}
=
t_{\mathrm{execute}}-t_{\mathrm{meas}}.
\]

它不等于某一个单程时延或传播 RTT。测量周期、物理层采样、第三层滤波、事件等待、上行报告、网络排队、调度和命令传播都可能进入 \(T_{\mathrm{age}}\)。因此两个具有相同卫星 RTT 的实现，也可能因为测量和调度策略不同而产生完全不同的信息老化。

滤波和预测可以位于测量、报告或决策侧，但判断信息是否过期时必须保留这四个时间戳。信道状态老化关注测量信道与数据执行信道的差异；TA/频率命令老化关注执行时的几何状态；重传反馈则必须等待某次传输的真实译码结果。它们共享长 RTT 背景，却不是同一种误差。

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
| 传输块成功/失败 | 一次传输块译码结果 | 必须等待真实 ACK/NACK，不能用几何预测代替 |

因此 NTN 的关键不是简单“所有反馈都放慢”，而是将可预测公共量、随机动态量和必须等待的协议事件分开。

> **原文定位：**TR 38.811 Clauses 7.3.2.2-7.3.3.3。预测/滤波/闭环三层拆分属于基于报告条件的工程组织。

### 2.2 TA 与频率补偿的持续维护

连接建立后，几何状态仍随卫星运动变化。适合 NTN 的分工不是在“完全开环”和“完全闭环”之间二选一，而是由预测器给出随时间变化的名义值，再由网络观测修正预测器无法包含的残差。名义值主要来自位置、星历和参考点；残差主要来自位置/星历误差、晶振、馈电链路群时延和测量老化。

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

在该符号约定下，卫星远离 UE 时 \(\dot d>0\)、多普勒为负；若实现使用“卫星指向 UE”的视距（Line of Sight，LOS）向量，公式符号必须相应翻转。该约定与第二篇的 \(f_D=-f_c\dot d/c\) 保持一致。

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
| UE 自主计算 TA | UE 根据位置、星历和必要的馈电链路信息计算完整 TA 或服务链路 TA | UE 计算差分量并持续外推 | 网络根据 UL 到达误差细化残差 |
| 网络指示公共 TA | 网络按波束提供公共 TA | UE 使用随机接入响应/后续 TA 命令处理差分与剩余量 | 维护公共参考、扩展必要的 TA 范围并指示定时漂移 |

单个波束使用一个参考点可简化公共广播与差分计算；多个参考点可能缩小局部差分范围，但其选择和 UE 关联在报告中仍未收敛。负方向 TA 修正、随机接入响应中的 TA 范围扩展以及定时漂移率的具体表示也属于研究阶段问题，不能直接写成所有 NTN Release 的统一配置。

#### 2.2.3 上行频率同步的候选路径

上行频率维护同样存在 UE 预测与网络闭环两条路径：UE 可以根据下行参考信号、位置和星历估计上行多普勒并预补偿；网络也可以根据随机接入前导或其他上行信号估计残余频偏，再向 UE 指示修正值。工程系统可以组合使用两者，即几何预测去除可预测公共量，网络估计校正晶振、位置、星历和馈电链路残差。

TR 38.821 认为可指示 UE 已补偿的频率值，但没有认为必须单独指示 Doppler drift rate。这里的“不需要漂移率指示”不表示漂移不存在，而是 UE 或网络可以通过周期更新、预测或频偏测量处理它。

#### 2.2.4 位置或星历不可用时的退化维护

位置与星历是 UE 自主预补偿的两个不同输入：只有卫星轨迹而不知道 UE 位置，不能唯一求出服务链路斜距和径向速度；只有 UE 位置而没有有效星历，也不能外推卫星状态。任一输入缺失或过期时，UE 可完成的几何预补偿都会下降，需要更多依赖网络广播的 Common TA、接收端时间/频率搜索和闭环修正。

本节负责连接建立后的持续维护；首次接入时如何选择补偿路径、随机接入前导需要覆盖多大的残余不确定性，由第 3.2 节继续展开。

> **原文定位：**TR 38.811 Clause 7.3.2.2；TR 38.821 Clause 6.3.4。UE autonomous TA、network-indicated common TA 和 timing drift rate 均属于报告研究的候选框架，应与后续 TS 的规范机制分开表述。

### 2.3 跨 DL-UL 逻辑时序

大 TA 会使 UE 本地 UL 时间线相对 DL 时间线大幅提前。若 gNB 在 DL 时隙 \(n\) 发送下行控制信息（Downlink Control Information，DCI），要求 UE 在 \(n+K_2\) 的 UL 时隙通过物理上行共享信道（Physical Uplink Shared Channel，PUSCH）发送数据，UE 收到命令时对应 UL 时隙可能已经过去。物理到达对齐正确，但协议调度关系失去因果性。

同样，物理下行共享信道（Physical Downlink Shared Channel，PDSCH）完成后，UE 还要在未来 UL 资源发送混合自动重传请求确认（Hybrid Automatic Repeat reQuest Acknowledgement，HARQ-ACK）。这些跨 DL-UL 关系需要额外逻辑偏移；纯下行内部关系则不一定需要。

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

例如，仅为说明索引关系，若 DCI 位于时隙 100、基线 \(K_2=4\)、附加 \(K_{\mathrm{offset}}=20\)，则 UE 被安排在时隙 124 发送 PUSCH，而不是时隙 104：

\[
n_{\mathrm{PUSCH}}
=100+4+20
=124.
\]

这里的 20 不是规范推荐值；例子的目的只是说明 \(K_{\mathrm{offset}}\) 把动作移动到 UE 收到命令后仍可执行的未来位置，而 TA 负责该 PUSCH 最终到达 gNB 时的波形对齐。

| 过程 | 基准关系或起点 | NTN修正方向 |
|---|---|---|
| DCI→PUSCH | DL 命令触发未来 UL | UL 时隙加偏移 |
| 随机接入响应→Msg3 | Msg2 中 UL Grant | Msg3 PUSCH 加偏移 |
| PDSCH→HARQ-ACK | DL 数据触发 UL 反馈 | ACK/NACK 所在时隙加偏移 |
| 媒体接入控制层控制单元生效 | ACK 后等待处理时间 | 延长并对齐生效时刻 |
| PUSCH上的信道状态报告 | DL DCI 触发未来 UL 报告 | PUSCH 时隙加偏移 |
| 信道状态参考资源 | 从 UL 报告反推过去 DL 资源 | DL 参考时隙减偏移 |
| 非周期探测参考信号 | DL DCI 触发未来 UL 参考信号 | 发送时隙加偏移 |

纯下行的物理下行控制信道（Physical Downlink Control Channel，PDCCH）到 PDSCH 关系只在 DL 时间线上定义，不应无条件增加同类偏移。\(K_{\mathrm{offset}}\) 只保证调度动作发生在 UE 可执行的未来，不能缩短 ACK 的传播、网络处理和重传形成的闭环。

不同过程所需的附加偏移不一定相同，偏移也可能按波束或按小区维护。TR 38.821 在研究阶段没有收敛它应由广播信息推导还是由高层信令给出，也保留了扩展 \(K_1/K_2\) 范围的讨论空间。因此这里用统一的 \(K_{\mathrm{offset}}\) 表示共同原理，而不是断言所有过程共用一个规范字段和值。

表中的媒体接入控制层控制单元（Medium Access Control Control Element，MAC CE）、信道状态信息（Channel State Information，CSI）和探测参考信号（Sounding Reference Signal，SRS）属于不同过程；把它们放在同一表中只是为了比较“跨 DL-UL 时序怎样修正”，不表示三者共享完全相同的规范字段。

> **原文定位：**TR 38.821 Clause 6.2.1、Figures 6.2.1-1 至 6.2.1-2；相关 NR 基线参见 TS 38.214 Clauses 5.2、6.1.2.1.1、6.2.1 和 TS 38.213 Clause 9。TR 中的偏移讨论属于研究方案，不能直接写成所有 Release 16 配置的既定参数。

### 2.4 重传反馈与缓存定时

混合自动重传请求（Hybrid Automatic Repeat reQuest，HARQ）在 PHY 和媒体接入控制（Medium Access Control，MAC）层使用反馈、冗余版本和软合并快速修复误块。NTN 的核心矛盾不是“能否重传”，而是一个 HARQ 进程从首次传输到可安全复用之间被长 RTT 占用。Release 17 因而从三个不同维度适配：增加进程数、允许按下行进程关闭反馈，以及为上行进程配置 Mode A/Mode B。三者不能混写成一个“关闭 HARQ”的开关。

三个机制分别改变“并行度、下行反馈关系、上行进程复用时机”：

| 机制 | 直接改变的对象 | 没有自动改变的对象 |
|---|---|---|
| 增加 HARQ 进程数 | 同时在途的传输上下文数量 | 单次 ACK/NACK 的物理传播时间 |
| 按下行进程关闭反馈 | UE 是否为对应 PDSCH 进程返回 HARQ-ACK | 数据是否已经正确接收 |
| 上行 Mode A/Mode B | 同一 PUSCH 进程何时可以再次调度 | DCI 中新传/重传及冗余版本的含义 |

这张表是理解后续细节的入口：扩展进程数是在“多等几个结果”，关闭反馈或采用更灵活的复用则是在“减少对结果到达的等待依赖”，两类方法的可靠性代价不同。

#### 2.4.1 进程占用与缓存压力

若每个进程必须等待一次完整 HARQ 周期才可复用，为保持连续流水，最小并行进程数可近似理解为：

\[
N_{\mathrm{HARQ,min}}
\approx
\left\lceil
\frac{T_{\mathrm{HARQ}}}{T_{\mathrm{slot}}}
\right\rceil.
\]

TR 在 15 kHz SCS、1 ms 时隙的参考条件下给出：

表中的 MEO、GEO 和 HEO 分别表示中地球轨道（Medium Earth Orbit）、地球静止轨道和高椭圆轨道（Highly Elliptical Orbit）。

| 场景 | 参考HARQ周期 | 参考最小进程数 | 报告判断 |
|---|---:|---:|---|
| 地面NR | 16 ms | 16 | Rel-15参考能力可行 |
| LEO | 50 ms | 50 | 扩展HARQ后可行 |
| MEO | 180 ms | 180 | 需研究传输块大小、调制编码配置和实现能力 |
| GEO/HEO | 600 ms | 600 | 简单线性扩展压力很大 |

该表解释研究阶段为何不能只靠增加进程数；它不是 Release 17 的配置表。Release 17 将 PDSCH、PUSCH 可配置进程数扩展到 32，但 32 个进程仍不足以按“一时隙一个新传输块”的方式覆盖 MEO/GEO 的完整 RTT。SCS 增大只会让同一 RTT 内包含更多时隙，不会缩短物理反馈时间。进程数扩展还会增加软缓存、并行编解码、冗余版本和调度上下文。仅用于理解的原始在途数据下界为：

\[
B_{\mathrm{flight}}
\gtrsim
R T_{\mathrm{ACK}}.
\]

若 \(R=100\,\mathrm{Mbit/s}\)、\(T_{\mathrm{ACK}}=0.6\,\mathrm{s}\)，原始在途数据约为 \(60\,\mathrm{Mbit}=7.5\,\mathrm{MB}\)。实际 HARQ 软缓存还与编码块、量化精度、多用户并发和冗余版本有关。

#### 2.4.2 下行HARQ反馈启停

下行方向由 gNB 发送 PDSCH，UE 在上行返回 HARQ-ACK。Release 17 的 `downlinkHARQ-FeedbackDisabled-r17` 是 32 位位图，按 HARQ 进程标识决定是否关闭对应进程的下行 HARQ 反馈；它不是整个 UE 只有一个总开关。反馈启用时，gNB 可以依据 ACK/NACK 决定是否发送增量冗余并保留软合并关系。反馈关闭时，同一 HARQ 进程标识可以在一个 HARQ RTT 尚未结束前再次调度，从而避免进程停顿，但 gNB 不再拥有该次传输的即时 UE 译码结果。

因此“反馈关闭”至少带来三个边界：

- 进程标识的提前复用不等于前一传输块已经正确接收；
- 不能再假设每次重传都由对应 NACK 精确触发，可靠性需更多依靠编码、重复、调度策略和上层自动重传；
- 半静态调度（Semi-Persistent Scheduling，SPS）使用哪些启用/关闭反馈的进程，由网络实现保证配置一致性。

> **原文定位：**TS 38.300 Clause 16.14.2.1；TS 38.331 `PDSCH-ServingCellConfig` 与 `DownlinkHARQ-FeedbackDisabled-r17`（Clause 6.3.2）；TS 38.321 Clauses 5.3.2.2、5.7。下行反馈启停是 Release 17 规范机制，不再只是 TR 38.821 的候选方案。

#### 2.4.3 上行HARQ Mode A与Mode B

上行方向由 UE 发送 PUSCH，gNB 通过后续 DCI 的新数据指示（New Data Indicator，NDI）、冗余版本和资源分配控制新传或重传。Release 17 的 `uplinkHARQ-mode-r17` 同样是 32 位位图：取值 1 表示对应进程采用 HARQ Mode A，取值 0 表示 Mode B。该配置作用于上行 HARQ 进程标识，并不表示 UE 选择两种接收机模式。

表中的非连续接收（Discontinuous Reception，DRX）定时器决定 UE 在何时进入期待重传授权的监听阶段：

| 对比项 | HARQ Mode A | HARQ Mode B |
|---|---|---|
| 同一进程的复用边界 | 保留一个 HARQ RTT 的等待边界 | 允许在一个 HARQ RTT 尚未结束前再次调度 |
| DRX中的重传等待 | 使用 `HARQ-RTT-TimerUL-NTN`，在基础 UL RTT timer 上加入最新可用 UE-gNB RTT | 使用基线 `drx-HARQ-RTT-TimerUL`，不增加 NTN RTT 等待 |
| 主要价值 | 保持“等待上次结果后再期待重传授权”的清晰时序 | 避免长 RTT 导致同一进程长期停顿，提高调度自由度 |
| 主要代价 | 需要更多并行进程和缓存，否则容易停顿 | 早期复用时网络必须避免混淆新传、重传和软缓存状态 |

Mode A 的 MAC 行为可以概括为：PUSCH 发送后，UE 启动带 NTN RTT 的等待定时器；定时器到期后才进入期待 UL 重传授权的窗口。Mode B 允许网络更早调度同一进程标识，但“允许提前调度”并不自动等于盲重传，也不等于关闭所有上行可靠性机制；具体是新传还是重传仍由 DCI、NDI、冗余版本和网络调度状态共同决定。

对配置授权（Configured Grant，CG），规范把进程选择与 HARQ mode 关联；网络实现应保证供某一 CG 使用的进程具有合适且一致的 Mode A/Mode B 配置。若 `nrofHARQ-ProcessesForPUSCH-r17` 未配置为 32，UE 使用 16 个 PUSCH HARQ 进程；Mode A/B 位图中未配置进程对应的 bit 被忽略。

> **原文定位：**TS 38.300 Clause 16.14.2.1；TS 38.331 `PUSCH-ServingCellConfig` 与 `UplinkHARQ-mode-r17`（Clause 6.3.2）；TS 38.321 Clauses 5.4.3.1、5.7。TS 38.300 明确 Mode B 允许在一次 HARQ RTT 结束前再次调度同一进程。

#### 2.4.4 无线链路控制层兜底与研究到规范边界

无线链路控制（Radio Link Control，RLC）的自动重传请求（Automatic Repeat reQuest，ARQ）位于 HARQ 之上，以状态报告和协议数据单元（Protocol Data Unit，PDU）重传恢复剩余错误。两层机制的恢复粒度、反馈方式和缓存对象不同：

| 机制 | 层次 | 接收端处理 | 长RTT下主要压力 |
|---|---|---|---|
| HARQ | PHY/MAC | 软合并和传输块反馈 | 进程数、软缓存、ACK/NACK关联 |
| RLC ARQ | RLC确认模式 | 状态报告和PDU重传 | 窗口、定时器、状态报告和缓存 |

限制或关闭部分 HARQ 反馈并不表示误块消失，而是将更多剩余错误交给 RLC ARQ、更强编码、重复或应用容错。RLC 的恢复需要跨越更长的状态报告和重传闭环，因此不能认为它能够无代价替代 HARQ。

TR 38.821 研究阶段给出了“增加进程/缓存”和“限制或关闭反馈并更多依靠 RLC”两条总体路线，还记录了 DCI 指示、上行控制信息（Uplink Control Information，UCI）中断报告、超过 8 个时隙的聚合、时间交织聚合和新的调制编码表等候选。后续 Release 17 规范最终形成了 32 个 HARQ 进程、下行按进程启停反馈和上行按进程配置 Mode A/B 等机制；这不意味着 TR 中列出的所有候选都被采纳。阅读时应把“TR 候选全集”和“TS 已落地子集”分开。

> **原文定位：**TR 38.811 Clauses 7.3.3.1-7.3.3.2，Figures 7.3.3.1-1、7.3.3.1.1-1，Table 7.3.3.1.1-1；TR 38.821 Clauses 6.4.1-6.4.2、9.1；TS 38.300 Clause 16.14.2.1。TR 表中的参考进程需求是研究评估；Release 17 规范机制应以 TS 38.300、TS 38.321 和 TS 38.331 为准。

### 2.5 信道状态预测与链路自适应

自适应编码调制（Adaptive Coding and Modulation，ACM）使用信道状态信息（Channel State Information，CSI）和信道质量指示（Channel Quality Indicator，CQI）支持调度与调制编码方案（Modulation and Coding Scheme，MCS）选择。长 RTT 下真正使用的不是测量时刻信道，而是数据到达接收端时的信道。

表中的视距（Line of Sight，LOS）和非视距（Non-Line of Sight，NLOS）描述直达径是否可用；NTN 抽头延迟线（Tapped Delay Line，TDL）字母后缀表示相应的标准化信道模型。

| 场景 | 主要变化 | 闭环适用性 |
|---|---|---|
| GEO Ka | 雨衰通常较慢 | ACM可工作，但需滞回避免MCS振荡 |
| GEO S | 遮挡和局部多径可能较快 | 难以逐衰落跟踪，需要更大裕量 |
| LEO | 斜距引起的大尺度损耗可预测 | 可预测和跟踪大尺度变化，仍难跟踪快速多径 |
| 手持终端 | 姿态、阴影和局部遮挡 | 需要预测、滤波和保守MCS |

TR 38.821 的链路级结果说明，CSI 老化不是只由反馈时延决定。在 3 km/h、10 dB 信噪比（Signal-to-Noise Ratio，SNR）的参考评估中，NTN-TDL-C（LOS）把反馈时延从约 6 ms 增加到 40-46 ms 时，吞吐量或频谱效率损失约为 10%-12%；NTN-TDL-A（NLOS）的对应损失约为 28%-38%。另一个评估在 30 km/h 条件下没有观察到反馈时延从 6 ms 增加到 201 ms 的明显额外损失。不同信道、速度和实现得出的趋势并不单调一致，因此不能仅按 RTT 给 CSI 固定扣减。

报告评估的长期信道平均在 3 km/h、理想信道估计条件下没有改善吞吐量；预测式 CSI 在部分 NLOS 评估中约有 10% 增益，但预测可以仅作为实现算法还是需要新增 UE 报告参数没有共识。由此，Rel-15 CSI 框架至少可作为 LOS NTN 链路自适应基线，进一步优化仍未收敛。

> **原文定位：**TR 38.821 Clause 6.2.3。百分比是特定 NTN-TDL、SNR、速度和反馈时延下的来源结果，不是通用 NTN 性能保证。

工程实现可使用下面的保守映射，其中误块率（Block Error Rate，BLER）是链路自适应的目标可靠性指标：

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
M_{\mathrm{feedback\ age}}
+
M_{\mathrm{loop\ uncertainty}}.
\]

例如，若预测器给出数据实际使用时刻的信干噪比（Signal-to-Interference-plus-Noise Ratio，SINR）为 \(8\,\mathrm{dB}\)，实现为估计误差、反馈老化和闭环不确定性合计保留 \(2\,\mathrm{dB}\) 裕量，则 \(\gamma_{\mathrm{design}}=6\,\mathrm{dB}\)。调度器选择“目标 BLER 门限不高于 6 dB 的最高 MCS”，而不是直接按测量时刻的 8 dB 选择。这里没有指定某个 MCS 索引，因为具体门限取决于链路曲线、接收机和目标 BLER；例子只展示裕量怎样进入决策。

该分解不是 TR 或 TS 直接定义的协议参数。3GPP 规定 MCS 索引与调制阶数、目标码率及相应信令；下行 PDSCH 和通常的上行 PUSCH MCS 由 gNB 调度器选择，具体预测器、CQI 到 MCS 映射、滞回、外环链路自适应和 NTN 裕量属于实现算法。

> **原文定位：**TR 38.811 Clause 7.3.3.3；TR 38.821 Clause 6.2.3；MCS 和链路自适应基线参见 TS 38.214 Clauses 5.1.3、5.2.2 和 6.1.4。TR 没有规定上述工程分解的固定数值。

### 2.6 参考信号功率滤波与上行功率控制

同一组同步信号块和信道状态信息参考信号测量可以服务不同闭环。本节关心它怎样形成路径损耗估计并进入上行功率；第 4 章关心它怎样形成小区量值并触发移动性。共享测量源不表示两个消费者使用相同门限、时间尺度或决策逻辑。

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

TS 38.331 定义的第三层（Layer 3，L3）一阶指数滤波为：

\[
F_i
=(1-a)F_{i-1}+aM_i,
\qquad
a=\frac{1}{2^{k/4}},
\]

其中 \(M_i\) 是最新测量，\(F_i\) 是滤波输出，\(k\) 是 `filterCoefficient`。例如 \(F_{i-1}=-90\,\mathrm{dBm}\)、\(M_i=-98\,\mathrm{dBm}\)：\(k=4\) 时 \(a=1/2\)，得到 \(-94\,\mathrm{dBm}\)；\(k=8\) 时 \(a=1/4\)，得到 \(-92\,\mathrm{dBm}\)。后者更平滑，却会在真实功率快速下降时暂时低估路径损耗。

从控制角度看，较强平滑可以减少阴影或测量噪声造成的功率振荡，却会使功控在链路快速变差时继续使用偏乐观的路径损耗；同一老化也可能让移动性事件触发偏晚。因此滤波系数不是单纯“越平滑越好”，而要与测量周期、卫星/波束运动速度和闭环执行时间共同选择。

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

### 2.7 闭环状态接口

第1章给出物理变化速度，本章把它转换成带时间戳的控制状态。向接入、移动性和仿真模块输出时，至少需要保留：

| 闭环状态 | 必需的附加信息 | 主要消费者 |
|---|---|---|
| TA与频率预测/残差 | 参考时刻、预测有效期和最后一次网络修正 | SSB后跟踪、随机接入与连接态维护 |
| \(K_{\mathrm{offset}}\) 等逻辑时序 | 适用过程、参数集与生效边界 | PUSCH、HARQ-ACK、CSI 和 SRS |
| HARQ进程与反馈模式 | 进程标识、模式、等待定时器和软缓存状态 | MAC调度、RLC恢复与时延统计 |
| CSI/CQI/MCS状态 | 测量时刻、预测时刻、裕量和目标BLER | 链路自适应与吞吐统计 |
| RSRP滤波与功控状态 | 滤波系数、路径损耗参考和闭环累计量 | PUSCH功率与第4章移动性测量 |

这组接口的共同特征是“数值必须与时间戳一起传递”。只保存最新 TA、CQI 或 RSRP 数值而丢失其生成时刻，会使后续模块无法判断它代表当前状态还是历史状态。

## 3 下行同步、系统信息与随机接入

本章按照 UE 建立接入认知和执行过程的先后关系组织：UE 首先接收同步信号块（Synchronization Signal Block，SSB），使用其中的主同步信号（Primary Synchronization Signal，PSS）和辅同步信号（Secondary Synchronization Signal，SSS）取得初始时频基准与物理小区标识（Physical Cell Identity，PCI），再解调物理广播信道（Physical Broadcast Channel，PBCH）并读取后续系统信息；获得物理随机接入信道（Physical Random Access Channel，PRACH）资源及可用 NTN 辅助信息后，UE 才发送 Msg1 并进入随机接入。

这一过程可以分成“观察、理解、行动”三层。PSS/SSS 和 PBCH 让 UE 观察并识别一个可解调的小区；主信息块（Master Information Block，MIB）、系统信息块1（System Information Block Type 1，SIB1）和系统信息块19（System Information Block Type 19，SIB19）让 UE 理解该小区怎样调度系统信息、是否允许 NTN 接入以及应采用什么时频辅助；PRACH、随机接入响应（Random Access Response，RAR）和 Msg3 才是 UE 与网络之间的首次双向接入行动。前一层的稳定输出是后一层的输入，不能跳过系统信息直接把第一次 SSB 捕获等同于“已经具备上行发送条件”。

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

PSS/SSS 并不是在 SSB 之前单独发送的另一组信号；上述顺序表示 UE 对同一个 SSB 的处理层次，以及 SSB 输出如何成为随机接入输入。

### 3.1 PSS/SSS、SSB与下行初始同步

同步信号块（Synchronization Signal Block，SSB）由主同步信号（Primary Synchronization Signal，PSS）、辅同步信号（Secondary Synchronization Signal，SSS）、物理广播信道（Physical Broadcast Channel，PBCH）及 PBCH 解调参考信号组成。它不是单纯的“同步序列”，而是把盲捕获逐步转换成系统信息读取入口。

#### 3.1.1 SSB内部处理链

UE 对同一个 SSB 的处理可以分成四层：

| 组成 | UE主要处理 | 稳定输出 |
|---|---|---|
| PSS | 候选时间位置、粗频偏和 \(N_{ID}^{(2)}\) 检测 | 粗符号边界与小区标识的一部分 |
| SSS | 细化时间/频率并检测 \(N_{ID}^{(1)}\) | 完整物理小区标识 PCI |
| PBCH DM-RS | PBCH信道估计和候选SSB假设校验 | PBCH相干解调条件 |
| PBCH/MIB | 译码系统帧号部分信息、公共 SCS、`pdcch-ConfigSIB1` 等 | 获取 SIB1 的控制资源集0（Control Resource Set 0，CORESET#0）与搜索空间0入口 |

PSS/SSS 完成的是“找到并识别这个 SSB”；PBCH/MIB 完成的是“从这个 SSB 进入小区广播配置”。因此，PSS/SSS 不在 SSB 之前独立存在，MIB 也不直接携带全部随机接入和 NTN 辅助参数。

> **原文定位：**TS 38.211 Clauses 7.3.3、7.4.2-7.4.3；TS 38.331 `MIB` 信息元素及 Clause 5.2.2.4.1。

#### 3.1.2 SSB多普勒公共预补偿与残余搜索

TR 使用地面 UE 约 5 ppm 初始频偏鲁棒性作为比较基线：

\[
5\,\mathrm{ppm}\times2\,\mathrm{GHz}
=10\,\mathrm{kHz},
\qquad
5\,\mathrm{ppm}\times20\,\mathrm{GHz}
=100\,\mathrm{kHz}.
\]

600 km LEO 的参考最大原始多普勒在 2 GHz 和 20 GHz 分别约为 ±48 kHz 和 ±480 kHz。若 UE 在完全未知频偏下直接搜索 PSS/SSS，地面 NR 的单次捕获范围可能不足。但一个 SSB 面向整个波束广播，网络不能为每个 UE 分别生成不同频移，只能围绕波束中心或上行时间同步参考点执行公共频率预补偿。

设网络针对参考点 \(r\) 给下行 SSB 施加频率预移 \(f_{\mathrm{pre}}^{\mathrm{DL}}\approx-\hat f_{D,r}^{\mathrm{DL}}\)，UE \(x\) 看到的同步残余可写成：

\[
\delta f_{\mathrm{SSB},x}
=
f_{D,x}^{\mathrm{DL}}
-\hat f_{D,r}^{\mathrm{DL}}
+e_{\mathrm{feeder}}
+e_{\mathrm{osc}},
\]

其中第一、二项之差是 UE 相对参考点的差分服务链路多普勒，\(e_{\mathrm{feeder}}\) 汇总馈电链路和转发器频率误差残差，\(e_{\mathrm{osc}}\) 表示 UE 晶振误差。UE 的 PSS/SSS 搜索器只需要覆盖这一残余，而不是完整原始卫星多普勒。

因此 SSB 多普勒处理有两个不同阶段：首次捕获前，网络侧尽可能去除公共量，UE 搜索残余；捕获后，UE 可利用 PSS/SSS、PBCH DM-RS 和后续参考信号继续跟踪频偏。SIB19 中的星历尚未获得，不能倒过来用于第一次 SSB 捕获。TR 38.821 的评估观察到：GEO 以及采用波束级公共频移预补偿的 LEO 可以复用 Rel-15 SSB；LEO 若不做公共预补偿，需要增加 UE 搜索复杂度，但未识别出修改 SSB 波形本身的必要性。

> **原文定位：**TR 38.811 Clause 7.3.2.3；TR 38.821 Clause 6.3.2。报告中约 13,000 km 的高度判断来自特定 5 ppm 与最大几何多普勒比较，不是通用部署门限。

#### 3.1.3 SSB时延公共处理与下行时间基准

传播时延对 SSB 的影响与多普勒不同。一个公共时延只会把整个 SSB 和后续下行帧整体向后平移；UE 仍可以在到达时刻检测 PSS/SSS，并把该到达位置建立为本地 DL 时间基准。真正困难的是不同 UE、不同波束和 DL/UL 参考点之间的时间关系，而不是 SSB 内部四个符号被同一个公共时延“拉长”。

若网络希望下行帧在参考点 \(r\) 对齐，工程上等价于将包含 SSB 的整个下行波形提前 \(\hat\tau_r\) 发出。UE \(x\) 的接收时间残余为：

\[
\delta\tau_{\mathrm{SSB},x}
=
\tau_x-\hat\tau_r.
\]

这里的 \(\hat\tau_r\) 是网络侧公共时延处理，\(\tau_x-\hat\tau_r\) 是 UE 相对参考点的差分时延。它不是一个写进 PSS、SSS 或 PBCH 的“SSB TA 字段”，也没有消除真实传播时延。Release 17 规定 DL 与 UL 在上行时间同步参考点按 \(N_{\mathrm{TA,offset}}\) 对齐；Common TA、\(K_{\mathrm{offset}}\) 和 \(K_{\mathrm{mac}}\) 随后分别处理公共 RTT、跨 DL-UL 调度关系和 gNB 侧 DL/UL 帧不对齐。

从 UE 视角看，首次 SSB 只给出“信号何时到达我这里”；在获得 SIB19 的 Common TA、星历和 epoch time 后，UE 才能把这个接收基准转换成相对于上行时间同步参考点的发射提前量。由此必须区分：

\[
\boxed{
\text{SSB到达时间捕获}
\neq
\text{网络侧公共DL时间处理}
\neq
\text{UE上行TA预补偿}.
}
\]

> **原文定位：**TS 38.300 Clauses 16.14.2.1-16.14.2.2；TR 38.821 Clauses 6.2.1、6.3.2。把参考点对齐理解为整个 DL 波形的公共提前发送是工程等效解释，不是新增的 SSB 专用信令。

#### 3.1.4 广播信息与 NTN 辅助获取链

SSB 后的系统信息链为：

\[
\text{PBCH/MIB}
\rightarrow
\text{CORESET\#0与SearchSpace\#0}
\rightarrow
\text{SIB1}
\rightarrow
\text{SIB19调度}
\rightarrow
\text{NTN辅助信息}.
\]

MIB 提供 UE 读取 SIB1 所需的基础入口。具备 NTN 能力的 UE 读取 SIB1 后，根据 `cellBarredNTN-r17` 判断该小区是否允许 NTN 接入；该字段缺省或为 `barred` 时，UE 不应把该小区作为可接入 NTN 小区。SIB1 的系统信息（System Information，SI）调度配置再给出其他 SI 消息的周期、窗口和 SIB 映射。SIB19 被调度时，对应 `si-BroadcastStatus` 设为 `broadcasting`，因此它属于 SSB 之后的无线资源控制（Radio Resource Control，RRC）系统信息，不属于 PBCH 载荷。

SIB19 的职责是承载卫星接入辅助信息。它的主要内容及消费者如下：

| SIB19字段或子字段 | 物理/协议含义 | 主要消费者 |
|---|---|---|
| `ephemerisInfo-r17` | 卫星位置速度状态向量或轨道参数 | 服务链路时延、多普勒与邻区测量预测 |
| `epochTime-r17` | 星历与Common TA参数的共同参考时刻 | 把广播参数外推到当前时刻 |
| `ta-Common-r17` | 网络控制的公共TA，可含网络认为必要的时间偏移 | UE到上行同步参考点的总TA计算 |
| `ta-CommonDrift-r17`、`ta-CommonDriftVariant-r17` | Common TA的一阶漂移及漂移变化 | 在两次广播之间外推公共TA |
| `ntn-UlSyncValidityDuration-r17` | 星历和Common TA辅助信息从epoch time起的有效期 | 有效性判断与SIB19重获取 |
| `cellSpecificKoffset-r17` | NTN修改时序关系使用的调度偏移 | PUSCH、HARQ-ACK、CSI等跨DL-UL调度 |
| `kmac-r17` | gNB侧DL/UL帧未对齐时的附加偏移 | MAC CE生效、RAR/MsgB窗口和RTT估计 |
| `ta-Report-r17` | 是否启用随机接入/连接态TA报告 | 网络侧TA维护 |
| `ntn-PolarizationDL/UL-r17` | 服务链路上下行极化指示 | UE射频/天线极化选择 |
| `t-Service-r17` | 准地球固定小区停止服务当前区域的时间 | 时间条件测量与移动性 |
| `referenceLocation-r17`、`distanceThresh-r17` | 服务小区参考位置与距离门限 | Idle/Inactive位置条件测量 |
| `ntn-NeighCellConfigList-r17` | 邻区频点、PCI及其NTN辅助配置 | NTN邻区测量和切换准备 |

为了阅读这张字段表，可以把 SIB19 看成四组状态，而不是十几个彼此无关的参数：

\[
\begin{aligned}
&\text{星历+epoch time}
\rightarrow
\text{当前几何},\\
&\text{Common TA+drift+有效期}
\rightarrow
\text{上行时基},\\
&K_{\mathrm{offset}}+K_{\mathrm{mac}}
\rightarrow
\text{跨DL-UL协议时序},\\
&t\text{-Service+参考位置+邻区配置}
\rightarrow
\text{移动性准备}.
\end{aligned}
\]

`epochTime` 不是普通的“消息接收时间”：它把星历和 Common TA 参数绑定到一个系统帧号（System Frame Number，SFN）/子帧参考点。处于 `RRC_CONNECTED` 状态的 UE 收到 SIB19 后，从 epoch time 指示的子帧启动或重启定时器 T430，并应在有效期结束前重获取 SIB19。`ntn-UlSyncValidityDuration` 也不是 SIB19 的广播周期，而是 UE 可以继续使用该组辅助信息的最长时间。星历、epoch time、Common TA 与有效期必须作为一组消费，不能只缓存星历坐标而忽略其时间标签。

Release 17 的接入假设比 TR 38.821 的候选讨论更明确：NTN UE 需要有效 GNSS 位置、有效星历和 Common TA；UE 根据这些量计算 UE 到上行时间同步参考点的 RTT，并对上行 TA 以及服务链路瞬时多普勒自主预补偿。馈电链路多普勒和转发器频率误差由网络实现处理。若上述必要量失效，UE 不应继续发射，直至重新获得有效信息。

把字段放回一次实际接入流程中，UE 会先通过 SSB、MIB 和 SIB1 找到 SIB19；再以 `epochTime-r17` 为时间原点，把星历和 Common TA 外推到当前发送时刻；随后结合自身位置计算服务链路传播时延和瞬时多普勒；最后检查位置、星历、Common TA 与 `ntn-UlSyncValidityDuration-r17` 均有效，才形成 PRACH 的上行时频预补偿。这个顺序也说明：SIB19 不是第一次 SSB 捕获的前提，却是 Release 17 NTN UE 首次上行发送的重要前提。

> **原文定位：**TS 38.331 Clauses 5.2.2.3、5.2.2.4.1-5.2.2.4.2、5.2.2.4.21，`SIB1`、`SIB19-r17`、`NTN-Config-r17` 与 `SI-SchedulingInfo`（Clause 6.3.2）；TS 38.300 Clauses 16.14.2.2、16.14.3.1、16.14.3.3。

#### 3.1.5 同步输出与后续消费者

SSB-RSRP 可进入波束测量、L3 滤波和路径损耗估计。其消费者分属不同笔记和章节：波束候选与小区组织由[天线波束与覆盖组织](./03_NTN天线波束与覆盖组织_学习笔记.md)负责；RSRP 滤波和 PUSCH 功控由第 2.6 节负责；SIB19 提供的星历和 Common TA 与 SSB 到随机接入时机的关联共同进入第 3.2 节；SSB/CSI-RS 到小区量值、事件判决和移动性执行的完整链条由第 4 章负责。

因此从首次 SSB 到 NTN 接入准备完成的稳定输出为：

\[
\boxed{
\text{下行时频基准}
+
\text{小区/SSB身份}
+
\text{SIB1调度入口}
+
\text{有效NTN辅助状态}.
}
\]

### 3.2 NTN 随机接入与初始时频对齐

随机接入把上游的传播时延、差分时延、残余 CFO 和时间参考具体化为 Msg1 检测、Msg2 监测、Msg3 调度和上行对齐问题。其核心不是把一个 TA 字段无限扩大，而是重组初始时延预测、网络细化和跨 DL-UL 调度。

与地面随机接入最直观的差别是：地面 UE 可以先发送 Msg1，再让网络从 Msg1 到达时间估计初始 TA；NTN UE 若完全不做预处理，Msg1 自身就可能落到传统接收窗之外。因此 NTN 将一部分粗时频对齐移到 Msg1 之前，RAR 中的 TA 再从“给出全部初始对齐”转为“细化 UE 或公共辅助已经完成的估计”。

#### 3.2.1 辅助能力与补偿路径

UE 能否在 Msg1 之前完成几何时频预补偿，取决于“UE 位置”和“卫星星历”两类信息是否同时可用且足够新鲜，并需要 SIB19 提供的 Common TA、epoch time 和有效期共同建立上行同步状态。为理解 TR 38.821 研究过的辅助条件，可以先区分四种输入状态：

| UE位置 | 卫星星历 | Msg1前可用处理 | PRACH侧剩余压力 |
|---|---|---|---|
| 可用 | 可用 | UE可预测服务链路 TA 和上行多普勒，并结合网络公共量预补偿 | 主要覆盖位置、星历、晶振及馈电链路残差 |
| 可用 | 不可用或过期 | 不能可靠外推卫星距离与径向速度 | 需要网络提供更多公共量并扩大时频搜索 |
| 不可用 | 可用 | 知道轨道但不能唯一确定 UE 到卫星的斜距和径向速度 | Common TA只能去除公共部分，仍需覆盖 UE 特定差异 |
| 不可用 | 不可用 | 无法执行完整几何预补偿 | 主要依赖网络广播、接收端搜索及后续闭环修正 |

该表是研究问题的机制分类，不是 Release 17 定义的四种 UE 工作模式。信息“存在”也不等于“有效”：位置误差、星历误差、epoch time、Common TA 和信息老化必须共同决定预补偿残差。对于 Release 17 规范化 NR NTN 接入，UE 是 GNSS-capable，并应在有效 GNSS 位置、星历和 Common TA 可用后才发射；因此表中后三行用于理解研究阶段的退化压力和接收机需求，不能解读为 Release 17 UE 可以在这些状态下继续发送 Msg1。连接后的持续维护由第 2.2 节负责。

> **原文定位：**TR 38.821 Clauses 6.3.2-6.3.3；TS 38.300 Clauses 16.14.1、16.14.2.2；TS 38.331 `SIB19-r17` 与 `NTN-Config-r17`（Clause 6.3.2）。

#### 3.2.2 四步随机接入与初始 TA

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

地面网络中，UE 可以先发送 Msg1，gNB 根据到达时间估计 TA，再在 RAR 中发送 TA 命令，使 Msg3 对齐。NTN 中 Msg1 在首次网络 TA 命令到来之前已经需要跨越几十至数百毫秒的传播路径，传统 RAR TA 范围也无法承担完整卫星绝对时延。

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

#### 3.2.3 PRACH 检测窗口与随机接入时机模糊

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

若所有 UE 使用同一个随机接入信道时机（Random Access Channel Occasion，RO），接收窗口至少需要覆盖：

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

一个简化时间例子可以直接看到这种模糊。假设 \(RO_1\) 在 \(0\,\mathrm{ms}\) 发送、\(RO_2\) 在 \(1\,\mathrm{ms}\) 发送，而接收机在 \(1.5\,\mathrm{ms}\) 观察到相同前导相关峰。它可能来自“\(RO_1\)+\(1.5\,\mathrm{ms}\) 传播/残余时延”，也可能来自“\(RO_2\)+\(0.5\,\mathrm{ms}\) 传播/残余时延”。如果没有额外时频结构或更小的不确定范围，仅凭峰位置无法决定应回推哪个发送 RO，两个解释对应的 TA 还相差 \(1\,\mathrm{ms}\)。该数值只用于解释机制，不是规范化 RO 配置。

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

#### 3.2.4 无预补偿时的 PRACH 候选

TR 38.821 Clause 6.3.3 根据 UE 是否完成时频预补偿给出不同判断。UE 具有足够准确的位置和星历、能够在 Msg1 前预补偿时间与频率偏移的情况下，Release 15 PRACH 格式和序列可以复用，是否通过重复或更大 SCS 改善覆盖可留到规范阶段讨论。若 UE 不进行预补偿，报告记录了四类增强候选。表中的 Zadoff-Chu（ZC）序列是一类具有良好相关性的前导序列，根序列与循环移位共同决定可区分的前导结构：

| 候选 | 序列或波形方向 | 主要意图 | 研究状态 |
|---|---|---|---|
| 单个 Zadoff-Chu 序列 | 更大 SCS 和/或重复，CP 与 \(N_{\mathrm{CS}}\) 待确定 | 提高大频偏和低 SNR 下的检测能力 | 候选，未收敛 |
| 多个 Zadoff-Chu 序列 | 使用不同根序列的多根 ZC | 增加时频不确定性下的可观测结构 | 候选，未收敛 |
| Gold 序列或 m 序列 | 配合处理或变换预编码 | 改变大时频偏移下的相关与估计特性 | 候选，未收敛 |
| ZC 加扰码 | 在 ZC 上增加扰码结构 | 改善检测或模糊区分能力 | 候选，未收敛 |

TR 38.821 没有完成这些候选的筛选，也没有把其中任一种写成 NTN 统一 PRACH。因此本节只保留候选族及其问题背景；不同序列、根序列、循环移位、重复和接收机算法的波形级比较应在后续 RAN1 提案专题或[链路、系统与多星仿真](./06_NTN链路系统与多星仿真_学习笔记.md)中展开。

> **原文定位：**TR 38.821 Clause 6.3.3、Clause 9.1。RAN1 后续工作的焦点是 UE 不做时频预补偿时的 PRACH 序列和/或格式增强，报告本身没有完成候选选择。

#### 3.2.5 RAR 监测与 Msg3 调度

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

若 UE 从 Msg1 发送完毕后立即长时间盲监听 PDCCH，会增加功耗。更合理的候选方式是利用已知公共 RTT 延后 RAR 监测窗的起点，再让窗口长度覆盖处理抖动和差分不确定性：

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

因此网络在 Msg2 后调度 Msg3 时可能仍缺少 UE 完整的 UL 时间线，只能基于小区最大传播时延或最大差分时延进行保守调度。RAR→Msg3 的逻辑关系还需要使用第 2.3 节定义的 \(K_{\mathrm{offset}}\)：

\[
n_{\mathrm{Msg3}}
=
n_{\mathrm{RAR}}
+K_2
+\Delta
+K_{\mathrm{offset}},
\]

其中 \(\Delta\) 表示首次 Msg3 PUSCH 的附加处理时间。

因此需要分别保存三个时间状态：RAR 监测窗的起点决定“何时开始监听”，窗口长度决定“监听多久”，\(K_{\mathrm{offset}}\) 决定“收到 RAR 后在哪个未来 UL 时隙发送 Msg3”。把三者都写成一个“扩大窗口”会掩盖不同的功耗、检测和调度约束。

> **原文定位：**TR 38.811 Clause 7.3.4.1.2；TR 38.821 Clauses 6.2.1、7.2.1.1.1.2，Figures 7.2.1.1.1.2-3、7.2.1.1.1.2-9。RAR 监测窗起点、绝对 TA 知识和 Msg3 调度应分别核算。

#### 3.2.6 TA 范围与波束尺寸

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
\text{PUSCH载荷}.
\]

从 TA 角度，它允许 UE 在更早阶段向网络报告已应用的初始 TA 或相关辅助信息，使网络更早建立对 UE 绝对 UL 时间基准的认知，减少四步随机接入中“Msg1 已对齐但网络仍不知道 UE 已应用多少 TA”的调度不确定性。

两步随机接入不会消除 MsgA 的传播时间，也不会自动解决残余 CFO、PRACH 窗口和竞争冲突；其价值主要在于更早交换时间状态。该内容属于 TR 38.821 对 NTN 随机接入候选方案的研究，不应写成所有 NTN UE 必须使用两步随机接入。

> **原文定位：**TR 38.821 Clause 7.2.1.1.2、Figures 7.2.1.1.2-1 至 7.2.1.1.2-2；相关 TA 框架见 Clause 6.3.4。具体两步随机接入字段和规范行为应以相应 TS 版本为准。

### 3.3 空口联合状态与下游接口

完整的波束足迹、逻辑小区、PCI、带宽部分（Bandwidth Part，BWP）、极化资源和波束管理流程由第三份笔记拥有。本篇只描述同步、接入和控制过程需要消费哪些联合空口状态：

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

这些状态不必同时改变。服务链路切换（service-link switch）可以更换服务卫星或物理波束而保持逻辑小区，也可能连同参考点、Common TA、公共多普勒预补偿、SSB/CSI-RS 和 BWP 一起更新；馈电链路切换（feeder-link switch）即使不改变 UE 侧覆盖，也可能改变公共时延、频率参考和透明转发路径。切换不能简化成“选择新的最大 RSRP 波束”，还要保证目标状态生效时的时频连续性和调度因果性。

例如，逻辑小区和 PCI 可以保持不变，但服务卫星从 A 切换到 B。对高层看似仍是同一小区，对空口却可能同时改变参考点、公共 TA、瞬时多普勒、SSB/CSI-RS 发送波束和跨 DL-UL 偏移。仿真若只更新卫星位置、不原子更新这些关联状态，就会得到“覆盖已经切换、定时和调度仍属于旧链路”的不一致中间状态。

TR 38.821 以 Release 15/16 波束管理和 BWP 操作为 NTN 基线，但频率复用条件下“一波束一 BWP”“一波束一分量载波（Component Carrier，CC）”及 DL/UL BWP 联动等候选没有收敛。报告还认为在部分 NTN 场景中指示右旋圆极化（Right-Hand Circular Polarization，RHCP）或左旋圆极化（Left-Hand Circular Polarization，LHCP）模式可能有益，但是否支持信令仍留待规范阶段讨论。因此本篇只记录 BWP 和极化变化会影响同步、资源与接入状态；具体空间/频率组织详见[天线波束与覆盖组织](./03_NTN天线波束与覆盖组织_学习笔记.md)。

> **原文定位：**TR 38.821 Clauses 6.2.4-6.2.5、9.1。波束/BWP映射、极化模式信令和 feeder-link switch PHY影响均属于基线之上的后续讨论项，不是报告已收敛的统一方案。

向[链路、系统与多星仿真](./06_NTN链路系统与多星仿真_学习笔记.md)输出时，应将物理到达时间、决策信息、接收机残差、反馈和队列状态分别建模，不能统一折算成固定 SINR 扣减：

| 状态组 | 下游仿真处理 |
|---|---|
| 时频对齐：Common/Differential/Residual TA、\(K_{\mathrm{offset}}\)、原始/公共/残余多普勒 | PRACH到达窗口、上行对齐、捕获范围、调度和残差映射 |
| 反馈控制：CSI/CQI/MCS、RSRP/功控、HARQ/RLC/队列 | 信息老化、链路自适应、发射功率、进程占用和时延统计 |
| 接入辅助：UE位置、星历、Common TA、系统信息、PRACH/HARQ反馈模式 | 选择预补偿或退化搜索路径、检测模型和反馈事件队列 |
| 服务资源：Beam/Satellite/Reference Point、BWP/Polarization/Feeder Link | 多速率状态迁移、资源、链路预算及公共时频状态更新 |

## 4 无线资源管理测量、预测与移动性执行

无线资源管理（Radio Resource Management，RRM）把前文已经建立的参考信号、系统信息、波束/小区映射和几何预测变成可执行的移动性决策。本章拥有的完整链条是：

\[
\boxed{
\begin{aligned}
&\text{配置测量对象}
\rightarrow
\text{形成波束/小区量值}
\rightarrow
\text{判断量值是否仍有效},\\
&\text{评估无线或几何事件}
\rightarrow
\text{准备并执行移动性动作}.
\end{aligned}
}
\]

TR 38.821 Clause 7.3 负责识别 NTN 的移动性问题和候选增强；TS 38.304 与 TS 38.331 才分别规定空闲/非激活态过程和 RRC 测量、报告、条件执行行为。以下每节都保留这条“研究问题 → 规范机制”的版本边界。

继续使用全文参考场景：地面 UE 可以静止，但服务它的 LEO 覆盖持续移动。当前服务小区的 RSRP 可能仍然良好，网络却已经知道该区域即将离开服务范围；相邻波束的 RSRP 也可能与服务波束十分接近，却只剩很短可用时间。RRM 因而需要同时回答三件事：目标现在是否可测、测量到执行时是否仍有效、目标未来是否值得进入。

### 4.1 状态、对象与术语边界

同一个 UE 在不同无线资源控制（Radio Resource Control，RRC）状态下，RRM 的动作不同。表中的无线接入网通知区域（Radio Access Network-based Notification Area，RNA）是 `RRC_INACTIVE` UE 的接入网侧通知范围：

| RRC状态 | UE主要动作 | 网络主要动作 | 典型输出 |
|---|---|---|---|
| `RRC_IDLE` | 小区选择/重选、跟踪区更新、监听寻呼 | 广播系统信息并在登记区域寻呼 | 驻留小区与登记区域 |
| `RRC_INACTIVE` | 执行非激活态移动性并监听寻呼 | 保留 UE 上下文并维护 RNA/寻呼范围 | 驻留小区、RNA与可恢复上下文 |
| `RRC_CONNECTED` | 按配置测量并报告，评估条件执行 | 配置测量、准备目标并下发切换条件 | 服务小区、候选目标与执行时刻 |

RRM 与无线链路监测（Radio Link Monitoring，RLM）也不是同一层问题。RRM 询问“应该使用哪个小区/波束以及何时移动”；RLM 询问“当前服务链路是否仍满足同步/失步判据”。TR 38.821 Clause 7.3.4 标为 `Void`，表示该研究报告没有在该节形成独立 NTN RLM 方案，不能据此写成“NTN 不需要 RLM”。

本章还出现两个都缩写为 TA、但物理含义完全不同的术语：

\[
\boxed{
\mathrm{TA}_{\mathrm{timing}}
=
\text{Timing Advance，定时提前量},
\qquad
\mathrm{TA}_{\mathrm{tracking}}
=
\text{Tracking Area，跟踪区}.
}
\]

前者调整上行波形发射时刻，已经在第 1.2 节定义；后者是空闲/非激活态登记与寻呼的地理管理对象。后文涉及跟踪区时使用全称或 \(\mathrm{TA}_{\mathrm{tracking}}\)，不再用裸写的“TA”代替。

> **原文定位：**TR 38.821 Clauses 7.3.1-7.3.4；RRC 状态和移动性基线参见 TS 38.300、TS 38.304 与 TS 38.331。Clause 7.3.4 的 `Void` 只描述报告内容，不是否定 NR RLM 基线。

### 4.2 从参考信号到小区测量

连接态测量配置不是“让 UE 测一下邻区”这一条命令，而是多个对象的绑定关系：

\[
\begin{aligned}
&\texttt{MeasObjectNR}
+\texttt{ReportConfigNR}
\xrightarrow{\texttt{MeasId}}
\text{测量与报告任务},\\
&\texttt{QuantityConfig}
+\texttt{MeasGapConfig}
+\text{SMTC}
\rightarrow
\text{量值滤波与可观测时机}.
\end{aligned}
\]

其中，测量对象给出频点、小区和参考信号范围；报告配置给出周期/事件、门限、滞回和触发时间；测量标识 `MeasId` 将二者绑定。同步信号/物理广播信道块测量定时配置（SS/PBCH Block Measurement Timing Configuration，SMTC）告诉 UE 预期在哪些时刻寻找 SSB，测量间隙（Measurement Gap）则给 UE 留出暂停当前服务频点收发、切换到目标频点完成测量的机会。二者一项是“目标何时出现”，一项是“UE 何时能去看”，不能互相替代。

对 SSB 的小区量值形成，规范允许两类直观路径：若未配置合并门限/数量，可由最强 SSB 波束代表小区；若配置合并，则选择门限以上最强的至多 \(N\) 个 SSB，在功率线性域合并后再转换回 dB。用于理解的等功率平均可写成：

\[
P_{\mathrm{cell,dB}}
=
10\log_{10}
\left(
\frac{1}{N}
\sum_{i=1}^{N}10^{P_{i,\mathrm{dB}}/10}
\right).
\]

例如，三个 SSB 波束的 RSRP 分别为 \(-90\)、\(-93\) 和 \(-100\,\mathrm{dBm}\)，合并门限为 \(-96\,\mathrm{dBm}\)，最多取两个波束，则第三个波束不参与，前两个在线性功率域等权平均得到：

\[
P_{\mathrm{cell,dB}}
=
10\log_{10}
\left(
\frac{10^{-90/10}+10^{-93/10}}{2}
\right)
\approx
-91.25\,\mathrm{dBm}.
\]

这个结果既不是最强波束的 \(-90\,\mathrm{dBm}\)，也不应通过直接对任意数量的 dB 值求平均得到。例子采用等权平均只为说明“先转线性功率、再合并、最后回到 dB”的计算顺序；实际参与数量和量值形成以测量配置及 TS 38.215 为准。

CSI-RS 可按同类规则形成小区量值。这里的第一层（Layer 1，L1）测量负责参考信号量值，L3 滤波负责跨采样平滑；触发时间（Time To Trigger，TTT）要求进入条件持续满足一段时间。由此，事件判决使用的不是未经处理的某一次波束采样，而是如下链条的输出：

\[
\text{SSB/CSI-RS的L1测量}
\rightarrow
\text{波束到小区合并}
\rightarrow
\text{L3滤波}
\rightarrow
\text{事件门限与TTT}
\rightarrow
\text{MeasurementReport}.
\]

第 2.6 节已拥有 L3 一阶滤波公式，本节只定义其在移动性链条中的位置，避免重复一套滤波定义。

> **原文定位：**TS 38.331 Clause 5.5.1（测量配置）、Clause 5.5.3.2（L3 filtering）、`MeasObjectNR`、`ReportConfigNR`、`MeasId` 与 `QuantityConfigNR`；SSB/CSI-RS 量值形成见 Clause 5.5.3.3 及 TS 38.215 对相应测量量的定义。

### 4.3 NTN 中的测量有效性与时窗失配

NTN 移动性问题的核心不是“卫星快”这一句，而是测量所描述的空间关系在动作执行时是否仍成立。定义移动性信息年龄：

\[
T_{\mathrm{age}}
=
t_{\mathrm{execute}}-t_{\mathrm{meas}}
=
T_{\mathrm{L1}}
+T_{\mathrm{L3}}
+T_{\mathrm{TTT}}
+T_{\mathrm{report}}
+T_{\mathrm{decision}}
+T_{\mathrm{command}}.
\]

若服务几何以相对速度 \(\mathbf v_{\mathrm{rel}}\) 变化，仅由信息年龄引起的位置失配量级约为：

\[
\Delta s_{\mathrm{age}}
\approx
\left\|\mathbf v_{\mathrm{rel}}\right\|T_{\mathrm{age}}.
\]

该量不等于 UE 自身位移；静止 UE 面对移动的 LEO 波束时，\(\mathbf v_{\mathrm{rel}}\) 仍可很大。TR 38.821 因而识别出四类耦合问题：

| 问题 | 地面假设为何可能失效 | 应保存的额外状态 |
|---|---|---|
| 小区中心/边缘 RSRP 差异变小 | 相邻卫星波束可同时具有接近的接收功率，仅靠相对无线质量排序不够稳定 | 位置、星历、目标可用时段与滞回 |
| 邻区集合快速变化 | 报告中的最强邻区到执行时可能已离开可服务区域 | 候选集合的生成时刻和预计驻留时间 |
| 频繁或群体切换 | 地面固定 UE 也会被同一移动覆盖边界成批扫过 | 每 UE/每波束切换到达率与资源准备容量 |
| SMTC/Gap与到达时间不一致 | 不同卫星传播时延使目标 SSB/CSI-RS 到达位置偏离预测窗口 | 每目标传播时延及其变化率 |

信息年龄也不等于 SIB19 或测量配置的协议有效期。一个报告可能仍处于配置允许使用的时间范围内，但其描述的卫星/波束几何已经明显变化；反过来，几何预测仍然准确，也不表示一次快速遮挡前的 RSRP 测量仍然代表当前链路。协议有效性、几何可预测性和无线测量新鲜度应分别判断。

对服务小区 1 和目标小区 2，参考信号传播时延差为：

\[
\Delta\tau_{21}(t)
=
\frac{d_2(t)-d_1(t)}{c}.
\]

若测量窗口只按服务小区时间基准开启、却没有覆盖 \(\Delta\tau_{21}\) 及其不确定度，UE 即使拥有正确的邻频间隙，也可能在目标 SSB/CSI-RS 尚未到达或已经结束时采样。解决方向可以是扩大/平移窗口，或由网络利用位置、星历和目标链路时延补偿测量时刻；TR 38.821 将这些列为增强方向，并未在报告中规定唯一算法。

目标“最强”还不等于目标“值得进入”。若预测的目标可服务剩余时间为 \(T_{\mathrm{stay}}\)，则至少应满足：

\[
T_{\mathrm{stay}}
>
T_{\mathrm{prepare}}
+T_{\mathrm{execute}}
+T_{\mathrm{use,min}},
\]

否则切换刚完成就可能再次离开。TR 38.821 Table 7.3.2.1.4-1 的特定 LEO 参考几何中，50 km 与 1000 km 距离尺度对应的驻留时间约为 6.61 s 与 132.38 s；这些数值用于说明尺度关系，不是任意轨道和波束的通用定时器。

> **原文定位：**TR 38.821 Clauses 7.3.2.1.2-7.3.2.1.6，尤其是 Table 7.3.2.1.4-1；测量时机与传播时延差问题见 Clauses 7.3.2.2-7.3.2.3。\(T_{\mathrm{age}}\) 分解和目标驻留不等式是基于标准过程的工程记账式。

### 4.4 无线事件与几何/时间事件

三类事件分别回答不同问题：A3 判断“目标无线质量是否相对更好”，D1 判断“UE 是否进入目标地理区域”，T1 判断“计划执行时间是否到达”。NR 的 A3 事件比较邻区与服务小区的滤波量值。TS 38.331 Clause 5.5.4.4 的进入条件为：

\[
M_n+O_{fn}+O_{cn}-H_{ys}
>
M_p+O_{fp}+O_{cp}+O_{\mathrm{A3}},
\]

其中 \(M_n\)、\(M_p\) 是邻区与服务小区量值，\(O_f\) 和 \(O_c\) 分别是频点与小区偏置，\(H_{ys}\) 是滞回，\(O_{\mathrm{A3}}\) 表示 A3 事件偏置。忽略偏置时，它可理解为“邻区比服务小区好出门限并持续 TTT”。在 NTN 中，相邻波束 RSRP 差异可能长期很小，单独调低门限会增加乒乓，单独加大 TTT 又会增加信息年龄，因此需要把无线条件与可预测几何分开建模。

位置事件 D1 使用 UE 到两个 RRC 配置参考位置的距离，而不是直接使用“UE 到两颗卫星的斜距”。设距离为 \(M_{l1}\)、\(M_{l2}\)，进入条件为：

\[
M_{l1}-H_{ys}>\mathrm{Thresh}_1,
\qquad
M_{l2}+H_{ys}<\mathrm{Thresh}_2.
\]

它表达“已离开参考位置 1 的邻域，同时进入参考位置 2 的邻域”。退出条件通过反向滞回形成回差：\(M_{l1}+H_{ys}<\mathrm{Thresh}_1\) 或 \(M_{l2}-H_{ys}>\mathrm{Thresh}_2\)。参考位置通常由服务/候选覆盖规划确定；把它们直接解释为卫星瞬时位置会混淆地理触发与链路斜距。

条件事件 T1 使用协调世界时（Coordinated Universal Time，UTC）。若开始门限为 \(T_1\)、持续时间为 \(T_{\mathrm{duration}}\)，其有效区间可理解为：

\[
T_1<t_{\mathrm{UTC}}\le T_1+T_{\mathrm{duration}}.
\]

TS 38.331 中门限由 UTC 时间计数表达，持续时间按相应字段步长配置。它适合表示准地球固定小区的预计服务终止时刻或计划性卫星/链路切换窗口，但必须保证 UE 时间基准和配置有效期一致。表中的参考信号接收质量（Reference Signal Received Quality，RSRQ）与前文已经定义的 RSRP、SINR 都是无线量值，但三者刻画的信号、干扰和资源占用侧重点不同。

| 触发类型 | 主要输入 | 优势 | 主要失效方式 | 状态 |
|---|---|---|---|---|
| A3/A4/A5 | L3滤波后的RSRP/RSRQ/SINR | 直接反映无线链路 | 小区差异小、报告老化、乒乓 | NR规范事件 |
| D1 | UE位置与两个参考位置 | 可在无线边缘前预测 | 位置过期、参考位置配置不当 | TS 38.331规范事件 |
| T1 | UTC时间与持续区间 | 适合计划性覆盖变化 | 时间不同步、计划变化 | TS 38.331规范事件 |
| TA、仰角、定时器等 | 几何/协议状态 | 具有物理可解释性 | 误差与配置复杂度 | TR 38.821候选，不能统称为已规范独立事件 |

> **原文定位：**A3 见 TS 38.331 Clause 5.5.4.4；D1 与 Conditional Event D1 见 Clause 5.5.4.15；Conditional Event T1 见 Clause 5.5.4.16。位置、时间、TA 与仰角作为 NTN 条件切换方向的研究讨论见 TR 38.821 Clause 7.3.2.3。

### 4.5 条件切换的准备、评估与执行

普通切换在当前无线条件触发后才开始目标准备，长传播和处理链会压缩剩余服务时间。条件切换（Conditional Handover，CHO）把“准备”和“执行”分开：

\[
\text{网络提前准备目标并下发候选配置}
\rightarrow
\text{UE持续评估条件}
\rightarrow
\text{条件满足时本地执行}
\rightarrow
\text{在目标侧完成接入}.
\]

CHO 没有消除传播时延，而是把可提前完成的配置和目标准备移到链路仍稳定的时间段。一个候选条件可绑定至多两个 `MeasId`；配置两个时，UE 只有在二者都满足后才执行。这个“与”关系使无线条件与几何/时间条件可以共同约束执行。

普通切换与 CHO 的差别可以按“何时准备、谁决定执行”理解：

| 对比项 | 普通切换 | 条件切换 |
|---|---|---|
| 目标准备时机 | 当前触发条件满足后再开始 | 服务链路仍稳定时提前完成 |
| 执行判决位置 | 网络根据报告下发切换命令 | UE按网络预配条件在本地判决 |
| 长时延下的主要风险 | 命令到达前源链路已明显恶化 | 目标配置过期或目标届时不可用 |
| 仍需完成的过程 | 目标接入和状态更新 | 目标接入和状态更新，同样不会消除传播时延 |

因此 CHO 缩短的是“触发之后仍需在线交换的准备链”，不是把切换过程变成无时延的波束选择。

版本边界必须明确：

| 版本 | NTN CHO条件约束 | 正确理解 |
|---|---|---|
| Release 17 | 配置 D1 或 T1 时，网络还配置第二个 A3/A4/A5 无线事件；D1 与 T1 不同时作为该对条件 | 几何/时间条件不能单独触发 CHO |
| Release 18 | 对声明 `ntn-CHO-OnlyLocationTimeTrigger-r18` 能力的 UE，可仅以 D1、D2 或 T1 之一构成条件 | 纯位置/时间触发是能力受限增强，不是所有 UE 的默认行为 |

因此，看到 `CondEvent D1/T1` 不能直接推出“网络已完全不依赖无线测量”。Release 17 的典型逻辑仍是：

\[
\text{目标无线条件可接受}
\land
\text{目标地理/时间窗口已到}
\Rightarrow
\text{执行CHO}.
\]

即使条件满足，执行前仍要确认目标配置未过期、目标小区仍在服务、随机接入资源可用，并将第 3.3 节的 `Satellite`、`Beam`、`ReferencePoint`、Common TA、公共频率补偿和 \(K_{\mathrm{offset}}\) 按同一生效边界更新。只替换“最大 RSRP 波束”而不更新联合空口状态，会造成目标侧时频不连续。

> **原文定位：**TS 38.331 Clause 5.3.5.13.3-5.3.5.13.4（条件重配置评估与执行）及相关条件执行字段；Release 17 的 D1/T1 配对约束见 V17.13.0，Release 18 纯位置/时间能力见 V18.6.0 的 `ntn-CHO-OnlyLocationTimeTrigger-r18`。TR 38.821 Clause 7.3.2.3 记录的是进入规范前的候选与利弊。

### 4.6 空闲/非激活态移动性、跟踪区与寻呼

空闲态和非激活态仍以 NR 小区选择/重选准则为基线。SIB19 的 `referenceLocation-r17`、`distanceThresh-r17` 和 `t-Service-r17` 提供的是“何时应开始测量”的 NTN 辅助，不是直接绕过 RSRP/RSRQ/SINR 准则指定目标小区。

在 TS 38.304 Clause 5.2.4.2 的相应前提下，具有有效位置的 NTN UE 位于 `distanceThresh` 内时可以不启动相关邻区测量；越过距离门限后应执行测量。对广播 `t-Service` 的准地球固定小区，在服务终止时刻之前，UE 需要及时开始频内、频间及必要的异系统测量，而不能只因当前服务小区无线质量仍好就继续等待。由此形成两级逻辑：

\[
\underbrace{\text{位置/时间规则}}_{\text{决定是否开始测量}}
\rightarrow
\underbrace{\text{无线量值与重选规则}}_{\text{决定选择哪个小区}}.
\]

跟踪区组织面对的是另一组权衡。若 \(\mathrm{TA}_{\mathrm{tracking}}\) 随卫星/波束移动，地面静止 UE 也可能因覆盖扫过而频繁更新登记；若跟踪区固定在地面，登记更新压力下降，但网络必须把固定地理登记区域映射到当时可用的卫星、小区和寻呼资源。TR 38.821 在工作项目建议中推荐固定跟踪区方向；这是研究报告的建议，具体部署仍取决于核心网登记、寻呼和小区规划。

寻呼负载与登记更新不能分开优化：扩大寻呼区域可减少移动导致的登记更新，却会让更多小区发送寻呼；缩小区域则相反。对移动覆盖还应保存“某个登记区域在时刻 \(t\) 由哪些小区可达”的映射，而不是把寻呼列表固定绑定到某颗卫星。

地球固定/地球移动波束与小区的几何定义由[天线波束与覆盖组织](./03_NTN天线波束与覆盖组织_学习笔记.md)拥有。本章只消费其结果：地球移动覆盖会使静止 UE 经历小区边界；准地球固定覆盖也只在有限服务时段内固定，仍需要利用 `t-Service` 或计划性切换处理卫星更替。

> **原文定位：**TS 38.304 Clause 5.2.4.2（位置与 `t-Service` 辅助测量规则）；TS 38.331 `SIB19-r17`；TR 38.821 Clause 7.3.1（Idle mode mobility）、Clause 7.3.3（Paging）与 Clause 7.4（Earth-fixed/Earth-moving cells）。固定跟踪区建议见 Clause 7.3.1 的结论/建议段。

### 4.7 无线资源管理输出与仿真闭环

RRM 不能只向系统仿真器输出一次“切换成功/失败”。最小状态应保留：

\[
\mathbf s_{\mathrm{RRM}}(t)
=
[\,
\mathcal M,
\mathcal N(t),
\mathbf q(t_{\mathrm{meas}}),
T_{\mathrm{age}},
\mathcal E(t),
\mathcal C_{\mathrm{CHO}},
T_{\mathrm{stay}},
\mathcal P_{\mathrm{paging}}
\,],
\]

其中 \(\mathcal M\) 是测量配置，\(\mathcal N(t)\) 是带时间戳的候选邻区集合，\(\mathbf q\) 是波束/小区量值，\(\mathcal E\) 是事件状态，\(\mathcal C_{\mathrm{CHO}}\) 是已准备的 CHO 候选，\(T_{\mathrm{stay}}\) 是预计目标驻留时间，\(\mathcal P_{\mathrm{paging}}\) 是登记区到当前可达小区的寻呼映射。

向[链路、系统与多星仿真](./06_NTN链路系统与多星仿真_学习笔记.md)输出时，至少分别检查：

| 仿真检查点 | 不能省略的时间/状态 |
|---|---|
| 邻区参考信号是否被测到 | SMTC、Measurement Gap、目标传播时延差与测量窗口 |
| 报告是否仍代表执行时几何 | 测量、滤波、TTT、报告、决策和命令时间戳 |
| 目标是否值得切入 | 目标可用性、预计驻留时间、准备/接入耗时 |
| CHO是否允许执行 | MeasId 的与逻辑、Release/UE能力、配置有效期 |
| 切换后空口是否连续 | 目标 Beam/Satellite/Reference Point、TA、频偏与调度偏移的共同生效 |
| Idle/Inactive负载是否可信 | 位置/时间测量门控、重选、登记更新与寻呼扇出 |

至此，前五篇笔记的接口形成闭环：轨道几何给出可预测的时空状态，波束笔记给出覆盖与候选资源，信道笔记给出可测无线量，本文把它们变成带有效期的 RRM 决策，第六篇再统计测量失败、切换中断、乒乓、群体切换、寻呼和端到端业务影响。
