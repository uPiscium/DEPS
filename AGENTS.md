# AGENTS.md

Agent playbook for this repository.
Use this file as the default operating guide for coding, validation, and style decisions.

## Repository Snapshot

- Project root: `DEPS`
- Primary language: Python (>=3.9)
- Runtime pattern: script entrypoint + Hydra config overrides
- Main entrypoint: `main.py`
- Core modules: `controller.py`, `planner.py`, `selector.py`, `src/models`, `src/utils`
- Configs: `configs/`
- Prompt/data files: `data/`
- Main dependencies: PyTorch, MineDojo, MineCLIP, Ray RLlib, Hydra, OpenAI SDK

## Cursor and Copilot Rules

Checked locations in this repository:

- `.cursorrules`: not present
- `.cursor/rules/`: not present
- `.github/copilot-instructions.md`: not present

There are currently no additional Cursor/Copilot instruction files to merge.

## Environment Setup

- Create env: `conda create -n planner python=3.9`
- Activate env: `conda activate planner`
- Install Python deps: `python -m pip install -r requirements.txt`
- Install MineCLIP: `python -m pip install git+https://github.com/MineDojo/MineCLIP`
- Install simulator dependency from CraftJarvis `MC-Simulator` (see `README.md`)

## Build, Lint, and Test Commands

This repo has no formal build system config (`Makefile`, `pyproject.toml`, `tox.ini`, `pytest.ini` not found).
Use the commands below.

### Build / Syntax Checks

- Full compile smoke check: `python -m compileall .`
- Targeted compile check: `python -m py_compile main.py controller.py planner.py selector.py`

### Lint / Format

- No pinned linter or formatter config is committed.
- Preferred optional lint (if installed): `ruff check .`
- Preferred optional format (if installed): `black .`
- Run tools on touched files when possible to avoid noisy churn.

### Tests

- No `tests/` directory or `test_*.py` suite is currently present.
- Use runtime smoke checks as validation:
  - `python main.py`
  - `python main.py eval.task_name=obtain_planks`

### Running a Single Test (Important)

When a test suite exists, use one of these patterns:

- Single file (pytest): `pytest path/to/test_file.py`
- Single test function: `pytest path/to/test_file.py::test_function_name`
- Single class method: `pytest path/to/test_file.py::TestClassName::test_method_name`
- Keyword selection: `pytest -k "keyword"`
- Single unittest test: `python -m unittest path.to.test_module.TestClass.test_method`

If asked to run a single test today, first state that no test suite is present, then run the closest smoke check instead.

## Common Runtime Commands

- Default run: `python main.py`
- Change task: `python main.py eval.task_name=obtain_stick`
- Change loops/workers: `python main.py eval.goal_ratio=1 eval.num_workers=0`
- Load checkpoint: `python main.py model.load_ckpt_path=/abs/path/to/checkpoint.pt`

## Architecture Notes

- `main.py`: builds `Evaluator`, loads task metadata, runs `single_task_evaluate()`
- `planner.py`: LLM planning/replanning, OpenAI key rotation, plan parsing
- `controller.py`: craft/mine control logic and policy/model wrappers
- `selector.py`: interface placeholder; methods intentionally raise `NotImplementedError`
- `src/models/simple.py`: `SimpleNetwork` policy architecture

## Code Style Guidelines

Follow existing style in each touched file and avoid unrelated rewrites.

### Imports

- Keep imports at module top unless there is a strong lazy-load reason.
- Group/order imports as standard library, third-party, local modules.
- Prefer explicit imports for all new code.
- `main.py` currently contains wildcard imports; do not add new wildcard imports elsewhere.
- Remove unused imports when you touch nearby code.

### Formatting

- Use PEP 8-compatible formatting and readable spacing.
- Target approximately 100-char line length unless local style clearly differs.
- Preserve existing quote style and layout within the file.
- Avoid broad formatting-only diffs unless explicitly requested.

### Types and Signatures

- Add type hints for new/edited functions when practical.
- Follow local conventions (`List`, `Dict`, `Tuple`, `Optional`, etc.) where already used.
- Keep annotations incremental; do not launch repository-wide typing refactors.
- Prefer clear return types on public methods and utility helpers.

### Naming Conventions

- Variables/functions/methods: `snake_case`
- Classes: `PascalCase`
- Constants/module-level immutable values: `UPPER_CASE`
- Config keys (Hydra/YAML): lowercase, stable, descriptive
- Use domain names aligned with current code (`goal`, `inventory`, `precondition`, `horizon`)

### Error Handling

- Use `NotImplementedError` for intentionally unimplemented interfaces.
- Use `assert` for programmer invariants, not runtime API/network failures.
- Prefer targeted `try/except Exception as e` around external boundaries (OpenAI/env I/O).
- Never swallow exceptions silently; log enough context to debug.
- Preserve retry semantics where they already exist (for unstable API calls).

### Logging and Console Output

- Match existing style (`print` and `rich.print` are both used).
- Keep logs concise and structured (`[INFO]`, `[ERROR]`, `[Progress]`).
- Log task/goal transitions and failures; avoid per-step spam unless debugging.

### Config, Paths, and I/O

- Prefer Hydra overrides to hardcoded runtime values.
- Use repository-relative paths for committed defaults.
- Avoid machine-specific absolute paths in code and configs.
- Keep file reads/writes explicit (`with open(...)`) and consistent with existing patterns.

### Dependencies

- Add new packages only when necessary for the task.
- Update `requirements.txt` minimally and intentionally.
- Do not opportunistically bump versions unrelated to the requested change.

### Security and Sensitive Files

- Treat `data/openai_keys.txt` as secret input.
- Never print, log, or commit keys/tokens/credentials.
- Be careful with generated artifacts under `logs/`, `recordings/`, and `outputs/`.

## Editing Workflow for Agents

- Keep edits small, focused, and behavior-preserving by default.
- Respect existing partially implemented interfaces (notably `selector.py`).
- Do not rewrite unrelated code to satisfy style preferences.
- Run at least one validation command after meaningful edits.
- Report exactly what was validated and what was not possible to validate.

## Quick Command Cheat Sheet

- Install deps: `python -m pip install -r requirements.txt`
- Default run: `python main.py`
- Task override: `python main.py eval.task_name=obtain_planks`
- Compile-all: `python -m compileall .`
- Future single-test example: `pytest tests/test_example.py::test_case`
