# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

单文件 HTML 应用 — 微型扬声器无限障板频响计算器。精密仪器风暗色主题，23 个 CSS 变量 token 系统。输入 TS 参数和驱动电压，输出 SPL (RMS)、位移 (峰值)、最大能力频响、EQ 电压、阻抗/相位曲线，支持非线性失真分析、时域波形和 FFT 频谱。默认预置 2110 水滴型微扬声器参数。

## 使用方式

- **运行**：浏览器打开 `speaker_baffle_calc.html`，无构建/编译
- **依赖**：Chart.js 4.4 CDN（`cdn.jsdelivr.net/npm/chart.js@4.4.0`），需联网
- **JS 语法验证**：`node -e "new Function(require('fs').readFileSync('speaker_baffle_calc.html','utf8').match(/<script>([\s\S]*?)<\/script>/)[1])"`
- **声学模型验证**：Node 内联计算脚本，与参考值对比

## 架构

单文件三层：**CSS（精密仪器暗色主题）→ HTML（左面板 + 右图表区 + 底部非线性折叠面板）→ JS（所有逻辑内联）**

### JS 模块划分

| 模块 | 入口 | 职责 |
|------|------|------|
| Complex 类 | `class Complex` | 复数四则运算 |
| 输入/转换 | `getParams()`, `toSI(p)` | 参数读取，单位→SI (Cms÷1000, Mms÷1e6, Sd÷1e6, **Le÷1e6**) |
| 频率范围 | `getFreqRange()` | fStart/fEnd/fPoints → 对数扫频数组 |
| 线性计算 | `calcLinearResponse()` | 等效电路频域求解 → `{spl, disp(峰值×√2), imp}` |
| 最大能力 & EQ | `calcAll()` 内联 | `V_to_Xmax = Xmax(peak)/(disp_per_V_peak)`，`V_limit = min(V_to_Xmax, Vmax)` |
| 阻抗/相位 | `calcAll()` 内联 | `\|Zin(f)\|` + `atan2` 相位 + 解卷曲(检测±180°跳变补360°) |
| 插值 | `AkimaSpline`, `SteffenSpline` | 保形样条，处理非对称 Bl(x)/Cms(x) |
| 非线性-快速 | `describingFunction()` | 描述函数迭代(≤30轮) → 扰动法 THD: F(t)→力谐波→Zm(nω)→位移谐波 |
| 非线性-精细 | `rk4Solve()` + `extractHarmonics()` | RK4 (256步/周期, 自适应settling, ωLe<0.1Re→代数电流) |
| 非线性EQ电压 | `calcNonlinear()` 内联 | 描述函数反解迭代 → 找到使位移=Xmax的Vrms |
| 时域/FFT分析 | `runTimeDomainAnalysis()` | 指定频点→RK4非线性+线性解析→位移/速度/加速度时域波形+FFT(H1-H10) |
| 拟合曲线 | `drawFittingChart()` | Bl(x)/Cms(x) 样条拟合+原始散点 |
| 图表 | `createChart()`, `updateLineChart()` | Chart.js 封装 |
| 导出 | `exportCSV_chart()`, `exportCSV_nl()`, `exportPNG()` | 线性/非线性按图CSV+PNG |
| UI | `toggleChart()`, `setXAxis()`, `setPhaseWrap()`, `switchTDView()` | 图表显隐、X轴对数/线性、相位卷曲、时域视图切换 |

### 图表结构（线性5图 + 非线性8图 + 时域2图）

| 图表 | 数据集 |
|------|--------|
| **chartSPL** | `[0]`当前SPL(蓝) `[1]`最大能力SPL(黄虚线) |
| **chartDisp** | `[0]`峰值位移(红) `[1]`Xmax(红虚线) |
| **chartEQ** | `[0]`EQ电压(青) `[1]`Vmax(青虚线) |
| **chartImp** | `[0]`阻抗模值(紫) |
| **chartPhase** | `[0]`相位(橙,默认非卷曲) |
| **chartBlFit/chartCmsFit** | 样条拟合+原始散点 |
| **chartNlSPL/chartNlDisp/chartNlEQ** | 线性(蓝) vs 非线性(红) |
| **chartTHD** | THD Total/H2/H3 |
| **chartTDdisp/vel/acc** | 时域波形 线性vs非线性 |
| **chartFFTdisp/vel/acc** | FFT柱状图 H1-H10 线性vs非线性 |

### 数据流

```
TS参数 + Vrms + Vmax → toSI() → calcDerived() → getFreqRange()
  → calcLinearResponse() → {spl, disp(peak), imp}
  → 最大能力: V_limit=min(V_to_Xmax,Vmax) → maxSPL + eqVoltage
  → 阻抗/相位: |Zin|, atan2相位 + 解卷曲
  → 渲染5图(图表选择器控显隐)
  → (可选) 非线性面板 → Bl/Cms数据表 → calcNonlinear()
    → 拟合图 + 描述函数/RK4 → SPL/位移/EQ对比 + THD
    → (可选) 时域/FFT: 输入频点 → RK4波形 → 位移/速度/加速度切换
  → (可选) 各图独立CSV/PNG导出
```

## 领域知识要点

- **物理模型**：无限障板半空间辐射，p = jω·ρ₀·Sd·v/(2πr)
- **电-机-声等效电路**：Ze(Re+Le) → Zmot(Bl²/Zm) → Zm(Rms+Mms+Cms) → U=Sd·v
- **单位约定**：输入 Cms(mm/N)、Mms(mg)、Sd(mm²)、**Le(μH)**，位移输出峰值(×√2)，SPL为RMS
- **THD扰动法**：描述函数收敛→时域力波形F(t)→DFT力谐波→Zm(nω)→位移谐波→THD
- **代数电流模式**：ωLe<0.1Re 时 i=(V-Bl·v)/Re，避免ODE刚性
- **相位解卷曲**：检测相邻频点±180°跳变，自动累加360°。2110水滴相位范围[-14°,+29°]无跳变

## 设计决策

- 精密仪器风：23 CSS变量token系统，32px网格背景纹理，卡片化图表，focus glow效果
- X轴对数/线性 radio 切换（全局生效），相位卷曲checkbox在图表内
- 非线性频段 20-1000Hz，时域150点/快速200点
- RK4自适应settling max(5,ceil(10τ·f))，256步/周期
- 时域/FFT分析默认显示位移，按钮切换速度/加速度
- Le默认30μH，预置2110水滴(Fs≈240Hz)
- 所有图表独立CSV/PNG导出，非线性含对比数据
