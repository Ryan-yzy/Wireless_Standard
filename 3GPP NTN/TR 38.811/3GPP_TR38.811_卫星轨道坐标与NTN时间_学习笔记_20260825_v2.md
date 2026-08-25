# 卫星轨道、坐标变换与 NTN 时间

> 本笔记面向已经具备无线通信基础、希望建立 NTN 几何计算能力的读者。全文按照“六根数确定卫星状态 → 坐标变换建立卫星—UE 几何 → 时间信息驱动轨道、时延和多普勒计算”的顺序组织。
>
> 本文采用满足 NTN 链路分析和一般系统级仿真的简化模型。高精度定轨所需的岁差、章动、极移、相对论修正和精密地球定向参数只说明其位置，不展开推导。

---

## 术语与简称

| 简称 | 英文全称 | 中文含义 | 本文中的作用 |
|---|---|---|---|
| ECI | Earth-Centered Inertial | 地心惯性坐标系 | 轨道传播和卫星状态表达 |
| ECEF | Earth-Centered, Earth-Fixed | 地心地固坐标系 | 地面 UE、网关及地理位置表达 |
| LLA | Latitude, Longitude, Altitude | 纬度、经度和高度 | UE、网关的常用位置输入 |
| ENU | East, North, Up | 东、北、天局部坐标系 | 计算 UE 观察卫星的方位角和仰角 |
| PQW | Perifocal Coordinate System | 轨道近焦坐标系 | 由轨道参数直接构造卫星位置和速度 |
| RAAN | Right Ascension of the Ascending Node | 升交点赤经 | 确定轨道平面在赤道面内的方向 |
| TLE | Two-Line Element Set | 两行轨道根数 | 实际卫星轨道数据的一种常见格式 |
| SGP4 | Simplified General Perturbations 4 | 简化通用摄动模型 4 | 与 TLE 配套的轨道传播模型 |
| UTC | Coordinated Universal Time | 协调世界时 | 时间戳、记录和对外时间表示 |
| TAI | International Atomic Time | 国际原子时 | 连续原子时间尺度 |
| UT1 | Universal Time 1 | 世界时 UT1 | 描述地球实际自转角 |
| GNSS | Global Navigation Satellite System | 全球导航卫星系统 | 为 UE 提供位置、绝对时间和辅助信息 |
| RTT | Round-Trip Time | 往返时延 | 衡量双向空间链路的传播时间 |
| LOS | Line of Sight | 视距方向 | 距离、仰角、多普勒和波束判定的基本方向 |
| Nadir | — | 星下方向、天底方向 | 从卫星指向地心附近的方向 |

---

## 1. 六根数与卫星状态

### 1.1 轨道、位置与状态的关系

“卫星在某条轨道上”和“卫星此刻位于哪里”是两个不同问题：

- 轨道的大小、形状和空间朝向决定卫星可能经过的空间曲线；
- 卫星在轨道上的相位决定卫星在给定时刻的具体位置；
- 卫星位置和速度共同构成该时刻的轨道状态。

因此，完整确定卫星状态至少需要：

1. 一组六根数，用来描述轨道和卫星在参考时刻的位置；
2. 轨道根数对应的历元 \(t_0\)；
3. 从历元传播到目标时刻 \(t\) 的轨道模型；
4. 明确的参考坐标系和时间尺度。

如果只有“轨道高度为 600 km”，只能近似确定圆轨道半径，不能确定轨道倾角、轨道平面方向以及卫星此刻所在的位置。

### 1.2 经典开普勒六根数

经典开普勒六根数可以写成：

\[
\left\{a_{\mathrm{orb}},e,i,\Omega,\omega,\nu_0\right\}
\]

其中：

| 轨道根数 | 名称 | 物理作用 |
|---|---|---|
| \(a_{\mathrm{orb}}\) | 半长轴 | 决定轨道尺度和轨道周期 |
| \(e\) | 偏心率 | 决定轨道形状，\(e=0\) 为圆轨道 |
| \(i\) | 轨道倾角 | 决定轨道平面相对赤道面的倾斜程度 |
| \(\Omega\) | 升交点赤经 | 决定升交线在赤道面内的方向 |
| \(\omega\) | 近地点幅角 | 决定近地点在轨道平面内的方向 |
| \(\nu_0\) | 历元处真近点角 | 决定卫星在历元 \(t_0\) 时位于轨道上的哪个位置 |

前三个量 \(a_{\mathrm{orb}},e,i\) 主要描述轨道的大小、形状和倾斜程度；\(\Omega,\omega\) 确定轨道在三维空间中的朝向；最后的 \(\nu_0\) 确定卫星在该轨道上的瞬时相位。

在轨道传播中，第六个量经常改用历元处的平均近点角 \(M_0\)：

\[
\left\{a_{\mathrm{orb}},e,i,\Omega,\omega,M_0;t_0\right\}
\]

这种写法更适合从 \(t_0\) 传播到任意时刻 \(t\)。因此，“六根数”并不意味着不需要历元；没有 \(t_0\)，就不能解释 \(M_0\) 或 \(\nu_0\) 对应的是哪个时刻。

### 1.3 六根数确定轨道的几何过程

可以将六根数的作用理解为连续完成四件事。

#### 1.3.1 确定轨道椭圆

半长轴和偏心率决定椭圆的大小与形状。半通径为：

\[
p=a_{\mathrm{orb}}\left(1-e^2\right)
\]

当卫星真近点角为 \(\nu\) 时，地心距为：

\[
r=\frac{p}{1+e\cos\nu}
\]

对于半径近似不变的圆轨道，\(e\approx 0\)，于是 \(r\approx a_{\mathrm{orb}}\)。若地球平均半径为 \(R_{\mathrm E}\)，圆轨道高度近似为：

\[
h\approx a_{\mathrm{orb}}-R_{\mathrm E}
\]

#### 1.3.2 确定轨道平面

轨道倾角 \(i\) 和升交点赤经 \(\Omega\) 共同确定轨道平面：

- \(i\) 表示轨道平面相对赤道面的倾斜角；
- \(\Omega\) 表示卫星从南向北穿过赤道时，升交点相对参考方向的位置。

仅给出倾角不能唯一确定轨道平面，因为具有相同倾角的轨道可以绕地轴旋转到不同方向。

#### 1.3.3 确定近地点方向

近地点幅角 \(\omega\) 从升交点开始，在轨道平面内量到近地点。它确定椭圆长轴在轨道平面中的方向。

对于严格圆轨道，近地点不存在唯一方向，此时 \(\omega\) 和 \(\nu\) 会退化。工程上常使用纬度幅角：

\[
u=\omega+\nu
\]

直接描述卫星相对升交点的轨道内角位置。

#### 1.3.4 确定卫星轨道相位

真近点角 \(\nu\) 或平均近点角 \(M\) 确定卫星当前位于轨道的哪个位置。前五个根数相同而第六个根数不同的两颗卫星，可以位于同一轨道的不同位置。

星座设计中的相位配置，本质上就是为多颗卫星安排不同的轨道平面和轨道内位置。

### 1.4 从历元传播到目标时刻

在二体模型下，平均角速度为：

\[
n=\sqrt{\frac{\mu}{a_{\mathrm{orb}}^3}}
\]

其中 \(\mu\) 为地球标准引力参数。目标时刻 \(t\) 的平均近点角为：

\[
M(t)=M_0+n(t-t_0)
\]

对椭圆轨道，平均近点角 \(M\)、偏近点角 \(E\) 和真近点角 \(\nu\) 之间满足：

\[
M=E-e\sin E
\]

\[
\nu=2\operatorname{atan2}\!\left(
\sqrt{1+e}\sin\frac{E}{2},
\sqrt{1-e}\cos\frac{E}{2}
\right)
\]

计算时通常先由 \(M\) 数值求解开普勒方程得到 \(E\)，再得到 \(\nu\) 和地心距 \(r\)。对于 NTN 基础仿真，牛顿迭代即可完成开普勒方程求解。

### 1.5 轨道近焦坐标系中的位置和速度

轨道近焦坐标系 PQW 固定在轨道平面内：

- \(P\) 轴指向近地点；
- \(Q\) 轴位于轨道平面内，与 \(P\) 轴正交；
- \(W\) 轴沿轨道角动量方向。

卫星在 PQW 坐标系中的位置为：

\[
\boldsymbol r_{\mathrm{PQW}}
=
\frac{p}{1+e\cos\nu}
\begin{bmatrix}
\cos\nu\\
\sin\nu\\
0
\end{bmatrix}
\]

速度为：

\[
\boldsymbol v_{\mathrm{PQW}}
=
\sqrt{\frac{\mu}{p}}
\begin{bmatrix}
-\sin\nu\\
e+\cos\nu\\
0
\end{bmatrix}
\]

这一步只在轨道自身的二维平面内计算，不涉及地球经纬度。

### 1.6 从 PQW 旋转到 ECI

将 PQW 中的状态旋转到地心惯性坐标系，可写为：

\[
\boldsymbol r_{\mathrm{ECI}}
=
\boldsymbol C_{\mathrm{ECI}\leftarrow\mathrm{PQW}}
\boldsymbol r_{\mathrm{PQW}}
\]

\[
\boldsymbol v_{\mathrm{ECI}}
=
\boldsymbol C_{\mathrm{ECI}\leftarrow\mathrm{PQW}}
\boldsymbol v_{\mathrm{PQW}}
\]

其中：

\[
\boldsymbol C_{\mathrm{ECI}\leftarrow\mathrm{PQW}}
=
\boldsymbol R_3(\Omega)
\boldsymbol R_1(i)
\boldsymbol R_3(\omega)
\]

旋转顺序对应：先在轨道平面内确定近地点方向，再设置轨道倾角，最后将升交线旋转到升交点赤经 \(\Omega\) 对应的方向。

令纬度幅角 \(u=\omega+\nu\)，位置分量也可以直接写成：

\[
\begin{aligned}
x_{\mathrm I}
&=r\left(\cos\Omega\cos u-\sin\Omega\sin u\cos i\right),\\
y_{\mathrm I}
&=r\left(\sin\Omega\cos u+\cos\Omega\sin u\cos i\right),\\
z_{\mathrm I}
&=r\sin u\sin i.
\end{aligned}
\]

这组公式直观地表明：\(u\) 决定卫星沿轨道运动的位置，\(i\) 和 \(\Omega\) 决定该位置如何嵌入三维空间。

不同资料对基本旋转矩阵 \(\boldsymbol R_1,\boldsymbol R_3\) 的主动/被动旋转定义可能不同，因而公式中的正负号可能呈现不同形式。实现时必须统一矩阵定义，并通过已知轨道点检查结果，而不能只复制旋转矩阵名称。

### 1.7 卫星位置并非“天然属于 ECI”

卫星的物理位置本身不天然属于某个坐标系。同一个空间点可以用 ECI、ECEF 或其他坐标系表示。轨道状态通常先在 ECI 或近似惯性坐标系中计算，原因是：

1. 二体引力方程在惯性系中形式最简单；
2. ECI 不随地球表面共同旋转，便于描述轨道平面；
3. 由开普勒根数构造的 PQW 状态可以直接旋转到惯性系。

因此，准确说法应是：**轨道传播通常选择在惯性参考系中完成，而不是卫星位置天然处于 ECI。**

还需要注意，SGP4 根据 TLE 输出的常见参考系为 TEME（True Equator, Mean Equinox），它是惯性型参考系，但不能在高精度处理中无条件等同于 J2000、GCRF 或笼统的“ECI”。NTN 教学仿真可以把它作为惯性状态入口，但坐标转换必须采用与 SGP4/TEME 匹配的实现。

### 1.8 TLE 与 SGP4 的使用边界

TLE 中保存的是与 SGP4 模型配套的平均轨道根数，不是可以任意代入纯二体开普勒公式的瞬时密切根数。其平均运动、阻力相关参数和其他字段共同服务于 SGP4 传播。

实际 NTN 仿真建议采用两条路线之一：

- **理论路线：**人为设置圆轨道或经典六根数，使用二体模型传播，用于理解几何、时延和多普勒；
- **实际星历路线：**读取 TLE，使用经过验证的 SGP4 库得到卫星位置和速度，再进行坐标变换和链路计算。

不建议读取 TLE 后只取出几个字段，再使用简单开普勒公式长期外推。TLE 的生成和使用都与 SGP4 模型相关。

---

## 2. 坐标变换与卫星—UE 几何

### 2.1 坐标变换的总体计算链

第 1 章已经从六根数得到了卫星在惯性空间中的位置和速度：

\[
\left(\boldsymbol r_{s,\mathrm I}(t),
\boldsymbol v_{s,\mathrm I}(t)\right)
\]

但这两个状态量还不能直接回答通信链路中的问题。地面 UE 通常用经纬高表示，并随地球共同旋转；卫星波束又定义在卫星本体或天线坐标系中。只有把不同对象统一到适合的坐标系，才能进一步得到：

- 卫星是否在 UE 的可见范围内；
- 卫星—UE 的斜距和传播时延；
- UE 观察卫星的方位角和仰角；
- UE 位于哪个卫星波束内，以及对应的离轴增益；
- 卫星—UE 距离变化多快，以及由此产生多大的多普勒频移。

因此，坐标变换不是单纯为了改变一组三维数字的表达形式，而是把“轨道状态”转换成“通信链路所需的几何量”。对于给定卫星、UE 和计算时刻 \(t\)，地面链路几何可以按以下主链计算：

\[
\boxed{
\text{轨道根数与历元}
\rightarrow
\left(\boldsymbol r_{s,\mathrm I},\boldsymbol v_{s,\mathrm I}\right)
\rightarrow
\left(\boldsymbol r_{s,\mathrm E},\boldsymbol v_{s,\mathrm E}\right)
\rightarrow
\boldsymbol\rho_{u\rightarrow s,\mathrm E}
\rightarrow
\boldsymbol\rho_{u\rightarrow s,\mathrm{ENU}}
}
\]

其中下标 \(s\) 表示卫星，\(u\) 表示 UE，\(\mathrm I\) 表示 ECI，\(\mathrm E\) 表示 ECEF。这条链依次完成：

1. 将卫星轨道状态传播到目标时刻；
2. 将卫星位置转换到随地球旋转的 ECEF；
3. 将 UE 的经纬高转换到同一个 ECEF；
4. 在共同坐标系中构造卫星—UE 视距向量；
5. 转到 UE 局部 ENU，得到方位角、仰角和斜距。

若还要判断 UE 落在哪个卫星波束中，则需要进一步引入卫星姿态和波束指向：

\[
\boldsymbol\rho_{s\rightarrow u}
\rightarrow
\text{卫星本体坐标系}
\rightarrow
\text{与各波束中心方向比较}
\]

最终得到的几何量与通信模型之间具有直接对应关系：

| 几何量 | 获得方式 | 后续用途 |
|---|---|---|
| 斜距 \(\rho\) | 卫星与 UE 的位置差 | 传播时延、自由空间损耗、链路预算 |
| 方位角 \(A\) | LOS 转到 UE 局部 ENU | UE 天线指向、可见卫星跟踪 |
| 仰角 \(\varepsilon\) | LOS 转到 UE 局部 ENU | 可见性、最低仰角、大气损耗和场景参数选择 |
| 离轴角 \(\alpha_k\) | LOS 转到卫星本体坐标系 | 波束归属、卫星天线方向增益 |
| 距离变化率 \(\dot\rho\) | LOS 上的相对速度投影 | 多普勒频移和多普勒变化率 |

后续各小节按照这条链展开，每一次坐标变换都对应一个明确的通信输出量。

### 2.2 ECI 与 ECEF

ECI 的坐标轴不随地球表面共同旋转，适合轨道传播。ECEF 与地球固连，地面固定 UE 的 ECEF 坐标在忽略板块运动时近似不变。

这里首先进行 ECI 到 ECEF 的变换，是因为第 1 章得到的卫星状态通常位于惯性系，而 UE、网关、服务区边界和地面波束脚印通常用经纬度或 ECEF 表示。把卫星转换到 ECEF 后，卫星和地面对象才能在同一幅“随地球旋转的地图”上比较。

在仅考虑地球自转的简化模型中，ECI 到 ECEF 的位置变换可写为：

\[
\boldsymbol r_{\mathrm E}(t)
=
\boldsymbol C_{\mathrm E\leftarrow\mathrm I}(t)
\boldsymbol r_{\mathrm I}(t)
\]

本文显式采用：

\[
\boldsymbol C_{\mathrm E\leftarrow\mathrm I}(t)
=
\begin{bmatrix}
\cos\theta(t) & \sin\theta(t) & 0\\
-\sin\theta(t) & \cos\theta(t) & 0\\
0 & 0 & 1
\end{bmatrix}
\]

其中 \(\theta(t)\) 为计算时刻对应的地球自转角。显式写出矩阵可以避免不同教材对 \(\boldsymbol R_3(\theta)\) 正负号定义不同造成的混淆。

速度不能只套用同一个旋转矩阵。由于 ECEF 自身在旋转，有：

\[
\boldsymbol v_{\mathrm E}
=
\boldsymbol C_{\mathrm E\leftarrow\mathrm I}
\left(
\boldsymbol v_{\mathrm I}
-\boldsymbol\omega_{\mathrm E}\times\boldsymbol r_{\mathrm I}
\right)
\]

其中 \(\boldsymbol\omega_{\mathrm E}\) 为地球自转角速度向量。位置变换的输出用于计算卫星相对 UE 的斜距和观察角；速度变换的输出用于计算距离变化率和多普勒。若只关心某一时刻的静态覆盖，可以先计算位置；若要分析多普勒，则位置和速度必须同时保持坐标系一致。

高精度 ECI/ECEF 转换还会包括岁差、章动、极移和地球定向参数。普通 NTN 链路级仿真可先采用“地球绕固定 \(z\) 轴旋转”的简化模型；若使用 TLE/SGP4 或高精度星历，应直接调用与数据参考系匹配的坐标转换库。

### 2.3 LLA 到 ECEF

卫星已经转换到 ECEF，但 UE 的原始输入通常是纬度 \(\phi\)、经度 \(\lambda\) 和椭球高 \(h_u\)。经纬高是便于描述地理位置的曲线坐标，不能直接与卫星的三维笛卡尔坐标相减。因此，需要先把 UE 也转换到 ECEF。

采用 WGS-84 椭球时，定义卯酉圈曲率半径：

\[
N(\phi)
=
\frac{a_{\mathrm E}}
{\sqrt{1-e_{\mathrm E}^2\sin^2\phi}}
\]

其中 \(a_{\mathrm E}\) 为地球椭球长半轴，\(e_{\mathrm E}\) 为地球椭球第一偏心率。UE 的 ECEF 坐标为：

\[
\begin{aligned}
x_u&=\left[N(\phi)+h_u\right]\cos\phi\cos\lambda,\\
y_u&=\left[N(\phi)+h_u\right]\cos\phi\sin\lambda,\\
z_u&=\left[N(\phi)(1-e_{\mathrm E}^2)+h_u\right]\sin\phi.
\end{aligned}
\]

这些公式来自参考椭球几何：赤道方向和极轴方向的曲率不同，因此不能简单地把地球半径 \(R_{\mathrm E}\) 同时乘到三个球坐标分量上。

需要区分两个容易混淆的量：

- \(a_{\mathrm{orb}}\)：卫星轨道半长轴；
- \(a_{\mathrm E}\)：WGS-84 地球椭球长半轴。

二者在部分教材中都写成 \(a\)，但物理含义完全不同。

### 2.4 UE 到卫星的相对向量

完成卫星 ECI 到 ECEF 的转换，并把 UE 的 LLA 转换为 ECEF 后，两者处于同一坐标系。UE 指向卫星的视距向量为：

\[
\boldsymbol\rho_{u\rightarrow s,\mathrm E}
=
\boldsymbol r_{s,\mathrm E}
-
\boldsymbol r_{u,\mathrm E}
\]

斜距为：

\[
\rho
=
\left\|\boldsymbol\rho_{u\rightarrow s,\mathrm E}\right\|
\]

相对向量是后续几何计算的共同入口：向量长度给出斜距，单位方向给出 LOS，位置差随时间的变化给出距离变化率。斜距进一步决定最基本的传播量：

\[
\tau=\frac{\rho}{c}
\]

\[
L_{\mathrm{FS}}
=
\left(\frac{4\pi\rho}{\lambda_c}\right)^2
\]

其中 \(\tau\) 为单程自由空间传播时延，\(L_{\mathrm{FS}}\) 为自由空间路径损耗的线性值，\(\lambda_c\) 为载波波长。因此，同一个斜距同时进入时间模型和链路预算。

这里并不需要“先构造 \(\boldsymbol\rho_{\mathrm{ECI}}\)”才能计算 UE 几何。只要卫星和 UE 处于同一个参考系，相对向量就可以直接相减。对于地面观察角，ECEF 是最自然的中间坐标系。

也可以先把 UE 从 ECEF 转到 ECI，再在 ECI 中相减，但这通常会增加不必要的转换步骤。两种方法理论上等价，前提是使用相同的时刻和一致的坐标定义。

### 2.5 ECEF 到 UE 局部 ENU

ECEF 适合做全球位置相减，但不便于直接解释“卫星在 UE 的哪个方向”。UE 侧的天线指向、最低仰角和可见性判断都以当地水平面为参考，因此要把 LOS 转换到 UE 的局部 ENU 坐标系：

\[
\boldsymbol\rho_{u\rightarrow s,\mathrm{ENU}}
=
\boldsymbol C_{\mathrm{ENU}\leftarrow\mathrm E}
\boldsymbol\rho_{u\rightarrow s,\mathrm E}
\]

其中：

\[
\boldsymbol C_{\mathrm{ENU}\leftarrow\mathrm E}
=
\begin{bmatrix}
-\sin\lambda & \cos\lambda & 0\\
-\sin\phi\cos\lambda & -\sin\phi\sin\lambda & \cos\phi\\
\cos\phi\cos\lambda & \cos\phi\sin\lambda & \sin\phi
\end{bmatrix}
\]

令：

\[
\boldsymbol\rho_{u\rightarrow s,\mathrm{ENU}}
=
\begin{bmatrix}E&N&U\end{bmatrix}^{\mathrm T}
\]

则方位角和仰角分别为：

\[
A=\operatorname{atan2}(E,N)
\]

\[
\varepsilon
=
\operatorname{atan2}
\left(U,\sqrt{E^2+N^2}\right)
\]

方位角通常从正北方向开始顺时针计量，计算后可映射到 \([0,2\pi)\)。方位角用于 UE 天线或跟踪机构的水平指向；仰角用于判断卫星是否可见，并进一步选择与仰角相关的大气损耗、阴影衰落和信道参数。当 \(\varepsilon\) 小于系统设定的最小仰角时，即使不存在纯几何地球遮挡，链路也可能不被纳入可服务范围。

### 2.6 UE 仰角与卫星波束方向

UE 观察卫星时使用：

\[
\boldsymbol\rho_{u\rightarrow s}
=
\boldsymbol r_s-\boldsymbol r_u
\]

卫星向 UE 发射或接收波束时使用相反方向：

\[
\boldsymbol\rho_{s\rightarrow u}
=
\boldsymbol r_u-\boldsymbol r_s
=
-\boldsymbol\rho_{u\rightarrow s}
\]

这两个向量长度相同，但方向相反：

- 计算 UE 的方位角和仰角，应使用 UE 指向卫星的向量；
- 判断 UE 位于卫星哪个波束内，应使用卫星指向 UE 的向量。

因此，波束判定不是简单复用 UE 的 ENU 角度，而是需要把卫星到 UE 的 LOS 方向转换到卫星本体坐标系或卫星天线坐标系。

### 2.7 Nadir、卫星姿态与多个波束

UE 侧 ENU 已经给出了“UE 如何看到卫星”，但波束判定需要回答相反的问题：“卫星天线如何看到 UE”。因此需要一个随卫星姿态运动的本体坐标系，把卫星到 UE 的 LOS 与各个波束中心方向放在同一个坐标系中比较。

Nadir 是从卫星指向地心附近的方向：

\[
\hat{\boldsymbol z}_{\mathrm{nadir}}
=
-\frac{\boldsymbol r_s}{\|\boldsymbol r_s\|}
\]

“卫星采用 nadir pointing”通常表示卫星平台或天线参考轴大致保持对地，而不是所有波束都指向地心。多波束卫星可以通过馈源阵列或相控阵，在 nadir 周围形成多个不同离轴方向的波束：

- 中心波束的波束中心可能接近 nadir；
- 其他波束具有不同的离轴角和方位；
- 电子扫描可以使波束脚印随星移动，也可以近似保持对地固定；
- 卫星本体姿态、天线安装矩阵和电子波束权值共同决定实际波束方向。

在简化的对地姿态模型中，可以构造一个卫星局部本体坐标系。例如：

\[
\hat{\boldsymbol z}_{\mathrm B}
=
-\frac{\boldsymbol r_s}{\|\boldsymbol r_s\|}
\]

\[
\hat{\boldsymbol y}_{\mathrm B}
=
-\frac{\boldsymbol r_s\times\boldsymbol v_s}
{\|\boldsymbol r_s\times\boldsymbol v_s\|}
\]

\[
\hat{\boldsymbol x}_{\mathrm B}
=
\hat{\boldsymbol y}_{\mathrm B}
\times
\hat{\boldsymbol z}_{\mathrm B}
\]

在近圆轨道下，\(\hat{\boldsymbol x}_{\mathrm B}\) 近似沿飞行方向，\(\hat{\boldsymbol z}_{\mathrm B}\) 指向地面，三者构成右手坐标系。具体工程项目可能采用不同的本体轴定义，因此仿真配置中必须明确轴方向。

### 2.8 波束归属判定

设卫星指向 UE 的单位向量已经转换到卫星本体坐标系：

\[
\hat{\boldsymbol\ell}_{u,\mathrm B}
=
\boldsymbol C_{\mathrm B\leftarrow\mathrm E}
\frac{\boldsymbol\rho_{s\rightarrow u,\mathrm E}}
{\left\|\boldsymbol\rho_{s\rightarrow u,\mathrm E}\right\|}
\]

第 \(k\) 个波束在本体坐标系中的中心方向为 \(\hat{\boldsymbol b}_{k,\mathrm B}\)。UE 相对该波束中心的离轴角为：

\[
\alpha_k
=
\arccos\left(
\hat{\boldsymbol\ell}_{u,\mathrm B}^{\mathrm T}
\hat{\boldsymbol b}_{k,\mathrm B}
\right)
\]

常见判定方式包括：

1. **几何边界判定：**若 \(\alpha_k\leq\alpha_{k,\mathrm{edge}}\)，则 UE 位于波束 \(k\) 的定义范围内；
2. **增益判定：**根据波束方向图计算 \(G_k(\alpha_k)\)，判断是否高于覆盖门限；
3. **服务波束判定：**在所有候选波束中选择增益最大或接收质量最好的波束；
4. **网络配置判定：**即使多个波束几何重叠，最终服务波束仍可能由频率规划、负载和移动性策略决定。

因此，“UE 落在哪个 beam”至少需要卫星位置、UE 位置、计算时刻、卫星姿态和每个波束的中心方向或方向图。只有卫星位置和 UE 位置，最多只能得到 LOS 方向，不能唯一确定波束编号。波束编号和离轴角进一步决定卫星端方向增益，是从纯几何模型进入链路预算和波束切换模型的接口。

### 2.9 距离变化率与多普勒入口

斜距回答“卫星离 UE 多远”，距离变化率回答“这个距离变化得多快”。后者只取相对速度在 LOS 方向上的投影，是卫星运动与通信载频之间的连接量。

在同一坐标系中给出卫星和 UE 的位置、速度后，LOS 距离变化率为：

\[
\dot\rho
=
\hat{\boldsymbol\rho}_{u\rightarrow s}^{\mathrm T}
\left(\boldsymbol v_s-\boldsymbol v_u\right)
\]

其中：

\[
\hat{\boldsymbol\rho}_{u\rightarrow s}
=
\frac{\boldsymbol r_s-\boldsymbol r_u}
{\|\boldsymbol r_s-\boldsymbol r_u\|}
\]

对于载频 \(f_c\)，采用“距离增大为正”的约定时，多普勒频移可以写为：

\[
f_{\mathrm D}
=
-\frac{f_c}{c}\dot\rho
\]

如果 \(\dot\rho<0\)，卫星与 UE 接近，接收频率升高；如果 \(\dot\rho>0\)，两者远离，接收频率降低。由此可见，轨道速度并不能直接代入多普勒公式，必须先投影到 LOS 方向。不同通信模型可能采用相反的多普勒符号约定，实现时应同时检查复指数的定义。

---

## 3. NTN 相关时间

### 3.1 时间作为几何状态的共同索引

第 2 章说明了如何在某一个时刻得到斜距、方位角、仰角、波束离轴角和距离变化率。但 LEO 卫星持续高速运动，这些量都不是固定参数，而是随时间变化的函数：

\[
\left\{
\boldsymbol r_s(t),
\boldsymbol v_s(t),
\rho(t),
A(t),
\varepsilon(t),
\alpha_k(t),
\dot\rho(t)
\right\}
\]

因此，时间的首要作用是把轨道传播、地球旋转和链路几何放到同一个时刻上。具体包括：

1. **确定卫星状态：**由轨道历元 \(t_0\) 传播到目标时刻 \(t\)，得到 \(\boldsymbol r_s(t)\) 和 \(\boldsymbol v_s(t)\)；
2. **确定地球姿态：**根据同一个 \(t\) 计算地球自转角，完成 ECI/ECEF 转换；
3. **确定链路几何：**在 \(t\) 时刻计算卫星与 UE 的相对位置、波束归属和距离变化率；
4. **描述信号传播：**区分信号离开发射端的 \(t_{\mathrm{tx}}\)、到达接收端的 \(t_{\mathrm{rx}}\) 以及二者之差 \(\tau\)。

这四个过程共同解释了为什么 NTN 几何模型不能只有轨道和坐标，还必须具有统一的时间基准。若不同步骤使用了不一致的时刻，即使每个空间公式本身正确，最终得到的波束、时延和多普勒仍可能错误。

### 3.2 NTN 几何需要的时间尺度

| 时间尺度 | 是否连续 | NTN 中的主要用途 | 学习深度 |
|---|---|---|---|
| TAI | 连续 | 连续原子时间基准 | 知道其与 UTC 的关系即可 |
| UTC | 含闰秒调整 | 星历时间戳、日志和跨系统时间交换 | 必须掌握 |
| GPS/GNSS 系统时间 | 通常连续，不插入 UTC 闰秒 | GNSS 定位、时间辅助和设备内部计算 | 必须掌握 |
| UT1 | 反映地球实际自转 | 高精度 ECI/ECEF 变换 | 理解用途，简单仿真可近似 |
| TT | 连续理论时间尺度 | 精密星历和天文学计算 | 普通 NTN 仿真不展开 |

#### 3.2.1 UTC、TAI 与 GNSS 时间

TAI 是连续的原子时间尺度。UTC 以原子秒为基础，但通过闰秒保持与地球自转时间接近，因此 UTC 时间轴可能出现闰秒。

GPS Time 等 GNSS 系统时间通常保持连续，不随 UTC 插入闰秒。GNSS 接收机通过导航消息或系统提供的参数获得 GNSS 时间与 UTC 的当前偏移关系。

工程实现中不应在代码里长期写死“GPS 时间与 UTC 固定相差多少秒”。正确做法是由时间库、导航数据或有效配置提供当前偏移量。

#### 3.2.2 UT1 与地球自转

UT1 根据地球的实际自转确定。严格的地球自转角以及高精度 ECI/ECEF 变换需要 UT1 和地球定向参数。

对于以链路预算、毫秒级时延和大尺度多普勒为主的 NTN 基础仿真，可以使用 UTC 近似生成地球自转角，或者直接使用成熟库完成转换。对于窄波束精确指向、长时间轨道预测和高精度定位，应使用与 UT1/EOP 一致的坐标转换模型。

### 3.3 轨道历元、计算时刻与星历有效性

轨道数据总是在某个历元 \(t_0\) 下给出。给定目标时刻 \(t\)，首先计算：

\[
\Delta t=t-t_0
\]

然后使用二体模型、SGP4 或其他轨道传播器得到：

\[
\left(
\boldsymbol r_s(t),
\boldsymbol v_s(t)
\right)
\]

这要求 \(t\) 和 \(t_0\) 使用一致的时间尺度。把本地时区时间、UTC、GPS Time 或 Unix 时间戳直接相减，而不先统一定义，会导致固定秒偏差甚至闰秒附近的不连续。

对 NTN 而言，星历误差会进一步转化为通信误差。若卫星位置误差为 \(\delta\boldsymbol r_s\)，其 LOS 投影引起的时延误差近似为：

\[
\delta\tau
\approx
\frac{1}{c}
\hat{\boldsymbol\rho}^{\mathrm T}
\delta\boldsymbol r_s
\]

若速度误差为 \(\delta\boldsymbol v_s\)，其引起的多普勒误差近似为：

\[
\delta f_{\mathrm D}
\approx
-\frac{f_c}{c}
\hat{\boldsymbol\rho}^{\mathrm T}
\delta\boldsymbol v_s
\]

因此，星历不仅用于显示卫星轨迹，还直接影响波束指向、传播时延预测和多普勒预测精度。

### 3.4 计算时刻与 ECI/ECEF 转换

卫星惯性坐标转换到地固坐标时，必须明确“在哪个时刻旋转地球”。正确关系是：

\[
\boldsymbol r_{s,\mathrm E}(t)
=
\boldsymbol C_{\mathrm E\leftarrow\mathrm I}(t)
\boldsymbol r_{s,\mathrm I}(t)
\]

位置状态和旋转矩阵必须对应同一个计算时刻 \(t\)。常见错误包括：

- 使用目标时刻的卫星 ECI 位置，却使用历元时刻的地球旋转角；
- 卫星位置使用 UTC，地球旋转角使用未经转换的 GPS Time；
- 只更新时间而没有重新传播卫星位置；
- 将经度角直接当作地球自转角。

对系统级仿真，可以统一设置一个仿真绝对起始时刻 \(t_{\mathrm{start}}\)，再用仿真时间 \(t_{\mathrm{sim}}\) 推进：

\[
t=t_{\mathrm{start}}+t_{\mathrm{sim}}
\]

每个仿真步使用同一个 \(t\) 更新轨道状态、地球旋转、UE 几何、波束归属、时延和多普勒。

### 3.5 发射时刻、接收时刻与传播时延

在最简单的同时刻几何模型中，单程传播时延为：

\[
\tau(t)
=
\frac{\|\boldsymbol r_s(t)-\boldsymbol r_u(t)\|}{c}
=
\frac{\rho(t)}{c}
\]

它适合多数 NTN 链路预算、时延曲线和系统级抽象。

更严格地说，信号在发射时刻 \(t_{\mathrm{tx}}\) 离开发射端，在接收时刻 \(t_{\mathrm{rx}}\) 到达接收端：

\[
t_{\mathrm{rx}}=t_{\mathrm{tx}}+\tau
\]

对于卫星发射、UE 接收的链路，应写成：

\[
\tau
=
\frac{
\left\|
\boldsymbol r_u(t_{\mathrm{rx}})
-
\boldsymbol r_s(t_{\mathrm{tx}})
\right\|
}{c}
\]

由于 \(t_{\mathrm{tx}}=t_{\mathrm{rx}}-\tau\)，可以迭代求解。这个处理有时称为发射时刻修正或迟滞时间计算。

例如，若某一时刻斜距为 1200 km，则单程传播时延约为：

\[
\tau\approx\frac{1200\times10^3}{3\times10^8}=4\ \mathrm{ms}
\]

若 LEO 卫星速度约为 \(7.5\ \mathrm{km/s}\)，4 ms 内卫星移动约 30 m。这对一般覆盖分析影响有限，但在高精度波束指向、多普勒预测或定位中可能需要考虑。

### 3.6 用户链路、馈电链路与总时延

NTN 时延取决于网络架构。

#### 3.6.1 再生载荷

如果卫星上集成 gNB 或具有再生处理能力，UE 与卫星之间的用户链路传播时延是无线接入侧的主要空间传播项：

\[
\tau_{\mathrm{service}}
=
\frac{\rho_{\mathrm{UE-SAT}}}{c}
\]

业务端到端时延还可能包括星间链路、核心网传输和处理时延，但这些不应全部混入“用户链路传播时延”。

#### 3.6.2 透明转发载荷

如果卫星进行透明转发，gNB 位于地面，空口信号路径通常包含用户链路和馈电链路：

\[
\tau_{\mathrm{space}}
\approx
\frac{\rho_{\mathrm{UE-SAT}}}{c}
+
\frac{\rho_{\mathrm{SAT-GW}}}{c}
\]

还需要叠加卫星转发、网关和地面网络处理时延。上下行路径相似时，RTT 近似为两个方向空间传播及处理时延之和，但不能在所有架构中简单写成某一条用户链路时延的两倍。

至此，单个时刻的空间链路时延已经可以由用户链路、馈电链路和处理环节组合得到。接下来只需要让计算时刻持续推进，就能得到这些物理量的时间演化。

### 3.7 时延与多普勒的时间演化

第 2 章得到的斜距 \(\rho\) 和距离变化率 \(\dot\rho\) 都以计算时刻 \(t\) 为索引。卫星持续运动，因此距离、时延和多普勒都是时间函数：

\[
\rho=\rho(t),\qquad
\tau(t)=\frac{\rho(t)}{c},\qquad
f_{\mathrm D}(t)=-\frac{f_c}{c}\dot\rho(t)
\]

多普勒变化率为：

\[
\dot f_{\mathrm D}(t)
=
-\frac{f_c}{c}\ddot\rho(t)
\]

若在时刻 \(t\) 计算一次几何量，并在 \(\Delta t\) 时间内保持不变，则一阶近似下：

\[
\Delta\tau
\approx
\frac{\dot\rho(t)}{c}\Delta t
\]

\[
\Delta f_{\mathrm D}
\approx
\dot f_{\mathrm D}(t)\Delta t
\]

这两个关系解释了为什么时延、多普勒和波束归属需要周期性更新，而不能把卫星几何作为整段仿真的固定参数。更新周期应根据：

- 卫星轨道高度与角速度；
- UE 位置和运动；
- 载频；
- 星历误差；
- 仿真或接收机允许的时延和频偏误差；
- 波束覆盖和切换策略；

共同确定。

### 3.8 NTN 几何与时间的完整计算流程

前两章解决“卫星在什么位置”和“如何从位置得到通信几何量”，本章补充“在哪个时刻计算这些量”。三者合在一起，一个不过度复杂、但物理链条完整的 NTN 几何模块可以按以下顺序实现。

#### 输入

- 卫星六根数或 TLE；
- 轨道历元 \(t_0\)；
- 计算绝对时刻 \(t\)；
- UE 的 LLA 位置和速度；
- 卫星姿态模型；
- 每个波束的中心方向和边界/方向图；
- 载频和网络架构。

#### 计算

1. 统一 \(t_0\) 和 \(t\) 的时间尺度；
2. 使用二体模型或 SGP4 得到 \(\boldsymbol r_{s,\mathrm I}(t)\)、\(\boldsymbol v_{s,\mathrm I}(t)\)；
3. 根据同一时刻的地球自转角转换到 ECEF；
4. 将 UE 的 LLA 转换到 ECEF；
5. 计算 UE 到卫星的 LOS、斜距、方位角和仰角；
6. 反转 LOS 方向并转换到卫星本体坐标系，判断波束归属；
7. 计算距离变化率、多普勒和多普勒变化率；
8. 根据网络架构组合用户链路、馈电链路和处理时延；
9. 将时刻推进到 \(t+\Delta t\)，重复上述过程，形成连续几何轨迹。

#### 输出

\[
\left\{
\boldsymbol r_s(t),
A(t),\varepsilon(t),\rho(t),
\text{beam ID}(t),
\tau(t),\dot\rho(t),f_{\mathrm D}(t),\dot f_{\mathrm D}(t)
\right\}
\]

这组时间序列构成后续链路预算、信道建模、波束切换和系统级仿真的共同几何基础。其核心逻辑可以收敛为：

\[
\boxed{
\text{绝对时刻}
\rightarrow
\text{卫星状态与地球姿态}
\rightarrow
\text{卫星—UE 相对几何}
\rightarrow
\text{时延、增益与多普勒}
}
\]

---

## 参考依据

1. [3GPP TR 38.811, *Study on New Radio (NR) to support non-terrestrial networks (NTN)*](https://www.3gpp.org/dynareport/38811.htm)：NTN 场景、传播时延、多普勒、架构和 NR 影响的基础技术报告。
2. [3GPP TR 38.821, *Solutions for NR to support Non-Terrestrial Networks (NTN)*](https://www.3gpp.org/dynareport/38821.htm)：NR NTN 时频关系和适配机制的研究报告。
3. [IERS Conventions (2010), Chapter 5](https://iers-conventions.obspm.fr/chapter5.php)：地心天球参考系与地固参考系之间的严格转换框架。
4. [D. A. Vallado et al., *Revisiting Spacetrack Report #3*](https://celestrak.org/publications/AIAA/2006-6753/AIAA-2006-6753-Rev2.pdf)：TLE/SGP4 轨道传播的模型、实现和验证依据。
5. WGS-84 参考椭球：LLA 与 ECEF 坐标转换所采用的地球几何基础。
