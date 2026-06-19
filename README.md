# hysteresis drawing skill

<p align="center">
  <a href="./README.md"><img alt="English" src="https://img.shields.io/badge/Language-English-blue"></a>
  <a href="./README.zh-CN.md"><img alt="中文" src="https://img.shields.io/badge/%E8%AF%AD%E8%A8%80-%E4%B8%AD%E6%96%87-lightgrey"></a>
  <a href="./LICENSE"><img alt="License" src="https://img.shields.io/badge/License-MIT-orange"></a>
</p>

Postprocess cyclic displacement-load hysteresis data into skeleton, stiffness,
energy, loop-detail, and plotting outputs for finite-element or experimental
comparison workflows.

## 1. Purpose

This skill postprocesses and draws cyclic loading results from hysteresis data.
It works with displacement-load histories, loop result tables, and optional
reference tables.

Main outputs:

- energy-displacement tables;
- stiffness-displacement tables;
- loop detail tables;
- hysteresis postprocessing plots.

All input and output paths must be passed explicitly through command-line
arguments. Scripts must not define default input directories, default output
directories, or machine-specific absolute paths.

## 2. Directory Layout

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

## 3. Environment

Use an isolated Python environment:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Dependencies:

- `openpyxl`: read and write `.xlsx` workbooks;
- `pandas`: read legacy `.xls` and table data;
- `xlrd`: pandas engine for `.xls` files;
- `numpy`: numeric support;
- `matplotlib`: PNG/PDF/SVG plot generation.

## 4. Inputs

Typical inputs:

- raw hysteresis workbook with displacement-load column pairs;
- loop result tables: `*_LoopAnalysisResults.csv`;
- loading ramp: `ramp.txt`;
- optional reference workbooks for skeleton, stiffness, or energy results.

The raw hysteresis workbook is expected to use paired columns:

```text
displacement, load, displacement, load, ...
mm, kN, mm, kN, ...
blank, specimen, blank, specimen, ...
numeric data begins on row 4
```

Loop CSV files should contain:

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

## 5. Outputs

`build_tables.py` writes:

- `skeleton_energy.xlsx`: accumulated loop energy versus displacement;
- `skeleton_stiffness.xlsx`: average secant stiffness versus displacement;
- `loop_details.xlsx`: complete loop-level audit table.

`plot_outputs.py` writes:

- plot image files into the directory passed by `--plot-dir`;
- recommended plot folder name: `plots`;
- supported formats: `png`, `pdf`, and `svg`.

## 6. Formulae And Implementation Workflow

### 6.1 LoopNo-to-Displacement Alignment

**Formula**

$$
\Delta_i = \left|\Delta^{+}_{i}\right|
$$

where $\Delta_i$ is the nominal displacement level of loop $i$, and
$\Delta^{+}_{i}$ is the $i$-th positive peak displacement in the ramp.

**Implementation workflow**

1. Read the displacement column from `ramp.txt`.
2. Skip the initial `0, 0` row.
3. Starting from the first positive peak, read every second row.
4. Build a `LoopNo -> Displacement` mapping.
5. Write both `LoopNo` and `Displacement` to `loop_details.xlsx`.
6. Use `Displacement`, not `LoopNo`, as the x-axis for displacement-based plots.

### 6.2 Loop Energy

**Formula**

$$
E_{\mathrm{loop}} = \oint F \, d\Delta
$$

For discrete data with trapezoidal integration:

$$
E_{\mathrm{loop}} \approx \left|\sum_{k=1}^{n-1}
\frac{F_k + F_{k+1}}{2}
\left(\Delta_{k+1}-\Delta_k\right)\right|
$$

If the loop table already provides `LoopEnergy`:

$$
E_{\mathrm{loop}} = \mathrm{LoopEnergy}
$$

**Implementation workflow**

1. Prefer the `LoopEnergy` field when it is available in the loop table.
2. If it is unavailable, integrate a complete closed hysteresis loop using the trapezoidal rule.
3. Mark non-closed loops or peak-to-peak branch data as approximate in metadata.
4. Preserve the original loop order.

### 6.3 Accumulated Energy-Displacement Curve

**Formula**

$$
E_{\mathrm{acc},i} = \sum_{j=1}^{i} E_{\mathrm{loop},j}
$$

The x-axis is:

$$
x_i = \Delta_i
$$

**Implementation workflow**

1. Read `LoopEnergy` in ascending `LoopNo` order.
2. Accumulate loop energy sequentially.
3. Use the `LoopNo -> Displacement` mapping for the x-axis.
4. Write `skeleton_energy.xlsx` as paired columns: `displacement`, `energy`.

### 6.4 Secant Stiffness-Displacement Curve

**Formula**

When the input table provides two half-cycle stiffness values:

$$
K_{\mathrm{avg},i} =
\frac{K_{\mathrm{1st},i}+K_{\mathrm{2nd},i}}{2}
$$

When stiffness is computed directly from positive and negative peaks:

$$
K_i =
\frac{\left|F_i^{+}\right|+\left|F_i^{-}\right|}
{\left|\Delta_i^{+}\right|+\left|\Delta_i^{-}\right|}
$$

**Implementation workflow**

1. If `1stSecantStiffness` and `2ndSecantStiffness` exist, average them by default.
2. If computing from raw peak points, pair positive and negative peaks at the same nominal displacement level.
3. Divide the sum of absolute peak loads by the sum of absolute peak displacements.
4. Do not mix the two stiffness methods in one final table unless the method is explicitly labeled.
5. Write `skeleton_stiffness.xlsx` as paired columns: `displacement`, `stiffness`.

### 6.5 Half-Cycle Energy

**Formula**

$$
E_{\mathrm{loop},i}
= E_{\mathrm{1stHalf},i}+E_{\mathrm{2ndHalf},i}
$$

Directional energy difference:

$$
\Delta E_{\mathrm{half},i}
= \left|E_{\mathrm{1stHalf},i}-E_{\mathrm{2ndHalf},i}\right|
$$

**Implementation workflow**

1. Read `1stHalfEnergy` and `2stHalfEnergy` from the loop table.
2. Preserve both values in `loop_details.xlsx`.
3. Use their sum as a consistency check against `LoopEnergy`.
4. Do not split the two half-cycle values in the default accumulated energy table.

### 6.6 Equivalent Viscous Damping Ratio

**Formula**

$$
\xi_{\mathrm{eq}} =
\frac{E_{\mathrm{loop}}}{4\pi E_{\mathrm{elastic}}}
$$

If peak-triangle elastic energy is used:

$$
E_{\mathrm{elastic}} \approx
\frac{1}{2}
\left(\left|F_i^{+}\Delta_i^{+}\right|
+\left|F_i^{-}\Delta_i^{-}\right|\right)
$$

**Implementation workflow**

1. If `1stEquivalentViscousDampRatio[%]` and `2ndEquivalentViscousDampRatio[%]` exist, preserve both directional values.
2. Do not force an average by default because positive and negative half cycles can be asymmetric.
3. If computing from raw data, compute closed-loop energy first and then equivalent elastic energy.
4. Write the result to `loop_details.xlsx`.

### 6.7 Skeleton Curve

**Formula**

Peak-point skeleton:

$$
\begin{aligned}
S_i^{+} &= \left(\Delta_i^{+},F_i^{+}\right) \\
S_i^{-} &= \left(\Delta_i^{-},F_i^{-}\right)
\end{aligned}
\qquad i=1,\ldots,m
$$

Outer-envelope skeleton:

$$
F_{\mathrm{env}}(\Delta)
= \max \left|F(\Delta)\right|
$$

**Implementation workflow**

1. `peak_point_skeleton`: extract positive and negative peak points at each displacement level.
2. `outer_envelope_skeleton`: extract the outer envelope from the full hysteresis history.
3. Use `peak_point_skeleton` for engineering comparison tables.
4. Use `outer_envelope_skeleton` for envelope figures when needed.
5. Declare the selected skeleton method in final outputs.

### 6.8 Plotting

**Formula**

Plotting does not transform the data; it draws:

$$
y = f(\Delta)
$$

where $\Delta$ is displacement, and $y$ can be load, accumulated energy,
secant stiffness, or another loop metric.

**Implementation workflow**

1. For wide tables, read two columns as one series: left column as displacement, right column as the metric.
2. Draw one curve for each specimen.
3. For `loop_details.xlsx`, draw `LoopEnergy`, `1stSecantStiffness`, and `2ndSecantStiffness` against `Displacement` by default.
4. Write image files into the directory specified by `--plot-dir`.
5. Use `--format` to choose `png`, `pdf`, or `svg`.

## 7. Command Examples

Generate tables:

```powershell
python -X utf8 scripts\build_tables.py `
  --energy-dir <loop-csv-dir> `
  --ramp-file <ramp.txt> `
  --output-dir <output-dir> `
  --specimens <name-1,name-2,name-3>
```

Generate plots:

```powershell
python -X utf8 scripts\plot_outputs.py `
  --input-dir <output-dir> `
  --plot-dir <output-dir>\plots `
  --tables skeleton_energy.xlsx,skeleton_stiffness.xlsx,loop_details.xlsx `
  --format png
```
