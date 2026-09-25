# Setup

## First run (about three minutes)

**1. Open the folder in VS Code.** File → Open Folder → select
`thesis-budget-transfer`. Open the whole folder, not an individual file, or the
workspace settings and run configurations will not load.

**2. Install the recommended extensions.** VS Code will prompt on first open
(Python, Pylance, Ruff, YAML, Rainbow CSV). Accept.

**3. Create the environment.** Open a terminal in VS Code (`` Ctrl+` ``) and run:

```bash
bash scripts/setup.sh          # macOS / Linux
scripts\setup.bat              # Windows
```

This creates `.venv`, installs the package in editable mode, and runs the tests.
Sixteen tests should pass. If they do, the harness is working.

**4. Select the interpreter.** `Ctrl+Shift+P` → "Python: Select Interpreter" →
choose the one inside `./.venv`. VS Code usually detects it automatically; do
this manually if imports show as unresolved.

**5. Run the pilot.**

```bash
bash scripts/run_pilot.sh      # macOS / Linux
scripts\run_pilot.bat          # Windows
```

Roughly twenty seconds. Tables land in `results/analysis/`, figures in
`results/figures/`.

## Daily workflow

| What you want | How |
|---|---|
| Run the tests | `Ctrl+Shift+P` → "Tasks: Run Test Task", or the beaker icon in the sidebar |
| Run the pilot | `Ctrl+Shift+B` (default build task) |
| Debug with breakpoints | Run and Debug panel (`Ctrl+Shift+D`) → pick a configuration → F5 |
| Step through one cell | Debug config "Single cell" — runs `--limit 1` so you are not waiting on the grid |
| Check grid size before committing hours | Debug config "Main grid: dry run" |
| Inspect a results CSV | Click it; Rainbow CSV colours the columns |

Because the package is installed editable, `python -m tsa.runner` works from any
terminal in the project root without setting `PYTHONPATH`. The `PYTHONPATH`
entries in `.vscode/settings.json` are a belt-and-braces fallback for terminals
opened before the install finishes.

## Where things go

```
configs/     experiment definitions — edit these, never the source
data/        your corpora (gitignored; see data/README.md)
results/     CSVs, analysis tables, figures (gitignored)
src/tsa/     the package
tests/       protocol invariants
scripts/     one-line entry points
```

## Version control

```bash
git init
git add .
git commit -m "Experiment harness: pilot passing, 16 tests green"
```

`.gitignore` already excludes `results/*.csv`, `data/`, and `.venv`. Commit the
small analysis tables when you produce real numbers:

```bash
git add -f results/analysis/*.csv
git commit -m "Main grid results, config hash 3062203366"
```

Every results row carries its config hash, so any table in the thesis traces to
an exact commit and an exact configuration. That pairing is what the
reproducibility appendix in week 16 is for, and it is far easier to maintain
from the start than to reconstruct later.

## Troubleshooting

**Imports unresolved / "tsa could not be resolved".** The interpreter is wrong.
`Ctrl+Shift+P` → "Python: Select Interpreter" → the `./.venv` one. Then reload
the window (`Ctrl+Shift+P` → "Developer: Reload Window").

**`ModuleNotFoundError: No module named 'tsa'` in the terminal.** The virtual
environment is not active. `source .venv/bin/activate` (or
`.venv\Scripts\activate` on Windows). The prompt should show `(.venv)`.

**Windows: "running scripts is disabled on this system".** PowerShell blocks
activation by default. Either use Command Prompt instead, or run
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once.

**`ERROR: results/pilot_results.csv exists.`** Deliberate — the runner refuses
to silently mix results from different configurations. Pass `--resume` to
continue an interrupted run, or delete the file to start clean.

**Matplotlib backend errors on a headless machine.** Already handled;
`plots.py` sets the Agg backend before importing pyplot.

**The main grid is slower than fifteen minutes.** Expected on the `files`
loader with real reviews, which are much longer than synthetic documents.
Reduce `max_features`, or `cv_folds` from 3 to 2, before reducing seeds — seeds
are what your confidence intervals are made of.
