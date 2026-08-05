---
title: "TR 38.811 NTN 对 NR 影响与适配机制学习笔记 v3"
author: ""
date: "2026-08-04"
---

# TR 38.811 NTN 对 NR 影响与适配机制学习笔记

> 本笔记基于 3GPP TR 38.811 V15.4.0（Release 15）Clause 7 整理，并结合 TS 38.211、TS 38.214 中的相关 NR 基线机制解释。正文按“平台运动与同步 → 长传播时延、闭环与双工 → 大波束随机接入 → 局部多径与 OFDM → 星上载荷 → 架构映射”的知识链展开。TR 中的 Problem statement、Assessment 和 NR impact consideration 分别表示问题提出、条件评估和潜在影响，不能直接等同于已经写入规范的修改。

## 1 Clause 7 影响分析框架

Clause 7 关注的不是单一“卫星信道参数”，而是六类物理或架构差异如何越过 NR 机制边界：平台运动改变覆盖、时延和频率；超长传播路径拉长反馈与收发切换；大波束放大用户间差分时延；局部多径造成频率选择性；星上硬件引入相位噪声与非线性；功能部署位置决定控制环路跨越哪些链路。判断时需要把每个现象转化为可与 NR 参数直接比较的物理量。

### 1.1 NTN 物理特征与 NR 机制边界

TR 38.811 Clause 7 的任务不是直接给出 NTN 解决方案，而是把 NTN 与地面蜂窝的物理差异逐项映射到可能受影响的 NR 功能。其基本判断过程可以写成：

\[
\text{NTN物理特征}
\rightarrow
\text{关键时空尺度}
\rightarrow
\text{NR基线容限}
\rightarrow
\text{失效条件}
\rightarrow
\text{影响类型}
\]

这里最重要的不是“某个量很大”，而是它在完成补偿后是否仍然超出具体机制的工作范围。例如，600 km LEO 在 2 GHz 下可产生约 \(48\,\mathrm{kHz}\) 的原始多普勒，但如果星历和位置辅助已经消除其中绝大部分，真正决定 OFDM 正交性的是残余频偏，而不是原始多普勒本身。

TR 38.811 将主要关系汇总为：

| NTN物理或架构特征 | 直接效应 | 主要受影响的NR机制 | 原文位置 |
|---|---|---|---|
| 平台运动 | 波束移动、时延漂移、多普勒 | 切换与寻呼、TA更新、初始同步、DM-RS时间密度 | Clauses 7.3.2.1-7.3.2.4 |
| 平台高度与传播路径 | 大绝对时延、长RTT和TDD保护时间 | HARQ、MAC/RLC、ACM、功控、FDD/TDD选择 | Clauses 7.3.3.1-7.3.3.3、7.3.6.1 |
| 大波束足迹 | 波束内差分时延 | PRACH、RAR窗口、RAR中的TA | Clauses 7.3.4.1-7.3.4.2 |
| 传播信道 | 时延扩展和频率选择性 | DM-RS频域密度、循环前缀 | Clauses 7.3.5.1-7.3.5.2 |
| 星上载荷 | 相位噪声、功放非线性 | PT-RS、PAPR和功率回退 | Clauses 7.3.7.1-7.3.7.2 |
| 载荷功能位置 | 控制环路和接口跨越的链路不同 | NG-RAN映射、接口定时器、移动性 | Clause 7.3.8.1 |

> **关键原文定位：**TR 38.811 Clauses 7.1-7.3.1，尤其是 Tables 7.1-1、7.2-1 和 7.3.1-1。

## 2 平台运动引起的移动性与短时同步

NGSO 平台运动同时改变“服务对象、传播距离和载波频率”：波束足迹扫过地面，斜距随时间变化，并产生随时间变化的几何多普勒。三种现象来自同一轨道运动，却作用于不同时间尺度，因此不能只用一个“卫星速度很快”概括。移动性关注分钟级波束驻留，TA 关注命令周期内的时延漂移，初始同步关注残余频偏能否落入捕获范围，DM-RS 则关注一个时隙内的剩余变化。

| 物理现象 | 关键比较量 | 主要影响机制 |
|---|---|---|
| 波束足迹移动 | 波束驻留时间与重选、切换和寻呼更新时间 | Cell、Tracking Area与移动性管理 |
| 斜距连续变化 | \(\dot\tau T_{\mathrm{update}}\) 与TA单次调整范围 | 上行定时对齐 |
| 大几何多普勒 | 预补偿后的残余频偏与PSS/SSS捕获范围 | 下行初始同步 |
| 多普勒随时间变化 | \(\dot f_D T_{\mathrm{DMRS}}\) 与接收机跟踪能力 | DM-RS时间密度 |

### 2.1 移动波束、小区与跟踪区

非地球静止轨道（Non-Geostationary Orbit，NGSO）卫星相对地面快速运动。TR 38.811 给出的示例中，约两小时轨道周期的 LEO 从地平线到地平线对固定 UE 可见约 20 分钟，而 UE 停留在单个波束中的时间通常只有几分钟。

即使 \(v_{\mathrm{UE}}=0\)，也会出现：

\[
\text{Serving beam}(t),\quad
\text{Serving satellite}(t),\quad
\text{Serving cell}(t)
\]

随时间变化。这里需要区分：

| 对象 | 含义 | 是否必须随卫星移动 |
|---|---|---|
| Beam | 物理天线方向图形成的空间波束 | 随星波束通常移动，对地固定波束可通过赋形补偿 |
| Cell | NR协议和资源管理中的小区 | 可以与波束绑定，也可以采用对地固定逻辑映射 |
| Tracking Area | 空闲态位置管理与寻呼使用的区域 | 通常更适合按地理区域设计，不宜永久绑定移动波束 |

如果把每个移动波束直接等同于一个固定小区或 Tracking Area，静止 UE 也会频繁触发小区重选、注册更新和寻呼区域变化。NGSO 的关键不是简单“加快切换”，而是用卫星星历、波束计划和 UE 位置建立地理区域与动态波束之间的映射。

相邻卫星波束还可能采用不同频率或不同极化，因此地面 NR 中基于同频相邻波束的部分 beam management 假设不能直接照搬。若 UE 位置和卫星星历可用，网络可提前知道覆盖持续时间、下一候选波束及预计切换时刻，从而减少测量和切换开销。

> **关键原文定位：**TR 38.811 Clause 7.3.2.1。报告认为 GEO 和多数 HAPS 场景不产生同等程度的协议影响；NGSO 的切换与寻呼适配需进一步研究。

### 2.2 传播时延漂移与 TA 更新

平台运动会使传播距离连续变化。对第 \(i\) 个 UE：

\[
\dot\tau_i(t)
=
\frac{1}{c}\frac{\mathrm d d_i(t)}{\mathrm dt}.
\]

Timing Advance（定时提前量，TA）的作用，是让 UE 的上行信号在 gNB 接收端对齐。接入后的 TA 更新处理的是当前 TA 的增量调整，而不是每次重新编码整个卫星绝对传播时延。

TR 38.811 以最大约 \(35\,\mu\mathrm{s/s}\) 的时延漂移评估现有 TA 更新。不同子载波间隔（Subcarrier Spacing，SCS）下，TA 量化步长和单次允许更新量不同：

| SCS | 时隙长度 | TA量化步长 | 报告估计的单次最大调整 | 跟踪最大漂移的命令量级 |
|---:|---:|---:|---:|---:|
| 15 kHz | 1 ms | 520.83 ns | 约16.6 μs | 约10次/s |
| 60 kHz | 0.25 ms | 130.21 ns | 约4.15 μs | 约40次/s |
| 120 kHz | 0.125 ms | 65.10 ns | 约2.1 μs | 约80次/s |

表中命令次数是 TR 按给定最大漂移和更新量得到的数量级判断。SCS 增大后，TA 分辨率更细，但单次安全调整量变小，因而可能需要更频繁更新。

“传播时延大于一个 TTI”不意味着 TA 不能跨时隙工作。NR 中 TTI 不是固定等于 1 ms 的唯一单位；调度可采用时隙或 mini-slot。真正的问题是：UE 的发送时刻需要跨越时隙边界进行整体平移，并保证网络和 UE 对公共时延参考保持一致。

> **关键原文定位：**TR 38.811 Clause 7.3.2.2、Table 7.3.2.2.2-1。报告结论是需要研究可预测 NTN 时延下的上行对齐方案，而不是认为 TA 必须在单个 TTI 内完成全部补偿。

### 2.3 原始多普勒与下行初始同步

UE 初始接入时首先检测主同步信号（Primary Synchronization Signal，PSS）和辅同步信号（Secondary Synchronization Signal，SSS），完成时频同步和物理小区标识检测。TR 使用地面 UE 的约 \(5\,\mathrm{ppm}\) 初始频偏鲁棒性作为比较基线：

\[
5\,\mathrm{ppm}\times 2\,\mathrm{GHz}=10\,\mathrm{kHz},
\]

\[
5\,\mathrm{ppm}\times 20\,\mathrm{GHz}=100\,\mathrm{kHz}.
\]

600 km LEO 的参考最大原始多普勒约为：

- S 波段 2 GHz：\(\pm48\,\mathrm{kHz}\)；
- Ka 波段 20 GHz：\(\pm480\,\mathrm{kHz}\)。

两者都明显超过相应的 \(5\,\mathrm{ppm}\) 数值。因此，如果 UE 在未知多普勒条件下直接搜索 PSS/SSS，地面 NR 的单次捕获范围可能不足。

但该比较不能直接推出“必须修改同步信号”。波束中心频率预补偿、较窄波束、UE 位置和卫星星历都能缩小每个波束内的实际搜索范围。可写成：

\[
\delta f_{\mathrm{sync}}
=
f_D
-
\hat f_{D,\mathrm{common}}
-
\hat f_{D,\mathrm{UE}},
\]

真正需要与同步捕获能力比较的是 \(\delta f_{\mathrm{sync}}\)。当卫星高度足够高，或公共预补偿已把残差压入原有范围时，报告不预期同步信号本身发生影响；否则需要进一步研究更宽捕获或辅助补偿。

> **关键原文定位：**TR 38.811 Clause 7.3.2.3。报告给出的约 13,000 km 高度判断来自其 \(5\,\mathrm{ppm}\) 与最大几何多普勒比较，不是通用系统部署门限。

### 2.4 多普勒变化率与 DM-RS 时间跟踪

解调参考信号（Demodulation Reference Signal，DM-RS）的时间密度用于跟踪时变信道和符号间频率/相位变化。600 km LEO 的最大多普勒变化率约为：

| 载频 | 最大变化率 | 1 ms内最大变化 |
|---:|---:|---:|
| 2 GHz | 544 Hz/s | 0.544 Hz |
| 20 GHz | 5.44 kHz/s | 5.44 Hz |
| 30 GHz | 8.16 kHz/s | 8.16 Hz |

TR 将这些数值与 1 ms 观察期内的 \(\pm0.1\,\mathrm{ppm}\) 频率误差量级比较：2、20、30 GHz 分别对应约 \(\pm200\,\mathrm{Hz}\)、\(\pm2\,\mathrm{kHz}\)、\(\pm3\,\mathrm{kHz}\)。因此，报告认为单个时隙内的多普勒变化不足以要求改变 DM-RS 在时间上的位置。

这里需要保留两个边界：

1. 该结论比较的是一个时隙内的多普勒**变化量**，不是数十至数百 kHz 的原始多普勒；原始多普勒仍需先捕获和补偿。
2. 该结论不覆盖较长闭环中的信息老化。若补偿信息年龄为 \(T_{\mathrm{age}}\)，残余误差近似为：

\[
\delta f_{\mathrm{stale}}
\approx
\dot f_D T_{\mathrm{age}}.
\]

例如 2 GHz、\(\dot f_D=544\,\mathrm{Hz/s}\) 时，1 ms 内只变化 0.544 Hz，但 100 ms 内会变化 54.4 Hz。前者支持“DM-RS时间位置无需改变”，后者已经可能影响反馈式补偿的时效性。

> **关键原文定位：**TR 38.811 Clause 7.3.2.4、Table 7.3.2.4.1-1。公共、残余和信息老化的分解属于基于原文条件的接收机工程解释。

## 3 长传播时延下的协议闭环与双工

卫星高度首先带来很大的一程传播时延 \(\tau\)，再由不同收发过程形成不同含义的“往返时间”。它一方面拉长测量、反馈、重传和确认闭环，另一方面使 TDD 的最后一个下行符号和第一个上行符号之间必须留出接近传播 RTT 的保护时间。因此，双工方式确实属于长传播时延的影响，但需要区分两种量：协议闭环使用包含处理与调度的 \(T_{\mathrm{loop}}\)，TDD 保护主要由纯传播量 \(\tau_{\mathrm{DL}}+\tau_{\mathrm{UL}}\) 决定。

| 受影响对象 | 关键比较量 | 主要后果 |
|---|---|---|
| HARQ流水 | HARQ周期与相邻调度机会 | 进程数、软缓存和在途数据增加 |
| MAC/RLC与接口过程 | 最大协议RTT与定时器、窗口 | 超时、缓存和重传参数需扩展 |
| ACM与功率控制 | 闭环时延与信道变化时间 | CQI老化、预测误差和额外裕量 |
| TDD上下行切换 | 传播RTT与可用保护时间 | 保护开销增大，推动部分场景采用FDD |

### 3.1 单程时延、传播 RTT 与协议闭环

传播往返时间（Round-Trip Time，RTT）必须先明确端点。透明载荷、地面 gNB 场景的一程路径同时经过服务链路（service link，UE 与卫星/HAPS 之间）和馈电链路（feeder link，卫星/HAPS 与地面网关之间）：

\[
\tau_{\mathrm{one-way}}
=
\tau_{\mathrm{service}}
+
\tau_{\mathrm{feeder}},
\]

而星上 gNB 的用户侧空口闭环主要只跨 service link。协议反馈时间还需加入发送、接收、处理和调度：

\[
T_{\mathrm{loop}}
=
T_{\mathrm{RTT,prop}}
+
T_{\mathrm{TX}}
+
T_{\mathrm{RX,proc}}
+
T_{\mathrm{schedule}}.
\]

因此，“600 km LEO 的 RTT”不是唯一固定数值。此前几何笔记中的 12.88 ms 和 28.408 ms，分别对应星上 gNB 用户链路与透明转发至地面 gNB 的物理传播 RTT；TR Clause 7 的简化评估则常用约 50 ms LEO、180 ms MEO 和 600 ms GEO/HEO 的 HARQ 周期量级，其中已经包含其选定场景和处理时序假设。

### 3.2 HARQ 流水与并行进程

混合自动重传请求（Hybrid Automatic Repeat reQuest，HARQ）在物理层/MAC 层进行快速反馈与软合并。一个 HARQ 进程发送传输块后，在 ACK/NACK 返回之前不能无条件复用同一进程状态。为了在等待反馈时继续发送，需要多个进程并行流水。

可用下式理解最小并行进程数：

\[
N_{\mathrm{HARQ,min}}
\approx
\left\lceil
\frac{T_{\mathrm{HARQ}}}{T_{\mathrm{slot}}}
\right\rceil,
\]

其中 \(T_{\mathrm{HARQ}}\) 是从初传到对应 ACK/NACK 完成解码的周期，\(T_{\mathrm{slot}}\) 是相邻可调度传输机会的时间间隔。该式说明：RTT 越长、时隙越短，为保持满流水所需的进程越多。

TR 在 15 kHz SCS、1 ms 时隙的参考条件下给出：

| 场景 | 参考HARQ周期 | 参考最小进程数 | 报告判断 |
|---|---:|---:|---|
| 地面NR | 16 ms | 16 | Rel-15可行 |
| LEO | 50 ms | 50 | 扩展HARQ后可行 |
| MEO | 180 ms | 180 | 待研究，涉及TBS/MCS和实现能力 |
| GEO/HEO | 600 ms | 600 | 待研究，直接线性扩展压力很大 |

若 SCS 变为 \(15\times2^\mu\,\mathrm{kHz}\)，时隙长度缩短为：

\[
T_{\mathrm{slot}}
=
\frac{1\,\mathrm{ms}}{2^\mu}.
\]

在保持每时隙一个新传输机会的假设下，同一 RTT 内所需进程数会近似按 \(2^\mu\) 增长。更大 SCS 不能消除传播 RTT，只会让调度时间单位更细。

进程数扩展同时带来：

- UE 和 gNB 需要保存更多传输块状态、软信息和冗余版本；
- 并行编解码与调度上下文增加；
- HARQ 进程编号、反馈关联和缓存管理更复杂；
- 大量数据处于“已发送但尚未确认”的在途状态。

用原始数据率给出一个仅用于理解的缓存下界：

\[
B_{\mathrm{flight}}
\gtrsim
R T_{\mathrm{ACK}}.
\]

若 \(R=100\,\mathrm{Mbit/s}\)、\(T_{\mathrm{ACK}}=0.6\,\mathrm{s}\)，仅原始在途数据就约为 \(60\,\mathrm{Mbit}=7.5\,\mathrm{MB}\)。实际 HARQ 软缓存还取决于编码块、量化精度、冗余版本和多用户并发，不能直接用该数值代替实现需求。

TR 给出的两类研究方向是：

1. 对低到中等 RTT 扩展 HARQ 进程与时序；
2. 对超长 RTT 限制或关闭部分 HARQ 能力。

关闭 HARQ 不表示可靠性问题消失，而是把剩余误块更多交给更高层 ARQ、应用容错或更强编码处理，代价通常是更长重传时延和更粗粒度的数据恢复。

> **关键原文定位：**TR 38.811 Clause 7.3.3.1，Figures 7.3.3.1-1、7.3.3.1.1-1，Table 7.3.3.1.1-1。表中数值是研究评估用参考，不是所有 NTN 系统的强制 HARQ 进程配置。

### 3.3 HARQ 与 RLC ARQ

HARQ 与自动重传请求（Automatic Repeat reQuest，ARQ）解决的是不同层级的可靠性：

| 机制 | 所在层次 | 反馈粒度 | 接收端处理 | 长RTT下的主要压力 |
|---|---|---|---|---|
| HARQ | PHY/MAC | 传输块及冗余版本 | 可进行软合并 | 并行进程数、软缓存、ACK/NACK关联 |
| RLC ARQ | RLC确认模式 | RLC PDU/分段 | 基于状态报告重传 | 发送窗口、重传定时器、状态报告和缓存 |

HARQ 试图在短时间内修复无线误块；RLC ARQ 负责处理 HARQ 未解决的剩余丢失。若 HARQ 被限制，RLC 可能看到更高的丢包率，但其一次恢复需要跨越更长的状态报告和重传闭环。因此不能把“关闭 HARQ”理解成“由 RLC 无代价替代”。

### 3.4 MAC/RLC 缓存、定时器与调度

长 RTT 会使发送方更久无法确认数据是否成功。TR 对 MAC/RLC 的核心结论是：协议逻辑本身未必需要重写，但参数范围必须覆盖 NTN 的最长闭环。

发送缓存至少需要覆盖：

\[
T_{\mathrm{buffer}}
>
T_{\mathrm{RTT,max}}
+
T_{\mathrm{processing}}
+
T_{\mathrm{retransmission\ margin}}.
\]

相应需要调整：

- 发送窗口与缓存容量；
- 重传超时和状态报告周期；
- 数据丢弃前允许的最大重传次数；
- 上行调度请求、授权和数据发送之间的时序参数。

LEO 的时延随仰角变化，因此缓存和超时不能只按星下点最小时延配置，而应覆盖最低工作仰角下的最大预期 RTT。若定时器只按平均值配置，卫星靠近覆盖边缘时容易误判超时。

> **关键原文定位：**TR 38.811 Clause 7.3.3.2。原文结论是 ARQ 协议本身不一定改变，但 buffer、timeout、重传次数和上行调度时序需按 NTN 长时延重新定尺寸。

### 3.5 ACM 与功率控制的反馈老化

自适应编码调制（Adaptive Coding and Modulation，ACM）和闭环功率控制都遵循：

\[
\text{测量}
\rightarrow
\text{反馈}
\rightarrow
\text{决策}
\rightarrow
\text{执行}.
\]

如果闭环时延 \(T_{\mathrm{loop}}\) 与信道变化时间 \(T_{\mathrm{var}}\) 同量级或更长，决策生效时使用的测量已经过期：

\[
T_{\mathrm{loop}}
\ll
T_{\mathrm{var}}
\quad\text{时闭环跟踪有效。}
\]

不同 NTN 场景中，可跟踪对象不同：

| 场景 | 主要变化 | 相对时间尺度 | ACM/功控能力 |
|---|---|---|---|
| GEO Ka | 雨衰 | 通常较慢 | ACM可发挥作用，但需要滞回避免MCS往返振荡 |
| GEO S | 遮挡和多径快衰落 | 可能快于约0.5 s RTT | 难以逐衰落跟踪，只能保留较大裕量 |
| LEO | 斜距引起的自由空间损耗变化 | 通常慢于约20-50 ms闭环 | 可预测和跟踪大尺度变化，仍不能跟踪快速多径 |
| 手持终端链路 | 阴影、姿态和局部遮挡 | 场景相关 | 需要预测、滞回和保守MCS，不能只依赖旧CQI |

卫星和手持 UE 都常处于功率受限状态。GEO 链路的可用功率余量可能很小，因此闭环功控更多用于跟踪慢变化，而不是像地面系统一样追踪快速衰落。

可用下式描述工程实现中的保守 MCS 选择：

\[
\gamma_{\mathrm{design}}
=
\hat\gamma_{\mathrm{predicted}}
-
M_{\mathrm{total}},
\]

\[
I_{\mathrm{MCS}}^\star
=
\max\left\{
i:
\gamma_{\mathrm{req}}(i,\mathrm{BLER}_{\mathrm{target}})
\le
\gamma_{\mathrm{design}}
\right\}.
\]

其中 \(\hat\gamma_{\mathrm{predicted}}\) 是预测数据真正到达接收端时的 SINR，而不是直接照搬过去的测量。总裕量可工程化分解为：

\[
M_{\mathrm{total}}
=
M_{\mathrm{estimation}}
+
M_{\mathrm{feedback\ aging}}
+
M_{\mathrm{loop\ delay\ uncertainty}}.
\]

需要明确：这个 \(M_{\mathrm{total}}\) 分解是便于理解的实现模型，不是 TR 38.811 或 NR 规范直接定义的协议参数。3GPP 规定 MCS 索引对应的调制阶数、目标码率和相关信令，但 CQI 到 MCS 的映射、预测器、滞回、外环链路自适应步长以及 NTN 额外裕量由设备实现决定。

MCS 也不是“由物理发射机随意选择”：

- 下行 PDSCH 由 gNB 调度器选择，通过 DCI 通知 UE；
- 上行 PUSCH 虽由 UE 发射，通常仍由 gNB 在 DCI 或 RAR UL grant 中给出 MCS；
- 弯管载荷下调度器通常位于地面 gNB，再生载荷下可位于星上。

因此标准与实现的边界是：标准约束可选 MCS、控制字段和收发行为；网络调度器选择 MCS；具体如何消化长 RTT 造成的反馈老化属于实现算法。

> **关键原文定位：**TR 38.811 Clause 7.3.3.3；MCS和物理层调度基线参见 TS 38.214 Clauses 5.1.3、5.2.2 和 6.1.4。TR 只要求进一步研究功控和 ACM 闭环所需裕量，没有规定上述工程分解的数值。

### 3.6 TDD 保护时间的传播来源

时分双工（Time Division Duplex，TDD）在同一频段内切换上下行。考虑下行转上行：gNB 在 \(t=0\) 发完最后一个下行符号，该符号经过 \(\tau_{\mathrm{DL}}\) 才被 UE 收完；UE 完成收发切换后发送第一个上行符号，该符号又经过 \(\tau_{\mathrm{UL}}\) 才到达 gNB。

因此从 gNB 时间轴看，保护时间至少满足：

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

若忽略收发切换和同步余量，并假设上下行近似对称：

\[
T_{\mathrm{guard}}
\approx
2\tau
=
T_{\mathrm{RTT,prop}}.
\]

这里的 RTT 只表示传播往返时间，不包含检测、调度、编解码或 HARQ 处理时间。其物理依据是“最后一个 DL 符号先到 UE，随后第一个 UL 符号再返回 gNB”，而不是等待一个完整协议请求-响应。

Timing Advance 不能凭空消除这一保护时间。如果 UE 在最后一个下行符号尚未接收完时就提前发送上行，半双工终端会发生收发冲突。多 UE 情况下应按最坏传播路径配置：

\[
T_{\mathrm{guard}}
\ge
\max_i
\left[
\tau_{\mathrm{DL},i}
+
\tau_{\mathrm{UL},i}
\right]
+
T_{\mathrm{sw}}
+
M_{\mathrm{timing}}.
\]

### 3.7 LEO 与 GEO 的 TDD 保护开销

TR 使用最大单程传播时延给出保守量级：

- 600 km LEO：\(2\times7\,\mathrm{ms}=14\,\mathrm{ms}\)；
- GEO：\(2\times270\,\mathrm{ms}=540\,\mathrm{ms}\)。

若用一个仅用于理解的 100 ms 有效数据周期，可定义：

\[
\eta
=
\frac{T_{\mathrm{data}}}
{T_{\mathrm{data}}+T_{\mathrm{guard}}}.
\]

则 LEO 和 GEO 的示意效率分别约为：

\[
\eta_{\mathrm{LEO}}
=
\frac{100}{100+14}
\approx87.7\%,
\]

\[
\eta_{\mathrm{GEO}}
=
\frac{100}{100+540}
\approx15.6\%.
\]

该例不是规范帧结构，只用于说明 GEO TDD 若按完整传播 RTT 留保护时间，资源利用率会非常低。LEO 可能支持 TDD，但还需处理 \(T_{\mathrm{guard}}(t)\) 随卫星位置和仰角连续变化的问题。

### 3.8 FDD 与 TDD 的场景选择

频分双工（Frequency Division Duplex，FDD）上下行使用不同频段，不需要为 DL/UL 切换预留上述超长传播保护时间。因此 TR 认为 FDD 是多数 NR NTN 接入的优选方式。

FDD 仍有自身条件：

- 需要成对频谱并满足 ITU-R 和各国频谱方向规定；
- UE 和载荷需要双工器及相应射频隔离；
- 上下行载频不同，需要分别预测和补偿多普勒；
- 频谱配置可能限制带宽和业务方向的灵活性。

所以最终 FDD/TDD 选择首先受频谱法规约束。法规允许时，HAPS 和 LEO 可继续评估 TDD；GEO 和 MEO 通常因保护时间过大而缺乏效率。

> **关键原文定位：**TR 38.811 Clause 7.3.6.1。\(T_{\mathrm{guard}}\approx2\tau\) 是报告采用的传播级保守估计，不是包含处理与调度的完整协议 RTT。

## 4 大波束覆盖下的随机接入

卫星波束可覆盖数百公里，同一波束内不同 UE 的斜距不再近似相同。随机接入面对的核心矛盾不是“卫星绝对时延太大”这一句话，而是三种时延量分别落到不同机制：可预测的公共时延决定监测起点，波束内差分与残余时延决定 PRACH 搜索范围和 TA 范围，完整上行、处理、下行时序决定 RAR 到达时间。

| 接入环节 | 应比较的物理量 | 机制边界 |
|---|---|---|
| Msg1检测 | 残余差分时延、残余CFO与PRACH CP/搜索窗 | 前导能否被无歧义检测 |
| Msg2监测 | \(\tau_{\mathrm{UL}}+T_{\mathrm{proc}}+\tau_{\mathrm{DL}}\) 与RAR监测时序 | UE何时开始监听、窗口需覆盖多大不确定性 |
| 上行对齐 | 波束内差分距离与RAR TA范围 | 公共绝对时延需否单独处理、波束能覆盖多大 |

### 4.1 公共时延、差分时延与残余时延

对同一波束中的第 \(i\) 个 UE，可将 UE 到 gNB 的传播时延写成：

\[
\tau_i
=
\tau_{\mathrm{common}}
+
\Delta\tau_i.
\]

公共时延来自选定参考点到网络端的共同传播路径；差分时延由 UE 在波束内的位置引起。若网络或 UE 能利用卫星星历、参考点和位置辅助处理 \(\tau_{\mathrm{common}}\)，PRACH 和传统 TA 主要面对 \(\Delta\tau_i\)。

这种分解解释了一个常见误区：卫星高度可能达到数百至数万公里，但现有 TA 范围在公共时延被单独处理后约束的是**波束内差分距离和波束尺寸**，不是直接约束卫星高度。

TR 以 200 km 小区半径评估差分距离 \(d_3=d_2-d_1\)：

| 最低仰角 | 参考差分距离 |
|---:|---:|
| 10° | 390 km |
| 20° | 372 km |
| 40° | 303 km |
| 60° | 197 km |
| 80° | 67 km |

最低仰角越低，斜距几何越倾斜，同样地面半径对应的传播距离差越大。报告据此指出：在低于约 60° 的部分卫星条件下，差分范围可能超过其用于比较的现有 NR PRACH 覆盖量级，需要预补偿或进一步研究前导和过程增强。

> **关键原文定位：**TR 38.811 Clause 7.3.4.1.1、Figure 7.3.4.1.1-1、Table 7.3.4.1.2-1。

### 4.2 PRACH 检测、RAR 窗口与 TA

物理随机接入信道（Physical Random Access Channel，PRACH）、随机接入响应（Random Access Response，RAR）窗口和 RAR 中的 TA 处理同一接入过程的不同环节：

| 环节 | 解决的问题 | 主要受什么约束 |
|---|---|---|
| PRACH前导与检测 | gNB能否在残余时延和频偏范围内检测Msg1 | 前导CP、序列相关、搜索窗、残余CFO |
| RAR窗口 | UE在何时监听并等待Msg2 | 上下行传播时延、gNB检测与调度时延、UE监测功耗 |
| RAR中的TA | gNB如何把测得的上行到达偏差告诉UE | TA命令范围和粒度、公共时延参考、差分距离 |

三者不能互相替代：延长 PRACH CP 不会自动延长 UE 等待 Msg2 的窗口；延长 RAR 窗口也不会扩大前导的无歧义时延范围；TA 命令只有在 Msg1 已被检测并收到 RAR 后才生效。

PRACH 的判断可按残余差分时延与最大前导 CP 比较：

\[
\Delta\tau_{\mathrm{res}}
\le
T_{\mathrm{CP,PRACH,max}}
\quad\Rightarrow\quad
\text{现有格式可能适用},
\]

\[
\Delta\tau_{\mathrm{res}}
>
T_{\mathrm{CP,PRACH,max}}
\quad\Rightarrow\quad
\text{需要预补偿、扩大搜索或格式增强}.
\]

TR 的标准状态判断是：如果 UE 可使用 GNSS 类位置辅助，报告未识别出 PRACH 波形/格式的必然规范影响；若位置辅助不可用，波形或格式增强可能需要研究。无论 GNSS 是否可用，长 RTT 下的 RACH 过程和 RAR 监测时序仍可能需要增强。

> **关键原文定位：**TR 38.811 Clauses 7.3.4.1.2-7.3.4.1.3。

### 4.3 RAR 响应中的上下行传播时延

从 UE 发送 PRACH 到 UE 收到 RAR 的时间可以写成：

\[
\begin{aligned}
t_{\mathrm{RAR,rx}}-t_{\mathrm{PRACH,tx}}
={}&
\tau_{\mathrm{UL}}(t_0)
+T_{\mathrm{detect}}
+T_{\mathrm{schedule}}\\
&+\tau_{\mathrm{DL}}(t_1),
\end{aligned}
\]

其中 \(t_1>t_0\)。在静止、同路且近似互易的链路中：

\[
\tau_{\mathrm{UL}}
\approx
\tau_{\mathrm{DL}}
\approx
\tau,
\]

于是接近 \(2\tau+T_{\mathrm{detect}}+T_{\mathrm{schedule}}\)。但上下行时延在概念上必须分开，因为：

- LEO 在检测和调度期间继续运动，返回 RAR 时斜距已改变；
- 上下行可能采用不同波束、网关或馈电路径；
- 不同载频可能产生少量不同的电离层群时延；
- 上下行设备链路还可能包含不同群时延，但应与纯传播时延分开记账。

透明载荷下：

\[
\tau_{\mathrm{UL}}
=
\tau_{\mathrm{UE\rightarrow sat}}
+
\tau_{\mathrm{sat\rightarrow gateway}},
\]

下行走反方向；星上 gNB 则主要只包含 service link。RAR 窗口需要覆盖的是上下行时延之和、处理调度和剩余不确定性，而不是简单等于一个固定的 \(2h/c\)。

如果只把现有 RAR window 无条件拉长到数百毫秒，UE 会在大量不可能收到响应的时段持续监测 PDCCH，增加功耗。更合理的方向是利用已知公共传播时延延后监测起点，再用窗口长度覆盖处理抖动和差分不确定性。

> **关键原文定位：**TR 38.811 Clause 7.3.4.1.2。GEO弯管场景的 RAR 往返量级可达约600 ms，报告因此认为 RAR窗口及其节能设计需重新评估。

### 4.4 TA 范围与波束尺寸

TR 38.811 Table 7.3.4.2.1-1 给出的参考最大链路距离随 SCS 增大而缩小：

| SCS | 报告参考最大链路距离 |
|---:|---:|
| 15 kHz | 300 km |
| 30 kHz | 150 km |
| 60 kHz | 75 km |
| 120 kHz | 37.5 km |
| 240 kHz | 18.75 km |

如果让 RAR 中的传统 TA 直接承担全部卫星绝对时延，即使 15 kHz 对 LEO 和 GEO 也不够。因此报告考虑第二种结构：公共传播时延由网络另行处理，RAR TA 只表示 UE 相对参考点的差分。

在最低仰角 \(10^\circ\)、SCS 为 15 kHz、允许最大差分距离 300 km 时，TR 给出的最大波束直径为：

| 平台 | 最大波束直径 |
|---|---:|
| GEO 35,786 km | 304.66 km |
| MEO 10,000 km | 304.73 km |
| LEO 1,500 km | 305.05 km |
| LEO 600 km | 305.50 km |

这些数值几乎相同，说明当卫星高度明显大于波束尺寸时，差分距离主要由波束直径和最低仰角决定，而不再强烈依赖卫星高度。

因此 RAR TA 的正确理解是：

\[
\boxed{
\text{公共绝对时延由辅助机制处理}
+
\text{RAR TA描述波束内剩余差分}
}
\]

> **关键原文定位：**TR 38.811 Clauses 7.3.4.2.1-7.3.4.2.3，Tables 7.3.4.2.1-1、7.3.4.2.2-1。表格是报告的参考评估，具体 Rel-17/后续 NTN TA 机制应以相应 TS 版本为准。

## 5 局部多径、信道估计与循环前缀

卫星高度决定绝对传播时延，地面附近的反射与散射则形成纳秒至微秒量级的超额时延。后者使频域信道起伏，并可能造成相邻 OFDM 符号干扰。这里有两条不同的适配判据：DM-RS 频域间隔应小于信道的相干带宽，以便完成频域插值；CP 应长于可见多径的最大超额时延与残余定时误差。绝对传播时延由同步、公共时延补偿和 TA 处理，不直接决定 DM-RS 频域密度或 CP 长度。

| 物理量关系 | 对应机制 | 判断目标 |
|---|---|---|
| \(\Delta f_{\mathrm{pilot}}\) 与 \(B_c\) | DM-RS配置和SCS | 导频之间能否可靠插值信道 |
| \(T_{\mathrm{CP}}\) 与 \(\tau_{\mathrm{excess,max}}+T_{\mathrm{timing,res}}\) | 循环前缀 | 是否避免ISI并保持循环卷积条件 |

### 5.1 DM-RS 时间跟踪与频域插值

DM-RS 的两个密度维度解决不同问题：

- 时间密度：跟踪多普勒变化、残余 CFO 和符号间信道变化；
- 频域密度：在频率选择性信道上插值 \(H(f)\)。

Clause 7.3.2.4 的结论是“短时间内多普勒变化不足以修改时间位置”；Clause 7.3.5.1 则需要把频域导频间隔与相干带宽比较。两者不能混为“DM-RS整体密度足够”。

### 5.2 DM-RS 频域配置 Type 1 与 Type 2

Type 1 和 Type 2 是 PDSCH/PUSCH DM-RS 的**频域配置类型**，不是 SCS 类型。以一个 PRB 的 12 个子载波和一个 CDM 组为例：

| 配置 | 典型子载波位置 | 最大相邻采样间隔 | 特点 |
|---|---|---:|---|
| Type 1 | \(0,2,4,6,8,10\) | \(2\Delta f_{\mathrm{SCS}}\) | 梳状均匀采样，适合较强频率选择性 |
| Type 2 | \((0,1),(6,7)\) | \(5\Delta f_{\mathrm{SCS}}\) | 成对RE结构，可组织更多CDM组与端口 |

Type 2 不是严格“每 5 个子载波放一个导频”。\((0,1)\) 与 \((6,7)\) 之间的最大相邻频差为：

\[
(6-1)\Delta f_{\mathrm{SCS}}
=
5\Delta f_{\mathrm{SCS}}.
\]

在 15 kHz SCS 下：

\[
\Delta f_{\mathrm{pilot,type1}}
=30\,\mathrm{kHz},
\qquad
\Delta f_{\mathrm{pilot,type2}}
=75\,\mathrm{kHz}.
\]

在单符号/双符号 DM-RS 的典型最大端口能力上，Type 1 可支持 4/8 端口，Type 2 可支持 6/12 端口。Type 2 以更大的最大无导频间隔换取更灵活的 CDM 组和多端口组织。

PBCH 和 PDCCH 的 DM-RS 约每 4 个子载波出现一次，对应物理间隔约为：

\[
\Delta f_{\mathrm{pilot}}
=
4\Delta f_{\mathrm{SCS}}.
\]

> **关键原文定位：**TR 38.811 Clause 7.3.5.1.1、Tables 7.3.5.1.1-1 至 7.3.5.1.1-2；精确RE和端口映射参见 TS 38.211 Clause 7.4.1.1.2。

### 5.3 时延扩展与相干带宽

TR 使用下式进行保守比较：

\[
B_c
=
\frac{1}{\alpha\tau_{\mathrm{DS}}},
\]

其中 \(B_c\) 是相干带宽，\(\tau_{\mathrm{DS}}\) 是最大时延扩展的评估量，\(\alpha\) 取 1-50；报告为保守选择取 \(\alpha=50\)。例如：

\[
\tau_{\mathrm{DS}}=100\,\mathrm{ns}
\quad\Rightarrow\quad
B_c
=
\frac{1}{50\times100\times10^{-9}}
=200\,\mathrm{kHz}.
\]

| 场景 | 报告最大时延扩展 | 按保守公式得到的相干带宽 |
|---|---:|---:|
| D1 GEO Ka | 10 ns | 约2 MHz，MHz量级 |
| D2 GEO S | 100 ns | 200 kHz |
| D3 LEO S | 100 ns | 200 kHz |
| D4 LEO Ka | 10 ns | 约2 MHz，MHz量级 |
| D5 HAPS S | 150 ns | 133 kHz |

Ka 波段终端通常使用高增益窄波束天线，许多大角度、长超额时延散射路径会被方向图滤除，因此有效时延扩展较小。S 波段手持终端更接近宽波束，会接收更多局部散射路径。

报告还指出，有效时延扩展与最低工作 SNR 有关。弱的晚到路径可能低于噪声底，物理上虽然存在，却不会进入接收机可见的有效信道模型。这也是 S 波段卫星场景与 HAPS 参考时延扩展不同的原因之一。

### 5.4 SCS 与 DM-RS 频域采样

保守的频域采样判据为：

\[
\Delta f_{\mathrm{pilot}}
\lesssim
B_c.
\]

以 HAPS S 波段 \(B_c\approx133\,\mathrm{kHz}\) 为例：

| SCS与配置 | 最大导频间隔 | 保守判断 |
|---|---:|---|
| 30 kHz，Type 1 | 60 kHz | 较安全 |
| 30 kHz，Type 2 | 150 kHz | 略超出 |
| 60 kHz，Type 1 | 120 kHz | 接近边界 |
| 60 kHz，Type 2 | 300 kHz | 不满足该保守判据 |

卫星 S 波段 \(B_c\approx200\,\mathrm{kHz}\) 时，30 kHz SCS 的两种配置均可满足该初步比较；60 kHz SCS 下 Type 1 较合适，而 Type 2 可能不足；120 kHz SCS 需要更谨慎。Ka 波段相干带宽达到 MHz 量级，现有 DM-RS 频域采样通常更充足。

因此 TR 的结论不是“所有配置的 DM-RS 频域密度都足够”，而是：可以通过限制 SCS 和选择 DM-RS 配置适配 NTN 信道，所以未必需要修改 DM-RS 规范。

还有一个重要边界：Table 7.3.5.1.1-1 明确采用“assumed no orthogonally required”的简化假设，主要比较采样间隔与相干带宽，没有完整证明：

- 所有多端口 CDM/OCC 配置都稳定；
- 频率选择性不会破坏频域正交码；
- 大残余 CFO 下仍能正常解扩。

所以该结论偏向单端口或理想正交条件，多端口、频率选择性和残余 CFO 的联合接收仍需单独评估。

> **关键原文定位：**TR 38.811 Clauses 7.3.5.1.1-7.3.5.1.3、Table 7.3.5.1.1-3。

### 5.5 绝对传播时延与超额时延

GEO 的绝对传播时延可达数百毫秒，但不需要数百毫秒的 OFDM 循环前缀（Cyclic Prefix，CP）。信道可写成：

\[
h(t)
=
\sum_{\ell}
a_\ell
\delta\!\left(t-\tau_0-\Delta\tau_\ell\right),
\]

其中 \(\tau_0\) 是直达径的绝对传播时延，\(\Delta\tau_\ell\) 是第 \(\ell\) 条多径相对于直达径的超额时延。同步和 TA 处理 \(\tau_0\)，CP 只需要覆盖：

\[
T_{\mathrm{CP}}
\gtrsim
\max_\ell\Delta\tau_\ell
+
T_{\mathrm{timing,res}}.
\]

因此 CP 由局部多径和残余定时误差决定，而不是由卫星高度决定。

TR 给出的传播数量级为：

- 2 GHz 卫星信道时延扩展约 180-250 ns，250 ns 可覆盖约 90% 情况；
- 40 GHz、全向天线测得相干带宽约 30 MHz；
- 按 \(B_c\approx1/(5T_m)\) 换算，最大时延扩展约 25 ns；
- Ka 波段使用定向天线后，长时延回波被方向图抑制，可近似平坦衰落。

### 5.6 现有 CP 的覆盖能力与开销

| SCS | 正常CP代表长度 |
|---:|---:|
| 15 kHz | 4.688 μs |
| 30 kHz | 2.344 μs |
| 60 kHz | 1.172 μs |
| 120 kHz | 0.586 μs |
| 240 kHz | 0.293 μs |

即使 S 波段按 250 ns 时延扩展考虑，现有正常 CP 基本能够覆盖；Ka 波段时延扩展更短，裕量更大。60 kHz 的扩展 CP 约 16.67 μs，对卫星信道明显过长。

15 kHz SCS 的有效 OFDM 符号长度为：

\[
T_u
=
\frac{1}{15\,\mathrm{kHz}}
=66.67\,\mu\mathrm{s}.
\]

若 NTN 实际只需要约 0.25 μs，而正常 CP 约 4.688 μs，TR 用下式估计过度配置造成的额外开销：

\[
\frac{4.688-0.25}{66.67}
\approx6.7\%.
\]

严格按含 CP 的完整符号计算，正常 CP 自身占比约为：

\[
\frac{4.688}{66.67+4.688}
\approx6.57\%.
\]

虽然较长 CP 会牺牲少量频谱效率，但为 NTN 新增一套短 CP 会增加 numerology、射频和协议复杂度。TR 因而认为现有 CP 与 NTN 时延扩展兼容，不预期修改 CP 规范。

这里还形成一个跨机制权衡：

\[
\boxed{
\begin{aligned}
\text{增大SCS}
&\Rightarrow
\text{归一化CFO减小、CP变短}\\
&\Rightarrow
\text{DM-RS绝对频率间隔增大}
\end{aligned}
}
\]

所以 SCS 不能只从多普勒一个维度选择。

> **关键原文定位：**TR 38.811 Clauses 7.3.5.2.1-7.3.5.2.3、Table 7.3.5.2.2-1。

## 6 星上载荷的相位噪声与功放约束

卫星载荷不仅转发“理想基带信号”。本振与上下变频链会叠加相位噪声，有限电源与散热条件又推动功放靠近饱和区工作。前者需要比较符号间公共相位变化、符号内 ICI 与参考信号跟踪能力；后者需要比较波形 PAPR、允许 OBO、有效 EIRP 和容量。两类问题都来自星上硬件，但一个作用于相位稳定性，一个作用于功率效率。

| 硬件现象 | 关键比较量 | 主要机制 |
|---|---|---|
| 本振与频率转换相位噪声 | 残余CFO、CPE/ICI与DM-RS/PT-RS跟踪间隔 | 粗频偏补偿、信道估计和相位跟踪 |
| 功放接近饱和 | PAPR、OBO、非线性损失与有效EIRP | 波形选择、功率回退和PAPR降低 |

### 6.1 端到端相位误差模型

FFT 后的第 \(l\) 个 OFDM 符号、第 \(k\) 个子载波可近似写成：

\[
Y_{k,l}
=
e^{j\phi_l}
H_{k,l}X_{k,l}
+
I_{k,l}^{\mathrm{ICI}}
+
N_{k,l},
\]

其中：

- \(\phi_l\) 是符号级公共相位误差（Common Phase Error，CPE）；
- \(I_{k,l}^{\mathrm{ICI}}\) 是符号内快速相位变化引起的载波间干扰；
- \(H_{k,l}\) 是无线信道；
- \(N_{k,l}\) 是噪声。

相位误差可来自 UE、星上载荷和网关的振荡器相位噪声，残余载波频偏（Carrier Frequency Offset，CFO），多普勒预补偿误差，以及弯管载荷频率转换链中的漂移。

### 6.2 DM-RS 与 PT-RS 的分工

相位跟踪参考信号（Phase-Tracking Reference Signal，PT-RS）与 DM-RS 的分工是：

| 参考信号 | 主要任务 | 典型时间尺度 |
|---|---|---|
| DM-RS | 估计复信道的幅度、相位和空间层 | 一个或若干DM-RS位置之间的信道变化 |
| PT-RS | 跟踪符号级剩余公共相位误差 | 更密集的OFDM符号级相位变化 |

相位变化较温和时，DM-RS 的信道估计可以吸收一部分相位旋转；当 DM-RS 之间的相位变化过快，PT-RS 提供更密的符号级跟踪。

PT-RS 不能代替粗 CFO 捕获。如果残余频偏已经大到造成严重 ICI，正确流程是：

\[
\text{粗频偏估计与补偿}
\rightarrow
\text{FFT}
\rightarrow
\text{DM-RS信道估计}
\rightarrow
\text{PT-RS相位跟踪}.
\]

弯管载荷虽然不解调信号，仍会经历上行频率转换、星上本振、下行频率转换、放大和滤波，这些环节会把载荷相位噪声叠加到端到端链路中。

TR 对直达家庭（Direct-to-Home，DTH）、甚小口径终端（Very Small Aperture Terminal，VSAT）和典型弯管载荷的相位噪声模板进行评估。其结论条件为：

- 载频不高于约 30 GHz；
- 没有很大的残余 CFO 或未补偿多普勒；
- 信号带宽不是特别大；
- 星上相位噪声模板与已有蜂窝模型接近。

在这些条件下，现有 NR PT-RS 设计可能足以补偿。大残余 CFO、特殊星上本振模板、更高载频、超大带宽和多用户差分频偏仍需进一步研究。

> **关键原文定位：**TR 38.811 Clause 7.3.7.1、Annex B。报告的“现有设计可补偿”带有明确条件，不能扩展成所有 NTN 相位噪声与多普勒场景都无影响。

### 6.3 PAPR、功放非线性与 OBO

峰均功率比（Peak-to-Average Power Ratio，PAPR）定义为：

\[
\mathrm{PAPR}
=
\frac{\max_n|x[n]|^2}
{\mathbb E\{|x[n]|^2\}}.
\]

CP-OFDM 由大量子载波叠加。当多个子载波相位偶然对齐时会产生高瞬时峰值，因此 PAPR 高于单载波或 DFT-s-OFDM。

星上功率放大器（Power Amplifier，PA）接近饱和点工作时效率最高，但非线性会产生：

- 星座旋转、压缩和聚集；
- 带内失真和误差矢量幅度恶化；
- 带外泄漏；
- 多载波互调。

为保持线性，需要输出功率回退（Output Back-Off，OBO）：

\[
\mathrm{OBO}
=
10\log_{10}
\frac{P_{\mathrm{sat}}}{P_{\mathrm{out}}}.
\]

OBO 太小会造成严重非线性，OBO 太大又会损失有效输出功率和效率，因此总损失随 OBO 呈 U 形并存在最优工作点。

TR 引用的典型总损失为：

| 波形 | QPSK | 16-QAM |
|---|---:|---:|
| CP-OFDM | 约6 dB | 约7.6 dB |
| DFT-s-OFDM | 约4 dB | 约6 dB |

差异约 1.6-2 dB。2 dB 输出功率差对应：

\[
10^{-2/10}
\approx0.63,
\]

即相同功放下的有效输出功率可能只剩约 63%，下降约 37%。TR 给出的容量影响范围为 20%-40%，具体取决于系统更接近功率受限还是带宽受限；不能把 37% 功率下降直接等同于固定 37% 容量下降。

### 6.4 转发器工作方式与波形选择

PAPR 的系统影响取决于 PA 如何承载信号：

**多载波转发器**中，一个宽带转发器同时承载多个 FDM 载波。为抑制载波间互调，功放原本就需要较大回退，单个 NR CP-OFDM 波形带来的额外影响相对有限。

**单个宽带 NR 下行载波**中，面向小型 UE 的卫星可能让一个功放发送单个宽带 CP-OFDM 信号，并希望尽量靠近饱和点运行。高 PAPR 会迫使系统增加 OBO、缩小载波带宽或降低 MCS，从而减少波束容量。这是报告认为 PAPR 影响最显著的情况。

HAPS 往往采用固态功率放大器（Solid-State Power Amplifier，SSPA），电源约束可能不如卫星严苛，但仍存在同样的 PAPR-OBO-容量权衡。

DFT-s-OFDM 的主要意义是：

\[
\boxed{
\text{较低PAPR}
\rightarrow
\text{较小功率回退}
\rightarrow
\text{更高有效EIRP}
}
\]

这不表示 DFT-s-OFDM 天然具有更低频谱效率。在相同 RE、调制阶数和编码率下，其净数据承载能力可以接近 CP-OFDM；主要差异来自功放工作点和波形约束。TR 没有规定新的下行波形，只建议进一步研究 CP-OFDM 的 PAPR 降低技术，并指出上行 DFT-s-OFDM 可能有利。

> **关键原文定位：**TR 38.811 Clause 7.3.7.2、Figure 7.3.7.2.1-1。总损失数值来自报告引用的 ETSI TR 103 297 示例。

## 7 NTN 物理部署下的 NG-RAN 架构映射

架构问题的物理根源不是“卫星上必须采用一套新协议”，而是 gNB 的无线处理功能可以留在地面，也可以部分或全部放到星上。功能终止点一旦变化，HARQ、调度或核心网接口跨越的物理链路随之变化。需要比较的关键量是控制环路端点、service/feeder link 的 RTT 与误码率，以及星上能够承担的计算、存储、功耗和维护复杂度。

| 星上处理位置 | 跨越feeder link的主要对象 | 直接影响 |
|---|---|---|
| 透明弯管载荷 | NR-Uu射频波形及完整无线闭环 | 地面升级容易，但用户侧闭环最长 |
| gNB-DU上星 | F1控制面与用户面 | HARQ/快速调度缩短，CU-DU过程仍受馈电链路影响 |
| 完整gNB上星 | N2/N3及核心网方向业务 | 无线闭环最短，星上实现与运维最复杂 |

### 7.1 网络节点、物理链路与逻辑接口

标准 NG-RAN 可以把一个 gNB 拆分为分布单元（gNB Distributed Unit，gNB-DU）和集中单元（gNB Central Unit，gNB-CU）。简化的逻辑关系为：

\[
\text{UE}
\xleftrightarrow{\mathrm{NR-Uu}}
\left[
\text{gNB-DU}
\xleftrightarrow{\mathrm{F1}}
\text{gNB-CU}
\right]
\xleftrightarrow{\mathrm{N2/N3}}
\text{5GC}.
\]

这些名称表示协议栈中的逻辑参考点，不等于某一根固定物理电缆：

| 名称 | 连接端点 | 层面与主要作用 | NTN中的位置含义 |
|---|---|---|---|
| NR-Uu | UE与gNB | NR无线接口，承载接入层控制和用户数据；低层无线处理通常位于DU，高层协议位于CU | service link始终承载NR-Uu；透明载荷时同一射频波形还继续经feeder link到地面gNB |
| F1 | gNB-CU与gNB-DU | gNB内部逻辑接口；F1-C承载控制面，F1-U承载用户面 | DU上星、CU在地面时，F1需要跨feeder link |
| N2 | NG-RAN节点与5GC中的AMF | 控制面接口，承载接入、连接、移动性和UE上下文相关信令 | 完整gNB上星时，N2经feeder link连接地面核心网 |
| N3 | NG-RAN节点与5GC中的UPF | 用户面接口，通常以GTP-U承载用户数据包 | 完整gNB上星时，N3经feeder link把业务送往地面UPF |
| N1 | UE与5GC中的AMF | 非接入层（Non-Access Stratum，NAS）逻辑参考点；N1消息由Uu和N2转送，并非UE到AMF的一条独立物理链路 | 用于说明NAS信令端点；其物理路径随gNB位置变化 |

其中，接入与移动性管理功能（Access and Mobility Management Function，AMF）负责注册、连接和移动性控制；用户面功能（User Plane Function，UPF）负责用户数据转发。F1-C 常由 F1AP/SCTP/IP 承载，F1-U 常由 GTP-U/UDP/IP 承载；N2 通常使用 NGAP/SCTP/IP，N3 通常使用 GTP-U/UDP/IP。这里列出承载栈是为了说明后续定时器和误码影响，TR 38.811 的核心结论仍是保留这些逻辑参考点。

NTN 的两条主要物理无线链路为：

- 服务链路（service link）：UE 与卫星/HAPS 之间的无线链路；
- 馈电链路（feeder link）：卫星/HAPS 与地面网关之间的无线链路。

卫星无线接口（Satellite Radio Interface，SRI）表示 feeder link 上采用的卫星承载方式。SRI 可以承载 NR-Uu 射频波形、F1 或 N2/N3，但不会把这些逻辑接口替换成同一种协议。

因此，架构研究的实质是把 NR-Uu、F1、N1/N2/N3 映射到 service link、星上处理节点、feeder link 和地面网关，而不是重新创造一套 NG-RAN 参考点。

### 7.2 三种载荷映射

| 映射 | 星上功能 | Service link | Feeder link承载 | 地面主要节点 |
|---|---|---|---|---|
| 1 弯管载荷 | 频率转换、模拟滤波、放大 | NR-Uu | NR-Uu射频波形 | 完整gNB |
| 2 gNB-DU再生载荷 | 部分gNB-DU功能 | NR-Uu | F1 over SRI | gNB-CU |
| 3 完整gNB再生载荷 | 完整gNB | NR-Uu | N2/N3 over SRI，并转送N1 NAS | 5GC接入侧 |

卫星无线接口（Satellite Radio Interface，SRI）在这里表示馈电链路的卫星承载方式。它不是用来替代 NR-Uu、F1 或 NG 的全新逻辑接口。

**映射1：弯管载荷。** 卫星只做射频转换、滤波、放大和转发。星上复杂度低，地面 gNB 易升级，但 HARQ、调度和无线闭环需要同时跨 service link 与 feeder link，传播时延最长，且网关可见性直接影响服务。

**映射2：gNB-DU上星。** 星上承载 DU 侧功能，HARQ 和快速调度可在 UE-星上 DU 之间终止，用户侧闭环不再完整穿过馈电链路。F1 仍经 feeder link 到地面 CU，因而 CU/DU 控制、PDCP-RLC 数据传输和 F1 定时器仍受长时延与误码影响。具体星上功能还取决于采用的 CU/DU 切分。

**映射3：完整gNB上星。** 星上部署完整 gNB，馈电链路直接承载核心网方向的 N2/N3，并转送 UE-AMF 之间的 N1 NAS 消息。无线控制环路更短，可在星上进行本地资源管理，但星上计算、存储、功耗、软件升级、安全上下文和故障恢复压力最大。

三种映射的本质权衡为：

\[
\boxed{
\text{星上处理越多}
\Rightarrow
\text{无线闭环越短，载荷实现越复杂}
}
\]

> **关键原文定位：**TR 38.811 Clause 7.3.8.1.2，Figures 7.3.8.1.2-1 至 7.3.8.1.2-3，Tables 7.3.8.1.2-1 至 7.3.8.1.2-2。

### 7.3 接口定时器与误码

逻辑接口可以保留，但承载这些接口的协议参数必须覆盖 feeder/service link 的长时延和更高误码率。可用下式表示定时器下界：

\[
T_{\mathrm{interface}}
>
T_{\mathrm{RTT,link}}
+
T_{\mathrm{processing}}
+
T_{\mathrm{retransmission\ margin}}.
\]

受影响对象包括 F1-C/F1-U、N1、N2、N3、GTP-U、SCTP 及其确认和重传定时器。若继续使用地面网络时延范围，消息可能在确认尚未返回前被误判超时。

TR 的结论是：通过合适映射，不需要修改 NG-RAN 的逻辑参考点、接口集合或基本功能；但 F1、N1/N2/N3 等承载协议可能需要适应 NTN 的长时延和 BER。

### 7.4 平台运动与移动性来源

NTN 移动性包含三类来源：

1. 平台运动使波束扫过静止 UE；
2. UE 自身从一个波束移动到另一个波束；
3. UE 在不同卫星、HAPS 和地面蜂窝系统之间移动。

GEO 对地固定波束的小区图样基本稳定，Tracking Area 可与地面覆盖对应，主要问题是移动性定时器延长。NGSO 的移动波束会使静止 UE 也经历重选、切换、寻呼区域变化和网络标识映射；对地固定波束虽能稳定 footprint，同一地理区域仍需在不同卫星间交接。

HAPS 若用一个 Tracking Area 覆盖完整服务区，影响较小；若服务区被划分成多个更小 Tracking Area，平台移动或旋转会出现类似 NGSO 的映射问题。

因此波束、小区、Tracking Area、卫星轨道和网络标识需要联合设计，不能简单把每个物理波束永久等同于一个地面小区。

> **关键原文定位：**TR 38.811 Clauses 7.3.8.1.1-7.3.8.1.3、Table 7.3.8.1.2-3。

## 8 跨机制适配准则与标准影响

前面各章分别讨论移动、时延、波束、多径、硬件和架构，本章只做横向收敛。共同判断方法是先识别可预测的几何公共量，再确定差分量和残余误差，最后把它们与相应机制的采样周期、反馈周期、范围或硬件裕量比较。不同功能看似分散，实际反复落在确定性公共量、剩余动态误差和星上资源约束三类根因上。

### 8.1 机制边界矩阵

| 物理问题 | 正确比较量 | 主要机制 | TR阶段的判断 |
|---|---|---|---|
| 大原始多普勒 | 预补偿后的残余频偏与同步捕获范围 | 星历辅助、PSS/SSS捕获、频偏估计 | 条件不足时需进一步研究 |
| 多普勒变化率 | \(\dot f_D T_{\mathrm{mech}}\) | DM-RS、预测与跟踪 | 时隙内无需改DM-RS时间位置，长闭环仍需考虑老化 |
| 大绝对RTT | 反馈周期与时隙/定时器 | HARQ、MAC/RLC | 需扩展、限制或关闭部分HARQ，并重定参数范围 |
| CQI/功控老化 | 闭环时间与信道变化时间 | ACM、功控、MCS裕量 | 多为实现调整，额外裕量待研究 |
| 波束内差分时延 | 残差与PRACH CP、TA范围 | PRACH、RAR、TA | GNSS条件影响格式结论，RACH过程仍可能增强 |
| 频率选择性 | 导频间隔与相干带宽 | DM-RS Type 1/2、SCS | 可通过配置约束适配，未必改规范 |
| 多径超额时延 | 时延扩展与CP | 正常/扩展CP | 现有CP足够，无预期规范增强 |
| TDD传播保护 | DL+UL传播时间 | Guard period、FDD/TDD | 多数NTN优选FDD，LEO/HAPS可继续评估TDD |
| 星上相位误差 | 残余CFO、CPE、ICI与PT-RS密度 | DM-RS、PT-RS | 典型30 GHz以下条件现有设计可能足够 |
| 高PAPR | OBO、非线性和有效EIRP | DFT-s-OFDM、PAPR降低 | 不限制NR可用性，但可能显著损失容量 |
| 星上功能位置 | 无线闭环与接口跨越的链路 | gNB-DU/CU映射 | 逻辑NG-RAN不必改变，承载参数和移动性可能适配 |

> **关键原文定位：**TR 38.811 Clauses 7.3.2-7.3.8 分别讨论平台运动、长传播时延、大波束、传播信道、双工、星上硬件和 NG-RAN 架构映射；本表用于横向比较，不改变各小节的标准状态判断。
