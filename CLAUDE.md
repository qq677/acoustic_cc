# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

单文件 HTML 应用 — 微型扬声器无限障板频响计算器。输入 TS 参数和驱动电压，输出频率响应 (SPL) 和振幅响应 (位移)，支持非线性失真分析。

## 使用方式

- **运行**：直接用浏览器打开 `speaker_baffle_calc.html`，无构建/编译步骤
- **依赖**：Chart.js 4.4 通过 CDN 加载，需联网
- **验证**：`node -e "new Function(require('fs').readFileSync('speaker_baffle_calc.html','utf8').match(/<script>([\\s\\S]*?)<\\/script>/)[1])"` 做 JS 语法检查；用 Node 跑内联计算脚本验算声学公式

## 架构

单文件三层结构：

1. **CSS（暗色主题）**：左侧 340px 参数面板 + 右侧弹性图表区 + 底部折叠非线性面板，CSS 变量控制配色
2. **HTML**：`left-panel` 输入区 → `right-panel` 线性图表区 → `nonlinear-panel` 折叠区（Bl(x)/Cms(x) 数据表 + 对比图）
3. **JS**：所有逻辑内联在 `<script>` 标签中

### JS 模块划分

| 模块 | 入口 | 职责 |
|------|------|------|
| Complex 类 | `class Complex` | 复数四则运算，频域等效电路计算的基础 |
| 输入/转换 | `getParams()`, `toSI(p)` | 页面参数读取，自定义单位→SI (Cms÷1000, Mms÷1e6, Sd÷1e6, Le÷1000) |
| 线性计算 | `calcLinearResponse()` | 等效电路法频域求解，输出 SPL(dB) + 位移(mm) |
| 插值 | `AkimaSpline`, `SteffenSpline` | 一维保形样条，处理非对称 Bl(x)/Cms(x) |
| 非线性-快速 | `describingFunction()` | 描述函数法：等效线性化迭代 + Bl(x) 谐波提取估算 THD |
| 非线性-精细 | `rk4Solve()` + `extractHarmonics()` | RK4 时域积分→稳态检测→DFT 提取 H1-H5→THD |
| 图表 | `createChart()`, `updateLineChart()` | Chart.js 封装，对数 X 轴 |
| UI | `calcAll()`, `calcNonlinear()`, `showNonlinearResults()` | 事件绑定，预置参数切换，进度反馈 |

### 数据流

```
用户输入 TS 参数 + Vrms
  → toSI() 单位转换
  → calcDerived() 计算 Fs/Qts/Qes/Qms/Vas/SPL₀
  → calcLinearResponse() 500 点对数扫频 → {spl, disp, imp}
  → Chart.js 渲染 SPL + 位移双图
  → (可选) 展开非线性面板 → 输入 Bl(x)/Cms(x) 数据表
  → calcNonlinear() → describingFunction() 或 rk4Solve() → 线性 vs 非线性对比 + THD
```

## 领域知识要点

- **物理模型**：无限障板 = 振膜嵌入无限大刚性平面，半空间辐射。远场声压 p ∝ jω·ρ₀·Sd·v/(2πr)
- **电-机-声等效电路**：Ze(Re+Le) → Zmot(Bl²/Zm) → Zm(Rms+Mms+Cms) → 体积速度 U=Sd·v → 辐射
- **单位约定**：输入 Cms(mm/N)、Mms(mg)、Sd(mm²)，内部全转 SI 计算，位移输出用 mm
- **Akima 优于自然三次样条**：Bl(x)/Cms(x) 往往非均匀网格且非对称，Akima 局部性强不过冲
- **THD 非对称判据**：Bl(x) 非对称 → H2 显著 ≠ 0；对称非线性 → H3 为主。H2/H3 比值可诊断非线性对称性

## 设计决策记录

- 非线性面板用折叠设计而不做独立页面：保持单一工作流，线性/非线性对比一目了然
- 时域求解默认 80 频点（快速 100 点）：平衡精度与浏览器 UI 响应
- 预置参数需保持物理合理性（Qts 1-2，Fs 800-1200Hz，SPL₀ 75-85dB @ 1W/1m）
- Vas 显示用 cm³ 而非 L：微扬声器 Vas 通常在 0.1-2 cm³ 量级
