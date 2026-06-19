# Runtime Environment

This portable copy was checked against the local Excel-processing runtime.

Third-party packages imported by the scripts:

- `openpyxl==3.1.5`: read/write `.xlsx` files.
- `pandas==3.0.3`: read raw hysteresis `.xls` files and table data.
- `xlrd==2.0.2`: pandas engine for legacy `.xls` input.
- `numpy==2.4.6`: available in the runtime and used indirectly by pandas.
- `matplotlib>=3.8`: recommended for PNG/PDF plot generation.

`matplotlib` is not needed for table-only generation, but it is recommended
for this release candidate because the skill name and workflow include
hysteresis, skeleton, stiffness, and energy drawing.

Example setup:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Example plot run:

```powershell
python -X utf8 scripts\plot_outputs.py `
  --input-dir <directory-containing-generated-xlsx> `
  --plot-dir <plot-output-directory> `
  --tables skeleton_energy.xlsx,skeleton_stiffness.xlsx,loop_details.xlsx
```

Example test run:

```powershell
$env:HYSTERESIS_TEST_DATA_DIR="<fixture-directory>"
$env:HYSTERESIS_RAMP_FILE="<ramp.txt>"
python -X utf8 tests\test_loop_table_generation.py
```
