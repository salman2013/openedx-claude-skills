---
name: modernize-python-tooling
description: Migrate an openedx Python package from legacy tooling (pip-tools, setup.cfg/setup.py) to modern tooling (uv, pyproject.toml PEP 621/735, python-semantic-release).
---

Migrate an openedx Python package from legacy tooling (pip-tools, setup.cfg/setup.py) to modern tooling (uv, pyproject.toml PEP 621/735, python-semantic-release).

## Usage
```
/modernize-python-tooling [repo-path]
```
If no path is provided, operate on the current working directory.

## Context

The openedx org is standardizing Python tooling. Reference implementation: `openedx/sample-plugin` — specifically `backend/pyproject.toml`, `backend/tox.ini`, `backend/Makefile`, `.github/workflows/backend-ci.yml`.

**Qualifying repos** must:
- Be a Python package published (or intended) to PyPI
- Use `pip-compile` / `requirements/*.txt`
- Use `setup.py` or `setup.cfg` (or `pyproject.toml` with `dynamic = ["dependencies"]`)

## Behavior

Work through the three phases in order. At the start, read existing `setup.cfg`, `setup.py`, `pyproject.toml`, `tox.ini`, `Makefile`, `requirements/*.in`, and `.github/workflows/*.yml` to understand current state before making changes.

**CRITICAL — assess before touching anything:**
Before making any change, compare the current state of each file against the phase checklist. For every item, mark it as already done or missing. Only implement what is genuinely missing. Never modify a file that already satisfies its phase requirement. If the user asks why you are changing a working file, stop and explain — do not proceed without their confirmation.

---

### Phase 1 — Consolidate metadata into `pyproject.toml`

1. Read `setup.cfg` / `setup.py` to extract: `name`, `version`, `description`, `long_description`, `author`, `license`, `classifiers`, `python_requires`, `install_requires`, `extras_require`, `packages`, `entry_points`
2. Create/update `pyproject.toml`:
   - `[build-system]`: use `setuptools>=64` and `setuptools-scm`
   - `[project]`: static `dependencies` list (not dynamic from requirements file); `dynamic = ["version"]`
   - `[tool.setuptools_scm]`: omit `root` setting (unlike sample-plugin which uses a subdirectory)
   - `[tool.setuptools.packages.find]`: set `where` if needed
3. Delete `setup.py` and `setup.cfg`

**Key rule:** `dependencies` must be a static list in `[project]`, not `dynamic = ["dependencies"]`.

---

### Phase 2 — Switch dependency management to uv

1. Read all `requirements/*.in` files to understand current deps
2. Add `[dependency-groups]` to `pyproject.toml` with standard groups:
   - `test-base`: pytest, coverage, etc. (no version pins — let uv.lock handle it)
   - `test`: `{include-group = "test-base"}` + any test extras
   - `quality`: pylint, ruff, edx-lint, etc.
   - `doc`: Sphinx and doc-building deps
   - `ci`: `{include-group = "test"}` + `{include-group = "quality"}` + tox, tox-uv
   - `dev`: `{include-group = "ci"}` + `{include-group = "doc"}` + dev conveniences
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
   - `requirements`: run `uv sync --group dev`
9. Update `.github/workflows/*.yml` CI files:
   - Add `astral-sh/setup-uv` action step
   - Replace `pip install -r requirements/ci.txt` with `uv sync --group ci`
   - Replace bare `tox` calls with `uv run tox`

---

### Phase 3 — Add semantic-release

1. Add `[tool.semantic_release]` to `pyproject.toml`:
   - Set `version_toml` to point at `[project].version` (but version is dynamic via setuptools-scm — use `build_command` to set `SETUPTOOLS_SCM_PRETEND_VERSION`)
   - `build_command`: must export `SETUPTOOLS_SCM_PRETEND_VERSION` before building
   - Do NOT override `minor_tags` unless explicitly required
2. Create `.github/workflows/release.yml` with this exact structure:
   - `run_tests` job: `uses: ./.github/workflows/python-tests.yml` (reusable — no inline matrix)
   - `release` job (`needs: run_tests`, `if: github.ref_name == 'main'`):
     - `concurrency` with `cancel-in-progress: false`
     - Checkout with `ref: ${{ github.ref_name }}` + `git reset --hard ${{ github.sha }}`
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
- **Phase 3 commit:** `chore: add python-semantic-release and commitlint workflows`

Use conventional commit format. Do not squash phases into a single commit.

---

## Important rules

- When editing `README.rst`, do NOT introduce new RST heading levels that conflict with existing ones. RST infers heading levels from the first occurrence of each underline character in the document. Adding a new sub-section level (e.g. `^^^^`) under a section that already has sibling sections using a different character (e.g. `----`) will cause `doc8` to report `D000 Title level inconsistent`. Always match the underline character used by existing sub-sections at the same depth.
- `[tool.uv].constraint-dependencies` is machine-managed — NEVER edit it directly. Repo-specific overrides go in `[tool.edx_lint].uv_constraints` only.
- Use `uv run tox` in CI — `uv sync` does not put tools on PATH.
- If new framework version drops older Python, bump `requires-python` and remove that Python from tox envlist.
- Do NOT set `root` in `[tool.setuptools_scm]` (unlike sample-plugin which needs it for subdirectory layout).
- Do NOT set `minor_tags` in `[tool.semantic_release]` unless the user explicitly requests it.
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

## Checklist (output at end)

```
Phase 1 - Metadata
[ ] pyproject.toml has [project] with static dependencies
[ ] setuptools-scm configured (no root setting)
[ ] setup.py deleted
[ ] setup.cfg deleted

Phase 2 - Dependency management
[ ] [dependency-groups] covers test-base, test, quality, doc, ci, dev
[ ] [tool.edx_lint].uv_constraints added (even if empty), edx_lint write_uv_constraints run
[ ] uv.lock generated and committed
[ ] requirements/ directory deleted
[ ] tox.ini uses tox-uv>=1 and uv-venv-lock-runner
[ ] Makefile updated (upgrade, requirements targets)
[ ] CI workflows use astral-sh/setup-uv and uv run tox

Phase 3 - Semantic release
[ ] [tool.semantic_release] in pyproject.toml with build_command
[ ] release.yml workflow added
[ ] commitlint.yml workflow added
[ ] User reminded to configure PyPI trusted publisher (OIDC)
```
