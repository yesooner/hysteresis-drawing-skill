# Output Format Rules

## Purpose

Use this reference whenever generating Excel or CSV outputs from hysteresis
postprocessing. Calculation algorithms and output layout are separate
decisions: do not change peak extraction, stiffness, or energy formulas only
because the requested workbook layout changes.

## Required Decision

Before writing result files, choose and state one output profile:

| Profile | Use case | Workbook layout |
|---|---|---|
| `local_wide_separate_workbooks` | Default for the current machine and existing project comparison files | One metric per workbook |
| `local_wide_multisheet_workbook` | User asks for one Excel file with several sub-sheets | One workbook, one sheet per metric |
| `tidy_long_tables` | Downstream statistics, plotting, or database-style processing | One table with explicit columns |
| `legacy_report_compatible` | Reproducing an existing legacy report output | Match the legacy file exactly and label it |

If the user asks for a different format, create a short task-specific profile
and record it in the sidecar summary. Do not force the default local format.

## Header Profiles

The default local wide-table header is:

```text
row 1: 位移, 荷载 / 位移, 刚度 / 位移, 耗能 repeated by specimen
row 2: mm, kN / mm, kN/mm / mm, kN·mm repeated by specimen
row 3: blank, specimen name repeated by specimen
row 4+: numeric data
```

This header is a user habit, not an algorithm requirement. It may be adjusted
when the user requests another style. Supported adjustments include:

- Chinese or English labels.
- Actual peak displacement or nominal displacement in the first column.
- Single-cycle energy or cumulative energy in the right column.
- Sheet names such as `骨架`, `刚度`, `耗能`, `阻尼`, `元数据`.
- Additional metadata rows, only when the target profile allows them.

For final project workbooks, do not use a `loop` column unless the user
explicitly asks for loop-index output. Replace it with `位移` or another
declared displacement-level label.

## Layout Profiles

### `local_wide_separate_workbooks`

Generate separate files:

```text
skeleton_combined.xlsx
skeleton_stiffness.xlsx
skeleton_energy.xlsx
```

For time-stamped or ODB-derived inputs, keep the source prefix:

```text
<source_prefix>_skeleton_combined.xlsx
<source_prefix>_skeleton_stiffness.xlsx
<source_prefix>_skeleton_energy.xlsx
```

Each workbook contains one data sheet by default. Sidecar metadata goes into
CSV/TXT files.

### `local_wide_multisheet_workbook`

Generate one workbook with separate sub-sheets. Use this when the user says
the output should not be a single table and should be split into sub-sheets.
Recommended sheet names:

```text
骨架
刚度
耗能
阻尼
元数据
```

Do not put skeleton, stiffness, and energy into one continuous worksheet.
Each metric gets its own sheet with its own header.

### `tidy_long_tables`

Use one record per specimen and displacement level. Required columns:

```text
specimen
nominal_displacement_mm
actual_positive_displacement_mm
actual_negative_displacement_mm
positive_load_kN
negative_load_kN
secant_stiffness_kN_per_mm
cycle_energy_kN_mm
cumulative_energy_kN_mm
algorithm_mode
cycle_rule
```

This profile is preferred for auditing, plotting, and scripted comparison, but
it is not the default manuscript/table format.

## Metadata

Every output profile must preserve metadata either in a sheet or sidecar file:

- input path;
- output path;
- specimen name mapping;
- algorithm mode;
- cycle splitting rule;
- skeleton method;
- stiffness formula;
- energy definition;
- nominal displacement tolerance;
- warnings about incomplete loops or approximations.

## Encoding

Write Chinese headers as real UTF-8/Excel Unicode text. Do not write mojibake,
question marks, or shell-encoded replacement text. After generating a workbook,
inspect at least the first three rows with `openpyxl` or equivalent when the
file is intended for final reporting.
