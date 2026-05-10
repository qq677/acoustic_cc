# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

单文件 HTML 应用 — 微型扬声器无限障板频响计算器。输入 TS 参数和驱动电压，输出频率响应 (SPL, RMS)、振幅响应 (位移, 峰值)、最大能力频响包络、EQ 电压曲线，支持非线性失真分析。默认预置 2110 水滴型微扬声器参数。

## 使用方式

- **运行**：直接用浏览器打开 `speaker_baffle_calc.html`，无构建/编译步骤
- **依赖**：Chart.js 4.4 通过 CDN 加载（`cdn.jsdelivr.net/npm/chart.js@4.4.0`），需联网
- **JS 语法验证**：提取 `<script>` 内 JS 后用 `new Function(js)` 检查
- **声学模型验证**：用 Node 跑内联计算脚本，与参考值对比

## 架构

单文件三层结构：

1. **CSS（暗色主题）**：左侧 340px 参数面板 + 右侧弹性图表区 + 底部折叠非线性面板，CSS 变量控制配色
2. **HTML**：`left-panel` 输入区（TS参数 + Vmax + 频率范围 + 导出按钮 + 预置切换）→ `right-panel` 三图区（SPL / 位移 / EQ电压）→ `nonlinear-panel` 折叠区
3. **JS**：所有逻辑内联在 `<script>` 标签中

### JS 模块划分

| 模块 | 入口 | 职责 |
|------|------|------|
| Complex 类 | `class Complex` | 复数四则运算，频域等效电路的基础 |
| 输入/转换 | `getParams()`, `toSI(p)` | 页面参数读取，自定义单位→SI (Cms÷1000, Mms÷1e6, Sd÷1e6, Le÷1000) |
| 频率范围 | `getFreqRange()` | 读取 fStart/fEnd/fPoints 生成对数扫频数组 |
| 线性计算 | `calcLinearResponse()` | 等效电路法频域求解，输出 `{spl, disp, imp}` 三个 Float64Array |
| 最大能力频响 & EQ电压 | `calcAll()` 内联 | 逐频点：`V_to_Xmax = Xmax/(disp/Vrms)`，`V_limit = min(V_to_Xmax, Vmax)`。输出 `maxSPL`（最大能力 SPL 曲线）和 `eqVoltage`（EQ 电压曲线） |
| 插值 | `AkimaSpline`, `SteffenSpline` | 一维保形样条，处理非对称 Bl(x)/Cms(x) |
| 非线性-快速 | `describingFunction()` | 描述函数法：等效线性化迭代 30 轮 + 128 点数值积分提取 Bl(x) 谐波→估算 THD |
| 非线性-精细 | `rk4Solve()` + `extractHarmonics()` | RK4 时域积分（200步/周期，4周期测量）→稳态 DFT 提取 H1-H5→THD |
| 图表 | `createChart()`, `updateLineChart()` | Chart.js 封装，对数 X 轴，多数据集管理 |
| 导出 | `exportCSV()`, `exportPNG(id)` | CSV 下载 (Freq/SPL/Disp)，PNG 截图 (canvas.toDataURL) |
| UI | `calcAll()`, `calcNonlinear()`, `showNonlinearResults()` | 事件绑定，预置参数切换，进度反馈 |

### 图表数据集结构（线性模式 3 图，非线性模式额外 3 图）

- **chartSPL**（2条线）：`[0]` 当前 SPL（蓝实线）、`[1]` 最大能力 SPL（黄虚线）
- **chartDisp**（2条线）：`[0]` 峰值位移（红实线）、`[1]` Xmax 参考线（红虚线）
- **chartEQ**（2条线）：`[0]` EQ 电压 Vrms（青实线）、`[1]` Vmax 参考线（青虚线）
- **chartNlSPL**（2条线）：`[0]` SPL 线性、`[1]` SPL 非线性
- **chartNlDisp**（3条线）：`[0]` 位移线性、`[1]` 位移非线性、`[2]` Xmax
- **chartTHD**（3条线）：`[0]` THD Total、`[1]` H2、`[2]` H3

### 数据流

```
用户输入 TS 参数 + Vrms + Vmax + 距离 + 频率范围
  → toSI() 单位转换
  → calcDerived() 计算 Fs/Qts/Qes/Qms/Vas/SPL₀
  → getFreqRange() 生成对数频率数组
  → calcLinearResponse() → {spl(SPL RMS), disp(位移峰值), imp}（disp 已 ×√2）
  → 最大能力频响：逐频点 V_limit = min(V_to_Xmax, Vmax) → maxSPL + eqVoltage
  → Chart.js 渲染 SPL(含maxSPL叠加) + 位移(含Xmax线) + EQ电压(含Vmax线) 三图
  → (可选) 展开非线性面板 → 输入 Bl(x)/Cms(x) 数据表
  → calcNonlinear() → describingFunction() 或 rk4Solve() → 三张对比图
  → (可选) exportCSV() / exportPNG() 导出数据
```

## 领域知识要点

- **物理模型**：无限障板 = 振膜嵌入无限大刚性平面，半空间辐射。远场声压 p = jω·ρ₀·Sd·v / (2πr)
- **电-机-声等效电路**：Ze(Re+Le) → Zmot(Bl²/Zm) → Zm(Rms+Mms+Cms) → 体积速度 U=Sd·v → 辐射
- **单位约定**：输入 Cms(mm/N)、Mms(mg)、Sd(mm²)，内部全转 SI。位移输出为**峰值** mm（×√2），SPL 为 **RMS** dB，Vas 显示用 cm³
- **EQ 电压**：达到最大能力 SPL 所需的 RMS 驱动电压。低频段 = Xmax/(disp_per_V)（随频率降低而升高，位移限制），高频段 = Vmax（电压限制）。曲线拐点即 Xmax→Vmax 过渡频率
- **Akima 优于自然三次样条**：Bl(x)/Cms(x) 往往非均匀网格且非对称，Akima 局部性强不过冲
- **THD 非对称判据**：Bl(x) 非对称 → H2 显著 ≠ 0；对称非线性 → H3 为主

## 设计决策

- 非线性面板用折叠设计：保持单一工作流，线性/非线性对比一目了然
- 时域求解默认 ≤80 频点、快速模式 ≤200 频点：平衡精度与浏览器 UI 响应
- 预置参数需保持物理合理性（Qts 1-2，Fs 800-1200Hz，SPL₀ 75-85dB @ 1W/1m）
- Vas 显示用 cm³：微扬声器 Vas 通常在 0.1-2 cm³ 量级
- 最大能力 SPL 用黄色虚线叠加于 SPL 图：直观看到当前驱动与极限之间的裕量
- EQ 电压单独成第三图（青色）：直观显示低频段需要多少电压增益才能推到 Xmax
- 默认预置 2110 水滴参数（Fs≈240Hz 较软悬挂微扬声器）
- 导出 PNG 用 canvas.toDataURL 直接下载，不依赖额外库
