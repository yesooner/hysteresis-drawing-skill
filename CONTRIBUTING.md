# Contributing

## Development Setup

Use an isolated Python environment:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

## Rules

- Keep all input and output paths explicit in command-line arguments.
- Do not commit generated workbooks, plots, local run outputs, credentials, or machine-specific paths.
- Keep calculation formulas independent from workbook layout.
- Update `README.md` and `README.zh-CN.md` when user-facing behavior changes.

## Verification

Run the test suite with fixture paths supplied through environment variables:

```powershell
$env:HYSTERESIS_TEST_DATA_DIR='<fixture-dir>'
$env:HYSTERESIS_RAMP_FILE='<ramp.txt>'
python -X utf8 tests\test_loop_table_generation.py
```
