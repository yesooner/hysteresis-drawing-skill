# hysteresis drawing skill

<p align="center">
  <a href="./README.md"><img alt="English" src="https://img.shields.io/badge/Language-English-lightgrey"></a>
  <a href="./README.zh-CN.md"><img alt="中文" src="https://img.shields.io/badge/%E8%AF%AD%E8%A8%80-%E4%B8%AD%E6%96%87-blue"></a>
  <a href="./LICENSE"><img alt="License" src="https://img.shields.io/badge/License-MIT-orange"></a>
</p>

本技能用于低周往复加载试验或有限元结果的滞回数据后处理，可从位移-荷载历史、循环结果表和加载位移制度生成骨架曲线、耗能曲线、刚度退化曲线、循环明细表和图像文件。

所有输入目录和输出目录都必须通过命令行显式传入。脚本不得写入默认输入目录、默认输出目录或本机绝对路径。

## 1. 功能定位

主要输出包括：

- 耗能-位移表；
- 刚度-位移表；
- loop 详细总表；
- 滞回后处理曲线图。

## 2. 目录结构

```text
hysteresis-drawing-skill/
  SKILL.md
  requirements.txt
  runtime_environment.md
  README.md
  README.zh-CN.md
  references/
    algorithm-rules.md
    output-format-rules.md
  scripts/
    build_tables.py
    verify_reference_outputs.py
    plot_outputs.py
  tests/
    test_loop_table_generation.py
```

## 3. 运行环境

建议使用独立 Python 环境：

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

依赖说明：

- `openpyxl`：读取和写入 `.xlsx` 表格；
- `pandas`：读取旧版 `.xls` 和表格数据；
- `xlrd`：作为 pandas 读取 `.xls` 的引擎；
- `numpy`：数值处理；
- `matplotlib`：输出 PNG/PDF/SVG 曲线图。

## 4. 输入数据

典型输入包括：

- 原始滞回总表：位移、荷载成对列；
- loop 结果表：`*_LoopAnalysisResults.csv`；
- ramp 位移制度：`ramp.txt`；
- 可选参考表：骨架曲线、刚度-位移或耗能-位移结果。

原始滞回总表的典型格式：

```text
位移, 荷载, 位移, 荷载, ...
mm, kN, mm, kN, ...
空白, 构件名, 空白, 构件名, ...
数据从第 4 行开始
```

loop 结果表应包含：

```text
LoopNo.
1stHalfEnergy
2stHalfEnergy
LoopEnergy
EnergyRatio[%]
AccumulatedEnergyRatio[%]
1stMaxDisp
1stMaxDispForce
2ndMaxDisp
2ndMaxDispForce
1stEquivalentViscousDampRatio[%]
2ndEquivalentViscousDampRatio[%]
1stSecantStiffness
2ndSecantStiffness
```

## 5. 输出数据

`build_tables.py` 输出：

- `skeleton_energy.xlsx`：累计耗能-位移；
- `skeleton_stiffness.xlsx`：平均割线刚度-位移；
- `loop_details.xlsx`：loop 详细总表。

`plot_outputs.py` 输出：

- 图像文件，输出到 `--plot-dir` 指定目录；
- 推荐图像目录名：`plots`；
- 支持格式：`png`、`pdf`、`svg`。

## 6. 曲线公式与实现流程

### 6.1 LoopNo 与位移对齐

**公式**

$$
\Delta_i = \left|\Delta^{+}_{i}\right|
$$

其中，$\Delta_i$ 为第 $i$ 个 loop 对应的名义位移级，$\Delta^{+}_{i}$ 为 ramp 中第 $i$ 个正向峰值位移。

**实现流程**

1. 读取 `ramp.txt` 的位移列。
2. 跳过第一行初始点 `0, 0`。
3. 从第一个正向峰值开始，每隔两行读取一次正向峰值。
4. 建立 `LoopNo -> Displacement` 映射。
5. 在 `loop_details.xlsx` 中同时写入 `LoopNo` 和 `Displacement`。
6. 位移类图表使用 `Displacement` 作为横坐标，不直接使用 `LoopNo`。

### 6.2 单圈耗能

**公式**

$$
E_{\mathrm{loop}} = \oint F \, d\Delta
$$

离散数据采用梯形积分时：

$$
E_{\mathrm{loop}} \approx \left|\sum_{k=1}^{n-1}
\frac{F_k + F_{k+1}}{2}
\left(\Delta_{k+1}-\Delta_k\right)\right|
$$

若输入 loop 表已经提供 `LoopEnergy`，则：

$$
E_{\mathrm{loop}} = \mathrm{LoopEnergy}
$$

**实现流程**

1. 优先读取 loop 表中的 `LoopEnergy` 字段。
2. 若没有该字段，则按完整闭合滞回环的位移-荷载点进行梯形积分。
3. 对未闭合循环或只有峰值分支的数据，在元数据中标记为近似。
4. 输出单圈耗能时保留原始 loop 顺序。

### 6.3 累计耗能-位移

**公式**

$$
E_{\mathrm{acc},i} = \sum_{j=1}^{i} E_{\mathrm{loop},j}
$$

横坐标为：

$$
x_i = \Delta_i
$$

**实现流程**

1. 按 `LoopNo` 升序读取每圈 `LoopEnergy`。
2. 逐圈累加得到 `AccumulatedEnergy`。
3. 使用 `LoopNo -> Displacement` 映射取得横坐标。
4. 输出到 `skeleton_energy.xlsx`，每个构件占两列：`位移`、`耗能`。

### 6.4 割线刚度-位移

**公式**

当输入表提供两个半循环刚度时：

$$
K_{\mathrm{avg},i} =
\frac{K_{\mathrm{1st},i}+K_{\mathrm{2nd},i}}{2}
$$

当从正负峰值直接计算同一级位移割线刚度时：

$$
K_i =
\frac{\left|F_i^{+}\right|+\left|F_i^{-}\right|}
{\left|\Delta_i^{+}\right|+\left|\Delta_i^{-}\right|}
$$

**实现流程**

1. 若 loop 表包含 `1stSecantStiffness` 和 `2ndSecantStiffness`，默认计算二者平均值。
2. 若从原始峰值点计算，则先配对同一级位移的正负峰值。
3. 使用正负峰值荷载绝对值之和除以正负峰值位移绝对值之和。
4. 两种刚度算法不能混用在同一最终表中，除非在元数据中明确标注。
5. 输出到 `skeleton_stiffness.xlsx`，每个构件占两列：`位移`、`刚度`。

### 6.5 半循环耗能

**公式**

$$
E_{\mathrm{loop},i}
= E_{\mathrm{1stHalf},i}+E_{\mathrm{2ndHalf},i}
$$

方向耗能差可表示为：

$$
\Delta E_{\mathrm{half},i}
= \left|E_{\mathrm{1stHalf},i}-E_{\mathrm{2ndHalf},i}\right|
$$

**实现流程**

1. 从 loop 表读取 `1stHalfEnergy` 和 `2stHalfEnergy`。
2. 两个半循环值原样写入 `loop_details.xlsx`。
3. 用两者之和检查 `LoopEnergy` 是否一致。
4. 不在默认累计耗能表中强制拆分两个半循环。

### 6.6 等效黏滞阻尼比

**公式**

$$
\xi_{\mathrm{eq}} =
\frac{E_{\mathrm{loop}}}{4\pi E_{\mathrm{elastic}}}
$$

若使用峰值三角形近似等效弹性能：

$$
E_{\mathrm{elastic}} \approx
\frac{1}{2}
\left(\left|F_i^{+}\Delta_i^{+}\right|
+\left|F_i^{-}\Delta_i^{-}\right|\right)
$$

**实现流程**

1. 若输入表提供 `1stEquivalentViscousDampRatio[%]` 和 `2ndEquivalentViscousDampRatio[%]`，则原样保留两个方向的值。
2. 默认不强制平均两个方向，因为正负半循环可能不对称。
3. 若从原始滞回环计算，先计算闭合环耗能，再计算等效弹性能。
4. 输出到 `loop_details.xlsx`。

### 6.7 骨架曲线

**公式**

峰值点骨架：

$$
\begin{aligned}
S_i^{+} &= \left(\Delta_i^{+},F_i^{+}\right) \\
S_i^{-} &= \left(\Delta_i^{-},F_i^{-}\right)
\end{aligned}
\qquad i=1,\ldots,m
$$

外包络骨架：

$$
F_{\mathrm{env}}(\Delta)
= \max \left|F(\Delta)\right|
$$

**实现流程**

1. `peak_point_skeleton`：按位移级识别每一级正负峰值点。
2. `outer_envelope_skeleton`：在完整滞回历史中提取外包络点。
3. 工程对比表建议使用 `peak_point_skeleton`。
4. 包络图可使用 `outer_envelope_skeleton`。
5. 最终表格必须声明所用骨架算法。

### 6.8 绘图

**公式**

绘图本身不改变数据，仅绘制：

$$
y = f(\Delta)
$$

其中，$\Delta$ 为位移，$y$ 可为荷载、累计耗能、割线刚度或其他 loop 指标。

**实现流程**

1. 对宽表，按两列一组读取：左列为位移，右列为指标。
2. 每个构件绘制一条曲线。
3. 对 `loop_details.xlsx`，默认绘制 `LoopEnergy`、`1stSecantStiffness` 和 `2ndSecantStiffness` 随 `Displacement` 的变化。
4. 图像输出到 `--plot-dir` 指定目录。
5. 文件格式由 `--format` 指定，可为 `png`、`pdf` 或 `svg`。

## 7. 命令示例

生成表格：

```powershell
python -X utf8 scripts\build_tables.py `
  --energy-dir <loop-csv-dir> `
  --ramp-file <ramp.txt> `
  --output-dir <output-dir> `
  --specimens <name-1,name-2,name-3>
```

绘图：

```powershell
python -X utf8 scripts\plot_outputs.py `
  --input-dir <output-dir> `
  --plot-dir <output-dir>\plots `
  --tables skeleton_energy.xlsx,skeleton_stiffness.xlsx,loop_details.xlsx `
  --format png
```
