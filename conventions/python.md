# Python conventions

For every Python repo, whether an app run from its checkout or a package published to PyPI. Where `all.md` is written for JavaScript tooling, this file says what it means in Python.

- **Python version**: `requires-python` in `pyproject.toml` is a lower bound (`">= 3.12"`), never a cap.
- **Environment**: Work in a `.venv` at the repo root (gitignored), with the repo installed editable: `pip install -e ".[dev]"`, adding any other extras the code needs. An editable install makes imports resolve to the checkout rather than some other installed copy.
- **`pyproject.toml` is the one config file**: the package, its dependencies, and every tool's settings (`[tool.ruff]`, `[tool.pyright]`, `[tool.pytest]`). No `setup.py`, `setup.cfg`, `requirements.txt`, or separate tool config files, so there's one place to look.
- **Dependencies**
  - **Ranges**: A lower bound only, written with `>=` down to the axis semver treats as breaking (`>=8` for a 1.0+ package, `>=0.45` for a 0.x one), for the same reason `all.md` trims a caret range. No upper bound: a cap in a published package keeps its users off a newer release that may well work, and they can't override it. Ruff is the one exception, below.
  - **Extras**: The tools only a developer runs go in a `dev` extra (Ruff, pyright, pytest, and any linter the repo adds), so a plain install doesn't pull them in. Anything heavy that only one feature imports goes in an extra named for that feature, so the rest of the package installs without it.
  - **Only what's imported**: Declare a dependency only where the repo's own code imports it, and declare everything it does import, so it never relies on one arriving through another package.
- **Formatting and linting**: Ruff is both linter and formatter. The repo's "format script", which `all.md` says to run before every commit, is `ruff check --fix .` then `ruff format .`.
  - **Line length**: Each repo picks its own, set once in `[tool.ruff]` so the formatter and linter agree.
  - **Rules**: `select = ["ALL"]`, every stable rule, with an `ignore` list in which each entry has a comment giving its reason. List whole families first, then single rules from families that are otherwise on. `extend-select` brings back any rule a family ignore would otherwise turn off.
  - **The usual ignores**: `D` (the docstrings follow `all.md` instead), `CPY`, `COM` and `ISC001` (they conflict with the formatter), and `E501` (the formatter wraps code, and what's left is long strings). A repo with more than a thin wrapper's code usually also ignores `ANN` (pyright checks the types), `PTH`, `FBT`, `EM`, `TD`, `FIX`, `C90`, `N`, `S101`, `TRY003`, `PLR2004`, `PLR09`, and `TC006`. Each still needs its own reason in the repo, since the reason is what says whether it still applies.
  - **One-off findings**: Silence a finding that's right as it is on its own line, with `# noqa: RULE` and a reason, rather than adding to the ignore list. Ignore a rule for a set of files (tests reaching into internals with `SLF001`, a CLI module printing with `T201`) through `[tool.ruff.lint.per-file-ignores]`, again with a reason.
  - **Imports**: Sorted by Ruff's isort rules into the standard library, third-party packages, then the repo's own modules, with `known-first-party` naming them and `combine-as-imports = true`. Imports are absolute from the package or repo root; no `sys.path` hacks to make a file importable.
  - **Pinned below its next minor**: `ruff>=0.17,<0.18` in the `dev` extra. Only a minor release adds stable rules or changes the formatting, and with `ALL` selected, a new rule would otherwise fail CI on an unrelated PR. Moving to the next minor is its own PR, deciding for each new rule whether to fix what it flags or ignore it.
- **Type checking**: pyright, configured in `[tool.pyright]`, which VS Code's Pylance reads too. Setting `exclude` replaces pyright's defaults, so restate them alongside `build` and `dist`: `["build", "dist", "**/__pycache__", "**/.*"]` (pyright doesn't read `.gitignore`, so a stale `build/` is otherwise checked as a second copy of the source).
  - **Silencing a finding**: `# pyright: ignore[rule]` with a reason, never a bare `# type: ignore`, which hides every error on the line.
- **Typing**
  - **Modern syntax**: Built-in generics (`list[str]`, `dict[str, int]`) and `X | None` rather than `typing.List` or `Optional`, and a `type` statement for an alias where `requires-python` allows it (3.12+).
  - **Structured data**: A `@dataclass` for a value the code builds, a `TypedDict` for a dict whose shape comes from outside (an API response, a config file), and a `StrEnum` for a fixed set of strings, so a typo is a type error.
  - **Imports only for types**: Import under `if TYPE_CHECKING:` when a module needs another only for its annotations, to avoid an import cycle or loading a heavy module at runtime.
- **Layout**
  - **Packages**: The src layout, `src/<import_name>/`, found through `[tool.setuptools.packages.find]` with `where = ["src"]`. The import name is the distribution name in snake_case (`serato-tools` imports as `serato_tools`).
  - **Apps run from their checkout**: List the packages and top-level modules in `[tool.setuptools]`, so the editable install makes them importable from anywhere in the venv, and a script in a subfolder runs directly as well as with `python -m`.
- **Logging**: A package logs through its module's logger, `logging.getLogger(__name__)`, never the root logger, so the app using it decides what's shown. An app may configure and log through the root logger.
- **Testing**: pytest, with `testpaths` set in `pyproject.toml` and run from the repo root.
  - **Style**: Plain test functions with bare `assert`s (with a message where it says what a check means), fixtures rather than setup methods, and `pytest.mark.parametrize` rather than a loop over cases. A class groups tests only where they share a fixture of their own.
  - **Fixtures**: Shared ones go in a `conftest.py`. Use `tmp_path` for files and `monkeypatch` to stand in for the network, never real user data or a real account.
  - **Platform-specific tests**: A test that only makes sense on one OS is skipped on the others with `pytest.mark.skipif`, beside a matching test for the other OS family where one applies.
- **CI**: `.github/workflows/ci.yml`, with `fail-fast: false` on every matrix, so one failure doesn't hide the rest.
  - **pyright on every supported version**: It checks against the running interpreter, so the lowest version catches use of anything newer and later ones catch deprecations and removals. The matrix runs from `requires-python` up to the newest version `setup-python` offers.
  - **Ruff once**: It checks against `requires-python`, not the running interpreter, so it runs only on one matrix entry: `ruff check --output-format=github .` and `ruff format --check --diff .`.
  - **Tests on every OS the package supports**, crossed with every supported version. An app, or a package for one OS, runs on that OS alone.
- **Editor**: Commit `.vscode/settings.json` with the venv as the interpreter (`"python.venvPath": ".venv"`), Ruff as the formatter on save, and `"ruff.importStrategy": "fromEnvironment"` so the editor uses the pinned Ruff rather than the extension's own. `.vscode/extensions.json` recommends `ms-python.python`, `ms-python.vscode-pylance`, and `charliermarsh.ruff`.
