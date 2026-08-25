# 3GPP TR 38.821 NTN 时序、功率控制与波束资源管理

> 学习笔记 | 版本 v2 | 2026-08-15  
> 主要标准基线：3GPP TR 38.821 V16.2.0，辅以 TS 38.214、TS 38.213、TS 38.331 与 TS 38.300 Release 16

本笔记按四条相互衔接的技术主线组织：跨 DL-UL 时序、测量与功率控制、NTN 波束管理，以及波束与频域和小区资源的映射。TR 38.821 是研究报告，其中的时序偏移、自适应滤波和波束-BWP 映射等内容包含研究阶段候选方案，不能直接视为已经采用的规范规则。

$$
\text{长传播时延}
\rightarrow
\text{时序修正}
\rightarrow
\text{测量与功控}
\rightarrow
\text{波束、BWP与小区管理}
$$

## 1. NTN 跨 DL-UL 时序

### 1.1 TA 与逻辑时序偏移

NTN 的长传播时延使 UE 需要采用较大的定时提前量（Timing Advance，TA）。TA 的作用是让 UE 提前发送上行波形，使不同 UE 的信号在 gNB 接收端对齐。它修正的是物理信号到达时刻，并不直接保证调度关系具有因果性。

若下行接收时间线和上行发射时间线仍使用相近的 slot 编号，大 TA 可能导致 UE 收到下行命令时，命令所指向的上行 slot 已经过期。TR 38.821 因此讨论在跨 DL-UL 关系中引入额外逻辑偏移 $K_{\mathrm{offset}}$：

| 参数 | 作用层面 | 主要目的 |
|---|---|---|
| TA | 上行波形的绝对发射时刻 | 使信号在 gNB 接收端对齐 |
| $K_{\mathrm{offset}}$ | DL 与 UL 之间的逻辑 slot 映射 | 保证 DL 命令之后仍存在可用的未来 UL 资源 |

两类时序修正方向可概括为：

$$
\text{DL命令}\rightarrow\text{未来UL动作：加 }K_{\mathrm{offset}}
$$

$$
\text{UL报告}\rightarrow\text{过去DL参考：减 }K_{\mathrm{offset}}
$$

不同过程可以采用不同偏移，报告也讨论过 per-beam 或 per-cell 的取值方式。具体取值、广播或专用信令方式在 TR 38.821 V16.2.0 中仍属于研究事项。

参见 TR 38.821 Clause 6.2.1、Figure 6.2.1-1 和 Figure 6.2.1-2。

### 1.2 受影响的时序关系

| 过程 | 基准关系或起点 | NTN 修正方向 |
|---|---|---|
| RAR 调度 Msg3 PUSCH | Msg2 RAR 中的 UL Grant | PUSCH slot 加偏移 |
| HARQ-ACK | PDSCH 到 PUCCH 的 $K_1$ 关系 | ACK/NACK slot 加偏移 |
| MAC CE 生效 | HARQ-ACK 后等待处理时间 | 延长并对齐配置生效时刻 |
| CSI on PUSCH | DCI 调度未来 PUSCH | 继承 PUSCH 的时序偏移 |
| CSI reference resource | 从 UL 上报反推 DL 参考资源 | DL 参考 slot 减偏移 |
| 非周期 SRS | DL DCI 触发未来 UL SRS | SRS slot 加偏移 |

随机接入中，若承载 RAR 的 PDSCH 在 slot $n$ 结束，地面 NR 的简化关系为：

$$
n_{\mathrm{Msg3}}=n+K_2+\Delta
$$

$K_2$ 由时间域资源分配确定，$\Delta$ 是首次 Msg3 PUSCH 的附加处理时间。NTN 候选形式为：

$$
n_{\mathrm{Msg3}}=n+K_2+\Delta+K_{\mathrm{offset}}
$$

RAR 发生在初始接入阶段，UE 尚未建立完整专用 RRC 上下文，因此偏移必须通过 UE 与网络都能预先理解的方式获得。

对于 HARQ-ACK，基准关系和候选增强分别为：

$$
n_{\mathrm{ACK}}=n+K_1
$$

$$
n_{\mathrm{ACK}}=n+K_1+K_{\mathrm{offset}}
$$

该偏移只保证 ACK 在译码后仍可发送，不能消除 ACK 传播、gNB 决策和重传共同形成的长 HARQ 往返时延。

CSI reference resource 的方向相反。设 CSI 上报所在 UL slot 换算后的 DL 时刻为 $n$：

$$
n_{\mathrm{resource}}
=n-n_{\mathrm{CSI,ref}}-K_{\mathrm{offset}}
$$

这里先减去 DL-UL 时间线偏移，再向过去寻找报告所描述的 DL 参考资源。CSI 在 PUSCH 上的实际发送时刻和非周期 SRS 则属于“DL 命令触发未来 UL 动作”，因此采用加偏移方向。

纯下行关系不必统一增加偏移。例如 PDCCH 调度 PDSCH 只在 DL 时间线上定义，不直接受 UE 的 DL-UL 时间线偏移影响。

参见 TR 38.821 Clause 6.2.1；TS 38.214 Clause 5.2、6.1.2.1.1 和 6.2.1；TS 38.213 Clause 9。

## 2. 测量滤波与上行功率控制

### 2.1 路径损耗与功率控制链

UE 可根据同步信号块（Synchronization Signal Block，SSB）或信道状态信息参考信号（Channel State Information Reference Signal，CSI-RS）的参考信号接收功率（Reference Signal Received Power，RSRP）估计路径损耗：

$$
\widehat{PL}
=P_{\mathrm{RS,tx}}-\mathrm{RSRP}_{\mathrm{filtered}}
$$

$P_{\mathrm{RS,tx}}$ 是网络提供的参考信号发射功率，$\mathrm{RSRP}_{\mathrm{filtered}}$ 是高层滤波后的测量值。PUSCH 功率控制可用下列简化结构理解：

$$
P_{\mathrm{PUSCH}}
=\min\left\{
P_{\mathrm{CMAX}},
P_0+10\log_{10}(2^\mu M_{\mathrm{RB}})
+\alpha\widehat{PL}
+\Delta_{\mathrm{TF}}+f_{\mathrm{CL}}
\right\}
$$

其中 $P_0$ 是目标功率基准，$M_{\mathrm{RB}}$ 是资源块数量，$\alpha$ 是路径损耗补偿因子，$\Delta_{\mathrm{TF}}$ 是传输格式修正，$f_{\mathrm{CL}}$ 是闭环功控项。由此可见，RSRP 滤波会通过路径损耗估计进入上行功率控制。

参见 TS 38.213 Clause 7.1.1。

### 2.2 Layer 3 滤波与 NTN 适应性

TS 38.331 定义一阶指数滤波：

$$
F_i=(1-a)F_{i-1}+aM_i,
\qquad
a=\frac{1}{2^{k/4}}
$$

$M_i$ 是最新物理层测量，$F_{i-1}$ 是上一滤波值，$F_i$ 是更新后的 Layer 3 结果，$k$ 是 RRC 参数 `filterCoefficient`。典型权重为：

| $k$ | $a$ | 特性 |
|---:|---:|---|
| 0 | 1 | 不进行 Layer 3 平滑 |
| 4 | $1/2$ | 跟踪较快 |
| 8 | $1/4$ | 平滑较强 |
| 16 | $1/16$ | 抗噪更强，但响应较慢 |

例如 $F_{i-1}=-90$ dBm、$M_i=-98$ dBm：当 $k=4$ 时得到 $F_i=-94$ dBm；当 $k=8$ 时得到 $F_i=-92$ dBm。后者更平滑，但真实 RSRP 已下降 8 dB，滤波结果只下降 2 dB，会暂时低估路径损耗。若变化主要来自噪声，过弱平滑又会使 PUSCH 功率频繁抖动。

TR 38.821 Clause 6.2.2 讨论过由网络配置多个 Layer 3 滤波系数、UE 根据 RSRP 自适应选择的方案，也讨论过按波束配置功控参数、利用星历预测功率和关闭闭环功控等方法，但没有收敛为统一增强机制。研究结论是以 NR Release 15 功率控制方案作为 NTN 基线。

协议层次关系可简化为：

$$
\text{SSB/CSI-RS}
\rightarrow M_i
\rightarrow F_i
\rightarrow \widehat{PL}
\rightarrow P_{\mathrm{PUSCH}}
$$

Layer 1 完成参考信号测量和物理功控；Layer 3 的 RRC 配置滤波系数并使用平滑结果；Layer 2 承担调度、HARQ、随机接入和 MAC CE 等功能，并不存在通用的 Layer 2 RSRP 滤波级。

参见 TR 38.821 Clause 6.2.2；TS 38.331 Clause 5.5.3.2。

## 3. NTN 波束管理

### 3.1 波束对象与动态来源

NTN 中需要区分三类相关但不等同的对象：

| 对象 | 含义 | 主要层面 |
|---|---|---|
| 卫星天线波束 | 阵列形成的物理主瓣和方向图 | 天线与射频 |
| 波束足迹 | 物理波束在地面的覆盖区域 | 地理空间 |
| NR 波束 | 与 SSB、CSI-RS、SRS 或 TCI 状态关联的收发方向 | PHY、MAC 与 RRC |

地面网络（Terrestrial Network，TN）中，基站固定，覆盖范围通常较稳定，波束动态主要来自 UE 移动、姿态和遮挡。低轨 NTN 中，即使 UE 静止，卫星高速运动也会使最佳波束持续变化。

| 维度 | TN | LEO NTN |
|---|---|---|
| 网络平台 | 基站固定 | 卫星高速运动 |
| 静止 UE 的最佳波束 | 通常较稳定 | 仍可能持续变化 |
| 波束足迹 | 通常固定 | 可对地移动或暂时对地固定 |
| 波束预测 | 主要依赖测量 | 可结合星历、位置和时间 |
| 闭环反馈 | RTT 较短 | 报告到达时可能已老化 |
| 切换联动 | 空间波束和 QCL 状态 | 还可能涉及 TA、多普勒、频率和极化 |

### 3.2 对地移动与对地固定波束

对地移动波束通常保持相对卫星平台的指向，地面足迹随卫星飞行扫过地面：

$$
\text{平台坐标系中方向近似不变}
\rightarrow
\text{地面足迹移动}
$$

对地固定波束通过电子或机械转向，使足迹在一段时间内保持在目标区域：

$$
\text{地面足迹暂时固定}
\rightarrow
\text{卫星阵列方向持续变化}
$$

因此“对地固定”不表示卫星侧波束控制静止。随着卫星离开可见区，服务仍需转交给下一物理波束或下一颗卫星。GEO 的几何关系更接近固定基站，但传播时延、覆盖面积和链路预算仍与 TN 显著不同。

参见 TR 38.821 Clause 7.4。

### 3.3 波束切换的联合约束

NTN 波束管理不仅是选择最大 RSRP 的方向，还要处理以下联动：

- **几何预测：**星历、UE 位置和时间可用于预测可见卫星、仰角与候选波束，减少完全依赖滞后测量的切换。
- **反馈老化：**测量报告经长传播时延到达网络时，原最佳波束可能已经变化，需要更早触发或预配置目标波束。
- **时频补偿：**不同波束可能采用不同参考点，切换时可能同时出现 $\Delta TA$、$\Delta f_{\mathrm{Doppler}}$ 和 $K_{\mathrm{offset}}$ 变化。
- **终端能力：**S 频段手持 UE 的终端波束较宽；Ka 频段 VSAT 或相控阵终端波束较窄，需要持续跟踪卫星。
- **频率与极化：**相邻波束可能使用不同频率颜色或 RHCP/LHCP，切换可能同时改变频率资源和极化模式。

由此，NTN 波束切换往往是空间方向、参考信号、TA、多普勒、频率资源和服务卫星的联合状态变化。

## 4. BWP、分量载波与小区映射

### 4.1 CC 与 BWP 的频域层级

分量载波（Component Carrier，CC）是载波聚合（Carrier Aggregation，CA）的组成单位。每个 CC 具有中心频率、载波带宽和资源块网格。带宽部分（Bandwidth Part，BWP）是一个 CC 内连续的频率工作窗口：

$$
\mathcal B_i\subseteq\mathcal C_m
$$

$\mathcal C_m$ 表示第 $m$ 个 CC，$\mathcal B_i$ 表示该 CC 内配置的第 $i$ 个 BWP。一个 BWP 不能跨越两个 CC。

| 维度 | BWP 切换 | CC/载波聚合 |
|---|---|---|
| 范围 | 同一个 CC 内 | 不同 CC 之间 |
| 作用 | 调整 UE 当前工作窗口 | 增加可同时使用的载波 |
| 是否增加载波数 | 否 | 是 |
| 同时工作关系 | 一个方向通常一个激活 BWP | 多个 CC 可同时激活 |

可以记为：CA 决定 UE 同时使用多少个载波；BWP 决定 UE 在每个载波内部实际监听和收发哪一部分。DL 与 UL BWP 在配对频谱中可以独立切换，在非配对频谱中同步切换。

TR 38.821 Clause 6.2.4 讨论了频率复用因子大于 1 时“一波束一 BWP”或“一波束一 CC”等候选映射，也提出用单条 DCI 同时切换 DL/UL BWP。它们是系统方案，不是 BWP 的固有定义，也未在该研究报告中收敛为唯一规则。

参见 TR 38.821 Clause 6.2.4；TS 38.300 Clause 5.1、6.10 和 7.8。

### 4.2 波束与小区的管理层级

波束描述空间收发方向，小区描述一套接入、广播、调度和移动性管理配置。一个小区可以包含多个 SSB 或 CSI-RS 波束，同一小区内的多个 SSB 波束具有相同 PCI 和基本系统信息，通过 SSB index 区分方向。UE 从一个 SSB 波束切到另一个波束，只要 PCI 和服务小区上下文不变，通常属于小区内波束切换。

| UE 观察到的主要变化 | 操作性质 |
|---|---|
| SSB/CSI-RS/TCI 状态改变，PCI 不变 | 小区内波束切换 |
| 激活 BWP 改变，服务小区不变 | BWP 切换 |
| SCell 激活或去激活 | CA/服务小区操作 |
| PCI、系统信息和 RRC 上下文改变 | 小区切换或重选 |
| 服务卫星改变，但小区身份保持 | 服务链路切换，不一定是小区切换 |

物理波束也可承载多个 CC、BWP 或频率层小区，因此物理波束、波束足迹和 NR 小区之间不存在固有的一一对应关系。

参见 TS 38.300 Clause 9.2.4。

### 4.3 NTN 小区映射与联合状态

NTN 中常见三种小区映射：

1. **多波束对应一个小区：**减少 RRC handover，但共同定时、多普勒参考、随机接入和频率复用管理更复杂。
2. **一个波束对应一个小区：**波束和 PCI 容易映射，但对地移动 LEO 波束可能引起频繁小区切换。
3. **对地固定逻辑小区由不同卫星接续：**网络保持逻辑小区身份，后台更换服务卫星、物理波束或馈电链路；能否对 UE 无感取决于时频连续性和网络架构。

一次 NTN 服务状态可抽象为：

$$
\mathcal S=
\{\mathrm{Cell},\mathrm{Satellite},\mathrm{Beam},\mathrm{CC},\mathrm{BWP},
\mathrm{TA},\mathrm{Doppler},K_{\mathrm{offset}}\}
$$

这些状态量不必同时变化。判断一次操作属于波束切换、BWP 切换、载波操作还是小区切换，关键不在地面覆盖图是否改变，而在 UE 的波束标识、PCI、RRC 上下文、服务载波和激活 BWP 是否改变。

全篇关系可归纳为：TA 修正波形到达时刻，$K_{\mathrm{offset}}$ 修正跨 DL-UL 逻辑时序；Layer 3 滤波通过路径损耗估计影响上行功率；波束属于空间层，BWP 和 CC 属于频域层，小区属于接入与移动性管理层。NTN 系统需要协调这些不同层次，不能把波束切换、BWP 切换和小区切换相互等同。

参见 TR 38.821 Clause 7.3.2、7.4 和 8.3。

**参考资料**

1. [3GPP TR 38.821 - Solutions for NR to support Non-Terrestrial Networks](https://www.3gpp.org/dynareport/38821.htm)，重点 Clause 6.2.1、6.2.2、6.2.4、7.3.2、7.4 和 8.3。
2. [ETSI TS 138 214 V16.7.0 - NR Physical layer procedures for data](https://www.etsi.org/deliver/etsi_ts/138200_138299/138214/16.07.00_60/ts_138214v160700p.pdf)，重点 Clause 5.2、6.1.2.1.1 和 6.2.1。
3. [ETSI TS 138 213 V16.3.0 - NR Physical layer procedures for control](https://www.etsi.org/deliver/etsi_ts/138200_138299/138213/16.03.00_60/ts_138213v160300p.pdf)，重点 Clause 7.1.1 和 Clause 9。
4. [ETSI TS 138 331 V16.18.0 - NR Radio Resource Control](https://www.etsi.org/deliver/etsi_ts/138300_138399/138331/16.18.00_60/ts_138331v161800p.pdf)，重点 Clause 5.5.3.2。
5. [ETSI TS 138 300 V16.17.0 - NR and NG-RAN Overall description](https://www.etsi.org/deliver/etsi_ts/138300_138399/138300/16.17.00_60/ts_138300v161700p.pdf)，重点 Clause 5.1、6.10、7.8 和 9.2.4。
