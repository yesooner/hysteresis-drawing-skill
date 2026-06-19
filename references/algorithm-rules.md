# Hysteresis Algorithm Rules

## Data Roots

Supply every data root at runtime. Do not hard-code machine-specific paths,
test directories, or project directories in reusable scripts.

```text
<project-data-root>
```

## Workbook And Series Rules

Input workbook layouts must be detected or declared by the caller. A common
layout is paired displacement-load columns:

```text
row 1: displacement, load, displacement, load, ...
row 2: mm, kN, mm, kN, ...
row 3: blank, specimen name, blank, specimen name, ...
row 4+: numeric data
```

For final outputs:

- preserve raw specimen names unless the caller supplies a mapping;
- do not hard-code excluded series names;
- do not hard-code article-facing names;
- keep the mapping rules in task-specific configuration or command-line
  arguments.

## Python Runtime

Use a Python interpreter with the packages listed in `requirements.txt`:

```powershell
python -X utf8 <script.py>
```

Use a dedicated virtual environment or conda environment for reproducibility.

## Output Profiles

Choose and state one output profile before writing result files:

```text
local_wide_separate_workbooks
local_wide_multisheet_workbook
tidy_long_tables
legacy_report_compatible
```

Calculation algorithms are independent from workbook layout. Do not change
peak extraction, stiffness, or energy formulas only because the requested
workbook layout changes.

## Calculation Modes

Always declare the calculation mode:

- `strict_engineering`: recommended for new analysis and manuscript-quality
  calculations.
- `legacy_report_compatible`: use only when reproducing old report values;
  label it as historical compatibility.

Do not mix values from the two modes in one table without labeling them.

## Strict Engineering Workflow

1. Read paired displacement-load arrays without sorting the raw history.
2. Preserve loading order for cycle splitting.
3. Detect displacement reversal points or use a declared loading protocol.
4. Group reversal points into nominal displacement levels.
5. Extract positive and negative peaks for each displacement level.
6. Compute full-cycle energy from a closed or nearly closed loop.
7. Compute equivalent viscous damping ratio from loop area and elastic energy.
8. Compute secant stiffness using paired positive and negative peaks.
9. Extract skeleton points using a declared method.
10. Write metadata describing all approximations.

## Nominal Displacement Level Inference

If no loading protocol is supplied:

1. Detect displacement reversal points from the raw displacement series.
2. Cluster absolute reversal displacements into nominal levels.
3. Use tolerance `max(1.2 mm, 10% nominal displacement)` unless the caller
   supplies another tolerance.
4. Record actual positive and negative reversal displacements in metadata.
5. For reporting tables, write the nominal level in the displacement column,
   but compute force, stiffness, and energy from actual peak coordinates.

## Skeleton Curve

Supported skeleton methods:

- `peak_point_skeleton`: peak points from each displacement level.
- `outer_envelope_skeleton`: outer envelope of the full hysteresis history.

Use `peak_point_skeleton` for engineering comparison tables unless the caller
asks for an envelope plot. Do not mix skeleton methods in one comparison table
without labeling them.

## Secant Stiffness

For paired positive and negative peaks at the same nominal displacement level:

```text
K_i = (|F_i+| + |F_i-|) / (|Delta_i+| + |Delta_i-|)
```

If using half-cycle stiffness values from an input loop table:

```text
K_avg = (K_1st + K_2nd) / 2
```

State which formula was used.

## Energy

For strict cycle energy, use the closed loop area:

```text
E_loop = integral(F dDelta)
```

Use trapezoidal integration for discrete data. If only peak-to-peak branches
are available, label the value as an approximation.

Accumulated energy:

```text
E_acc(i) = sum(E_loop(j)), j = 1 ... i
```

## Equivalent Viscous Damping Ratio

Use:

```text
xi_eq = E_loop / (4*pi*E_elastic)
```

When positive and negative half cycles are asymmetric, preserve directional
values instead of forcing a single average.

## Plotting

For displacement-based plots:

- use displacement, not loop index, as the x-axis;
- label units on axes;
- write plots into a caller-supplied output directory;
- do not overwrite source workbooks.

## Output Discipline

Every generated result should record:

- input paths;
- output paths;
- specimen names and optional mappings;
- calculation mode;
- cycle splitting rule;
- skeleton method;
- stiffness formula;
- energy definition;
- warnings for incomplete loops or approximations.
