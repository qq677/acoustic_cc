# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

三款单文件 HTML 微型扬声器声学计算器，共享同一套精密仪器风暗色主题（23 CSS token）、Chart.js 4.4 CDN、Complex 类及样条插值器。无构建/编译，浏览器直接打开。

| 文件 | 模型 | 额外输入 |
|------|------|----------|
| `speaker_baffle_calc.html` | 无限障板（半空间辐射） | — |
| `speaker_closed_box_calc.html` | 密闭箱（后腔空气刚度） | Vb(cm³), Mmb(mg) |
| `speaker_bass_reflex_calc.html` | 倒相箱（Helmholtz 谐振器，4 阶） | Vb, Fb(Hz), Dv(mm), Mmb |

## 使用方式

- **运行**：浏览器打开对应 `.html` 文件
- **依赖**：Chart.js 4.4 CDN，需联网
- **JS 语法验证**：
  ```
  node -e "new Function(require('fs').readFileSync('speaker_baffle_calc.html','utf8').match(/<script>([\s\S]*?)<\/script>/)[1])"
  ```
  替换文件名可验证其他两个计算器
- **声学模型验证**：`node -e "..."` 内联脚本，手工验证关键物理量（Fc、位移、SPL）

## 架构

三文件均为**单文件三层结构**：CSS → HTML（左面板+右图表区+底部非线性折叠面板）→ JS（所有逻辑内联）。

三个计算器共享大部分代码，差异集中在物理模型层：

### 三文件共享的核心模块

| 模块 | 关键函数 | 职责 |
|------|----------|------|
| Complex 类 | `class Complex` | 复数四则运算，三文件完全相同 |
| 样条插值 | `AkimaSpline`, `SteffenSpline` | 保形样条，处理非对称 Bl(x)/Cms(x) |
| 频率范围 | `getFreqRange()` | 对数扫频数组 |
| 图表 | `createChart()`, `updateLineChart()` | Chart.js 封装 |
| 导出 | `exportCSV_chart()`, `exportCSV_nl()`, `exportPNG()` | 线性/非线性按图 CSV+PNG |
| UI | `toggleChart()`, `setXAxis()`, `setPhaseWrap()`, `switchTDView()` | 图表显隐、X轴对数/线性、相位卷曲 |
| 拟合曲线 | `drawFittingChart()` | Bl(x)/Cms(x) 样条拟合+散点 |
| 谐波提取 | `extractHarmonics()` | DFT H1-H5 + THD |
| 扰动力 | `parseDataTable()` | 解析 "x y" 格式数据表 |

### 各计算器物理模型差异

**障板** — `calcLinearResponse()`: `Zm = Rms + jω·Mms − j/(ω·Cms)`，SPL ∝ jω·Sd·v/(2πr)

**密闭箱** — 新增 `boxForce(x, pSI)`：绝热气体定律 `F = −P₀·Sd·[(Vb/(Vb−Sd·x))^γ − 1]`。`toSI()` 中 `Cms_box = 1/(1/Cms_driver + k_box)`，`Mms_box = (Mms+Mmb)/1e6`。非线性层通过 `boxForce()` 和 `kEq_box`（独立描述函数积分）引入箱体空气非线性的偶次谐波。

**倒相箱** — 新增 `calcZmb(w, pSI)` 和 `calcZmt(w, kTotal, pSI)`：端口∥箱体声顺的机械域阻抗。线性响应 `Z_mt = Zm_driver + Z_mb`，SPL ∝ |Sd·v·(1+H_port)|。非线性 RK4 扩展为 5 状态 ODE（x, v, pb, Up, i），额外包含 `RapNl(Up)` 端口湍流模型。

### 数据流（三文件通用）

```
TS参数 + 箱体/端口参数 → toSI() → calcDerived() → getFreqRange()
  → calcLinearResponse() → {spl, disp(peak), imp}
  → 最大能力: V_limit=min(V_to_Xmax,Vmax) → maxSPL + eqVoltage
  → 阻抗/相位: |Zin|, atan2相位 + 解卷曲
  → 渲染5图（默认仅SPL可见，勾选展开）
  → (可选) 非线性面板 → Bl/Cms数据表 → calcNonlinear()
    → 拟合图 + 描述函数/RK4 → SPL/位移/EQ对比 + THD
    → (可选) 时域/FFT: 输入频点 → RK4波形 → 位移/速度/加速度切换
  → (可选) 各图独立CSV/PNG导出
```

## 领域知识要点

- **单位约定**：输入 Cms(mm/N)、Mms(mg)、Sd(mm²)、Le(μH)、Vb(cm³)、Mmb(mg)、Dv(mm)，位移输出峰值(×√2)，SPL 为 RMS
- **障板辐射**：半空间 p = jω·ρ₀·Sd·v/(2πr)
- **密闭箱刚度**：k_box = γ·P₀·Sd²/Vb，linear；非线性力 `F_box(x)` 产生偶次谐波
- **倒相箱端口**：Fb = 1/(2π√(Map·Cab))，Map 由 Fb 反算，Lv = Map·Sv/ρ₀ − 0.85·Dv；Qb = 7（经验值）；在 Fb 处位移出现凹陷
- **THD 扰动法**：描述函数收敛→时域力波形 F(t)→DFT 力谐波→Zm(nω)→位移谐波→THD
- **代数电流模式**：ωLe < 0.1Re 时 i = (V−Bl·v)/Re，避免 ODE 刚性
- **相位解卷曲**：检测相邻频点 ±180° 跳变，自动累加 360°
- **箱体非线性**（密闭/倒相共享）：绝热气体 `F_box(x) = −P₀·Sd·[(Vb/(Vb−Sd·x))^γ − 1]`，γ=1.4，P₀=101325Pa

## 设计决策

- 精密仪器风：23 CSS 变量 token 系统，32px 网格背景，卡片化图表
- 图表默认仅显示 SPL，其他四图通过 checkbox 展开后自动 resize
- X 轴对数/线性 radio 全局切换；相位卷曲 checkbox 在图表内
- 非线性频段 20-1000Hz，时域 150 点/快速 200 点
- RK4 自适应 settling max(5, ceil(10τ·f))，256 步/周期
- 时域/FFT 默认位移，按钮切换速度/加速度
- 2110 水滴预置（Fs≈240Hz, Vb=8cm³, Fb=150Hz, Dv=4mm）
- 所有图表独立 CSV/PNG 导出
- 三个计算器代码高度重复但保持独立——修改公式时需同步检查三文件对应段落
