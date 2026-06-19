---
name: hysteresis-drawing-skill
description: Postprocess and draw cyclic hysteresis data. Use when Claude Code needs to read displacement-load Excel data, extract skeleton curves, split loading cycles, compute cumulative energy, equivalent viscous damping ratio, secant stiffness degradation, bearing capacity, ductility, or draw hysteresis-related plots while distinguishing strict engineering algorithms from legacy report-compatible calculations.
---

# Hysteresis Drawing Skill

## Scope

Use this skill for cyclic-load postprocessing from displacement-load histories, skeleton curves, loop result tables, and stiffness spreadsheets.

Primary project data should be supplied by the caller. Do not hard-code a
machine-specific data root in scripts or instructions. Typical data layout:

```text
<project-data-root>
```

Before computing final values, read `references/algorithm-rules.md`.
Before writing Excel/CSV outputs, read `references/output-format-rules.md`
and choose an output profile. Output layout is configurable; calculation
formulas are not changed by workbook layout.

## Python Environment

Use a Python environment with the packages listed in `requirements.txt`:

```powershell
python -X utf8 <script.py>
```

For local development, use a project-specific virtual environment or conda
environment. Do not install packages into a shared base environment.

For local loop-result table generation, explicit input/output arguments are
required. Do not define default input/output directories in code; defaults can
silently mix fixture, project, and production results.

```powershell
python -X utf8 scripts\build_tables.py `
  --energy-dir <directory-containing-LoopAnalysisResults-csv> `
  --ramp-file <ramp.txt> `
  --output-dir <output-directory> `
  --specimens SHELL-1,SHELL-2,SHELL-3
```

Never write generated files back over reference input files. Use a dedicated
output directory for generated tables, reports, and plots.

## Data Inputs

Expected files depend on the task, but commonly include:

- raw hysteresis workbook with displacement-load column pairs;
- loop result CSV files;
- ramp displacement protocol;
- optional reference workbooks for skeleton, stiffness, or energy results.

Use paired columns:

```text
displacement, load, displacement.1, load.1, ...
```

Rows 0-2 commonly contain headers, units, and specimen labels; numeric data
usually starts at row 3 when using `header=None`. Confirm the actual workbook
layout before processing.

## Algorithm Mode

Always declare the calculation mode:

- `strict_engineering`: recommended for new analysis and manuscript-quality calculations.
- `legacy_report_compatible`: use only when explicitly reproducing older report values; label it as historical compatibility.

Do not mix values from the two modes in one table without labeling them.

## Recommended Workflow

1. Read and clean paired displacement-load arrays.
2. Preserve load history order; do not sort the raw hysteresis curve before cycle splitting.
3. For project tables, identify displacement reversal points and pair positive/negative peaks by known or inferred nominal displacement levels.
4. Use displacement zero crossings only for full-cycle loop checks; do not use adjacent load zero crossings except in `legacy_report_compatible` mode.
5. Compute full-cycle energy from a closed or nearly closed cycle. If only peak-to-peak branches exist, label energy as `closed_peak_to_peak_area`.
6. Compute equivalent viscous damping ratio from full-cycle loop area and positive/negative peak triangles.
7. Extract skeleton points using a declared method: `peak_point_skeleton` for article tables or `outer_envelope_skeleton` for envelope plots.
8. Compute secant stiffness using paired positive and negative peak values at the same nominal displacement level.
9. Compute ductility from a skeleton curve; use strict energy-equivalent bilinear yield when possible. If using a simplified yield rule, label it.
10. Cross-check every reported number against JSON/Excel/report sources and state the mode used.

GitHub algorithm check: public hysteresis tools such as `cslotboom/hysteresis`,
`GarGarfie/HysAnalysis`, and `tufailmab/Hysteresis-Analyzer` support using
reversal points, peak-point/backbone extraction, and integrated loop area for
force-displacement hysteresis analysis. This project follows that direction
for `strict_engineering` outputs.

## Output Discipline

For every generated result table, include:

- input file path;
- specimen name;
- calculation mode;
- cycle splitting rule;
- formula used;
- whether values came from raw hysteresis data, independent skeleton files, or historical JSON;
- warnings for approximations or incomplete descending branches.

When loop-level energy/stiffness audit data are available, also generate a
loop-detail workbook or CSV. The loop-detail table must align `LoopNo` with
the corresponding nominal displacement level from the loading ramp, and keep
the following columns together:

```text
LoopNo
Displacement
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

Do not use `LoopNo` alone as the displacement axis in final plots or tables.
Use the ramp-aligned `Displacement` column for displacement-based comparison,
and retain `LoopNo` only as the cycle identifier.

## Output Format

This section supersedes any older fixed-format wording below. Output format is
profile-based. The current user's default habit is
`local_wide_separate_workbooks`, but this is not mandatory when the user asks
for a different structure.

Supported profiles are defined in `references/output-format-rules.md`:

- `local_wide_separate_workbooks`: one metric per workbook, matching existing
  project files such as `skeleton_combined.xlsx`, `skeleton_stiffness.xlsx`,
  and `skeleton_energy.xlsx`.
- `local_wide_multisheet_workbook`: one workbook with separate sheets such as
  `骨架`, `刚度`, `耗能`, and `元数据`; use this when the user asks to split
  results into sub-sheets.
- `tidy_long_tables`: one row per specimen/displacement level for audit,
  plotting, and scripted comparison.
- `legacy_report_compatible`: exact old/report layout, clearly labeled.

Headers are configurable by profile. The default local header is:

```text
row 1: 位移, 荷载 / 位移, 刚度 / 位移, 耗能 repeated by specimen
row 2: mm, kN / mm, kN/mm / mm, kN·mm repeated by specimen
row 3: blank, specimen name repeated by specimen
row 4+: numeric data
```

Do not write `loop` as a final project-table column unless the user explicitly
requests loop-index output. Use `位移` or another declared displacement-level
label instead.

Write Chinese table headers as real UTF-8/Excel Unicode text, not mojibake or
`??`. When generating final reporting workbooks, verify at least the first
three rows after writing.

## Legacy Default Profile

The historical fixed layout is retained only as the default
`local_wide_separate_workbooks` profile. Do not treat it as the only allowed
output format. When exact historical files are required, follow
`references/output-format-rules.md` instead of copying old mojibake headers.

Use the separate-workbook filenames, multi-sheet layout, header choices, and
sidecar metadata names defined in `references/output-format-rules.md`. Do not
duplicate old fixed table text in this file.
