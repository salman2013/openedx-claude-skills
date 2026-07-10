---
name: modernize-python-tooling
description: Migrate an openedx Python package or platform repo from legacy tooling (pip-tools, setup.cfg/setup.py) to modern tooling (uv, pyproject.toml PEP 621/735, python-semantic-release).
---

Migrate an openedx Python package or platform repo from legacy tooling (pip-tools, setup.cfg/setup.py) to modern tooling (uv, pyproject.toml PEP 621/735, python-semantic-release).

## Usage
```
/modernize-python-tooling [repo-path]
```
If no path is provided, operate on the current working directory.

## Context

The openedx org is standardizing Python tooling. Reference implementation: `openedx/sample-plugin` — specifically `backend/pyproject.toml`, `backend/tox.ini`, `backend/Makefile`, `.github/workflows/backend-ci.yml`.

**Two repo types are supported:**

**Library repos** (PyPI packages):
- Published (or intended) to PyPI
- Use `pip-compile` / `requirements/*.txt`
- Use `setup.py` or `setup.cfg` (or `pyproject.toml` with `dynamic = ["dependencies"]`)
- All three phases apply.

**Application/platform repos** (not published to PyPI):
- Large Django applications like `edx-platform`
- Use `pip-compile` / `requirements/*.txt` with a layered structure (base, testing, development, etc.)
- May already have a `pyproject.toml` with minimal `[project].dependencies`
- **Phase 3 (semantic-release) does not apply** — skip it entirely.
- `[project].dependencies` stays minimal (e.g. `["setuptools"]`); runtime deps go in a `base` dependency group.
- Map the existing layered requirements structure to dependency groups (see Phase 2 guidance below).

**Auto-detect repo type**: if there is no `setup.py`/`setup.cfg` and `[project].dependencies` is minimal (≤ 3 packages), treat it as an application repo.

## Behavior

Work through the three phases in order. At the start, read existing `setup.cfg`, `setup.py`, `pyproject.toml`, `tox.ini`, `Makefile`, `requirements/*.in`, and `.github/workflows/*.yml` to understand current state before making changes.

**CRITICAL — assess before touching anything:**
Before making any change, compare the current state of each file against the phase checklist. For every item, mark it as already done or missing. Only implement what is genuinely missing. Never modify a file that already satisfies its phase requirement. If the user asks why you are changing a working file, stop and explain — do not proceed without their confirmation.

---

### Phase 1 — Consolidate metadata into `pyproject.toml`

1. Read `setup.cfg` / `setup.py` to extract: `name`, `version`, `description`, `long_description`, `author`, `license`, `classifiers`, `python_requires`, `install_requires`, `extras_require`, `packages`, `entry_points`
2. Create/update `pyproject.toml`:
   - `[build-system]`: use `setuptools>=64` and `setuptools-scm>=8.0`
   - `[project]`: static `dependencies` list (not dynamic from requirements file); `dynamic = ["version"]`
     - Framework dependencies (Django, etc.) must include a lower bound matching the oldest tested version (e.g. `"django>=4.2"`) — never unconstrained
     - Include framework classifiers matching tested versions (e.g. `"Framework :: Django :: 4.2"`, `"Framework :: Django :: 5.2"`)
   - `[tool.setuptools_scm]`: omit `root` setting (unlike sample-plugin which uses a subdirectory)
   - `[tool.setuptools.packages.find]`: set `where` if needed
   - `[tool.setuptools.exclude-package-data]`: exclude test files from the wheel — use **kebab-case** (not underscore, which causes a setuptools schema validation error):
     ```toml
     [tool.setuptools.exclude-package-data]
     "*" = ["tests*", "*.tests*", "spec*", "*.spec*"]
     ```
   - `[tool.ruff]`: set `line-length = 120`, `target-version = "py312"`
   - `[tool.ruff.lint]`: use expanded select with inline comments and `ignore = ["E501"]`:
     ```toml
     select = [
         "E",      # pycodestyle errors
         "W",      # pycodestyle warnings
         "F",      # pyflakes
         "I",      # isort
         "B",      # flake8-bugbear
         "C4",     # flake8-comprehensions
         "UP",     # pyupgrade
         "DJ",     # flake8-django
     ]
     ignore = [
         "E501",   # line too long (handled by formatter)
     ]
     ```
   - `[tool.ruff.lint.isort]`: set `known-third-party = ["django", "xblock"]` (adjust for the repo's main frameworks)
   - `[tool.ruff.format]`: set `quote-style = "double"` and `indent-style = "space"`
   - `[tool.coverage.run]`: set `branch = true`, `source`, and `omit` patterns:
     ```toml
     omit = ["*/tests/*", "*/migrations/*", "*/__pycache__/*", "*/settings.py"]
     ```
   - `[tool.coverage.report]`: set `show_missing = true` and `exclude_lines`. Only add `fail_under` if the original `.coveragerc` already had one — **do not invent a threshold that didn't exist before**:
     ```toml
     exclude_lines = [
         "pragma: no cover",
         "def __repr__",
         "raise AssertionError",
         "raise NotImplementedError",
         "if __name__ == .__main__.:",
         "if TYPE_CHECKING:",
     ]
     ```
   - `[tool.coverage.html]`: set `directory = "htmlcov"`
3. Delete `setup.py` and `setup.cfg`
4. Remove hardcoded `__version__` from `__init__.py` — with `setuptools-scm` deriving the version from git tags, this will be permanently stale after any new tag. If the version is needed at runtime, use `importlib.metadata` instead:
   ```python
   from importlib.metadata import version, PackageNotFoundError
   try:
       __version__ = version("your-package-name")
   except PackageNotFoundError:
       pass
   ```
   Also update `docs/conf.py` if it reads `__version__` via regex from `__init__.py` — replace the `get_version()` helper with:
   ```python
   from importlib.metadata import version, PackageNotFoundError
   try:
       VERSION = version("your-package-name")
   except PackageNotFoundError:
       VERSION = "unknown"
   ```
   Remove the now-unused `re` import and any `sys.path.append` that was only there to import the package for version reading.

**Key rule:** `dependencies` must be a static list in `[project]`, not `dynamic = ["dependencies"]`.

---

### Phase 2 — Switch dependency management to uv

1. Read all `requirements/*.in` files to understand current deps
2. Add `[dependency-groups]` to `pyproject.toml`. Groups depend on repo type:

   **Library repos** — standard groups:
   - `test-base`: pytest, coverage, etc. (no version pins — let uv.lock handle it)
   - `test`: `{include-group = "test-base"}` + any test extras
   - `quality`: pylint, ruff, edx-lint, etc.
   - `doc`: Sphinx and doc-building deps
   - `ci`: `{include-group = "test"}` + `{include-group = "quality"}` + tox, tox-uv
   - `dev`: `{include-group = "ci"}` + `{include-group = "doc"}` + dev conveniences
   - Add version-matrix groups if the repo tests multiple framework versions

   **Application/platform repos** — extended groups:
   - `base`: all runtime deps from the production requirements file (e.g. `kernel.in` / `base.in`), including any GitHub-hosted deps. These stay out of `[project].dependencies` because the platform is not a distributable library.
   - `bundled`: optional/third-party add-ons that extend the platform (e.g. third-party XBlocks, integrations). Include `{include-group = "base"}`.
   - `test`: `{include-group = "base"}` + testing-specific deps (from `testing.in`, excluding its `-r base.txt` line)
   - `coverage`: coverage measurement tools (from `coverage.in`)
   - `quality`: linting tools — pylint, ruff, edx-lint, import-linter, etc. Include `{include-group = "test"}`.
   - `doc`: Sphinx + doc deps. Include `{include-group = "base"}`.
   - `assets`: frontend build deps (from `assets.in`) — no include needed, standalone group.
   - `semgrep`: semgrep and related tools (from `semgrep.in`) — standalone group.
   - `ci`: tox + tox-uv only (minimal — CI installs this first, then calls `uv run tox`).
   - `dev`: `{include-group = "test"}` + `{include-group = "doc"}` + `{include-group = "assets"}` + dev-only tools (django-debug-toolbar, mypy stubs, click utilities, etc.).
   - When reading `.in` files, strip `-r other.txt` / `-r other.in` references (replaced by `include-group`) and strip `-c constraints.txt` lines (handled via `[tool.uv].constraint-dependencies`).
   - Add version-matrix groups if the repo tests multiple framework versions
3. Add `[tool.edx_lint]` section with `uv_constraints` for any repo-specific version overrides
4. Run `edx_lint write_uv_constraints` to populate `[tool.uv].constraint-dependencies` — this is machine-managed, never edit it directly
5. Run `uv lock` to generate `uv.lock`
6. Delete the `requirements/` directory
7. Update `tox.ini`:
   - Add `requires = tox-uv>=1`
   - Change runner to `uv-venv-lock-runner`
   - Replace `deps =` with `dependency_groups =` referencing the groups above
   - Remove `pip install` steps
8. Update `Makefile`:
   - `upgrade`: run `uv lock --upgrade` (not `pip-compile`)
   - `compile-requirements`: remove or replace with `uv lock`
   - `requirements`: run `uv sync --group dev` only — do NOT add `uv tool install tox --with tox-uv` (installs unpinned global tox; use `uv run tox` instead)
   - All tox-invoking targets (`lint`, `test`, `docs`): use `uv run tox` not bare `tox`
9. Update `.github/workflows/python-tests.yml` CI file:
   - Use `astral-sh/setup-uv` pinned to a full commit SHA (e.g. `@08807647e7069bb48b6ef5acd8ec9567f424441b # v8.1.0`) — never a floating tag like `@v6`
   - Pass `enable-cache: true` and `python-version: "${{ matrix.python-version }}"` to `setup-uv` — this handles Python installation, so **remove** any separate `actions/setup-python` step
   - Replace `pip install -r requirements/ci.txt` with `uv sync --group ci`
   - Replace bare `tox` calls with `uv run tox -e ${{ matrix.toxenv }}`
   - Set matrix job `name: ${{ matrix.toxenv }}` so failed jobs are immediately identifiable in the GitHub UI
   - Add `permissions: contents: read` at the job level
   - Add `workflow_call:` trigger so `release.yml` can reuse it

---

### Phase 3 — Add semantic-release (library repos only — skip for application repos)

**If this is an application/platform repo, stop after Phase 2. Do not add semantic-release.**



1. Add `[tool.semantic_release]` to `pyproject.toml`:
   - Set `version_toml` to point at `[project].version` (but version is dynamic via setuptools-scm — use `build_command` to set `SETUPTOOLS_SCM_PRETEND_VERSION`)
   - `build_command`: must use `python -m build` (not `uv build`) — `python-semantic-release` does not have uv available in its action environment:
     ```
     build_command = "python -m pip install --upgrade build && SETUPTOOLS_SCM_PRETEND_VERSION=$NEW_VERSION python -m build"
     ```
   - Do NOT override `minor_tags` unless explicitly required
2. Create `.github/workflows/release.yml` with this exact structure:
   - `run_tests` job: `uses: ./.github/workflows/python-tests.yml` (reusable — no inline matrix)
     - Add `secrets: inherit` so `CODECOV_TOKEN` and other secrets are passed to the called workflow
     - Add `permissions: contents: read`
   - `release` job (`needs: run_tests`, `if: github.ref_name == 'main'`):
     - `concurrency` with `cancel-in-progress: false`
     - Checkout with `ref: ${{ github.ref_name }}` + `git reset --hard ${{ github.sha }}`
     - **No `setup-uv` step** — PSR's action environment does not use uv; `build_command` installs `build` via pip
     - PSR action: `python-semantic-release/python-semantic-release@v10.5.3`
     - PSR params: `git_committer_name`, `git_committer_email`, `changelog: "false"`
     - Upload built `dist/` as GitHub Actions artifact
     - Expose `released` + `version` outputs
     - `permissions: contents: write` only (NOT `id-token`)
   - `publish_to_pypi` job (`needs: release`, `if: released == 'true'`):
     - Download artifact, publish via OIDC (`id-token: write`)
3. Update `.github/workflows/python-tests.yml`: add `workflow_call:` trigger, remove `push: [main]` trigger (release.yml owns that path — avoids running tests twice on main)
4. Create `.github/workflows/commitlint.yml`: enforce conventional commit format on PRs
4. Remind the user to confirm PyPI trusted publisher (OIDC) is configured for the repo — this requires a manual step in PyPI settings

---

## Committing

After completing each phase, **pause and show the user a summary of all changes made**, then ask for explicit approval before committing. Only commit once the user confirms.

Suggested review prompt after each phase:
> "Here's what changed in Phase N — please review before I commit:"
> (list files created/modified/deleted with a one-line description of each change)
> "Shall I commit with message: `<commit message>`?"

Once approved, create a separate git commit per phase:

- **Phase 1 commit:** `chore: migrate package metadata to pyproject.toml (PEP 621)`
- **Phase 2 commit:** `chore: switch dependency management from pip-tools to uv`
- **Phase 3 commit (library repos only):** `chore: add python-semantic-release and commitlint workflows`

Use conventional commit format. Do not squash phases into a single commit.

---

## Important rules

- When editing `README.rst`, do NOT introduce new RST heading levels that conflict with existing ones. RST infers heading levels from the first occurrence of each underline character in the document. Adding a new sub-section level (e.g. `^^^^`) under a section that already has sibling sections using a different character (e.g. `----`) will cause `doc8` to report `D000 Title level inconsistent`. Always match the underline character used by existing sub-sections at the same depth.
- `[tool.uv].constraint-dependencies` is machine-managed — NEVER edit it directly. Repo-specific overrides go in `[tool.edx_lint].uv_constraints` only.
- Use `uv run tox` in CI — `uv sync` does not put tools on PATH.
- If new framework version drops older Python, bump `requires-python` and remove that Python from tox envlist.
- Do NOT set `root` in `[tool.setuptools_scm]` (unlike sample-plugin which needs it for subdirectory layout).
- Do NOT set `minor_tags` in `[tool.semantic_release]` unless the user explicitly requests it.
- `build_command` in `[tool.semantic_release]` must use `python -m build`, never `uv build`. The `python-semantic-release` action does not have uv available. Install the `build` package via pip inside the command itself. Do NOT add a `setup-uv` step to `release.yml`. (Fix from openedx/XBlock#928, released as XBlock 6.3.0.)
- Always read the current state of files before modifying them.
- **Never touch a file that already satisfies its phase requirement.** If something is already implemented correctly, skip it entirely — do not rewrite, reformat, or "improve" it.
- **Never modify an existing working file** (e.g. `release.yml`, `ci.yml`, `pyproject.toml` sections) unless (a) it is missing a specific checklist item, or (b) the user explicitly asks for a change. When in doubt, ask first.
- If the repo already has `uv.lock`, do not run `uv lock` unless the `pyproject.toml` dependencies actually changed as part of this migration. Re-running `uv lock` unnecessarily can downgrade pinned package versions.
- `license` must use SPDX string format (PEP 639): `license = "AGPL-3.0"` + `license-files = ["LICENSE.txt"]` — NOT the old `license = { text = "..." }` inline table.
- `[tool.setuptools_scm]` must include `version_scheme = "only-version"` and `local_scheme = "no-local-version"` — the local scheme prevents dirty version suffixes from being rejected by PyPI.
- `[tool.uv]` must include `package = true` to explicitly mark the project as a package (not a bare workspace root).
- Two framework-version dependency groups (e.g. `test` + `django42`) can coexist in a single `uv.lock` using `[tool.uv].conflicts`. Always use this instead of falling back to `uv-venv-runner`. Add a comment explaining how to update the matrix.
- `test` group must have the current default framework version pinned directly (e.g. `Django>=5.2,<6.0`). Legacy versions get their own group (e.g. `django42`).
- `quality` and `doc` groups must include `{include-group = "test"}` so linting/docs run with the same Django version.
- `ci` group must be minimal — just `tox` and `tox-uv`. CI installs it via `uv sync --group ci`, then calls `uv run tox`.
- `dev` group should be `quality` + `ci` + `doc` includes — not a chain through `ci`.
- CI workflows must add `uv sync --group ci` step before `uv run tox`.
- Makefile `upgrade` target must use `uv run --with edx-lint edx_lint write_uv_constraints pyproject.toml` — no pre-install of edx-lint needed.
- Makefile `requirements` target must NOT include `uv tool install tox --with tox-uv` — this installs an unpinned global tox outside `uv.lock`. Since `tox` and `tox-uv` are in the `ci`/`dev` dependency groups, use `uv run tox` instead of bare `tox` everywhere.
- All GitHub Actions must be pinned to full commit SHAs (not floating tags like `@v6`). Always use `@<sha> # vX.Y.Z` format. The current SHA for `astral-sh/setup-uv` v8.1.0 is `08807647e7069bb48b6ef5acd8ec9567f424441b`.
- `astral-sh/setup-uv` handles Python installation — do NOT add a separate `actions/setup-python` step. Pass `python-version` and `enable-cache: true` to `setup-uv` instead.
- CI jobs that only run tests must have `permissions: contents: read` — least privilege.
- `release.yml` `run_tests` job must include `secrets: inherit` so `CODECOV_TOKEN` is passed to the called reusable workflow, otherwise Codecov upload silently fails during the release pipeline.
- Matrix job names must be `name: ${{ matrix.toxenv }}` (not a static string) so each parallel job is identifiable by name in the GitHub UI when it fails.
- `[tool.setuptools.exclude-package-data]` must use **kebab-case** (hyphen, not underscore). `[tool.setuptools.exclude_package_data]` (underscore) is not recognized by the setuptools schema and causes a build failure: `configuration error: tool.setuptools must not contain {'exclude_package_data'} properties`.
- Do NOT invent a `fail_under` coverage threshold if the original `.coveragerc` did not have one. Only migrate an existing threshold — never add one that wasn't there before.
- Remove hardcoded `__version__` from package `__init__.py` when migrating to `setuptools-scm`. The hardcoded value becomes permanently stale after the first new tag. Replace with `importlib.metadata` or omit entirely.

## Checklist (output at end)

```
Phase 1 - Metadata
[ ] pyproject.toml has [project] with static dependencies
[ ] setuptools-scm configured (no root setting)  [library repos only]
[ ] [tool.setuptools.exclude-package-data] added (kebab-case) to exclude test files from wheel
[ ] Hardcoded __version__ removed from __init__.py; docs/conf.py updated to use importlib.metadata
[ ] fail_under only present in [tool.coverage.report] if original .coveragerc had one
[ ] setup.py deleted
[ ] setup.cfg deleted

Phase 2 - Dependency management
[ ] [dependency-groups] covers expected groups (library: test-base/test/quality/doc/ci/dev;
    application: base/test/quality/doc/assets/semgrep/bundled/ci/dev)
[ ] [tool.edx_lint].uv_constraints added (even if empty), edx_lint write_uv_constraints run
[ ] uv.lock generated and committed
[ ] requirements/ directory deleted
[ ] tox.ini uses tox-uv>=1 and uv-venv-lock-runner
[ ] pyproject.toml: framework deps have lower bound (e.g. django>=4.2), framework classifiers added
[ ] pyproject.toml: [tool.ruff.lint] uses expanded select (E/W/F/I/B/C4/UP/DJ) with ignore E501
[ ] pyproject.toml: [tool.ruff.lint.isort] has known-third-party
[ ] pyproject.toml: [tool.ruff.format] has quote-style and indent-style
[ ] pyproject.toml: [tool.coverage.run] has omit patterns
[ ] pyproject.toml: [tool.coverage.report] has exclude_lines and show_missing = true
[ ] Makefile: requirements uses uv sync --group dev only (no uv tool install tox)
[ ] Makefile: lint/test/docs targets use uv run tox (not bare tox)
[ ] CI workflow: setup-uv pinned to SHA, no separate setup-python step
[ ] CI workflow: matrix job name is ${{ matrix.toxenv }}
[ ] CI workflow: jobs have permissions: contents: read
[ ] release.yml: run_tests job has secrets: inherit and permissions: contents: read

Phase 3 - Semantic release (library repos only — skip for application repos)
[ ] [tool.semantic_release] in pyproject.toml with build_command
[ ] release.yml workflow added
[ ] commitlint.yml workflow added
[ ] User reminded to configure PyPI trusted publisher (OIDC)
```
